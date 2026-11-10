# OpenBMC 平台通訊通道：MCTP 與 PLDM

作者：Jeremy Kerr <jk@ozlabs.org> `jk`

## 問題描述

目前，host 與 BMC 之間有幾種不同的通訊方式。這些方式主要基於
IPMI，但也包含一些硬體專用的旁路通道（side-channel），例如 hiomap。
至少在 OpenPOWER 硬體上，我們無疑已經開始遇到 IPMI 的一些限制
（例如，我們需要超過 255 個感測器），以及 IPMI 通常所使用的硬體
通道上的限制。

本設計的目標是使用 Management Component Transport Protocol（MCTP），
為 OpenBMC 平台提供的多種通道建立一個共通的傳輸層。接著，在 MCTP
之上，我們有機會改用較新的 host/BMC 訊息傳遞協定，以克服我們在使用
IPMI 時遇到的一些限制。

## 背景與參考資料

將目前技術堆疊中的「傳輸（transport）」與「訊息傳遞協定
（messaging protocol）」兩部分分開，可以讓我們分別設計這些部分。
目前 IPMI 同時定義了這兩者：我們現在有 BT 與 KCS（兩者皆為
IPMI 2.0 標準的一部分）作為傳輸層，而 IPMI 本身則是訊息傳遞協定。

過去曾有人嘗試改進 IPMI 的硬體傳輸機制，但至今都不是以跨實作的方式
進行。這也無法解決 IPMI 資料模型的一些限制。

MCTP 定義了一個標準的傳輸協定，以及許多獨立的實體層綁定
（physical layer bindings），用於實際傳輸 MCTP 封包。這些標準由
DMTF 的 Platform Management Working group 定義，標準文件可於以下
網址取得：

<https://www.dmtf.org/standards/pmci>

下圖顯示這些標準對應到我們可能在 OpenBMC 中實作的功能領域。圖中
提供的 DSP 編號是參照 DMTF 標準文件的編號。

![MCTP Diagram](mctp-standards.svg)

這裡的一個關鍵概念是將傳輸協定與實體層綁定分離；這代表一個 MCTP
「堆疊（stack）」可以使用 I2C、PCI、Serial 或自訂硬體通道，而該
堆疊的上層不需要知道硬體的實作細節。這些較高的層級只需要知道它們
正在與某個由 Entity ID（MCTP EID）定義的實體通訊即可。這些實體可以
是平台中任何透過 MCTP 通訊的元素——例如 host 裝置、BMC，或任何其他
系統週邊裝置——可以是靜態的，也可以是熱插拔的。

本文件聚焦於平台設計中的「傳輸（transport）」部分。雖然這確實能
讓新的訊息傳遞協定（主要是 PLDM）得以運作，但那些元件並未在此詳細
涵蓋；我們將以獨立的設計文件來提出那些部分。例如
[pldm-stack.md] 中的 PLDM 設計。

作為本設計的一部分，文中對 MCTP「訊息（messages）」與「封包
（packets）」的用詞是刻意為之，以符合 MCTP 標準中的定義。MCTP
訊息是在 MCTP 端點之間傳遞的高階資料；封包通常較小，是實際透過
硬體傳送的內容。大於硬體 Maximum Transmit Unit（MTU）的訊息會由
傳輸端實作切割成個別封包，並在接收端實作重新組裝。

## 需求

host 與 BMC 之間的任何通道都應該：

- 具備簡單的序列化與反序列化格式，以便在 host 韌體中實作，因為
  host 韌體的執行期能力差異很大

- 允許不同的硬體通道，因為 OpenBMC 的目標平台種類繁多

- 能在簡單的硬體實作上使用，但同時具備機制，讓需要更高頻寬的
  平台可以進行高頻寬的訊息傳遞。

- 理想上，能與較新的訊息傳遞協定整合

## 提議設計

OpenBMC 中的 MCTP 基礎設施有兩種實作方式：

- 以使用者空間（userspace）為基礎的方式，使用一個核心函式庫，
  加上負責多工解多工（demultiplexing）的 daemon。這在
  [MCTP Userspace](mctp-userspace.md) 中有說明。

  少數平台正在使用這種方式，但新設計應避免採用。

- 以核心（kernel）為基礎的方式，為用戶端與伺服器應用程式提供
  sockets API。這種方式推薦用於新設計，並在 [MCTP Kernel](mctp-kernel.md)
  中說明。

兩種設計的設計細節都涵蓋在各自的相關文件中，但兩者共用由本文件
定義的問題描述、背景與參考資料、需求、已評估的替代方案以及影響
等章節。

## 已評估的替代方案

對於在 OpenBMC 中實作 MCTP，主要有兩個替代方案：

繼續使用 IPMI，但開始更多運用 OEM 擴充功能以配合新平台的需求。
然而，由於 IPMI 標準已不再活躍開發，我們最終可能會產生大量平台
專屬的客製化修改。這也沒有以標準化的方式解決硬體通道的問題。

在 host 與 BMC 之間使用 Redfish。這代表 host 韌體需要一個 HTTP
用戶端、一個 TCP/IP 堆疊、一個 JSON 序列化/反序列化器，以及對
Redfish schema 的支援。雖然某些環境（例如基於 UEFI 的韌體）可能
具備這些元件，但這對所有 host 韌體實作（例如 OpenPOWER）未必可行。
我們有可能執行一個簡化的 Redfish 堆疊——事實上，MCTP 已有一個
Redfish-over-MCTP 通道的提案（DSP0218），它使用簡化的序列化格式，
且不要求 HTTP。然而，這可能會讓 host 韌體變得非常複雜。

## 影響

需要投入開發能量來實作 MCTP 傳輸層，以及任何 MCTP 訊息傳遞的新
使用者（例如 PLDM 實作）。這些工作在某種程度上會與我們現有的
IPMI handler 工作重複。

我們會希望讓 IPMI 繼續平行運作，因此「升級」路徑應該會相當順利。

設計與開發工作需要潛在的 host、管理控制器（management controllers）
以及受管裝置（managed device）實作方共同參與。
