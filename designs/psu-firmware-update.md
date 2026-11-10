# PSU 韌體更新

作者：Lei YU <mine260309@gmail.com> `LeiYU`

其他貢獻者：

- Su Xiao <suxiao@inspur.com>
- Derek Howard <derekh@us.ibm.com>

建立日期：2019-06-03

## 問題描述

OpenBMC 不支援更新 PSU 的韌體。

## 背景與參考

在 OpenBMC 中，已存在一套用於[軟體更新][1]的介面。

更新流程包含：

1. 將映像檔上傳到 BMC；
2. 處理映像檔以檢查其版本與用途；
3. 驗證並啟用映像檔。

目前 BMC 與 BIOS 的韌體更新已受支援：

- [phosphor-bmc-code-mgmt][2] 實作 BMC 程式碼更新，並支援上述全部 3 個流程。
- [openpower-pnor-code-mgmt][3] 實作 BIOS 程式碼更新，但僅實作「驗證並啟用」映像檔，
  並共用上述第 1、2 流程的功能。
- 上述兩者也使用相同的 [Software DBus 介面][1]。

對於 PSU 韌體更新，建議重複使用上述第 1、2 流程的相同功能。

## 需求

在以下情況下應更新 PSU 韌體：

1. 使用者手動呼叫 API 進行更新；
2. BMC 程式碼更新後，若 BMC 檔案系統中有較新的 PSU 映像檔，BMC 應更新 PSU 韌體；
3. 當 PSU 被更換且其版本比 BMC 檔案系統中的版本還舊時，BMC 應更新 PSU 韌體。
4. 可指定選擇性的 'override' 參數，在上述情況下不論哪個 PSU 映像檔較新都執行更新。
5. 有些系統可能使用不同型號的 PSU，因此需要支援不同的 PSU 韌體映像檔。

對某些 PSU 而言，在主機執行中進行 PSU 程式碼更新會有斷電風險。這應由 PSU
廠商專屬工具處理，而非在通用框架中處理。

註：下文所稱的「廠商專屬」是指 PSU 廠商專屬。

因此以下檢查為選擇性項目，預期由廠商專屬工具處理：

1. 主機是否已關機；
2. 備援 PSU 是否全部連接；
3. 所有 PSU 的交流輸入與直流待機輸出是否正常；

## 建議設計

如上述需求所述，PSU 韌體被更新的情況有幾種：

- 當 API 被呼叫時；
- 當有新版本隨 BMC 程式碼更新一併更新時；
- 當 PSU 被更換為較舊韌體版本時。

### 透過 API 更新

此方法通常供手動更新 PSU 韌體的使用者使用。

它會重複使用現有的介面來上傳、驗證並啟用映像檔。

1. 需要擴充 "Version" 介面：
   - 為 PSU 新增一個新的 [VersionPurpose][4]；
   - 重複使用現有的 `ExtendedVersion` 作為廠商專屬用途的額外字串，例如用來表示
     PSU 型號。
2. 重複使用 [phosphor-bmc-code-mgmt][2] 實作的現有功能來上傳並處理映像檔。
   - PSU 更新映像檔應是一個 tarball，包含 MANIFEST、映像檔與簽章。
   - 當 PSU 映像檔被上傳並處理後，應建立一個 `VersionObject` 來表示其版本與用途。
3. 將有一個新的服務實作 [Activation][5] 介面來更新 PSU 韌體。
   - 該服務會在 BMC 啟動時預設啟動；
   - 啟動時，服務會檢查 PSU 現有的韌體並建立 `Version` 與 `Activation` 介面。
   - 服務會監聽在 `/xyz/openbmc_project/Software` 上新增的介面。
   - 當新增了帶有 PSU `VersionPurpose` 的物件時，服務會驗證映像檔的簽章；
   - 服務應檢查 `ExtendedVersion` 以確認映像檔符合 PSU 型號。
   - 服務會有一份設定檔來描述 PSU 型號及其相關的廠商專屬工具。
   - 服務會找到相符的廠商專屬工具來執行程式碼更新。舉例來說，若在
     `psu-update@foo.service` 中設定了廠商專屬工具 `foo`，其執行
     `foo psu.bin`，則服務會找到 `psu-update@foo.service` 並透過 systemd 啟動它，
     由其執行更新。
   - 當 PSU 程式碼更新完成時，應建立一筆資訊層級的事件記錄。
   - 當 PSU 程式碼更新完成時，映像檔、MANIFEST 以及選擇性的簽章會被儲存到
     讀寫檔案系統中一個預先定義的目錄以供日後使用，以防插入了帶有舊韌體的新 PSU。
4. 廠商專屬工具應在 PSU 更新前後執行其需要執行的所有檢查，並回傳狀態給上述服務
   以表示結果。
5. 當廠商專屬工具回傳錯誤時，PSU 更新將中止，並應建立一筆錯誤事件記錄。
6. 在更新期間，服務應將相關感測器設為非功能狀態，更新完成後再將相關感測器設回
   功能狀態。

### 透過新 BMC 映像檔更新

當 BMC 更新且內含較新版本的 PSU 韌體時，應更新至 PSU。這將由上述同一個服務完成。

1. 啟動時，服務會檢查其檔案系統中的 PSU 映像檔、型號與版本，並與 PSU 硬體中的
   版本比較，以決定是否應執行 PSU 韌體更新。
2. 可能有兩處放置 PSU 映像檔：
   - 讀唯檔案系統中預先定義的目錄，它是 BMC 映像檔的一部分。
   - 讀寫檔案系統中另一個預先定義的目錄，它是透過 API 更新時儲存 PSU 映像檔的
     位置。兩處都應檢查，並選取較新的版本來與 PSU 硬體比較。
3. 若需要 PSU 更新，服務會找到相符的廠商專屬工具來執行程式碼更新。
4. 後續流程將與[透過 API 更新]相同。

### 更換 PSU 時更新

當 PSU 被更換，且其韌體版本比 BMC 檔案系統中的版本還舊（或若指定了選擇性的
'override' 參數）時，應進行更新。這將由上述同一個服務完成。

1. 啟動時，服務會訂閱 PSU 物件路徑的 PropertiesChanged 訊號以監控 PSU 的存在狀態。
   （或者訂閱 InterfacesAdded/Removed 訊號？）
2. 當某 PSU 的存在狀態從 false 變為 true（或發生 PropertiesChanged 事件）時，
   服務會檢查新 PSU 的型號與韌體版本，以決定是否需要更新韌體。
3. 若是，服務會找到相符的廠商專屬工具來執行程式碼更新。
4. 後續流程將與[透過 API 更新]相同。

## 已評估的替代方案

### 通用實作

PSU 韌體更新可以由獨立的 recipe 實作，僅呼叫廠商專屬工具。這樣會稍微簡單些，
但會失去 OpenBMC 現有[軟體更新介面][1]所提供的統一介面，因此將難以使用標準 API
來進行 PSU 韌體更新。

### VersionPurpose

可以重複使用 `VersionPurpose.Other` 來表示 PSU 映像檔的版本用途。但那需要關於
映像檔的額外資訊，否則無法分辨該映像檔是用於 PSU、CPLD 還是其他週邊設備。
新的 `VersionPurpose.PSU` 更具體，讓實作更容易且對使用者更友善。

### 額外字串

本設計提案使用 `ExtendedVersion` 作為廠商專屬用途的額外字串，例如用來表示 PSU
型號，以便實作時可以檢查並比較映像檔是否符合 PSU 型號。也可以讓它變成選擇性項目
或移除這個額外字串，如此實作就不會驗證映像檔是否符合 PSU。如果我們信任上傳正確
映像檔的使用者，尤其是映像檔應當經過簽章，這樣做或許可以接受。但万一映像檔不符合
PSU，或在更新了錯誤的 PSU 韌體時造成非預期的損壞，始終是有風險的。

## 影響

本設計僅在 dbus 介面中引入一個新的 `VersionPurpose` 列舉值。新引入的 PSU 韌體更新
服務將是一個實作現有 [Activation][5] 介面的新服務。將有新的設定檔供服務使用，以：

- 將廠商專屬工具與 PSU 型號連結。
- 取得與 PSU 相關的感測器。
- 等等。

因此對現有系統的影響極小。

## 測試

需要手動測試來驗證 PSU 程式碼更新流程。

- 驗證 PSU 程式碼更新在所有 PSU 上皆成功完成；
- 驗證當廠商專屬工具在前置條件檢查失敗，或在更新 PSU 時失敗時，PSU 程式碼更新會
  失敗。
- 驗證在更新了內含 PSU 韌體新版本（或使用 'override' 時為不同版本）的新 BMC
  映像檔後，會執行 PSU 程式碼更新。
- 驗證在插入了帶有舊韌體（或使用 'override' 時為不同版本）的 PSU 後，會執行 PSU
  程式碼更新。

[1]:
  https://github.com/openbmc/phosphor-dbus-interfaces/tree/master/yaml/xyz/openbmc_project/Software
[2]: https://github.com/openbmc/phosphor-bmc-code-mgmt/
[3]: https://github.com/openbmc/openpower-pnor-code-mgmt/
[4]:
  https://github.com/openbmc/phosphor-dbus-interfaces/blob/57b878d048f929643276f1bf7fdf750abc4bde8b/xyz/openbmc_project/Software/Version.interface.yaml#L14
[5]:
  https://github.com/openbmc/phosphor-dbus-interfaces/blob/master/yaml/xyz/openbmc_project/Software/Activation.interface.yaml
