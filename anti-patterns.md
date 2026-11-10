# OpenBMC 反模式

來自 [Wikipedia](https://en.wikipedia.org/wiki/Anti-pattern)：

「反模式是指針對反覆出現的問題所產生的常見回應，通常沒有成效，而且很可能造成嚴重的反效果。」

OpenBMC 的開發者並不能百分之百的時候都做對百分之百的決定。
這個事實，加上軟體開發經常是在複製貼上，使得同樣的錯誤一再發生。

本頁面希望能記錄 OpenBMC 中存在的若干反模式，以減輕程式碼審查者的工作。
如果發現某個反模式，不必一再重複同樣的說明，只需要提供本文件的連結即可。

<!-- begin copy/paste on next line -->

## 反模式模板 [一行說明]

### 識別方式

（1 段）描述如何辨識這個反模式。

### 說明

（1 段）描述這個反模式的負面影響。

### 背景

（1 段）描述這個反模式為何存在。如果你不知道，試著執行 git blame，
看看當初是誰寫的這段程式碼，並在郵件列表或 Discord 上詢問他們原本的意圖，
以便在此記錄下來（你也可能因此發現它並不像你想像的那麼算是一個反模式）。
如果你無法判斷這個反模式為何存在，請在此填入："Unknown"。

### 解決方式

（1 段）描述解決該反模式所要處理問題的首選做法，以及以所述方式解決它的正面影響。

<!-- end copy/paste on previous line -->

## 自訂的 ArgumentParser 物件

### 識別方式

ArgumentParser 物件通常存在於包裝取得選項的呼叫。它把解析過程抽象化，
並提供一個 `[]` 運算子來存取參數。

### 說明

撰寫自訂的 ArgumentParser 物件會在版本庫中產生幾乎重複的程式碼。
ArgumentParser 本身是相同的，不同的只是所提供的選項。撰寫自訂的參數解析器
等於是把 c++ 命令列參數解析這個輪子重新發明了。

### 背景

ArgumentParser 之所以存在，是因為它很早就已到位，之後被複製到每個新的版本庫，
作為處理參數解析的一個簡便方法。

### 解決方式

CLI11 函式庫是專為支援現代化參數解析而設計與實作的。它能處理 OpenBMC
_daemon_ 中常見的情境，有一些方便的內建驗證器，並允許對驗證進行容易的自訂。

## 在 bitbake metadata 的 RDEPENDS 中明確列出共用函式套件

### 識別方式

```bitbake
RDEPENDS_${PN} = "libsystemd"
```

### 說明

bitbake 預設會檢查建置出來的應用程式，自動加入執行期相依性，因而確保任何
函式套件的相依性都會被自動加入映像、SDK 等。不需要在 recipe 中明確列出它們。

相依性會隨著時間改變，明確列出很容易出錯——最終的結果就是不必要的共用函式套件
被安裝進映像之中。

請參考 [Yocto Project Mega-Manual][mega-manual-rdepends]，了解何時該使用
明確的執行期相依性。

[mega-manual-rdepends]:
  https://www.yoctoproject.org/docs/latest/mega-manual/mega-manual.html#var-RDEPENDS

### 背景

OpenBMC 最初的 bitbake metadata 作者並不知道 bitbake 會自動加入這些相依性。
因此最初的 bitbake metadata 明確列出了共用函式相依性，之後便被複製貼上。

### 解決方式

不要在 RDEPENDS 中列出共用函式套件。這樣可以排除因為 bitbake metadata 中
未維護的函式相依性清單，而安裝不必要的共用函式套件的可能性。

## 在 systemd service 檔案中使用 /usr/bin/env

### 識別方式

在 systemd unit 檔案中：

```systemd
[Service]

ExecStart=/usr/bin/env some-application
```

### 說明

在 OpenBMC 之外，大多數提供 systemd unit 檔案的應用程式並不會以這種方式啟動
應用程式。所以就算不為别的，這看起來也很奇怪，並且違反了
[最少意外原則](https://en.wikipedia.org/wiki/Principle_of_least_astonishment)。

### 背景

這個反模式之所以存在，是因為有一項需求要求在唯讀檔案系統上能夠對應用程式
進行即時補丁（live patching）。以這種方式啟動應用程式，是滿足即時補丁需求的
實作之一部分。例如：

```bash
/usr/bin/phosphor-hwmon
```

在唯讀檔案系統上會變成：

```bash
/usr/local/bin/phosphor-hwmon`
```

也就是放在一個可寫的 /usr/local 檔案系統上。

### 解決方式

/usr/bin/env 的方法只能啟用到對應用程式的即時補丁。現在已經出現了一種能够
對唯讀檔案系統中任何檔案進行即時補丁的方法。假設 BMC 上某處存在一個可寫的
檔案系統，類似下面的做法：

```bash
mkdir -p /var/persist/usr
mkdir -p /var/persist/work/usr
mount -t overlay -o lowerdir=/usr,upperdir=/var/persist/usr,workdir=/var/persist/work/usr overlay /usr
```

就能啟用即時系統補丁，而且對 systemd service 檔案中如何啟動應用程式沒有任何
額外要求。這是啟用到即時系統補丁的首選方法，因為它讓 OpenBMC 開發者能以與
大多數其他專案相同的方式撰寫 systemd service 檔案。

要移除這個反模式的現有案例，請從 systemd service 檔案中移除 /usr/bin/env，
並替換成要啟動應用程式的完整路徑。例如，假設某個應用程式位於 /usr/bin：

```bash
sed -i s,/usr/bin/env ,/usr/bin/, foo.service
```

## 將可執行檔錯誤地放置於 /sbin、/usr/sbin 或 /bin、/usr/bin

### 識別方式

安裝在 `/usr/sbin` 的 OpenBMC 可執行檔。bitbake metadata、makefile 或 configure
腳本中的 `$sbindir`。指向 `/bin` 或 `/usr/bin` 可執行檔的 systemd service 檔案。

### 說明

將 OpenBMC 應用程式安裝在不正確的位置，違反了
[最少意外原則](https://en.wikipedia.org/wiki/Principle_of_least_astonishment)，
更重要的是違反了
[FHS](https://en.wikipedia.org/wiki/Filesystem_Hierarchy_Standard)。

### 背景

通常有兩種類型的可執行檔：

1. 由 systemd service 檔案等啟動的長時間執行_daemon_，並且*不*打算讓使用者
   直接執行。
2. 打算讓使用者作為 CLI 使用的工具程式。

類型 1 的可執行檔不應該放在 `${PATH}` 中的任何位置，因為這對使用者來說容易
混淆且容易出錯，而應該放在 `/usr/libexec/<package>` 子目錄中。

類型 2 的可執行檔應該放在 `/usr/bin`，因為它們是打算讓使用者使用的，應該要
在 `${PATH}` 中（此外，在我們過渡到擁有非 root 存取權的過程中，`sbin` 並不適當）。

sbin 這個反模式之所以存在，是因為在 OpenBMC 專案歷史的早期，FHS 被誤解了，
從那之後便持續擴散。

根據 hier(7) man page：

```bash
/usr/bin This is the primary directory for executable programs.  Most programs
executed by normal users which are not needed for booting or for repairing the
system and which are not installed locally should be placed in this directory.

/usr/sbin This directory contains program binaries for system administration
which are not essential for the boot process, for mounting /usr, or for system
repair.

/usr/libexec Directory  contains  binaries for internal use only and they are
not meant to be executed directly by users shell or scripts.
```

`/usr/sbin` 的 FHS 說明中提到「系統管理」，但實際上約定俗成的解讀是其所管理的
系統指的是作業系統的安裝，而不是 OpenBMC 意義下的受管理系統。因此 OpenBMC
應用程式應該安裝在 `/usr/bin`。

在 Linux 中，將_daemon_移至 `libexec`，並從控制其啟動的 systemd service 檔案
的角度將其視為「內部使用」，正變得越來越常見。

### 解決方式

將 OpenBMC 應用程式視情況安裝在 `/usr/libexec` 或 `/usr/bin/`。

## 處理未預期的錯誤碼與例外

### 識別方式

這個反模式是指應用程式在遇到以錯誤碼和例外形式出現的未預期狀況後仍繼續處理，
而且沒有擷取解決問題所需的資料。

C++ 程式碼範例：

```cpp
using InternalFailure = sdbusplus::xyz::openbmc_project::Common::Error::InternalFailure;
try
{
  ... use d-Bus APIs...
}
catch (InternalFailure& e)
{
    phosphor::logging::commit<InternalFailure>();
}
```

### 說明

抑制未預期的錯誤可能導致應用程式出現不正確或不穩定的行為，進而影響它所提供的
服務以及整體系統。

寫入一筆日誌記錄而非產生核心傾印（core dump），可能無法提供足夠的資訊來除錯
一個真正未預期的問題，因此開發者沒有機會調查問題，系統的可靠度也不會隨時間改善。

### 背景

程式設計師希望在應用程式遇到可從中復原的狀況時能繼續處理。有時他們太用力了，
在不適合繼續的時候仍繼續下去。

程式設計師也希望記錄應用程式中發生的事情，所以他們寫入日誌記錄，在出問題時
提供除錯資料，通常是針對他們自己的用途。他們沒有考慮到日誌記錄會如何被 BMC
管理員或自動化服務工具所使用。

[Phosphor logging README][] 中的 `InternalFailure` 被過度使用了。

[phosphor logging readme]:
  https://github.com/openbmc/phosphor-logging/blob/master/README.md

### 解決方式

需要幾件事：

1. 檢查所有會給出回傳碼或 errno 值的地方。認真考慮預設的錯誤處理程式應該擲出
   例外，例如 `std::system_error`。
2. 要有充分的理由來處理特定的錯誤碼。請參考下方。
3. 要有充分的理由來處理特定的例外。讓其他例外向外傳播。
4. 記錄（以對其他服務的影響而言）當這個服務失敗、停止或重新啟動時會發生什麼事。
   用這些資訊來決定復原策略。

在錯誤處理程式中：

1. 考慮應該記錄哪些資料（如果有的話）。決定誰會使用這筆日誌記錄：BMC 開發者、
   管理員，還是分析工具。通常答案會是多於其中一種。

   以下的日誌記錄範例使用一個 IPMI 請求來開啟網路存取，但使用者輸入無效。
   - BMC 開發者：引用開發者會熟悉的內部應用程式、服務、pid 等。

     範例：
     `ipmid service successfully processed a network setting packet, however the user input of USB0 is not a valid network interface to configure.`

   - 管理員：引用 BMC 的對外介面，例如 REST API。他們能夠針對無效輸入或需要
     重新啟動 BMC 的回饋作出回應。

     範例：
     `The network interface of USB0 is not a valid option. Retry the IPMI command with a valid interface.`

   - 分析工具：考慮將日誌拆解並納入多個分析工具能够利用的屬性。例如，將日誌
     標記為 'Internal' 沒有幫助。然而，把它拆解成類似
     `[UserInput][IPMI][Network]` 的東西，就能一眼看出透過 IPMI 命令設定網路時
     所收到的輸入是無效的。建立給分析應用程式使用的日誌時，分類與系統影響是
     需要著重的關鍵要素。

     範例：
     `[UserInput][IPMI][Network][Config][Warning] Interface USB0 not valid.`

2. 判斷應用程式能否完全從該狀況中復原。如果不能，就不要繼續。讓系統自行決定
   是否寫入核心傾印或重新啟動服務。如果服務失敗時會造成嚴重影響，考慮使用更好的
   錯誤復原機制。

## 非標準的除錯應用程式選項與日誌記錄

### 識別方式

應用程式在啟動時使用非標準的方法來表示詳細日誌記錄，及／或不使用標準的
systemd-journald 除錯層級來記錄日誌。

### 說明

在除錯 OpenBMC 中橫跨多個應用程式的問題時，如果不同的應用程式有不同啟用除錯的
機制，要啟用適當的除錯會非常困難。例如，不同的 OpenBMC 應用程式會接受以下這些
作為啟用額外除錯的命令列參數：

- 0xff, --vv, -vv, -v, --verbose, `<and more>`

依同樣的道理，有些應用程式接著有它們自己內部寫入除錯資料的方法。它們使用
std::cout、printf、fprintf 等。這樣做在試圖比較不同應用程式的除錯資料時會造成
其他問題，因為這些應用程式的緩衝區可能在不同時間被清除（並被 journald 接手）。

### 背景

每個人都有自己除錯的方法。OpenBMC 內部對於該怎麼做並沒有真正的標準，所以每個
人都用他們自己熟悉的方式。

### 解決方式

如果一個 OpenBMC 應用程式要透過命令列支援增強的除錯，它就會支援標準的
"-v,--verbose" 選項。

一般來說，OpenBMC 開發者應該使用 "debug" journald 層級來記錄除錯訊息。這可以
全域啟用／停用，並適用於所有應用程式。如果開發者認為這在某些情況下會產生太多
除錯資料，他們可以把這些 journald 除錯呼叫包在 --verbose 命令列選項的保護之下。

## 代表 GPIO 的 DBus 介面

### 識別方式

想要暴露一個 DBus 介面來驅動 GPIO，例如：

- <https://lore.kernel.org/openbmc/YV21cD3HOOGi7K2f@heinlein/>
- <https://lore.kernel.org/openbmc/CAH2-KxBV9_0Dt79Quy0f4HkXXPdHfBw9FsG=4KwdWXBYNEA-ew@mail.gmail.com/>
- <https://lore.kernel.org/openbmc/YtPrcDzaxXiM6vYJ@heinlein.stwcx.org.github.beta.tailscale.net/>

### 說明

由 GPIO 選取的平台功能，若系統設計理念有所轉變，同樣也能用其他方式選取。因此
GPIO 是一項（硬體）實作細節。在 DBus 上暴露這個實作，會迫使平台的設計細節散布
到多個應用程式之中，這違反了軟體設計中
[低耦合][coupling]的原則，並影響我們對維護的信心。

[coupling]: https://en.wikipedia.org/wiki/Coupling_%28computer_programming%29

### 背景

GPIO 常用來選取那些難以通用化、因此也難以抽象化的功能。如果是這種情況，那麼
相關的功能最好被包裝成一個更普遍適用之行為的實作細節（例如主機開機程序）。

### 解決方式

考慮這個 GPIO 提供了什麼功能，並設計或利用一個現有的介面來暴露該行為。

## 非常長的 lambda 回呼函式

### 識別方式

類似以下的 C++ 程式碼：

```cpp
dbus::utility::getSubTree("/", interfaces,
                         [asyncResp](boost::system::error_code& ec,
                                     MapperGetSubTreeResult& res){
                            <too many lines of code>
                         })
```

### 說明

內嵌 lambda 雖然在某些情境下很有用，但難以閱讀，而且與 `clang-format` 這類工具的
格式化結果不一致，這在審查時，以及在套用可能有小衝突的補丁集時，都會造成嚴重
問題。此外，因為它們是在函式作用域中宣告的，所以難以進行單元測試，而且由於它們
沒有名稱，所產生的日誌訊息也很難閱讀。它們還難以除錯，因為沒有任何符號名稱可以
用來掛上斷點或追蹤點。

### 背景

在非同步 IO 的情境中，lambda 是實作回呼函式的一種自然方法。此外，Boost 和
sdbusplus 的 ASIO API 傾向於鼓勵這種做法。要採用 lambda 以外的方式需要更多努力，
因此若沒有壓力，這些選項往往不會被選擇。

### 解決方式

優先使用 `std::bind_front` 搭配一個方法或靜態函式來處理回傳值，或者使用一個少於
10 行程式碼的 lambda 來內嵌處理錯誤。在無法使用 `std::bind_front` 的情況下，例如
在 `sdbusplus::asio::connection::async_method_call` 中，請讓 lambda 長度少於 10 行，
並呼叫適當的函式來處理非平凡的轉換。

```cpp
void afterGetSubTree(std::shared_ptr<bmcweb::AsyncResp>& asyncResp,
                     boost::system::error_code& ec,
                     MapperGetSubTreeResult& res){
   <many lines of code>
}
dbus::utility::getSubTree("/xyz/openbmc_project/inventory", interfaces,
                          std::bind_front(afterGetSubTree, asyncResp));
```

另請參考 [Cpp Core Guidelines][]，其中關於 lambda 何時適用有通则性的指引。上述建議
與 Cpp Core Guidelines 一致。

[Cpp Core Guidelines]:
  https://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines#f11-use-an-unnamed-lambda-if-you-need-a-simple-function-object-in-one-place-only

## 將內部標頭檔放在平行的子樹中

### 識別方式

宣告那些不参與函式庫公開 API 的型別與函式，卻把它們放在來源樹中沒有與其實作
並存（adjacent）的標頭檔裡。

### 說明

沒有理由把內部標頭檔放在平行的子樹中（例如 `include/`）。這樣整理起來更費力，
讓宣告與定義之間產生了不必要的距離，並增加了建置系統設定所需的心力。

### 背景

在 C 與 C++ 中，標頭檔把函式庫的公開 API 暴露給它的使用者，需要被安裝到開發者
系統中一個編譯器已知的位置。為了區分哪些標頭檔要安裝、哪些不要，公開的標頭檔
常被放進來源樹的一個 `include/` 子目錄中，以標明它們的重要性。

任何屬於函式庫實作細節的函式或結構，都不應該出現在它已安裝的標頭檔中。忽略這個
理念會過度暴露函式庫的設計，並可能在未來導致本來不必要的 API 或 ABI 破壞。

此外，其產出物只有應用程式二進位檔的專案，在函式庫的意義上並沒有公開的 API 或
ABI。這類專案的來源樹中任何標頭檔都不需要安裝到開發者的系統上，把它們在路徑上
與實作隔離並沒有任何用處。

### 解決方式

把內部標頭檔直接放在包含對應實作的來源檔旁。

## `lg2` 訊息字串中定義不良的資料結構化

### 識別方式

試圖把資訊編碼進日誌的 MESSAGE 字串中，而這些資訊最多只能用 regex 勉強解析，
同時又降低了人類可讀性。例如：

```cpp
error(
    "Error getting time, PATH={BMC_TIME_PATH} TIME INTERFACE={TIME_INTF}  ERROR={ERR_EXCEP}",
    "BMC_TIME_PATH", bmcTimePath, "TIME_INTF", timeInterface,
    "ERR_EXCEP", e);
```

### 說明

[`lg2` 是 OpenBMC 首選的 C++ 日誌記錄介面][phosphor-logging-lg2]，它實作在 systemd
日誌及其函式庫 API 之上。[systemd-journald 提供結構化日誌記錄]
[systemd-structured-logging]，讓我們能在所提供的訊息旁擷取額外中繼資料。

[phosphor-logging-lg2]:
  https://github.com/openbmc/phosphor-logging/blob/master/docs/structured-logging.md
[systemd-structured-logging]:
  https://0pointer.de/blog/projects/journal-submit.html

結構化日誌記錄的概念，讓我們能以方便的方式以程式存取與日誌事件相關的中繼資料。
日誌會為每筆記錄的訊息擷取一組預設的中繼資料。然而，這些記錄項暴露給使用者的
主要方式，是 `journalctl` 預設的行為——只印出每筆日誌記錄的 `MESSAGE` 值。這可能
讓人誤以為印出的訊息是暴露相關中繼資料以調查瑕疵的唯一方式。

為了人因上的考量，`lg2` 提供了一種能力，可以把結構化資料內插（interpolate）進
日誌的 `MESSAGE` 字串。這有助於人工檢視日誌，因為可以突顯某個日誌事件的重要
中繼資料。然而，這並不打算被用於臨時的、定義不良地試圖暴露中繼資料給自動化分析。

提供給 `lg2` 日誌記錄 API 的所有鍵值對都會被擷取進結構化的日誌事件中，不論某個
特定的鍵是否有被內插進 `MESSAGE` 字串。即使資訊沒有被擷取進 `MESSAGE` 字串，也總是
能够還原與該日誌事件相關的資訊。

`phosphor-time-manager` 示範了一個對 `lg2` API 的合理用法。程式碼中的一個日誌記錄
範例[如下][phosphor-time-manager-lg2]：

[phosphor-time-manager-lg2]:
  https://github.com/openbmc/phosphor-time-manager/blob/5ce9ac0e56440312997b25771507585905e8b360/manager.cpp#L98

```cpp
info("Time mode has been changed to {MODE}", "MODE", newMode);
```

預設情況下，這在 `journalctl` 的輸出中會呈現為：

```text
Sep 23 06:09:57 bmc phosphor-time-manager[373]: Time mode has been changed to xyz.openbmc_project.Time.Synchronization.Method.NTP
```

然而，我們可以使用一些 journalctl 命令列選項來檢視與該日誌記錄項相關的結構化資料：

```bash
# journalctl --identifier=phosphor-time-manager --boot --output=verbose | grep -v '^    _' | head -n 9
Sat 2023-09-23 06:09:57.645208 UTC [s=85c1cb5f8e02445aa110a5164c9c07f6;i=244;b=ffd111d3cdca41c8893bb728a1c6cb20;m=133a5a0;t=606009314d0d9;x=9a54e8714754a6cb]
    PRIORITY=6
    MESSAGE=Time mode has been changed to xyz.openbmc_project.Time.Synchronization.Method.NTP
    LOG2_FMTMSG=Time mode has been changed to {MODE}
    CODE_FILE=/usr/src/debug/phosphor-time-manager/1.0+git/manager.cpp
    CODE_LINE=98
    CODE_FUNC=bool phosphor::time::Manager::setCurrentTimeMode(const std::string&)
    MODE=xyz.openbmc_project.Time.Synchronization.Method.NTP
    SYSLOG_IDENTIFIER=phosphor-time-manager
```

在這裡我們發現 `MODE` 及其值被擷取為結構化資料中一個獨立的中繼資料記錄項，同時也
如要求般被內插進 `MESSAGE`。此外，從日誌記錄項中我們可以用記錄在 `LOG2_FMTMSG`
中繼資料記錄項裡的格式字串，找出 `MODE` 是*如何*被內插進 `MESSAGE` 的。

`LOG2_FMTMSG` 也提供了一個穩定的把手，用以識別日誌中某個特定類別日誌事件的存在，
進一步協助自動化分析。

### 背景

各種補丁，例如 [PLDM:Catching exception precisely and printing
it][openbmc-gerrit-67994]，加入了一些臨時的、定義不良、試圖透過 `MESSAGE` 記錄項
提供所有中繼資料的做法。

[openbmc-gerrit-67994]: https://gerrit.openbmc.org/c/openbmc/pldm/+/67994

### 解決方式

`lg2` 訊息的格式應該是要讓人類來使用的。它們不應該包含臨時的、定義不良、試圖
整合中繼資料給自動化分析的做法。
