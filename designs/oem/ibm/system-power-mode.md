# 系統電源模式與 Idle Power Saver 支援

作者：Chris Cain

其他貢獻者：Martha Broyles

建立日期：2020/12/03

## 問題說明

POWER 平台的電源管理需要 BMC 協助管理系統電源模式與閒置省電（idle power save）模式。我們需要能夠設定並查詢系統電源模式，同時也要能控制與設定閒置省電參數。本設計僅適用於 POWER 處理器。

## 背景與參考資料

每顆 POWER 處理器內含一顆嵌入式的 PowerPC 405 處理器，稱為晶片內控制器（On Chip Controller，OCC）。OCC 提供即時電源與溫度監控及控制。當系統開機上電時，OCC 會進入 Active 狀態。每當 OCC 狀態變更為 active 時，BMC 都需要寄送一次模式變更與閒置省電（IPS，idle power saver）設定給 OCC。若使用者變更模式或閒置省電參數，BMC 同樣需要寄送這些指令。

## 需求

當系統開機啟動後，OCC 會移轉至 ACTIVE 狀態。在 ACTIVE 狀態下，OCC 負責管理處理器頻率、電源消耗，並監控系統的溫度感測器。在某些錯誤情況下，可能需要重設 OCC。此時 OCC 會離開 ACTIVE 狀態。復原之後，OCC 會被重新放回 ACTIVE 狀態。每當 OCC 狀態變更為 ACTIVE，或客戶在執行時期（runtime）更新這些新參數時，BMC 都需要將模式與閒置省電設定寄送給 OCC。

PowerMode 已加入 Redfish Schema Supplement 2021.1 版本：
<https://www.dmtf.org/sites/default/files/standards/documents/DSP0268_2021.1.pdf>

目前可由客戶設定、並會傳送至各 OCC 的系統電源模式如下：

- Static
- Power Saver
- Maximum Performance

關於新增 Idle Power Saving 參數的提案已提交給 Redfish 委員會，若經核准將會採用。

可由客戶設定的閒置省電參數：

- Enabled / Disabled
- Enter Delay Time（進入延遲時間，單位為秒）
- Enter Utilization threshold（進入使用率門檻，百分比）
- Exit Delay Time（離開延遲時間，單位為秒）
- Exit Utilization threshold（離開使用率門檻，百分比）

預設值需要可由系統擁有者設定（透過 JSON 檔案）

## 提案設計

新程式碼將屬於 openpower-occ-control 儲存庫的一部分。新程式碼會由以下事件觸發：

- OCC 輪詢（poll）回應資料顯示出新的 Active（0x03）狀態
- OCC Active 感測器啟用（可能已被上述項目涵蓋）
- 客戶更新系統電源模式的使用者介面或 Redfish 介面
- 客戶更新閒置省電設定的使用者介面或 Redfish 介面

程式碼需要依據 OCC 狀態變更來觸發。核心目前會定期（每秒一次）對 OCC 送出一個 POLL 指令。OCC 狀態包含在 OCC 的輪詢回應資料中。

當被觸發時，新程式碼會對 OCC 送出一個 SET_MODE_AND_STATE 指令（0x20），以及一個攜帶 Idle Power Saver 參數的 SET_CONFIG_DATA（0x21）指令。這些指令定義於 OCC Interface Spec：
<https://github.com/open-power/docs/blob/master/occ/OCC_P9_FW_Interfaces.pdf>

系統也會為 Power Mode 與 Idle Power Saver 參數定義預設值。若客戶尚未設定任何這些參數，將會使用這些預設值。一旦客戶設定了其中任何參數，該新的客戶參數就會變為現行值，預設值將不再被使用。

客戶要求的 PowerMode 與 Idle Power Saver 參數將會以 D-Bus 物件的形式儲存於 phosphor-dbus-interfaces 儲存庫中：
xyz/openbmc_project/Control/Power/Mode.interface.yaml 一旦設定完成，這些數值會在電源循環、AC 斷電、程式碼更新等情況下持續保留。

## 其他考量的替代方案

- 使用硬編碼（hardcoded）的電源模式與閒置省電參數（無法彈性控制系統電源用量）

## 影響

需求章節中所描述的新介面將會實作。參數應該要能透過使用者介面或透過 Redfish 來設定。API 影響——為新參數新增 Redfish 支援，並新增使用者介面以讓客戶設定電源模式與閒置省電設定
安全性影響——這些參數的更新應該要能限制為特定的使用者／群組（可能不希望任何使用者都能更新這些參數）
文件影響——需要為新參數撰寫文件
效能影響——極小，新程式碼僅在 OCC 狀態變更時執行，而通常這只會在開機時或使用者變更參數時發生一次。新程式碼僅多送出 2 個指令，應能在數秒內完成。
開發人員影響——由 OCC 團隊撰寫程式碼，並由 OpenBMC 電源管理團隊提供指導
可升級性影響——無

## 測試

這將可以直接更新電源模式與閒置省電設定來進行測試。此測試將會自動化
