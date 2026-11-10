# OpenBMC 上的系統開機進度

作者：Andrew Geissler `geissonator`

其他貢獻者：無

建立日期：2020 年 5 月 27 日

## 問題描述

由 OpenBMC 管理的系統，其開機進度相當複雜。共有 phosphor D-Bus 屬性、IPMI
感測器、PLDM 感測器以及 Redfish 屬性來呈現這個進度。本設計文件的目的，就是
突顯這些不同的實體，並為系統實作者提供一份關於該實作哪些內容的基礎依據。

## 背景與參考

[phosphor-state-manager][1] 實作了用來追蹤 BMC、Chassis 與 Host 狀態的 D-Bus
屬性。本設計主要聚焦在 Host 的狀態，因為它最為複雜。具體而言，目標是要提供
關於主機在開機期間正在做什麼的詳細資訊。

[HostState][2] 提供基本的 `Off`/`Running` 類型資訊，另有一個錯誤狀態
（`Quiesced`）與除錯狀態（`DiagnosticMode`）。本文件的焦點是 `Running` 狀態，
它僅表示主機處理器上是否有東西正在執行。

phosphor-state-manager 另外還實作了幾個代表主機開機進度的 D-Bus 屬性：

- [xyz.openbmc_project.State.Boot.Progress][3]
- [xyz.openbmc_project.State.OperatingSystem.Status][4]

這兩個 D-Bus 屬性非常以 IPMI 為中心。它們是根據兩個 IPMI 感測器所定義，由
主機韌體在開機啟動系統的過程中設定。

PLDM 也有一個開機進度感測器。請在此[文件][5]中搜尋 "Boot Progress"。其中有一
子集可以相當貼切地對應到上述的 IPMI 感測器。這個 PLDM 感測器目前尚未在
OpenBMC 程式碼中實作。

Redfish 以多種方式來呈現系統狀態。BMC、Chassis 與 System 都實作了一個
[Status][6] 物件。它提供一份適用於 Redfish 物件的通用 `State` 選項清單。針對
phosphor-state-manager 對應到 Redfish System [Status][7]，OpenBMC 有下列對應
方式：

- `xyz.openbmc_project.State.Host.HostState.Running` : `Enabled`
- `xyz.openbmc_project.State.Host.HostState.Quiesced` : `Quiesced`
- `xyz.openbmc_project.State.Host.HostState.DiagnosticMode` : `InTest`

自 [ComputerSystem.v1_13_0][7] 起，`BootProgress` 物件已正式納入 Redfish
ComputerSystem schema。

總結來說，這個新的 `BootProgress` 物件下的 `LastState` 屬性會追蹤系統的下列
開機狀態：

- None
- PrimaryProcessorInitializationStarted
- BusInitializationStarted
- MemoryInitializationStarted
- SecondaryProcessorInitializationStarted
- PCIResourceConfigStarted
- SystemHardwareInitializationComplete
- OSBootStarted
- OSRunning
- OEM

這個新的 BootProgress 物件還有一個相關的 `LastStateTime`，它應該要更新為
`LastState` 最後一次更新的日期與時間。

最終的目標，是要能夠將 IPMI 或 PLDM 的開機感測器對應到這個新 Redfish 屬性中
的全部或部分值。

請注意，本設計並未納入多主機（multi-host）電腦系統的概念，但也不會將它們
排除在外。`BootProgress` D-Bus 屬性會與一個 `/xyz/openbmc_project/state/host<X>`
實例產生關聯，其中 `<X>` 是主機實例編號。同樣地，Redfish 的系統物件也是一個
以實例為基礎的物件。

## 需求

- 增強既有的 [BootProgress][3] D-Bus 屬性，以涵蓋所有支援的 PLDM 開機進度碼
- 建立一個新的 `BootProgressLastUpdate` D-Bus 屬性，用來存放 `BootProgress`
  最後一次更新的日期與時間
- 確保在 IPMI 與 PLDM 兩類堆疊（stack）中，`BootProgress` 與
  `BootProgressLastUpdate` 屬性都能被適當地更新
  - 在 D-Bus 上更新 `BootProgress` 屬性是 IPMI 或 PLDM 實作的責任
  - 當 phosphor-state-manager 看到 `BootProgress` 更新時，在 D-Bus 上更新
    `BootProgressLastUpdate` 屬性是它的責任
- 確保新的 Redfish `LastState` 與 `LastStateTime` 屬性，對 `BootProgress` 與
  `BootProgressLastUpdate` D-Bus 屬性有適當的對應

不同的 OpenBMC 系統可能支援不同的開機進度碼，並以不同的順序支援它們。本文件
不嘗試對此設下任何需求，只要求它們全部必須定義在 `BootProgress` D-Bus 屬性
之內，而那些能對應到 Redfish BootProgress 物件的，將會透過 Redfish 介面顯示
出來。

## 建議設計

[BootProgress][3] 將被增強，以支援一個涵蓋 IPMI 與 PLDM 所提供內容的超集合
（superset）。

IPMI 程式碼已經會根據設定檔來設定 `BootProgress` D-Bus 屬性。請在此[檔案][8]
中尋找 `fw_boot_sensor`，作為某個特定 IPMI 感測器如何對應到此 D-Bus 屬性的
範例。

PLDM 軟體將會做類似的事情。

接著 bmcweb 會使用這個 `BootProgress` D-Bus 介面，來適當地回應 Redfish 請求。

## 考慮過的替代方案

許多系統 BIOS 都提供某種形式的詳細開機進度碼。UEFI 有 POST 碼，POWER 有 istep
進度碼。如果需要更細部的資訊，我們可以考慮使用這些。但目前的需求只是高階的
開機進度。請注意，這些 POST/istep 碼若有需要是可以對應到適當的 BootProgress
值的（或者，適用於主機不支援內含 BootProgress 的 IPMI 或 PLDM 堆疊的情況）。

## 影響

每個系統都需要記錄它們支援哪些 `BootProgress` 碼，以及系統開機時這些碼的預期
順序。

## 測試

確保 IPMI 與 PLDM 基礎的系統能夠開機，並如預期地更新 `BootProgress` D-Bus 屬性。

[1]:
  https://github.com/openbmc/phosphor-state-manager#state-tracking-and-control
[2]:
  https://github.com/openbmc/phosphor-dbus-interfaces/blob/master/yaml/xyz/openbmc_project/State/Host.interface.yaml
[3]:
  https://github.com/openbmc/phosphor-dbus-interfaces/blob/master/yaml/xyz/openbmc_project/State/Boot/Progress.interface.yaml
[4]:
  https://github.com/openbmc/phosphor-dbus-interfaces/blob/master/yaml/xyz/openbmc_project/State/OperatingSystem/Status.interface.yaml
[5]:
  https://www.dmtf.org/sites/default/files/standards/documents/DSP0249_1.0.0.pdf
[6]: http://redfish.dmtf.org/schemas/v1/Resource.json#/definitions/Status
[7]: https://redfish.dmtf.org/schemas/v1/ComputerSystem.v1_13_0.json
[8]:
  https://github.com/openbmc/meta-ibm/blob/master/recipes-phosphor/configuration/acx22-yaml-config/acx22-ipmi-sensors-mrw.yaml
