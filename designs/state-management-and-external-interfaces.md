# OpenBMC 狀態管理與外部介面

Author: Andrew Geissler (geissonator)

其他貢獻者：Jason Bills (jmbills)

建立日期：2020 年 1 月 22 日

## 問題描述

隨著 OpenBMC 邁向完整支援 Redfish 協定，在 OpenBMC 內部針對 Resource schema
中的 [ResetType][1] 提供適當的支援相當重要。OpenBMC 目前僅有有限的支援，而本
設計的目標就是要讓這個支援變得更完整。

請注意，本文件的重點在於下列 `ResetType`
執行個體：`redfish/v1/Systems/system/Actions/ComputerSystem.Reset`

這項支援也會對應到現有的 IPMI Chassis Control 指令。

## 背景與參考

[phoshor-state-manager][2] 實作了 xyz.openbmc_project.State.\*
介面。這些介面會控制並追蹤一個由 OpenBMC 管理的系統中，BMC、Chassis（機殼）
以及 Host（主機）的狀態。該 repository 中的 README
可以提供更多背景資訊。[bmcweb][3] 是 OpenBMC 的 web 伺服器及前端 Redfish
介面，它會將對應到 ResetType 物件的指令對應到適當的
xyz.openbmc_project.State.\* D-Bus 介面。

本設計的目標是強化 xyz.openbmc_project.State.\*
介面，以支援更多 Redfish ResetType。具體來說，本設計希望支援的能力是：在不對
機殼切換電源的情況下，重新啟動系統上的作業系統。

目前 phosphor-state-manager 支援以下項目：

- Chassis：On/Off
- Host：On/Off/Reboot

目前對 host 的 `Reboot` 會導致對機殼進行一次電源切換（power cycle）。

### Redfish

Redfish 的 [ResetType][1] 相關聯的操作如下：

```json
"ResetType": {
    "enum": [
        "On",
        "ForceOff",
        "GracefulShutdown",
        "GracefulRestart",
        "ForceRestart",
        "Nmi",
        "ForceOn",
        "PushPowerButton",
        "PowerCycle"
    ],
    "enumDescriptions": {
        "ForceOff": "Turn off the unit immediately (non-graceful shutdown).",
        "ForceOn": "Turn on the unit immediately.",
        "ForceRestart": "Shut down immediately and non-gracefully and restart
          the system.",
        "GracefulRestart": "Shut down gracefully and restart the system.",
        "GracefulShutdown": "Shut down gracefully and power off.",
        "Nmi": "Generate a diagnostic interrupt, which is usually an NMI on x86
          systems, to stop normal operations, complete diagnostic actions, and,
          typically, halt the system.",
        "On": "Turn on the unit.",
        "PowerCycle": "Power cycle the unit.",
        "PushPowerButton": "Simulate the pressing of the physical power button
          on this unit."
    },
    "type": "string"
},
```

### IPMI

IPMI 規格定義了一個 Chassis Control Command，其機殼控制參數如下：

| 選項                         | 說明                                                                                                                                                                                                                                                                                                                                                            |
| ------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| power down                     | 強制系統進入 soft off（S4/S45）狀態。這是用於「緊急」的管理性電源關閉操作。此指令不會在關閉系統電源之前，先啟動作業系統的乾淨關機程序。                                                                                                                                                           |
| power up                       |                                                                                                                                                                                                                                                                                                                                                                        |
| power cycle                    | 此指令在主電源子系統對系統的 POWERGOOD 狀態撤除（deassert）之後，提供至少 1 秒的電源關閉間隔。建議在選擇此操作時若系統電源已關閉（S4/S5），則不執行任何動作，並回傳 D5h「Request parameter(s) not supported in present state.」錯誤完成碼。 |
| hard reset                     | 在某些實作中，BMC 可能不知道重設會造成何種特定影響，而不論電源狀態如何都會對系統重設訊號發出脈波。                                                                                                                                                                                                       |
| pulse Diagnostic Interrupt     | 對直接連接到處理器（們）的診斷中斷發出脈波。這通常用於讓作業系統進行診斷轉儲（取決於作業系統）。                                                                                                                                                                                         |
| Initiate a soft-shutdown of OS |                                                                                                                                                                                                                                                                                                                                                                        |

## 需求

- 保留既有支援：`xyz.openbmc_project.State.Host.Transition.Reboot`
  會導致 host 優雅關機（graceful shutdown）、對機殼進行電源切換，以及啟動
  host。
- 支援在機殼保持開電的狀態下重新啟動 host
  - 支援 `GracefulRestart`（host 會收到重啟通知）
  - 支援 `ForceRestart`（host 不會收到重啟通知）
- 根據系統目前的狀態，將 `PowerCycle`
  對應到 host 或機殼操作。
  - 若 host 正在執行，則 `PowerCycle`
    應對機殼切換電源並啟動 host。
  - 若 host 未在執行，則 `PowerCycle`
    應僅對機殼切換電源。

## 建議設計

建立兩個新的 `xyz.openbmc_project.State.Host.Transition`
選項：

- `ForceWarmReboot`、`GracefulWarmReboot`

建立一個新的 `xyz.openbmc_project.State.Chassis.Transition` 選項：

- `PowerCycle`

現有的 bmcweb 程式碼使用了一些額外的 xyz.openbmc_project.State.\*
介面，這些介面並未在 phosphor-dbus-interfaces 中定義。它們實作於
x86-power-control repository 中，是 phosphor-state-manager
的另一種替代實作。它對這些非 phosphor-dbus-interfaces
的介面有以下對應關係：

- `ForceRestart` -> `xyz.openbmc_project.State.Chassis.Transition.Reset`
- `PowerCycle` -> `xyz.openbmc_project.State.Chassis.Transition.PowerCycle`

`ForceRestart` 應該重新啟動 host，而非機殼。本提案要將 bmcweb 目前對
`ForceRestart` 的對應改為新的 host 過渡狀態：
`xyz.openbmc_project.State.Host.Transition.ForceWarmReboot`

`GracefulRestart` 將對應到我們新的 host 過渡狀態：
`xyz.openbmc_project.State.Host.Transition.GracefulWarmReboot`

`PowerCycle` 操作取決於 host 目前的狀態。若 host 為開機狀態，它會對應到
`xyz.openbmc_project.State.Host.Transition.Reboot`，否則會對應到
`xyz.openbmc_project.State.Chassis.Transition.PowerCycle`

總結新的 Redfish 到 xyz.openbmc_project.State.\* 對應關係：

- `ForceRestart` -> `xyz.openbmc_project.State.Host.Transition.ForceWarmReboot`
- `GracefulRestart`->
  `xyz.openbmc_project.State.Host.Transition.GracefulWarmReboot`
- `PowerCycle`：
  - 若 host 開機：`xyz.openbmc_project.State.Host.Transition.Reboot`
  - 若 host 關機：`xyz.openbmc_project.State.Chassis.Transition.PowerCycle`

Redfish 與 IPMI 到 xyz.openbmc_project.State.\* 的完整對應關係如下：

| Redfish               | IPMI        | xyz.openbmc_project.State.Transition |
| --------------------- | ----------- | ------------------------------------ |
| ForceOff              | power down  | Chassis.Off                          |
| ForceOn               | power up    | Host.On                              |
| ForceRestart          | hard reset  | Host.ForceWarmReboot                 |
| GracefulRestart       |             | Host.GracefulWarmReboot              |
| GracefulShutdown      | soft off    | Host.Off                             |
| On                    | power up    | Host.On                              |
| PowerCycle (host on)  | power cycle | Host.Reboot                          |
| PowerCycle (host off) |             | Chassis.PowerCycle                   |

## 已考量的替代方案

未考量其他替代方案。

## 影響

現有介面將維持不變。需要對 x86-power-control 做一些修改，以確保 bmcweb
的對應能如預期地搭配新介面運作。

需要對 phosphor-host-ipmid 做一些修改，以支援新的狀態過渡。

## 測試

這些新的狀態過渡將由 systemd targets 作為後端支援。這些 targets
會經過徹底測試。OpenBMC 測試團隊將被接洽，以確保這些項目在自動化測試中被測試。

[1]: http://redfish.dmtf.org/schemas/v1/Resource.json#/definitions/ResetType
[2]: https://github.com/openbmc/phosphor-state-manager
[3]: https://github.com/openbmc/bmcweb
