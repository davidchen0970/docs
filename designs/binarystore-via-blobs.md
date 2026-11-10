# 透過 OEM IPMI Blob 傳輸實作 BMC 通用二進位 Blob 儲存空間

作者：Kun Yi <kunyi@google.com> `kunyi!`

建立日期：2018-12-07

## 問題說明

伺服器平台的宿主作業系統經常需要儲存及／或存取與機器本身綁定的資料，而這種資料在大多數情況下與主機板呈現 1:1 的對應關係。傳統上，這是透過直接存取 FRU EEPROM，或是透過 IPMI FRU 指令經由 BMC 來達成。然而，由於以下原因，這樣的作法未必可行或理想：

- FRU 資料可能由製造（MFG）階段寫入，並被視為唯讀
- FRU 資料可能不是 IPMI FRU 格式
- 要儲存的資料可能不符合 IPMI FRU 規格中所定義的資料型別
- 宿主可能希望在例如 BMC EEPROM 中儲存多份複本

BMC 通用 IPMI blobs 二進位儲存空間，簡稱「binary store」，滿足一個單純的目的：透過 IPMI blobs 傳輸層提供一個讀取／寫入／序列化的抽象層，讓使用者能把二進位資料存放在 BMC 可存取的持久化位置上。

儘管名稱如此，這個二進位 blob 儲存空間並不是什麼都能用。

- 它不是宿主／BMC 的信箱（mailbox）。一般來說，BMC 應保留 blob store 的空間，並避免嘗試寫入它，以避免並發方面的疑慮。預期唯一的存取者就是 IPMI blob store 指令。
- 它不是金鑰儲存空間（key store）。由於儲存的資料對 IPMI 使用者而言是可存取的，而且並未以任何方式加密，因此必須格外謹慎。它並不適合用於儲存包含任何金鑰或機密資訊的資料。
- 要儲存的資料不應過大。由於每個 IPMI 封包的大小受限，嘗試傳送過大的二進位資料會耗時過久，而 IPMI 傳輸層所帶來的額外負擔（overhead）可能讓這種做法得不償失。

## 背景與參考

請先閱讀 IPMI Blob 協定設計作為[入門導讀](https://github.com/openbmc/phosphor-ipmi-blobs/blob/master/README.md)。

在底層，二進位 blob 是以二進位的
[protocol buffer](https://github.com/protocolbuffers/protobuf)，簡稱「protobuf」的方式儲存。

## 需求

1. BMC 映像檔應已安裝 `phosphor-ipmi-blobs`。
1. 宿主應知道它打算操作的特定 blob base id。就本設計而言，探索（discovery）操作是有限的。
1. 宿主只應儲存適合透過此傳輸機制處理的二進位資料。如前所述，它並不適合用於信箱通訊、金鑰儲存空間或大型二進位資料。

## 建議設計

本節說明處理程序 `phosphor-ipmi-blobs-binarystore` 如何定義 IPMI Blob 協定的各個處理常式（handler）。

### Blob ID 定義

「blob id」是用來識別某個 blob 的唯一字串。Binary Store 處理常式可能會顯示兩種 blob id：「base id」與「file id」。它們只應包含 ASCII 英數字母字元與正斜線（'/'）。

「base id」以正斜線開頭並以正斜線結尾，類比於 Unix 的目錄路徑。binary store 處理常式會為每個儲存位置指派一個唯一的 base id（詳細說明請見下一節）。

「file id」以正斜線開頭，但結尾必須沒有斜線，類比於 Unix 的檔案路徑。任何以其最長相符前綴為有效 base id 的 file id，都會被視為在該儲存空間中保留為一個二進位 blob。

舉例來說，若 `/test/` 與 `/test/blob0` 是初始的 blob id，我們就知道有一個二進位儲存位置，且其中已建立一個 blob。若要建立一個名為 `/test/blob1` 的新 blob，只需用該 id 呼叫 open，再以回傳的 session id 進行 write/commit 即可。開啟無效的 id，例如 `/foo/bar` 或 `/test/nested/dir`，將會失敗。

### 平台設定

對於 binary store 處理常式，設定檔會提供 base id、要存放資料的檔案，以及在該檔案中的偏移位置（offset）。此外可以選擇性地指定一個「max_size」參數，用以表示此二進位儲存的總大小不應超過該限制。若「max_size」指定為 -1 或未指定，則儲存空間可以增長到實體儲存媒體所能容許的上限。

```none
base_id: /bmc_store/
sysfile_path: /sys/class/i2c-dev/i2c-1/device/1-0050/eeprom
offset: 256
max_size: 1024
```

[1] 設定範例

### Binary Store Protobuf 定義

資料以一個二進位 protobuf 儲存，其中包含數量可變的多個二進位 blob，每個 blob 都有一個唯一的 blob_id 字串，且這些 blob_id 都以 base id 作為共同的前綴。

```none
message BinaryBlob {
  optional string blob_id = 1;
  optional bytes data = 2;
}

message BinaryBlobStore {
  optional string blob_base_id = 1;
  repeated BinaryBlob blob = 2;
  optional uint32 max_size = 3;
}
```

將資料以 protobuf 儲存，能讓格式更具彈性與可擴充性，並允許日後對儲存格式進行修改。

### IPMI Blob 傳輸指令原語

binary store 處理常式將實作以下原語（primitives）：

#### BmcBlobGetCount/BmcBlobEnumerate

起初，在列舉現存的 blobs 時只會出現 base id。一旦某個有效的二進位資料成功 commit 之後，它的 blob id 就會出現在清單中。

#### BmcBlobOpen

`flags` 可以是代表唯讀存取的 `READ`，或是 `READ|WRITE`。`blob_id` 可以是任何具有相符前綴的字串。如果目前還沒有一個以所提供 `blob_id` 存放的有效二進位資料，處理常式會將之視為建立此 blob 的請求。

回傳的 `session_id` 應由其餘基於 session 的指令使用，以對該 blob 進行操作。如果已有一個開啟中的 session，此呼叫將會失敗。

注意：新建立的 blob 在 `BmcBlobCommit` 被呼叫之前，不會被序列化並儲存。

#### BmcBlobRead

回傳指定偏移位置與大小所對應的位元組。若沒有足夠的位元組，處理常式會回傳所有可用的位元組。

請注意，此操作是從記憶體讀取。務必確認狀態為 'COMMITTED'，這表示記憶體內容與序列化到儲存裝置的資料一致。

#### BmcBlobWrite

將位元組寫入指定的偏移位置。成功時回傳寫入的位元組數，失敗時回傳零。

#### BmcBlobCommit

將序列化後的 BinaryBlobStore 儲存到關聯的系統檔案。

#### BmcBlobClose

將該 session 標記為已關閉。任何尚未 commit 的 blob 狀態變更都會遺失。

#### BmcBlobDelete

刪除與 `blob_id` 關聯的二進位資料。刪除 base_id（即「目錄」層級）會無害地失敗。

#### BmcBlobStat

回傳的 `size` 等於 protobuf 中 `data` 位元組的長度。`blob_state` 會依情況設定為 `OPEN_R`、`OPEN_W`，及／或 `COMMITTED`。

#### BmcBlobSessionStat/BmcBlobWriteMeta

不支援。

### 宿主指令流程範例

#### 尚無二進位資料，寫入資料

1. 先 `BmcBlobGetCount`，接著 `BmcBlobEnumerate`。由於還沒有存放有效二進位資料的 blob，BMC 處理常式只會依照平台設定填入 `base_id`，例如 `/bmc_store/`。
1. 以 `blob_id = /bmc_store/blob0` 呼叫 `BmcBlobOpen`，BMC 會接受該請求並回傳 `session_id = 0`。
1. 多次呼叫 `BmcBlobWrite`，將資料寫入該 blob。
1. 呼叫 `BmcBlobCommit`。BMC 會將資料寫入設定的路徑，例如寫入 EEPROM。
1. 呼叫 `BmcBlobClose`

#### 讀取既有資料

1. 先 `BmcBlobGetCount`，接著 `BmcBlobEnumerate`，會顯示 `/bmc_store/` 與 `/bmc_store/blob0`。
1. 對 `/bmc_store/blob0` 呼叫 `BmcBlobStat`，會顯示非零的大小以及 `COMMITTED` 狀態。
1. 以 `blob_id = /bmc_store/blob0` 呼叫 `BmcBlobOpen`。
1. 多次呼叫 `BmcBlobRead` 來讀取資料。
1. 呼叫 `BmcBlobClose`。

## 曾考慮的替代方案

最初考慮的替代方案是透過 IPMI FRU 指令儲存資料；如問題說明中所述，這並不一定可行。

有一個 Google OEM 的 I2C-over-IPMI 驅動程式，可讓宿主讀寫連接在 BMC 上的 I2C 裝置。相較之下，此處提出的 blob store 方法提供更多抽象化，並且在資料存放位置上更具彈性。

## 影響

**_安全性_**：如前所述，binary store 傳輸機制不提供加密功能。透過此機制儲存資料的各方應了解這一點，並選擇不要傳輸涉及安全性的關鍵資料，或是在其上自行提供安全機制。

**_BMC 效能_**：由於處理常式需要 protobuf 套件，若先前未安裝該套件，可能會增加 BMC 映像檔的大小。

**_宿主相容性_**：若資料已使用 IPMI blob binary store 儲存，則宿主將需要一個能理解 blob 傳輸語意的處理常式才能讀取該資料。

## 測試

在可行的範圍內，會使用可模擬（mockable）的介面來對程式碼的邏輯進行單元測試。
