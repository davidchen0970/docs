# OpenBMC 伺服器電源回復

作者：Andrew Geissler (geissonator)

其他貢獻者：

建立日期：2021 年 10 月 11 日

## 問題描述

現代電腦系統具備一項自動開機回復（automated power-on recovery）功能，簡言之，
就是讓系統知道在發生電源問題時該該怎麼做。如果系統遭遇停電（black out，也就是
系統電源被完全切斷），它應該要自動開機嗎？應該保持關閉嗎？或者使用者可能希望
系統回到斷電前的那個狀態。

也有一些情況是使用者不希望自動電源回復發生的。舉例來說，有些系統配有操作面板
（op-panel），而這些操作面板上可能有一個針孔式重設孔（pin hole reset）。這是一種
手動機制，讓使用者在 BMC 當死或沒有回應時強制對 BMC 進行硬重設。在這些情況下，
使用者可能希望系統不要自動開機，因為他們想要除錯 BMC 錯誤的原因。

在停電場景中，系統擁有者可能有一組需要在電力恢復後執行的服務。舉例來說，IBM
要求在停電時把所有 LED 切換為關閉。OpenBMC 需要提供的機制，讓系統擁有者能在這種
場景下執行服務。

降壓（brownout）是另一種常會運用自動開機回復功能的場景。降壓是指 BMC 韌體偵測到
（或被告知）機箱電力已無法繼續供應，但 BMC 的電力仍會維持的一種情況。在某些系統
上，會希望利用自動開機功能，在降壓情況一結束就立刻把機箱電力重新開啟。

有些系統擁有者可能會選擇為其系統接上不斷電系統（Uninterrupted Power Supply，UPS）。
UPS 能在停電或降壓場景中持續為系統供電。UPS 的電力有限，因此它的主要用途是應對
短暫的電力中斷，或是讓主機韌體能夠有秩序地關機。

本設計文件的目標，是說明 OpenBMC 韌體將如何處理這些問題。

## 背景與參考

BMC 在此領域已經實作了一小部分功能。位於 phosphor-dbus-interface 中的
[PowerRestorePolicy][pdi-restore] 屬性定義了此功能能力。

在較小型的伺服器中，此功能通常位於進階設定與電源介面（Advanced Configuration and
Power Interface，ACPI）之內。

[openbmc/phosphor-state-manager][state-mgr] 支援 phosphor-dbus-interface 中所定義的
這個屬性。

## 需求

### 自動開機回復

OpenBMC 軟體必須確保它會持久化儲存機箱電力的狀態，以便在需要時知道要回復到哪種
狀態

OpenBMC 軟體必須支援以下選項：

- 當系統電力流失時不做任何事（這將是預設值）
- 一律開啟系統電源並開機主機
- 一律關閉系統電源（之前電源是開啟的，現在電源關閉，執行所有機箱關機服務以確保
  軟體與硬體處於乾淨的狀態）
- 回復機箱電力與主機的先前狀態

這些選項只會在 BMC 於重新啟動後並未偵測到機箱電力已經開啟的情況下才會被檢查與
套用。

OpenBMC 軟體也必須支援一次性（one_time）電源回復策略的概念。這是 `PowerRestorePolicy`
的一個獨立實例，會掛在一個以 "one_time" 結尾的 D-Bus 物件路徑之下。如果這個 one_time
設定不是預設值 `None`，那麼軟體就會執行其所定義的策略，然後把 one_time 屬性重設為
`None`。這個 one_time 功能是讓軟體在 firmware update 等其他領域也能運用自動開機回復
功能的一種方式，例如在更新完成後希望有某種特定的開機行為。

### BMC 與系統回復路徑

在 BMC 或系統進入不良狀態、且使用者已發起某種可被 BMC 偵測為使用者發起的手動重設
的情況下，BMC 軟體必須：

- 在 [BMC state interface][bmc-state] 中填入適當的 `RebootCause`
  - 至少會新增 `PinholeReset`，其他原因可依需要新增
- 記錄一筆錯誤，指出曾發生使用者發起的強制重設
- 若針孔重設之前機箱電力是開啟的，則不記錄一筆指出發生停電的錯誤
- 不對系統套用任何電源回復策略
- 一旦 BMC 正常重新啟動後，就把電源回復重新開啟

### 停電（Blackout）

當系統的 AC 電源被切斷時就會發生停電，若沒有安裝 UPS 來維持系統運作，將導致電力
完全流失。為了在 BMC 重新啟動後辨識這個場景，chassis-state-manager 會檢查斷電前最後
一次的電源狀態，並與 pgood 腳位進行比較。停電可能是由使用者刻意觸發的（例如針孔
重設），或在嚴重情況下因某種外部供電中斷而發生。無論哪種情況，BMC 都必須把這個
有害狀態納入考量。當這個情況發生時，BMC 可能會（依設定而定）：

- 提供一個通用目標 `obmc-chassis-blackout@.target`，在偵測到停電時被呼叫
- 遵循目前的電源回復策略

BMC 韌體也必須能夠：

- 找出系統為何處於停電情況。可能是電力流失或使用者操作所致。

### 降壓（Brownout）

如前所述，降壓情況是指無法繼續供應 AC 電力給機箱，但 BMC 仍能持續取得電力並運作。

當這個情況發生時，BMC 必須：

- 依情況所需盡快地關閉系統電源（或在電力毫無預警地流失時優雅地處理斷電）
- 記錄一筆錯誤，指出降壓事件已發生
- 支援讓主機韌體在降壓完成時可指定一次性電源回復策略（如果他們希望如此）
- 辨識降壓情況何時完成
- 等待降壓完成並套用一次性的電源回復策略。如果未定義任何一次性策略，則執行為該系統
  所定義的標準電源回復策略

BMC 韌體也必須能夠：

- 判斷系統是否處於降壓情況
  - 在 BMC 首次啟動時執行，以知道是否應該套用任何自動開機回復
- 當降壓正在發生時不執行任何開機回復邏輯
- 當是由主機韌體開機時，告知該主機韌體這是一次由自動開機回復所引發的開機

### 不斷電系統（UPS）

當有 UPS 存在且發生停電或降壓情況時，BMC 必須：

- 記錄一筆錯誤以指出該情況已發生
- 若主機韌體正在執行，則通知主機韌體此市電失效（utility failure）情況（此行為可在
  build-time 設定）
- 若 UPS 電池電量偏低且主機韌體正在執行，則通知主機韌體此情況，指出需要快速關機
  （此行為可在 build-time 設定）
- 若 UPS 電池電量偏低且整個系統的電力流失即將發生（亦即 BMC 也將斷電且 UPS 即將
  耗盡電力的停電場景），則記錄一筆錯誤
- 不執行任何自動開機回復邏輯，以避免開機/關機反覆抖動（此行為可在 build-time 設定）

## 建議設計

### 自動開機回復

會在機箱與主機狀態已判定之後執行一個應用程式，且只有在機箱電源未開啟時才會執行。

這個應用程式會尋找 one_time 設定，若其值不是 `None` 就使用它。如果它確實使用了
one_time 設定，那麼在讀取之後就會把它重設為 `None`。否則這個應用程式會讀取
`PowerRestorePolicy` 的持久化值。接著這個應用程式會依照上文「需求」中所定義的邏輯
執行。

此功能將掛在 phosphor-state-manger 中，可能也會有 x86-power-control。

### BMC 與系統回復路徑

BMC state manager 應用程式目前會檢查 sysfs 中的一個檔案，試圖判斷 BMC 重新啟動的
原因。然後它會把這個原因放入 `RebootCause` 屬性中。

BMC 重設的可能原因之一是外部重設（EXTRST）。外部重設可能因各種原因發生。有些系統
會新增 GPIO，以提供這類重設的額外細節。

將在 [device-tree-gpio-naming.md][dev-tree] 中新增一個新的 GPIO 名稱，用以回報在 BMC
上一次重新啟動時是否發生過針孔重設。BMC state manager 應用程式會強化其對
`RebootCause` 的支援，去尋找這個 GPIO，若存在則讀取它，並在無法透過 sysfs 判定重新
啟動原因、或 sysfs 回報 EXTRST 原因時（此時會利用 GPIO 來強化重設原因）據以設定
`RebootCause`。

如果電源回復軟體在 `RebootCause` 中看到 `PinholeReset` 原因，那麼它就不會套用其任何
策略。日後並非由針孔重設引起的 BMC 重新啟動，會讓 `RebootCause` 回到預設值，因此
在該次 BMC 開機時電源回復策略會重新啟用。

phosphor-state-manager 的 chassis 軟體在看到 `PinholeReset` 原因（或任何指出使用者
已發起系統重設的其他原因）時，不會記錄停電錯誤。

### 停電（Blackout）

應新增一個新的 systemd 目標 `obmc-chassis-blackout.target`，讓系統維護者能在此情況下
呼叫服務。這個新目標會在 BMC 偵測到停電時被呼叫。此目標讓系統擁有者能把他們自己
的特定服務新增到這個新目標。Phosphor-chassis-state-manager 會確保在停電之後會呼叫
`obmc-chassis-blackout.target`。

### 降壓（Brownout）

現有的 `xyz.openbmc_project.State.Chassis` 介面將被強化，以支援一個 `CurrentPowerStatus`
屬性。現有的 phosphor-chassis-state-manager（在系統中針對每個機箱實例各建立一個）
會支援對這個屬性的讀取。以下是目標機箱電源狀態可能回傳的值：

- `Undefined`
- `BrownOut`
- `UninterruptiblePowerSupply`
- `Good`

phosphor-power 函式庫中的 phosphor-psu-monitor 應用程式將負責監控降壓情況。它會支援
一個 per-chassis 介面，代表進入目標機箱的電力狀態。這個介面是通用的，其他應用程式
也可以掛載它以回報電力狀態。state-manager 軟體會利用 mapper 去尋找其機箱對應此介面的
所有實作，並彙整狀態（亦即只要有任何一個回報降壓，就會回傳 `BrownOut`）。這個介面將
在本文件的後續更新中定義。

負責偵測與回報機箱電力的應用程式會在啟動時執行，並判定其屬性正確的狀態。這些
應用程式會在降壓發生時記錄一筆錯誤，並發起快速關機。

如果系統設計有需要，則會使用 phosphor-state-manager 所提供的現有自動開機策略一次性
功能，供降壓完成時使用。

當 phosphor-power 應用程式偵測到降壓情況已完成時，它會把代表電源狀態的介面重設為
good，並啟動執行自動開機邏輯的 state-manager 服務。

phosphor-state-manager 會確保自動開機回復邏輯只在電源供應介面回報電源狀態為 good
時才執行。如果系統中有多個機箱及/或主機實例，那麼只有與電源狀態不良的機箱所關聯的
主機實例會被禁止開機。

### 不斷電系統（UPS）

將定義一個新的 phosphor-dbus-interface 來代表 UPS。一個 BMC 應用程式會針對連接到系統的
每個 UPS 各實作一個。這個應用程式會監控 UPS 狀態並監控以下情況：

- UPS 市電失效（系統電力已失效且 UPS 正在供應系統電力）
- UPS 電池電量偏低（UPS 即將耗盡電力）

如果應用程式發現電力已流失且系統正由 UPS 電池供電，那麼它會監控 UPS 中剩餘的電力，
並在需要時通知主機必須關機。這個應用程式也將負責記錄一筆錯誤，指出已切換至 UPS
備用電力，並在系統無法繼續維持開啟時於其介面中設定適當的屬性，以指出此情況存在。
phosphor-state-manager 會向 mapper 查詢這個新 UPS 介面的實作，並在決定其
`CurrentPowerStatus` 要回傳的值時，將其與電源供應降壓狀態一併運用。

與上述降壓場景類似，phosphor-state-manager 會確保在 `PowerStatus` 未設定為 `Good` 時
不執行自動開機回復邏輯。此行為可在 phosphor-state-manager 中於 build-time 設定。

## 已考慮的替代方案

無，這是一項非常基本的功能，沒有太多替代方案（除了乾脆不做之外）。

## 影響

無

## 測試

此策略的控制已可透過 Redfish API 設定。

```bash
#  Power Restore Policy
curl -k -H "Content-Type: application/json" -X PATCH -d '{"PowerRestorePolicy":"AlwaysOn"}' https://${bmc}/redfish/v1/Systems/system
curl -k -H "Content-Type: application/json" -X PATCH -d '{"PowerRestorePolicy":"AlwaysOff"}' https://${bmc}/redfish/v1/Systems/system
curl -k -H "Content-Type: application/json" -X PATCH -d '{"PowerRestorePolicy":"LastState"}' https://${bmc}/redfish/v1/Systems/system
```

在測試時，每個策略都應該被設定並驗證。也應該針對 one_time 層面檢查每個可能的值，
並驗證它只會被使用一次。

驗證當發生多次停電時，在策略為 `AlwaysOn` 或 `Restore` 時，韌體會持續嘗試開啟系統
電源。

在支援的系統上，應該在一個策略設定為一律開機的系統上執行針孔重設。測試人員應驗證
系統在針孔重設後不會自動開機。驗證當對 BMC 進行正常重新啟動時它會自動開機。

應該對系統注入一個降壓情況，並驗證適當的路徑：

- 產生錯誤記錄
- 通知主機（若在執行且可通知）
- 系統被快速關閉電源
- 當降壓存在時不執行電源回復功能
- 當降壓情況結束時系統會自動重新開機（假設一次性或系統自動開機回復策略為
  `AlwaysOn` 或 `Restore`）

將 UPS 接到系統上，並確保在系統電力被切斷時會記錄一筆錯誤，且主機會被通知並被允許
關機。

[pdi-restore]:
  https://github.com/openbmc/phosphor-dbus-interfaces/blob/master/yaml/xyz/openbmc_project/Control/Power/RestorePolicy.interface.yaml
[state-mgr]: https://github.com/openbmc/phosphor-state-manager
[bmc-state]:
  https://github.com/openbmc/phosphor-dbus-interfaces/blob/master/yaml/xyz/openbmc_project/State/BMC.interface.yaml
[dev-tree]:
  https://github.com/openbmc/docs/blob/master/designs/device-tree-gpio-naming.md
