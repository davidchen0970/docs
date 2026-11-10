# BIOS->BMC SMM 錯誤記錄佇列 daemon

Author:

- Brandon Kim <brandonkim@google.com> `@brandonk`

其他貢獻者：

- Marco Cruz-Heredia <mcruzheredia@google.com>

建立日期：2022 年 3 月 15 日

## 問題描述

我們已識別出一些使用情境，其中 BIOS 會進入 System Management Mode（SMM）
以提供錯誤記錄給 BMC，由於所處環境的時間限制，訊息必須盡快送出，且不需要
BMC 回覆握手 / ack。我們提案的這個 daemon 的目標，是實作一個環形緩衝區
（circular buffer），讓 BIOS 可以對共用的 BIOS->BMC 緩衝區以
fire-and-forget 的方式寫入。

## 背景與參考

BMC 與 BIOS 之間有多種通訊方式，但只有少數幾種不需要握手、且能讓資料保留在
共用記憶體中。這些方式將列在「已考慮的替代方案」章節中。

不同的 BMC 廠商支援不同的方法，例如 Shared Memory（SHM，透過 LPC / eSPI）
以及 P2A 或 PCI Mailbox，但目前使用這些方法的既有 daemon 會透過 IPMI blob
來溝通資料傳輸的位置與數量（請見
[phosphor-ipmi-flash](https://github.com/openbmc/phosphor-ipmi-flash) 與
[libmctp/astlpc](https://github.com/openbmc/libmctp/blob/master/docs/bindings/vendor-ibm-astlpc.md)）

## 需求

此 daemon 的基本需求如下：

1. BMC 應以一種方式初始化共用緩衝區，讓 BIOS 能夠辨識何時可以寫入該緩衝區
2. 初始化之後，BIOS 在對共用緩衝區進行任何寫入之前，不應需要等待 BMC 的 ack
   回覆（**無同步機制**）
3. BIOS 應為共用緩衝區的主要寫入者，BMC 主要負責讀取 payload，僅在更新標頭
   時才寫入緩衝區
4. BMC 應從共用緩衝區讀取新的 payload 以進行後續處理
5. BIOS 必須能在 50µs 內將一個 payload（約 1KB）寫入緩衝區

共用緩衝區的大小會依特定 BMC 平台上通訊協定所允許的大小而定（例如 Nuvoton
NPCM 7xx 的 PCI Mailbox 為 16KB），而每個 payload 估計小於 1KB。

此 daemon 假設不會有其他流量透過該通訊協定傳輸。環形緩衝區及其標頭會提供
一些防止資料破壞的保護，但不應過度依賴。

## 提案設計

與共用緩衝區介接的實作將非常貼近地遵循
[phosphor-ipmi-flash](https://github.com/openbmc/phosphor-ipmi-flash)。未來，
將 PCI Mailbox、P2A 與 LPC 抽取為獨立的函式庫、由
`phosphor-ipmi-flash` 與此 daemon 共用，可能會是明智之舉，以減少重複的程式碼。

摘錄自 Marco <mcruzheredia@google.com> 內部關於環形緩衝區的設計文件，其標頭
的資料結構看起來如下：

| 名稱                                | 大小                             | 偏移位置      | 寫入者   | 說明                                                                                                                                                                                                                                                                                                                                                                              |
| ----------------------------------- | -------------------------------- | ----------- | ------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| BMC Interface Version               | 4 bytes                          | 0x0         | BMC at init  | 讓 BIOS 得以判斷自身是否與 BMC 相容                                                                                                                                                                                                                                                                                                                            |
| BIOS Interface Version              | 4 bytes                          | 0x4         | BIOS at init | 讓 BMC 得以判斷自身是否與 BIOS 相容                                                                                                                                                                                                                                                                                                                            |
| Magic Number                        | 16 bytes                         | 0x8         | BMC at init  | 用於設定佇列狀態的 magic number，說明如下。當記憶體區塊可供使用時，由 BMC 寫入一次。BIOS 在記錄前必須檢查此值。當 BMC 懷疑資料遭破壞時，可以改變此數值，以防止 BIOS 在重新初始化期間寫入任何內容                                                                                              |
| Queue size                          | 3 bytes                          | 0x18        | BMC at init  | 表示為環形佇列所配置區域的大小。僅在初始化時由 BMC 寫入，執行期間不應改變。**此值包含標頭與 UE 區域的大小**                                                                                                                                                                                            |
| Uncorrectable Error region size     | 2 bytes                          | 0x1b        | BMC at init  | 表示保留給 Uncorrectable Error（UE）記錄的區域大小。僅在初始化時由 BMC 寫入，執行期間不應改變                                                                                                                                                                                                                                               |
| BMC flags                           | 4 bytes                          | 0x1d        | BMC          | **BIT0 - BMC UE reserved region「switch」**：當 BMC 從保留區域讀取一筆 UE 時切換。<br>**BIT1 - Overflow**：讓 BIOS 知道 BMC 已看見溢位事件。當 BMC 確認（ack）該溢位事件時切換<br>**BIT2 - BMC_READY**：BMC 在收到從 BIOS 取得的所有初始化資訊、準備好接收記錄之後，會設定此位元。 |
| BMC read pointer                    | 3 bytes                          | 0x21        | BMC          | 用於讓 BIOS 偵測 BMC 何時來不及讀取先前的錯誤記錄，進而導致環形緩衝區溢位。                                                                                                                                                                                                                                        |
| Padding                             | 4 bytes                          | 0x24        | Reserved     | 為 8 byte 對齊而做的填充                                                                                                                                                                                                                                                                                                                                                             |
| BIOS flags                          | 4 bytes                          | 0x28        | BIOS         | **BIT0 - BIOS UE reserved region「switch」**：當 BIOS 寫入一筆 UE 到保留區域時切換。<br>**BIT1 - Overflow**：讓 BMC 知道它遺漏了一筆錯誤記錄。當 BIOS 看見溢位且尚未處於溢位狀態時切換<br>**BIT2 - Incomplete Initialization**：當 BIOS 嘗試初始化但未在 BMC flags 中看到 BMC 以 `BMC_READY` 位元回覆 ack 時設定   |
| BIOS write pointer                  | 3 bytes                          | 0x2c        | BIOS         | 指出 BIOS 下一步將在哪裡寫入記錄。用於告訴 BMC 何時應讀取新的記錄                                                                                                                                                                                                                                                                                     |
| Padding                             | 1 byte                           | 0x2f        | Reserved     | 為 8 byte 對齊而做的填充                                                                                                                                                                                                                                                                                                                                                             |
| Uncorrectable Error reserved region | TBD1                             | 0x30        | BIOS         | 僅供 UE 記錄使用的保留區域。只有在緩衝區其餘部分即將溢位、且該區域中已沒有未讀取的 UE 記錄時，才會使用此區域。                                                                                                                                                                                                                           |
| Error Logs from BIOS                | Size of the Buffer - 0x30 - TBD1 | 0x30 + TBD1 | BIOS         | 記錄因類型而異，因此每筆記錄都會以標頭自我描述。此區域會填滿緩衝區剩餘的部分                                                                                                                                                                                                                                                                         |

### 初始化

此 daemon 會先對整個緩衝區寫入零來初始化共用緩衝區，接著初始化標頭的
`BMC at init` 欄位，然後再寫入 `Magic Number`。一旦 `Magic Number` 被寫入，
BIOS 就會認為共用緩衝區已正確初始化，並能開始寫入項目。

如果 BIOS 與 BMC 之間還需要任何額外的初始化，BMC 必須在初始化完成後設定
BMC flags 中的 `BMC_READY` 位元。如果 BIOS 未看見該旗標被設定，BIOS 應設定
`Incomplete Initialization` 旗標，以通知 BMC 重新初始化緩衝區。

### 讀取與處理

此 daemon 會以固定的時間間隔輪詢緩衝區（確切的數值將可設定，因為不同平台的
處理時間與效能可能需要不同的輪詢速率），一旦偵測到新的 payload，該 payload
將由一個同樣可在編譯時期選擇與設定的函式庫來處理。

請注意，Uncorrectable Error 記錄有一塊保留區域，因為它們包含我們不希望遺失
的重要資訊，應優先於一般的錯誤記錄處理。只有在一般錯誤記錄佇列即將溢位、且
BMC 已透過 `BMC flag` 的 Bit0 確認此區域中原有的 UE 記錄已被讀取時，才會使用
這塊保留區域來記錄一筆 UE 記錄。

處理函式庫的一個範例（也是我們希望在這個 daemon 的初始版本中推出的功能），
是一個 RDE 解碼器，用於處理 Redfish Device Enablement（RDE）指令的子集合，
並解碼其附帶的 Binary Encoded JSON（BEJ）payload。

## 已考慮的替代方案

- 曾考慮 IPMI，但未符合我們在約 50 微秒內寫入 1KB 條目的速度需求。
  - 作為參考，最初的 PCI Mailbox 效能量測顯示 1KB 條目的寫入約需 10 微秒。
- 也曾考慮 LPC / eSPI，但我們 BMC 的 SHM 緩衝區受限於 4KB，對我們的使用情境
  而言不夠。
- 曾考慮 `libmctp` 與 MCTP PCIe VDM。
  - `libmctp` 目前的實作依賴 LPC 作為傳輸綁定，並使用 IPMI KCS 進行同步。如前
    所述，LPC 不符合我們目前的需求，而同步機制也無法運作。
  - 在未來的平台上，我們可能會使用 MCTP PCIe VDM，屆時我們將在 BMC 與 BIOS
    兩端擁有更多具備專業知識的資源（目前我們缺乏這些資源）以配合目前的專案
    時程。

## 影響

從緩衝區讀取並處理可能會影響 BMC 的效能，特別是當輪詢速率設定過高時。

### 組織層面

此設計需要 2 個 repository：

- bios-bmc-smm-error-logger
  - 此 repository 將實作本文件所述的 daemon
  - 建議的維護者：<wltu@google.com> , <brandonkim@google.com>
- libbej
  - 此 repository 將盡可能遵循 PLDM RDE 規範，以進行 RDE BEJ 解碼（初期而言，
    編碼未來可能會加入），並將託管一個以 C 撰寫的函式庫
  - 建議的維護者：<wltu@google.com> , <brandonkim@google.com>

## 測試

單元測試將涵蓋 daemon 的各個部分，主要為：

- 初始化
- 環形緩衝區處理
- 解碼 / 處理函式庫
