# 使用 OpenBMC 進行 Host 管理

本文件說明 OpenBMC 物件結構中可透過 REST 存取的 host-management 介面。

注意：認證

關於認證的詳細資訊請參閱
[REST-cheatsheet](https://github.com/openbmc/docs/blob/master/REST-cheatsheet.md#establish-rest-connection-session)。

本文件使用 token 認證方式：

```bash
export bmc=xx.xx.xx.xx
export token=`curl -k -H "Content-Type: application/json" -X POST https://${bmc}/login -d '{"username" :  "root", "password" :  "0penBmc"}' | grep token | awk '{print $2;}' | tr -d '"'`
curl -k -H "X-Auth-Token: $token" https://${bmc}/xyz/openbmc_project/...
```

## Inventory

系統 inventory 結構位於 `/xyz/openbmc_project/inventory` 階層之下。

在 OpenBMC 中，inventory 以路徑表示，該路徑與實體系統拓撲呈階層對應。
inventory 中的項目稱為 inventory 項目（inventory items），且不一定是 FRU
（field-replaceable units，現場可更換單元）。若系統含有一個機殼（chassis）、
一塊主機板，以及主機板上的一顆 CPU，則該 inventory 項目的路徑會是：

`inventory/system/chassis0/motherboard0/cpu0`

與 inventory 項目關聯的屬性取決於該項目本身。一些常見的屬性包括：

- `Version`：與此項目相關聯的程式碼版本。
- `Present`：表示此項目是否存在於系統中（True/False）。
- `Functional`：表示此項目在系統中是否正常運作（True/False）。

一般性的 `list` 與 `enumerate` REST 查詢可用來存取系統 inventory 結構。
例如，要列舉所有 inventory 項目及其屬性：

```bash
curl -k -H "X-Auth-Token: $token" https://${bmc}/xyz/openbmc_project/inventory/enumerate
```

要列出某個項目的屬性：

```bash
curl -k -H "X-Auth-Token: $token" https://${bmc}/xyz/openbmc_project/inventory/system/chassis/motherboard
```

## Sensors

系統 sensor 結構位於 `/xyz/openbmc_project/sensors` 階層之下。

此介面可監控如溫度或高度等系統屬性，並以類似 inventory 的方式表示：即頂層
`sensors` 物件名稱之下的物件路徑。路徑用於對 sensor 分類並顯示該 sensor
所代表的意義，但不一定代表系統的實體拓撲。

例如，所有溫度 sensor 都位於 `sensors/temperature` 之下。CPU 溫度 sensor
會是 `sensors/temperature/cpu[n]`。

以下是一些常見屬性：

- `Value`：sensor 的目前值
- `Unit`：值以及 "Critical" 和 "Warning" 值的單位
- `Scale`：值以及 "Critical" 和 "Warning" 值的比例尺度
- `CriticalHigh` 與 `CriticalLow`：sensor 裝置的上／下臨界（critical）門檻邊界
- `CriticalAlarmHigh` 與 `CriticalAlarmLow`：若 sensor 已超過臨界門檻邊界則為 True
- `WarningHigh` 與 `WarningLow`：sensor 裝置的上／下警告（warning）門檻邊界
- `WarningAlarmHigh` 與 `WarningAlarmLow`：若 sensor 已超過警告門檻邊界則為 True

一個溫度 sensor 可能看起來像這樣：

```bash
    curl -k -H "X-Auth-Token: $token" https://${bmc}/xyz/openbmc_project/sensors/temperature/ocp_zone
    {
      "data": {
        "CriticalAlarmHigh": false,
        "CriticalAlarmLow": false,
        "CriticalHigh": 65000,
        "CriticalLow": 0,
        "Functional": true,
        "MaxValue": 0,
        "MinValue": 0,
        "Scale": -3,
        "Unit": "xyz.openbmc_project.Sensor.Value.Unit.DegreesC",
        "Value": 34625,
        "WarningAlarmHigh": false,
        "WarningAlarmLow": false,
        "WarningHigh": 63000,
        "WarningLow": 0
      },
      "message": "200 OK",
      "status": "ok"
    }
```

請注意此 sensor 的值為 34.625C（34625 \* 10^-3）。

不同於 IPMI，OpenBMC 中沒有「functional」sensor；功能狀態是以 inventory
來表示。

要列舉系統中的所有 sensor：

```bash
curl -k -H "X-Auth-Token: $token" https://${bmc}/xyz/openbmc_project/sensors/enumerate
```

列出某個 inventory 項目的屬性：

```bash
curl -k -H "X-Auth-Token: $token" https://${bmc}/xyz/openbmc_project/sensors/temperature/outlet
```

## Event Logs

事件記錄（event log）結構位於 `/xyz/openbmc_project/logging/entry` 階層之下。
每個事件都是此結構下一個獨立的物件，以編號參照。

在 POWER 架構的伺服器上，BMC 與 host 韌體皆可向 BMC 回報事件記錄。通常，
這些事件記錄是在 host 韌體無法啟動 OS，或無法可靠地記錄到 OS 的情況下回報。

與事件記錄關聯的屬性如下：

- `Message`：事件記錄的類型（例如
  "xyz.openbmc_project.Inventory.Error.NotPresent"）。
- `Resolved` ：表示該事件是否已解決。
- `Severity`：問題的層級（"Info"、"Error" 等）。
- `Timestamp`：事件記錄的日期，以 epoch 時間表示。
- `Associations`：指向故障 inventory 零件的 URI。

要列出所有已回報的事件記錄：

```bash
    curl -k -H "X-Auth-Token: $token" https://${bmc}/xyz/openbmc_project/logging/entry
    {
      "data": [
        "/xyz/openbmc_project/logging/entry/3",
        "/xyz/openbmc_project/logging/entry/2",
        "/xyz/openbmc_project/logging/entry/1",
        "/xyz/openbmc_project/logging/entry/7",
        "/xyz/openbmc_project/logging/entry/6",
        "/xyz/openbmc_project/logging/entry/5",
        "/xyz/openbmc_project/logging/entry/4"
      ],
      "message": "200 OK",
      "status": "ok"
    }
```

要讀取特定的事件記錄：

```bash
    curl -k -H "X-Auth-Token: $token" https://${bmc}/xyz/openbmc_project/logging/entry/1
    {
      "data": {
        "AdditionalData": [
          "_PID=183"
        ],
        "Id": 1,
        "Message": "xyz.openbmc_project.Common.Error.InternalFailure",
        "Purpose": "xyz.openbmc_project.Software.Version.VersionPurpose.BMC",
        "Resolved": false,
        "Severity": "xyz.openbmc_project.Logging.Entry.Level.Error",
        "Timestamp": 1563191362822,
        "Version": "2.8.0-dev-132-gd1c1b74-dirty",
        "associations": []
      },
      "message": "200 OK",
      "status": "ok"
    }
```

要刪除事件記錄（此範例中為 log 1），請對該事件呼叫 `Delete` 方法：

```bash
curl -k -H "X-Auth-Token: $token" -H "Content-Type: application/json" -X POST -d '{"data" : []}' https://${bmc}/xyz/openbmc_project/logging/entry/1/action/Delete
```

要清除所有事件記錄，請呼叫頂層的 `DeleteAll` 方法：

```bash
curl -k -H "X-Auth-Token: $token" -H "Content-Type: application/json" -X POST -d '{"data" : []}' https://${bmc}/xyz/openbmc_project/logging/action/DeleteAll
```

## Host 開機選項

在 OpenBMC 中，Host 開機選項是以 D-Bus 屬性儲存在 `control/host0/boot` 路徑
之下。屬性包括
[`BootMode`](https://github.com/openbmc/phosphor-dbus-interfaces/blob/master/yaml/xyz/openbmc_project/Control/Boot/Mode.interface.yaml)、
[`BootSource`](https://github.com/openbmc/phosphor-dbus-interfaces/blob/master/yaml/xyz/openbmc_project/Control/Boot/Source.interface.yaml)，
以及若 host 採用 x86 CPU，還包括
[`BootType`](https://github.com/openbmc/phosphor-dbus-interfaces/blob/master/yaml/xyz/openbmc_project/Control/Boot/Type.interface.yaml)。

- 設定開機模式：

```bash
curl -k -H "X-Auth-Token: $token" -H "Content-Type: application/json" -X PUT https://${bmc}/xyz/openbmc_project/control/host0/boot/attr/BootMode -d '{"data": "xyz.openbmc_project.Control.Boot.Mode.Modes.Regular"}'
```

- 設定開機來源：

```bash
curl -k -H "X-Auth-Token: $token" -H "Content-Type: application/json" -X PUT https://${bmc}/xyz/openbmc_project/control/host0/boot/attr/BootSource -d '{"data": "xyz.openbmc_project.Control.Boot.Source.Sources.Default"}'
```

- 設定開機類型（僅當 host 採用 x86 CPU 時有效）：

```bash
curl -k -H "X-Auth-Token: $token" -H "Content-Type: application/json" -X PUT https://${bmc}/xyz/openbmc_project/control/host0/boot/attr/BootType -d '{"data": "xyz.openbmc_project.Control.Boot.Type.Types.EFI"}'
```

此外還有布林值的
[`Enable`](https://github.com/openbmc/phosphor-dbus-interfaces/blob/master/yaml/xyz/openbmc_project/Object/Enable.interface.yaml)
屬性，用以控制開機來源覆寫（override）是永久生效還是僅一次性生效，以及該
覆寫是否啟用。

- 設定開機覆寫的一次性（one-time）旗標：

```bash
curl -k -H "X-Auth-Token: $token" -H "Content-Type: application/json" -X PUT https://${bmc}/xyz/openbmc_project/control/host0/boot/one_time/attr/Enabled -d '{"data": "true"}'
```

- 啟用開機覆寫：

```bash
curl -k -H "X-Auth-Token: $token" -H "Content-Type: application/json" -X PUT https://${bmc}/xyz/openbmc_project/control/host0/boot/attr/Enabled -d '{"data": "true"}'
```

## Host 狀態控制

Host 可以透過 `host` 物件來控制。該物件實作了多種動作，包括開機（power on）
與關機（power off）。這些動作對應到 IPMI 的 `power on` 與 `power off` 指令。

假設你已登入，以下指令將開啟 host 電源：

```bash
curl -k -H "X-Auth-Token: $token" -H "Content-Type: application/json" -d '{"data": "xyz.openbmc_project.State.Host.Transition.On"}' -X PUT https://${bmc}/xyz/openbmc_project/state/host0/attr/RequestedHostTransition
```

要關閉 host 電源：

```bash
curl -k -H "X-Auth-Token: $token" -H "Content-Type: application/json" -d '{"data": "xyz.openbmc_project.State.Host.Transition.Off"}' -X PUT https://${bmc}/xyz/openbmc_project/state/host0/attr/RequestedHostTransition
```

要執行強制關機（hard power off，透過關閉機殼電源來達成）：

```bash
curl -k -H "X-Auth-Token: $token" -H "Content-Type: application/json" -X PUT -d '{"data":"xyz.openbmc_project.State.Chassis.Transition.Off"}' https://${bmc}//xyz/openbmc_project/state/chassis0/attr/RequestedPowerTransition
```

要重新啟動 host：

```bash
curl -k -H "X-Auth-Token: $token" -H "Content-Type: application/json" -X PUT -d '{"data":"xyz.openbmc_project.State.Host.Transition.Reboot"}' https://${bmc}/xyz/openbmc_project/state/host0/attr/RequestedHostTransition
```

關於 Host 狀態管理的更多資訊請見此處：
<https://github.com/openbmc/phosphor-dbus-interfaces/tree/master/yaml/xyz/openbmc_project/State>

## Host 清除 GARD

在 OpenPOWER 系統上，host 會在 GARD 分割區中維護一份關於損壞或無法運作元件
的記錄。host 在後續開機時會參考這份記錄，以決定應忽略哪些零件。

BMC 實作了一個可清除此分割區的功能。該功能可如下呼叫：

- 方法 1：從 BMC 命令列：

```bash
  busctl call org.open_power.Software.Host.Updater \
    /org/open_power/control/gard \
    xyz.openbmc_project.Common.FactoryReset Reset
```

- 方法 2：使用 REST API：

```bash
curl -k -H "X-Auth-Token: $token" -H "Content-Type: application/json" -X POST -d '{"data":[]}' https://${bmc}/org/open_power/control/gard/action/Reset
```

實作： <https://github.com/openbmc/openpower-pnor-code-mgmt>

## Host Watchdog

host watchdog 服務負責確保 host 能在合理的時間內啟動並完成開機。當 host
啟動時，watchdog 也會啟動，並預期 host 會在開機過程中透過 inband 介面定期對
watchdog 送出 ping。若 host 未在逾時（timeout）時間內對 watchdog 送出 ping，
則 host watchdog 會啟動一個 systemd target 以進入 quiesce target。之後系統
設定將決定從該狀態開始的復原行為，例如嘗試重新啟動系統。

host watchdog 使用通用的 [phosphor-watchdog][1] 存放庫。host watchdog 服務
提供 2 個檔案作為 phosphor-watchdog 的設定選項：

```text
    /lib/systemd/system/phosphor-watchdog@poweron.service.d/poweron.conf
    /etc/default/obmc/watchdog/poweron
```

`poweron.conf` 包含「Conflicts」關係，以確保 watchdog 服務在正確的時機被停止。
`poweron` 則包含 phosphor-watchdog 所需的資訊（關於這些檔案的更多資訊可在
[phosphor-watchdog][1] 存放庫中找到）。

與 host watchdog 相關的 2 個服務檔案為：

```text
    phosphor-watchdog@poweron.service
    obmc-enable-host-watchdog@0.service
```

`phosphor-watchdog@poweron` 會啟動 host watchdog 服務，而
`obmc-enable-host-watchdog` 會啟動 watchdog 計時器。兩者皆作為
`obmc-host-startmin@.target` 的一部分執行。服務相依性確保服務會在 enable
被呼叫之前啟動。

預設的 watchdog 逾時時間可在
[dbus 介面規格][2]（Interval 屬性）中找到。

host 在啟動後會控制 watchdog 的逾時時間以及啟用／停用。

[1]: https://github.com/openbmc/phosphor-watchdog
[2]:
  https://github.com/openbmc/phosphor-dbus-interfaces/blob/master/yaml/xyz/openbmc_project/State/Watchdog.interface.yaml
