# 在使用 OpenBMC 的伺服器上發送 NMI 的設計提案

作者：Lakshminarayana Kammath

其他貢獻者：Jayanth Othayoth

建立日期：2019-05-21

## 問題描述

目前，使用 OpenBMC 的伺服器無法在 host 無回應或當機時擷取相關的除錯
資料。這些系統需要能夠診斷當機的根因，並在收集除錯資料的同時執行復原。

## 背景與參考資料

在客戶端／實驗室有發生 host 失去回應而導致系統當機的情形
（<https://github.com/ibm-openbmc/dev/issues/457>）。這代表在 host 處於
當機狀態時，完全無法找出問題出在哪裡，只能在沒有任何相關除錯資料的情況下
復原系統。

無論 host 是無回應或在執行中，管理員都需要能夠觸發 NMI 事件，進而觸發
一個與架構相關的程序，對系統上所有可用的處理器發出中斷。

## 針對使用 OpenBMC 伺服器的提案設計

本提案旨在觸發 NMI，進而呼叫一個與架構相關的程序，該程序會先啟用資料
收集，接著復原 Host。這將讓 Host／OS 開發團隊得以分析並修復任何出現
host 當機與系統無回應的問題。

### D-Bus

在 control.host 命名空間中引入新的 D-Bus 介面
（/openbmc/phosphor-dbus-interfaces/xyz/openbmc_project/Control/Host/
NMI.interface.yaml），並針對各處理器專屬的目標實作新的 D-Bus 後端。

### BMC 支援

啟用 NMI D-Bus phosphor 介面，並透過 Redfish 提供支援。

### 使用的 Redfish Schema

- 參考：DSP2046 2018.3，
- ComputerSystem 1.6.0 schema 提供了一個名為 #ComputerSystem.Reset 的
  動作，此動作用於重設系統。ResetType 參數用來指出要執行的重設類型。
  在本案中，我們可以使用 NMI 類型
  - Nmi：產生一個診斷中斷（在 x86 系統上通常是 NMI），以停止正常運作、
    執行診斷動作，並通常會使系統暫停。

## 高階流程

1. Host／OS 當機或無回應，或者需要擷取 kernel dump 以除錯某些錯誤情況。
2. 管理員／使用者可以使用允許 POST 作業的 Redfish URI ComputerSystem.Reset，
   並將 Action 與 ResetType 屬性變更為
   {"Action":"ComputerSystem.Reset","ResetType":"Nmi"} 以觸發 NMI。
3. Redfish URI 會呼叫 D-Bus NMI 後端呼叫，後端會使用
   xyz.openbmc_project.Control.Host.NMI 與架構相關的後端實作，對系統上
   所有處理器觸發 NMI。
4. 收到 NMI 後，host 會自動呼叫與架構相關的動作。其中一種動作可以是
   先呼叫 kdump，接著重新開機。

- 注意：NMI 可以在任何狀態下傳送至 host，不限於無回應的狀態。

## 曾考慮的替代方案

擴充現有的 D-Bus 介面 state.Host 命名空間
（/openbmc/phosphor-dbus-interfaces/xyz/openbmc_project/State/Host.interface.yaml），
以支援一個名為 Nmi 的新 RequestedHostTransition 屬性。D-Bus 後端可以在內部
呼叫處理器專屬的目標來觸發 NMI 並執行相關動作。

有強力的理由要放棄上述做法。phosphor-state-manager 一直以來都專注於 BMC、
Chassis 與 Host 的「狀態」，而 NMI 比較像是對 host 的一種「動作」，而不是一種
狀態。

## 影響

此實作只需要在 NMI 啟動時對系統狀態做些改變，不論 host OS 處於何種狀態都一樣，
因此對系統其他部分的影響極小。

## 測試

根據平台硬體設計，本測試需要一個 host OS kernel 模組驅動程式來製造
hard lockup／當機，然後檢查該情境是否正常。此外，也可以呼叫 NMI 以取得
crash dump，並透過 console 記錄確認 HOST 已收到 NMI。
