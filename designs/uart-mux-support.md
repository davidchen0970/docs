# uart-mux-support 設計文件

作者：Alexander Hansen <alexander.hansen@9elements.com>

其他貢獻者：Andrew Jeffery <andrew@codeconstruct.com.au> @arj、Jeremy
Kerr <jk@ozlabs.org>、Patrick Williams <patrick@stwcx.xyz>

建立日期：2024 年 6 月 17 日

## 問題描述

有些硬體配置配備了一個 UART mux，可以透過 GPIO 來切換。
為了支援這種配置，obmc-console 需要提供一種主控台（console）選擇
機制，以避免手動設定 GPIO。

## 背景與參考

目前已經有[針對 obmc-console 的開放變更][obmc-console-uart-mux-series]，
但大家認定這個功能需要一份設計文件。

[obmc-console-uart-mux-series]:
  https://gerrit.openbmc.org/c/openbmc/obmc-console/+/71864

這裡的背景是，有一些設計選擇可能會影響其他子專案——不是以造成
功能倒退的方式，而是在日後這些專案需要支援前述硬體配置的時候。

## 需求

- 使用者可以選擇要進入 mux 的主控台

- 平台政策（不論由哪個服務實作）可以根據主機狀態及其他資訊
  來選擇合適的主控台。

- 任何讀取該主控台記錄檔的人，都能清楚知道主控台何時透過 mux 控制
  進行連線或中斷連線。記錄檔中不應出現無法解釋的缺口。

- mux 配置可以在單一檔案中指定

- 主控台選擇（意味著 mux 控制）必須可以由外部應用程式
  執行。

此變更的範圍涵蓋 obmc-console 以及依賴其所暴露 API 的其他專案。

此變更不會影響沒有這種硬體配置的使用者。

## 設計考量

要在 obmc-console 中加入 mux 支援，有多種選擇可用：

1. 「連線端點」（Unix domain socket、D-Bus 物件）代表什麼。
   可以是以下其中之一：
   1. Linux 所暴露的 TTY 裝置
   2. 預期的下游 mux 埠

2. mux 狀態如何控制。我們可以透過以下任何方式控制：
   1. 一個頻外（out-of-band）命令（例如透過某個以某種方式與連線端點
      關聯的 D-Bus 方法）
   2. 一個頻內（in-band）命令（例如引入類似 SSH 的轉義序列）
   3. 根據使用者所連線到的端點來選擇 mux 埠

3. 我們允許 mux 狀態在什麼情況下被變更
   1. 活躍的連線會阻止 mux 狀態被變更
   2. mux 狀態隨時可以變更，但會終止任何現存的衝突連線
   3. mux 狀態隨時可以變更，且對現存的衝突連線沒有影響

4. 我們是否希望特定連線上的資料串流代表：
   1. 不論 mux 狀態如何的主控台 IO
   2. 隔離到特定 mux 埠的主控台 IO

這些選項的某些組合存在限制。舉例來說：

- 如果連線端點代表 Linux 所暴露的 TTY 裝置（1.1），
  那麼我們就無法根據使用者所連線到的端點來選擇 mux 埠（2.3），
  因為我們根本沒有所需的資訊

- 如果連線端點代表預期的下游 mux 埠（1.2），
  那麼實作一個用於變更 mux 狀態的頻內命令（2.2）就沒有意義，
  因為這違反了抽象化

- 如果連線端點代表預期的下游 mux 埠（1.2），
  那麼它就無法提供另一個 mux 埠的主控台 IO（4.1），
  因為這與其定義相違背。

考量這些因素後，我們得到以下的設計選項表：

| ID  | 連線端點 (1) | mux 控制之定義方式 (2) | mux 控制政策 (3) | 串流資料 (4) |
| --- | ----------------------- | -------------------------- | ------------------------------------------------ | ----------------- |
| A   | TTY (1.1)               | 頻外命令 (2.1)  | 活躍連線阻止 mux 變更 (3.1)      | 隔離 (4.2)    |
| B   | TTY                     | 頻外命令        | 連帶中斷連線的 mux 變更 (3.2)             | 隔離          |
| C   | TTY                     | 頻外命令        | 不中斷連線的 mux 變更 (3.3)          | 多工 (4.1) |
| D   | TTY                     | 頻內命令 (2.2)      | 不中斷連線的 mux 變更                | 多工       |
| E   | mux 埠 (1.2)          | 以連線為基礎 (2.3)     | 衝突連線阻止 mux 變更 (3.1) | 隔離          |
| F   | mux 埠                | 以連線為基礎           | 連帶中斷連線的 mux 變更                   | 隔離          |
| G   | mux 埠                | 以連線為基礎           | 不中斷連線的 mux 變更                | 隔離          |
| H   | mux 埠                | 頻外命令        | 衝突連線阻止 mux 變更       | 隔離          |
| I   | mux 埠                | 頻外命令        | 連帶中斷連線的 mux 變更                   | 隔離          |
| J   | mux 埠                | 頻外命令        | 不中斷連線的 mux 變更                | 隔離          |

### 情境與使用案例

1. 一個 UART mux 在blade 上的衛星 BMC 與 blade 主機之間進行選擇

   衛星 BMC 上正在進行軟體更新，而 mux 已被切換以擷取衛星正在輸出的
   任何內容。記錄更新過程的輸出對於了解可能發生的任何失敗至關重要。

   在衛星 BMC 更新進行期間，使用者選擇連線到主機主控台。

2. 一個 blade 的衛星 BMC、CPLD 與主機都位於某個 UART
   mux 的個別埠上，且必須擷取 blade 開機過程中的相關輸出

   blade 的開機過程需要跨其衛星 BMC、CPLD 與主機的一連串動作。
   每個元件都提供關於開機過程的關鍵資訊，這些資訊在各種時間點
   輸出到各自的主控台上。

   為了便於關聯，它們的輸出應該一起被記錄下來。

### 討論

情境 1 是有問題的。它突顯了 mux 狀態擁有權這個根本疑慮。在該情境中，
系統處於一個敏感狀態，需要特定的 mux 配置（以從衛星 BMC 輸出更新
進度），但使用者已經表現出要選擇另一個的意圖（以與主機主控台互動）。

應該發生什麼事？而這個選擇是否會影響我們控制 mux 的方式？

採用以連線為基礎的方式來設定 mux 狀態（2.3），會導致連線到主機
主控台端點的使用者立即打斷來自衛星 BMC 的更新進度輸出。

相對地，透過頻外命令（2.1）而非在建立連線時（2.3）來設定 mux 狀態，
連線到主機主控台的使用者就不會立即打斷來自衛星 BMC 的更新進度輸出。

然而，我們可以推定使用者連線到主機主控台端點是有其原因的。透過
額外動作，使用頻外命令介面，他們同樣可能選擇切換 mux 而不顧慮
系統狀態，打斷來自衛星 BMC 的更新進度輸出。

這突顯出根本問題在於多個使用者存取系統，而他們彼此之間、以及
與系統狀態之間都不進行協調。隨之而來的問題是：

協調本來不協調的使用者，是否應該是 obmc-console 的責任？

這是一個政策問題：這些使用者應該如何被協調，很可能會因以下疑慮
而有非常不同的面貌——平台在較大系統中的角色、與之互動的使用者的
角色與需求，以及平台本身的具體設計。

obmc-console 應該實作一個控制 mux 狀態的機制，但很可能不應該套用
任何管理對被 mux 主控台存取權限的政策。

對於頻外命令方式，另一個疑慮是它與其他暴露主控台的元件之間的互動：

1. 透過 SSH 暴露主控台的 dropbear/obmc-console-client 整合
2. [bmcweb](https://github.com/openbmc/bmcweb/blob/master/include/obmc_console.hpp)
3. [phosphor-net-ipmid](https://github.com/openbmc/phosphor-net-ipmid/blob/master/sol/sol_manager.hpp)

採用頻外命令方式時，這些元件必須在以下之間做出選擇：

- 不提供任何變更 mux 狀態的能力；相反地，它們退讓，讓使用者
  自行透過 SSH 登入來完成變更

- 以它們自己的外部介面提供某種設定 mux 狀態的機制

- 假設連線到所暴露主控台端點的使用者，想要選擇該主控台（如果它
  位於 mux 之後）

第一項假設 SSH 有被暴露，且需要存取被 mux 主控台的使用者可以存取它。
目前還不清楚這是否是一個合理的期望。

第二項假設這些外部介面有能力對這個問題建模。目前尚不清楚 IPMI 或
Redfish 是否具備這種能力，而透過 SSH 的 serial 則確定沒有。

第三項意味著我們必須為全部三個元件加入能力，讓它們在收到特定主控台的
連線時驅動頻外命令介面。最終結果與 obmc-console 自行實作此事（2.3）
在行為上沒有差異，但卻增加了整個系統的複雜度。

## 實作考量

### 被 mux 的主控台在 D-Bus 上如何表示？

每個主控台都會有自己的 D-Bus 名稱，因為這與目前的實作具有
向後相容性。

多個主控台可以表示為拆分（split）或統一（unified）的物件樹。

### 在 D-Bus 上統一對比拆分物件樹的取捨

在拆分樹中，哪些主控台同屬一個 UART mux 並不清楚，但在統一樹中
這點是清楚的。

在統一樹中，一個主控台可以透過另一個主控台的 D-Bus 名稱被存取，
實際上創造了做多件事的多種方式。

範例：

```bash
busctl set-property xyz.openbmc_project.Console.host1 \
/xyz/openbmc_project/console/host2 \
xyz.openbmc_project.Console.Access Connect ""
```

因此，必須就如何在 dbus 上表示多個主控台，以及需要向其他子專案
暴露哪些資訊，做出選擇。

統一樹：

```text
busctl tree --user xyz.openbmc_project.Console.host1
└─/xyz
  └─/xyz/openbmc_project
    └─/xyz/openbmc_project/console
      ├─/xyz/openbmc_project/console/host1
      └─/xyz/openbmc_project/console/host2
```

拆分樹：

```text
busctl tree --user xyz.openbmc_project.Console.host1
└─/xyz
  └─/xyz/openbmc_project
    └─/xyz/openbmc_project/console
      └─/xyz/openbmc_project/console/host1

busctl tree --user xyz.openbmc_project.Console.host2
└─/xyz
  └─/xyz/openbmc_project
    └─/xyz/openbmc_project/console
      └─/xyz/openbmc_project/console/host2
```

表示方式的選擇會影響 mux 在 D-Bus 上如何被描述，如果選定頻外命令
策略（2.1），這點是必要的。在 D-Bus 上暴露頻外 mux 控制的兩種可能
方式是：

1. 在每個主控台物件上實作一個介面，定義一個布林 `Active`
   屬性，以及一個 `Activate()` 方法。`Activate()` 方法由於實作在
   主控台物件上，天生就擁有切換 mux 所需的全部脈絡，無需呼叫端
   提供參數。當 mux 為相關主控台配置時，`Active` 屬性為 `true`，
   否則為 `false`。針對 `Active` 變數的 `PropertiesChanged` D-Bus 訊號
   可以提醒本地使用者 mux 狀態的變更。

2. 在一個由 mux 所暴露之所有主控台共用的物件上實作一個 `Mux`
   介面。`Mux` 介面可能有一個可寫入的字串 `Selected` 屬性，
   代表 mux 的狀態，並提供一個機制將其切換到特定主控台。

這兩種方式都已在 [phosphor-dbus-interfaces 的一個現有修補程式上
討論過][pdi-uart-mux-control-interface]。

[pdi-uart-mux-control-interface]:
  https://gerrit.openbmc.org/c/openbmc/phosphor-dbus-interfaces/+/71878/comment/dd34b099_66dbc49e/

第二種方式相當明確——直接表示 mux 狀態使得發現系統狀態變得容易。
然而，它促使我們選擇統一物件樹，以提供一個共用的物件路徑來容納
`Mux` 介面（例如在 `/xyz/openbmc_project/console`）。這樣做是期望避免
「一事多種表示」問題的另一個例證——這個問題在討論統一物件樹宣告
多個匯流排名稱時曾被提及：如果樹不是統一的，這個 `Mux` 介面就必須
跨多個 D-Bus 連線在物件上被表示並同步。

第一種方式沒有這個限制。然而，它確實有前述的取捨，即系統中任何
主控台彼此如何關聯並不清楚，以及啟用其中任何一個可能帶來什麼影響。

如果我們要在 D-Bus API 中加入東西，也就是採用頻外命令設計點（2.1），
就必須為 D-Bus 表示方式選擇一個策略。然而，如果選定設計選項 2.2 或
2.3，這個選擇就更像是實作細節。在那些情況下，這個選擇反而是由我們
希望描述主控台之間關係的清晰度所驅動。

## 修剪設計決策樹

為了協助塑造這些選擇，我們有 obmc-console 現有的行為，
[已在 PDI 修補程式上討論過][pdi-uart-mux-control-interface]：

1. 我們已經支援並發的主控台伺服器實例

2. 並發主控台支援是以每個 Linux TTY 裝置對應一個
   obmc-console-server 程序的方式實作

3. 由於每個 Linux TTY 裝置都與其 obmc-console-server 程序配對，每個
   obmc-console-server 的 D-Bus 連線都需要一個唯一名稱

4. 我們使用唯一的 console-id 來命名全域資源，包括 D-Bus 連線與
   該實例的 unix domain socket。

如同連結中的討論，鑑於 `console-id` 值對於一般的非 mux 主控台，
實際上代表的是 BMC 的 TTY 裝置遠端是什麼，道理上我們應該對被 mux
的主控台延續這個策略。採用這種方式避免了在 obmc-console 中加入新的
端點 ABI，並排除了設計選項 A 到 D（含）。

此外，基於在殘留網路連線存在時會令人受挫的行為，以現有連線為由
阻止 mux 變更似乎是一條不好的前進路線。

這讓我們剩下設計選項 `F`、`G`、`I` 與 `J`，它們的區別在於 mux 如何
被切換，以及它對已連線客戶端的影響。

聚焦於 mux 如何被切換，基於上述關於 D-Bus 表示方式的討論、PDI
修補程式上的討論，以及對相關應用程式的影響，我們可以合理地說頻外
命令方式（2.1）存在一些複雜性。

相對地，我們可以考慮另一個替代方案：讓 mux 狀態反映最近一次連線的
端點。這樣做的好處是無需額外工作就能同時適用於 Unix domain socket
與 D-Bus 存取。bmcweb 與 phosphor-net-ipmid 都不需要被打修補程式。
這個選擇也消除了上述的 D-Bus 複雜性，因為不需要額外的 D-Bus 介面。

這個推理讓我們在設計選項 `F` 與 `G` 之間選擇。

`F` 與 `G` 的區別在於我們是否會中斷那些並非 mux 所選取端點上的
連線。關於這個主題在其他地方有一些反覆的討論[[1][drop-connections-discussion-1]]
[[2][drop-connections-discussion-2]]，但看起來中斷客戶端實際上會是
設計選項 `C` 一個較差的實作，而我們已經排除了 `C`。它比 `C` 更差，
因為我們可能有 `N` 個連線對應 `N` 個 mux 埠，其中 `(N - 1)` 個是閒置的，
而不是只有 1 個連線。不僅如此，那 `(N - 1)` 個連線實際上變成了殭屍，
因為它們沒有辦法將 mux 切換回它們關聯的埠，除非再建立另一個連線。
由此推論，如果我們要為了切換 mux 而建立後續連線，那不如直接中斷
現有的工作階段，既然如此，那它在 mux 一開始切換走時就被中斷也無妨
[^1]。

[drop-connections-discussion-1]:
  https://gerrit.openbmc.org/c/openbmc/obmc-console/+/71228/comment/62a5fce9_60c3ad3e/
[drop-connections-discussion-2]:
  https://gerrit.openbmc.org/c/openbmc/obmc-console/+/71867/comment/756f0abe_5ebe8d66/

[^1]: 這樣也節省了資源

這些論點綜合起來排除了除了選項 `F` 之外的所有選項。它在現有 ABI、
期望行為與實作複雜度三方面似乎都落在一個恰到好處的樞紐上。

附註：到目前為止的討論都是圍繞在一個能達成期望主控台行為的
*最小* 設計。值得注意的是，設計選項 `F`（以連線為基礎的 mux 控制，
會中斷衝突的客戶端）讓我們可以*選擇性地*額外實作一個頻外命令介面，
因為可觀察到的行為與接受一個新連線沒有不同：衝突的客戶端被中斷，
mux 被切換。這對於圍繞記錄實作平台政策可能有幫助。

## 提議的設計

我們提議使用一個 obmc-console-server 程序來暴露連接到一個 UART mux 的
`N` 個主控台，其中每個主控台代表一個 mux 埠。mux 根據最近一次客戶端
連線的端點來切換，而任何衝突的客戶端會被中斷連線。這就是上表中的
設計選項 `F`。

obmc-console 的內部資料結構將會變更以配合這個設計。

我們將為這 `N` 個被 mux 的主控台使用一個設定檔。該配置將提供一種
類似 [i2c-mux-gpio devicetree binding][linux-i2c-mux-gpio] 所使用的方式
來指定 mux GPIO。

[linux-i2c-mux-gpio]:
  https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/Documentation/devicetree/bindings/i2c/i2c-mux-gpio.yaml?h=v6.9#n12

以下是軟體與硬體元件之間關係的方塊圖：

```text
                                          +--------------------+
                                          | server.conf        |
                                          +--------------------+
                                               |
                                               |
                                               |
                                               |
                                          +----+----+                                 +-----+     +-------+
                                          |         |                                 |     |     |       |
                                          |         |     +-------+     +-------+     |     +-----+ UART1 |
 +-----------------------------------+     |         |     |       |     |       |     |     |     |       |
 | xyz.openbmc_project.Console.host1 +-----+         +-----+ ttyS0 +-----+ UART0 +-----+     |     +-------+
 +-----------------------------------+     |         |     |       |     |       |     |     |
                                          |  obmc   |     +-------+     +-------+     |     |
                                          | console |                                 | MUX |
                                          | server  |                   +-------+     |     |
 +-----------------------------------+     |         |                   |       |     |     |
 | xyz.openbmc_project.Console.host2 +-----+         +-------------------+ GPIO  +-----+     |     +-------+
 +-----------------------------------+     |         |                   |       |     |     |     |       |
                                          |         |                   +-------+     |     +-----+ UART2 |
                                          |         |                                 |     |     |       |
                                          +----+----+                                 +-----+     +-------+

```

為了讓可能在讀取主控台記錄檔的人獲得資訊，透過 mux 控制進行的
主控台連線與中斷連線事件，將會對客戶端以及在記錄檔中產生訊息。

需求是：

- 讓清楚呈現這則訊息來自 obmc-console
- 時間戳記
- 連線/中斷連線的指示

這些訊息並非作為一個 API 或取得 mux 狀態資訊的可靠途徑。 uart
另一端的任何應用程式也可能產生完全相同的訊息，即使機會不大。

這些訊息的初始格式將會類似：

```text
[obmc-console] %Y-%m-%d %H:%M:%S UTC CONNECTED
[obmc-console] %Y-%m-%d %H:%M:%S UTC DISCONNECTED
```

分別對應連線與中斷連線的情況。

對於 D-Bus 表示方式，我們選擇統一樹。

## 其他曾考慮的替代方案

### 核心實作

沒有採用，因為這個支援可以在使用者空間實作。而且它可能不會被
合併，因為它所支援的硬體配置可能並未被廣泛使用。先有一個使用者空間
實作可供日後參考也許更好，万一之後有人想進行核心實作。

### 為多個主控台使用多個 obmc-console-server 程序

這曾被考慮過並實作為一個 PoC，但後來被捨棄，因為在單一程序中同步
一切會更容易。

### 為多個主控台使用多個設定檔

這曾被考慮過，但它會重複配置，例如 mux GPIO 的定義。各檔案之間的
不一致之處也需要被管理。

## 影響

### API 影響

### 效能影響

極小到沒有。

### 開發者影響

極小。現有使用者不需要改變他們配置中的任何東西。

### 組織層面

- 這個倉庫是否需要一個新倉庫？不需要
- 誰會是這個倉庫的初始維護者？
- 預計會修改哪些倉庫來執行這個設計？
  obmc-console、docs
- 列出一份清單，並將所列倉庫的維護者加入 gerrit 審查。

## 測試

gerrit 上已經有針對這個功能的整合測試可用。
