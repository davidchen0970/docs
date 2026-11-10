# OpenBMC 中 Device Tree 的 GPIO 命名

作者：Andrew Geissler (geissonator)

其他貢獻者：< 無 >

建立日期：2020 年 4 月 3 日

## 問題說明

Linux 核心已棄用（deprecated）使用 sysfs 來與 GPIO 子系統互動。取代它的是「基於描述元（descriptor-based）」的字元裝置介面。

[libgpiod][1] 是一套用 C 與 C++ 實作的工具程式與函式庫集合，為這個新的字元裝置 GPIO 介面提供了一層抽象。

libgpiod 提供了一項功能，讓你可以透過在核心 device tree 檔案中為 GPIO 所取的名稱來存取它們。問題在於這些 GPIO 名稱並沒有命名慣例，如果你希望使用者空間（userspace）的程式碼能夠在不同機器之間保持一致，這些名稱就必須是一致的。

## 背景與參考資料

核心的[文件][2]對 GPIO 子系統有良好的摘要說明。在 DTS 中用來為 GPIO 命名的特定欄位是 `gpio-line-names`。這個[補丁（patch）][3]展示了一個為系統命名 GPIO 的範例。

GPIO 被用於各種任意的用途。面對無窮無盡的潛在使用情境，要有一套連貫一致的命名方案其實是相當困難的。

把問題範圍縮小到 OpenBMC 的廣闊領域，就能大幅縮小可能性，並讓一套命名方案得以浮現。

## 需求

- 確保 OpenBMC 中功能常見的 GPIO 採用相同的命名慣例

## 提議的設計

以下是標準的分類。每一節中的「Pattern（樣式）」描述命名慣例，接著的子項目清單列出會使用的常見 GPIO 名稱（在 OpenBMC 系統上可用時）。所有常見的 GPIO 都必須遵循這個命名慣例。

下面的清單包含了 OpenBMC 中所有常見的 GPIO。任何提供下列其中一個 GPIO 的 OpenBMC 系統，都必須以本文件所列的方式為其命名。當新增常見的 GPIO 時，必須更新本文件。

### 按鈕（Buttons）

Pattern: `*-button`

按鈕都應該被視為對 BMC 的「輸入（input）」。可能會有一些沒有 BMC 的機器使用按鈕來觸發系統行為，而在由 BMC 管理的系統中，這個訊號是由 BMC 模擬產生（作為輸出）。這些在本文件中不應被視為按鈕，而應列為另一種訊號類型。

#### power-button

### Host Ready

以下是 Host ready 專用的輸入 GPIO 名稱。Host ready GPIO 的名稱取決於 Host 的索引，而有效狀態（active state）為高準位或低準位。

Pattern:

- `host*-ready`：Host ready，高準位有效（active high）
- `host*-ready-n`：Host ready，低準位有效（active low）

已定義：

- host0-ready
- host1-ready-n
- ...

### LED

Pattern: `led-*`

#### led-fault

#### led-identify

#### led-power

#### led-sys-boot-status

#### led-attention

#### led-hdd-fault

#### led-rear-fault

#### led-rear-power

#### led-rear-id

### 電源與穩壓器（Power and Regulators）

Pattern: `power-*`、`regulator-*`

#### power-chassis-control

設定此值以啟動機箱（chassis）的開機或關機。

#### power-chassis-good

表示機箱的電源正常（power good）狀態。

#### power-config-full-load

由電源管理應用程式設定的輸出 GPIO，用來向硬體指出：對於支援兩種不同電源供應器數量（例如 2 個或 4 個）之機型的系統，此類型系統預期會存在且正常運作的電源供應器數量門檻。預設值為這兩種機型中預期的較低電源供應器數量。當現有的電源供應器數量少於此 GPIO 所指示的數量時，硬體即可採取因應動作，例如降低系統效能，以維持其開機狀態。

#### power-ffs-sync-history

由電源管理應用程式設定的輸出 GPIO。IBM Common Form Factor 電源供應器將此線路作為輸入使用。當設定為低準位並維持低準位時，電源供應器的風扇會以全速運轉（Fans Full Speed）。當由高準位切換為低準位再回到高準位（toggled low, then high）時，會重設電源供應器的輸入歷史紀錄（一段時間範圍內的平均與最大功率值）。

#### regulator-standby-faulted

此 GPIO 值代表待機（standby）電源穩壓器故障偵測邏輯的狀態。此 GPIO 僅作為輸入。當交流電源重新上電（AC power cycle）後且不存在待機電源穩壓器故障狀況時，狀態將反映穩壓器無故障（non-faulted）的情況。當偵測到待機電源意外下降時，狀態將反映待機穩壓器電源故障（faulted）的情況。

#### rtc-battery-voltage-read-enable

輸出接腳，讓 ADC 得以讀取主機板上 RTC 電池的電壓。

### 存在偵測（Presence）

Pattern: `presence-*`

#### presence-ps0、presence-ps1、...、presence-ps\<N>

### 重設原因（Reset Cause）

這些 GPIO 提供關於 BMC 重設原因的更多細節。BMC 硬體通常會在 BMC 重新啟動時提供一些資訊，例如 EXTRST（也就是 BMC 的重設是由某個外部來源所觸發）。但有時候韌體需要關於重設原因的更多細節。硬體可以被設定為將某個事件鎖存（latch）到一個 GPIO 中，供韌體後續用於不同的軟體邏輯。

Pattern: `reset-cause-*`

#### reset-cause-pinhole

BMC 韌體會使用 pinhole 重設原因，來得知自己何時是因為使用者主動觸發的 pinhole 重設而被重設。這通常發生在錯誤情境下，例如 BMC 當住（hanging）或以其他方式無回應。請注意，此 GPIO 並非用於實際觸發重設，它是一個可以在 BMC 重設之後被讀取的 GPIO，用來得知重新啟動的原因是一次 pinhole 重設。

### 安全開機（Secure Boot）

#### bmc-secure-boot

輸入接腳，表示 BMC 已被設定為在強制執行安全防護的情況下開機。

預設為上拉（pull up，安全）。置入跳線（jumper）會將此接腳下拉（繞過安全機制）。

### 特殊（Special）

這些是特殊案例及/或沿用（grandfathered in）的接腳名稱。

#### air-water

表示系統是以氣冷還是水冷方式散熱

#### factory-reset-toggle

軟體記錄此 GPIO 的狀態，並在重新啟動時檢查該狀態自上次重啟以來是否有所改變。如果有改變，即表示應該執行原廠重設（factory reset）。

### POWER 專屬 GPIO

以下是 POWER 處理器架構伺服器專用的 GPIO 名稱。

#### 特殊（Special）

這些是特殊案例及/或沿用（grandfathered in）的接腳名稱。

##### cfam-reset

用於對 IBM POWER 處理器發出處理器邏輯重設（processor logic reset）。

##### checkstop

用於表示 IBM POWER 處理器已進入無法回復的錯誤狀態。

## 曾考慮的替代方案

- 針對每種系統類型持續硬編碼（hard code）一個設定檔，其中包含 gpio bank 與接腳編號。這樣可以移除對 device tree 必須具備一致名稱的依賴，但會增加支援每個新系統的負擔。

- 讓 device tree 的 GPIO 名稱與硬體電路圖相符，另外再透過一個使用者空間的設定檔，在電路圖名稱與邏輯接腳名稱之間進行對應。這樣可以讓 GPIO 對電路圖的對應變得簡單，但因使用者空間設定檔而多增加了一層額外的工作。

## 影響

需要確保 OpenBMC 的 device tree 符合上述命名慣例。

## 測試

使用者空間對 GPIO 名稱的使用，將在 CI 期間提供一些測試涵蓋率。

[1]: https://git.kernel.org/pub/scm/libs/libgpiod/libgpiod.git/about/
[2]: https://www.kernel.org/doc/html/latest/driver-api/gpio/index.html
[3]:
  https://lore.kernel.org/linux-arm-kernel/20200306170218.79698-1-geissonator@yahoo.com/
