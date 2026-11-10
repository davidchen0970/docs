# 平台韌體韌性（PFR）的 Redfish 資源補充

作者：AppaRao Puli

其他貢獻者：無

建立日期：2019-09-12

## 問題描述

平台是一組用於啟動與操作系統所需的基本硬體與韌體元件。NIST SP 800-193 中的平台韌體韌性（Platform Firmware Resiliency，PFR）提供了技術指引與建議，以支援平台韌體與資料抵禦潛在破壞性攻擊的韌性能力。

目前 DMTF 所定義的 Redfish schema 並未涵蓋用以表示 PFR 已佈署（provisioned）與已鎖定（locked）狀態的屬性或資源。

本文件說明在 ComputerSystem 資源下新增的 OEM 屬性，用以表示 PFR 的佈署狀態，例如平台韌體是否已佈署，以及佈署是否已鎖定。本文件也涵蓋用於記錄與 PFR 相關事件的 Redfish OpenBMC 訊息登錄（message registry）中繼資料。

## 背景與參考資料

NIST SP 800-93 中的平台韌體韌性技術為實作者提供了共通指引，包括原廠設備製造商（OEM）與元件／裝置供應商，以便在平台中建立更強的安全性機制。伺服器平台由多個裝置組成，這些裝置必須透過保護、偵測與復原平台資產來提供韌性。執行於伺服器平台上的管理控制器可用來表示韌性狀態，以及與其相關的事件記錄。

- [NIST.SP.180-193](https://nvlpubs.nist.gov/nistpubs/SpecialPublications/NIST.SP.800-193.pdf)
- [Redfish schema 補充](https://www.dmtf.org/sites/default/files/standards/documents/DSP0268_2019.1a.pdf)
- [Redfish Logging in bmcweb](https://github.com/openbmc/docs/blob/master/architecture/redfish-logging-in-bmcweb.md)

## 需求

高階需求：

- BMC 應提供一種方式，透過 Redfish 表示平台韌體韌性的佈署狀態。

- 針對平台韌體韌性，應將事件記錄寫入 Redfish。

## 建議設計

不同的 OEM 各自有其實作平台韌體韌性的方式，皆是依據 NIST SP 800-193 所提供的指引。在此保護範圍下的部分元件包括（但不限於）ES/SIO、BMC/ME Flash、主機處理器、信任平台模組（TPM）、電源供應器（PSU）、記憶體等。舉例來說，Intel 使用「Intel PFR」設計來為平台元件提供韌性。

注意：本文件並不涵蓋各 OEM 如何實作平台韌體韌性的設計層面。它僅涵蓋與實作方式無關（implementation agnostic）的通用 Redfish ComputerSystem OEM 屬性，以及 Redfish 訊息登錄中繼資料項目。

OpenBMC 正逐步採用 Redfish 作為其帶外管理（out of band management）的標準。bmcweb 實作了所有 Redfish schema，以表示不同的屬性與資源。ComputerSystem schema 並未涵蓋與平台韌體韌性相關的屬性，但它提供了 OEM 物件，供製造商／供應商專屬延伸使用。

以下定義了用以表示平台韌體韌性佈署狀態的屬性。

- ProvisioningStatus：此屬性的值表示平台韌體的佈署狀態。它是一個 Enum 型別，具有以下值。
  1. NotProvisioned：表示平台韌體尚未佈署。這是一種未受保護的狀態。

  2. ProvisionedButNotLocked：表示平台韌體已佈署但尚未鎖定。因此在這個狀態下允許重新佈署。

  3. ProvisionedAndLocked：表示平台韌體已佈署且已鎖定。因此在這個狀態下不允許重新佈署。

啟用 PFR 的平台在處於「NotProvisioned」或「ProvisionedButNotLocked」狀態時，可以多次佈署或重新佈署平台韌性。但一旦平台轉換為「ProvisionedAndLocked」狀態後，即無法再重新佈署。

新的 OemComputerSystem schema 可於
[in review](https://gerrit.openbmc.org/#/c/openbmc/bmcweb/+/24253/) 中找到。

OpenBMC 中的 PFR 必須支援記錄韌性錯誤的偵測與修正。以下是 Redfish 訊息登錄中用於 Redfish 事件記錄能力的兩個中繼資料項目。關於如何記錄 Redfish 事件的更多細節，請參閱文件
[Redfish logging in bmcweb](https://github.com/openbmc/docs/blob/master/redfish-logging-in-bmcweb.md)。

以下幾類事件應作為 PFR 或佈署的一部分予以記錄。這些 Redfish 事件是為 BIOS、BMC、CPLD 與 ME 等基本韌體元件所建立，並且必須在 PFR 下支援新的韌體模組時一併延伸。

1. Platform Firmware Resiliency Errors（平台韌體韌性錯誤）：此用於記錄平台韌體韌性錯誤。

   MessageID：
   - BIOSFirmwareResiliencyError
   - BMCFirmwareResiliencyError
   - CPLDFirmwareResiliencyError
   - MEFirmwareResiliencyError
   - FirmwareResiliencyError Severity：Critical

   以下是針對特定元件記錄韌體韌性錯誤事件的幾種情境。
   - 開機失敗（Boot failure）
   - 更新失敗（Update Failure）

2. Platform Firmware Panic reason（平台韌體 Panic 原因）：此用於記錄平台韌體 panic 的原因。

   MessageID：
   - BIOSFirmwarePanicReason
   - BMCFirmwarePanicReason
   - CPLDFirmwarePanicReason
   - MEFirmwarePanicReason Severity：Warning

   以下是可以記錄這些事件的幾種情境。
   - 開機時間看門狗逾時（Boot time watchdog expired）
   - 韌體認證失敗（Firmware authentication failure）

3. Platform Firmware Recovery reasons（平台韌體復原原因）：此用於記錄最近一次平台韌體元件復原的原因。

   MessageID：
   - BIOSFirmwareRecoveryReason
   - BMCFirmwareRecoveryReason
   - CPLDFirmwareRecoveryReason
   - MEFirmwareRecoveryReason Severity：Warning

   以下是可以記錄這些事件的幾種情境。
   - 載入失敗（Launch failures）
   - 更新失敗（Update failures）
   - 認證失敗（Authentication failures）

此處以 BMC 元件相關的事件記錄作為範例。

```text
    MessageEntry{"BMCFirmwareResiliencyError",
                 {
                     "Indicates BMC firmware encountered resilience error",
                     "BMC firmware resiliency error. Error reason: %1.",
                     "Critical",
                     1,
                     {
                         "string",
                     },
                     "None.",
                 }},
    MessageEntry{"BMCFirmwarePanicReason",
                 {
                     "Indicates the reason for last BMC firmware panic.",
                     "BMC firmware panic occurred due to %1.",
                     "Warning",
                     1,
                     {
                         "string",
                     },
                     "None.",
                 }},
    MessageEntry{"BMCFirmwareRecoveryReason",
                 {
                     "Indicates the reason for last BMC firmware recovery.",
                     "BMC firmware recovery occurred due to %1.",
                     "Warning",
                     1,
                     {
                         "string",
                     },
                     "None.",
                 }},
```

## 已考量的替代方案

無

## 影響

無

## 測試

佈署狀態：

- 使用者可以 OEM 專屬的方式佈署 PFR，並使用以下的 URI 與 Method 進行測試。Intel 使用「Intel PFR」設計（透過 BIOS）來佈署並鎖定 PFR 的佈署狀態。

```text
URI: /redfish/v1/Systems/system
METHOD: GET
RESPONSE:

{
  "@odata.context": "/redfish/v1/$metadata#ComputerSystem.ComputerSystem",
  "@odata.id": "/redfish/v1/Systems/system",
  "@odata.type": "#ComputerSystem.v1_6_0.ComputerSystem",
  ...................
  ...................
  "Description": "Computer System",
  "Oem": {
    "OpenBmc": {
      "FirmwareProvisioning": {
        "ProvisioningStatus": "NotProvisioned"
      }
    }
  },
  ...................
  ...................
}
```

事件記錄：

- 使用者可以誘發安全性攻擊，並使用以下的 URI 來驗證 panic 事件記錄以及復原機制。

   攻擊韌體元件並驗證 PFR 的幾個範例：
   1. 破壞 BMC/BIOS 等韌體，並檢查 Panic 事件與復原動作的事件。

   2. 使用已知方法誘發硬體看門狗觸發，並進行檢查等。

   3. 直接模擬硬體來破壞安全性金鑰，並進行驗證。

```text
URI: /redfish/v1/Systems/system/LogServices/EventLog/Entries
METHOD: GET
```
