# OpenBMC Flash 佈局與檔案系統文件

本檔專注於提供 code update 應用程式開箱即用所需的 flash 設定相關資訊，其中
包含 Linux 檔案系統的設定方式、檔案系統佈局、overlays、開機選項，以及
code update 如何更新 flash 模組並開機至新映像檔。關於 code update 介面的
詳細資訊，請參閱 [code-update.md](code-update.md)。

## 設計考量

### 開機載入與初始化

系統初始化與啟動（bootstrap）選用 [Das U-Boot][] 作為開機載入程式（bootloader）。

完成系統的基本初始化後，bootloader 可能會顯示提示字元，及/或啟動自動開機。
用於選取開機映像檔的指令及/或資料儲存在 bootloader 環境（environment）中。
bootloader 將壓縮的核心、initrd 映像檔與裝置樹（device tree）複製到記憶體，
然後將控制權轉移給核心。核心使用裝置樹中所傳遞的資訊（包含 flash 分區，
以及內嵌於樹中的核心命令列）來初始化核心本身與系統。

### 執行期管理

執行期管理選用 [systemd][] 系統與服務管理程式，原因是它具備設定、相依性
與觸發動作（triggered action）支援，以及健全的復原能力。

在開始執行之前，systemd 需要 root 檔案系統與所有執行檔都已掛載完畢。
/dev、/sys 與 /proc 的檔案系統可以在啟動 systemd 之前先行掛載。
請參考 [systemd File Hierarchy Requirements][]。

### Root 檔案系統

root 檔案系統的儲存選用唯讀磁區（read-only volume）。這樣做法讓檔案系統的
大部分內容，包含所有執行檔與靜態資料檔，都能儲存在唯讀的檔案系統映像檔中。
取代唯讀檔案系統映像檔可讓內容所使用的空間在建置時即得以確認，並可選用
不支援變更（mutations）的壓縮檔案系統。

我們已盡力遵循檔案系統階層標準 [FHS][]。特別是，僅在目前這次開機有效的
短暫資料（ephemeral data）儲存在 /run，而大部分應用程式資料則儲存在 /var
之下。仍有部分資訊儲存在系統設定資料目錄 /etc 中，主要為傳統上的設定，
例如網路位址、使用者識別與 ssh 主機金鑰。

為了節省 flash 空間，選用搭配 xz 壓縮的 squashfs 來儲存唯讀檔案系統內容。
這適用於附掛 flash 儲存空間有限的系統（請參考下方的 JFFS2 與 UBI 選項），
而非 eMMC。

為載入 root 檔案系統，initramfs 會尋找並掛載 squashfs 與可寫入檔案系統，
以 overlayfs 將兩者合併，對結果執行 chroot 後啟動 systemd。或者，也可以用
來尋找 BMC 作用中映像檔的資訊儲存在 U-Boot 環境中，並由一支 init 腳本
掛載映像檔後啟動 systemd。此選擇取決於平台實作，詳細內容請參考下方的
「支援的檔案系統選項」章節。

## 支援的檔案系統選項

OpenBMC 支援下列幾種檔案系統類型的 code update。每種檔案系統都會提供
額外資訊，例如檔案系統如何儲存在 flash 上、於系統開機/初始化期間檔案系統
如何實例化，以及 code update 如何處理該檔案系統。

### 可寫入檔案系統選項

#### MTD 分區上的 JFFS2

檔案系統的主要部分以唯讀 squashfs 的形式，使用區塊模擬驅動程式
（mtdblock）儲存在 MTD 分區中。第二個 MTD 分區則以 JFFS2 檔案系統掛載為
可讀寫。這個可讀寫檔案系統會掛載於整個檔案系統空間之上，使得所有檔案
與目錄都能被寫入。

這個檔案系統堆疊需要從 initramfs 執行掛載動作。initramfs 由一個以 busybox
為基礎的基礎系統，以及三支自訂腳本（init、shutdown 與 update）組成，
它們會依名稱尋找 MTD 分區。這些腳本由 [obmc-phosphor-initfs][] 安裝。

在 code update 模式下，initramfs 會將 squashfs 映像檔以及可讀寫檔案系統中
白名單內的檔案複製到 RAM，用以組裝出 root overlayfs 實例，使 flash 保持
空閒以便在執行期安裝映像檔時加以修改。正常關機（orderly shutdown）會將
剩餘的映像檔寫入同名（like-named）的原始 MTD 分區，並將白名單內的檔案
寫入可寫入的 overlay 檔案系統。或者，如果並未選取 code update 模式，
映像檔的更新就必須延後到正常關機期間分區被卸載時再進行。

這是 OpenBMC 的預設檔案系統。它用於多款以 Aspeed Technology 的 AST2400
與 AST2500 系統單晶片（system-on-chip）控制器為核心的 BMC 系統。這些 SoC
支援 1 與 2 GB 的 DDR RAM，而附掛的 flash 儲存空間通常只有數十 MB，
因此將檔案系統暫存（staging）到 RAM 並非問題。

#### MTD 分區上的 UBI

檔案系統的主要部分以唯讀 squashfs 的形式，使用 UBI 區塊模擬驅動程式
（ubiblock）儲存在靜態 UBI 磁區（static UBI volume）中。為儲存檔案的更新，
使用一個 UBIFS 磁區作為 /var，並以 overlayfs 掛載於 /etc 與 /home 目錄之上。
這些掛載由 [preinit-mounts][] 套件安裝的 `init` 腳本，在 `systemd` 啟動之前
執行。選用 UBI 可讓對可讀寫 overlay 的寫入分散到整個 UBI 區域，而不只是
侷限於可讀寫的 MTD 分區。

Das U-boot 的環境仍儲存在 flash 上的固定磁區（fixed sectors）中。Das U-boot
環境包含足夠的 MTD 分區定義，得以讀取同一個 flash 中某個 UBI 裝置內的
UBI 磁區。bootcmd 腳本從 FIT 映像檔載入核心，並將其透過 bootargs 傳入，
以尋找並掛載配對（paired）UBI 磁區中的 squashfs。

這個選項透過 `obmc-ubi-fs` 這個 OpenBMC distro feature 啟用。用於與 JFFS2
選項相同的 BMC 子系統，但目標是針對有足夠 flash 儲存空間、至少可容納
檔案系統兩份複本的設定。這可以透過雙重 flash 儲存來達成。有些控制器，
例如 AST2500 中的控制器，允許在開機失敗時從另一顆 flash 開機，而這個
UBI 選項支援此功能。為了這個支援，每顆 flash 上各儲存一份核心的複本，
並由 U-Boot 環境來選取要使用哪一份核心。

#### eMMC 上的 ext4

這是一項進行中的工作。請參考 [eMMC Design Document][]。

### 輔助檔案系統

/tmp、/run 及類似目錄使用 tmpfs，而 /dev、/proc 與 /sys 則由其一般性的
核心特殊檔案系統支援，如 FHS 所規範。

## 其他

另有針對 Raspberry Pi 與 x86 QEMU 機器的 Bitbake layer 設定，但它們主要
是供程式開發與探索之用。這些環境並不支援 code update。

[das u-boot]: https://www.denx.de/wiki/U-Boot
[systemd]:
  https://github.com/openbmc/docs/blob/master/architecture/openbmc-systemd.md
[systemd file hierarchy requirements]:
  https://www.freedesktop.org/wiki/Software/systemd/FileHierarchy/
[fhs]: https://refspecs.linuxfoundation.org/fhs.shtml
[obmc-phosphor-initfs]:
  https://github.com/openbmc/openbmc/blob/master/meta-phosphor/recipes-phosphor/initrdscripts/obmc-phosphor-initfs.bb
[preinit-mounts]:
  https://github.com/openbmc/openbmc/tree/master/meta-phosphor/recipes-phosphor/preinit-mounts
[emmc design document]:
  https://github.com/openbmc/docs/blob/master/architecture/code-update/emmc-storage-design.md
