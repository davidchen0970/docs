# 硬體錯誤時中止開機

作者：Andrew Geissler (geissonator)

其他貢獻者：

建立日期：2020 年 2 月 20 日 更新日期：2022 年 4 月 12 日

## 問題說明

有些團隊，例如製造團隊，會要求 BMC 韌體在產生指認（call out）某項硬體元件的錯誤記錄時，中止系統運作。這樣做的原因是確保只要有硬體問題，就不會將系統出貨給客戶。它也能確保一旦發現錯誤，就能立即識別出來，並停止所有動作直到問題修復為止。如果系統在離開製造階段出貨之後才發生硬體問題，則 BMC 韌體的行為應該是回報錯誤，但允許系統繼續開機並運作。

OpenBMC 韌體需要一個機制來支援這個使用案例。

## 背景與參考

在 IBM 內部，這個功能是由稱為「製造旗標」（manufacturing flags）的設定來啟用／停用。這些旗標是由使用者在 registry 變數中設定的位元，韌體會接著查詢這些變數。這些 registry 變數只能由對系統具有 admin 權限的人員設定。這些旗標不會在製造與測試之外使用。

phosphor-logging 內的擴充元件（extensions）可能會處理並非總是透過標準 phosphor-logging 介面傳入的記錄（例如主機（host）傳下來的記錄）。在這些情況下，如果那些記錄包含硬體指認（hardware callouts），系統仍必須中止。

一封關於這個主題的 [郵件][1] 討論串已寄到郵件清單。

## 需求

- 提供一個機制，讓 OpenBMC 韌體在產生帶有 inventory callout 的 phosphor-logging 記錄時中止系統運作
  - 啟用／停用這個功能的機制不需要是外部 API（例如 Redfish）。它可以只是一個在 ssh 登入 BMC 後執行的 busctl 指令
  - 中止發生時，必須讓使用者明顯察覺
    - 造成中止的那筆記錄必須可以被識別
  - 中止必須只停止遇到該錯誤的 chassis／host 實例
  - 中止必須讓 host 韌體有機會優雅地（gracefully）自行關機
  - 中止必須停止與該錯誤關聯的 host（執行 `obmc-host-stop@X.target`），並設法讓系統停留在失敗狀態（例如若 chassis 電源原本開啟，則維持開啟）
  - 在造成中止的那筆記錄被解決或刪除之前，不允許該 chassis／host 實例對（pair）開機
    - 重設 BMC 會清除這個開機阻止
- 確保這個用於在 inventory callout 時中止韌體的機制，也能被 phosphor-logging 擴充元件利用，以針對其他原因中止韌體
  - 這些原因將在各擴充元件的文件中定義
- 在這種失敗期間讓相關聯的 host 進入靜止狀態（quiesce）

**特別說明：** 初期，相關聯的 host 與 chassis 會硬編碼（hard coded）為 chassis0 與 host0。整個 BMC 堆疊還需要更多工作才能處理多個 chassis 與 host。本設計允許此類功能在稍後再啟用。

## 建議設計

建立一個 [phosphor-settingsd][2] 設定，`xyz.openbmc_project.Logging.Settings`。在其內建立一個名為 QuiesceOnHwError 的布林屬性。這個屬性將由 xyz.openbmc_project.Settings 服務提供（host）。

定義一個新的 D-Bus 介面，用來表示已產生一筆會阻止某個 chassis／host 實例開機的錯誤：
`xyz.openbmc_project.Logging.ErrorBlocksTransition`

這個介面將由一個基於實例（instance based）的 D-Bus 物件提供：
`/xyz/openbmc_project/logging/blockX`，其中 X 是被阻止的 chassis／host 實例對的實例編號。

當透過 phosphor-logging 介面產生錯誤時，軟體會檢查該錯誤是否有 callout，若有則會檢查新的
`xyz.openbmc_project.Logging.Settings.QuiesceOnHwError`。如果此值為 true，則 phosphor-logging 會在
`/xyz/openbmc_project/logging/blockX` 建立一個 D-Bus 物件，其下帶有 `xyz.openbmc_project.Logging.ErrorBlocksTransition`
介面。並會在這筆記錄與這個新 D-Bus 物件之間建立一個 mapper [association][3]。對應的 host 實例將由
phosphor-logging 使其進入靜止狀態（quiesce）。

可以透過重啟 BMC 或清除造成阻止的那筆記錄來脫離阻止狀態。其他系統專屬的政策可以放在適當的 targets 中（例如，若 chassis 關機應該要清除阻止）。

關於 callouts 的更多資訊，請參閱 phosphor-logging 的 [callout][4] 設計。

將啟動一個新的 `obmc-host-graceful-quiesce@.target` systemd target。這個新的 target 會確保對 host 發起優雅關機，接著啟動 `obmc-host-quiesce@.target`，後者會停止 host 並將 host 狀態移至 Quiesced。

obmcutil 將被增強，以尋找這些 block 介面，並在 block 啟用時透過 `obmcutil state` 指令通知使用者，以及哪個記錄與之關聯。

目標是當未來的設計工作允許開發者將某些錯誤記錄與「在記錄被處理前中止系統」建立關聯時，能基於這個概念進一步建構。

## Host 錯誤

在某些情境下，當 host 將某些錯誤傳給 BMC 時，也希望能中止開機並阻止其重新開機。

這些錯誤可能是 SEL 格式，也可能是 OEM 專屬格式，例如 IBM 使用的 [PEL 格式][5]。

phosphor-logging 內用於處理硬體 callout 情境的介面，可以重新用途（repurposed）應用於這個使用案例。

## 已考量的替代方案

目前這個功能是 phosphor-logging 基礎設計的一部分。如果除了 IBM 之外沒有人認為它有價值，我們可以將它併入 phosphor-logging 中 PEL 專屬的部分。

## 影響

這將需要對回報的記錄進行一些額外檢查，但應該只有極小的額外負擔（overhead）。

除非使用者開啟這個新設定，否則不會改變系統行為。

## 測試

將執行單元測試（unit tests），以確保偵測記錄中錯誤的邏輯，並驗證新設定的兩種可能值。

測試案例將需要尋找這個新的阻止用 D-Bus 物件並適當處理。

[1]: https://lists.ozlabs.org/pipermail/openbmc/2020-February/020575.html
[2]: https://github.com/openbmc/phosphor-settingsd
[3]:
  https://github.com/openbmc/docs/blob/master/architecture/object-mapper.md#associations
[4]:
  https://github.com/openbmc/phosphor-dbus-interfaces/blob/master/yaml/xyz/openbmc_project/Common/Callout/README.md
[5]:
  https://github.com/openbmc/phosphor-logging/blob/master/extensions/openpower-pels/README.md
