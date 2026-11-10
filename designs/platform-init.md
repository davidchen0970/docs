# 平台初始化

作者：Ed Tanous (edtanous)

建立日期：2025-07-21

## 問題說明

有許多平台需要處理某些狀態，而這些處理必須以整合的方式與硬體互動。
一些例子如下：

1. 重置一個初始狀態下沒有供電的 i2c 驅動器。
2. 在 BMC 的控制下啟動一條電源軌。
3. 依據 plug detect pin 的狀態來初始化一條電源軌。

## 背景與參考

以及其他各種類似的狀況。此類初始化的具體範例可以在這裡找到。[1]

實務上經常使用大量的 bash 腳本來執行部分此類初始化工作。這些
bash 腳本在啟動時會有些微的效能損耗，而且通常會被簽入 meta layer 中，
降低了各平台之間的重複使用。

## 需求

1. OpenBMC 必須能夠初始化一個系統，其上的裝置位於由 OpenBMC 控制的
   電源軌上。
2. 在控制前述電源軌的過程中，OpenBMC 必須能夠監測 GPIO 訊號。
3. OpenBMC 必須能夠重置那些可能無法使用的 i2c 線路。
4. （次要需求）必須能夠從單一 binary（dc-scm）初始化多個平台。

## 提案設計

起初，會建立一個新的 repository 來存放 gb200nvl 平台的 platform-init，
並取得靜態分析 autobumps，以及在各平台之間重複使用的 recipes。

各平台會以資料夾來區隔，以確保平台擁有者能夠依需求審查並建立其平台
設定方式，與現存的 meta-layer 流程相同。

頂層結構將為

platform_init.cpp platform1/init.cpp platform2/init.cpp

起初會使用一個 cli 參數來為特定平台編譯適當的路徑。未來某個時間點，
_可能_會加入某種程度的偵測機制，以便在執行期偵測平台。該部分的設計
將依未來的需求再行決定。

Guard rails 需要隨著時間演進，但起初這個 repository 不會與 DBus 或
OpenBMC model 互動。依照（目前）的定義，它純粹只用於硬體的初始啟動。

## 曾考慮的替代方案

維持現狀，繼續以 bash 建置初始化腳本。這會導致缺乏程式碼重複使用，
雖然某些 layer 在維持程式碼整潔方面做得很好，但要進行重構或整合仍然
非常困難。

## 影響

1. 將需要一個新的 repository、recipe 與 CI build。
2. 將會為選擇採用 platform-init 的平台啟動一個新的服務。

### 組織方面

- 這個提案是否需要新的 repository？需要。repository 的申請已經提出 [2]
- 誰會是這個 repository 的初始維護者？Ed Tanous 與 Patrick Williams
- 預期需要修改哪些 repository 來執行這個設計？
  openbmc/openbmc 新的 repository

## 測試

測試章節省略。平台開機將使用各個別平台來測試，不需做任何變更。

[1]:
  https://github.com/openbmc/openbmc/blob/master/meta-nvidia/meta-gb200nvl-obmc/recipes-nvidia/platform-init/files/platform_init.cpp
[2]: https://github.com/openbmc/technical-oversight-forum/issues/51
