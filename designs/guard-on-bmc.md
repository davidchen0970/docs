# Guard on BMC

## 問題描述

在具有多個處理器單元及其他冗餘關鍵資源的系統上，可以透過隔離故障元件來避免系統停機。本文件討論 BMC 對此類隔離程序的支持設計。故障元件可以持續保持隔離狀態，直到更換為止。隔離元件所需的大部分動作會取決於架構，並由 host 負責處理。但 BMC 需要支持幾個步驟，例如通知使用者哪些元件處於隔離狀態、清除隔離、隔離疑似故障的元件，或在 host 因嚴重故障而當機時進行隔離。本文件將隔離的流程稱為 guard，意即保護系統免受故障元件的影響。

## 術語表

**Guard**：透過永久隔離故障單元來保護系統免受故障影響。

**Guard Records**：存放於永久儲存裝置中的檔案，記錄被永久隔離的元件清單。

**Manual guard**：手動將某個單元加入隔離清單的操作。此操作有助於在不將疑似故障元件實際從伺服器上拔除的情況下將其隔離。

## 背景與參考

伺服器中的 guard 用於管理故障元件的記錄，使其停止服務。故障但已被 guard 的元件清單可以根據元件的所屬權儲存在多個位置。如何儲存或管理該記錄將由對應的元件自行決定。一些範例包括：由 host 管理的主機板元件 guard、可由風扇控制應用程式管理的風扇 guard，或由電源管理應用程式管理的電源元件 guard。

當在可隔離的元素上遇到錯誤時，就會建立這些記錄。是否建立記錄的決策取決於錯誤類型、使用案例，以及是否有冗餘硬體資源可讓系統保持在可用狀態。為隔離元件而建立的記錄命名為 Guard Record。Guard record 會在修復動作後，或由服務人員手動刪除。大多數情況下，guard record 是由 host 建立的，因為 host 負責伺服器系統中硬體資源的初始化與使用。BMC 僅在有限的一組單元上建立 guard record，例如在早期開機期間、在導致 host 當機的嚴重故障之後，或是在可由 BMC 控制的元件（如電源供應器或風扇）上。BMC 會從各種 guard record 來源取得 guard record，以便呈現在外部介面上供客戶及服務人員檢視。

## 需求

### 主要需求

![Guard Usecases](https://user-images.githubusercontent.com/16666879/70852658-0edfda80-1eca-11ea-9d70-c81a690c78f2.jpeg)

- 當使用者要求時，根據硬體元件在正確的 guard record 儲存庫中建立記錄。
- 應提供選項讓使用者為 DIMM 及處理器核心建立 guard record，以手動方式使其停止服務。
- 應提供選項讓使用者刪除 guard record。
- 應提供選項讓使用者列出 guard record。
- 應提供選項讓使用者刪除所有 guard record。
- 應提供產業標準介面來對 guard record 執行這些操作。

### 假設

- 在由 host 管理的單元上的 guard record，只會在後續的開機中套用。
- 擁有該硬體資源的子系統會套用 guard record 並隔離單元。
- 在更換故障元件後清除記錄，應由擁有該 guard record 的元件負責。
- guard record 中的 key 與 BMC 中用於元件的 key 之間應存在對應關係。

## 建議設計

此處提議的 guard 是一個彙整器，用於彙整來自各種來源的 guard record，並提供一個通用介面來建立、刪除及列出這些記錄。

### D-Bus 介面

在 BMC 上，會有一個 guard manager 物件，以及每個項目的對應物件。

#### Guard Manager

Guard manager 用於提供管理 guard record 及管理 guard record 相關資訊的外部介面。Guard manager 的方法與屬性如下。

- Create Guard：為 DIMM 或處理器核心建立 guard record。輸入：待 guard 的 DIMM 或處理器核心的 Inventory 路徑。

- Delete Guard：刪除現有的 guard record。刪除 guard record 的 D-Bus 項目時，應同時刪除底層的記錄。

- Clear all guard：清除系統中所有的 guard record。

- List Guard：列出所有被 guard 的元件。

**Note：** 在某些平台上，系統或硬體可能處於無法執行「永久」或「暫時」guard 操作的狀態。

#### Guard Entry

每個 guard entry 的屬性都會是此物件的一部分。屬性：

- ID：記錄的 ID，為 entry 物件路徑的一部分。
- Associations（關聯）：
  - Guarded hardware inventory path（被 guard 的硬體 inventory 路徑）：
    - Forward Name 必須為 "isolated_hw"。
    - Reverse Name 必須為 "isolated_hw_entry"。
  - BMC Error Log：
    - Forward Name 必須為 "isolated_hw_errorlog"。
    - Reverse Name 必須為 "isolated_hw_entry"。

      Error Log 關聯可以是選擇性的，因為使用者可能嘗試 guard 硬體，這並非錯誤情況。

- Severity：Guard 的類型
  - Manual - 手動 Guard
  - Critical - 基於嚴重錯誤進行的 Guard
  - Warning - 基於非嚴重錯誤進行的 Guard，但最終可能導致嚴重故障。
- Resolved：被 guard 硬體的狀態
  - 用於指出被 guard 的硬體是否已修復或更換。

    **Note：** 將其設為 "true" 不會刪除此項目，因為在某些系統平台上，被 guard 的硬體記錄可能不會被刪除，而會保留供進一步分析使用。

底層的 guard 功能由管理硬體單元的應用程式實作。實作通用 guard entry 介面的應用程式，應在 entry 與原始 guard record 儲存庫中的底層 guard record 之間建立對應。

## Redfish 介面

### Manual guard

建立 manual guard record：將 inventory 中現有單元的 "Enabled" 屬性設為 "false"，以手動方式 guard 該單元。

#### redfish » v1 » Systems » system » Processors » CPU1

```json
{
  "@odata.type": "#Processor.v1_7_0.Processor",
  "Id":view details "CPU1",
  "Name": "Processor",
  "Socket": "CPU 1",
  "ProcessorType": "CPU",
  "ProcessorId":
   {
       "VendorId": "XXXX",
       "IdentificationRegisters": "XXXX",
   } ,
   "MaxSpeedMHz": 3700,
   "TotalCores": 8,
   "TotalThreads": 16,
   "Status":
   {
        "State": "UnavailableOffline",
        "Health": "Critical"
        "Enabled": "False" <--- guarded a CPU1
   } ,
   "@odata.id":view details "/redfish/v1/Systems/system/Processors/CPU1"
}
```

### 列出 guard record

#### redfish >> v1 >> Systems >> system >> LogServices >> IsolatedHardware

#### >> Entries

```json
{
  "@odata.id": "/redfish/v1/Systems/system/LogServices/IsolatedHardware/Entries",
  "@odata.type": "#LogEntryCollection.LogEntryCollection",
  "Description": "Collection of Isolated Hardware Components",
  "Members": [
    {
      "@odata.id":
"/redfish/v1/Systems/system/LogServices/IsolatedHardware/Entries/1",
      "@odata.type": "#LogEntry.v1_7_0.LogEntry",
      "Created": "2020-10-15T10:30:08+00:00",
      "EntryType": "Event",
      "Id": "1",
      "Resolved": "false",
      "Name": "Processor 1",
      "links":  {
                 "OriginOfCondition": {
                       "@odata.id": "/redfish/v1/Systems/system/Processors/cpu1"
                 }
               },
      "Severity": "Critical",
      "SensorType" : "Processor",
      "AdditionalDataURI": “/redfish/v1/Systems/system/LogServices/EventLog/attachment/111"
      “AdditionalDataSizeBytes": "1024"
    }
  ],
  "Members@odata.count": 1,
  "Name": "Isolated Hardware Entries"
  }
```

## 考慮過的替代方案

guard record 可以為任何具冗餘性且可隔離的元件建立，以防對硬體或資料造成損害。一旦建立記錄，就需要一個隔離程序將單元從服務中隔離。某些由 host 控制的單元只能在重新開機後隔離，而由 BMC 控制的單元則可以立即從服務中移除。替代方案如下：

- 由 host 建立、套用並呈現 guard record：在這種情況下，BMC 沒有任何控制權，且 host 需要提供商者介面，因此不同類型的 host 之間可能不會有一致的通用介面。針對由 BMC 及由 host 建立的 guard record，將需要不同的使用者介面。
- 由 BMC 管理 guard 的外部介面：將有一個統一的位置來呈現或管理由多個 host 或 BMC 本身建立的 guard record。有些 guard record 是在 host 發生嚴重故障後建立的；作為系統控制實體，由 BMC 來處理這類情況並建立記錄會更為容易。

## 影響

- Guard record 將以日誌擴充的形式呈現。
- Redfish 實作會受到影響，需要提供使用者進行 guard record 管理所需操作的能力。由於 Redfish 標準中尚未提供列出被隔離單元的方法，因此已規劃針對該方法提出標準化請求。

## 測試

所需進行的測試包括建立、刪除及列出 guard record，這些應予以自動化；而針對每種單元隔離類型的進一步測試則取決於實作。
