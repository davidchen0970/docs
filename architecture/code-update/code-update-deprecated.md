# OpenBMC 程式碼更新

**重要說明：**

- 下文所述的 REST API 已棄用（deprecated）。請改遵循
  [code-update.md](code-update.md) 使用新的 API 來進行程式碼更新。
- 本文件的其餘部分仍然有效。

主機（host）的程式碼更新請見此處：
[host-code-update.md](host-code-update.md)

建置 OpenBMC 之後，您會在 `tmp/deploy/images/<platform>/` 中得到一組映像檔。
其中的 `image-*` 符號連結（symlink）對應到 BMC 上可被更新的各個元件：

- `image-bmc` → `obmc-phosphor-image-<platform>-<timestamp>.static.mtd`

  BMC 的完整閃存（flash）映像檔

- `image-kernel` → `fitImage-obmc-phosphor-initramfs-<platform>.bin`

  OpenBMC 核心的 FIT 映像檔（包含核心、裝置樹（device tree）與
  initramfs）

- `image-rofs` → `obmc-phosphor-image-<platform>.squashfs-xz`

  唯讀的 OpenBMC 檔案系統

- `image-rwfs` → `rwfs.jffs2`

  供 OpenBMC 檔案系統存放永久性變更的讀寫檔案系統

- `image-u-boot` → `u-boot.bin`

  OpenBMC 開機載入程式（bootloader）

此外，還會建立兩個 tarball，可透過 REST 部署並解壓：

- `<platform>-<timestamp>.all.tar`

  完整的 BMC 閃存內容：單一檔案（`image-bmc`）包裝在 tar 封存檔中。

- `<platform>-<timestamp>.tar`

  分割區的 BMC 閃存內容：多個檔案包裝在 tar 封存檔中，u-boot、核心、
  唯讀（ro）與讀寫（rw）每個分區各佔一個檔案。

## 為 BMC 程式碼更新做準備

BMC 通常在掛載了讀寫與唯讀檔案系統的情況下執行，這代表這些映像檔在任何時候
都可能被讀取（而讀寫檔案系統還可能被寫入）。由於更新是以完整的檔案系統映像檔
形式來分散佈署，因此必須先卸載（unmount）這些檔案系統，才能將它們替換為新的
映像檔。要卸載這些檔案系統，必須停止所有應用程式。

預設情況下，正常的 `reboot` 會停止所有應用程式並卸載根檔案系統，而複製到
`/run/initramfs` 目錄中的映像檔會在那個時點、重新啟動之前被套用。這同樣適用於
`shutdown` 與 `halt` 指令——它們會在停止之前先寫入閃存。

作為另一種做法，initramfs 中的 `init` 指令腳本可以解析一個選項，將這些檔案系統
所需的內容複製到 RAM 中，如此一来就可以在其餘應用程式堆疊仍在執行時套用映像檔，
並透過網路監控進度。接著可以在系統運作中呼叫 `update` 腳本來寫入映像檔，並監控其
進度輸出。

## 從 OpenBMC shell 更新

若要從 OpenBMC shell 進行更新，請遵循本節的步驟。

建議先為更新準備好 BMC（請注意，由於 U-Boot 具有冗餘環境（redundant
environments），該環境變數需要被設定兩次，initramfs 才能讀取到它）：

    fw_setenv openbmconce copy-files-to-ram copy-base-filesystem-to-ram
    fw_setenv openbmconce copy-files-to-ram copy-base-filesystem-to-ram
    reboot

將一個或多個 `image-*` 檔案複製到此目錄：

    /run/initramfs/

（保留檔名），然後執行 `update` 腳本來套用映像檔：

    /run/initramfs/update

接著重新開機以完成套用：

    reboot

在重新開機的過程中，`update` 腳本會在檔案系統被卸載之後被呼叫，以完成更新程序。

另有一些選用功能可用，詳情請見說明：

    /run/initramfs/update --help

## 透過 REST 更新

OpenBMC 系統可以從 TFTP 伺服器下載更新映像檔，並透過 REST 控制來套用更新。
一般的程序為：

1. 為更新準備系統
2. 設定更新選項
3. 啟動更新
4. 檢查閃存狀態
5. 套用更新
6. 重新啟動 BMC

### 為更新準備系統

執行 POST 以呼叫 `/flash/bmc` 物件的 `PrepareForUpdate` 方法：

    curl -b cjar -k -H "Content-Type: application/json" -X POST \
        -d '{"data":  []}' \
        https://${bmc}/org/openbmc/control/flash/bmc/action/prepareForUpdate

這會設定 u-boot 環境並重新啟動 BMC。若沒有其他待處理的映像檔，BMC 應在約
2 分鐘後回來。

### 設定更新選項

有一些選項可用來控制更新程序：

- `preserve_network_settings`：保留網路設定，只有在更新整個閃存時才需要
- `restore_application_defaults`：更新（清除）讀寫檔案系統
- `update_kernel_and_apps`：更新核心與 initramfs。若將使用分割區 tarball
  來進行更新，則必須設定此選項。否則，若將使用完整 tarball，則不可設定此選項。
- `clear_persistent_files`：在重設應用程式預設值時忽略永久檔案清單
- `auto_apply`：嘗試在映像檔解壓後，透過呼叫 `Apply` 方法來寫入映像檔。

若要設定更新選項，請對 `/control/flash/bmc/attr/<setting>` 執行 REST PUT。
舉例來說：

    curl -b cjar -k -H "Content-Type: application/json" -X PUT \
        -d '{"data": 1}' \
        https://${bmc}/org/openbmc/control/flash/bmc/attr/preserve_network_settings

### 啟動更新

執行 POST 以呼叫 `/flash/bmc` 物件的 `updateViaTftp` 方法：

    curl -b cjar -k -H "Content-Type: application/json" -X POST \
        -d '{"data": ["<TFTP server IP address>", "<filename>"]}' \
        https://${bmc}/org/openbmc/control/flash/bmc/action/updateViaTftp

請注意 `<filename>` 應為一個 tarball。

### 檢查閃存狀態

您可以用一個簡單的 GET 請求來查詢下載與映像檔驗證的進度：

    curl -b cjar -k https://${bmc}/org/openbmc/control/flash/bmc

或執行 POST 以呼叫 `/flash/bmc` 物件的 `GetUpdateProgress` 方法：

    curl -b cjar -k -H "Content-Type: application/json" -X POST \
        -d '{"data": []}' \
        https://${bmc}/org/openbmc/control/flash/bmc/action/GetUpdateProgress

注意：

- 在下載 tarball 期間，進度狀態為 `Downloading`
- 在 tarball 下載並驗證完成後，進度狀態會變成
  `Image ready to apply`。

### 套用更新

若狀態為 `Image ready to apply.`，您可以選擇啟動重新開機，或呼叫 Apply
方法以開始寫入閃存的程序：

    curl -b cjar -k -H "Content-Type: application/json" -X POST \
        -d '{"data": []}' \
        https://${bmc}/org/openbmc/control/flash/bmc/action/Apply

此時映像檔正在被燒錄（flash），您也可以用上述步驟的指令來檢查進度。

- 在燒錄映像檔期間，狀態會變成 `Writing images to flash`
- 在燒錄並驗證完成後，狀態會變成
  `Apply Complete. Reboot to take effect.`

### 重新啟動 BMC

若要開始使用新的映像檔，請使用 BMC 控制物件的 warmReset 方法來重新啟動 BMC：

    curl -b cjar -k -H "Content-Type: application/json" -X POST \
        -d '{"data": []}' \
        https://${bmc}/org/openbmc/control/bmc0/action/warmReset
