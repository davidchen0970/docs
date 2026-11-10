# OpenBMC 平台遙測

Author: Piotr Matuszczak <piotr.matuszczak@intel.com>

其他貢獻者：

- Pawel Rapkiewicz <pawel.rapkiewicz@intel.com> `pawelr`
- Kamil Kowalski <kamil.kowalski@intel.com>

建立日期：2019-08-07

## 問題描述

伺服器平台上的 BMC 會蒐集大量遙測資料，這些資料必須以乾淨、易讀且標準化的格式呈現。本文件聚焦於透過 Redfish 傳遞的遙測，因為它是平台管理的標準 API。

## 背景與參考

- OpenBMC 平台遙測應善用 DMTF 的 [Redfish Telemetry Model][1]，以透過網路暴露平台遙測。
- OpenBMC 平台遙測應善用 [OpenBMC 感測器架構實作][2]。
- OpenBMC 平台遙測應實作一個稱為 Telemetry 的服務，用來處理指標報告（metric report）與觸發器（trigger）管理。此服務於本文件後續章節說明。
- 雖然我們使用 [hwmon][3] 從實體感測器擷取讀值，但此架構並不依賴它，因為 Telemetry 服務元件依賴的是 [OpenBMC D-Bus 感測器][2]。

## 需求

- 支援 [OpenBMC D-Bus 感測器][2]。這也是一項設計限制，因為 Telemetry 服務要求遙測來源必須以 D-Bus 感測器的形式實作。

## 建議設計

Redfish Telemetry Model 應實作 Telemetry Service，並包含以下的集合資源（collection resources）：

- Metric Definitions - 包含指標的中繼資料（單位、準確度等）
- Metric Report Definitions - 定義指標報告應如何建立（應包含哪些指標、應多久產生一次等）
- Metric Reports - 包含實際的指標報告，內含依 Metric Report Definitions 所產生的遙測資料
- Metric Triggers - 包含套用於特定指標的閾值與動作

下圖展示了 OpenBMC 遙測架構。

```ascii
   +--------------+               +----------------+     +-----------------+
   |hwmon|        |               |Dbus sensors|   |     |Telemetry|       |
   +-----/        |               +------------/   |     +---------/       |
   |              +--filesystem--->                |     |                 |
   |              |               |                |     |                 |
   +--------------+               +--------^-------+     +--------^--------+
                                           |                      |
                                           |                      |
<------------------------------------------v-----^--DBus----------v----------->
                                                 |
                                                 |
+-------+---------------------------------------------------------------------+
|bmcweb |                                        |                            |
+-------/                                        |                            |
|                                                |                            |
| +--------+-------------------------------------v--------------------------+ |
| |Redfish |                                                                | |
| +--------/                                            +---------+-------+ | |
| |                                                     |Existing |       | | |
| | +------------------------------------------------+  |Redfish  |       | | |
| | |Telemetry Service|                              |  |resources|       | | |
| | +----------------+/                              |  +---------/       | | |
| | |  +----------+  +-----------+  +-------------+  |  |   +---------+   | | |
| | |  |  Metric  |  |  Metric   |  |Metric report|  |  |   | Redfish |   | | |
| | |  | triggers |  |definitions|  |definitions  <---------+ sensors |   | | |
| | |  |          |  |           |  |             |  |  |   |         |   | | |
| | |  +----+-----+  +-----+-----+  +------+------+  |  |   +---------+   | | |
| | |       |              |               |         |  |                 | | |
| | |       |              |               |         |  |                 | | |
| | |       |              |               |         |  |                 | | |
| | |       |        +-----v-----+         |         |  |                 | | |
| | |       |        |   Metric  |         |         |  |                 | | |
| | |       +-------->   report  <---------+         |  |                 | | |
| | |                |           |                   |  |                 | | |
| | |                +-----------+                   |  |                 | | |
| | |                                                |  |                 | | |
| | +------------------------------------------------+  +-----------------+ | |
| |                                                                         | |
| +-------------------------------------------------------------------------+ |
|                                                                             |
+-----------------------------------------------------------------------------+
```

telemetry service 元件是 Redfish 的一部分，並實作了 DMTF 的 [Redfish Telemetry Model][1]。Metric report definitions 使用 Redfish 感測器 URI 來建立指標報告。這些感測器也用於取得 URI 對 D-Bus 感測器的對應關係。Redfish Telemetry Service 扮演遙測的呈現層（presentation layer），而 Telemetry 服務則負責從 D-Bus 感測器蒐集指標，並將其暴露為 D-Bus 物件。Telemetry 服務支援不同的監控模式（週期性、變更時、按需），以及彙總運算：

- SINGLE - 目前的讀值
- AVERAGE - 在定義的時間區段內的平均值
- MAX - 在定義的時間區段內的最大讀值
- MIN - 在定義的時間區段內的最小讀值
- SUM - 在定義的時間區段內的讀值總和

計算彙總指標所使用的時間區段，取自每個感測器指標對應的 Redfish Metric Report Definition 資源。

Telemetry 服務支援建立與管理指標報告，報告可包含來自感測器的單一或多個指標。此指標報告會對應到 Redfish Telemetry Service 的 Metric Report。

下圖展示了指標報告的建立與更新流程。

```ascii
+----+              +------+              +---------+                 +-------+
|User|              |bmcweb|              |Telemetry|                 | D-Bus |
+-+--+              +--+---+              +----+----+                 |Sensors|
  |                    |                       |                      +---+---+
  |                    |                       |                          |
+-----------------------------------------------------------------------------+
|Metric report definition flow|                |                          |   |
+-----------------------------+                |                          |   |
| |                    |                       |                          |   |
| |                    |                       |                          |   |
| |    POST request    |                       |                          |   |
| |    with metric     |                       |                          |   |
| |    report          |                       |                          |   |
| |    definition      |                       |                          |   |
| +-------------------->  Invoke AddReport     |  Register for D-Bus      |   |
| |                    |  method on D-Bus      |  sensors                 |   |
| |                    +----------------------->  PropertiesChanged       |   |
| |                    |                       |  signals                 |   |
| |                    |                       +-------------------------->   |
| |                    |                       |-------------------------->   |
| |                    |                       +-------------------------->   |
| |                    |                       |                          |   |
| |  HTTP response     |                       +-+Create Report           |   |
| |  code 201 with     |  Return created       | |D-Bus object            |   |
| |  Metric Report     |  Report D-Bus path    <-+                        |   |
| |  Definition's URI  <-----------------------+                          |   |
| <--------------------+                       |                          |   |
| |                    |                       |                          |   |
| |                    |                       |                          |   |
+-----------------------------------------------------------------------------+
  |                    |                       |                          |
+-----------------------------------------------------------------------------+
|Periodic metric report update flow|           |                          |   |
+----------------------------------+           +-+Metric report           |   |
| |                    |                       | |timer triggers          |   |
| |                    |                       <-+report update           |   |
| |                    |                       |                          |   |
+----------------------------------Optional-----------------------------------+
| |                    |                       |                          |   |
| |  Send report as SSE or push-style event    |                          |   |
| |  using Redfish Event Service (not shown    |                          |   |
| |  here) if configured to do so.             |                          |   |
| <--------------------------------------------+                          |   |
| |                    |                       |                          |   |
+-----------------------------------------------------------------------------+
| |  GET on Metric     |                       |                          |   |
| |  Report URI        |                       |   Sensor's Properties-   |   |
| +-------------------->                       |   Changed signal         |   |
| |                    +-+Map report's URI     <--------------------------+   |
| |                    | |to D-Bus path        |                          |   |
| |                    <-+                     | +----------------------+ |   |
| |                    | Invoke GetAll method  | |Note that sensor's    | |   |
| |                    | on report D-Bus       | |PropertiesChanged     | |   |
| |                    | object                | |signal is asynchronous| |   |
| |                    +-----------------------> |to metric report timer| |   |
| |                    |                       | |This timer is the only| |   |
| |  Return metric     | Return report data    | |thing that triggers   | |   |
| |  report in JSON    <-----------------------+ |metric report update  | |   |
| |  format            |                       | +----------------------+ |   |
| <--------------------+                       |                          |   |
| |                    |                       |                          |   |
+-----------------------------------------------------------------------------+
  |                    |                       |                          |
+-----------------------------------------------------------------------------+
|On change metric report update flow|          |   Sensor's Properties-   |   |
+-----------------------------------+          |   Changed signal         |   |
| |                    |                       <--------------------------+   |
| |                    |                       |                          |   |
| |                    |                       +-+Sensor's signal         |   |
| |                    |                       | |triggers report         |   |
| |                    |                       <-+update                  |   |
| |                    |                       |                          |   |
+----------------------------------Optional-----------------------------------+
| |                    |                       |                          |   |
| |  Send report as SSE or push-style event    |                          |   |
| |  using Redfish Event Service (not shown    |                          |   |
| |  here) if configured to do so.             |                          |   |
| <--------------------------------------------+                          |   |
| |                    |                       |                          |   |
+-----------------------------------------------------------------------------+
| |  GET on Metric     |                       |                          |   |
| |  Report URI        |                       |                          |   |
| +-------------------->                       |                          |   |
| |                    +-+Map report's URI     |                          |   |
| |                    | |to D-Bus path        | +----------------------+ |   |
| |                    <-+                     | |Note that sensor's    | |   |
| |                    | Invoke GetAll method  | |PropertiesChanged     | |   |
| |                    | on report D-Bus       | |signal triggers the   | |   |
| |                    | object                | |report update. It is  | |   |
| |                    +-----------------------> |sufficient that the   | |   |
| |                    |                       | |signal from only one  | |   |
| |  Return metric     | Return report data    | |sensor triggers report| |   |
| |  report in JSON    <-----------------------+ |update.               | |   |
| |  format            |                       | +----------------------+ |   |
| <--------------------+                       |                          |   |
| |                    |                       |                          |   |
+-----------------------------------------------------------------------------+
  |                    |                       |                          |
+-+--------------------+------------------------------------------------------+
|On demand metric report update flow|          |                          |   |
+-+--------------------+------------+          |                          |   |
| |                    |                       |                          |   |
| |  GET on Metric     |                       |                          |   |
| |  Report URI        |                       |                          |   |
| +-------------------->                       |                          |   |
| |                    +-+Map report's URI     |                          |   |
| |                    | |to D-Bus path        |                          |   |
| |                    <-+                     |                          |   |
| |                    |                       |                          |   |
| |                    |  Invoke the Update    |                          |   |
| |                    |  method for report    |                          |   |
| |                    |  D-Bus object         |                          |   |
| |                    +----------------------->                          |   |
| |                    |                       +-+Update method triggers  |   |
| |                    |                       | |report to be updated    |   |
| |                    |                       | |with the latest known   |   |
| |                    |                       | |sensor's readings.      |   |
| |                    |                       | |No additional sensor    |   |
| |                    |                       <-+readings are performed. |   |
+----------------------------------Optional-----------------------------------+
| |                    |                       |                          |   |
| |  Send report as SSE or push-style event    |                          |   |
| |  using Redfish Event Service (not shown    |                          |   |
| |  here) if configured to do so.             |                          |   |
| <--------------------------------------------+                          |   |
| |                    |                       |                          |   |
+-----------------------------------------------------------------------------+
| |                    | Update method call    |                          |   |
| |                    | result                |                          |   |
| |                    <-----------------------+                          |   |
| |                    |                       |                          |   |
| |                    | Invoke GetAll method  |                          |   |
| |                    | on report D-Bus       |                          |   |
| |                    | object                |                          |   |
| |                    +----------------------->                          |   |
| |                    |                       |                          |   |
| |  Return metric     | Return report data    |                          |   |
| |  report in JSON    <-----------------------+                          |   |
| |  format            |                       |                          |   |
| <--------------------+                       |                          |   |
| |                    |                       |                          |   |
+-----------------------------------------------------------------------------+
  |                    |                       |                          |
```

bmcweb 中的 Redfish 實作是無狀態（stateless）的，因此它無法儲存指標報告。所有對指標報告的操作都應在 Telemetry 服務中完成。以 SSE 或 push-style 事件傳送指標報告，應透過 [Redfish Event Service][6] 進行。此步驟標示為選擇性，因為指標報告不一定要設定為透過事件推送其資料。

在按需更新指標報告的情況下，Telemetry 服務不會執行額外的感測器讀取，因為它已經擁有最新的數值——這些數值會透過 D-Bus 感測器的 PropertiesChanged 訊號更新。

### Telemetry 服務於 [D-Bus][4] 上

Telemetry 服務在 D-Bus 上暴露特定的介面。其中一個將用於讀取報告管理。另一個則用於觸發器管理。

### 讀取報告管理

讀取報告管理的 D-Bus 物件：

```ascii
xyz.openbmc_project.Telemetry.ReportManager
/xyz/openbmc_project/Telemetry/Reports
```

`ReportManager` 實作了 D-Bus 介面 [`xyz.openbmc_project.Telemetry.ReportManager`][8] 以進行報告管理。此介面在 phosphor-dbus-interfaces 中有說明。此介面實作了 `AddReport` 方法，用於建立指標報告。報告可包含單一或多個感測器讀值。BMC 儲存報告的方式由該方法的其中一個參數定義。`ReportManager` 物件實作了一個屬性，用於儲存同時支援的最大報告數量。

`AddReport` 方法會回傳新建立報告物件的路徑。報告物件實作了 [`xyz.openbmc_project.Object.Delete`][10] 與 [`xyz.openbmc_project.Telemetry.Report`][9] 介面。[`Delete`][10] 介面定義用於支援移除 Report 物件，而 [`Report`][9] 介面則實作了 Report 管理所需的方法與屬性，以及包含遙測讀值的屬性。每個報告物件都包含其最後一次更新的時間戳。報告物件包含一個結構陣列，每個結構含有帶中繼資料的讀值，以及該指標最後一次更新的時間戳。每個報告也有一個屬性，用於儲存更新間隔（供週期性更新的報告使用）。

### 觸發器管理

觸發器管理的 D-Bus 物件：

```ascii
xyz.openbmc_project.Telemetry.TriggerManager
/xyz/openbmc_project/Telemetry/Triggers
```

`TriggerManager` 支援 `xyz.openbmc_project.Telemetry.TriggerManager` 介面，該介面實作了 `AddTrigger` 方法。此方法應用於為特定指標建立新的觸發器。方法的參數可定義設定觸發器的指標類型（離散型或數值型）。依此設定，此方法會接受不同的觸發器參數集合。

對於離散型指標，觸發器參數包含：

| 欄位 | 類型 | 說明 |
| ---- | ---- | ---- |
| TriggerCondition | enum | 離散型觸發條件： <br> "Changed" - 當指標值變更時觸發； <br> "Specified" - 當指標值變成離散觸發器中所列舉的值之一時觸發。 |
| DiscreteTriggers | array of structures | 離散型觸發器結構的陣列。 |

DiscreteTriggers 陣列的成員：

| 欄位 | 類型 | 說明 |
| ---- | ---- | ---- |
| TriggerId | string | 唯一的觸發器 Id |
| Severity | enum | 嚴重程度： <br> "OK" - 正常<br> "Warning" - 需要注意<br> "Critical" - 需要立即處理 |
| Value | variant | 離散型指標的值，該值構成一個觸發事件。 |
| DwellTime | uint64 | 觸發事件持續存在的毫秒數，超過後才會執行此觸發器所定義的動作。 |

對於數值型指標，觸發器參數包含數值閾值。數值閾值結構最多包含 4 個閾值：upper warning、upper critical、lower warning 與 lower critical。因此它將包含最多 4 個如下所示的結構：

| 欄位 | 類型 | 說明 |
| ---- | ---- | ---- |
| ThresholdType | enum | 數值型觸發器類型： <br> "UpperCritical" - 讀值高於正常範圍且需要立即處理<br>"UpperWarning" - 讀值高於正常範圍且可能需要注意<br>"LowerCritical" - 讀值低於正常範圍且需要立即處理<br>"LowerWarning" - 讀值低於正常範圍且可能需要注意 |
| DwellTime | uint64 | 觸發事件持續存在的毫秒數，超過後才會執行此觸發器所定義的動作。 |
| Activation | enum | 指出會觸發該閾值動作的越界方向：<br> "Increasing" - 當讀值由低於閾值變為高於閾值時觸發動作<br> "Decreasing" - 當讀值由高於閾值變為低於閾值時觸發動作<br> "Either" - 當讀值以上述任一方向跨越閾值時觸發動作 |
| ThresholdValue | variant | 會觸發該閾值的讀值 |

`AddTrigger` 方法也可定義觸發器啟動時的特定動作。觸發器啟動時，允許三種可能的動作：將事件記錄至記錄服務（log service）、透過事件服務傳送事件，以及觸發指標報告更新。

為了將觸發器指派給特定指標，定義了一個指標參數。其結構包含以下資料：

| 欄位 | 類型 | 說明 |
| ---- | ---- | ---- |
| SensorPath | object path | 定義觸發器之感測器的 D-Bus 路徑。 |
| MetricId | string | 包含唯一的指標 id，可對應到 Redfish MetricId。 |
| ReportPath | object path | Telemetry 讀取報告的 D-Bus 路徑，當觸發條件發生時將觸發其更新。這是選擇性的，僅當觸發器動作設定為更新指標報告時才應填入。 |

`AddTrigger` 方法也可設定觸發器的持續性（whether trigger shall be stored in the BMC's non-volatile memory，即觸發器是否應儲存於 BMC 的非揮發性記憶體中）。

`AddTrigger` 方法回傳：

```ascii
String for created trigger - ie. '/xyz/openbmc_project/Telemetry/Triggers/{Domain}/{Name}'
```

如此建立的觸發器實作了 `xyz.openbmc_project.Object.Delete` 與 `xyz.openbmc_project.Telemetry.Trigger` 介面。每個觸發器物件都包含唯讀資訊，說明其為哪種指標類型建立（離散型或數值型）。此資訊決定觸發器物件內儲存了哪些觸發器。

若觸發器是為離散型指標定義，則它包含看起來如下的觸發器資訊：

| 類型 | 說明 |
| ---- | ---- |
| enum | 離散型觸發條件： <br> "Changed" - 當指標值變更時觸發；<br>"Specified" - 當指標值變成離散觸發器中所列舉的值之一時觸發。 |
| array of structures | 離散型觸發器結構的陣列。 |

離散型觸發器結構：

| 類型 | 說明 |
| ---- | ---- |
| string | 唯一的觸發器 Id |
| enum | 嚴重程度： <br> "OK" - 正常<br>"Warning" - 需要注意<br> "Critical" - 需要立即處理 |
| variant | 離散型指標的值，該值構成一個觸發事件。 |
| uint64 | 觸發事件持續存在的毫秒數，超過後才會執行 `ActionType` 中所定義的動作。 |

若觸發器是為數值型指標定義，則它包含數值觸發器的資訊，為下述 4 個結構組成的陣列：

| 類型 | 說明 |
| ---- | ---- |
| enum | 數值型觸發器類型： <br> "UpperWarning"<br>"UpperWarning"<br>"LowerCritical"<br>"LowerWarning" |
| uint64 | 觸發事件持續存在的毫秒數，超過後才會執行 `ActionType` 中所定義的動作。 |
| enum | 指出會觸發該閾值動作的越界方向：<br> "Increasing"<br>"Decreasing"<br>"Either" |
| variant | 會觸發該閾值的讀值 |

觸發器物件也包含關於其定義觸發器之讀值的資訊。其形式為由三個欄位組成的結構。

| 欄位類型 | 說明 |
| -------- | ---- |
| object path | 感測器的 D-Bus 路徑。這是為其定義讀值觸發器之感測器的路徑。 |
| string | 唯一的指標 Id |
| object path | 既有讀取報告的 D-Bus 路徑。當觸發器動作是更新指標報告時，這是必需的。此路徑應指向 Telemetry 服務內既有的讀取報告。 |

### 觸發器操作\*\*

觸發器支援三種操作類型：Log、Event 與 Update。每一種都有各自的處理方式。

1. 對於 Log 動作，事件應記錄至系統 journal。在這種情況下，Telemetry 服務使用 libjournal 將資料寫入系統 journal。接著 Redfish 記錄服務會透過讀取系統 journal 來取得這些資料。全部如圖所示。

   ```ascii
   +---------------------------+
   |bmcweb|                    |         +----------------------+
   +------/    +-----------+-+ |         |Telemetry|            |
   |           |Redfish    | | |         +---------/            |
   |           |log service| | |         |                      |
   |           +-----------/ | |         |                      |
   |           |             | |         |                      |
   |           |             | |         |                      |
   |           +------^------+ |         +-----------+----------+
   +---------------------------+                     |
                      |                              |
                      +----collect----+            event
                        journal entry |      (write to journal)
                                      |              |
          +------------------------------------+     |
          |systemd|                   |        |     |
          +-------/ +----------+  +---+------+ |     |
          |         |journal|  |  |libjournal| |     |
          |         +-------/  <-->          <-------+
          |         |          |  +----------+ |
          |         |          |               |
          |         |          |               |
          |         +----------+               |
          |                                    |
          +------------------------------------+
   ```

2. 對於 Event 動作，Telemetry 服務應使用 [Redfish Event Service][6]，以 push-style 事件或 SSE 的方式傳送事件。

3. 對於 Update 動作，Telemetry 服務會觸發更新，其目標讀取報告由觸發器物件屬性中所含的 D-Bus 路徑指出。此次更新會使讀取報告的 D-Bus 物件發出屬性變更訊號。若 Redfish Metric Report 已設定為串流輸出，這將導致它被串流輸出。

### Redfish Telemetry Service API

Redfish Telemetry Service 應支援 2019.1 版 Redfish schemas 以處理遙測資源。Metric report definitions 決定哪些指標要納入指標報告。Metric definition 會指派給特定的指標類型，並描述該指標應如何解讀。應支援以下的資源 schemas：

- TelemetryService 1.1.2
- MetricDefinition 1.0.3
- MetricReportDefinition 1.3.0
- MetricReport 1.2.0
- Triggers 1.1.1

下圖展示了這些資源之間的關係。

```ascii
 +----------------------------------------------------------------------------+
 |                             Service root                                   |
 +----------------------------------+-------------------------------+---------+
                                    |                               |
                                    |                               |
                                    |                               |
 +----------------------------------v-----------------+  +----------v---------+
 |                                                    |  |Chassis             |
 |                Telemetry Service                   |  |                    |
 |                                                    |  |                    |
 |                                                    |  |  +---------------+ |
 +---------+--------------+------------------+--------+  |  |               | |
           |              |                  |           |  |   Chassis 1   | |
           |              |                  |           |  |               | |
           |              |                  |           |  +---------+-----+ |
           |              |                  |           |            |       |
+----------v--+  +--------v----+  +----------v-----+     +--------------------+
|Triggers     |  |Metric       |  |Metric report   |                  |
|             |  |definition   |  |                |                  |
|             |  | +---------+ |  |                | Reads            |
| +---------+ |  | |Reading  | |  | +-----------+  | ReadingVolts  +--v------+
| |         | |  | |Volts    <------+           +------------------>         |
| |Trigger 1| |  | +---------+ |  | |  Metric   |  |               |         |
| |         | |  |             |  | | report 1  |  | Reads         |  Power  |
| |         | |  | +---------+ |  | |           |  | PowerConsumed |         |
| |         | |  | |         | |  | |           |  | Watts         |         |
| +--+---+--+ |  | |Power    <------+           +------------------>         |
|    |   |    |  | |Consumed | |  | +-----^-----+  |               +----^----+
|    |   |    |  | |Watts    | |  |       |        |                    |
|    |   |    |  | +---------+ |  |       |        |                    |
|    |   |    |  |             |  |       |        |                    |
+-------------+  +-------------+  +----------------+                    |
     |   |                                |                             |
     |   | Triggers report update         |                             |
     |   | (when applicable)              |                             |
     |   +--------------------------------+                             |
     |                                                                  |
     |   Monitors PowerConsumedWatts to check                           |
     |   whether trigger value is exceeded                              |
     +------------------------------------------------------------------+
```

此圖展示了 Redfish 資源之間的關係。指標報告被定義為週期性、按需或變更時產生。Metric Report 中的每個指標都包含指向其 metric definition 的 URI，以及呈現其讀值的 Redfish 感測器。然而，在此呈現層之下，Telemetry 會蒐集 D-Bus 感測器的讀值，並透過 D-Bus 以讀取報告的形式暴露給 Telemetry Service。每個 D-Bus 感測器都會對應到一個 Redfish 感測器。

以下展示了 Telemetry Service 的 Redfish 資源範例。

Telemetry Service 的 Redfish 資源範例：

```json
{
    "@odata.type": "#TelemetryService.v1_1_2.TelemetryService",
    "Id": "TelemetryService",
    "Name": "Telemetry Service",
    "Status": {
        "State": "Enabled",
        "Health": "OK"
    },
    "MinCollectionInterval": "T00:00:10s",
    "SupportedCollectionFunctions": [],
    "MaxReports": <max_no_of_reports>,
    "MetricDefinitions": {
        "@odata.id": "/redfish/v1/TelemetryService/MetricDefinitions"
    },
    "MetricReportDefinitions": {
        "@odata.id": "/redfish/v1/TelemetryService/MetricReportDefinitions"
    },
    "MetricReports": {
        "@odata.id": "/redfish/v1/TelemetryService/MetricReports"
    },
    "Triggers": {
        "@odata.id": "/redfish/v1/TelemetryService/Triggers"
    },
    "LogService": {
        "@odata.id": "/redfish/v1/Managers/<manager_name>/LogServices/Journal"
    },
    "@odata.context": "/redfish/v1/$metadata#TelemetryService",
    "@odata.id": "/redfish/v1/TelemetryService"
}
```

指標報告定義範例：

```json
{
  "@odata.type": "#MetricReportDefinition.v1_3_0.MetricReportDefinition",
  "Id": "SampleMetric",
  "Name": "Sample Metric Report Definition",
  "MetricReportDefinitionType": "Periodic",
  "Schedule": {
    "RecurrenceInterval": "T00:00:10"
  },
  "ReportActions": ["LogToMetricReportsCollection"],
  "ReportUpdates": "Overwrite",
  "MetricReport": {
    "@odata.id": "/redfish/v1/TelemetryService/MetricReports/SampleMetric"
  },
  "Status": {
    "State": "Enabled"
  },
  "Metrics": [
    {
      "MetricId": "Test",
      "MetricProperties": [
        "/redfish/v1/Chassis/NC_Baseboard/Power#/PowerControl/0/PowerConsumedWatts"
      ]
    }
  ],
  "@odata.context": "/redfish/v1/$metadata#MetricReportDefinition.MetricReportDefinition",
  "@odata.id": "/redfish/v1/TelemetryService/MetricReportDefinitions/PlatformPowerUsage"
}
```

指標報告範例：

```json
{
  "@odata.type": "#MetricReport.v1_2_0.MetricReport",
  "Id": "SampleMetric",
  "Name": "Sample Metric Report",
  "ReportSequence": "0",
  "MetricReportDefinition": {
    "@odata.id": "/redfish/v1/TelemetryService/MetricReportDefinitions/SampleMetric"
  },
  "MetricValues": [
    {
      "MetricDefinition": {
        "@odata.id": "/redfish/v1/TelemetryService/MetricDefinitions/SampleMetricDefinition"
      },
      "MetricId": "Test",
      "MetricValue": "100",
      "Timestamp": "2016-11-08T12:25:00-05:00",
      "MetricProperty": "/redfish/v1/Chassis/Tray_1/Power#/0/PowerConsumedWatts"
    }
  ],
  "@odata.context": "/redfish/v1/$metadata#MetricReport.MetricReport",
  "@odata.id": "/redfish/v1/TelemetryService/MetricReports/AvgPlatformPowerUsage"
}
```

觸發器範例，它將觸發指標報告更新：

```json
{
  "@odata.type": "#Triggers.v1_1_1.Triggers",
  "Id": "SampleTrigger",
  "Name": "Sample Trigger",
  "MetricType": "Numeric",
  "Links": {
    "MetricReportDefinitions": [
      "/redfish/v1/TelemetryService/MetricReportDefinitions/SampleMetric"
    ]
  },
  "Status": {
    "State": "Enabled"
  },
  "TriggerActions": ["RedfishMetricReport"],
  "NumericThresholds": {
    "UpperCritical": {
      "Reading": 50,
      "Activation": "Increasing",
      "DwellTime": "PT0.001S"
    },
    "UpperWarning": {
      "Reading": 48.1,
      "Activation": "Increasing",
      "DwellTime": "PT0.004S"
    }
  },
  "MetricProperties": [
    "/redfish/v1/Chassis/Tray_1/Power#/0/PowerConsumedWatts"
  ],
  "@odata.context": "/redfish/v1/$metadata#Triggers.Triggers",
  "@odata.id": "/redfish/v1/TelemetryService/Triggers/PlatformPowerCapTriggers"
}
```

### 效能測試

效能測試是在配有 64 MB flash 與 512 MB RAM 的 AST2500 系統上進行的。Telemetry 服務的 flash 佔用量為 197.5 kB。執行期統計資料列於下表。讀取報告對應到單一 Metric Report。執行期資料僅針對 Telemetry 元件蒐集。所有報告建立時都將 `xyz.openbmc_project.Telemetry.Metric.OnChange` 屬性設定為啟用，以使工作量最大化。在 50 個報告與 50 個感測器的設定下，約為每秒 200 個新讀值，每秒產生 200 份讀取報告。下表顯示 CPU 使用率與記憶體使用率。VSZ 是對應到程序位址空間的記憶體量，包含由程序的執行檔與共用程式庫所支援的頁面、其堆積（heap）與堆疊（stack），以及它所對應的任何其他內容。

| Telemetry 服務狀態 | VSZ | %VSZ | %CPU |
| ------------------ | --- | ---- | ---- |
| 閒置（0 個報告，0 個感測器） | 5188 B | 1% | 0% |
| 1 個報告，1 個感測器 | 5188 B | 1% | 1% |
| 2 個報告，1 個感測器 | 5188 B | 1% | 1% |
| 2 個報告，2 個感測器（每報告 1 個感測器） | 5188 B | 1% | 1% |
| 1 個報告，10 個感測器 | 5188 B | 1% | 1% |
| 10 個報告，10 個感測器（各報告皆相同） | 5320 B | 1% | 1-2% |
| 2 個報告，20 個感測器（每報告 10 個） | 5188 B | 1% | 1% |
| 30 個報告，30 個感測器（每報告 10 個） | 5444 B | 1% | 5-9% |
| 50 個報告，50 個感測器（每報告 10 個） | 5572 B | 1% | 11-14% |

最後兩種設定每個讀取報告使用 10 個感測器，因此形成 3 種或 5 種不同的設定。每種此類設定都用來建立 10 份讀取報告，以達到所需的 30 或 50 份讀取報告數量。

在此架構中，每當 POST 一份 Redfish Metric Report Definition（建立新的 Metric Report）時，就會建立一份讀取報告。

## 其他考慮過的替代方案

[基於 collectd/librrd 的框架][5]曾作為替代設計被納入考量。雖然它看起來是一個多樣且可擴充的解決方案，但從我們的觀點來看，它有一些缺點：

- Collectd 在最小可用設定下的佔用量約為 2.6 MB，而 OpenBMC 的可用空間僅限於 64 MB。
- 在此設計中，librrd 用於將指標儲存於 BMC 的非揮發性儲存裝置上，當擷取大量指標並儲存到 OpenBMC 有限的儲存空間時，這可能會成為問題。此外，當指標擷取頻繁（例如每秒一次）時，也可能發生 flash 磨耗問題。
- Telemetry 服務與 Redfish Telemetry Service API 直接相容，這表示 Telemetry 的讀取報告可以直接對應到 Redfish Metric Reports。
- Telemetry 服務統一了 BMC 遙測透過 Redfish 暴露的方式，且可搭配多個前端使用，因此要新增對 IPMI 或任何其他 API 的遙測支援並無困難。

由於此設計假設了彈性與模組化，因此將 collectd 與 Telemetry 合作使用並沒有障礙。下圖展示了一種可能的設定。

```ascii
   +-----------------+      +-----------------+
   |  D-Bus sensors  |      |   Telemetry     |
   +--------^--------+      +--------^--------+
            |                        |
            |                        |
            |                        |
<--------^--v-----------D-Bus--------v-^---------->
         |                             |
         |                             |
         |                             |
 +-------v------------+     +----------v--------+
 |  collectd metrics  |     |                   |
 |  exposed as D-Bus  |     |     bmcweb        |
 |      sensors       |     |  (with Redfish    |
 +---------^----------+     |    Telemetry      |
           |                |     Service)      |
           |                |                   |
    +------+-------+        +-------------------+
    |              |
    |   collectd   |
    |              |
    +--------------+
```

在這裡，collectd 用作某組指標的來源。它將這些指標暴露為 D-Bus 感測器，bmcweb 與 Telemetry 服務都可以輕鬆地消耗這些感測器，而無需對其 D-Bus 介面做任何變更。在這種設定下，Telemetry 服務提供指標報告與觸發器管理。

另一種可能的設定是在不使用 Telemetry 服務的情況下使用 collectd，但在這種情況下，collectd 不提供與 Redfish 相容的指標報告與觸發器支援。此時，Redfish Telemetry Service 將不被支援，或者必須由 collectd 提供指標報告與觸發器的支援。

## 影響

此設計會影響 bmcweb 元件的架構，因為它將 Redfish Telemetry Service 的實作新增為既有 Redfish API 實作的一個元件。

## 測試

以下是所提議測試集合的高階說明。測試應在三個基本層級進行：

- 單元測試
- 功能測試
- 效能測試

### 單元測試

Telemetry 的程式碼應由單元測試覆蓋。首選的框架是 [GTest/GMock][7]。在程式碼變更要提交之前，應執行單元測試，以確保現有功能沒有被破壞。此外，當引入新程式碼時，應依測試驅動開發（test-driven development）原則，與其一起提交一組新的單元測試。單元測試也應仔細審查。

### 功能測試

功能測試將分為兩個步驟。

第一步用於測試 Telemetry 指標報告管理。測試情境應包含：透過 POST 適當的指標報告定義來建立指標報告、讀取指標報告（對正確的 URI 使用 GET），以及刪除指標報告。此測試所需的設定為 D-Bus 感測器（至少其中一些），以及實作了 Redfish Telemetry Service 的 bmcweb。這些測試應在真實硬體上執行。為便於指標測試，可提供虛擬（dummy）D-Bus 感測器以提供特別準備好的指標。此設定也應能測試彙總運算（MIN、MAX、SUM、AVG）。

第二步是測試觸發器與事件產生。這還需要同時實作 Event Service 與 Log Service。測試應涵蓋所有情境：將指標報告作為事件傳送、觸發指標報告更新，以及記錄事件。

### 效能測試

效能測試應使用完整的 OpenBMC 設定，並包含所有必要的功能集合。測試應建立大量指標報告（直到最大數量），以及所有可能的觸發器。量測應涵蓋週期性指標報告的抖動（jitter）、事件記錄或傳送的延遲、BMC 的 CPU 使用率，以及對其他服務的效能影響。

[1]:
  https://www.dmtf.org/sites/default/files/standards/documents/DSP2051_0.1.0a.zip
[2]:
  https://github.com/openbmc/docs/blob/master/architecture/sensor-architecture.md
[3]: https://www.kernel.org/doc/Documentation/hwmon/
[4]: https://www.freedesktop.org/wiki/Software/dbus/
[5]: https://gerrit.openbmc.org/c/openbmc/docs/+/22257
[6]: https://gerrit.openbmc.org/c/openbmc/docs/+/24749
[7]: https://github.com/google/googletest
[8]:
  https://github.com/openbmc/phosphor-dbus-interfaces/blob/master/yaml/xyz/openbmc_project/Telemetry/ReportManager.interface.yaml
[9]:
  https://github.com/openbmc/phosphor-dbus-interfaces/blob/master/yaml/xyz/openbmc_project/Telemetry/Report.interface.yaml
[10]:
  https://github.com/openbmc/phosphor-dbus-interfaces/blob/master/yaml/xyz/openbmc_project/Object/Delete.interface.yaml
