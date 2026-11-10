# OpenBMC 上的 PLDM 堆疊

作者：Deepak Kodihalli <dkodihal@linux.vnet.ibm.com> `dkodihal`

建立日期：2019-01-22

## 問題描述

在 OpenBMC 上，帶內（in-band）IPMI 目前是 BMC 與主機韌體之間溝通的主要產業標準方式。我們在 OpenPOWER 伺服器上已經開始碰到 IPMI 一些與生俱來的限制：感測器數量有限，以及缺乏通用的控制機制（感測器是一種通用的監控機制），這兩項是最主要的問題。我們需要改進溝通協定，但同時又不希望自創一套自訂協定。

本設計打算採用 Platform Level Data Model（PLDM），這是由 DMTF 定義的標準應用層溝通協定。PLDM 參考了 IPMI 的概念，但克服了後者大多數的限制。PLDM 也被設計成可執行在標準傳輸協定之上，例如 MCTP（同樣由 DMTF 設計）。MCTP 透過定義硬體綁定（hardware bindings），提供了跨多種實體通道的通用傳輸層。PLDM over MCTP 這套解決方案也有助於克服 IPMI 所使用之硬體通道的部分限制。

PLDM 的目的是要能夠支援各種「機箱內溝通」（inside the box communication）：BMC - 主機、BMC - BMC、BMC - 網路控制器，以及 BMC - 其他（例如感測器）裝置。

## 背景與參考資料

PLDM 的設計目標是一個有效的介面與資料模型，能有效率地存取低層次的平台清單、監控、控制、事件，以及資料/參數傳輸功能。舉例來說，溫度、電壓或風扇感測器都可以有 PLDM 的表示方式，並可透過一組 PLDM 訊息來監控與控制平台。PLDM 定義了資料表示方式與命令，將平台管理硬體抽象化。

PLDM 將命令歸類在較廣泛的功能之下，並為每一項這些功能（也稱為 PLDM「Type」）定義獨立的規格。目前定義的 Type（及對應的規格）包括：PLDM base（及其相關的 IDs 與 states 規格）、BIOS、FRU、Platform monitoring and control、Firmware Update 與 SMBIOS。所有這些規格都可以在以下網址取得：

<https://www.dmtf.org/standards/pmci>

PLDM 看起來前景看好的部分原因（其中一些是相對 IPMI 的優勢）：

- 通用的帶內溝通協定。

- 已經存在的 PLDM Type 規格涵蓋了最常見的溝通需求。最多可以定義 64 種 PLDM Type（最後一種是 OEM）。目前定義了 6 種。每一種 PLDM type 最多可容納 256 個 PLDM 命令。

- PLDM 感測器的長度為 2 bytes。

- PLDM 引入了 effecters 的概念——一種控制機制。感測器與 effecters 都關聯到實體（entity，與 IPMI 類似，實體可以是實體的或邏輯的），其中感測器是用來監控的機制，effecters 則是用來控制的機制。Effecters 可以是數值型或狀態型。PLDM 定義了常用的實體及其 ID，但另外保留了 8K 個槽位可供定義 OEM 實體。

- DMTF 內有一個非常活躍的 PLDM 相關工作群組。

計畫是在 MCTP 上執行 PLDM。MCTP 有其本身的規格定義，而一份關於 MCTP 設計的提案也已經在討論之中。將會有一個中介的 PLDM over MCTP 綁定層，讓我們能夠透過 MCTP 傳送 PLDM 訊息。這也有其本身的規格定義，而這個綁定的設計將會另行提案。

## 需求

不同的 BMC 應用程式如何使用 PLDM 訊息，不在本需求文件的範圍之內。此處所列的需求是與 PLDM 協定堆疊及 request/response 模型相關：

- 必須實作 PLDM 訊息的序列化（marshalling）與反序列化（unmarshalling），這些定義在各個 PLDM Type 規格中。當然，這可以根據特定 Type 與功能的需求分階段進行。由於這純粹只是 PLDM 訊息的編碼與解碼，因此可以做成一個程式庫，讓 BMC 與其他韌體堆疊共用。各個 PLDM Type 的細節（例如 FRU 表格結構、感測器 PDR 結構等）都由這個 lib 實作。

- 必須實作將 PLDM 概念對應到 OpenBMC 原生概念的映射。例如：將 PLDM 感測器對應到由 phosphor-hwmon 託管的 D-Bus 物件、將 PLDM FRU 資料對應到由 phosphor-inventory-manager 託管的 D-Bus 物件等。這種映射不應受限於只有 D-Bus（意思是說，不應為了服務 PLDM 請求就非要把物件放到 Bus 上不可——這是 phosphor-host-ipmid 目前存在的問題）。本質上這些就是平台專屬的 PLDM 訊息處理常式（handlers）。

- BMC 應該要能夠同時扮演 PLDM responder 與 PLDM requester 的角色。作為 PLDM requester，BMC 可以監控/控制其他裝置。作為 PLDM responder，BMC 可以回應平台中經由各 requester 送給它的 PLDM 訊息。

- 作為 PLDM requester，BMC 必須能夠探索平台中其他支援 PLDM 的元件。

- 作為 PLDM requester，BMC 必須能夠同時發送訊息給不同的 responders。

- 作為 PLDM requester，BMC 必須能夠處理亂序的回應。

- 作為 PLDM responder，BMC 可以同時回應來自不同 requesters 的訊息，但規格並未強制要求這一點。換句話說，responder 可以是單執行緒的。

- 應該要能夠將 OEM 的 PLDM types/功能掛載（plug-in）到 PLDM 堆疊中。

- 作為 PLDM 感測器監控常駐程式（daemon），BMC 必須能夠列舉並監控衛星管理控制器（satellite Management Controller）、板載裝置或 PCIe 附加卡（add-on card）上的靜態或自我描述（使用 PDRs）的 PLDM 感測器。

## 建議設計

本文件涵蓋架構、介面與設計細節。它提供了對實作的建議，但實作細節不在本文件的範圍之內。

本設計的目標是用單一一個 PLDM 常駐程式同時服務 requester 與 responder 兩項功能，並擁有傳輸層專屬的端點（endpoints）以便在不同的通道上溝通。

本設計支援 requester 與 responder 功能的並行層面，但目標是在可能的情況下採用非同步 IO 與事件迴圈（event loops），而不是使用多執行緒。

以下是本設計的高階結構元素：

### PLDM 編碼/解碼程式庫

這個程式庫會接收一個 PLDM 訊息，將其解碼並擷取出訊息中各欄位的值。反之，給予一個 PLDM Type、命令碼（command code）以及該命令的資料欄位，它就能組出一個 PLDM 訊息。我們的想法是將其設計成一個通用程式庫，可供 BMC 與其他韌體堆疊使用，因為它就是編碼/解碼與協定那部分（而不是訊息的處理）。

### PLDM provider 程式庫

這些程式庫會實作對進入的 PLDM 請求的平台專屬處理（基本上就是協助 PLDM responder 的實作，見下一個項目），所以舉例來說，它們會查詢 D-Bus 物件（或甚至是像 JSON 檔案之類的東西）來取得平台專屬的資訊，以回應 PLDM 訊息。它們會與編碼/解碼 lib 連結。

應該要能夠掛載（plug-in）一個 provider 程式庫，讓使用者可以為新的 PLDM（標準及 OEM）Types 新增功能。這些程式庫會實作一個「register」API，以便掛載特定 PLDM 訊息的處理常式。類似這樣：

template <typename Handler, typename... args> auto register(uint8_t type,
uint8_t command, Handler handler);

這讓我們可以提供一個強型別（strongly-typed）的 C++ 處理常式註冊機制。同時也能夠在編譯時期驗證傳給處理常式的參數。

### Request/Response 模型

PLDM 常駐程式會連結編碼/解碼與 provider 程式庫。這個常駐程式必須實作以下功能：

#### Receiver/Responder

接收器在收到來自遠端 PLDM 裝置的進入 PLDM 訊息通知時（透過 D-Bus 訊號或來自傳輸層的回呼）會被喚醒。如果訊息類型是「Request」，它會將這些訊息路由給某個 PLDM provider 程式庫。透過該程式庫，將發起非同步的 D-Bus 呼叫（使用 sdbusplus-asio），如此接收器就可以註冊一個處理常式來處理 D-Bus 回應，而不必等待 D-Bus 回應。這樣它就能回到聆聽進入 PLDM 訊息的狀態。

在 D-Bus 回應處理常式中，接收器會透過傳輸層的 send message API 送出 PLDM 回應訊息。如果傳輸層的 send message API 會阻塞相當長的時間，那麼它就必須在自己的執行緒中執行。

如果進入的 PLDM 訊息類型是「Response」，那麼接收器會發出一個指向該回應訊息的 D-Bus 訊號。每當訊息太大而無法放進 D-Bus 的 payload 時，訊息會被寫入檔案，並在 D-Bus 訊號中包含一個指向該檔案的唯讀檔案描述元（file descriptor）。

#### Requester

把 BMC 設計成 PLDM requester 是件有趣的事。在 IPMI 上我們不曾有這種情況，因為 BMC 通常是 IPMI 伺服器。PLDM requester 的功能會分散在多個 OpenBMC 應用程式中（而不是單一一個大型的 requester 應用程式）——依據它們所對話的 responder 以及它們實作的高階功能而定。舉例來說，可能會有一個讓 BMC 使用 PLDM 來升級其他裝置韌體的應用程式——從某種意義上來說這會是一個通用應用程式，因為無論另一邊的裝置是什麼，都可能需要執行同一組命令。也可能會有一個針對遠端裝置進行風扇控制的應用程式，它是根據該裝置的感測器以及該裝置專屬的演算法來動作。

##### 建議的 requester 設計

一個 requester 應用程式/流程包含以下幾項：

- 與一個 PLDM 編碼/解碼程式庫連結，以便能够打包（pack）PLDM 請求並解包（unpack）PLDM 回應。

- 一個 D-Bus API，用來產生一個唯一的 PLDM instance id。這個 id 需要在所有送出的 PLDM 訊息（可能來自不同的程序）之間保持唯一。這需要放在 D-Bus 上，因為這個 id 需要在各個 PLDM requester 應用程式程序之間保持唯一。

- 一個 requester 用戶端 API，提供阻塞與非阻塞的函式，用以透過 MCTP 傳輸一個 PLDM 請求訊息並接收對應的回應訊息（阻塞式的 send() 會回傳一個 PLDM 回應）。這會是 mctp demux 常駐程式所提供的 socket API 上的一層薄封裝。這會為常見工作提供 APIs，如此就不必在每個 PLDM requester 應用程式中重新實作同樣的東西。這組 API 會被整合進編碼/解碼程式庫中（所以 libpldm 會容納編碼/解碼 APIs，並根據一個編譯時期旗標，也容納 requester APIs）。一個 PLDM requester 應用程式可以選擇不使用 requester 用戶端 APIs，而改為直接與 MCTP demux 常駐程式溝通。

##### 建議的 requester 設計 - 流程圖

a) 使用阻塞 API

```text
+---------------+               +----------------+            +----------------+               +-----------------+
|BMC requester/ |               |PLDM requester  |            |PLDM responder  |               |PLDM Daemon      |
|client app     |               |lib (part of    |            |                |               |                 |
|               |               |libpldm)        |            |                |               |                 |
+-------+-------+               +-------+--------+            +--------+-------+               +---------+-------+
        |                               |                              |                                 |
        |App starts                     |                              |                                 |
        |                               |                              |                                 |
        +------------------------------->setup connection with         |                                 |
        |init(non_block=false)          |MCTP daemon                   |                                 |
        |                               |                              |                                 |
        +<-------+return_code+----------+                              |                                 |
        |                               |                              |                                 |
        |                               |                              |                                 |
        |                               |                              |                                 |
        +------------------------------>+                              |                                 |
        |encode_pldm_cmd(cmd code, args)|                              |                                 |
        |                               |                              |                                 |
        +<----+returns pldm_msg+--------+                              |                                 |
        |                               |                              |                                 |
        |                               |                              |                                 |
        |----------------------------------------------------------------------------------------------->|
        |DBus.getPLDMInstanceId()       |                              |                                 |
        |                               |                              |                                 |
        |<-------------------------returns PLDM instance id----------------------------------------------|
        |                               |                              |                                 |
        +------------------------------>+                              |                                 |
        |send_msg(mctp_eids, pldm_msg)  +----------------------------->+                                 |
        |                               |write msg to MCTP socket      |                                 |
        |                               +----------------------------->+                                 |
        |                               |call blocking recv() on socket|                                 |
        |                               |                              |                                 |
        |                               +<-+returns pldm_response+-----+                                 |
        |                               |                              |                                 |
        |                               +----+                         |                                 |
        |                               |    | verify eids, instance id|                                 |
        |                               +<---+                         |                                 |
        |                               |                              |                                 |
        +<--+returns pldm_response+-----+                              |                                 |
        |                               |                              |                                 |
        |                               |                              |                                 |
        |                               |                              |                                 |
        +------------------------------>+                              |                                 |
        |decode_pldm_cmd(pldm_resp,     |                              |                                 |
        |                output args)   |                              |                                 |
        |                               |                              |                                 |
        +------------------------------>+                              |                                 |
        |close_connection()             |                              |                                 |
        +                               +                              +                                 +
```

b) 使用非阻塞 API

```text
+---------------+               +----------------+            +----------------+             +---------------+
|BMC requester/ |               |PLDM requester  |            |PLDM responder  |             |PLDM daemon    |
|client app     |               |lib (part of    |            |                |             |               |
|               |               |libpldm)        |            |                |             |               |
+-------+-------+               +-------+--------+            +--------+-------+             +--------+------+
        |                               |                              |                              |
        |App starts                     |                              |                              |
        |                               |                              |                              |
        +------------------------------->setup connection with         |                              |
        |init(non_block=true            |MCTP daemon                   |                              |
        |     int* o_mctp_fd)           |                              |                              |
        |                               |                              |                              |
        +<-------+return_code+----------+                              |                              |
        |                               |                              |                              |
        |                               |                              |                              |
        |                               |                              |                              |
        +------------------------------>+                              |                              |
        |encode_pldm_cmd(cmd code, args)|                              |                              |
        |                               |                              |                              |
        +<----+returns pldm_msg+--------+                              |                              |
        |                               |                              |                              |
        |-------------------------------------------------------------------------------------------->|
        |DBus.getPLDMInstanceId()       |                              |                              |
        |                               |                              |                              |
        |<-------------------------returns PLDM instance id-------------------------------------------|
        |                               |                              |                              |
        |                               |                              |                              |
        +------------------------------>+                              |                              |
        |send_msg(eids, pldm_msg,       +----------------------------->+                              |
        |         non_block=true)       |write msg to MCTP socket      |                              |
        |                               +<---+return_code+-------------+                              |
        +<-+returns rc, doesn't block+--+                              |                              |
        |                               |                              |                              |
        +------+                        |                              |                              |
        |      |Add EPOLLIN on mctp_fd  |                              |                              |
        |      |to self.event_loop      |                              |                              |
        +<-----+                        |                              |                              |
        |                               +                              |                              |
        +<----------------------+PLDM response msg written to mctp_fd+-+                              |
        |                               +                              |                              |
        +------+EPOLLIN on mctp_fd      |                              |                              |
        |      |received                |                              |                              |
        |      |                        |                              |                              |
        +<-----+                        |                              |                              |
        |                               |                              |                              |
        +------------------------------>+                              |                              |
        |decode_pldm_cmd(pldm_response) |                              |                              |
        |                               |                              |                              |
        +------------------------------>+                              |                              |
        |close_connection()             |                              |                              |
        +                               +                              +                              +
```

##### 建議 requester 設計的替代方案

a) 定義用於發送與接收 PLDM 訊息的 D-Bus 介面：

```python
method sendPLDM(uint8 mctp_eid, uint8 msg[])

signal recvPLDM(uint8 mctp_eid, uint8 pldm_instance_id, uint8 msg[])
```

PLDM requester 應用程式接著就可以呼叫上述的這些應用程式。雖然這簡化了使用者的工作，但它有兩個缺點：

- 實作這種介面的應用程式可能會成為單一方失效點（single point of failure），此外並行發送訊息也會是個挑戰。
- 訊息的 payload 可能會很大（好幾個頁面），而為了 D-Bus 傳輸去複製它可能並不理想。

### 多個傳輸通道

PLDM 常駐程式可能需要透過不同的通道與遠端 PLDM 裝置溝通。雖然 MCTP 可能會提供某層級的抽象化，但 PLDM 常駐程式必須實作一個 D-Bus 介面來鎖定特定的傳輸通道，如此 BMC 上的 requester 應用程式就能透過該傳輸層發送訊息。此外，應該要能夠掛載平台專屬的 D-Bus 物件，這些物件實作了鎖定平台專屬傳輸層的介面。

### 處理由主機韌體傳下來的 PLDM FRU 資訊

注意：雖然這特別是指主機與 BMC 之間的溝通，但這裡大部分的內容可能也適用於處理從連接到 BMC 的裝置所收到的 PLDM FRU 資訊。

需求是要 BMC 消費從主機韌體收到的 PLDM FRU 資訊，然後透過 Redfish 將同樣的資訊揭露出來。一個例子是主機韌體透過 PLDM FRU 命令傳下處理器與核心資訊，而 BMC 透過 Processor 與 ProcessorCollection schemas 讓這些資訊可供取用。

這個設計是圍繞著 BMC 上的 pldmd 來建構的：

- pldmd 會向主機韌體的 PLDM 堆疊要求主機的 FRU record table，方式是對它發送 PLDM GetFRURecordTable 命令。如果主機表示支援 PLDM FRU 規格，pldmd 就應該發送這個命令。pldmd 會從主機韌體收到一個 PLDM FRU record table（
  <https://www.dmtf.org/sites/default/files/standards/documents/DSP0257_1.0.0.pdf>）。
  這個常駐程式會解析 FRU record table，並將 PLDM FRU 資訊託管在 D-Bus 上。

- Pldmd 也會託管來自 xyz.openbmc_project. Inventory 命名空間的 FRU 清單（inventory）D-Bus。可以將這個套用（apply）在這些物件上，做法是把 PLDM FRU 屬性值轉換成 xyz.openbmc_project.Inventory.Decorator.Asset、
  xyz.openbmc_project.Inventory.Decorator.Revision、
  xyz.openbmc_project.Inventory.Decorator.AssetTag 與
  xyz.openbmc_project.Inventory.Decorator.Compatible 介面的屬性值，例如 Part Number、Serial Number、Version、Names 與 AssetTag。Bmcweb 可以根據 D-Bus 介面找到這些 FRU 清單物件，就像它今天所做的那樣。

### MCTP 端點探索

`pldmd`（PLDM 常駐程式）利用
[MCTP D-Bus interfaces](https://github.com/openbmc/phosphor-dbus-interfaces/tree/master/yaml/xyz/openbmc_project/MCTP)
來列舉系統中所有的 MCTP 端點。MCTP D-Bus 介面實作了 `SupportedMessageTypes`，用以表示每個端點支援哪些 Message type。`pldmd` 會監看來自 `mctpd` 的 `InterfacesAdded` D-Bus 訊號。它也會比對 `InterfaceRemoved` D-Bus 訊號，以從 `mctpd` 找出被移除的端點 EIDs。

### Termin 管理與探索

`pldmd` 會維護一張 terminus 表格來管理系統中的 PLDM terminus。當 `pldmd` 從 MCTP D-Bus 介面收到更新後的 EID 表格時，`pldmd` 應該檢查該 EID 是否支援 PLDM message type（0x01），然後將尚未在 terminus 表格中的 EID 加入。當 terminus EID 從 MCTP D-Bus 介面被移除時，`pldmd` 也應該從 terminus 表格中清理掉被移除的端點。

對於表格中的每一個 terminus，`pldmd` 都會走一遍以下的步驟：

- Terminus 初始化
- Terminus 探索
- Terminus 監控與控制

當它從 terminus 表格中被移除時，所有新增的 D-Bus 物件路徑與 D-Bus 介面、以及該 terminus 的 Monitoring/Controlling 工作都會被移除。

#### Terminus 初始化

PLDM 介面中的每個 terminus 都由 terminus ID（TID）來識別。這個 TID 是一個唯一的編號 `TID#`。當一個新的 terminus 被加入 terminus 表格時，`pldmd` 應該發送 `GetTID` 以取得 `TID#`。當收到的 `TID#` 已經存在於 TID 池中時，`pldmd` 會呼叫 `SetTID` 命令，為該 terminus 指派一個新的 TID。

除了 `TID#` 之外，terminus 也可以有 `$TerminusName` 或 `$DeviceName`，這可以被編碼在 Terminus 的 `Entity Auxiliary Names PDR`（DSP0248 1.2.1 第 28.18 節）中，或被編碼在 MCTP Entity-manager 端點 EID 設定檔案中
[Entity-Manager EID configuration](https://github.com/openbmc/entity-manager/blob/master/configurations/yosemite4_floatingfalls.json#L7)。

因為 `$TerminusName` 會被包含在 Terminus 的感測器、effecters 與狀態（states）的 D-Bus 物件路徑中，所以建議要將 EM EID 設定或 Terminus 的 `Entity Auxiliary Names PDR` 納入，以便支援感測器、effecters 或狀態的 Terminus。當 EM EID 設定不可用時，應該要加入 `Entity Auxiliary name PDR`，如此所有的感測器就不會以任何方式在裡面帶有 terminus 編號 `TID#`。

#### Teminus 探索

在 TID 指派步驟之後，`pldmd` 應該走一遍 `Terminus Discovery` 的步驟：

- 發送 `GetPLDMType` 與 `GetPLDMVersions` 命令給該 terminus，以記錄其支援的 PLDM type message/version。
- 如果該 terminus 支援 `GetPDRs` 命令類型，`pldmd` 就會發送該命令以取得 terminus 的 PDRs。根據所取得的 PDRs，`pldmd` 會蒐集：
  - 系統中各實體之間的關聯，使用 `Entity Association PDR`（DSP0248 1.2.1 第 28.17 節）。
  - 實體名稱，使用 `Entity Auxiliary Names PDR`（DSP0248 1.2.1 第 28.18 節）。
  - terminus 的感測器/effecter/state PDRs 中所描述的各實體裡的 sensor/effecter/state 資訊（DSP0248 1.2.1 第 28.4、28.6、28.8、28.11、28.14、28.15、28.25 等節）。
  - Fru 資訊，使用 FRU PDRs（DSP0248 1.2.1 第 28.22 節）。
  - 其他資訊，使用 DSP0248 1.2.1 第 28.x 節中的其他 PDRs。

  上述這些 PDRs 也可以在 JSON 設定檔案中被設定。當 `PDR configuration` 可用時，`pldmd` 常駐程式會略過 `GetPDRs` 步驟，改為讀取那些檔案來蒐集這些資訊。設定檔案的範本可以遵循目前
  [PDRs configuration files](https://github.com/openbmc/pldm/tree/master/configurations)
  的格式。

  在這一步之後，`pldmd` 就會擁有來自 Terminus 的 PDRs 或 Terminus 的 EM JSON 設定檔案的 `$sensorAuxNames` 清單。這個 `$sensorAuxNames` 會被包含在 `Sensor creating and monitor` 小節中的 PLDM Sensors D-Bus 物件路徑裡。

- `pldmd` 接著會建立 Terminus 的清單、感測器、effecters 的 D-Bus 物件路徑。
- 在 `terminus discovery` 的最後幾個步驟中，`pldmd` 會發送 `SetEventReceiver` 來通知 BMC 已準備好接收來自該 terminus 的事件訊息。

#### Terminus 監控與控制

在完成探索步驟之後，常駐程式會開始監控感測器、回應來自 terminus 的事件，並處理來自使用者的 terminus 控制動作。

### 感測器的建立與監控

要找出 PLDM terminus 中的所有感測器，`pldmd` 應該透過 PDR Repository 命令（`GetPDRRepositoryInfo`、`GetPDR`）為必要的參數（例如 `sensorID#`、`$SensorAuxName`、單位等）擷取所有 Sensor PDRs。
`pldmd` 可以使用 libpldm 編碼/解碼 APIs（`encode_get_pdr_repository_info_req()`、
`decode_get_pdr_repository_info_resp()`、`encode_get_pdr_req()`、
`decode_get_pdr_resp()`）來建立命令訊息，然後將它發送給 PLDM terminus。

關於 DSP0248 1.2.1 第 8.3.1 節所述明的靜態裝置，該裝置僅使用 PLDM 來存取，並不支援 PDRs。該裝置的 PDRs 需要由平台開發人員透過平台專屬的 PDR JSON 檔案來編碼。`pldmd` 會產生這些由 JSON 檔案編碼的感測器 PDRs，並將其解析得與從 PLDM terminus 取得的 PDRs 相同。

`pldmd` 應該將找到的 PLDM 感測器揭露到 D-Bus 物件路徑
`/xyz/openbmc*project/sensors/<sensor_type>/SensorName`。`sensorName` 的格式可以是 `$TerminusName_$SensorAuxName` 或 `$TerminusName_SensorID#`。只要 `$SensorAuxName` 存在，就會被包含在 `sensorName` 中。為了將感測器狀態揭露到 D-Bus，`pldmd` 應該在 PLDM 感測器的 D-Bus 物件路徑上實作以下的 D-Bus 介面。EM EID 設定或 Terminus 的 `Entity Auxiliary name PDR` 會提供 `$TerminusName`。而 `$SensorAuxName` 可以在 EM EID 感測器設定或感測器 PDRs 中找到。

- [xyz.openbmc_project.Sensor.Value](https://github.com/openbmc/phosphor-dbus-interfaces/blob/master/yaml/xyz/openbmc_project/Sensor/Value.interface.yaml)，
  這個介面揭露感測器的讀值單位、值、Max/Min Value。

- [xyz.openbmc_project.State.Decorator.OperationalStatus](https://github.com/openbmc/phosphor-dbus-interfaces/blob/master/yaml/xyz/openbmc_project/State/Decorator/OperationalStatus.interface.yaml)，
  這個介面揭露感測器是否功能正常（functional）的狀態。

在完成 PLDM 感測器的探索之後，`pldmd` 應該用必要的命令（例如 `SetNumericSensorEnable`、
`SetSensorThresholds`、`SetSensorHysteresis` 與 `InitNumericSensor`）初始化所有找到的感測器，然後依據 PLDM terminus 的能力，以輪詢（polling）或非同步事件的方式開始將感測器狀態更新到 D-Bus 物件上。

`pldmd` 應該在成功取得 `GetSensorReading` 命令的回應之後，更新 `Sensor.Value` D-Bus 介面的 value 屬性。如果 `pldmd` 無法從 PLDM terminus 取得回應，或 PLDM terminus 回傳的 completion code 不是 `PLDM_SUCCESS`，那麼 `State.Decorator.OperationalStatus` D-Bus 介面的 Functional 屬性應該被更新為 false。

#### 輪詢 v.s. 非同步方式

對於每個 terminus，`pldmd` 都維護一個清單來輪詢該 Terminus 的感測器並將狀態揭露到 D-Bus。`pldmd` 有一個輪詢計時器，帶有可設定的間隔，會週期性地更新該 terminus 的 PLDM 感測器。清單中的 PLDM 感測器有一個 `updateTime`，它被初始化為感測器 PDRs 中所定義的 `updateInterval` 值。當輪詢計時器逾時時，就會使用 `GetSensorReading` 命令讀取該 terminus 的感測器。讀取的條件是從 `last read timestamp` 到 `current timestamp` 的 `elapsed time` 超過該感測器的 `updateTime`。`pldmd` 應該要有 APIs 可供其他工作暫停（paused）與恢復（resumed）（例如在韌體更新期間暫停感測器輪詢，以將頻寬最大化）。

要讓某個感測器能夠以非同步事件的方式將其狀態更新給 `pldmd`，`pldmd` 需要實作
[DSP0248 1.2.1](https://www.dmtf.org/sites/default/files/standards/documents/DSP0248_1.2.1.pdf)
第 13.1 節 PLDM Event Message 中所描述之 `PlatformEventMessage` 命令的 responder。`pldmd` 會檢查來自 PLDM terminus 的 `EventMessageSupported` 命令回應，以判斷它是否能產生事件。如果同一個 PLDM terminus 中所有感測器的 `updateInterval` 都長於最終的輪詢時間，那麼一個 PLDM 感測器就可以以事件非同步方式運作。在 `pldmd` 開始接收來自 PLDM terminus 的非同步事件之前，`pldmd` 應該先將該感測器從輪詢清單中移除，然後發送必要的命令（例如 `EventMessageBufferSize` 與 `SetEventReceiver`）給 PLDM terminus 以進行初始化。

## 曾考慮過的替代方案

繼續使用 IPMI，但開始更多地運用 OEM 擴充功能來配合新平台的需求。然而，鑑於 IPMI 標準已不再處於活躍的開發狀態，我們很可能最終會得到大量平台專屬的客製化。這也沒有以標準化的方式解決硬體通道的問題。至少在 OpenPOWER 硬體上，我們已經開始碰到 IPMI 的一些限制（例如，我們需要超過 255 個感測器）。

## 影響

將會需要開發工作來實作 PLDM 協定、request/response 模型，以及平台專屬的處理。為了實作各 PLDM Types 的協定細節，需要低層次的設計。這類低層次的設計不包含在本提案之中。

設計與開發工作需要納入平台管理子系統之管理控制器與管理裝置的韌體堆疊。

## 測試

可以在不依賴底層傳輸層的情況下進行測試。

responder 的功能可以透過模擬（mocking）一個 requester 與傳輸層來測試：這基本上會測試協定處理與平台專屬處理。requester 的功能可以透過模擬一個 responder 來測試：這會測試 instance id 的處理以及發送/接收函式。

共用程式庫的 APIs 可以透過模糊測試（fuzzing）來測試。

用於從 PLDM terminus 解析 PDRs 的 APIs 可以透過一個模擬的 responder 來測試。提供了一個範例 JSON 檔案，用以測試那些用於為靜態 PLDM 感測器模擬 PDRs 的 APIs。
