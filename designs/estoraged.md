# eStoraged 設計文件 - 加密輔助儲存裝置管理 daemon

作者：John Wedig <johnwedig@google.com>

其他貢獻者：

- John Broadbent <jebr@google.com>
- Benjamin Fair <benjaminfair@google.com>
- Nancy Yuenn <yuenn@google.com>

建立日期：2021 年 9 月 2 日

## 問題描述

這個 daemon 將作為加密 storage device 的抽象層，將安全性功能封裝起來，並提供
一個 D-Bus 介面來管理裝置上的加密檔案系統。其他軟體元件可以透過這個 D-Bus
介面與 eStoraged 互動，執行例如建立新的加密檔案系統、清除其內容、鎖定／解鎖
裝置，或變更密碼等操作。

## 背景與參考資料

本設計旨在管理輔助儲存裝置（secondary storage device），無法用於 root 檔案系統，也就是說，BMC
必須能在裝置仍處於鎖定狀態時完成開機。

本設計使用
[cryptsetup](https://gitlab.com/cryptsetup/cryptsetup) 工具，該工具反過來使用
[dm-crypt](https://en.wikipedia.org/wiki/Dm-crypt) kernel 子系統。Dm-crypt
提供加密與裝置對應（device mapping）能力，而 Cryptsetup 提供
[LUKS](https://en.wikipedia.org/wiki/Linux_Unified_Key_Setup) 能力，有助於密碼
管理，讓我們能在不必重新加密整個裝置的情況下變更密碼等操作。

本設計特別針對 eMMC device 使用，我們計畫在 eMMC 硬體層級使用鎖定／解鎖功能
（CMD42）作為額外安全措施。此功能會在鎖定狀態下禁止對裝置的所有讀取或寫入
存取。關於此功能的一些文件可參考
[JEDEC 標準 JESD84-B51A](https://www.jedec.org/document_search?search_api_views_fulltext=jesd84-b51)，
或這份文件：
[Enabling SD/uSD Card Lock/Unlock Feature in Linux](https://media-www.micron.com/-/media/client/global/documents/products/technical-note/sd-cards/tnsd01_enable_sd_lock_unlock_in_linux.pdf?rev=03f03a6bc0f8435fafa93a8fc8e88988)。

本文件中參考到幾種不同類型的金鑰：

- Volume key（磁碟區金鑰）：用於加密 block device 上資料的主要加密金鑰。
- Encryption Password（加密密碼）：將 volume key 載入 RAM 並解密檔案系統所需的
  密碼。
- Device Password（裝置密碼）：鎖定或解鎖裝置硬體的密碼。

## 需求

本設計應針對下列功能提供介面：

- 在裝置上建立新的 LUKS 加密檔案系統
- 安全地清除裝置並驗證資料確已清除
- 鎖定裝置
- 解鎖裝置
- 變更密碼

此外，eStoraged 應該：

- 使用具備足夠熵（entropy）的亂數產生器來產生 volume key，使其足夠隨機且安全。
- 善用硬體所提供的任何安全功能（作為深度防禦（defense-in-depth）措施）。
- 使用足夠通用的介面，以便在需要時可延伸支援其他類型的 storage device。例如
  不同裝置在裝置鎖定上會有不同的指令集，例如 MMC、SATA、NVMe。初期我們計畫
  僅將 eStoraged 用於 eMMC device，但未來我們可能希望將其應用於其他類型的
  storage device。

本設計的使用者可以是 BMC 中的任何其他軟體元件。BMC 上的某個 client daemon
會與 eStoraged 互動，在 eMMC 上設定新的加密檔案系統。此外，該 client daemon
可以設定為在 BMC 開機時解鎖 eMMC device。提供密碼是 client daemon 的責任。
舉例來說，這個密碼可以來自使用者輸入、從安全位置擷取，或由 client daemon
自行產生密碼。

## 建議設計

eStoraged 會將每個 eMMC device 表示為 D-Bus 上的一個物件，該物件實作一個介面，
提供下列方法與屬性：

- (method) Format
- (method) Erase
- (method) Lock
- (method) Unlock
- (method) Change Password
- (property) Locked
- (property) Status

啟動時，eStoraged 會為系統中的每個 eMMC device 建立一個 D-Bus 物件。具體來說，
我們會使用 udev 為每個 eMMC 啟動一個 eStoraged 實例。bus name 與 object name
將如下：

Bus Name: xyz.openbmc_project.eStorage.\<device name\> Object Path:
/xyz/openbmc_project/storage/\<device name\>

object path 的設計意在足夠通用，使我們最終可以讓多個 daemon 管理同一個
storage device，同時使用相同的 object path。例如，本 daemon 負責處理加密，而
另一個 daemon 可以為同一個裝置提供統計資訊。

為了管理加密檔案系統，我們會使用
[cryptsetup API](https://mbroz.fedorapeople.org/libcryptsetup_API/)。這個函式庫
提供建立新 LUKS header、設定密碼等功能。

對於 eMMC device，我們計畫啟用密碼鎖定功能（CMD42），以在鎖定狀態下禁止對裝置
的所有讀取或寫入存取。因此，"Locked" 屬性將同時代表硬體層級的鎖定與加密層級
的鎖定。我們可能會使用 ioctl 介面將相關指令傳送到 eMMC，類似
[mmc utils](https://git.kernel.org/pub/scm/utils/mmc/mmc-utils.git/) 的做法。

其他類型裝置的硬體鎖定支援可視需要加入，但至少，即使某個特定裝置不支援硬體
鎖定，仍可提供僅加密的鎖定方式。

如前所述，client 會提供一個密碼。eStoraged 會使用這個密碼產生兩個不同的密碼：
加密密碼與裝置密碼（若有硬體鎖定可用的話）。這兩個密碼會不同，這樣一來，一旦
某個密碼外洩，我們仍能依靠另一個密碼獲得某種程度的保護。

Erase 方法應提供一種方式來指定清除的類型，例如全部寫入零，或做其他處理。不同
組織對於何謂安全清除可能會有不同的看法。

由於部分 D-Bus 方法可能花費較長時間（例如安裝新的加密檔案系統），D-Bus 介面
會是非同步的，並帶有 "Status" 屬性可供查詢，用以表示下列狀態之一：成功、錯誤
或進行中。

## 考慮過的替代方案

一種替代方案是使用 systemd targets 來管理 eMMC。例如，
[systemd-cryptsetup@.service](https://www.freedesktop.org/software/systemd/man/systemd-cryptsetup@.service.html)
常用於解鎖加密的 block device，它會從 key file 或使用者輸入取得密碼。然而，
OpenBMC 架構要求使用 D-Bus 作為主要的程序間溝通（inter-process communication）
形式。此外，使用具備明確定義 D-Bus 介面的 daemon，可以讓安全性功能更隔離、
更易於維護，也更容易測試。

另一個相關的軟體是 UDisks2，它也會為系統中每個 storage device 匯出一個 D-Bus
物件。它能夠透過 Format 方法設定加密的 block device：
[org.freedesktop.UDisks2.Format](http://storaged.org/doc/udisks2-api/latest/gdbus-org.freedesktop.UDisks2.Block.html#gdbus-method-org-freedesktop-UDisks2-Block.Format)。
並提供數個與加密相關的額外方法：Lock、Unlock 與 ChangePassphrase。請參考
D-Bus 介面
[org.freedesktop.UDisks2.Encrypted](http://storaged.org/doc/udisks2-api/2.7.5/gdbus-org.freedesktop.UDisks2.Encrypted.html)。
妨礙我們採用此工具的主要問題是它會讓 image 大小增加過多。我們發現由於會拉入
transitive dependencies，例如 mozjs 與 python，壓縮後的 image 大小增加了
22 MB。

## 影響

若要使用 eStoraged，可能需要提供另一個 client daemon 來管理密碼並呼叫 eStoraged
的 D-Bus API。由於密碼管理機制對於不同平台與組織可能各不相同，這超出了本設計的
範圍。

## 測試

- 單元測試（Unit tests）以驗證 eStoraged 中的各項程式路徑。
- 迴歸測試（Regression tests）會操作各項 D-Bus 方法：加密、清除、鎖定、解鎖，
  以及變更密碼。
