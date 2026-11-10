# BMC 於主機開機狀態下重設

作者：Andrew Geissler (geissonator)

其他貢獻者：

建立日期：2020 年 6 月 23 日

## 問題描述

BMC 可能因為許多不同的原因而重新開機，例如使用者要求、韌體更新，以及各種不同的錯誤情境。當 BMC 在主機開機並運行的期間重新開機時，必須有一套流程讓兩者彼此同步，使 BMC 進入一個與主機狀態相符的狀態。

## 背景與參考資料

其中有相當大的一部分已在 phosphor-state-manager 的 [README][1] 中說明。

本設計文件的目的是將設計正式化，並補充更多關於同時處理與主機之間 IPMI 及 PLDM 通訊的細節，以及在無法與主機通訊時所期望的行為。

高階流程是：OpenBMC 軟體首先檢查 pgood，若已設定，接著檢查主機是否已開機並存活。若電源已開啟且主機正在執行，則會在檔案系統中建立檔案以表示此狀態：

- /run/openbmc/chassis@0-on
- /run/openbmc/host@0-on

需要注意的是，雖然對於多機箱（multi-chassis）與多主機（multi-host）系統的完整支援尚未到位，但框架已具備可供進一步建構的基礎。`op-reset-chassis-running@.service` 是一個範本化服務（templated service），會在其執行個體（instance）所屬的電源域（power domain）中檢查 pgood。它會在檔案系統中建立一個檔案 /run/openbmc/chassis@%i-on，以表示該執行個體的電源已開啟。主機也採用了類似的實作，透過 `phosphor-reset-host-check@.service` 以及檔案 /run/openbmc/host@%i-on 來完成。

若機箱電源已開啟且主機已開機，則會啟動 `obmc-chassis-poweron@.target` 與 `obmc-host-start@.target`。

/run/ 下的檔案供 OpenBMC 服務使用，以判斷當機箱與主機目標（target）啟動時是否需要執行。舉例來說，如果機箱已經開啟電源且主機已在執行，就真的不需要再實際開啟電源或啟動主機。所期望的行為是這些服務會「啟動」但什麼都不做。這通常在 systemd 服務檔案中以下列方式達成：

- `ConditionPathExists=!/run/openbmc/chassis@%i-on`
- `ConditionPathExists=!/run/openbmc/host@%i-on`

這讓目標得以啟動，並讓 BMC 與機箱及主機的狀態保持同步，而無需任何特殊的軟體檢查。

不同的系統對於「機箱電源已開啟但主機無法連線」時的行為應如何，有著不同的需求。本設計必須允許這種客製化。舉例來說，有些系統傾向只是將系統維持在其當前狀態，讓使用者自行修正。有些系統希望為使用者自動復原（亦即重啟主機）。有些系統則採用混合式做法，視主機在開機過程中的進度位置，BMC 可能維持現狀或進行復原。

## 需求

- 同時支援 IPMI 與 PLDM 作為判斷主機是否在執行的機制。
  - 允許啟用其中一種或兩種皆啟用
- 當機箱電源已開啟但 BMC 無法與主機通訊時，支援自訂行為
- IPMI 與 PLDM 堆疊（stack）都會給主機一段固定的時間來回應。若在此時間限制內未獲回應，BMC 可能會採取復原動作。
  - 此時間限制必須可在建置時（build time）設定
- IPMI 與 PLDM 將實作一個 phosphor-dbus-interface 介面 `xyz.openbmc_project.Condition.HostFirmware`，其中有一個 `CurrentFirmwareCondition` 屬性，其他應用程式可讀取該屬性以判斷主機是否在執行。

### IPMI 詳細需求

- IPMI 將持續使用 SMS_ATN 命令，向主機表示已提出「Get Message Flags Command」請求。當主機發出該命令時，即被視為已開機並正在執行

### PLDM 詳細需求

- PLDM 將使用對主機的 GetTID 命令來判斷其是否在執行
- 在適用之處，PLDM 將提供一種機制來區分不同的主機韌體堆疊
  - 舉例來說，在 IBM 系統上，hostboot（主機初始化）韌體與 Hypervisor 韌體堆疊之間有所差異。兩者都是主機韌體且都透過 PLDM 通訊，但 BMC 的復原路徑會因執行的是哪一個而不同。`CurrentFirmwareCondition` 屬性除非 Hypervisor 韌體正在執行，否則不應回傳 "Running"。

## 建議設計

高階流程：

- 檢查 pgood
- 呼叫 mapper 以取得 `xyz.openbmc_project.Condition.HostFirmware` PDI 介面的所有實作
- 讀取所有介面的 `CurrentFirmwareCondition` 屬性。若任何呼叫回傳某主機正在執行，則建立檔案並啟動主機目標。
- 否則，透過任何自訂機制檢查主機
- 如有需要，對主機執行自動化復原

IPMI 與 PLDM 軟體將視適用情況啟動。systemd 服務與 phosphor-state-manager 內的應用程式的組合，將協調 pgood 的檢查，若已設定，則要求 IPMI 與 PLDM 應用程式去探索主機是否在執行。根據這些查詢的回應，phosphor-state-manager 中的軟體將採取適當的動作——建立 /run 檔案並啟動機箱與主機目標，或進入主機的復原程序。

負責此程序的任何相關服務以及任何通用服務，將由 phosphor-state-manager  hosting。任何系統或公司專屬的服務可安裝在下列通用目標中：

- `obmc-chassis-powerreset@.target.require`
- `obmc-host-reset@.target.requires`

### 主機未回應時的自動化復原

將在 phosphor-state-manager 內建立一個獨立的服務與應用程式，在機箱電源已開啟但主機未能回應任何一種與之通訊的機制的情況下，執行下列邏輯：

- 若機箱電源已開啟 (/run/openbmc/chassis@%i-on)
- 且主機已關閉 (!ConditionPathExists=!/run/openbmc/host@%i-on)
- 且還原（restored）的 BootProgress 不為 None
- 則（假設主機在 BMC 重啟前正在開機）
  - 記錄錯誤以指出此情境
  - 將主機移至 Quiesce 狀態，讓自動化復原得以啟動

### 關於 IBM 系統自訂機制的說明

IBM 系統將使用一個處理器 CFAM 暫存器。該特定暫存器為 **Mailbox scratch register 12**。

若機箱電源已開啟，但 BMC 無法透過 IPMI 或 PLDM 與主機通訊，則 BMC 將讀取此處理器 CFAM 暫存器。

主機程式碼會將 `0xA5000001` 寫入此暫存器，以表示它已到達一個可以在不需要 BMC 的情況下開機作業系統的狀態。若 BMC 在 CFAM 暫存器中看到此值已被寫入，則即使它無法透過 IPMI 或 PLDM 與主機通訊，它也會讓主機維持現狀。它會記錄一個錯誤，指出無法與主機通訊，但同時也會將主機狀態顯示為 `Running`。

若該暫存器的值不是 `0xA5000001`，則 BMC 將遵循為主機發生失敗時所定義的任何復原機制（很可能會是重啟主機）。

在系統開機過程中視需要設定此暫存器，是主機韌體的責任。主機韌體會在上電關閉（shutdown）情境中清除此暫存器。為處理不同的主機當機（crash）情境，此暫存器也會由 BMC 韌體在電源關閉（power off）、系統檢查停止（checkstop）以及記憶體保留（Memory Preserving）重啟期間清除。

## 曾考慮的替代方案

有一種想法是完全避免使用 IPMI/PLDM，而只使用某種「最小公分母」（lowest common denominator）的硬體暫存器。這樣做的問題在於你會開始建立自己的協定，不知不覺間你又做出了類似 IPMI 或 PLDM 的東西，只不過是你自行客製的。所以有 99% 的情況，IPMI 或 PLDM 路徑就足夠了；而作為備援選項，系統擁有者可以放入他們自己的自訂主機偵測應用程式。

## 影響

無

## 測試

IPMI 與 PLDM 的正常路徑很容易測試。開機你的系統並重啟 BMC。驗證 BMC 的機箱與主機狀態是否正確，並驗證主機在整個 BMC 重設過程中持續無問題地運行。

較複雜的測試將是錯誤路徑：

- 在主機正在開機、但尚未進入可在沒有 BMC 的情況下繼續運行的狀態時重啟 BMC。驗證 BMC 在從重啟恢復後偵測到此錯誤情境，並對主機採取正確的復原動作。
- 當主機已開機並運行時重啟 BMC，但在主機上停用 IPMI/PLDM 堆疊，使其在 BMC 從重啟恢復後不回應 BMC。確認 BMC 在此情況下遵循該系統所定義的復原程序。

[1]:
  https://github.com/openbmc/phosphor-state-manager#bmc-reset-with-host-andor-chassis-on
