# NVMe-MI over SMBus

作者：Tony Lee <tony.lee@quantatw.com>

建立日期：3-8-2019

## 問題說明

目前 OpenBMC 不支援 NVMe 硬碟資訊。NVMe-MI 規格定義了一條指令，可以直接透過
SMBus 讀取 NVMe 硬碟資訊。NVMe 硬碟可以提供它的資訊或狀態，例如廠商
ID（vendor ID）、溫度等。本提案的目標是讓使用者能夠監控 NVMe 硬碟，以便採取
適當的處置。

## 背景與參考資料

NVMe-MI 規格定義了一條名為 `NVM Express Basic Management Command` 的指令，
可以直接透過 SMBus 讀取 NVMe 硬碟資訊 [1]。這條指令使用 SMBus 規格所定義的
SMBus Block Read 通訊協定 [2]。

由於我們的目的是擷取 NVMe 硬碟資訊，因此使用 NVMe-MI 規格中所說明的 NVM
Express Basic Management Command 與 NVMe 硬碟通訊。依據不同平台，溫度感測器、
在位狀態、LED 與電源時序都會各自客製化。

[1]:
  https://nvmexpress.org/wp-content/uploads/NVM_Express_Management_Interface_1_0a_2017.04.08_-_gold.pdf
  "NVM Express Management Interface Revision 1.0a April 8, 2017 in Appendix A."
[2]:
  http://smbus.org/specs/SMBus_3_0_20141220.pdf
  "System Management Bus (SMBus) Specification Version 3.0 20 Dec 2014"

## 需求

實作應具備以下能力：

- 提供一個 daemon 來監控 NVMe 硬碟。要監控的參數包括 Status Flags、SMART
  Warnings、溫度（Temperature）、Percentage Drive Life Used、Vendor ID 以及
  Serial Number。
- 提供一個 D-bus 介面，讓其他服務能夠存取資料。
- 具備透過硬體通道 I2C 與 NVMe 硬碟通訊的能力。
- 能夠依據 SmartWarnings 對每個硬碟開啟／關閉故障 LED，前提是故障 LED 的
  物件路徑已在設定檔中定義。

## 建議設計

建立一個 D-bus 服務 "xyz.openbmc_project.nvme.manager"，為每個 NVMe 感測器
提供物件路徑："/xyz/openbmc_project/sensors/temperature/nvme0"、
"/xyz/openbmc_project/sensors/temperature/nvme1" 等。有一份 JSON 設定檔，
記錄每個硬碟的硬碟索引（drive index）、匯流排 ID（bus ID）以及故障 LED 的
物件路徑。例如，

```json
{
  "NvmeDriveIndex": 0,
  "NVMeDriveBusID": 16,
  "NVMeDriveFaultLEDGroupPath": "/xyz/openbmc_project/led/groups/led_u2_0_fault",
  "NVMeDrivePresentPin": 148,
  "NVMeDrivePwrGoodPin": 161
},
{
  "NvmeDriveIndex": 1,
  "NVMeDriveBusID": 17,
  "NVMeDriveFaultLEDGroupPath": "/xyz/openbmc_project/led/groups/led_u2_0_fault",
  "NVMeDrivePresentPin": 149,
  "NVMeDrivePwrGoodPin": 162
}
```

結構如下：

在名為 "xyz.openbmc_project.nvme.manager" 的 D-bus 之下：

```text
    /xyz/openbmc_project
    └─/xyz/openbmc_project/sensors
      └─/xyz/openbmc_project/sensors/temperature/nvme0
```

/xyz/openbmc_project/sensors/temperature/nvme0 實作了下列介面：

- xyz.openbmc_project.Sensor.Value
- xyz.openbmc_project.Sensor.Threshold.Warning
- xyz.openbmc_project.Sensor.Threshold.Critical

在名為 "xyz.openbmc_project.Inventory.Manager" 的 D-bus 之下：

```text
/xyz/openbmc_project
    └─/xyz/openbmc_project/inventory
      └─/xyz/openbmc_project/inventory/system
        └─/xyz/openbmc_project/inventory/system/chassis
          └─/xyz/openbmc_project/inventory/system/chassis/motherboard
           └─/xyz/openbmc_project/inventory/system/chassis/motherboard/nvme0
```

/xyz/openbmc_project/inventory/system/chassis/motherboard/nvme0 實作了
下列介面：

- xyz.openbmc_project.Inventory.Item
- xyz.openbmc_project.Inventory.Decorator.Asset
- xyz.openbmc_project.Nvme.Status

介面 `xyz.openbmc_project.Sensor.Value`，供 hwmon 監控溫度使用，具有以下
屬性：

| Property | Type   | Description          |
| -------- | ------ | -------------------- |
| MaxValue | int64  | 感測器最大值 |
| MinValue | int64  | 感測器最小值 |
| Scale    | int64  | 感測器數值比例   |
| Unit     | string | 感測器單位          |
| Value    | int64  | 感測器數值         |

介面 `xyz.openbmc_project.Nvme.Status` 具有以下屬性：

| Property          | Type   | Description                                  |
| ----------------- | ------ | -------------------------------------------- |
| SmartWarnings     | string | 表示硬碟狀態的 smart warnings       |
| StatusFlags       | string | 表示硬碟的狀態           |
| DriveLifeUsed     | string | 由廠商估算的使用壽命百分比 |
| TemperatureFault  | bool   | 是否發生與溫度相關的警告類型   |
| BackupdrivesFault | bool   | 是否發生與備份硬碟相關的警告類型 |
| CapacityFault     | bool   | 是否發生與容量相關的警告類型      |
| DegradesFault     | bool   | 是否發生與效能衰退相關的警告類型      |
| MediaFault        | bool   | 是否發生與媒體相關的警告類型         |

介面 `xyz.openbmc_project.Inventory.Item` 具有以下屬性：

| Property   | Type   | Description                         |
| ---------- | ------ | ----------------------------------- |
| PrettyName | string | 該項目的人類可讀名稱 |
| Present    | bool   | 該項目是否在位  |

介面 `xyz.openbmc_project.Inventory.Decorator.Asset` 具有以下
屬性：

| Property     | Type   | Description                                       |
| ------------ | ------ | ------------------------------------------------- |
| PartNumber   | string | 該項目的料號，通常為庫存編號 |
| SerialNumber | string | 該項目的序號                            |
| Manufacturer | string | 該項目的製造商                             |
| BuildDate    | bool   | 該項目的製造日期，格式為 YYYYMMDD   |
| Model        | bool   | 該項目的型號                             |

### xyz.openbmc_project.nvme.manager.service

這個服務有以下幾個步驟：

1. 它會註冊一個名為 `xyz.openbmc_project.nvme.manager` 的 D-bus，
   說明如上文所述。
2. 從上述 json 檔案取得硬碟索引、匯流排 ID、GPIO 在位腳位（present pin）、
   電源正常腳位（power good pin）以及故障 LED 物件路徑。
3. 每個循環會執行下列步驟：
   1. 檢查目標硬碟的在位腳位是否為 true，若為 true 表示硬碟存在，並進入下一步。
      若不是，表示硬碟不存在，並依硬碟索引從 D-bus 移除其物件路徑。
   2. 檢查目標硬碟的電源正常腳位是否為 true，若為 true 表示硬碟已準備就緒，
      接著依硬碟索引建立物件路徑並進入下一步。若不是，表示硬碟電源異常，
      開啟故障 LED 並記錄到 journal。
   3. 依目標硬碟的匯流排 ID，透過 SMBus Block Read 通訊協定送出一條 NVMe-MI
      指令以取得資料。從 NVMe 硬碟取得的資料為 "Status Flags"、"SMART
      Warnings"、"Temperature"、"Percentage Drive Life Used"、"Vendor ID"
      以及 "Serial Number"。
   4. 這些資料會被設定到 D-bus 中的屬性。

這個服務會自動執行，並每秒查找一次 NVMe 硬碟。

### 故障 LED

當從指令取得的數值對應到其中一種警告類型時，會觸發對應裝置的故障 LED 並發出
事件。

### 新增與 NVMe 相關的 SEL

會為 NVMe 錯誤產生 `TemperatureFault`、`BackupdrivesFault`、`CapacityFault`、
`DegradesFault` 與 `MediaFault` 等事件。

- Temperature Fault log：當 `TemperatureFault` 屬性被設定為 true 時
- Backupdrives Fault log：當 `BackupdrivesFault` 屬性被設定為 true 時
- Capacity Fault log：當 `CapacityFault` 屬性被設定為 true 時
- Degrades Fault log：當 `DegradesFault` 屬性被設定為 true 時
- Media Fault log：當 `MediaFault` 屬性被設定為 true 時

## 曾考慮的替代方案

NVMe-MI 規格定義了多條可以透過 MCTP 通訊協定與 NVMe 硬碟通訊的指令。NVMe-MI
over MCTP 具有以下幾項關鍵能力：

- 發現在位的硬碟並了解每個硬碟的能力。
- 儲存關於主機環境的資料，讓 Management Controller 之後可以查詢這些資料。
- 提供 VPD 的標準格式，以及讀取／寫入 VPD 內容的定義機制。
- 清點（inventorying）、設定與監控。

就監控 NVMe 硬碟而言，直接使用 SMBus 上的 NVM Express Basic Management Command
遠比使用 MCTP 通訊協定上的 NVMe-MI 簡單得多。

## 影響

這個應用程式透過 SMBus 監控 NVMe 硬碟並將數值設定到 D-bus。對系統的影響應該
很小。

## 測試

這個實作使用 SMBus 上的 NVMe-MI-Basic 指令，然後將回應資料設定到 D-bus。測試
會送出 SMBus 指令給硬碟以取得資訊，並與 D-bus 中的屬性比較，以確認兩者相同。
測試可以在不同製造商的不同 NVMe 硬碟上執行。例如：Intel P4500/P4600 以及
Micron 9200 Max/Pro。

單元測試將依功能進行測試：

- 在取得 NVMe 資訊的函式中，測試回應資料的長度是否與設計相同。
- 測試將數值設定到 D-bus 的函式是否與設計相同。
- 測試依據不同 Smartwarnings 數值開啟／關閉對應 LED 的函式。
