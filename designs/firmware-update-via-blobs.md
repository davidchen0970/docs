# 使用 OEM IPMI Blob 傳輸方式進行 BMC 韌體（及其他）的帶內更新

作者：Patrick Venture <venture!>

建立日期：2018-10-18

## 問題描述

BMC 需要一種機制，能透過多種方式從主機接收新的韌體映像檔。最好的做法是使用單一協定，並讓多種做法都能路由到這個協定之中。

## 背景與參考

BMC 硬體至少會提供某種用於傳送與接收 IPMI 訊息的介面。此硬體也可能提供可記憶體映射（memory mapped）的區域，以便 BMC 與主機之間進行更高速的通訊。某些基礎架構並未提供 BMC 的網路存取，因此必須提供一種能在主機與 BMC 之間以帶內（in-band）方式執行的更新機制。

這裡所說的帶內，指的是直接連接於主機與 BMC 之間的通訊通道。

1. Serial
1. IPMI over LPC
1. IPMI over i2c
1. LPC Memory-Mapped Region
1. P2A bridge

## 入門

請閱讀 IPMI BLOB 協定設計作為[入門](https://github.com/openbmc/phosphor-ipmi-blobs/blob/master/README.md)。

## 需求

以下陳述反映最初的需求。

- 任何更新機制都必須同時支援 UBI tarball 與傳統（靜態佈局）的 flash 映像檔。善用 BLOB 協定，只要實作處理某種映像檔類型的機制，系統就能支援該類型。

- 任何更新機制都必須能在映像檔被使用之前，觸發一個映像檔驗證步驟。

- 任何更新機制都必須能以不同的帶內機制來實作資料暫存（staging）。

- 任何更新機制都必須提供一個 handshake 或等效的協定，用以協調資料傳輸。例如，BMC 是否應該啟用 P2A bridge 以及要使用哪個區域，或者是否要開啟 LPC 記憶體映射 bridge。

- 任何更新機制都必須盡力維持安全性，例如預設不要開放某個記憶體區域。舉例來說，在開始驗證步驟之前，必須確保主機仍然無法存取已暫存的韌體映像檔。

## 擬議設計

OpenBMC 支援一個提供基礎原語（primitives）的 BLOB 協定。這些原語讓多種不同的「handler」得以存在，針對特定的「blob」實作這些原語。在此情境下，blob 是一個嚴格唯一（strictly unique）的檔案路徑。

透過 BLOB 協定傳送韌體映像檔，將是把 [phosphor-ipmi-flash 設計](https://github.com/openbmc/phosphor-ipmi-flash/blob/master/README.md)路由到一個 BLOB handler 來完成。這旨在取代 `phosphor-ipmi-flash` 目前的作法，以集中到單一具彈性的 handler。

### 序列控制

為了強制執行序列控制，本設計要求同一時間只能開啟一個 blob。如果驗證 blob 已開啟，則其他 blob 就無法開啟；相對地，如果某個 client 已開啟一個資料 blob，則驗證 blob 就無法開啟。

### 定義 Blob

BLOB 協定允許 handler 指定一份 blob id 清單。將利用這份清單來指明平台是否支援傳統（靜態佈局）機制、UBI 機制，或兩者皆支援。提供給 open 命令的 flags 用以辨識由 client 端所選取的機制。stat 命令會回傳該 blob 所支援的機制清單。

各機制對應的 blob id 如下：

| Flash Blob Id    | 類型            |
| ---------------- | --------------- |
| `/flash/image`   | Static Layout   |
| `/flash/tarball` | UBI             |
| `/flash/bios`    | Host BIOS image |

flash handler 會根據提供給 open 的 flags，針對所開啟的 blob 決定它應該預期收到哪些命令、以及會回傳哪些回應。

flash handler 在一組命令序列中只允許開啟上述其中一個 blob，因此你無法在未完成（或稍後中止）第一個已啟動的更新程序之前，先開啟 `/flash/image` 接著又開啟 `/flash/bios`。

以下 blob id 定義為用來儲存映像檔的雜湊值：

| Hash Blob     | Id 機制                    |
| ------------- | ------------------------------- |
| `/flash/hash` |  whichever flash blob 已開啟 |

flash handler 一次只允許一個開啟的檔案，因此如果主機嘗試透過 IPMI BlockTransfer 傳送韌體映像檔，它不會允許主機在 BlockTransfer 檔案關閉之前啟動 PCI 傳送。

只有一種雜湊「檔案」機制。所使用的確切雜湊值僅對你的驗證服務有意義。所提供的值將被寫入一個已知的位置。

當傳輸進行中時，它會建立一個 blob_id `/flash/active/image` 與 `/flash/active/hash`。

#### 驗證 Blob

一旦映像檔或雜湊值的上傳開始後，便會定義以下 blob id。其用途在於觸發並監控韌體驗證程序。因此，在雜湊與映像檔檔案都關閉之前，BmcBlobOpen 命令都會失敗。關於理想的命令序列請見後文。

| Trigger Blob    | 說明                     |
| --------------- | ------------------------ |
| `/flash/verify` | Verify Trigger Mechanism |

當驗證檔案關閉時，如果驗證已成功完成，它會新增一個更新 blob id，定義如下。

本設計並不定義所使用的驗證程序。

#### 更新 Blob

當 `/flash/verify` 以有效的映像檔或 tarball 關閉後，更新 blob id 便可用。需要開啟更新 blob 並對該 blob id 呼叫 commit() 以觸發更新機制。

可以定期對更新 blob id 呼叫 stat() 來檢查更新程序。

| Update Blob     | 說明                     |
| --------------- | ------------------------ |
| `/flash/update` | Trigger Update Mechanism |

本設計並不定義所使用的更新程序。

#### 清理 Blob

清理 blob id 永遠存在。此 blob 的目標是在失敗或成功時處理更新產物（update artifacts）的刪除。它可以實作成做任何需要的清理，但對於記憶體壓力較大的系統而言，它是一個方便的清理機制。

清理 blob 沒有狀態也沒有知識，旨在提供一個簡單的系統清理機制。這也可以透過對 BMC 執行暖重啟（warm reboot）來達成。清理 blob 會刪除一份檔案清單。清理 blob 對更新程序沒有狀態辨識能力，因此可能會干擾正在進行的更新程序。主機端工具只會在失敗的情況下使用它。任何其他開發的工具都應該尊重這點，除非目標是清理產物，否則不應使用它。

要觸發清理，只要開啟 blob、commit 並 close 即可。它對更新程序沒有任何認知。這樣的簡化是透過將它設計成一個便利機制、而非必要機制來達成。

| Cleanup Blob     | 說明                      |
| ---------------- | ------------------------- |
| `/flash/cleanup` | Trigger Cleanup Mechanism |

### 快取映像檔

與 OEM IPMI Flash 協定類似，flash 映像檔將被暫存在一個編譯期設定的位置。

可以透過在 handler 中新增更多 blob id 或 flags，輕易地加入其他機制。

### 命令

更新機制會根據所選取的傳輸機制，預期一個特定的命令序列。有些機制需要 handshake。

#### BlockTransfer 序列

1. Open（用於 Image 或 tarball）
1. Write
1. Close
1. Open（`/flash/hash`）
1. Write
1. Close
1. Open（`/flash/verify`）
1. Commit
1. SessionStat（用以讀回驗證狀態）
1. Close
1. Open（`/flash/update`）
1. Commit
1. SessionStat（用以讀回更新狀態）
1. Close

#### P2A 序列

1. Open（用於 Image 或 tarball）
1. SessionStat（用於 P2A 映射的 P2A Region）
1. Write
1. Close
1. Open（`/flash/hash`）
1. SessionStat（P2A Region）
1. Write
1. Close
1. Open（`/flash/verify`）
1. Commit
1. SessionStat（用以讀回驗證狀態）
1. Close
1. Open（`/flash/update`）
1. Commit
1. SessionStat（用以讀回更新狀態）
1. Close

#### LPC 序列

1. Open（用於 image 或 tarball）
1. WriteMeta（指定來自主機、供 LPC 使用的區域資訊）
1. SessionStat（驗證上述內容）
1. Write
1. Close
1. Open（`/flash/hash`）
1. WriteMeta（LPC Region）
1. SessionStat（驗證 LPC 設定）
1. Write
1. Close
1. Open（`/flash/verify`）
1. Commit
1. SessionStat（用以讀回驗證狀態）
1. Close
1. Open（`/flash/update`）
1. Commit
1. SessionStat（用以讀回更新狀態）
1. Close

### 過期映像檔

如果一個映像檔更新程序已啟動但變得過期（stale），有多種機制可確保清理。如果一個 session 在 blob 逾時期間之後仍未關閉，它將被關閉。由於過期與關閉並非同一種動作，快取將被清除，且任何已暫存的片段都會被刪除。

在傳統（靜態佈局）模式下，映像檔本身將被放置並命名成某種方式，使得 BMC 重啟後它就會消失。在 UBI 的情況下，檔案會被存放在 `/tmp` 並適當地刪除。

在上傳程序的任何時點，都可以透過關閉已開啟的 blob 並依名稱刪除它們來中止。

### Blob 原語

更新機制將如下實作 Blob 原語。

#### BmcBlobOpen

blob open 原語允許提供 blob 專屬的 flags。這些 flags 用以指定傳輸機制。若要取得平台上支援的機制清單，請見下方的 `Stat` 命令。

```cpp
enum OpenFlags
{
    read = (1 << 0),
    write = (1 << 1),
};

/* These bits start in the blob specific range of the flags. */
enum FirmwareUpdateFlags
{
    bt = (1 << 8),   /* Expect to send contents over IPMI BlockTransfer. */
    p2a = (1 << 9),  /* Expect to send contents over P2A bridge. */
    lpc = (1 << 10), /* Expect to send contents over LPC bridge. */
};
```

一個 open 請求必須指明它是以寫入方式開啟並指定一個傳輸機制，否則會被拒絕。如果請求同時設為讀取，這不會被拒絕，但目前不提供額外價值。

一旦開啟後，blob_id 清單中會出現一個新檔案（對映像檔與雜湊皆是），表示它們正在進行中。名稱將是 `flash/active/image` 與 `flash/active/hash`，除了代表目前進行中的更新之外別無意義。關閉檔案並不會刪除已暫存的映像檔，只有 delete 才會。

**_注意_** 進行中的映像檔 blob_id 無法被開啟。這在之後可以重新考慮。

#### BmcBlobRead

這最初不會執行任何功能，並會回傳成功且 0 位元組。

#### BmcBlobWrite

write 命令的內容將取決於傳輸機制。此命令在將資料從映射區域複製到暫存緩衝區、或寫入到暫存檔案之前，不得返回。命令如何從映射區域讀取資料，超出本設計的範圍。

##### 若為 BT

負載（payload）的資料區塊純粹是資料。

##### 若為 P2A

負載的資料區塊是以下結構：

```cpp
struct ExtChunkHdr
{
    uint32_t length; /* Length of the data queued (little endian). */
};
```

##### 若為 LPC

負載的資料區塊是以下結構：

```cpp
struct ExtChunkHdr
{
    uint32_t length; /* Length of the data queued (little endian). */
};
```

#### BmcBlobCommit

如果此命令是在韌體映像檔本身的 session 上呼叫，目前不會有任何動作，它會回傳一個 no-op 成功。

如果此命令是在雜湊映像檔的 session 上呼叫，目前不會有任何動作，它會回傳一個 no-op 成功。

如果此命令是在 verify blob id 的 session 上呼叫，它會觸發一個 systemd 服務 `verify_image.service` 來嘗試驗證映像檔。在執行這一步之前，如果傳輸機制不是 IPMI BT，它會關閉用於傳輸的機制，以防止主機更新任何內容。

當這一步啟動後，只有 BmcBlobSessionStat 命令會回應。關於該回應的細節請見下方 BmcBlobSessionStat。

#### BmcBlobClose

在開啟 verify blob 之前，必須對韌體映像檔與雜湊檔案呼叫 Close。

如果 `verify_image.service` 回傳成功，關閉 verify 檔案會根據更新類型而有特定的行為。如果是 UBI，它會執行安裝；如果是傳統（靜態佈局），它不會做任何事。在傳統情況下，verify_image 服務負責將檔案放到正確的暫存位置。一個 BMC 暖重啟命令將啟動韌體更新程序。

如果映像檔驗證失敗，它會自動刪除與該更新相關的任何檔案。

**_注意：_** 在開發測試期間，開發者會想要上傳未簽章的檔案。因此，會在 flags 中新增一個額外的位元來改變這個行為。

#### BmcBlobDelete

中止任何進行中的更新：

1. 如果已啟動，則停止 verify_image.service。
1. 刪除任何已暫存的檔案。

如果更新已經進行中，例如使用了 tarball 機制且正在更新檔案的中途，則無法中止。

#### BmcBlobStat

對某個 blob_id 做 blob stat（而非 SessionStat）會回傳該 blob_id handler 的能力。

```cpp
struct BmcBlobStatRx {
    uint16_t crc16;
    /* This will have the bits set from the FirmwareUpdateFlags enum. */
    uint16_t blob_state;
    uint32_t size; /* 0 - it's set to zero when there's no session */
    uint8_t  metadata_len; /* 0 */
};
```

#### BmcBlobSessionStat

如果在 commit 之前呼叫，它會回傳以下資訊：

```cpp
struct BmcBlobStatRx {
    uint16_t crc16;
    uint16_t blob_state; /* OpenFlags::write | (one of the interfaces) */
    uint32_t size; /* Size in bytes so far written */
    uint8_t  metadata_len; /* 0. */
};
```

如果呼叫時資料傳輸機制是 P2A，它會回傳一個 32 位址，作為 `BmcBlobStatRx` 中 metadata 區塊的一部分，用以設定 P2A 區域。

```cpp
struct BmcBlobStatRx {
    uint16_t crc16;
    uint16_t blob_state; /* OpenFlags::write | (one of the interfaces) */
    uint32_t size; /* Size in bytes so far written */
    uint8_t  metadata_len = sizeof(struct P2ARegion);
    struct P2ARegion {
        uint32_t address;
    };
};
```

如果在 verify 檔案 session 上於 commit 之後呼叫，它會回傳：

```cpp
struct BmcBlobStatRx {
    uint16_t crc16;
    uint16_t blob_state; /* OPEN_W | (one of the interfaces) */
    uint32_t size; /* Size in bytes so far written */
    uint8_t  metadata_len; /* 1. */
    uint8_t  verify_response; /* one byte from the below enum */
};

enum VerifyCheckResponses
{
    VerifyRunning = 0x00,
    VerifySuccess = 0x01,
    VerifyFailed  = 0x02,
    VerifyOther   = 0x03,
};
```

如果在 update 檔案 session 上於 commit 之後呼叫，它會回傳：

```cpp
struct BmcBlobStatRx {
    uint16_t crc16;
    uint16_t blob_state; /* OPEN_W | (one of the interfaces) */
    uint32_t size; /* Size in bytes so far written */
    uint8_t  update_response; /* one by from the below enum */
};

enum UpdateStatus
{
    UpdateRunning = 0x00,
    UpdateSuccessful = 0x01,
    UpdateFailed = 0x02,
    UpdateStatusUnknown = 0x03
};
```

`UpdateStatus` 與 `VerifyCheckResponses` 目前內容相同，但隨著時間可能會改變。

#### BmcBlobWriteMeta

write metadata 命令旨在讓主機能夠為帶內更新提供特定的設定資料給 BMC。目前這僅針對 LPC，它需要被告知記憶體位址，才能設定視窗（window）。

write meta 命令的 blob 將會是這個結構：

```cpp
struct LpcRegion
{
    uint32_t address; /* Host LPC address where the chunk is to be mapped. */
    uint32_t length; /* Size of the chunk to be mapped. */
};
```

## 已考慮的替代方案

Google 目前使用另一種實作，它善用相同的機制，然而它沒那麼有彈性，因為每個命令都是一個自訂的部分。將其映射到 blobs 原語，可以在維持向後相容性的同時，讓未來的修改更容易（而不只是新增一個獨立的 OEM 函式庫來處理新程序等）。

## 影響

這會影響安全性，因為它可以善用記憶體映射視窗。預計不會有效能影響，因為 blob handler 的存在僅在 blob enumerate 命令的回應中多產生幾個條目。

## 測試

在可能的情況下（幾乎所有地方），都會使用可 mock 的介面，使得整個程序都有獨立的單元測試，來驗證 flags 是否被檢查，以及狀態與序列是否正確。

### 情境

#### 傳送一個具有壞雜湊值的映像檔

一個必要功能測試是：將一個映像檔傳送到 BMC，但該映像檔的簽章無效。預期結果是驗證步驟會回傳失敗，且檔案會在無需使用者介入的情況下從 BMC 刪除。

#### 傳送一個具有良好雜湊值的映像檔

一個必要功能測試是：將一個具有有效簽章的映像檔傳送到 BMC。預期結果是驗證步驟會回傳成功。

## 設定

請參閱 [Secure Flash Update Mechanism](https://github.com/openbmc/phosphor-ipmi-flash/blob/master/README.md) 的設定章節。
