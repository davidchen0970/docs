# 透過 Redfish 進行韌體更新

作者：Andrew Geissler (geissonator)

建立日期：2019-02-11

## 問題說明

OpenBMC 正逐步採用 [Redfish][1] 作為其帶外管理（out of band management）的標準。
Redfish 有自己更新系統韌體的 API 與方法，實作這些內容將需要一些變更（以及
與 DMTF 潛在的上游協作）。

本設計文件的目標，是說明要把 OpenBMC 現有的韌體更新實作轉換到 Redfish
所需要進行的變更。

## 背景與參考資料

OpenBMC 現有的韌體更新細節可參考[架構文件][2]。它使用一套自訂的 REST api
來上傳並接著啟用（activate）輸入的映像檔。

用於韌體更新的 Redfish schema 可參考 [DMTF schemas][3]。
請注意，所參考的文件指向最新版本的 schema，那才是你要載入瀏覽器的內容
（在撰寫本文時為 [v1_4_0][4]）。

Redfish API 與 OpenBMC 現有 API 之間的一些差異：

- Redfish 只有一套上傳與更新的 API。OpenBMC 則有透過一個 API 上傳映像檔、
  再用另一個 API 啟用的概念。
- Redfish 不支援將多個韌體映像檔關聯到同一個目標（target）。OpenBMC 確實
  支援這個概念（例如當某個目標像 BMC 有多個關聯的 flash 晶片時）。已於
  [Issue 3357][10] 與 DMTF 就此展開討論。
  - OpenBMC 有一個優先權（priority）的概念，讓使用者可以挑選與某個目標
    關聯的映像檔來啟用。這讓使用者能夠在單一目標的多個映像檔之間輕鬆來回
    切換，而不必每次重新上傳每個映像檔。
- Redfish 不支援刪除韌體映像檔（當新映像檔上傳時會預設發生刪除）。
- Redfish 支援用同一個指令以同一個映像檔更新多個目標。OpenBMC 目前不支援
  這個功能。
- Redfish 透過 SimpleUpdate 動作提供支援，讓使用者可以傳入各式各樣的遠端
  位置來抓取檔案（FTP、HTTP、SCP……）。現有的 OpenBMC 只支援 TFTP。
- Redfish 能夠排定何時套用韌體更新（Immediate、OnReset、
  AtMaintenanceWindowStart……）。現有的 OpenBMC 韌體沒有這個概念。

許多正在實作 Redfish 的公司選擇為其韌體更新實作建立 OEM 指令
（[ref a][5]、[ref b][6]）。

OpenBMC 內的 Redfish 韌體更新已在 [bmcweb][7] 中展開，本設計將以它為基礎
提出增強方案。

## 需求

分成兩個部分。第一部分是為了下一次 OpenBMC 2.7 發布應完成的項目。另一部分
是未來會完成的項目，但不保證能在 2.7 發布中完成。2.7 的目標是要與現有
OpenBMC REST api 所提供的功能達到對等。

### 2.7 需求

- 在 UpdateService 下支援 BMC 與 BIOS 的 FirmwareInventory 定義
  - 在 Managers/BMC 下支援 FirmwareVersion
  - 在 Systems/system 下支援 BiosVersion
- 使用現有 Redfish API（/redfish/v1/UpdateService）支援所有受支援目標
  （BMC、BIOS）的韌體更新
  - 請注意，經與 DMTF 進一步討論後，現有的 UpdateService 被視為 OEM。
    [Issue 3296][11] 處理了這個問題，並將在核准後由 OpenBMC 實作。既有 API
    仍會在 Redfish 規範中持續受支援，因此 OpenBMC 2.7 發布將支援它。
  - 必須在更新期間就使用者的映像檔提供狀態資訊
- 支援 ApplyTime 的一個子集（Immediate、OnReset）
- 支援以 TFTP 為基礎的 SimpleUpdate
  - 這必須可在 bmcweb 中透過編譯選項進行設定（啟用/停用）

### 未來需求

**注意：** TODO：本節的目標是在較高層級上記錄 2.7 發布之後所需的內容。
在這些需求實作之前，必須先更新本設計文件。將它們列在此處是為了確保基礎
設計不會阻礙日後實作這些功能的能力。

- 支援 [PR 3420][12] 中定義的新概念，以便為單一目標支援多個韌體映像檔。
- 支援 [Issue 3296][11] 中定義的新 UpdateService 韌體更新實作
- 支援其他目標（電源供應器、穩壓器……）的韌體更新
- 支援一次以同一個韌體映像檔更新多個目標
- 支援排定更新發生的時間（維護時段 Maintenance windows）
- 支援 UpdateService 中其餘的 TransferProtocolTypes（CIFS、FTP、SFTP、
  HTTP、HTTPS、NSF、SCP、NFS）
  - TODO：任何本身不具傳輸安全的更新機制（例如 FTP）都需要一個次级的驗證
    機制。具有傳輸安全的更新機制則需要一個可以調整受信任根憑證的方式。
- 支援 Task schema，以便在上傳與啟用階段向使用者提供進度

## 建議設計

### 更新映像檔

更新的偽流程為：

```bash
Discover UpdateService location
HttpPushUri = GET https://${bmc}/redfish/v1/UpdateService
POST ApplyTime property in
  UpdateService/HttpPushUriOptions->HttpPushUriApplyTime object
  (Immediate or OnReset)
POST <image> https://${bmc}/${HttpPushUri}
```

這會在後端引發以下動作：

- 接收映像檔
- 將它內部複製到檔案系統中所需的位置（/tmp/images）
- 等待來自 /xyz/openbmc_project/software 的 InterfacesAdded 訊號
- 啟用新的映像檔
- 若 ApplyTime 為 Immediate，則重新啟動 BMC 或主機
- 若 ApplyTime 為 OnReset，則不做任何事（使用者需自行發起主機或 bmc 重開
  才能套用映像檔）

請注意，ApplyTime 屬性會在啟用結束時由軟體管理堆疊處理。請注意，ApplyTime
屬性對所有韌體更新套件而言是全域共用的，並會被用於所有後續的韌體更新套件。

### 映像檔的狀態

使用者需要知道其韌體映像檔的狀態，尤其在更新期間。Task 概念在 DMTF 內仍
尚在定義中，將來會在上傳與啟用期間提供映像檔的完整狀態。與軟體庫存項目
關聯的 Status 物件將被用來作為向使用者提供庫存項目狀態的機制。以下是
[phosphor 啟用狀態][13] 對應到 [Redfish Status States][14] 的對照表。

```text
NotReady   -> Disabled
Invalid    -> Disabled
Ready      -> Disabled
Activating -> Updating
Active     -> Enabled
Failed     -> Disabled
```

### 變更映像檔的優先權

在為單一目標支援多個 flash 映像檔的 OpenBMC 系統上，仍會透過 Redfish 對此
提供偽支援。第一個映像檔會被上傳到系統、寫入 flash 晶片 0 並啟用。若上傳
另一個映像檔，它會被寫入 flash 晶片 1 並啟用。第一個映像檔仍存在於 flash
晶片 0 中，並可由使用者重新啟用。

如 DMTF issue [3357][10] 所定義，使用 Redfish Settings 物件內的
ActiveSoftwareImage 概念，讓使用者選取在下一個啟用時段要啟用的映像檔。
OpenBMC 內部有為與目標關聯的每個映像檔指定優先權的概念。OpenBMC 韌體將
持續這樣做，但只會把「ActiveSoftwareImage」映像檔設為優先權 0（最高），
以確保它在下一個啟用時段被啟用。將優先權設為 0 後，軟體管理器會自動視需要
更新其他映像檔的優先權。

### 以遠端映像檔下載為基礎的更新

如上所述，Redfish 在 UpdateService 下支援 SimpleUpdate 物件。SimpleUpdate
schema 支援多種傳輸協定（CIFS、FTP、TFTP……）。OpenBMC 現有的後端只支援
TFTP，因此一開始透過 Redfish 在 OpenBMC 上也只會支援 TFTP。

Redfish API 接受一個參數 ImageURI，其中同時包含伺服器位址資訊與檔案名稱。
OpenBMC 上的後端軟體管理器介面需要兩個參數，即 TFTP 伺服器位址與檔案名稱，
因此需要做一些解析。

更新的偽流程為：

```bash
# Discover SimpleUpdate URI Action location
GET https://${bmc}/redfish/v1/UpdateService

# Configure when the new image should be applied
POST ApplyTime property in
  UpdateService/HttpPushUriOptions->HttpPushUriApplyTime object
  (Immediate or OnReset)

# Request OpenBMC to download from TFTP server and activate the image
POST https://${bmc}/redfish/v1/UpdateService/Actions/UpdateService.SimpleUpdate
    [ImageURI=<tftp server ip>/<file name>, TransferProtocol=TFTP]
```

TFTP 是不安全的協定。它沒有關聯的使用者名稱或密碼，也沒有對輸入 URL 的
驗證。OpenBMC 確實有簽章映像檔支援，且預設是更新流程的一部分。請求此更新的
使用者必須具有管理權限，因此假設他們知道自己在做什麼以及 TFTP 所能提供的
安全等級。

由於 TFTP 的安全性疑慮，此功能在 bmcweb 中預設為停用，但可透過編譯旗標啟用
（詳情請參閱 bmcweb 存放庫內的 CMakeLists.txt）。

### 刪除映像檔

將不提供刪除映像檔的支援（直到 DMTF 提供為止）。當上傳新映像檔時，它們
預設沒有優先權。當它們被啟用時，會被賦予最高優先權，且所有其他映像檔的
優先權會視需要更新。當 flash 檔案系統沒有更多空間時，最低優先權的映像檔
會在新映像檔上傳並啟用時被刪除。

## 曾考慮的替代方案

可以乾脆建立看起來像 OpenBMC 目前自訂 REST api 的 Redfish OEM api。這樣的
移植工作量很小，但 OpenBMC 就不會符合任何標準的 Redfish API。其他廠商已經
這麼做，但 Redfish DMTF 正朝著強化 UpdateService 的方向努力，以納入打造
企業級韌體更新 API 所需的功能。OpenBMC 應該保持在這些 DMTF 變更的最前線，
並確保它們在發布時就加以實作。

## 影響

這在內部將使用既有的 D-Bus API 進行韌體更新，因此與先前的 REST 實作之間
應該只有最小的影響。

OpenBMC 有兩套韌體更新的實作。不同的系統實作了不同的版本。說明清楚一點，
本設計將使用 [Software Version Management][8] 實作背後的 D-Bus API。

## 測試

正針對 BMC 與 BIOS 韌體的韌體更新撰寫端對端[測試][9]，這些測試必須通過。
此外也必須撰寫單元測試，並將所需的 D-Bus API 進行 mock，以確保適當的程式碼
涵蓋率。

[1]: https://redfish.dmtf.org/
[2]:
  https://github.com/openbmc/docs/blob/master/architecture/code-update/code-update.md#steps-to-update
[3]: http://redfish.dmtf.org/schemas/v1/UpdateService.json
[4]:
  http://redfish.dmtf.org/schemas/v1/UpdateService.v1_4_0.json#/definitions/UpdateService
[5]: https://www.supermicro.com/manuals/other/RedfishRefGuide.pdf
[6]:
  https://github.com/dell/iDRAC-Redfish-Scripting/blob/master/Redfish%20Python/DeviceFirmwareDellUpdateServiceREDFISH.py
[7]:
  https://github.com/openbmc/bmcweb/blob/master/redfish-core/lib/update_service.hpp
[8]:
  https://github.com/openbmc/phosphor-dbus-interfaces/blob/master/yaml/xyz/openbmc_project/Software/README.md
[9]: https://github.com/openbmc/openbmc-test-automation
[10]: https://github.com/DMTF/Redfish/issues/3357
[11]: https://github.com/DMTF/Redfish/pull/3296
[12]: https://github.com/DMTF/Redfish/pull/3420
[13]:
  https://github.com/openbmc/phosphor-dbus-interfaces/blob/master/yaml/xyz/openbmc_project/Software/Activation.interface.yaml
[14]: http://redfish.dmtf.org/schemas/v1/Resource.json#/definitions/Status
