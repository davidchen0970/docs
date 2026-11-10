# 基於 GPIO 的纜線在位偵測

作者：Chu Lin <linchuyuan@google.com>

建立日期：2021-07-29

## 問題描述

這個新 daemon 設計的立意在於回報基於 GPIO 的纜線在位（presence）狀態。IPMI
目前不支援將離散值（discrete values）分組，而且最多只能支援 255 個 sensor ID。
在一個有 25 條纜線、255 個 sensor ID 的系統上，若每個插槽佔用一個 sensor，就會
用掉可用空間的 10%，相當浪費。因此，我們需要一個能把多個在位狀態
收納進更少 SDR ID 的解決方案。

## 背景與參考資料

1. <https://www.intel.com/content/www/us/en/products/docs/servers/ipmi/ipmi-second-gen-interface-spec-v2-rev1-1.html>
2. <https://www.dmtf.org/sites/default/files/Redfish_Cable_Management_Proposal_WIP_04-2021.pdf>

## 需求

1. OpenBMC 的 IPMI 介面應支援暴露纜線的在位狀態。
2. OpenBMC 的 dbus 介面應與 Redfish 相容。

## 建議設計

我們將引入 `Inventory.Item.Cable` 作為一個新的 dbus 介面，用於查詢纜線物件。
我們還需要引入一個新的 dbus daemon 來收集在位（Presence）狀態。這個新 daemon
將位於 openbmc/dbus-sensors 中。它的執行期設定（runtime configuration）將在啟動
時由 EntityManager 提供。一個提議的設定檔如下所示：

```json
{
  "Exposes": [
    {
      "Name": "cable0",
      "GpioLine": "osfp0",
      "Polarity": "active_low",
      "Type": "GPIOCableSensing",
      "FaultLedGroup": ["attention"]
    },
    {
      "Name": "cable1",
      "GpioLine": "osfp1",
      "Polarity": "active_high",
      "Type": "GPIOCableSensing",
      "FaultLedGroup": ["attention"]
    }
  ]
}
```

`Name` 屬性是設定檔中每個物件的唯一名稱。`GpioLine` 是來自 device tree 的 gpio
line label。`Polarity` 用來告訴 daemon 是否需要反轉 GPIO 訊號。設定檔中的每一個
項目都會被轉換成 dbus 上的一個物件。此外，daemon 會以 10 秒為間隔輪詢（poll）
這些物件的 gpio 在位訊號。當纜線沒有確實插入到位時，daemon 會致動（assert）對應
的 fault led group。這應該會讓對應的 led 閃爍或亮起，端視 phosphor-led-manager
如何設定。若 `FaultLedGroup` 為空，則不採取任何動作。

在 IPMI 這一側，為了替 ipmi 節省 SDR ID，在位狀態會被分組到更少的 SDR ID 中。
以下列範例來說明，

```text
└─/xyz
  └─/xyz/openbmc_project
    └─/xyz/openbmc_project/inventory
      └─/xyz/openbmc_project/inventory/item
        └─/xyz/openbmc_project/inventory/item/cable0
        └─/xyz/openbmc_project/inventory/item/cable1
```

在這個範例中，我們同時有 cable0 與 cable1，預期會把它們分組在一起。分組處理程式
（grouping handler）會先移除 SDR 名稱末尾所有的索引數字。在此例中，SDR 名稱是
cable0 與 cable1。接著，cable0 與 cable1 都會變成 cable。因此，在移除末尾索引後
名稱相同的物件，就代表它們需要被分組。分組之後，新的 SDR 名稱是 cable[0-1]。
這個 SDR 名稱表示此 SDR 擁有 cable0 與 cable1 的在位狀態。同時，cable0 的
bit 位置是 0，cable1 的 bit 位置是 1。當在位狀態超過 14 個時，分組處理程式會自動
改用一個新的 SDR。舉例來說，若有 20 條纜線、索引從 0 到 19，我們會看到兩個 SDR：
一個是 cable0-13，另一個是 cable[14-19]。若物件路徑沒有被使用者加上索引，它將
佔用一個 SDR ID。

```text
ipmitool sdr list event
# /xyz/openbmc_project/inventory/item/cdfp0 to
# /xyz/openbmc_project/inventory/item/cdfp3
cdfp[0-3]        | Event-Only        | ns
# /xyz/openbmc_project/inventory/item/osfp0 and
# /xyz/openbmc_project/inventory/item/osfp1
osfp[0-1]        | Event-Only        | ns
```

另一方面，若設定檔中的物件名稱沒有加上索引，例如
`/xyz/openbmc_project/inventory/item/cable`，分組處理程式就不會試圖把它與任何
東西分組，並為它的在位狀態使用 1 個 SDR ID。以下是一個輸出範例。

```text
ipmitool sdr list event
cdfp[0-3]        | Event-Only        | ns
# /xyz/openbmc_project/inventory/item/osfp
osfp             | Event-Only        | ns
osfp[1-2]        | Event-Only        | ns
```

在 fetch 操作期間，一旦 ipmi daemon 收到客戶端請求，它會把該請求轉給 dynamic
ipmi commands。在 get 方法中，範圍字串會從末尾的方括號（square bracket）中擷取
出來。接著即可根據索引重組出原本的 dbus 路徑。然後，在從各個獨立的 dbus 物件
收集 OperationalStatus 之後，便可以建構出 SDR 的致動值（assertion value）。

這個設計也與 Redfish daemon 相容。Redfish 可以呼叫 GetSubTree 方法來列出 cable
介面底下的所有實例（instances）。此外，Redfish 並沒有 256 個 SDR ID 的限制，
因此不需要實作分組機制，它應該可以開箱即用。

## 曾考慮的替代方案

- 探索改用 type12 record 而非離散感測器（discrete sensors）來報待在位狀態的
  選項。此處的挑戰在於 type12 record 同樣有 255 個 ID 的上限，而且它不支援把多個
  在位狀態堆疊在單一 ID 之中。

- 我們也可以讓使用者在設定檔中自行定義如何為不同的纜線分組。然而，這要求使用者
  深入了解 ipmid 的運作方式。由 ipmi daemon 自行處理這一切會更好。

- 相較於采用 sleep-poll（休眠輪詢）機制，更好的做法是對 gpio 訊號設立事件監聽器
  （event listener）。然而，gpio_event_wait api 要求所有的 line 都必須來自同一顆
  chip。請參見
  <https://github.com/brgl/libgpiod/blob/cc23ef61f66d5a9055b0e9fbdcfe6744c8a637ae/lib/core.c#L993>
  我們可以為每顆 chip 各生成一個執行緒來監聽，但我認為我們不該讓如此簡單的一個
  daemon 增加這種複雜度。

- 輪詢間隔為 10 秒。對某些其他專案而言，這是否太長／太短？目前，我們先把它做成
  一個編譯選項。

- IPMI 名稱字串最多只能有 16 個字元。使用方括號會佔掉 2 個字元。然而，使用方括號
  能更清楚地標示索引字串的起始位置。因此，索引字串將以方括號包圍。

## 測試

測試可以透過自動化或手動測試來完成，以驗證：

- GpioCableSevice 所報得的在位狀態，與 linux gpioget 指令的結果相符。

- 給定來自 `ipmitool sdr list event` 的 SDR 名稱，例如 cable[0-1]，確認第 zeroth
  bit 與 dbus 上物件 `/xyz/openbmc_project/inventory/item/cable0` 的在位狀態相符，
  且第一個 bit 與物件 `/xyz/openbmc_project/inventory/item/cable` 的狀態相符。

- 設定檔中未加索引（unindex）的項目應有其專屬的 SDR ID。舉例來說，
  `/xyz/openbmc_project/inventory/item/cable` 不應被分到另一個群組。同時，只有
  zero bit 會被用來報待在位狀態。

- 當纜線插入或拔出系統後，纜線在位狀態必須在 10 秒內隨之改變。
