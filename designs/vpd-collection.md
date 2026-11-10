# OpenBMC 上的 VPD 收集應用程式

作者：Santosh Puranik <santosh.puranik@in.ibm.com> <santosh.puranik>

建立日期：2019-06-11

## 問題描述

在 OpenBMC 上，目前重要產品資料（Vital Product Data，VPD）的收集僅限於一兩個可場勘更換單元（Field Replaceable Units，FRU）——其中一個例子就是 BMC FRU。在 OpenPower 系統上，BMC 也只支援一種 VPD 格式，也就是 [OpenPower VPD] [1] 格式。作為其企業級伺服器的一部分，IBM 將使用 IPZ 格式的 VPD，而 BMC 目前並不支援這種格式。某些 FRU 也具有關鍵字（keyword）格式的 VPD。

BMC 因為多项原因而需要讀取所有 FRU 的 VPD：

- 部分 VPD 資訊，例如 FRU 料號（part number）、序號（serial number），必須包含在平台錯誤紀錄（Platform Error Logs，PEL）中，以便在維修時指出特定的 FRU。

- BMC 上有若干使用案例需要應用程式依據所插入 FRU 的層級來決定某個程式碼路徑。舉例來說，控制電壓調節器（voltage regulator）的應用程式可能需要根據電壓調節器 FRU 的版本來設定不同的暫存器。

- 有些使用案例需要 BMC 將 VPD 資料傳送至主機端的管理程式／作業系統（透過 PLDM）。這主要是為了讓主機能夠取得其無法直接存取的 VPD。

VPD 資料本身可能存放在 EEPROM（常見情況），也可能是由 FRU 的某些參數合成而來（不常見——針對沒有 EEPROM 的 FRU）。

本設計文件旨在為 OpenBMC 上的 VPD 收集與 VPD 存取定義一套高階設計。它_不_涵蓋低階的 API 細節。

## 背景與參考資料

從本質上來說，IPZ VPD 結構由稱為關鍵字（keyword）的鍵值對組成。每個關鍵字可用來包含關於 FRU 的特定資料。舉例來說，SN 關鍵字會包含一個序號，用以唯一識別某個料件的實例。關鍵字本身可以被合併或分組成記錄（record）。舉例來說，可以使用單一記錄來分組具有相似功能的關鍵字，例如序號、料號。

關鍵字格式的 VPD 不包含記錄，而是只有關鍵字一個接一個地排列。

IPZ 格式在其結構上與 OpenPower 格式相當類似，除了以下幾點細節：

- IPZ VPD 有不同的記錄與關鍵字。

- IPZ VPD 要求實作並驗證 OpenPower 規格中所定義的 ECC。BMC 程式碼目前並未驗證／使用 ECC，儘管規格確實有定義它，但針對 IBM 的企業級伺服器將需要使用 ECC。

- 關鍵字格式的 VPD 同樣也是由鍵值對組成，但沒有用以將關鍵字分組在一起的記錄概念。關鍵字格式 VPD 的 ECC 只是一個 CRC。

## 需求

以下是 OpenBMC 上 VPD 功能的需求：

- BMC 必須在它進入待機（Standby，又稱 Ready 狀態，即 BMC 可從此狀態開始進行 CEC 開機）之前，為所有它能直接存取的 FRU 收集 VPD。這基於多項原因是必要的：

  BMC 應用程式需要能夠讀取 FRU 的 VPD，以判斷（例如）FRU 的硬體層級。

  部分 VPD 需要與主機交換。

  製造與維修工程師需要能在不開機的情況下檢視及寫入 FRU VPD。

  某些系統層級的 VPD 也會被 BMC 上的應用程式用來判斷它所執行於其上的系統類型與型號。

- 若某個 FRU 沒有 VPD 儲存裝置（例如 EEPROM），BMC 應該能夠為此類 FRU 合成 VPD。關於 VPD 合成的細節將會放在它自己的設計文件中，此處不涵蓋。

- 當 BMC 正在執行時，BMC 應該能夠為可熱插拔或可場勘維修的 FRU 重新收集 VPD。

- 若有任何 VPD 無法正確解析或未通過 ECC 檢查，BMC 必須記錄錯誤。

- BMC 必須為它收集 VPD 的所有 FRU 建立／更新 FRU 庫存（inventory）物件。庫存 D-Bus 物件必須包含（除其他細節外）FRU 的序號、料號與 CCIN（IBM 用來區分同一 FRU 不同版本的方式）。

- BMC 上的應用程式必須能夠查詢某個特定 FRU 的任一指定 VPD 關鍵字。

- 應用程式也需要能夠更新 FRU 中的 VPD 內容。舉例來說，在包含系統 VPD 的 FRU 於現場被更換的情況下，BMC 需要能夠更新系統 VPD 的能力。

- 需要有一個工具／API，能允許檢視及更新任何指定 FRU 的 VPD。這包含 BMC 並非直接為其收集 VPD 的 FRU（例如可由 BMC 與主機兩者存取、但由主機收集 VPD 的 FRU）。

## 建議設計

本文件涵蓋架構、介面與設計細節。它為實作提供了建議，但實作細節不在本文件的範圍之內。

此處的提案是建立在 open power 現行使用的 VPD 收集設計之上。目前的實作包含以下幾項實體：

- [op-vpd-parser] [2] 服務，它會解析包含 OpenPower VPD 格式 VPD 的 EEPROM 內容。

- 一條 udev [rule] [3]，由 systemd 用來在 EEPROM 裝置接上時啟動上述服務。

- 一組設定 [files] [4]，用來描述針對某個指定 FRU 該更新哪些庫存物件與 D-Bus 屬性。

為了滿足前一節所列的需求，將會進行以下更新：

- 將建立兩個新的服務來處理新的 VPD 格式：ipz-vpd-parser 與 keyword-vpd-parser。這些服務將以範本（template）方式設計，以允許執行多個實例。

- 將建立另一個服務，用以使用位置碼（location code）資訊來更新庫存。由於位置碼展開來自系統 VPD，因此這個服務只能在地系統 VPD（存放在背板上）收集完成之後才能啟動。

- 系統上每個 BMC 必須收集 VPD 的 EEPROM 各會有一條 udev rule。我們也可以只設一條 rule，但那代表我們必須以某種方式過濾掉非 VPD 的 EEPROM。

- 每條 udev rule 都會設定為啟動其中一個 VPD 解析服務的一個實例（任何指定 EEPROM 中 VPD 的格式在建置時期即已知，因為它們是系統專屬的）。

- 該服務（ipz-vpd-parser 或 keyword-vpd-parser 的一個實例）在啟動後，將讀取 EEPROM、解析其內容，並使用設定檔來決定要在庫存中存放哪些 VPD 內容。

- 該服務將以下列格式使用 VPD 的內容來更新庫存 D-Bus 物件：每個記錄會有一個介面（例如 VINI 記錄），而該介面會將每個關鍵字作為一個屬性（例如 FN、PN）。這將讓我們得以支援可能具有相同關鍵字的多個記錄，同時也是一種在庫存中對關鍵字進行邏輯分組的方式，與它們在實際 VPD 中的分組方式相當類似。舉例來說（此處有些名稱是虛構的，但它們有助於說明要點），對於包含關鍵字 SN、FN 與 CCIN 的 VINI 記錄，其在 D-Bus 上的表示方式將如下所示：

```text
Interface: com.ibm.ipzvpd.VINI
Properties:
    SN
    FN
    CCIN
```

- 就關鍵字格式的 VPD 而言，所有關鍵字都將作為屬性放在單一介面之下。

- 解析服務在更新上述屬性之前，不會以任何方式格式化或轉換 VPD 中的資料，這些屬性將以位元組陣列（byte array）的形式儲存。不過請注意，這些服務仍會持續將常用的 FRU 屬性（例如 SerialNumber、PartNumber）更新為字串，就如同 openpower-vpd-parser 所做的那樣。

- 為了處理 VPD 寫入，在所有 VPD 讀取服務完成之後將啟動另一個 systemd 服務。這個服務將是一個常駐程式（daemon），負責管理對 EEPROM 的平行寫入。VPD 寫入服務將暴露 D-Bus 介面，用以在給定庫存路徑的情況下更新某個 FRU 的 VPD。

- udev rule 與設定檔的產生將採分層方式設計，以便能針對每個系統進行調整。

### 待決事項

一些尚未決定的問題：

- 對於併行維護（concurrent maintenance，即在主機開機並運行的情況下更換 FRU）將如何處理，還需要更多思考。這將會在它自己的設計文件中提出。

## 曾考慮的替代方案

曾考慮以下幾種替代設計：

### 撰寫一個獨立的 VPD 伺服器應用程式

曾考慮的選項之一是在 BMC 上開發一個獨立、萬能的 VPD 應用程式，在 BMC 進入待機之前收集所有 VPD。該應用程式必須是一個常駐程式，將暴露一組 D-Bus 介面以：

- 收集／重新收集所有 VPD。
- 查詢某個指定 FRU 的 VPD。例如讀取 VRM 的序號。
- 更新某個指定 FRU 的 VPD 關鍵字。
- 對某個 FRU 進行併行維護，這將依序執行移除／新增／更換的操作。

該應用程式將由一份設定檔驅動，該設定檔列出某個指定系統上可用的 FRU、它們的 I2C 匯流排位址、從位址等。

此選項因以下原因被否決：

- 這樣的設計會讓該應用程式過於特定於某個系統或某一組 VPD 格式。雖然該應用程式可以撰寫成允許插入對不同 VPD 格式的支援，但經評估認為，目前這種針對非常特定需求的小型應用程式的做法較佳。
- 此設計未能善用其所選方案讓我們得以採取的分層設計方法。

### 建立在 entity manager 之上

使用 entity manager：<https://github.com/openbmc/entity-manager>。Entity manager 有一個稱為 FruDevice 的應用程式，它會探測 /dev/i2c/ 以尋找 EEPROM、讀取（IPMI 格式）VPD 並將其存放在 D-Bus 上。

可以增強該應用程式以：

- 加入對其他 VPD 格式的支援，例如 IPZ 與關鍵字格式。
- 或許將不同的一組資料更新到不同的 D-Bus 物件，例如 Inventory manager。
- 變更用來讀寫 FRU 資料的外部 D-Bus 介面，使其改為接受庫存路徑（而非目前所接受的 I2C 路徑、位址）。

此選項因以下原因被否決：

- 我們並不需要 entity manager 所提供的全方位功能，也就是說我們不想取代現有的 inventory manager。讓 Power 系統改用其他方式取代 inventory manager，不在本文件的範圍之內。
- 該程式碼看起來不太具有良好的可插拔性來加入對新 VPD 格式的支援，我們最終可能不得不使用 #ifdef 來區隔功能。
- 沒有辦法為 FRU 裝置判斷系統藍圖（blueprint），它會掃描整個 /dev/ 樹狀結構來挑出 EEPROM。

## 影響

已識別出以下影響：

- 用於解析 VPD 並將其存放在庫存中的服務，會為 BMC 開機流程增加一些時間。應透過在這些服務的啟動上達成最大限度的平行化，將影響降到最低。
- 需要讀取 VPD 關鍵字的應用程式會受到影響，因為它們將必須使用庫存介面來取得它們感兴趣的資料。

## 測試

可以透過在檔案系統上模擬（fake out）VPD EEPROM 檔案來測試 VPD 解析功能。這類測試也能確保正確的一組 VPD 資料進入 OpenBMC 庫存。此外也有一項提案要在應用程式中內建檔案模式（file mode）。檔案模式不需要真實硬體即可測試程式碼功能，但可以使用 BMC 上的檔案來模擬 EEPROM。

可以透過撰寫一個小型命令列工具，呼叫 VPD 寫入應用程式的 API 來寫入 VPD，藉以測試 VPD 寫入。

[1]:
  https://www-355.ibm.com/systems/power/openpower/posting.xhtml?postingId=1D060729AC96891885257E1B0053BC95
[2]:
  https://github.com/openbmc/meta-openpower/blob/master/recipes-phosphor/vpd/openpower-fru-vpd/op-vpd-parser.service
[3]:
  https://github.com/openbmc/meta-openpower/blob/master/recipes-phosphor/vpd/openpower-fru-vpd/70-op-vpd.rules
[4]:
  https://github.com/openbmc/meta-openpower/blob/master/recipes-phosphor/vpd/openpower-fru-vpd-layout/layout.yaml
