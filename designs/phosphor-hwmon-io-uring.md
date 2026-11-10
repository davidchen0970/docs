# 在 BMC 中使用 io_uring 進行非同步感測器讀取

作者：Jerry Zhu ([jerryzhu@google.com](mailto:jerryzhu@google.com))

其他貢獻者：

- Brandon Kim ([brandonkim@google.com](mailto:brandonkim@google.com), brandonk)
- William A. Kennington III ([wak@google.com](mailto:wak@google.com), wak)

建立日期：2021 年 6 月 9 日

## 問題描述

目前，OpenBMC 中有些程式碼會為感測器執行 I2C 讀取，而這些讀取作業可能花費比預期更長的時間。這些 IO 作業目前是同步的，因此可能會阻塞其他功能，例如 IPMI。本專案將著手檢視目前可能存在此缺點的 OpenBMC 程式庫（特別是
[phosphor-hwmon](https://github.com/openbmc/phosphor-hwmon)），並使用新的 io_uring
程式庫加入非同步介面。

## 背景與參考資料

io_uring 是 Linux I/O 介面的全新非同步框架（於 Linux kernel 5.1 加入，建議使用 5.10）。它是先前非同步 IO（稱為 AIO）的升級版本，而 AIO 在 OpenBMC 感測器讀取的使用情境下有其限制。

[brandonkim@google.com](mailto:brandonkim@google.com) 先前已建立了一種方法，可在感測器未能盡快回報失敗時，防止其阻塞 phosphor-hwmon 程式庫中所有其他的感測器讀取與 D-Bus
（[變更連結](https://gerrit.openbmc.org/c/openbmc/phosphor-hwmon/+/24337)）。
Google 內部 BMC 的工作也已著重於將 io_uring 程式庫引入其程式碼。

## 需求

使用 io_uring 時，非同步感測器讀取必須在各 daemon 中維持與目前同步讀取相同的準確度。可能因這個程式庫而受惠的潛在 OpenBMC 程式庫包括：

- [phosphor-hwmon](https://github.com/openbmc/phosphor-hwmon)
- [phosphor-nvme](https://github.com/openbmc/phosphor-nvme)
- [dbus-sensors](https://github.com/openbmc/dbus-sensors)
- 其他任何適當的程式庫

本專案的重點是為 phosphor-hwmon 程式庫加入非同步感測器讀取，這比在 dbus-sensors 中加入非同步感測器讀取更容易實作。

使用者將需要能夠選擇是否要使用這種新的非同步感測器讀取方式，或維持傳統的同步方式。此外，還需要針對每個 daemon 計算使用新 io_uring 程式庫所帶來的效能提升。

## 提案設計

在 phosphor-hwmon 程式庫中，需要修改的主要檔案是
[sensor.cpp](https://github.com/openbmc/phosphor-hwmon/blob/master/sensor.cpp)/[.hpp](https://github.com/openbmc/phosphor-hwmon/blob/master/sensor.hpp)
以及
[mainloop.cpp](https://github.com/openbmc/phosphor-hwmon/blob/master/mainloop.cpp)/[.hpp](https://github.com/openbmc/phosphor-hwmon/blob/master/mainloop.hpp)，
另外還要為感測器讀取的結果加入一層快取。

在目前的 mainloop.cpp 中，負責讀取 hwmon sysfs 條目的 `read()` 函式會遍歷所有感測器，並對每一個感測器呼叫 `_ioAccess->read(...)`；這個作業可能會阻塞。

重構將維持這個遍歷所有感測器的迴圈，但改用 io_uring 包裝器讓讀取作業變為非阻塞。將會使用一層快取來儲存讀取結果，而這層快取將是 mainloop.cpp 中取得感測器讀取值的主要存取點。

```ascii
               Interface Layer
+--------------------------------------------+
|                                            |
|   +------------+         +-------------+   |
|   |            |         |             |   |
|   |   Redfish  |         |     IPMI    |   |
|   |            |         |             |   |
|   +-----+------+         +-------+-----+   |
|         ^                        ^         |
+---------|------------------------|---------+
          |                        |
          v                        v
+---------+------------------------+---------+
|                                            |
|                  DBus                      |
|                                            |
+---------^------------------------^---------+
          |                        |
  +-------v-------+       +--------v-------+
  |               |       |                |
  |phosphor-hwmon |       |  dbus-sensors  |
  |               |       |                |
  +-------^-------+       +--------^-------+
          | <--------------------- | <------- caching layer at this level
 +--------v------------------------v--------+
 |                                          |
 |               Linux kernel               |
 |                                          |
 +----------^---------------------^---------+
            |                     |
       +----v-----+         +-----v----+
       |          |         |          |
       |i2c sensor|         |i2c sensor|
       |          |         |          |
       +----------+         +----------+

```

透過一個旗標變數（很可能會放在每個 hwmon 感測器的 .conf 檔案中），使用者將能基於相容性考量決定是否使用這個新的 io_uring 實作。

## 詳細設計

讀取快取使用一個 {感測器 hwmon 路徑: 讀取結果} 的 `unordered_map` 來實作。讀取結果是一個 struct，用來記錄處理讀取值與處理錯誤時所需的一切資訊。這類資訊包括來自 `open()` 系統呼叫的開啟檔案描述元、發生錯誤時讀取此感測器的剩餘重試次數等等。

在讀取快取中，每次存取某個特定感測器讀取值的呼叫，不僅會回傳快取的值，也會為該感測器提交一個 SQE（submission queue event）給 io_uring；這個 SQE 就像一個會被送往核心的讀取請求。此實作維護著一組感測器集合，用來追蹤任何已存在的提交，以避免同一個感測器提交並重疊多個 SQE；當使用 CQE（completion queue event）成功回傳讀取結果時，該集合物件會被清除。接著 CQE 會被處理，其資訊會更新快取 map。

此實作的非同步特性來自於一次送出所有可能的 SQE 請求（一個非阻塞作業），而不是像同步實作那樣被緩慢的感測器讀取阻塞。核心會處理這些請求，而在下一次感測器讀取迴圈迭代之前，快取會嘗試處理任何已回傳的 CQE，這同樣是一個非阻塞作業。

簡而言之，存取讀取快取中某個「Sensor A」會在底層建立一個讀取請求，盡力在下一次感測器讀取迴圈（預設為 1 秒）透過快取取得「Sensor A」的值之前，先更新「Sensor A」的值。

## 已考慮的替代方案

Linux 確實有一個原生的非同步 IO 介面，單純稱為 AIO；然而它有不少限制。AIO 最大的限制是它只對 un-buffered 讀取支援真正的非同步 IO。此外，IO 提交有多種情況最終可能會阻塞，例如當執行 IO 需要 metadata 時。另外，AIO 的記憶體成本比 io_uring 更為昂貴。

基於這些主要原因，本非同步讀取實作將不考慮使用原生 AIO 程式庫。

## 影響

這個專案最初會影響所有 openbmc/phosphor-hwmon 的 OpenBMC 開發者。它改善了 phosphor-hwmon 的延遲效能；吞吐量也已顯示有所提升（請注意，吞吐量量測比延遲量測更為隨機）。這些效能變化必須在不同機器上做更詳盡的計算。

不會有任何安全性影響。

## 測試

此項變更將使用 gTest 框架，以確保被修改程式庫中原有的程式碼功能維持不變。
