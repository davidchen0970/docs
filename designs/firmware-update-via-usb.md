# 透過 USB 進行帶內更新 BMC 韌體

作者：George Liu <liuxiwei!>

建立日期：2021-10-12

## 問題說明

當無法使用 Redfish 或 scp 時，BMC 需要一個新的機制讓映像檔能夠進入機器。

## 背景與參考

openbmc 專案目前有一個 [phosphor-software-manager][1] 存放庫（repository）。
為了執行更新，必須先把映像檔帶入 BMC 目錄（/tmp/images）。然而目前僅支援
TFTP 和 HTTP，尚未支援 USB。

這個新應用程式設計的目的，是要讓 BMC 的 USB 驅動程式能夠把新的映像檔帶入
BMC。

## 需求

以下說明反映了最初的需求。

- 監控 USB 隨身碟是否已插入。
- 將 USB 裝置檔案排序清單中找到的第一個 tar 檔案複製到 `/tmp/images`。
- 手動觸發韌體升級。
- 在升級完成後停用 BMC 韌體的自動重新啟動，以避免在插入隨身碟的情況下發生
  潛在的迴圈。
- 此機制試圖維持安全性，例如預設停用此功能，或可透過 Redfish 啟用或停用。

## 建議設計

新程式碼將會是 phosphor-software-manager 存放庫的一部分（例如：
phosphor-usb-code-update）。設計流程如下：

- 在 [phosphor-software-manager][1] 存放庫中定義一個巨集開關
  （`usb-code-update`），用以判斷是否啟用 USB Code Update 功能，預設為
  _啟用_。
- 若啟用了 `usb-code-update`，則在編譯期間將 udev 規則檔案安裝到
  `/lib/udev/rules.d`。
- 一旦符合 udev 規則，便直接觸發 systemd 服務並啟動
  phosphor-usb-code-update 常駐程式（daemon）。
- 此常駐程式驗證 `/run/media/usb/sda1` 目錄，並將目錄中的第一個 `.tar`
  檔案複製到 `/tmp/images`，接著開始驗證。
- 將 ApplyTime 設為 OnReset，使得建議的 usb code update 應用程式在啟用後
  不會重新啟動 BMC。
- 將 RequestedActivation 設為 Active，跟隨更新狀態，開始更新韌體，並在完成後
  重新啟動 BMC。
- 結束 phosphor-usb-code-update 常駐程式。

## 虛擬碼（Pseudocode）

udev 規則檔案範例如下：

```udev
SUBSYSTEM=="block", ACTION=="add", ENV{ID_USB_DRIVER}=="usb-storage", ENV{DEVTYPE}=="partition", ENV{SYSTEMD_WANTS}="usb-code-update@%k", TAG+="systemd"
```

## 安全性

- 建議執行一次本機 CI，並透過 cppcheck 分析並避免潛在的漏洞。
- 假設 USB 磁碟機存在實體安全漏洞（例如記憶體溢位等），應透過 Redfish 停用
  「USB code update」。待漏洞修補完成後，再透過 Redfish 重新啟用「USB code
  update」。

## 曾考慮的替代方案

如果作業系統因錯誤而無法開機，導致無法透過作業系統進行韌體更新，或是網路發生
故障，無法透過 Redfish 或 scp 進行更新，那麼伺服器支援人員就只能拆下 flash
晶片並重新燒錄，這是並不合理的；服務支援應該要能本機存取機器，並將系統更新到
可用的韌體層級。

## 影響

這會影響安全性，因為它可以透過外部 USB 隨身碟將檔案複製到 BMC。預期不會有
效能影響，因為此程序僅在執行期間複製檔案，並在完成後自動結束。

## 測試

- 當 USB code update 停用時，服務會直接回傳而不進行任何更新。
- 手動插入帶有韌體升級套裝程式的 USB 隨身碟，並透過記錄檔檢查升級檔案是否
  正確。
- 在 qemu 上使用一些測試腳本模擬 `dev/sda1`，並啟動服務（例如：
  `systemcl start usb-code-update@sda1.service`）
- 驗證 ApplyTime 屬性值已設為 OnRest。
- 驗證 RequestedActivation 屬性值已設為 Active。
- 驗證韌體更新已成功。

[1]: https://github.com/openbmc/phosphor-bmc-code-mgmt
