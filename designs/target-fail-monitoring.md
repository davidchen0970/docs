# 監控與記錄 OpenBMC Systemd Target 及 Service 失敗

Author: Andrew Geissler < geissonator >

Created: June 18, 2019 Last Updated: Feb 21, 2022

## 問題描述

OpenBMC 使用 systemd 來協調應用程式的啟動與停止。舉例來說，systemd 負責啟動
OpenBMC 的所有初始服務、為其所管理的伺服器開機與關機，以及執行韌體更新等
操作。

[openbmc-systemd.md][1] 對 systemd 及其在 OpenBMC 中的基本用法有良好的摘要。

從高層次來看，systemd 由 unit 組成。對 OpenBMC 而言，兩種關鍵的 unit 類型是
target 與 service。

有些情況下 OpenBMC 的 systemd target 會失敗。它們失敗是因為其中的 target 或
service 遇到了某種錯誤狀態。在某些情況下，造成失敗的 unit 會記錄一筆錯誤到
phosphor-logging，但在另一些情況下（segfault、未處理的例外、未處理的錯誤路徑
⋯），最終使用者不會收到任何失敗的跡象。如果 OpenBMC 內部的 systemd target
失敗，務必通知使用者，這是非常重要的。

也有一些場景是系統已成功啟動所有 target，但該 target 中一個執行中的 service
隨後失敗。有些 service 並不是那麼關鍵，但像是 fan-control 或電源監控服務這類
東西可能非常關鍵。起碼必須確保系統的使用者能被告知某個關鍵 service 已經失敗。
本文件所提出的方案並不排斥 service 的擁有者自行做其他復原動作，這個方案只是
確保在失敗發生時，至少會完成基本層級的回報。

## 背景與參考

關於 OpenBMC 中狀態管理與 systemd 的背景資訊，請參閱
[phosphor-state-manager][2] repository。

systemd 會在 unit 完成時發出信號，並提供該已完成 unit 的狀態。請參閱
[systemd dbus API][3] 中的 JobNew()/JobRemoved() 章節。六種不同的結果為：

```text
done, canceled, timeout, failed, dependency, skipped
```

phosphor-state-manager 的程式碼已經在監控這些信號，但只尋找 `done` 狀態，用以
判斷何時該把對應的 dbus object 標記為 ready(bmc)/on(chassis)/running(host)。

本文件中的提案是同時監控其他幾種結果，並將適當的錯誤記錄到 phosphor-logging。

一個作為 service 的 systemd unit，只會在 systemd 定義的所有重試都執行完畢後，
才會進入 `failed` 狀態。對 OpenBMC 系統而言，這代表在 30 秒的時間視窗內重啟
2 次。

## 需求

### Systemd Target Units

- 必須能夠監控任意 systemd target，並根據 target 遇到的失敗類型記錄一個定義好的
  錯誤
- 必須可設定
  - Target：選擇任意 systemd target
  - Status：選擇要監控的一種或多種狀態
  - 要記錄的錯誤：指定偵測到錯誤時要記錄的錯誤
- 必須能在啟動時接受多個設定檔作為輸入
- 針對要監控的錯誤支援 `default` 預設值
  - 預設值為 `timeout`、`failed` 與 `dependency`
- 記錄的錯誤永遠是使用設定指定的錯誤，並附上額外資料指出失敗的狀態（例如
  canceled、timeout ⋯）
  - 範例：

```text
    xyz.openbmc_project.State.BMC.Error.MultiUserTargetFailure
    STATUS=timeout
```

- 預設情況下，為所有關鍵的 OpenBMC target 啟用這個監控與記錄服務
  - 關鍵的 systemd target 是指 phosphor-state-manager 所使用的 target
    - BMC：multi-user.target
    - Chassis：`obmc-chassis-poweron@0.target, obmc-chassis-poweroff@0.target`
    - Host：`obmc-host-start@0.target, obmc-host-startmin@0.target`、
      `obmc-host-shutdown@0.target`、`obmc-host-stop@0.target`、
      `obmc-host-reboot@0.target`
  - 這些 target 的錯誤必須定義在 phosphor-dbus-interfaces 中
- 限制：
  - 必須輸入完整限定的 target 名稱（亦即不支援模板／萬用字元 target）

### Systemd Service Units

- 必須能夠監控 OpenBMC 中任意 systemd service
  - 要監控的 service 必須可設定
- 記錄一筆錯誤以表示某個 service 已失敗（日誌中包含 service 名稱）
  - `xyz.openbmc_project.State.Error.CriticalServiceFailure`
- 收集一份 BMC dump
- 變更 BMC 狀態（CurrentBMCState）以表示 BMC 處於降級模式
- 透過 Redfish managers/bmc state 對外回報狀態變更

### Immediate-Quiesce Service 監控

上述的 service 監控依賴 systemd 的 `JobRemoved` 信號及其 `result` 欄位。這對於
用盡重試上限的 `Restart=no` service 運作良好，但有兩個盲點：

1. **`Restart=always` 的 service** —— 當這類 service 當機時，systemd 會立即排程
   重啟。如果重啟成功，`JobRemoved` 的結果是 `done`，監控因此永遠不會觸發。
   然而，即使是關鍵 service（例如 ObjectMapper）短暫當機，也可能讓相依的
   D-Bus 呼叫永久失敗，使系統停留在降級狀態而毫無跡象。

2. **正常運作期間 `Restart=no` 的 service** —— 如果這類 service 在 target 轉換
   之外當機，不會建立新的 job，也不會發出 `JobRemoved` 信號，監控同樣看不到。

為了解決這個問題，可以在用於 service 監控（`-s`）的同一份 JSON 檔案中加入一個
選用的 `immediate_quiesce_services` 鍵。對於列在此鍵下的每個 service，監控程式
會在其 systemd unit 的 `ActiveState` 屬性上安裝一個 D-Bus `PropertiesChanged`
match。當 `ActiveState` 變成 `"failed"` 時，監控程式會立即：

- 收集一份 BMC dump
- 記錄 `xyz.openbmc_project.State.Error.CriticalServiceFailure`
- 觸發 BMC quiesce（`obmc-bmc-service-quiesce@0.target`）

這會在 systemd 將 unit 標記為 failed 的當下觸發——在任何重啟之前——因此不論
unit 的重啟政策如何，失敗一定會被捕捉到。

請注意，只有 `"failed"` 會觸發 quiesce。正常的管理操作（`systemctl stop`、
`systemctl restart`、BMC 關機）會經過 `"inactive"` 而非 `"failed"`，因此不會
造成誤報。

**重要：** BMC quiesce 是一個重大操作——BMC 會進入通常需要手動介入才能恢復的
降級狀態。只有在單次短暫當機確實會造成無法修復的損害時，才將 service 加入
`immediate_quiesce_services`。大多數 service 在 systemd 重啟後都能正常恢復，
**不應該**列在這裡。

## 建議設計

在 phosphor-state-manager 中建立一個新的獨立應用程式，於啟動時載入 json 檔案。

針對 targets，json 檔案的格式如下：

```json
{
  "targets": [
    {
      "name": "multi-user.target",
      "errorsToMonitor": ["default"],
      "errorToLog": "xyz.openbmc_project.State.BMC.Error.MultiUserTargetFailure"
    },
    {
      "name": "obmc-chassis-poweron@0.target",
      "errorsToMonitor": ["timeout", "failed"],
      "errorToLog": "xyz.openbmc_project.State.Chassis.Error.PowerOnTargetFailure"
    }
  ]
}
```

針對 services，json 檔案的格式如下：

```json
{
  "services": [
    "xyz.openbmc_project.biosconfig_manager.service",
    "xyz.openbmc_project.Dump.Manager.service"
  ]
}
```

同一份 json 檔案中可以包含一個選用的 `immediate_quiesce_services` 鍵。列在這裡
的 service 會透過其 systemd unit `ActiveState` 屬性上的 `PropertiesChanged` 來
監控，並在失敗時立即觸發 BMC quiesce：

```json
{
  "services": [
    "xyz.openbmc_project.biosconfig_manager.service",
    "xyz.openbmc_project.Logging.service"
  ],
  "immediate_quiesce_services": [
    "xyz.openbmc_project.ObjectMapper.service",
    "phosphor-multi-gpio-monitor.service"
  ]
}
```

這兩個清單對應不同的嚴重程度，通常應該包含不同的 service：

- `services` —— 只有在 systemd 的所有重試都用盡後才 quiesce（透過 JobRemoved）。
  適合通常在重啟後就能正常恢復的 service。
- `immediate_quiesce_services` —— 在**第一次** `ActiveState=failed` 時就 quiesce，
  甚至在 systemd 嘗試重啟之前。適合即使短暫當機也會造成無法修復損害的 service。

列在 `immediate_quiesce_services` 中的 service 不需要同時出現在 `services` 中
——立即機制會先觸發，使 JobRemoved 的備援機制變成多餘。沒有 `immediate_quiesce_services`
鍵的檔案會在該部分被靜默跳過（向後相容）。

啟動時，所有輸入的 json 檔案都會被載入並完成監控設定。

這個應用程式不會在 D-Bus 上註冊任何介面，但會訂閱 systemd 的 dbus 信號：
JobRemoved（target 與 service 監控），以及個別 unit object 上的
PropertiesChanged（immediate-quiesce service 監控）。所有的 D-Bus 通訊都會使用
sdeventplus。

為了額外的除錯用途，可以將錯誤註冊到 BMC Dump 功能，以確保能判斷失敗的原因。
這需要將本服務記錄的錯誤放入
`phosphor-debug-errors/errors_watch.yaml`。

對於 service 失敗，預設會收集一份 dump，因為 BMC 會被轉入 Quiesced 狀態。

請注意，那些負責讓系統從一個 target 轉換到另一個 target 的短時間執行應用程式
（例如 chassis 開機／關機、host 啟動／停止⋯）是關鍵 service，但它們不是本設計
要監控的那種 service。它們的失敗會造成 systemd target 失敗，進而落入本設計中
target 監控的範疇。service 監控是針對在 target 完成後仍持續執行的長時間執行
service。

## 曾考慮的替代方案

Unit 有 OnError 指令。可以將錯誤記錄邏輯放在那條路徑中，但這會為 OpenBMC 的
systemd 用法引入更多複雜度，而其現況已經相當複雜。

也可以在現有的 state manager 應用程式中實作，因為它們已經在監控部分這類 unit。
最終選擇了獨立應用程式以及可監控任意 unit 的通用能力，作為較佳的選項。

## 影響

當上述列出的某個 unit 失敗時，會記錄一筆 phosphor-logging 事件。系統的擁有者
可以檢視這筆事件。可能有些情況同一個問題會產生兩次日誌。舉例來說，如果電源
應用程式在開機過程中偵測到問題並記錄到 phosphor-logging，然後對 systemd 回傳
非零值，該 service 所屬的 target 也會失敗，而本設計中定義的程式邏輯同樣會記錄
一筆錯誤。

我們的想法是：兩筆錯誤總比沒有好。此外，phosphor-logging 正在定義一套外掛
（plugin）架構。使用者可以在 phosphor-logging 的外掛架構中監控第一筆錯誤，並
並將其定義為一個動作，取消該 systemd target。target 狀態為 `canceled` 時，不會
導致 phosphor-state-manager 產生錯誤。

## 測試

需要讓本設計中提到的所有 unit 失敗。它們必須針對本設計中定義的每種原因失敗，
並驗證每種情境所產生的錯誤。

關於 immediate-quiesce service 監控，殺掉一個受監控的 service（例如
`systemctl kill -s SIGKILL xyz.openbmc_project.ObjectMapper.service`）並驗證：

- 記錄了一筆 `CriticalServiceFailure` 錯誤，且包含正確的 unit 名稱
- 收集了一份 BMC dump
- `CurrentBMCState` 轉換為 `Quiesced`
- 即使 service 隨後成功重啟，quiesce 仍會觸發

[1]: https://github.com/openbmc/docs/blob/master/architecture/openbmc-systemd.md
[2]: https://github.com/openbmc/phosphor-state-manager
[3]: https://www.freedesktop.org/wiki/Software/systemd/dbus/
