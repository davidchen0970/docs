# POWER Systems 上的記憶體保留重啟與 System Dump 擷取流程

作者：Dhruvaraj S <dhruvaraj@in.ibm.com>

建立日期：11/06/2019

## 問題說明

在基於 POWER 的伺服器上，由 hypervisor 韌體負責管理資源，並分配給執行於
伺服器上的邏輯分割區（logical partitions）。若此 hypervisor 發生錯誤而無法
繼續執行管理工作，伺服器就需要重新啟動。一般的伺服器重啟會清除主記憶體中的
內容，其中包括邏輯分割區目前的執行設定，以及除錯該故障所需的資料。基於 POWER
的系統上有部分 hypervisor 無法存取非揮發性儲存裝置，無法在故障後儲存這些內容。
基於 POWER 的伺服器需要以保留主記憶體的方式進行暖重啟（warm reboot），以產生
除錯所需的記憶體傾印（memory dump）。本文件說明暖重啟的高階流程，以及如何從
hypervisor 記憶體中擷取所產生的傾印檔。

## 詞彙表

- **Boot（啟動）**：初始化電腦系統中的硬體元件並載入作業系統的處理程序。

- **Hostboot**：執行於主機處理器上的韌體，在基於 POWER 的伺服器上執行所有
  處理器、匯流排與記憶體的初始化。
  [read more](https://github.com/open-power/docs/blob/master/hostboot/HostBoot_PG.md)

- **Self Boot Engine (SBE)**：內建於 POWER 系統主機處理器中的微控制器，用於在
  啟動期間協助初始化處理器。它也是多項處理器硬體存取作業的進入點。
  [read more](https://sched.co/SPZP)

- **Master Processor（主處理器）**：最先被初始化以執行啟動韌體的處理器。

- **POWER Hardware Abstraction Layer (PHAL)**：BMC 上提供 POWER 硬體存取資訊的
  軟體元件。

- **Hypervisor**：Hypervisor（又稱虛擬機器監視器，VMM）是一種建立並執行虛擬機器
  的電腦軟體、韌體或硬體。
  [read more](https://en.wikipedia.org/wiki/Hypervisor)

- **System Dump（系統傾印）**：為除錯 hypervisor 中的故障而產生的主記憶體與硬體
  狀態傾印。

- **Memory Preserving Reboot (MPR，記憶體保留重啟)**：一種在重啟時保留揮發性記憶體
  內容的方法。

- **Terminate Immediate (TI)**：hypervisor 發生嚴重錯誤、無法繼續正常運作的狀態。

- **Attention（注意信號）**：由硬體或韌體針對特定事件所產生的信號。

- **Redfish**：Redfish 標準是一組規範，提供符合產業標準的通訊協定，以 RESTful
  介面管理伺服器、儲存、網路與融合基礎架構。
  [Read More](<https://en.wikipedia.org/wiki/Redfish_(specification)>)

- **OCC**：On-Chip Controller (OCC) 是直接內嵌於 POWER 處理器晶片（die）上的協同
  處理器。OCC 可用於控制處理器頻率、功耗與溫度，以達到高效能並降低能耗。

[Read More](https://openpowerfoundation.org/on-chip-controller-occ/)

- **Checkstop**：處理器核心內部發生的嚴重錯誤，會使該處理器核心停止所有處理活動。

- **PNOR**：PNOR 是存放韌體的主機 NOR flash。

## 背景與參考資料

當基於 POWER 的伺服器發生故障而需要重新啟動時，會通知 BMC 以啟動記憶體保留重啟。
BMC 透過告知各顆處理器上的 SBE 來開始重啟。SBE 停止執行中的核心，並以指定格式
收集硬體狀態，儲存到主機記憶體中。資料收集完成後，SBE 將控制權交回 BMC。接著
BMC 啟動記憶體保留重啟。系統完成啟動後，hypervisor 收集硬體資料與記憶體內容，
在主機記憶體中建立傾印檔。

## 需求

### 主要需求

- 無論故障發生時是否有外部實體可用於搬運（offload）傾印檔，都應能收集系統傾印。

- 應提供機制讓使用者可以要求產生系統傾印。

- 伺服器應能重新啟動回到執行期（runtime）。

- Hypervisor 應發送特殊的 attention 給 BMC，以通知發生嚴重故障。

- BMC 應能接收來自 hypervisor 的特殊 TI attention。

- BMC 應將主機狀態變更為 'DiagnosticMode'。

- BMC 應告知 SBE 開始記憶體保留重啟並收集硬體資料。

- 與傾印相關的錯誤紀錄需要納入傾印套件（dump package）中。

- 應建立傾印摘要（dump summary），包含大小及其他傾印詳細資訊。

- 傾印產生後，hypervisor 應通知 BMC。

- Hypervisor 應將傾印搬運給 BMC，以便傳輸至外部客戶端。

- 提供 Redfish 介面以管理傾印。

- 提供可從伺服器收集傾印的工具。

- 提供可解析傾印內容的方法。

## 建議設計

### 流程

記憶體保留重啟與系統傾印搬運的流程
![Memory preserving reboot and dump extraction flow](https://user-images.githubusercontent.com/16666879/77680635-40347000-6fba-11ea-8957-8f7fbc93f57e.jpeg)

#### 1 - 伺服器故障並通知 BMC

發生故障時，hypervisor 會產生 attention。BMC 上的 attention 監聽器（listener）
偵測到該 attention。在基於 OpenPOWER 的 Linux 系統上，會額外發送一個 s0 中斷給
SBE，以立即停止核心。

#### 2 - 分析錯誤資料

BMC 上的 attention 監聽器呼叫一個 chip-op，以分析產生 attention 的原因。

#### 3 - 啟動 System Dump

BMC 上的 attention 將重啟的 Diagnostic 目標（target）設定好，以啟動記憶體保留重啟。

#### 4 - 啟動記憶體保留轉換

作為重啟目標的一部分，會執行以下步驟：將系統狀態設為 DiagnosticMode、停止 OCC、
停用 checkstop 監控、對每顆 SBE 發送 enter_mpipl chip-op。

#### 5 - SBE 收集硬體資料

每顆 SBE 收集架構化狀態（architected states），並將其儲存到預先定義的位置。

#### 6 - BMC 啟動暖開機

SBE 完成硬體收集後，會執行以下動作以在保留記憶體的情況下啟動系統：重置 VPNOR、
啟用 watchdog、啟用 checkstop 監控、執行 istep proc_select_boot_master、執行
istep sbe_config_update、在主處理器上發送 continue_mpipl chip-op 取代 start_cbs。

#### 7 - Hostboot 啟動

SBE 啟動後會啟動 hostboot，hostboot 將架構化狀態複製到正確位置，並搬移記憶體內容
以建立傾印。

#### 8 - Hypervisor 格式化傾印並發送通知給 BMC

Hypervisor 啟動後會格式化傾印，並透過 PLDM 搭配傾印大小向 BMC 發送通知。PLDM 呼叫
dump manager 介面以通知傾印。Dump manager 為新傾印建立一個 dBus 物件，狀態為尚未
搬運（not offloaded）並記錄傾印大小。BMC web 擷取到物件建立信號後通知 HMC。

#### 9 - HMC 發送傾印搬運要求

HMC 準備好搬運時，會建立 NBD 伺服器並向 BMC 發送傾印搬運要求。BMCWeb 建立一個 NBD
客戶端與 NBD 代理（proxy）以搬運傾印。BMC dump manager 使用 hypervisor 提供的傾印
id 與 NBD 裝置 id 進行 PLDM 呼叫。PLDM 帶著傾印 id 將搬運要求發送給 hypervisor。

#### 10 - Hypervisor 開始傾印搬運

Hypervisor 開始透過 DMA 向下傳送傾印封包。PLDM 讀取 DUMP 並寫入 NBD 客戶端端點。
資料抵達 HMC 上的 NBD 伺服器，並寫入傾印檔。

#### 11 - Hypervisor 向下傳送搬運完成訊息

Hypervisor 向 BMC 向下傳送搬運完成訊息，BMC 再將其傳送給 HMC。NBD 端點會被清除。

#### 12 - HMC 驗證傾印並對 BMC 發送 dump DELETE

HMC 驗證傾印並向 BMC 發送傾印刪除要求。BMC 將傾印刪除訊息傳送給 hypervisor。
Hypervisor 刪除主機記憶體中的傾印。

### 記憶體保留重啟時序

![Memory preserve reboot sequence](https://user-images.githubusercontent.com/16666879/77681484-64448100-6fbb-11ea-94b4-9f2256241b1c.jpeg)

### 傾印搬運時序

![Dump offload sequence](https://user-images.githubusercontent.com/16666879/77681614-9e158780-6fbb-11ea-8fac-fbcffd563bef.jpeg)

## 已考量的替代方案

從 hypervisor 直接將傾印搬運給外部傾印收集應用程式，而非透過 BMC 搬運。但選擇透過
BMC 搬運是基於以下原因：BMC 為搬運所有傾印提供了統一的入口；在原型開發期間發現
透過 BMC 搬運效能較佳；透過 BMC 搬運對主機的開發衝擊較小。

## 影響

- BMC 與主機上的 PLDM —— 擴充 PLDM 實作，以傳遞傾印類型，並向 dump manager 通知
  新傾印檔。
  [PLDM Design]([https://github.com/openbmc/docs/blob/7c8847e95203ebfaeddb82a6e1b9888bc6a11311/designs/pldm-stack.md])

- BMC 上的 Dump manager —— BMC dump manager 支援存放於 BMC 上的傾印，需要擴充以
  支援主機傾印。

- 外部傾印搬運應用程式需要支援以 NBD 為基礎的搬運。

- 提議為傾印作業建立新的 redfish 結構描述（schema）。
  [Redfish Dump Proposal](https://lists.ozlabs.org/pipermail/openbmc/2019-December/019827.html)

- BMC Web 需要為傾印實作新的 redfish 規範。

- 在 openpower-hw-diags 中新增支援，以擷取特殊 attention 並啟動記憶體保留重啟。

- SBE 需要支援一項新作業，以分析來自主機的 attention。介面的更新尚未發布。

## 測試

- 單元測試計畫：使用 busctl 測試 dump manager 介面；透過設定 diag mode 目標測試
  重啟；使用獨立呼叫測試 SBE chip-op；使用 hypervisor 偵錯指令測試 PLDM；使用
  curl 測試 BMCWeb 介面。

- 整合測試：
  - 使用者發起的傾印測試，會呼叫記憶體保留重啟以收集傾印。
  - 透過注入主機錯誤來啟動記憶體保留重啟。
  - 搬運在主機中收集的傾印。

- System Dump 測試計畫：
  - 作為測試集（test bucket）的一部分，以自動化測試來啟動並搬運傾印。
  - 使用者發起的傾印與錯誤注入都應嘗試。
