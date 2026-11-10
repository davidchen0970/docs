# 穩壓器設定

作者：Shawn McCarney（Discord ShawnMcCarney）

其他貢獻者：Derek Howard（Discord derekh55）、Matt Spinler（Discord
mspinler）

建立日期：2019-07-13

## 問題描述

[穩壓器][1]有許多可設定的屬性，例如輸出電壓、過電流限制以及 pgood
臨界值。這些設定通常取決於系統類型與供電軌（rail）類型。穩壓器具有一組由硬體工程師在系統設計早期定義的硬體預設設定。然而，這組預設設定有時必須由韌體修改。因此需要一個新的應用程式來設定穩壓器。它應該是資料驅動的，以支援多種穩壓器類型並避免硬編碼的邏輯。

## 背景與參考資料

### 穩壓器設定資料（「設定檔」）

硬體工程師必須為穩壓器指定許多低階的設定值。一些簡單的例子包括輸出電壓、過電流限制以及
pgood 臨界值。這些值通常會依系統類型與供電軌類型而有所不同。

依穩壓器類型而定，硬體工程師可能會將設定值輸入到某個工具中，該工具會產生一個「設定檔」。

穩壓器的設定資訊會傳送到製造端，並儲存在穩壓器上的非揮發性記憶體中。這提供了硬體／上電時的預設設定。

這些預設設定值後來經常需要更改。可能的原因包括以下幾點：

- 在硬體測試期間發現的新資訊。例如，下游客體硬體需要較高的電壓，或耗用的電流比預期更多。
- 硬體替代方案（workaround）。穩壓器硬體或相關硬體的問題有時需要新的設定值來排除問題。這些問題可能已在新版硬體中解決，但韌體仍必須支援舊版硬體。基於這個原因，硬體替代方案有時是有條件的，僅套用於具有特定版本暫存器值或[關鍵產品資料（VPD）][2]
  關鍵字值的穩壓器。
- 提升製造良率。有時必須更改穩壓器的設定值，讓更多硬體能通過製造測試。例如，可能需要提高電壓以克服輕微的製造缺陷。

### OpenBMC 穩壓器設定腳本

在某些 OpenBMC 系統上，穩壓器的設定變更會使用 shell
腳本來執行。例如，以下腳本會在 Witherspoon 系統上設定穩壓器：

- [vrm-control.sh][3]
- [power-workarounds.sh][4]

### Hwmon 與 IIO 裝置驅動框架

Linux 的[硬體監控 Kernel API（hwmon）][5]與[工業 I/O
子系統（IIO）][6]提供了用於監控硬體並讓應用程式可取得感測器值的裝置驅動框架。它們並未提供可用於執行低階穩壓器設定的介面。

### 電壓與電流穩壓器裝置框架

Linux 的[電壓與電流穩壓器 API][7]為電壓與電流穩壓器提供了裝置驅動框架。它提供了一種機制，讓「消費端（consumer）」裝置的裝置驅動可以向穩壓器請求特定的電壓或電流準位。它並未提供可用於執行低階穩壓器設定的介面。

## 需求

- 提供修改任何透過 PMBus 或 I2C 連線至 BMC
  的電壓穩壓器設定的能力。
- 在開機早期、穩壓器啟用之前套用這些設定變更。
- 若在嘗試設定某個穩壓器時發生錯誤，記錄該錯誤並繼續處理下一個穩壓器。
- 從執行時期讀取的資料檔讀取設定變更。
- 提供一種方法，可透過將新的資料檔複製到 BMC
  來測試新的設定變更。

### 非需求

- 啟用／停用電壓穩壓器並監控其 pgood 訊號。
  - 這由電源循序器（power sequencer）晶片及相關韌體處理。
- 在穩壓器啟用之後修改其設定。
  - 在系統執行期間修改穩壓器設定，通常是由主機端使用不同的匯流排連線來完成。
- 驗證系統中存在正確數量與類型的穩壓器。
- 穩壓器的並行維護或熱插拔，即在系統執行期間移除／新增／更換穩壓器。

## 建議設計

### 應用程式

將建立一個名為 `phosphor-regulators` 的新應用程式來設定電壓穩壓器。該應用程式將放置於建議的新
`phosphor-power` 存放庫中。

_方向說明：此應用程式未來將提供其他以穩壓器為基礎的功能，例如偵測冗餘相位故障。_

### 應用程式資料檔

該應用程式將在執行時期讀取一個 JSON
檔案，該檔案定義所有穩壓器的設定變更。這個 JSON
檔案會依系統類型而不同，因此它會儲存在適當建置層級（例如
meta-ibm）的 GitHub 存放庫中。

BMC 上的 JSON 檔案位置：

- 標準目錄：`/usr/share/phosphor-regulators`（唯讀）
- 測試目錄：`/etc/phosphor-regulators`（可寫入）

可以透過將新版本的 JSON 檔案複製到測試目錄來測試它。該應用程式會搜尋這兩個目錄，而測試目錄中的檔案會覆寫標準目錄中的檔案。若應用程式收到
SIGHUP 訊號，它會重新讀取 JSON 檔案。

將提供一份文件，說明 JSON
檔案中可出現的物件與屬性。同時也將提供一個驗證工具，用來檢查 JSON
檔案的語法與內容。

_方向說明：未來將使用這個 JSON 檔案來定義其他穩壓器操作，例如如何偵測冗餘相位故障。_

### D-Bus 介面

將建立以下新的 D-Bus 介面：

- 名稱：`xyz.openbmc_project.Regulator.Collection.Configure`
- 說明：設定所有電壓穩壓器。
- 屬性：無
- 方法：Configure()

_方向說明：適用於所有穩壓器的新介面將使用相同的
xyz.openbmc_project.Regulator.Collection
命名空間。適用於單一穩壓器的新介面將使用 xyz.openbmc_project.Regulator
命名空間。_

### D-Bus 路徑

新的 `xyz.openbmc_project.Regulator.Collection.Configure`
介面將實作於物件路徑 `/xyz/openbmc_project/regulators` 上。

_方向說明：適用於所有穩壓器的新介面將實作在同一個物件路徑上。各個穩壓器將由物件路徑
`/xyz/openbmc_project/regulators/<regulator_name>`
表示，適用於單一穩壓器的介面將實作在該路徑上。_

### Systemd

該應用程式將在 BMC
處於待機狀態時，使用 systemd 啟動。服務檔將命名為
`xyz.openbmc_project.Regulators.service`。

在機殼上電的開機過程中，將呼叫新的
`xyz.openbmc_project.Regulator.Collection.Configure`
介面上的 Configure()
方法。這必須在啟用穩壓器（為其上電）的 systemd
目標（target）之前完成。

### 穩壓器裝置介面

該應用程式將使用 [i2c-dev][8] 使用者空間
I2C 介面直接與電壓穩壓器通訊。之所以採用直接通訊而非裝置驅動，是因為目前的
Linux 裝置驅動框架並未提供所需的功能。請參閱[已考慮的替代方案](#alternatives-considered)一節以取得更多資訊。

使用此應用程式設定的電壓穩壓器，將不會在 Linux
裝置樹（device tree）中綁定到裝置驅動。當系統上電時，該應用程式會取得穩壓器的感測器值，並以與
`phosphor-hwmon` 相同的格式將它們儲存在 D-Bus 上。

_方向說明：若未來開發出支援低階穩壓器設定的驅動框架，則將增強此應用程式以使用這些驅動。目標是讓此應用程式支援具彈性的裝置介面，在可能的情況下使用驅動，必要時則使用
i2c-dev。_

## 已考慮的替代方案

### 使用標準 Linux 裝置驅動來設定穩壓器

理想上，可以使用下列其中一個裝置驅動框架來設定穩壓器，而非使用
i2c-dev：

- [硬體監控 Kernel API][5]
- [工業 I/O 子系統][6]
- [電壓與電流穩壓器 API][7]

不幸的是，這些都沒有所需的功能：

- 能夠執行依系統與依供電軌而定的低階設定。
- 能夠根據穩壓器版本暫存器值或 [VPD][2]
  值執行動態設定。
- 能夠快速且反覆地測試新的設定值，而不需修改裝置驅動或重新建置核心。

曾與 Joel Stanley 及 Andrew Jeffery
召開會議討論此問題。他們認為長期而言可以設計出一個新的驅動框架來提供此功能，並能被
Linux 上游社群接受。若發生這種情況，將修改此應用程式以使用新的驅動框架，而不再從使用者空間執行直接的
I2C 通訊。

### 使用 Shell 腳本來設定穩壓器

目前在某些
OpenBMC 系統（例如 Witherspoon）上，會使用 shell
腳本來設定穩壓器。在開機期間，這些 shell
腳本會解除裝置驅動的綁定，執行 `i2cset` 命令來設定穩壓器，然後重新綁定裝置驅動。

使用 shell 腳本有以下缺點：

- 通常不會執行錯誤檢查來確認 `i2cset`
  命令是否成功。若要根據回傳碼加入錯誤檢查，將難以記錄包含所有必要資訊的適當錯誤訊息。
- 在具有許多穩壓器與暫存器更新的大型系統上，shell
  腳本的執行速度會比應用程式慢，並對開機速度造成負面影響。每個 `i2cset`
  命令都需要啟動一個獨立的子程序。
- 腳本通常是硬編碼用於單一系統類型，並未提供通用的、資料驅動的解決方案。

## 影響

- 預期不會對現有的 API、安全性、文件、效能或可升級性造成重大影響。
- 新的 D-Bus 介面已記錄在[建議設計](#proposed-design)一節中。
- 該應用程式應該能使用 i2c-dev
  以與使用 `i2cset` 的等效 shell
  腳本相同或更快的速度來設定穩壓器。
- 穩壓器的感測器值將以與 `phosphor-hwmon`
  一致的方式儲存在 D-Bus 上。

## 測試

- 將撰寫自動化測試案例，盡可能測試應用程式程式碼的各個部分。
- 將使用模擬的裝置介面來測試是否發生了正確的 I2C
  讀取與寫入，而不需存取實體硬體。
- 將執行端到端的開機測試，以確保發生了正確的 I2C
  讀取／寫入、系統能正常開機，且不會發生錯誤。
- 會開機系統的 CI 測試將間接測試此應用程式。

[1]: https://en.wikipedia.org/wiki/Voltage_regulator_module
[2]: https://en.wikipedia.org/wiki/Vital_Product_Data
[3]:
  https://github.com/openbmc/meta-ibm/blob/master/meta-witherspoon/recipes-phosphor/chassis/vrm-control/vrm-control.sh
[4]:
  https://github.com/openbmc/meta-ibm/blob/master/meta-witherspoon/recipes-phosphor/chassis/power-workarounds/witherspoon/power-workarounds.sh
[5]: https://www.kernel.org/doc/html/latest/hwmon/hwmon-kernel-api.html
[6]: https://www.kernel.org/doc/html/latest/driver-api/iio/index.html
[7]: https://www.kernel.org/doc/html/latest/driver-api/regulator.html
[8]: https://www.kernel.org/doc/Documentation/i2c/dev-interface
