# BMC 的 ECC Error SEL

Author: Will Liang

Created: 2019-02-26

## 問題描述

IPMI SEL 僅針對主機（host）記憶體定義了 Error Correction Code (ECC) 錯誤，
並未涵蓋 BMC 本身的記憶體。

本提案的目標是將來自 BMC 的 ECC 事件記錄到 IPMI System Event Log (SEL) 中。
每當發生 ECC 時，BMC 便會產生一個帶有適當資訊的事件，並將其加入 SEL。

## 背景與參考

IPMI 規格定義了關於 ECC/其他可修正錯誤，以及 ECC/其他不可修正錯誤的記憶體
system event log，以及 ECC/其他可修正記憶體錯誤的記錄上限是否已達到的相關
內容。[1] BMC ECC SEL 將遵循 IPMI SEL 格式，建立 BMC 記憶體 ECC 事件記錄。

OpenBMC 目前支援透過解析 D-Bus event log 來產生 SEL 條目。但 OpenBMC 專案
尚未支援 BMC ECC SEL 此功能。因此，記憶體 ECC 資訊將註冊到 D-Bus，並同
樣產生記憶體 ECC SEL。

[[1]Intelligent Platform Management Interface Specification v2.0 rev 1.1, section 41](https://www.intel.com/content/www/us/en/servers/ipmi/ipmi-second-gen-interface-spec-v2-rev1-1.html)

## 需求

目前 OpenBMC 專案不支援 D-Bus 中的 ECC 事件記錄，因為 OpenBMC 的 D-Bus
架構中沒有相關的 ECC 資訊。新的 ECC D-Bus 資訊將加入 OpenBMC 專案，並建立
一個 ECC 監控服務，從 EDAC 驅動程式取得 ECC 計數（ce_count/ue_count）。
同時必須確保 EDAC 驅動程式已載入，且 ECC/其他可修正或 ECC/其他不可修正的
計數需要從 EDAC 驅動程式取得。

## 建議設計

具備 ECC 功能的記憶體控制器可以在允許偵測與修正記憶體錯誤的作業系統（例如
某些版本的 Linux、macOS 與 Windows）中偵測並修正錯誤，這有助於在問題惡化成
災難性的記憶體模組故障之前先識別出來。

許多 ECC 記憶體系統在 CPU 與記憶體之間使用一顆「外部」的 EDAC 來修正記憶體
錯誤。多數主機則將 EDAC 整合進 CPU 的內建記憶體控制器中。

根據 IPMI 規格的 Section 42.2、Table 42 [2]，這些 SEL sensor type 將被定義為
`Memory`，而 `Event Data 3` 欄位可用來提供事件延伸資訊。因此，BMC ECC 事件
將 "Event Data 3" 設為值 FEh，以辨識這是 BMC ECC 錯誤。

[[2] Intelligent Platform Management Interface Specification v2.0 rev 1.1, section 42.2](https://www.intel.com/content/www/us/en/servers/ipmi/ipmi-second-gen-interface-spec-v2-rev1-1.html)

此功能的主要目的，是提供 BMC 記錄 ECC 錯誤 SEL 的能力。

本設計有兩個新的應用程式：

- 輪詢（poll）ECC 錯誤計數
- 建立 ECC SEL

此外也設計了一種機制來限制日誌的「最大數量」，以避免產生大量可修正 ECC 的
日誌。當達到 `maximum quantity` 時，ECC 服務將停止記錄 ECC 日誌。
`maximum quantity`（預設值：100）存放於設定檔中，使用者可在必要時修改此
值。

### phosphor-ecc.service

這會在服務啟動後持續執行該應用程式，並每秒查詢一次 ECC 錯誤計數。首次啟動
時，它會將 EDAC 驅動程式中所有可修正 ECC 計數與不可修正 ECC 計數重設。

它同時在 D-Bus 上提供以下路徑：

- bus name : `xyz.openbmc_project.Memory.ECC`
- object path : `/xyz/openbmc_project/metrics/memory/BmcECC`
- interface : `xyz.openbmc_project.Memory.MemoryECC`

此介面具有以下屬性： | Property | Type | Description | |
-------- | ---- | ----------- | | isLoggingLimitReached | bool | ECC 記錄
已達上限| | ceCount| int64 | 可修正 ECC 事件 | | ueCount| int64 |
不可修正 ECC 事件 | | state| string | bmc ECC 事件狀態 |

`xyz::openbmc_project::Memory::Ecc::Error::ceCount` 與 `ueCount` 以及
`isLoggingLimitReached` 的錯誤類型將被建立，用以產生 ECC 日誌的錯誤類型。

### 建立 ECC SEL

使用 `phosphor-sel-logger` 套件，以 BMC SEL 格式記錄以下日誌。

- correctable ECC log：當從 EDAC 驅動程式參數取得 `ce_count`，且該計數超過
  先前的計數時。
- uncorrectable ECC log：當從 EDAC 驅動程式參數取得 `ue_count`，且該計數超過
  先前計數時。
- logging limit reached log：當可修正 ECC 日誌達到 `maximum quantity` 時。

## 曾考慮的替代方案

另一個考量是完全不採用停止記錄 ECC 的機制。只要檢查 `ce_count` 且其值超過
先前的值，就記錄一筆 ECC 日誌。但這樣會遇到大量的 ECC 日誌，同時也會佔用
BMC 的記憶體。

## 影響

此應用程式的實作只需要在建立 event log 時做一些修改，因此對系統其餘部分的
影響極小。

## 測試

依平台硬體設計而定，此測試需要一個 ECC 驅動程式來製造假造的 ECC 錯誤，接著
檢查該情境是否正確。
