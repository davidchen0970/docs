# 主機韌體更新

參考資料：
<https://github.com/openbmc/phosphor-dbus-interfaces/blob/master/yaml/xyz/openbmc_project/Software>

以下是更新主機韌體（或稱「BIOS」）的步驟。這裡假設主機目前並未存取其韌體。

1. 取得 squashfs 映像檔：
   - 建置 op-build：<https://github.com/open-power/op-build>
   - 建置完成後，映像檔應為位於 output/images
     目錄下、名為 `<system type>.pnor.squashfs.tar` 的 tarball

2. 透過下列其中一種方式，將產生的 squashfs 映像檔傳輸到 BMC：
   - 方式 1：透過 scp：將產生的 squashfs 映像檔複製到 BMC 上的
     `/tmp/images/` 目錄。
   - 方式 2：透過 REST 上傳：
     <https://github.com/openbmc/docs/blob/master/rest-api.md#uploading-images>
   - 方式 3：透過 TFTP：發送 POST 請求以呼叫 `/xyz/openbmc_project/software`
     的 `DownloadViaTFTP` 方法。

   ```bash
     curl -b cjar -k -H "Content-Type: application/json" -X POST \
       -d '{"data": ["<filename>", "<TFTP server IP address>"]}' \
       https://${bmc}/xyz/openbmc_project/software/action/DownloadViaTFTP
   ```

3. 記錄該映像檔所產生的版本 id。版本 id 是一個由 8 個十六進位數字組成的雜湊值，
   它是對映像檔中所含的版本字串進行 SHA-512 雜湊運算後取其前 8 個字元而來。
   請透過下列其中一種方式取得版本 id：
   - 方式 1：從 BMC 命令列，記錄在 `/tmp/images/` 下最新建立的目錄名稱，
     在本例中為 `2a1022fe`：

   ```bash
     # ls -l /tmp/images/
     total 0
     drwx------    2 root     root            80 Aug 22 07:54 2a1022fe
     drwx------    2 root     root            80 Aug 22 07:53 488449a2
   ```

   - 方式 2：這個方式*只有*在傳輸映像檔開始時沒有任何處於 `Ready`
     狀態的映像檔時才會生效。使用 REST API，記錄 Activation 屬性被設為 Ready
     的那個物件，在本例中為 `2a1022fe`：

   ```json
     $ curl -b cjar -k https://${bmc}/xyz/openbmc_project/software/enumerate
     {
       "data": {
         "/xyz/openbmc_project/software/2a1022fe": {
           "Activation": "xyz.openbmc_project.Software.Activation.Activations.Ready",
   ```

   - 方式 3：事先從映像檔計算出版本 id，指令為：

   ```bash
     tar xfO <squashfs image tar> MANIFEST | sed -ne '/version=/ {s/version=//;p}' | head -n1 | tr -d '\n' | sha512sum | cut -b 1-8
   ```

4. 若要啟動更新，請將所需映像檔的 `RequestedActivation` 屬性設為 `Active`，
   並將 `<id>` 替換為前一步驟中記錄的雜湊值，這會透過下列其中一種方式，
   將映像檔的內容寫入 PNOR 晶片中的 UBI volume：
   - 方式 1：從 BMC 命令列：

   ```bash
     busctl set-property org.open_power.Software.Host.Updater \
       /xyz/openbmc_project/software/<id> \
       xyz.openbmc_project.Software.Activation RequestedActivation s \
       xyz.openbmc_project.Software.Activation.RequestedActivations.Active

   ```

   - 方式 2：使用 REST API：

   ```bash
     curl -b cjar -k -H "Content-Type: application/json" -X PUT \
       -d '{"data":
       "xyz.openbmc_project.Software.Activation.RequestedActivations.Active"}' \
       https://${bmc}/xyz/openbmc_project/software/<id>/attr/RequestedActivation
   ```

5. （選用）檢查刷寫進度。這個介面只在啟動（activation）進行期間可用，
   一旦啟動完成後便不復存在，請透過下列其中一種方式檢查：
   - 方式 1：從 BMC 命令列：

   ```bash
     busctl get-property org.open_power.Software.Host.Updater \
       /xyz/openbmc_project/software/<id> \
       xyz.openbmc_project.Software.Activation Progress
   ```

   - 方式 2：使用 REST API：

   ```bash
     curl -b cjar -k https://${bmc}/xyz/openbmc_project/software/<id>/attr/Progress
   ```

6. 透過下列其中一種方式，確認 Activation 屬性已被設為 Active，
   以檢查啟動是否完成：
   - 方式 1：從 BMC 命令列：

   ```bash
     busctl get-property org.open_power.Software.Host.Updater \
       /xyz/openbmc_project/software/<id> \
       xyz.openbmc_project.Software.Activation Activation
   ```

   - 方式 2：使用 REST API：

   ```bash
     curl -b cjar -k https://${bmc}/xyz/openbmc_project/software/<id>
   ```

## 修補主機韌體

將分割區的二進位檔複製到 BMC 上的 `/usr/local/share/pnor/`。

該分割區二進位檔的名稱必須與要修補的分割區名稱相同，例如：`ATTR_TMP`。

## Associations、MANIFEST 檔案、刪除映像檔、軟體 Field Mode 與軟體原廠重設

關於 associations、MANIFEST 檔案、刪除映像檔、軟體 field mode
以及軟體原廠重設的資訊，請參考
[code-update.md#associations](code-update.md#associations)

## 實作

<https://github.com/openbmc/openpower-pnor-code-mgmt>
