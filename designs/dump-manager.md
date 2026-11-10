# Dump Manager 設計

作者：Dhruvaraj Subhashchandran <dhruvaraj@in.ibm.com>

其他貢獻者：

建立日期：12/12/2019

## 問題說明

當發生當機或主機故障時，事件監控機制會產生一筆錯誤紀錄，但錯誤紀錄的大小限制在
幾 KB 之內，因此當機或故障時的所有資料可能無法全部放入錯誤紀錄中。除錯所需的
額外資料必須以 dump 的形式收集。現有的 OpenBMC dump 介面僅支援在 BMC 上產生的
dump，且 dump manager 不支援下載操作。

## 字彙表

- **System Dump**：主機主記憶體與暫存器的 dump。
  [Read More](https://en.wikipedia.org/wiki/Core_dump)
- **Memory Preserving Reboot(MPR)**：一種在重新開機時保留記憶體內容的方法。
- **PLDM**：一種用於存取底層平台清單、監控、控制、事件，以及資料／參數傳輸
  功能的介面與資料模型。
  [ReadMore](https://github.com/openbmc/docs/blob/master/designs/pldm-stack.md)
- **Machine Check Exception**：處理器核心內部的嚴重錯誤，會使該處理器核心停止
  所有處理活動。
- **BMCWeb**：OpenBMC 的嵌入式網頁伺服器。
  [More Info](https://github.com/openbmc/bmcweb/blob/master/README.md)

## 背景與參考資料

根據故障的類型與來源，會建立各種不同類型的 dump。負責協調收集與卸載（offload）
動作的 dump manager，必須提供建立 dump、儲存 dump 詳細資訊，以及將其卸載的方法。
此外，某些來源允許在沒有發生故障的情況下手動擷取 dump，以了解目前狀態或分析
疑似的問題。

### 支援的 dump 類型

以下是 dump manager 所支援的部分 dump 類型。

#### BMC Dump

當 BMC 發生故障時，連同各種除錯資訊一起收集的 dump。使用者也可以自行產生這種
dump，以取得 BMC 的目前狀態。這種 dump 會在 BMC 上收集並儲存在 BMC 上。

#### System Dump

System dump 是從主機收集的除錯資訊集合，可能包含主機記憶體及／或暫存器資料。
這種 dump 可以由 BMC 發起，且在收集 dump 的過程中可能會重新啟動主機。dump 會
儲存在主機記憶體中並透過 BMC 卸載，或直接收集到 BMC 上，端視 dump 內容的大小
以及 BMC 上可用於儲存 dump 的空間而定。

#### Resource dump

一種特殊的主機 dump，由主機根據使用者的請求來發起並收集。在收集這種 dump 的
過程中，可能不需要改變系統狀態。可以使用 resource indicator 來指出要收集的資料。
dump 的內容可由主機決定。這種 dump 可以儲存在主機記憶體中並透過 BMC 卸載，或由
主機在收集完成後，根據 dump 的大小以及 BMC 上可用空間的狀況，將此 dump 傳送到
BMC。

#### Hostboot dump

在主機開機失敗時可以收集的 dump。這種 dump 可能包含、也可能不包含主記憶體及／或
處理器暫存器的內容。

#### Hardware dump

這種 dump 可以在主機已開機並執行期間、當硬體元件（例如處理器）發生嚴重故障時
收集。在此 dump 期間主機可能會停止運作，並收集各種處理器狀態及／或記憶體內容，
以協助除錯該故障。

## 需求

![Dump use cases - Users are examples, not a mandatory part of implementation](https://user-images.githubusercontent.com/16666879/70888651-d8f44080-2006-11ea-8596-ed4c321cfaa6.png)

### Dump manager 需提供介面以下列用途

- 建立 dump：根據錯誤條件或使用者請求，發起 dump 的建立。
- 列出 dump：列出 BMC 上所有現存的 dump。
- 取得 dump：將 dump 卸載至外部實體。
- 通知：通知 dump manager 有新的 dump 已建立。
- 刪除 dump。
- 將 dump 標記為已卸載至外部實體。
- 設定 dump 政策，例如停用某種型別的 dump，或 dump 覆寫政策。

## 建議設計

dump 有多種不同的類型；大多數 dump 的介面都是標準化的，但無法儲存在 BMC 上的
大型 dump 則需要額外支援。本文件將說明各種 dump 類型的設計。dump 的分類依據是
它在哪裡收集、在哪裡儲存，以及如何擷取。兩大主要類型為：

- 由 BMC 收集並儲存在 BMC 上。
- 在相連的實體上收集並儲存，但透過 BMC 卸載。

本提案的重點在於重複使用現有的
[phosphor-debug-collector](https://github.com/openbmc/phosphor-debug-collector)，
它負責從 BMC 收集 dump。

![phosphor-debug-collector](https://user-images.githubusercontent.com/16666879/72070844-7b56c980-3310-11ea-8d26-07d33b84b980.jpeg)

### 現有 phosphor-debug-collector 的設計重點摘要

- 一個 create 介面，假設型別為 BMC dump，並針對使用者發起的 dump 回傳一個 ID
  給呼叫端。
- 外部的 dump 請求會被視為使用者發起的 BMC dump，並以名為 dreport 的工具、
  以 manual dump 型別來發起 BMC dump 的收集。
- dreport 會在 dump manager 程式碼所提供的 dump 路徑中建立 dump 檔案。
- dump manager 會 fork 出一個 watch 程序，用以捕捉該路徑上檔案關閉的信號，
  並據以假設 dump 收集已完成。
- watch 程序會呼叫內部的 dbus 介面，以帶著大小與時間戳記建立 dump 項目。
- dump 的路徑以預定義的基礎路徑加上 dump 的 id 為基礎。
- 當收到卸載請求時，檔案會從 dump 基礎路徑 +id 下載，且不會更新該項目中是否
  已卸載的狀態。
- 刪除 dump 時會刪除該項目，內部檔案也會一併刪除。
- 有基於錯誤紀錄或 core dump 的系統產生型 dump，其運作方式與使用者發起的
  dump 類似，但有以下差異。
- 不需要外部的 create D-Bus 介面，但會有一個監控應用程式去監控特定條件，
  例如 core 檔案的存在，或若干精選類型之新錯誤紀錄建立的信號。
- 一旦事件發生，監控程式會呼叫內部的 D-Bus 介面 dump manager，以該 dump 的
  型別來建立 dump。
- dump manager 會以監控程式取得的 dump 型別呼叫 dreport，並將資料寫入以
  dump id 為基礎的路徑。

### 針對現有 phosphor-debug-collector 提出的更新

- 外部的 D-Bus 介面需要指明 dump 的型別，因為使用者可以請求多種型別的 dump。
- Create 會回傳一個 id，當實際的 dump 建立完成後可與之對應。
- 提供一個 Notify 介面，用於通知在 BMC 之外建立、但透過 BMC 卸載的 dump。
- 將實作 InitiateOffload 功能以下載 dump。
- dump 的狀態（是否已卸載）將被加入 dump 項目中。

### Dump manager 介面

- Dump Manager DBus 物件提供建立與管理 dump 的介面。

- 介面
  - **Create**：建立 dump 的介面，由客戶端呼叫以發起使用者發起的 dump。
    - AdditionalData：發起 dump 時的額外資料（若有）。此處的 key 應為針對
      xyz 或某個 domain 中特定 dump 型別所定義的、實作特定的 enum。其值可以是
      字串或 64 位元數字。enum 格式的字串必須來自與此特定 Enum 名稱平行的類別。
      所有的 Enum 字串都應採用
      'domain.Dump.Create.CreateParameters.ParamName' 的格式。例如：
      { "key1": "value1", "key2": "value2" } 在 AdditionalData 中會變成
      ["KEY1=value1", "KEY2=value2"]。

  - **Notify**：通知 dump manager 有新的 dump 已建立。
    - ID：dump 的 ID，若不為 0，則此值將為 dump 的外部 id。
    - Type：所建立 dump 的型別。
    - Size：dump 的大小。

### Dump 項目介面

    -  **InitiateOffload**：發起 dump 的卸載。
        - OffloadUri：dump 應被卸載到的 URI。

#### 所有 dump 共有的屬性

將有一個基礎的 dump 項目，帶有所有型別 dump 共有的屬性。

- ID：dump 的 Id。
- Timestamp：dump 的建立時間戳記。
- Size：dump 的總大小。
- OffloadComplete：當卸載完成時設為 true。
- OffloadURI：用於卸載 dump 的 URI，在發起卸載時設定。特定型別需要從這個共用的
  dump 項目類別繼承，並加入其專屬的屬性。

#### 依 dump 型別而定的額外屬性

##### BMC Dump

- 無額外屬性。

##### System Dump

- External Source ID：由主機提供的 ID，這個 id 將用於與 dump 來源（在此例為
  主機）之間的所有溝通。

### 在主機上收集並儲存的 dump 流程

這裡以 PLDM 作為主機與 BMC 之間 dump 傳輸與通知機制的範例。

- Create：發起在主機上建立 dump 的方法。
- 在主機上產生 dump。
- 主機透過 PLDM 向 BMC 通知 dump 的建立。
- PLDM 呼叫 Notify 以建立 dump 項目。
- InitiateOffload：dump manager 請求主機開始卸載。
- 主機透過 PLDM 傳送 dump，而 BMC 上的 PLDM 再將其對外送出。

## 曾考慮的替代方案

- 透過主機而非 BMC 來卸載主機 dump，但最終仍選擇 BMC 的方案，理由如下——在多數
  伺服器上，BMC 被視為「管理路徑」，而主機往往並未連接到卸載位置所需的網路。
  ——對於系統中產生的所有 dump，BMC 為外部管理設備提供了一個統一的匯集點。

## 影響

- 需要重複使用現有的 BMC dump 介面。目前的介面不接受 dump 型別，因此會為 BMC
  dump 另提供一個帶有型別的新 create 介面，而不更動現有的介面。
- 修改 BMC dump 基礎架構以支援額外的 dump。
- openpower-proc-control 將被更新，以呼叫 memory preserving chip-ops，並在
  POWER 平台上處理 memory preserving reboot。
- 於執行 memory preserving reboot 期間，增加一個新的系統狀態來表示系統正在
  收集除錯資料。

## 測試

- 單元測試，以確保 dump manager 的介面運作正常。
- 將執行以下整合測試，以確保 dump manager 如預期般運作。——測試建立主機 dump
  並將其卸載。——測試刪除主機 dump——建立／列出／卸載／刪除 BMC dump，以確保
  現有的 dump manager 功能未被破壞。
- 針對 dump 的 Create／List／Offload／Delete 進行自動化測試，以避免功能迴歸。
