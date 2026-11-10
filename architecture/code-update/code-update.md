# OpenBMC 程式碼更新

有兩種 BMC 程式碼佈局可供選擇：

- 靜態、非 UBI 佈局
- UBI 佈局 - 透過 `obmc-ubi-fs` distro feature 啟用

本文件說明同時支援這兩種佈局的程式碼更新。

## 更新步驟

以下是更新 BMC 的步驟。

1. 取得 BMC 映像 tar 檔：建置 OpenBMC 之後，您會在
   `tmp/deploy/images/<platform>/` 中得到一組映像檔。
   - UBI 佈局映像為
     `obmc-phosphor-image-<platform>-<timestamp>.ubi.mtd.tar`
   - 靜態佈局映像為
     `obmc-phosphor-image-<platform>-<timestamp>.static.mtd.tar`

   BMC tar 映像包含 5 個檔案：u-boot、kernel、ro 與 rw 分割區，以及 MANIFEST
   檔案。MANIFEST 檔案含有映像相關的資訊，例如映像用途、版本、KeyType（用於簽章的金鑰
   類型）、HashType（用於金鑰產生的 SHA 類型）以及 MachineName（建置映像時所使用的機器
   名稱，將用於驗證映像的建置）。MANIFEST 檔案可能如下所示：

   ```text
   purpose=xyz.openbmc_project.Software.Version.VersionPurpose.BMC
   version=2.7.0-dev
   KeyType=OpenBMC
   HashType=RSA-SHA256
   MachineName=tiogapass
   ```

2. 透過下列其中一種方法將產生的 BMC 映像傳輸到 BMC：
   - 方法 1：透過 Redfish 上傳：
     <https://github.com/openbmc/docs/blob/master/REDFISH-cheatsheet.md#firmware-update>.
     若使用此方法，請直接跳到步驟 5！
   - 方法 2：透過 scp：將產生的 BMC 映像複製到 BMC 上的 `/tmp/images/`
     目錄。
   - 方法 3：透過 REST 上傳：
     <https://github.com/openbmc/docs/blob/master/rest-api.md#uploading-images>
   - 方法 4：透過 TFTP：對 `/xyz/openbmc_project/software` 發送 POST 請求以呼叫
     `DownloadViaTFTP` 方法。

   方法 3 與方法 4 需要在 bmcweb 中啟用額外選項。

3. 記錄該映像檔所產生的 version id。version id 是一個由 8 個十六進位數字組成的
   雜湊值，其產生方式為對映像中所含的版本字串進行 SHA-512 雜湊，並取其前 8
   個字元。透過下列其中一種方法取得 version id：
   - 方法 1：從 BMC 命令列，記錄在 `/tmp/images/` 下建立的最新目錄名稱，在此例中
     為 `2a1022fe`：

   ```text
   # ls -l /tmp/images/
     total 0
     drwx------    2 root     root            80 Aug 22 07:54 2a1022fe
     drwx------    2 root     root            80 Aug 22 07:53 488449a2
   ```

   - 方法 2：此方法在傳輸映像開始時沒有任何 `Ready` 映像的情況下才*有效*。使用
     REST API，記錄其 Activation 屬性設為 Ready 的物件，在此例中為
     `2a1022fe`：

   ```text
     $ curl -b cjar -k https://${bmc}/xyz/openbmc_project/software/enumerate
     {
       "data": {
         "/xyz/openbmc_project/software/2a1022fe": {
           "Activation": "xyz.openbmc_project.Software.Activation.Activations.Ready",
   ```

   - 方法 3：事先從映像計算 version id，指令如下：

   ```bash
     tar xfO <BMC tar image> MANIFEST | sed -ne '/version=/ {s/version=//;p}' | head -n1 | tr -d '\n' | sha512sum | cut -b 1-8
   ```

4. 若要啟動更新，請將目標映像的 `RequestedActivation` 屬性設為 `Active`，並將
   `<id>` 取代為前一個步驟中所記錄的雜湊值。這會透過下列其中一種方法將映像的內容寫入
   BMC 晶片：
   - 方法 1：從 BMC 命令列：

   ```bash
     busctl set-property xyz.openbmc_project.Software.BMC.Updater \
       /xyz/openbmc_project/software/<id> \
       xyz.openbmc_project.Software.Activation RequestedActivation s \
       xyz.openbmc_project.Software.Activation.RequestedActivations.Active

   ```

   - 方法 2：使用 REST API：

   ```bash
     curl -b cjar -k -H "Content-Type: application/json" -X PUT \
       -d '{"data":
       "xyz.openbmc_project.Software.Activation.RequestedActivations.Active"}' \
       https://${bmc}/xyz/openbmc_project/software/<id>/attr/RequestedActivation
   ```

5. （選擇性）檢查刷寫進度。此介面僅在啟用過程中可用，啟用完成後即不存在，可透過
   下列其中一種方式檢查：
   - 方法 1：從 Redfish：Redfish 上傳會回傳一個 task，可使用該 task 來監控進度。

   ```bash
    curl -k https://${bmc}/redfish/v1/TaskService/Tasks/0
   ```

   - 方法 2：從 BMC 命令列：

   ```bash
     busctl get-property xyz.openbmc_project.Software.BMC.Updater  \
       /xyz/openbmc_project/software/<id> \
       xyz.openbmc_project.Software.ActivationProgress Progress
   ```

   - 方法 3：使用 REST API：

   ```bash
     curl -b cjar -k https://${bmc}/xyz/openbmc_project/software/<id>/attr/Progress
   ```

6. 透過下列其中一種方法驗證 "Activation" 屬性已設為 "Active"，以確認啟用已完成：
   - 方法 1：從 Redfish：檢查 Redfish 上傳所回傳的 task。

   ```bash
     curl -k https://${bmc}/redfish/v1/TaskService/Tasks/0
   ```

   - 方法 2：從 BMC 命令列：

   ```bash
     busctl get-property xyz.openbmc_project.Software.BMC.Updater \
       /xyz/openbmc_project/software/<id> \
       xyz.openbmc_project.Software.Activation Activation
   ```

   - 方法 3：使用 REST API：

   ```bash
     curl -b cjar -k https://${bmc}/xyz/openbmc_project/software/<id>
   ```

7. 重新啟動 BMC，讓映像生效。
   - 方法 1：從 Redfish：若 ApplyTime 設為 "Immediate"，BMC 將自動重新啟動：
     <https://github.com/openbmc/docs/blob/master/REDFISH-cheatsheet.md#firmware-applytime>.
     若需手動重新啟動 BMC，請參閱：
     <https://github.com/openbmc/docs/blob/master/REDFISH-cheatsheet.md#bmc-reboot>.

   - 方法 2：從 BMC 命令列：

   ```bash
     reboot
   ```

   - 方法 3：使用 REST API：

   ```bash
     curl -c cjar -b cjar -k -H "Content-Type: application/json" -X PUT \
         -d '{"data": "xyz.openbmc_project.State.BMC.Transition.Reboot"}' \
         https://${bmc}/xyz/openbmc_project/state/bmc0/attr/RequestedBMCTransition
   ```

## 關聯性（Associations）

除了所有軟體映像之外，`/xyz/openbmc_project/software/` 下還列有數個關聯性：

```bash
curl -c cjar -b cjar -k -H "Content-Type: application/json" -X GET  \
    https://${bmc}/xyz/openbmc_project/software/
{
  "data": [
    "/xyz/openbmc_project/software/46e65782",
    "/xyz/openbmc_project/software/493a00ad",
    "/xyz/openbmc_project/software/88c153b1",
    "/xyz/openbmc_project/software/active",
    "/xyz/openbmc_project/software/functional"
  ],
  "message": "200 OK",
  "status": "ok"
}
```

1. 指向「執行中（running）」BMC 與 host 映像的 "functional" 關聯性

   每個 BMC 只有一個 functional 關聯性，每個 host 也只有一個 functional
   關聯性。functional/running 的 BMC 映像是指在重新啟動 BMC 時優先級最低的那個 BMC
   映像。functional 映像不會更新，直到 BMC 重新啟動為止。functional 的 host
   映像行為相同，差別在於它會在 host 開機或重新啟動時更新。

   ```bash
   curl -c cjar -b cjar -k -H "Content-Type: application/json" -X GET \
       https://${bmc}/xyz/openbmc_project/software/functional
   {
     "data": {
       "endpoints": [
         "/xyz/openbmc_project/software/46e65782",
         "/xyz/openbmc_project/software/493a00ad"
       ]
     },
     "message": "200 OK",
     "status": "ok"
   }
   ```

2. 指向作用中（active）BMC 與 host 映像的 "active" 關聯性

   注意：可能有多個 BMC 映像處於 active 狀態，host 映像也是如此。

   ```bash
   curl -c cjar -b cjar -k -H "Content-Type: application/json" -X GET \
       https://${bmc}/xyz/openbmc_project/software/active
   {
     "data": {
       "endpoints": [
         "/xyz/openbmc_project/software/46e65782",
         "/xyz/openbmc_project/software/493a00ad",
         "/xyz/openbmc_project/software/88c153b1"
       ]
     },
     "message": "200 OK",
     "status": "ok"
   }
   ```

3. 指向可程式化元件的 "updateable" 關聯性

   這用於識別可透過 BMC 的 OOB 介面（例如 Redfish/IPMI）進行程式燒錄的韌體元件。所有
   可更新的韌體元件都必須暴露 updateable 關聯性，讓上層應用程式（例如
   Redfish/IPMI）能夠知道有哪些可更新的韌體。

   若要了解可更新的軟體元件：

   ```bash
    busctl call xyz.openbmc_project.ObjectMapper \
     /xyz/openbmc_project/software/updatable org.freedesktop.DBus.Properties \
     Get ss xyz.openbmc_project.Association endpoints
   v as 1 "/xyz/openbmc_project/software/1201fc36"
   ```

   Redfish 介面在 SoftwareInventory schema 中使用 'updateable' 關聯性。

4. 另有一個關聯性位於 `/xyz/openbmc_project/software/<id>/inventory`，用於將軟體映像
   與某個 inventory 項目「關聯（associating）」在一起。

   ```bash
   curl -c cjar -b cjar -k -H "Content-Type: application/json" -X GET \
      https://${bmc}/xyz/openbmc_project/software/493a00ad/inventory
   {
     "data": {
       "endpoints": [
         "/xyz/openbmc_project/inventory/system/chassis/motherboard/boxelder/bmc"
       ]
     },
     "message": "200 OK",
     "status": "ok"
   }
   ```

   若要取得與某個 inventory 項目相關聯的所有軟體映像：

   ```bash
   curl -c cjar -b cjar -k -H "Content-Type: application/json" -X GET  \
       https://${bmc}/xyz/openbmc_project/inventory/system/chassis/activation
   {
     "data": {
       "endpoints": [
         "/xyz/openbmc_project/software/46e65782"
       ]
     },
     "message": "200 OK",
     "status": "ok"
   }
   ```

## MANIFEST 檔案

任何上傳、透過 TFTP 下載或複製到 BMC 的映像 tar 檔中，都必須包含一個名為 "MANIFEST"
的檔案。

MANIFEST 檔案格式必須為 key=value（例如 version=v1.99.10）。它應包含下列欄位：

- version - 映像的版本
- purpose - 映像的用途（例如
  xyz.openbmc_project.Software.Version.VersionPurpose.BMC 或
  xyz.openbmc_project.Software.Version.VersionPurpose.Host）。可接受的 purpose
  值可在
  [Version interface](https://github.com/openbmc/phosphor-dbus-interfaces/blob/6f69ae5b33ee224358cb4c2061f4ad44c6b36d70/xyz/openbmc_project/Software/Version.interface.yaml)
  中的 "VersionPurpose" 值下找到。
- MachineName - 此映像所針對的機器（平台）名稱。此值會與執行中映像的 os-release
  檔案中所定義的 OPENBMC_TARGET_MACHINE 值進行比較。若此檢查失敗，映像將不會被升級。基於
  向後相容性，對於目前釋出的映像，若 MachineName 未定義，此檢查會略過失敗情形，但自
  2.9 版起的釋出將把它設為必填欄位。

其他選擇性欄位為：

- extended_version - 更詳細的版本，可能包含映像中各個不同元件的版本。

## 刪除映像

若要刪除映像：

```bash
curl -c cjar -b cjar -k -H "Content-Type: application/json" \
    -X POST https://${bmc}/xyz/openbmc_project/software/<$id>/action/delete \
    -d "{\"data\": [] }"
```

注意：該映像必須為非 functional（「非執行中」）。

若要刪除所有非 functional 的映像，無論是 BMC 或 host 映像：

```bash
curl -c cjar -b cjar -k -H "Content-Type: application/json" \
    -X POST https://${bmc}/xyz/openbmc_project/software/action/deleteAll \
    -d "{\"data\": [] }"
```

## 軟體 Field Mode

Field mode 適用於從製造端出貨給客戶的系統。Field mode
提供了一種方式來強化安全性，並確保不會意外地在系統上載入錯誤的修補程式。field mode
介面的軟體實作會透過不掛載 `/usr/local` 來停用 BMC
的修補，進而停用位於 `/usr/local/share/pnor/` 的 host
修補。啟用 field mode 預期是一個單向操作，這表示一旦啟用，便不會提供可將停用的 REST
API。

可透過執行下列命令啟用 field mode：

```bash
curl -b cjar -k -H 'Content-Type: application/json' -X PUT -d '{"data":1}'  \
    https://${bmc}/xyz/openbmc_project/software/attr/FieldModeEnabled

```

雖然 field mode 預期為單向操作，但具有管理員權限的使用者可透過在 BMC
上執行下列命令將其停用：

```bash
fw_setenv fieldmode

systemctl unmask usr-local.mount

reboot
```

有關 field mode 的更多資訊可在這裡找到：
<https://github.com/openbmc/phosphor-dbus-interfaces/blob/master/yaml/xyz/openbmc_project/Control/FieldMode.interface.yaml>

## 軟體原廠重設

軟體原廠重設會透過清除所有讀寫資料，將 BMC 與 host 韌體重設回其原廠狀態。若要執行軟體原廠重設，請
執行下列命令，然後重新啟動 BMC：

```bash
curl -b cjar -k -H 'Content-Type: application/json' -X POST -d '{"data":[]}' \
    https://${bmc}/xyz/openbmc_project/software/action/Reset

```

BMC 端的原廠重設會清除 `/var`、`/home` 與 `/etc`。在 host
端，原廠重設會清除系統上每個 host 映像的讀寫磁碟區、清除共用的 preserve host 磁碟區
pnor-prsv，並清除位於 `/usr/local/share/pnor/` 的所有 host 修補程式。

原廠重設介面可在這裡找到：
<https://github.com/openbmc/phosphor-dbus-interfaces/blob/02b39246d45ea029a1652a49cc20eab7723dd63b/xyz/openbmc_project/Common/FactoryReset.interface.yaml>

## 映像儲存位置

### 靜態佈局

當 BMC 映像被啟用時，每個 `image-<name>` 都會被寫入 BMC 晶片中由 `<name>` 所指示的分割區：

- image-u-boot
- image-kernel
- image-rofs
- image-rwfs

### UBI 佈局

當 BMC 映像被啟用時（也就是當 "RequestedActivation" 設為 "Active"
時），會在 BMC 晶片上為該映像建立 UBI 磁碟區。備援（alternate）BMC
晶片也可用於儲存映像。這由 "BMC_RO_MTD" 決定。同時使用備援 BMC 晶片與 BMC
晶片可讓多個 BMC 映像得以儲存。預設情況下僅使用 BMC 晶片。若要同時使用兩者，請將
"BMC_RO_MTD" 設為 "alt-bmc+bmc"。

## 實作

關於程式碼更新實作的更多資訊可在以下位置找到：
<https://github.com/openbmc/phosphor-dbus-interfaces/blob/master/yaml/xyz/openbmc_project/Software>
以及 <https://github.com/openbmc/phosphor-bmc-code-mgmt>
