# Redfish Client 設計

作者：Xinyuan Wang: <wang.x.6f@gmail.com>

其他貢獻者：Patrick Williams: <patrick@stwcx.xyz>

建立日期：2024 年 11 月 04 日

## 問題描述

某些衛星管理控制器（Satellite Management Controllers，SMC），例如 Nvidia 的 HMC，採用 Redfish 作為其主要通訊協定。由於目前 OpenBMC 僅提供有限的能力來作為 Redfish 用戶端進行溝通，例如 bmcweb 的 Redfish 聚合（aggregation），因此 BMC 應用程式無法存取由這類 SMC 所管理的資源（關於 bmcweb Redfish 聚合為何不足以滿足需求的詳細理由，請參閱「已評估的替代方案」章節）。在某些情況下，讓 BMC 應用程式能夠存取這類 SMC 的資源是必要條件。舉例來說，散熱校準（thermal calibration）需要來自 SMC 的感測器讀值。此外，某些 BMC 使用者可能會對 BMC 應如何對外揭露資源訂定特定規則（例如感測器命名）。因此，在將 SMC 資源對外揭露之前，能夠對這些資源套用相同的規則就顯得至關重要。

## 背景與參考資料

- 什麼是衛星管理控制器（Satellite Management Controller）？
  [Satellite Management Controller (SMC) Specification](https://www.opencompute.org/documents/smc-specification-1-0-final-pdf-1)
- 一個採用 Redfish 作為主要通訊協定的 SMC 範例：
  [Redfish APIs Support — NVIDIA DGX H100/H200](https://docs.nvidia.com/dgx/dgxh100-user-guide/redfish-api-supp.html)
- 先前已有一份
  [設計提案](https://gerrit.openbmc.org/c/openbmc/docs/+/58954)，
  提議建立一個 daemon 透過 Redfish 讀取感測器資料，其內容如 Redfish 感測器設計提案（Redfish Sensor Design Proposal）所述。該提案與目前的提案類似，但它僅支援感測器監控功能。更詳細的比較請參閱「已評估的替代方案」章節。
- 雖然可以利用
  [bmcweb 支援的 Redfish 聚合](https://github.com/openbmc/bmcweb/blob/master/AGGREGATION.md)
  從 BMC 外部存取 SMC 資源，但這種做法並未提供給在 BMC 內部執行的應用程式直接存取這些資源的能力。更詳細的比較以及對此做法限制的分析，請參閱「已評估的替代方案」章節。
- 下游實作參考：
  [Meta OpenBMC redfish-client](https://github.com/facebook/openbmc/tree/helium/common/lfopenbmc/recipes-downstream/redfish-client)

## 需求

BMC 應用程式能夠（直接或間接）存取以下由採用 Redfish 作為主要通訊協定的 SMC 所管理的資源：

- 感測器
  [Sensor](https://www.dmtf.org/sites/default/files/standards/documents/DSP0268_2024.4.html#sensor-1101)
  集合
- 記錄項目
  [LogEntry](https://www.dmtf.org/sites/default/files/standards/documents/DSP0268_2024.4.html#logentry-1170)
  集合
- [軟體庫存 Software Inventory](https://www.dmtf.org/sites/default/files/standards/documents/DSP0268_2024.4.html#softwareinventory-1102)

每一項資源都應該建立其對應的 D-Bus 介面，使得這些資源與一般資源無法區分，且部分轉換規則應可設定。

在向 SMC 發送 Redfish 請求時，需要以下功能：

- HTTP keep-alive，以避免讓 SMC 負載過重
- 能夠併發地發送與接收多個請求
- 與最新發布的 DMTF Redfish schema 相差最多一年的版本相容

除非目標 SMC 有要求，否則以下功能為選擇性：

- 驗證（Authentication）
- HTTPS

以下功能將不會被支援：

- OEM schema
- 偏離規範（out of spec）的 SMC

## 提案設計

本設計提案在 BMC 上執行一個 Redfish 用戶端 daemon，以執行必要任務並滿足上述需求。如前所述，它負責以下事項：

1. 設定
2. 透過 Redfish 從目標 SMC 收集資源
3. 將收集到的資源從 Redfish 表示轉換為 D-Bus 表示
4. 託管（hosting）資源

### 設定

在設定方面，每個平台都需要提供自己的設定檔，格式如下：

```json
{
  "SatelliteControllerName": "HMC",
  "Compatible": ["com.meta.Hardware.BMC.Model.Catalina"],
  "SensorConfig": {
    "Mappers": [
      {
        "FromId": "HGX_Chassis_0_TotalGPU_Power_0",
        "ToId": "HGX_TOTALGPU_PWR_W"
      },
      {
        "FromId": "ProcessorModule_0_CPU_0_CoreUtil_0",
        "ToId": "HGX_CPU0_CORE0_UTIL_PCT"
      }
    ],
    "IntervalMilliseconds": 2000,
    "MaxRetries": 3,
    "RetryIntervalMilliseconds": 200,
    "MaxExpandLevels": 3
  },
  "LogServiceConfig": {
    "Targets": [
      {
        "SystemId": "HGX_Baseboard_0",
        "LogServiceId": "EventLog"
      },
      {
        "ManagerId": "HGX_BMC_0",
        "LogServiceId": "Journal"
      }
    ],
    "IntervalMilliseconds": 2000,
    "MaxRetries": 2,
    "RetryIntervalMilliseconds": 300
  },
  "UpdateServiceConfig": {
    "FirmwareMappers": [
      {
        "FromId": "HGX_FW_CPU_0",
        "ToId": "HGX_cpu_0"
      }
    ],
    "SoftwareMappers": [
      {
        "FromId": "HGX_Driver_GPU_0",
        "ToId": "HGX_gpu_driver_0"
      }
    ],
    "IntervalMilliseconds": 60000,
    "MaxRetries": 3,
    "RetryIntervalMilliseconds": 2000
  }
}
```

為了決定要使用哪些設定檔，daemon 會先收集由 entity-manager 管理、且實作了 `xyz.openbmc_project.Inventory.Decorator.Compatible` D-Bus 介面的所有實體名稱。如果某個設定檔的「Compatible」欄位中所指定的名稱是該清單的子集合，則該設定檔會被視為已啟用，並在稍後據以建立任務。因此，當有多個設定檔符合條件時，多個設定檔可以同時被啟用。

舉例來說，如果系統具有以下實體，上述設定檔會被視為處於啟用狀態，因為它符合 `["com.meta.Hardware.BMC.Model.Catalina"]`：

```text
NAME                                               TYPE      SIGNATURE RESULT/VALUE                             FLAGS
org.freedesktop.DBus.Introspectable                interface -         -                                        -
.Introspect                                        method    -         s                                        -
org.freedesktop.DBus.Peer                          interface -         -                                        -
.GetMachineId                                      method    -         s                                        -
.Ping                                              method    -         -                                        -
org.freedesktop.DBus.Properties                    interface -         -                                        -
.Get                                               method    ss        v                                        -
.GetAll                                            method    s         a{sv}                                    -
.Set                                               method    ssv       -                                        -
.PropertiesChanged                                 signal    sa{sv}as  -                                        -
xyz.openbmc_project.AddObject                      interface -         -                                        -
.AddObject                                         method    a{sv}     -                                        -
xyz.openbmc_project.Inventory.Decorator.Compatible interface -         -                                        -
.Names                                             property  as        1 "com.meta.Hardware.BMC.Catalina"       emits-change
xyz.openbmc_project.Inventory.Decorator.Revision   interface -         -                                        -
.Version                                           property  s         "MP"                                     emits-change
xyz.openbmc_project.Inventory.Item.Board           interface -         -                                        -
.Name                                              property  s         "Catalina SCM"                           emits-change
.Probe                                             property  s         "xyz.openbmc_project.FruDevice({\'BOARD… emits-change
.Type                                              property  s         "Board"                                  emits-change
```

### 透過 Redfish 收集資源

對於每個已啟用的設定檔，daemon 需要先找出目標 SMC，然後根據設定建立一個或多個任務，從目標 SMC 收集資源。

為了找出目標 SMC，daemon 會尋找
[SatelliteController](https://github.com/openbmc/entity-manager/blob/e25a3890392093fec8f49b84c5bbce99055c6768/schemas/satellite_controller.json)，
其名稱必須與設定檔中 `SatelliteControllerName` 欄位所指定的名稱完全相同。如果找到，`HostName` 與 `Port` 將用於稍後建立 HTTP 連線。如果找不到，該設定將變為未啟用狀態。daemon 也會定期檢查目標 SMC 是否存在，以動態地啟用／停用設定。

舉例來說，如果系統具有以下實體，上述範例設定檔會被視為已啟用，並且會使用主機名稱 `0.1.1.1` 加上埠號 `80` 來建立 HTTP 連線，因為它符合 `HMC`：

```text
NAME                                                  TYPE      SIGNATURE RESULT/VALUE          FLAGS
org.freedesktop.DBus.Introspectable                   interface -         -                     -
.Introspect                                           method    -         s                     -
org.freedesktop.DBus.Peer                             interface -         -                     -
.GetMachineId                                         method    -         s                     -
.Ping                                                 method    -         -                     -
org.freedesktop.DBus.Properties                       interface -         -                     -
.Get                                                  method    ss        v                     -
.GetAll                                               method    s         a{sv}                 -
.Set                                                  method    ssv       -                     -
.PropertiesChanged                                    signal    sa{sv}as  -                     -
xyz.openbmc_project.Configuration.SatelliteController interface -         -                     -
.AuthType                                             property  s         "None"                emits-change
.Hostname                                             property  s         "0.1.1.1"             emits-change
.Name                                                 property  s         "HMC"                 emits-change
.Port                                                 property  t         80                    emits-change
.Type                                                 property  s         "SatelliteController" emits-change
```

在目標 SMC 確定之後，daemon 會根據設定建立一個或多個任務。在本設計提案提出時，可以建立的任務有三種類型：

1. 感測器收集任務：此任務由 `SensorConfig` 設定，會定期向目標 SMC 發送 Redfish 請求，以取得所有感兴趣的感測器。請求的頻率由 `IntervalMilliseconds` 指定。最大重試次數由 `MaxRetries` 指定。重試之間的間隔由 `RetryIntervalMilliseconds` 指定。根據設定中指定的 `MaxExpandLevels`，在每個週期中，該任務會發送單一請求以取得所有機殼（chassis）的所有感測器，即
   `/​redfish/​v1/​Chassis?$expand=.($levels=3)`，
   或者它會發送多個請求，個別地取得每個機殼的感測器，即
   `/​redfish/​v1/​Chassis/​{ChassisId}/​Sensors?$expand=.($levels=1)`。

2. 記錄收集任務：此任務由 `LogServiceConfig` 設定，會定期向目標 SMC 發送 Redfish 請求，以取得所有感兴趣的記錄項目。請求的頻率由 `IntervalMilliseconds` 指定。最大重試次數由 `MaxRetries` 指定。重試之間的間隔由 `RetryIntervalMilliseconds` 指定。設定中指定的 `Targets` 將用於建構請求的 URI。舉例來說，以下的目標：

   ```json
   {
     "SystemId": "HGX_Baseboard_0",
     "LogServiceId": "EventLog"
   }
   ```

   將導致向 `/redfish/v1/Systems/HGX_Baseboard_0/LogServices/EventLog/Entries` 發送請求。

3. 軟體／韌體庫存收集任務：此任務由 `UpdateServiceConfig` 設定，會定期向目標 SMC 發送 Redfish 請求，以取得所有感兴趣的軟體／韌體庫存項目。請求的頻率由 `IntervalMilliseconds` 指定。最大重試次數由 `MaxRetries` 指定。重試之間的間隔由 `RetryIntervalMilliseconds` 指定。如果 `FirmwareMappers` 有被指定且非空，該任務會向 `/redfish/v1/UpdateService/FirmwareInventory?$expand=.($levels=1)` 發送請求。如果 `SoftwareMappers` 有被指定且非空，該任務會向 `/redfish/v1/UpdateService/SoftwareInventory?$expand=.($levels=1)` 發送請求。

### 從 Redfish 轉換為 D-Bus 表示

在資源收集完成後，每個任務都需要將它們從 Redfish 表示轉換為 D-Bus 表示。對於感測器收集任務，`SensorConfig` 中的 `Mappers` 將用於告知任務哪些感測器是感兴趣的，以及額外的自訂轉換規則。任何找不到符合項目的感測器都會被丟棄。對於每個感兴趣的感測器，會建立一個路徑為 `/xyz/openbmc_project/sensors/{namespace}/{id}` 的 D-Bus 物件。

舉例來說，使用上述範例設定，來自 Redfish 的以下感測器：

```json
{
  "@odata.id": "/redfish/v1/Chassis/HGX_Chassis_0/Sensors/HGX_Chassis_0_TotalGPU_Power_0",
  "@odata.type": "#Sensor.v1_7_0.Sensor",
  "Id": "HGX_Chassis_0_TotalGPU_Power_0",
  "Implementation": "Synthesized",
  "Name": "HGX Chassis 0 TotalGPU Power 0",
  "PhysicalContext": "GPU",
  "Reading": 810.718,
  "ReadingType": "Power",
  "ReadingUnits": "W",
  "RelatedItem": [
    {
      "@odata.id": "/redfish/v1/Systems/HGX_Baseboard_0"
    }
  ],
  "Status": {
    "Conditions": [],
    "Health": "OK",
    "HealthRollup": "OK",
    "State": "Enabled"
  }
}
```

將被轉換為一個路徑為 `/xyz/openbmc_project/sensors/power/HGX_TOTALGPU_PWR_W` 的 D-Bus 物件：

```text
NAME                                        TYPE      SIGNATURE RESULT/VALUE                                  FLAGS
org.freedesktop.DBus.Introspectable         interface -         -                                             -
.Introspect                                 method    -         s                                             -
org.freedesktop.DBus.Peer                   interface -         -                                             -
.GetMachineId                               method    -         s                                             -
.Ping                                       method    -         -                                             -
org.freedesktop.DBus.Properties             interface -         -                                             -
.Get                                        method    ss        v                                             -
.GetAll                                     method    s         a{sv}                                         -
.Set                                        method    ssv       -                                             -
.PropertiesChanged                          signal    sa{sv}as  -                                             -
xyz.openbmc_project.Sensor.Value            interface -         -                                             -
.MaxValue                                   property  d         inf                                           emits-change writable
.MinValue                                   property  d         -inf                                          emits-change writable
.Unit                                       property  s         "xyz.openbmc_project.Sensor.Value.Unit.Watts" emits-change writable
.Value                                      property  d         810.718                                       emits-change writable
```

對於軟體／韌體庫存收集任務，它會有一個與感測器收集任務類似的轉換流程。它會在路徑 `/xyz/openbmc_project/software/` 下建立 D-Bus 物件，並實作 `xyz.openbmc_project.Software.Version` 介面。

對於記錄收集任務，它不會建立 D-Bus 物件，而是使用 phosphor-logging 提供的 `lg2::commit()` API，將記錄項目提交（commit）給記錄服務。為了防止重複提交記錄項目，該任務會在記憶體中記住已提交記錄項目的 `Id` 來追蹤它們。此外，為了防止在 daemon 重新啟動時重複提交記錄項目，該任務也會將記住的 `Id` 持久化到磁碟，並在 daemon 重新啟動時重新載入。

### 資源託管

對於感測器收集任務與軟體／韌體庫存收集任務，這些任務會託管轉換後的 D-Bus 物件，並在每個週期偵測到變更時更新它們。

### Redfish 請求處理

daemon 會善用
[sdbusplus](https://github.com/openbmc/sdbusplus)
提供的非同步函式庫來處理 D-Bus 操作與 Redfish 請求。對於 D-Bus 操作，該函式庫已經有良好的支援。對於 Redfish 請求，將使用 libcurl 與 `sdbusplus/async` 函式庫整合，使得整個 daemon 可以在單一 `sdbusplus::async::context` 下執行，無需考慮額外的執行緒模型，同時確保併發性。這個整合可以透過善用 curl multi 介面提供的 [`curl_multi_waitfds`](https://curl.se/libcurl/c/curl_multi_waitfds.html) API，並使用 `sdbusplus::async::fdio` 來等待回傳的檔案描述元完成。此外，curl multi 介面提供連線快取，因此每個任務可以在各週期之間重複使用連線，因為它們目標是相同的 URL。

為了解析 Redfish 回應，將根據 Redfish schema 建立必要的資料類別，並與 `nlohmann/json` 函式庫整合以進行 JSON 解析。在第一版中，這些類別將以手動方式撰寫，之後可以改為從 Redfish schema 產生程式碼。

### 錯誤處理

如果任何任務失敗，daemon 不應該當機。取而代之，對於在重試次數耗盡後仍然失敗的任務，該任務應該發出一則錯誤記錄，並移除對應的、由其託管的 D-Bus 物件。在下一個週期中，該任務應該繼續嘗試從目標 SMC 收集資源，如果成功，該任務會重新建立 D-Bus 物件並再次開始託管它們。

### 可擴展性

本設計不與任何特定的 SMC 廠商綁定，也不限於前述的資源。在本文件中，Nvidia HMC 僅用於說明目的，但本設計支援任何符合 Redfish 規範的裝置。可擴展性是透過以下機制達成的：

- **多裝置支援**：多個設定檔可以同時被啟用，每個設定檔針對不同的衛星控制器。單一 BMC 可以同時與來自不同廠商的多個 SMC 通訊。只要裝置揭露標準的 Redfish 資源，支援新廠商只需要一個新的設定檔，無需修改 daemon 程式碼。
- **新的資源類型**：每種資源類型都是一個獨立的任務，擁有自己的設定、收集邏輯與 D-Bus 轉換。可以在不修改現有任務的情況下新增新的資源類型。
- **可設定的轉換**：`Mappers` 機制允許針對每個平台自訂如何將 Redfish 資源對應到 D-Bus 物件，將 SMC 的 Redfish schema 與 BMC 的 D-Bus 表示解耦。

### 未來工作

- 支援韌體／軟體更新工作流程
- 支援傾印檔（dump file）擷取工作流程
- 啟用對 SMC 的事件訂閱（Event subscription）

## 已評估的替代方案

### Redfish 聚合

可以利用
[bmcweb](https://github.com/openbmc/bmcweb/blob/master/AGGREGATION.md)
上的 Redfish 聚合來啟用對 SMC 的外部存取，但會有一些限制與開發成本。

以下功能可以由 Redfish 用戶端達成，但要透過 Redfish 聚合來支援則具有挑戰性：

- 讓 BMC 內部的應用程式能夠存取 SMC 資源
  - 散熱校準：「phosphor-pid-control」需要來自 SMC 的感測器值，作為風扇控制演算法的輸入。
  - 感測器閾值監控：某些 SMC 廠商並未實作感測器閾值監控；BMC 必須從 SMC 取得感測器值來執行監控並據以發出事件記錄。
- 用於 SMC 資源揭露的中間層控制：BMC 內部的中間層控制 SMC 資源如何對 BMC 外部揭露，將帶有 SMC 廠商色彩的 Redfish 與 BMC 外部的工具解耦。
  - 在對 BMC 外部揭露 SMC 資源時，使其與一般資源無法區分，從而消除外部工具為了處理 SMC 資源而需要進行的額外開發。舉例來說，來自 SMC 的感測器名稱可以被轉換，以符合外部工具所期望的命名慣例。
  - 變更管理：確保 SMC 的變更不會導致每個下游工具都必須隨之變更。舉例來說：
    - 假設有一個由 SMC 管理、名為 X 的感測器，且有多個下游工具依賴它才能正常運作。
    - 在某個時間點，SMC 廠商決定將感測器 X 重新命名為 Y。
    - 如果感測器 X 直接以其原始形式揭露，那麼廠商只要把 X 改成 Y，所有下游工具就都必須變更。
    - 然而，如果有一個中間層，具有某種將感測器 X 對應到 X* 的設定，那麼只要將該設定從 X 對應到 X* 改為將 Y 對應到 X*，就足以確保所有下游工具在更新後的 SMC 下仍能正常運作。
  - 由 SMC 產生的一些資源或資料可能並非預期要對 BMC 外部揭露。
    - 舉例來說，不揭露 Redfish OEM 指令有助於防止外部工具依賴它們，從而避免由於這些指令僅適用於特定系統或韌體版本而可能產生的相容性問題。

以下功能可以由 Redfish 用戶端達成，且也可能可以由 Redfish 聚合達成，但需要額外的開發工作：

- 由 SMC 產生的 Redfish URI 需要對 BMC 外部可存取。
  - bmcweb 的 Redfish 聚合中已有調整 SMC 回傳 URI 的現有邏輯。然而，這段邏輯可能需要擴充，以處理「messageArgs」可能包含 URI 的情況。
- 事件服務支援：在事件產生時，如果有事件訂閱者，SMC 發出的事件應該被推送給該事件訂閱者。
- 多 SMC 支援

使用 Redfish 聚合來實作上述功能需要大量的開發工作。相比之下，Redfish 用戶端可以開箱即用地提供這些能力。

使用 Redfish 聚合還有一個相關的安全疑慮，涉及到將 SMC 直接暴露於外部網路，並可能在外部呈現 SMC 的安全弱點。SMC 的更新節奏可能與 BMC 的更新節奏脫鉤，因此雖然任何弱點可能都能迅速為 BMC 修復，但要為 SMC 開發並部署修復程式可能需要長得多的時間。

### 在對應的功能儲存庫中實作 Redfish 用戶端

與其建立一個支援多種功能的 Redfish 用戶端 daemon，另一個替代方案是將感測器支援放進 dbus-sensors、將韌體更新支援放進 phosphor-bmc-code-mgmt、將記錄支援放進 phosphor-logging 等等。這種做法的問題在於，Redfish 用戶端的實作需要在多個儲存庫之間重複，導致在進行橫向變更（例如新增新的 Redfish 驗證方法或實作安全性相關的修補程式）時，維護工作與複雜度都會增加，因為在這種情況下需要在多個儲存庫中進行變更。此外，[pldm](https://github.com/openbmc/pldm) 採用了類似的做法，其中 pldmd 是集中式的 daemon，負責溝通 pldm 並支援多種功能，例如韌體更新與 BIOS 控制。

### 設定

與其使用獨立的設定檔，另一個替代方案可能是擴充
[SatelliteController](https://github.com/openbmc/entity-manager/blob/e25a3890392093fec8f49b84c5bbce99055c6768/schemas/satellite_controller.json)
的設定選項，以滿足 Redfish 用戶端 daemon 的需求。這種做法唯一的問題在於，這會讓設定永遠綁定在揭露衛星控制器的實體上，使得無法在包含相同實體但不同機器之間微調設定。

### Libredfish

Libredfish 是建立在 libcurl 之上的 C 函式庫，如果我們選擇依賴它，就需要一個 C++ 包裝層（wrapper），因為我們希望盡可能減少 C 程式碼，並善用 C++ 協程（coroutines）進行非同步操作。然而，與直接在 libcurl 之上建立 C++ 包裝層相比，使用 libredfish 進行非同步操作幾乎沒有好處，因為它的執行緒與佇列模型很可能不同。libredfish 提供的另一項功能是 Redpath，但它大約只有
[200 行的字串解析程式碼](https://github.com/DMTF/libredfish/blob/main/src/redpath.c)，
如有需要，可以輕鬆地在現代 C++ 程式碼中複製。

### Redfish-Schema-C-Struct-Generator

這個工具會產生包含純指標與 char\* 的純 C 結構（struct），這在現代 C++ 開發中並非推薦的做法，因為它可能導致錯誤，並使程式碼更容易發生記憶體洩漏。此外，這個工具似乎維護不良或缺乏活躍開發，最後一次提交是在六個月前，且其相容的 Redfish schema 可追溯至 `DSP8010_Schema_2020.4`。

### Boost.Beast

Boost.Beast 是 daemon 可以使用的另一種 HTTP 函式庫。然而，由於這個函式庫建立在 Boost.Asio 之上，它使用由 `boost::asio::io_context` 驅動的 `boost::asio::awaitable<T>`，而 sdbusplus 使用 stdexec 的 sender/receiver 模式，搭配由 `execution::run_loop` 驅動的 `exec::task<>`。這兩種協程類型無法互換——`boost::asio::awaitable<T>` 無法在 `exec::task<>` 內被 `co_await`，反之亦然。為了橋接兩者，必須在額外的執行緒上執行一個獨立的 `io_context`，並使用搭配 `sdbusplus::async::fdio` 的 `eventfd` 來跨事件迴圈傳送完成訊號。這會為 Asio 的事件迴圈增加一個額外的執行緒，需要手動訊號來跨事件迴圈橋接完成狀態，並在執行緒之間搬移結果時引入執行緒安全性的疑慮。相比之下，搭配 `curl_multi_waitfds` 的 libcurl 可以透過提供可以直接由 `sdbusplus::async::fdio` 監控的檔案描述元，自然地與 sdbusplus 整合，無需額外的執行緒或事件迴圈橋接。關於將 libcurl 與 sdbusplus 整合的下游實作參考，可以在
[async_http_client.cpp](https://github.com/facebook/openbmc/blob/helium/common/lfopenbmc/recipes-downstream/redfish-client/files/src/async_http_client.cpp)
中找到。

## 影響

儘管開發人員需要撰寫額外的設定檔才能啟用 Redfish 用戶端功能，但對其他 BMC 應用程式沒有 API 變更。

如果沒有 Redfish 用戶端 daemon 需要溝通的 SMC，它應該只會造成最小的影響。如果 Redfish 用戶端 daemon 被設定為從 SMC 拉取資料，它可能會消耗 BMC 的一些 CPU 時間，但應該在合理的範圍內。

將在組織內建立一個新的儲存庫。將在 OpenBMC 中新增一個新的 recipe。新儲存庫中會有額外的文件，但本設計將作為起始點。

## 組織面

### 本設計是否需要一個新的儲存庫？

是，daemon 與設定檔都需要一個新的儲存庫。

### 誰將是這個儲存庫的初始維護者？

Patrick Williams。

## 測試

### 單元測試

單元測試將驗證設計中所述將實作的所有程式路徑。

### 整合測試

端到端整合測試將驗證 Redfish 用戶端 daemon 能夠讀取設定檔並執行預期的工作。
