# 電源硬體抽象層（pHAL）的錯誤處理

作者：Devender Rao <devenrao@in.ibm.com> `devenrao`

其他貢獻者：無

建立日期：14/01/2020

## 問題描述

提出一個機制，將電源硬體抽象層（pHAL）函式庫呼叫所擷取的失敗資料，轉換為
[Platform Event Log][1]（PEL）格式。

## 背景與參考資料

OpenBMC 應用程式使用 pHAL 層進行硬體存取與硬體初始化，pHAL 層回傳的任何
軟體／硬體錯誤都需要轉換為 PEL 格式，以便記錄錯誤項目。PEL 有助於在產品開發、
製造以及客戶環境中，改善韌體與平台的可服務性。

錯誤資料包含暫存器資料、要 [guard][2] 的目標，以及 callout。Guard 指的
是「保護（guarding）」故障硬體、使其不影響後續系統動作的動作。Callout 則指向
伺服器內與已識別錯誤相關的特定硬體。

建立 PEL 會使用 [Phosphor-logging][3] 的 [Create][4] 介面。

pHAL 層包含下列函式庫，而這些函式庫會回傳不同的回傳碼。

1. libipl，用於初始程式載入（initial program load）
2. libfdt，用於裝置樹存取
3. libekb，用於硬體程序執行
4. libpdbg，用於硬體存取

本提案將回傳資料結構化為標準回傳碼格式，讓呼叫端只需處理單一回傳碼格式，
即可轉換為 PEL。

### 術語表

pHAL：電源硬體抽象層（power Hardware Abstraction Layer）。pHAL 是一組執行於
BMC 上的函式庫。Open Power 專屬應用程式會使用這些函式庫，進行複雜的硬體互動、
hostboot 與 Self Boot Engine 初始化、診斷與偵錯。

libfdt：pHAL 使用它來建構所有目標的記憶體中樹狀結構。[參考資料][5]

libpdbg：讓 BMC 得以從主控端偵錯 POWER 處理器的函式庫。[參考資料][6]

MRW：機器可讀取工作手冊（Machine readable workbook）。由系統擁有者所指定、
以 XML 描述的機器說明。

HWP：硬體程序（Hardware procedure）。由硬體團隊提供的「黑盒子」程式碼模組，
以與平台無關的方式初始化主控端處理器與記憶體子系統。

Device Tree（裝置樹）：裝置樹是一種資料結構，用以描述特定電腦的硬體元件，
讓作業系統核心能夠使用並管理這些元件，包含 CPU、記憶體、匯流排與週邊設備。
[參考資料][7]

EKB：EKB 函式庫包含特定平台的所有硬體程序（HWP），以及每個硬體程序對應的
錯誤 XML 檔案。

PEL：[Platform Entity Log][1]

## 需求

### libekb

EKB 函式庫包含特定平台的硬體程序，以及每個硬體程序對應的錯誤 xml 檔案。
錯誤 XML 會針對特定錯誤，指定屬性資料、要 callout 的目標、要 guard 的目標，
以及要 deconfigure 的目標。EKB 函式庫中的解析器會解析錯誤 XML 檔案，並產生
C++ 標頭檔，供硬體程序在擷取失敗資料時使用。

在 libekb 中加入解析器以解析錯誤 XML 檔案，並提供可解析硬體程序方法所回傳
失敗資料的方法，將資料以鍵值（key、value）配對的形式回傳，使其可用於建立
PEL。

### libipl

初始程式載入函式庫，用於開機系統。該函式庫內部會呼叫 EKB 函式庫的硬體程序
（HWP）。硬體程序的執行狀態需要回傳給呼叫端，讓呼叫端能在硬體程序執行失敗時
建立 PEL。

### libpdbg

libpdbg 函式庫用於硬體存取，任何硬體存取錯誤都需要擷取並納入 PEL。

### 訊息登錄項目（Message Registry Entries）

若要在 pHal 中引發錯誤，需要在[訊息登錄][8]中建立對應的錯誤訊息登錄項目。

## 設計提案

### 硬體程序失敗

在 libekb 中加入解析器以解析錯誤 XML 檔案，並提供可解析硬體程序方法所回傳
失敗資料的方法，將資料以鍵值（key、value）配對的形式回傳，使其可用於
[Create][4] 介面以建立 PEL。

需要將要 Callout／Guard／Deconfig 目標的庫存（inventory）字串，加入 Create
介面的額外資料（additional data）區段。

應用程式需要在 libekb 函式庫中註冊回呼（callback）方法，以取回錯誤記錄追蹤
資訊。

透過回呼方法回傳的偵錯追蹤資訊將加入 PEL。

### libipl 內部失敗

應用程式需要在 libipl 函式庫中註冊回呼（callback）方法，以取回錯誤記錄追蹤
資訊。

透過回呼方法回傳的偵錯追蹤資訊將加入 PEL。

### libpdbg 內部失敗

應用程式需要註冊回呼（callback）方法，以從 libpdbg 函式庫取得偵錯追蹤資訊。

透過回呼方法回傳的偵錯追蹤資訊將加入 PEL。

## 序列圖

### 註冊偵錯追蹤與開機錯誤

![image](https://user-images.githubusercontent.com/26330444/76838214-e4e7dc80-6859-11ea-818c-031bf5a191d6.png)

### 處理偵錯追蹤

![image](https://user-images.githubusercontent.com/26330444/76838355-152f7b00-685a-11ea-9975-4091ae1064cc.png)

### 處理開機失敗

![image](https://user-images.githubusercontent.com/26330444/76838503-3a23ee00-685a-11ea-9f2a-559e233b408f.png)

## 已考慮的替代方案

無

## 影響

無

## 未來變更

目前使用 [Create][4]，並以 std::map 格式提供資料；當加入可將 JSON 檔案傳遞給
Create 介面對應支援後，將會改為 JSON 格式。

## 測試

1. 模擬硬體程序失敗，並檢查是否已建立 PEL。

[1]:
  (https://github.com/openbmc/phosphor-logging/blob/master/extensions/openpower-pels/README.md)
[2]:
  (https://gerrit.openbmc.org/#/c/openbmc/docs/+/27804/2/designs/gard_on_bmc.md)
[3]: (https://github.com/openbmc/phosphor-logging)
[4]:
  (https://github.com/openbmc/phosphor-dbus-interfaces/blob/master/yaml/xyz/openbmc_project/Logging/Create.interface.yaml)
[5]: (https://github.com/dgibson/dtc)
[6]: (https://github.com/open-power/pdbg)
[7]: (https://elinux.org/Device_Tree_Reference)
[8]:
  (https://github.com/openbmc/phosphor-logging/blob/master/extensions/openpower-pels/registry/message_registry.json)
