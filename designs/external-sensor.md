# dbus-sensors 中的 ExternalSensor

作者：Josh Lehan[1]

其他貢獻者：Ed Tanous、Peter Lundgren、Alex Qiu

建立日期：2021 年 3 月 19 日

## 簡介

在 OpenBMC 中，_dbus-sensors_[2] 套件包含一整套的感測器守護程序（sensor daemon）。
每個守護程序負責監控某一特定類型的感測器。本文件說明新增另一個感測器守護程序
_ExternalSensor_ 的理由與動機，並提供一些使用範例。

## 動機

_dbus-sensors_ 中已有 10 個現有的感測器守護程序。為什麼還要再加一個感測器守護程序？

- 現有的感測器守護程序多半綁定於它所量測的某個特定物理量，例如溫度，且以此方式
  寫死在程式中。由外部更新的感測器沒有這種限制，應該要有足夠的彈性，能量測
  OpenBMC 目前支援的任何物理量。

- 幾乎所有現有的感測器守護程序，都是透過讀取本地硬體（通常是讀取 Linux 核心
  _hwmon_[3] 子系統所提供的虛擬檔案）來取得它們發布到 D-Bus 上的感測器數值。目前沒有任何
  一個守護程序是以「接受從外部來源推入的數值」為設計目的。雖然確實存在一些
  除錯功能可以為其他感測器守護程序加入這項能力[25]，但這並非它們設計時的主要用途。

- 即使使用某個现有守護程序的除錯功能，该守護程序仍然需要一份綁定到已辨識硬體的
  有效設定（由 _entity-manager_[4] 偵測到），守護程序才能正確地完成自我初始化並
  參與 OpenBMC 軟體堆疊。

- 正如同現有的感測器守護程序需要偵測並正確反映其底層硬體故障，_ExternalSensor_
  也希望能夠偵測並正確反映外部來源未按時提供感測器更新的情形。這是一項新功能，
  無法乾淨地套用到任何現有感測器守護程序的架構中，因此針對這種行為，新增一個
  新的守護程序才是正確的選擇。

基於這些理由，_ExternalSensor_ 已經被加入[5]，成為 _dbus-sensors_ 中的第十一個
感測器守護程序。

## 設計

經過一些討論後，發表了一個概念驗證（proof-of-concept）的 _HostSensor_[6]。
它只是一個骨架（stub），但揭示了仍能完整初始化並參與 OpenBMC 軟體堆疊的
最小實作。_ExternalSensor_ 就是以這個範例 _HostSensor_，以及現有的最簡單
感測器守護程序之一 _HwmonTempSensor_[7]，作為參考基準建構而成。

依目前的寫法，在初始化期間驗證完參數之後，_ExternalSensor_ 基本上沒有任何工作
要做。主迴圈大多處於idle 狀態，多半阻塞在 Boost ASIO[8] 函式庫中，隨著 D-Bus
請求到來時加以處理。這裡利用了底層 _Sensor_[9] 類別的功能，該類別已經包含了
從外部來源接收數值所需的 D-Bus 掛鉤（hooks）。

外部來源的一個例子是 IPMI 服務[10]，它透過 IPMI 的 "Set Sensor Reading"
指令[11] 從主機接收數值。_ExternalSensor_ 的設計是來源無關（source-agnostic）的，
所以不論這是 IPMI、Redfish[12]，或未來其他東西都沒關係，只要它們以類似的方式
透過 D-Bus 傳遞即可。

### 逾時（Timeout）

逾時功能是把 _ExternalSensor_ 與其他感測器守護程序區別出來的主要功能。一旦
外部來源開始提供更新，就預期該外部來源要持續地、按時地提供更新。每次更新都會
以浮點數值，按照所有感測器守護程序慣常的方式，正確地發布到 D-Bus 上。

這裡使用了計時器，也就是其他感測器守護程序用來輪詢其硬體的同一個 Boost
ASIO[13] 計時器機制，只是在此處用來管理距離最近一次已知良好的外部更新已經
過了多久。當計時器逾時，該感測器數值就會被視為過期（stale），並被替換成
浮點數的靜默 _NaN_[14]。

### NaN

使用浮點數 _NaN_ 的好處在於，它是感測器有效浮點數值的即插即用（drop-in）替代品。
先前 OpenBMC 感測器 "Value" schema 從整數改為浮點數，其中一個細微的差異是：
這個欄位基本上現在變成可為空值（nullable）的了。與其必須任意挑選某個整數值
（例如 -1、9999 之類）來表示「無效」，浮點數明確地有 _NaN_ 來表示這件事。
因此，不會有被誤認為有效感測器數值的混淆可能，因為 _NaN_ 字面上就是「不是
數字」（not a number），所以不可能被誤解析成有效的感測器讀值。這樣也省去必須
新增第二個欄位來可靠地表示有效性，而那會破壞現有的 schema[15]。

用 _NaN_ 來表示過期的替代做法是使用時間戳記，但這會引入在 OpenBMC 內解析與
比較時間戳記的複雜性，以及隨之而來的各種細微難題[16]。更何況，新增第二個欄位
可能需要第二則 D-Bus 訊息來更新，而 D-Bus 訊息在運算上相當昂貴[17]，應該盡量
少用。像感測器這類週期性的東西，會發出定期更新，很容易就造成頻繁的 D-Bus
流量，因此應該盡可能保持精簡。最後，變更 Value schema 會造成巨大的影響範圍
（blast radius），不論在設計上還是程式碼上都一樣，需要進行一次遠超出
_ExternalSensor_ 所需範圍的大規模重構。

### 設定

為配合 _ExternalSensor_ 使用而設定感測器，應該採用與其他感測器守護程序搭配使用時
慣常的做法[18]，也就是：一個 JSON 字典，作為要由 _entity-manager_ 讀取的 JSON
設定檔中 "Exposes" 陣列裡的一個元素。在該 JSON 字典中，有效的名稱列於下方。
除非特別標註為選擇性（optional），否則這些都是必填參數。下方列為 "Numeric"
的欄位，表示它可以是整數，也可以是有效的浮點數。

- "Name"：字串（String）。感測器名稱，也就是此感測器所使用的名稱。這是
  `entity-manager` 設定以及所產生之 D-Bus 物件路徑的必填组成部分。

- "Units"：字串（String）。這個參數是 _ExternalSensor_ 獨有的。由於
  _ExternalSensor_ 不綁定於任何特定實體硬體，它可以使用 OpenBMC 所支援的
  任何物理量進行量測。這個字串會透過一張查詢表[19] 轉換為另一個字串，並構成
  D-Bus 物件路徑的另一個必填组成部分。

- "MinValue"：數值（Numeric）。此感測器的最小有效值。雖然 _ExternalSensor_ 本身
  並不直接使用它，但對像是 IPMI 這類服務而言，這是很有價值的提示，因為它們需要
  知道感測器的最小與最大有效值，才能準確地換算其回報範圍。由於 _ExternalSensor_
  不綁定於某一特定物理量，因此最小值與最大值都沒有合適的預設值。所以，不同於
  這個參數屬於選擇性的其他感測器守護程序，在 _ExternalSensor_ 中它是必填的。

- "MaxValue"：數值（Numeric）。此感測器的最大有效值。它的處理方式與 "MinValue"
  類似。

- "Timeout"：數值（Numeric）。這個參數是 _ExternalSensor_ 獨有的。它是逾時值，
  單位為秒。如果經過這段時間，仍然沒有從外部來源透過 D-Bus 收到新的更新，此
  感測器就會被視為過期。此感測器的值會被替換成浮點數 _NaN_，如前述說明。這個
  欄位是選擇性的。若未提供，此感測器的逾時功能將被停用（因此它永遠不會被視為
  過期）。

- "Type"：字串（String）。必須精確地為 "ExternalSensor"。這個字串由
  _ExternalSensor_ 在初始化期間用來向 _entity-manager_ 取得設定資訊。這個字串
  就是用來區分要提供給 _ExternalSensor_ 的 JSON 段落，與要提供給其他
  _dbus-sensors_ 感測器守護程序的 JSON 段落。

- "Thresholds"：JSON 字典。這個欄位是選擇性的。它會在初始化期間傳遞給主
  _Sensor_ 類別，與其他感測器守護程序類似。除此之外，_ExternalSensor_ 並不會
  使用它。

- "PowerState"：字串（String）。這個欄位是選擇性的。與 "Thresholds" 一樣，它會在
  初始化期間傳遞給主 _Sensor_ 類別。

以下是一個範例。由這個段落所建立的感測器會形成這個物件路徑：
/xyz/openbmc_project/sensors/temperature/HostDevTemp

```json
{
  "Name": "HostDevTemp",
  "Units": "DegreesC",
  "MinValue": -16.0,
  "MaxValue": 111.5,
  "Timeout": 4.0,
  "Type": "ExternalSensor"
}
```

設定中可以有多個 _ExternalSensor_ 感測器。感測器數量沒有固定上限，除了像是
IPMI 這類服務所支援的數量之外。

## 實作

就目前的情況而言，_ExternalSensor_ 已經上線運作[20]。不過，逾時功能最初是在
IPMI 層實作的。經過進一步調查後發現，IPMI 是實作這個功能的不當位置，它應該被
移入 _ExternalSensor_ 本身[21]。原本認為逾時功能會是一項對所有 IPMI 感測器都有用的
有用增強功能，然而，幾乎所有外部感測器更新的預期用法都是一次性的調整（例如，
有人想要改變某個電壓調節器的設定，或風扇轉速的設定）。在這種情況下，逾時功能
不但不必要，反而會礙事，並且需要額外撰寫程式碼[22] 來補償預期外的 _NaN_ 值。
只有預定要與 _ExternalSensor_ 搭配使用的感測器，才會預期從外部來源收到持續的
週期性更新，所以把這個逾時功能移入 _ExternalSensor_ 是合理的。這個變更還有一個
好處，就是讓 _ExternalSensor_ 不再依賴 IPMI 作為外部更新的唯一來源。

不過，將逾時功能-generalize（一般化）到 _ExternalSensor_ 的一項挑戰是：既有的
_Sensor_ 基底類別目前不允許客製化它現有的 D-Bus setter 掛鉤。加入這個功能並不
複雜[23]。其中一個限制是：既有的 _Sensor_ 類別依設計會丟棄與現有感測器數值
重複的更新。但為了與 _ExternalSensor_ 搭配使用，我們希望辨識出所有收到的更新，
即使是重複的也一樣，因為牠們對於餵養看門狗（pet the watchdog）、避免不小心觸發
逾時功能來說很重要。不過，避免為重複的讀值多餘地傳送 D-Bus 的 _PropertiesChanged_
事件，仍然很重要。

逾時值最初是一個編譯期常數。如果 _ExternalSensor_ 要成功成為一個通用工具，這
必須是可設定的。再加一個可設定的參數[24] 來接受這個逾時值並不困難，如上方
"Parameters" 所示。

然而，最艱難的任務，莫過於讓它在上游被接受。如果你正在阅讀本文，那很可能它
已經成功了！

## 附註

[1]: https://gerrit.openbmc.org/q/owner:krellan%2540google.com
[2]: https://github.com/openbmc/dbus-sensors/blob/master/README.md
[3]: https://www.kernel.org/doc/html/latest/hwmon/index.html
[4]: https://github.com/openbmc/entity-manager/blob/master/README.md
[5]: https://gerrit.openbmc.org/c/openbmc/dbus-sensors/+/36206
[6]: https://gerrit.openbmc.org/c/openbmc/dbus-sensors/+/35476
[7]: https://github.com/openbmc/dbus-sensors/blob/master/src/HwmonTempMain.cpp
[8]: https://think-async.com/Asio/
[9]: https://github.com/openbmc/dbus-sensors/blob/master/include/sensor.hpp
[10]:
  https://github.com/openbmc/docs/blob/master/architecture/ipmi-architecture.md
[11]:
  https://www.intel.com/content/www/us/en/servers/ipmi/ipmi-intelligent-platform-mgt-interface-spec-2nd-gen-v2-0-spec-update.html
[12]: https://www.dmtf.org/standards/redfish
[13]:
  https://www.boost.org/doc/libs/1_75_0/doc/html/boost_asio/overview/timers.html
[14]: https://anniecherkaev.com/the-secret-life-of-nan
[15]:
  https://github.com/openbmc/phosphor-dbus-interfaces/blob/master/yaml/xyz/openbmc_project/Sensor/Value.interface.yaml
[16]: https://cr.yp.to/proto/utctai.html
[17]: https://github.com/openbmc/openbmc/issues/1892
[18]:
  https://github.com/openbmc/entity-manager/blob/master/docs/my_first_sensors.md
[19]: https://github.com/openbmc/dbus-sensors/blob/master/src/SensorPaths.cpp
[20]: https://gerrit.openbmc.org/c/openbmc/dbus-sensors/+/36206
[21]: https://gerrit.openbmc.org/c/openbmc/dbus-sensors/+/41398
[22]: https://gerrit.openbmc.org/c/openbmc/dbus-sensors/+/39294
[23]: https://gerrit.openbmc.org/c/openbmc/dbus-sensors/+/41394
[24]: https://gerrit.openbmc.org/c/openbmc/entity-manager/+/41397
[25]: https://gerrit.openbmc.org/c/openbmc/dbus-sensors/+/16177
