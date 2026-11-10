# eMMC 儲存設計

作者：Adriana Kobylak < anoo! >

其他貢獻者：Joel Stanley < shenki! >、Milton Miller

建立日期：2019-06-20

## 問題描述

提出一份用於定義 eMMC 裝置初始儲存設計的方案。內容包含檔案系統類型、分割、
磁碟區管理、開機選項與初始化等。

## 背景與參考

OpenBMC 目前支援原始 flash，例如 AST2400 與 AST2500 系統中所使用的 SPI NOR，
但針對受管理的 NAND（managed NAND）尚無設計。

## 需求

- 安全性：能夠強制唯讀，並對正式版（production）所用的官方／簽章映像進行驗證。

- 可更新：確保檔案系統設計能夠實作有效且簡潔的更新機制。

- 簡潔性：讓系統易於理解，進而便於開發、測試、使用與復原。

- 程式碼重複使用：盡量使用既有的方案，不要重複發明輪子。

## 提議設計

- eMMC 映像的佈局與特性在 meta layer 中指定。這讓 OpenBMC 得以支援不同的
  佈局與設定。用於執行程式碼更新的 tarball 仍由 image_types_phosphor 建置，
  因此若要支援不同的檔案系統類型，就需要另外建立一個 IMAGE_TYPES。

- 程式碼更新：在 flash 上支援兩個版本。這麼做可以保留一個已知良好的映像，
  並驗證新的映像。

- 為 eMMC User Data Area 使用 GPT 分割：選擇它而非動態分割，是因為缺少可用於
  建置 LVM 映像的離線工具（請參閱下方「替代方案」章節中的 Logical Volumes）。

- Initramfs：需要一個 initramfs，在首次開機時執行 sgdisk，把次要 GPT 移到裝置
  末端它應在的位置，因為 yocto 的 wic 工具目前不支援建置指定大小的映像，所以
  產生的映像大小可能不完全等於要被燒錄的裝置大小。

- 唯讀與可讀寫檔案系統：ext4。這是穩定且廣泛用於 eMMC 的檔案系統。

- 檔案系統佈局：根檔案系統託管在唯讀磁碟區中。/var 目錄掛載於一個可讀寫、且在
  程式碼更新後仍持續保留的磁碟區。/home 目錄需要可寫，以存放諸如 ssh 金鑰的使用者
  資料，因此它是掛往可讀寫磁碟區中某個目錄的 bind mount。bind mount 比 overlay
  更可靠，且存在時間更久。由於 /home 目錄沒有由映像提供的內容，因此可以使用
  bind mount。另一方面，/etc 目錄有由映像提供的內容，所以它是 overlayfs，以便在
  出廠重設（factory reset）時能回復其設定內容。

        +------------------+ +-----------------------------+
        | Read-only volume | | Read-write volume           |
        |------------------| |-----------------------------|
        |                  | |                             |
        | / (rootfs)       | | /var                        |
        |                  | |                             |
        | /etc  +------------->/var/etc-work/  (overlayfs) |
        |                  | |                             |
        | /home +------------->/var/home-work/ (bind mount)|
        |                  | |                             |
        |                  | |                             |
        +------------------+ +-----------------------------+

- 佈署（Provisioning）：OpenBMC 會像目前對 NOR 晶片所做的那樣，產出一個可燒錄的
  eMMC 映像作為建置產物。

## 已考量的替代方案

- 將 U-Boot 與 Linux 核心存放在獨立的 SPI NOR flash 裝置中，因為諸如 AST2500 這類
  SOC 不支援從 eMMC 執行 U-Boot。此外，把 Linux 核心放在 NOR 上可免去要求 U-Boot
  支援 eMMC。U-Boot 與核心的大小不到 10MB，因此一顆相當小的晶片，例如 32MB 的，
  就足夠使用。因此，為了支援兩個韌體版本，每個版本的核心都需要存放在 NOR 中。
  可以再加一顆 NOR 裝置作為冗餘，以防 U-Boot 或核心無法執行。

  按照目前支援 UBI 的系統做法來格式化 NOR：一個給 U-Boot 的固定 MTD 分割、一個給
  其環境（environment）的分割，以及一個跨越剩餘 flash 的 UBI 磁碟區。將雙核心磁碟區
  存放在 UBI 分割中。這種做法可以重複使用既有的程式碼更新介面，因為靜態方式目前不
  支援存放兩個核心映像。選取所需的核心映像會沿用既有的 U-Boot 環境方式來完成。

  可以建立靜態 MTD 分割來存放核心映像，但要引入一種選取所需核心映像的新方法就需要
  額外的工作，因為靜態佈局目前沒有雙映像支援。

  AST2600 支援從 eMMC 執行 U-Boot，這帶來了彈性：系統上可以只放 eMMC 晶片，或者
  仍然把 U-Boot 放在獨立晶片中，以便在 eMMC 損壞時進行復原。

- 檔案系統：f2fs（Flash-Friendly File System）。f2fs 是一個正在興起的檔案系統，
  因此可能被視為不如 ext4 檔案系統成熟與穩定，儘管兩者在 OpenBMC 環境中的
  效能表現尚未知。

  一個合適的替代方案是 btrfs，它對檔案系統的中繼資料與資料都有checksum（校驗和），
  因此在資料完整性上提供更強的保證。

- 把所有程式碼更新的產物合併成單一映像。

  這帶來簡潔的程式碼維護：映像要嘛完整、要嘛不完整，要嘛可用、要嘛不可用，沒有額外的
  片段四處散落。U-Boot 只需做一個選擇——要載入哪個映像——並只需向核心傳遞一項資訊。

  為了透過限制將不必要磁區讀入記憶體的 IO 來縮短開機時間，在分割的開頭放置一個小的
  FS，用於容納 U-Boot 必須存取的產物。

  這個檔案系統會從 ext2、FAT12 與 cramfs 中選取，因為這三者都同時獲得 Linux 核心與
  U-Boot 支援。（若我們希望 U-Boot 環境能各側（per-side）獨立，則從 ext2 或 FAT12
  之中選一個（squashfs 的支援尚未合併，最後一次更新是在 2018 年——兩年前）。

- 不用 initramfs：或許可以透過把邏輯磁碟區的 UUID 傳遞給核心來開機 rootfs，儘管
  很可能仍需要一個 [pre-init script][]。因此，擁有一個 initramfs 能為初始化提供更
  標準的實作方式。

- FAT MBR 分割：FAT 是一個簡單且廣為人知的分割表格式。它有 4 個獨立分割的空間。
  或者，其中一個槽（slot）可以鏈入擴充分割，但鏈中每個分割都依賴於前一個分割。
  四個分割可能足以滿足共享（單一）開機檔案系統設計的初始需求（boot、rofs-a、
  rofs-b 與 read-write）。若是雙開機磁碟區設計則需要額外的分割。

  若 U-Boot 環境需要共用空間，它以檔案形式冗餘地存放在分割 1。U-Boot SPL 會放在這裡。
  若不需要，分割 1 可以保持未配置。

  兩個程式碼側（code side）建立於槽 2 與槽 3。

  可讀寫檔案系統佔用分割 4。

  若未來有額外分割的需求，可以在未來的程式碼更新中把分割移入擴充分割。

- Device Mapper：使用 device-mapper 的 linear target 來劃分 eMMC，這麼做可以在需要時
  擴充裝置而不必實體重新分割，因為 device-mapper 裝置會暴露邏輯磁區。這是透過修改
  提供給核心的 device-mapper 設定表項目，以追加未使用的實體磁區來達成。

- 邏輯磁碟區（Logical Volumes）：
  - 磁碟區管理：LVM。這允許動態分割／移除，與目前的 UBI 實作類似。LVM 支援會讓核心
    增大约 100kB，但為了在需要時能調整分割大小，這個空間增加是值得的。此外，UBI 磁碟區
    管理以類似的方式運作，因此在程式碼更新應用程式中實作 LVM 管理並不會太複雜。

  - 分割：若 eMMC 用於儲存開機載入程式，則一個 ext4（或 vfat）分割會容納包含核心、
    initrd 與裝置樹的 FIT 映像。這個磁碟區會掛載為 /boot。這讓 U-Boot 能夠載入核心，
    因為它不支援 LVM。在開機分割之後，將剩餘的 eMMC flash 指定為單一實體磁碟區（physical
    volume）以容納邏輯磁碟區，而非固定大小的分割。這對於某個分割的內容成長超過固定大小
    的情況提供了彈性。這也代表其他韌體映像，例如 BIOS 與 PSU，可以存放在單一 eMMC 裝置的
    磁碟區中。

  - Initramfs：使用 initramfs（這是 OpenBMC 的預設）從邏輯磁碟區開機 rootfs。initramfs
    在需要額外開機動作時（例如掛載 overlay）提供了彈性。它還提供了一個起點（環境）來佈署
    並格式化 eMMC 磁碟區。為了開機 rootfs，initramfs 會搜尋要掛載的目標 rootfs 磁碟區，
    而不使用 U-Boot 環境。

  - 掛載點：對於目前存放在獨立 SPI NOR 模組中的 BIOS 等韌體映像，eMMC 中的邏輯磁碟區會
    掛載在相同的路徑上，以避免改動那些依賴該資料位置的應用程式。

  - 佈署（Provisioning）：由於 LVM 使用者空間工具不提供離線模式，從 bitbake task 組裝
    一個 LVM 磁碟映像並不直覺。因此，讓 initramfs 建立 LVM 磁碟區，並從外部來源取得
    rootfs 檔案到 tmpfs 中以燒錄該磁碟區。rootfs 檔案可以用 DHCP、UART、USB 磁碟等方式
    取得。另一個替代選項包括從 QEMU 建置映像，這需要在建置程序的一部分中開機 QEMU，以
    設定 LVM 磁碟區並建立映像檔案。

## 影響

這個設計會影響 OpenBMC 的建置程序與程式碼更新的內部實作，但不應影響外部介面。

- openbmc/linux：為支援 eMMC 晶片及其檔案系統所需的核心改動。
- openbmc/openbmc：為建立 eMMC 映像所需的改動。
- openbmc/openpower-pnor-code-mgmt：為支援更新新檔案系統所需的改動。
- openbmc/phosphor-bmc-code-mgmt：為支援更新新檔案系統所需的改動。

## 測試

在含有 eMMC 的系統上驗證 OpenBMC 功能。這個系統可以加入 CI 池。

[pre-init script]:
  https://github.com/openbmc/openbmc/blob/master/meta-phosphor/recipes-phosphor/preinit-mounts/preinit-mounts/init
