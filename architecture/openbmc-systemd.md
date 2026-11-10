# OpenBMC 與 Systemd

OpenBMC 使用 [systemd](https://www.freedesktop.org/wiki/Software/systemd/) 來
管理所有程序。它有一整套自己的 target 與 service unit，用來控制哪些程序會被
啟動。關於 systemd 有為數众多的文件，而要做 OpenBMC 狀態相關的開發工作，你
勢必得閱讀其中一些。以下是重點：

[Unit](https://www.freedesktop.org/software/systemd/man/systemd.unit.html#) -
Unit 是所有 systemd 工作的基本框架。
[Service](https://www.freedesktop.org/software/systemd/man/systemd.service.html) -
Service 是 unit 的一種類型，用來定義要執行與運作的程序。
[Target](https://www.freedesktop.org/software/systemd/man/systemd.target.html) -
Target 是另一種類型的 unit，它們有兩個用途：

1. 定義各 service 之間的同步點。
2. 定義針對某個 target 會執行的 service。

在 OpenBMC 系統上，你可以到 /lib/systemd/system/ 查看系統上所有的 systemd
unit。你可以輕易地用 cat 檢視這些檔案，開始觀察它們之間的關係。Service 檔案
也可以在 /etc/systemd/system 與 /run/systemd/system 中找到。

[systemctl](https://www.freedesktop.org/software/systemd/man/systemctl.html)
是你用來與 systemd 及其 unit 互動的主要工具。

---

## 初始上電

當 OpenBMC 系統第一次通電時，它會啟動 "default.target"，除非在 kernel 命令
列上指定了其他 target。在 Phosphor OpenBMC 中，從 `default.target` 有一個連結
指向 `multi-user.target`。

你會找到所有與 `multi-user.target` 相關的 phosphor service。

## 伺服器開機

當 OpenBMC 用於伺服器中時，驅動系統開機的是
[obmc-host-start@.target](https://github.com/openbmc/phosphor-state-manager/blob/master/target_files/obmc-host-start%40.target)。

要啟動它，你會執行 `systemctl start obmc-host-start@0.target`。

如果你深入檢視它的 .requires 關係，你會看到檔案系統中有以下內容

```shell
ls -1 /lib/systemd/system/obmc-host-start@0.target.requires/
obmc-host-startmin@0.target
phosphor-reset-host-reboot-attempts@0.service
```

[obmc-host-startmin@.target](https://github.com/openbmc/phosphor-state-manager/blob/master/target_files/obmc-host-startmin%40.target)
代表啟動 host 所需的最少 service 與 target。這個 target 也會在 host 重啟的
情境中使用。這種區分 host-start 與 host-startmin target 的做法，讓使用者可以
把只應在 host 初次開機時執行（而在 host 重啟時不執行）的 service 放進
`obmc-host-start@.target`。例如，從上面的輸出你可以看到使用者只希望在全新的
host 開機嘗試時才執行 `phosphor-reset-host-reboot-attempts@0.service`。

接下來如果我們檢視 `obmc-host-startmin@0.target`，會看到：

```shell
ls -1 /lib/systemd/system/obmc-host-startmin@0.target.requires/
obmc-chassis-poweron@0.target
start_host@0.service
```

你可以看到在 `obmc-host-startmin@0.target` 中還有一個 target，即
`obmc-chassis-poweron@0.target`，以及一個名副其實命名為
`start_host@0.service` 的 service。

`obmc-chassis-poweron@0.target` 有相關聯的 service：

```shell
ls -1 /lib/systemd/system/obmc-chassis-poweron@0.target.requires/
op-power-start@0.service
op-wait-power-on@0.service
```

如果你執行 `systemctl start obmc-host-start@0.target`，systemd 就會開始執行
上述所有相關聯的 target 與 service。

這些 service 內部設有相依性，用來控制每個 service 的執行（例如，
op-power-start.service 會在 op-wait-power-on.service 之前執行）。這些相依性是
使用 target 以及 Wants、Before、After 關鍵字來設定的。

## 伺服器軟關機

軟關機功能封裝在 `obmc-host-shutdown@.target` 中。這個 target 之所以是「軟」
的，是因為它會通知 host 有降電請求，並給予它一段時間讓自身關機。

## 伺服器硬關機

硬關機封裝在 `obmc-chassis-hard-poweroff@.target` 中。這個 target 會在軟關機
service 正在執行時強制停止它，並立即切斷系統電源。

## 伺服器重啟

伺服器重啟封裝在 `obmc-host-reboot@.target` 中。這個 target 會使用軟關機
target，然後在該步驟完成後啟動 host 開機 target。

## 伺服器靜止（Quiesce）

`obmc-host-quiesce@.target` 用於 host 錯誤情境。當進入
`obmc-host-quiesce@0.target` 時，它會把 host 狀態的 D-Bus
[物件](https://github.com/openbmc/phosphor-dbus-interfaces/blob/master/yaml/xyz/openbmc_project/State/Host.interface.yaml)
置於 `Quiesced` 狀態。

## 因錯誤導致的伺服器緊急關機

`obmc-chassis-emergency-poweroff@.target` 是包在
`obmc-chassis-hard-poweroff@.target` 與 `obmc-host-quiesce@.target` 外層的一個
wrapper target。它由那些偵測到需要立即關閉系統的嚴重散熱或電源錯誤的應用程式
使用。它會關閉機箱，並把 host 置於 Quiesce 狀態（如果 host 正在執行）。關機
路徑中的某些非關鍵 service 可能會與這個 target 衝突，以確保在此路徑中只執行
最關鍵的 service。

如果啟動了這個 target，就不會進行自動錯誤復原（也就是 host 重啟）。需要使用者
介入才能離開此狀態。使用者可以依需求請求開機，或是在想脫離 quiesce 時請求 host
停止／降電。

## OpenBMC 中的 Systemd 控制

OpenBMC 中有一組 service 會與 systemd 及其 unit 檔案互動，為 OpenBMC 系統及
systemd 的使用者提供某種程度的抽象層。關於此功能的更多資訊請參閱
[state](https://github.com/openbmc/phosphor-dbus-interfaces/tree/master/yaml/xyz/openbmc_project/State)
介面。

例如，如果你想執行伺服器開機功能，你會這樣做：

> busctl set-property xyz.openbmc_project.State.Host
> /xyz/openbmc_project/state/host0 xyz.openbmc_project.State.Host
> RequestedHostTransition s xyz.openbmc_project.State.Host.Transition.On

在底層，這其實就是在用伺服器開機 target 呼叫 systemd。

## Systemd Service 或監控應用程式

建立新的 OpenBMC 應用程式時，一個常見的問題是：當這些應用程式需要在 systemd
target 的脈絡下執行某些邏輯時，究竟應該由 systemd service 觸發，還是應該監控
適當的 D-Bus 訊號，以得知它們所關注 target 的啟動／停止。

關於何時該建立 systemd service 的基本準則如下：

- 如果你的應用程式邏輯相依於其他基於 systemd 的 service，那麼就把它做成
  systemd service，並使用 Wants/After/Before 的 service 功能。
- 如果其他應用程式相依於你的應用程式邏輯，那麼它就應該是一個 systemd service。
- 如果你的應用程式在 target 啟動期間失敗，可能會影響在其之後執行的 target 或
  service，那麼它就應該是一個 systemd service。這能確保在你的應用程式失敗時，
  相依的 target 不會被啟動。

## Systemd 的錯誤處理

target 與 service 的數量龐大，也帶來了極高的失敗機率。為了讓 OpenBMC 成為一個
穩健且高效能的系統，它必須針對 service 及其 target 失敗時設有一套錯誤處理策略。

當失敗發生時，OpenBMC 軟體需要通知系統的使用者，並提供機制讓系統自動
重試失敗的操作（也就是重啟系統），或是維持在靜止（quiesced）狀態，以便收集錯誤
資料並調查失敗原因。

就 OpenBMC 與 systemd 的使用而言，有兩種主要的失敗情境：

1. target 內的某個 service 失敗
   - 如果該 service 是 "oneshot" 類型，且被 target 標為 required（而非 wanted），
     那麼當 service 失敗時 target 就會失敗 - 使用 "OnFailure" 選項定義 target
     失敗時的行為（例如，只要有任一 required service 失敗就進入新的失敗 target）
   - 如果該 service 不是 "oneshot"，那麼它就不能讓 target 失敗（target 只知道它
     成功啟動了）- 為 service 失敗時定義一個（OnFailure）行為選項。- 該 service
     不能有 "RemainAfterExit=yes"，否則 OnFailure 動作要等到 service 停止時（而不是
     失敗時）才會發生 - \*關於 [RemainAfterExit](#remainafterexit)，請參閱下方的
     更多資訊

2. 在一般 systemd target/service 之外發生的失敗（host 看門狗逾時、偵測到 host
   checkstop）
   - 偵測到此失敗的 service 負責記錄適當的錯誤，並指示 systemd 進入適當的 target

在 OpenBMC 中有一個 host quiesce target。這是其他與 host 相關的 target 在遭遇
失敗時應該進入的 target。OpenBMC 中的其他軟體接著可以監控是否進入了這個 quiesce
target，並處理「停止」與「自動重啟」的功能。

非 host 相關的 target，在其錯誤處理方面需要特別考量。例如，負責施加機箱電源的
target `obmc-chassis-poweron@0.target`，會有一條
`OnFailure=obmc-chassis-poweroff@%i.target` 錯誤路徑。也就是說，如果機箱開機
target 失敗，就關閉機箱電源。

以上的資訊為我們與 host 相關的 target 與 service 建立了若干**準則**：

- 所有 target 都應該有一個 `OnFailure=obmc-quiesce-host@.target`
- 所有為了讓 target 達成其功能而必需的 service，都應該透過 RequiredBy 掛到該
  target 上（而不是 WantedBy）
- 所有 service 都應該先設法做成 "Type=oneshot"，如此我們就能只依賴 target 的
  失敗路徑
- 如果某個 service 不能是 "Type=oneshot"，那麼它需要有一個
  `OnFailure=obmc-quiesce-host@.target`，並且最好設定 "RemainAfterExit=no"
  （但請參閱下方對此的注意事項）
- 如果某個 service 以上皆非，那麼就由該 service 應用程式自行負責在失敗時用
  `obmc-quiesce-host@.target` 來呼叫 systemd

### RemainAfterExit

多數 OpenBMC service 都將它設為 "yes"，以處理有人對同一個 target 啟動兩次的情況。
如果與該 target 關聯的 service 未在執行（也就是 RemainAfterExit=no），那麼該
service 會被再次執行。想像有人不小心把 `obmc-chassis-poweron@.target` 執行了兩次。
如果你在作業系統已經啟動並運行的情況下再次執行它，而那個切換 pgood 腳位的 service
被重新執行，你就會讓系統當機。基於這個理由，目標應該始終是撰寫 RemainAfterExit
設為 yes 的 "oneshot" service。

## Target 與 Service 相依性細節

市面上有一些工具可以視覺化 systemd 的 service 與 target 相依性
（systemd-analyze），但由於我們設計的複雜度，它們無法產生什麼有用的東西。

目前，先在此記錄一個 witherspoon 系統上的現行相依性以供參考。

```text
R = Requires
W = Wants
A = After
B = Before
S = Start (runs a command to start another target or service)
(S) = Synchronization Target
```

### 軟關機

```text
obmc-host-shutdown.target
  R: xyz.openbmc_project.Ipmi.Internal.SoftPowerOff.service
     W: obmc-host-stopping.target (S)
     B: obmc-host-stopping.target (S)
  R: obmc-chassis-poweroff.target
     R: obmc-host-stop.target
        R: op-occ-disable.service
           B: obmc-host-stop-pre.target
     R: op-power-stop.service
        W: obmc-power-stop.target (S)
        B: obmc-power-stop.target (S)
        W: obmc-power-stop-pre.target (S)
        A: obmc-power-stop-pre.target (S)
        W: mapper-wait@-org-openbmc-control-power.service
        A: mapper-wait@-org-openbmc-control-power.service
     R: op-wait-power-off.service
        B: obmc-power-off.target (S)
        W: obmc-power-stop.target (S)
        B: obmc-power-stop.target (S)
        W: obmc-power-stop-pre.target (S)
        A: obmc-power-stop-pre.target (S)
        W: mapper-wait@-org-openbmc-control-power.service
        A: mapper-wait@-org-openbmc-control-power.service
     R: op-powered-off.service
        A: op-wait-power-off.service
        R: op-wait-power-off.service
        S: obmc-chassis-powered-off.target
     W: pcie-poweroff.service
        B: op-power-stop.service
        A: obmc-power-stop-pre@.target
```

#### 同步 Target 相依性

```text
obmc-power-stop.target
  W: obmc-power-stop-pre.target
  A: obmc-power-stop-pre.target

obmc-power-stop-pre.target
  W: obmc-host-stopped.target
  A: obmc-host-stopped.target

obmc-host-stopped.target
  W: obmc-host-stopping.target
  A: obmc-host-stopping.target
  B: obmc-power-stop-pre.target

obmc-host-stopping.target
  W: obmc-host-stop-pre.target
  A: obmc-host-stop-pre.target
  B: obmc-host-stopped.target

obmc-host-stop-pre.target
  B: obmc-host-stopping.target
```
