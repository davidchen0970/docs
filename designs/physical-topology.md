# 盤點項目的實體拓撲

作者：Benjamin Fair `benjaminfair`

其他貢獻者：Ed Tanous `edtanous`

建立日期：2022 年 6 月 1 日

## 問題描述

複雜系統可能包含許多盤點物件（例如機箱、電源供應器、纜線、風扇等），
而這些物件之間存在不同類型的關係。舉例來說，一個機箱可以包含另一個機箱、
由一組電源供應器供電、連接纜線，並由風扇散熱。OpenBMC 目前
還沒有標準方式來表示這些類型的關係。

## 背景與參考資料

本設計延續先前的
[提案](https://gerrit.openbmc.org/c/openbmc/docs/+/41468)，但
指定對所有關係都使用 Associations（提案 II），而非
路徑階層（提案 I）。

本設計的主要驅動力來自 Redfish，特別是
[Chassis schema](https://redfish.dmtf.org/schemas/Chassis.v1_20_0.json)
中的 Links 區塊。

記錄新 Associations 的 phosphor-dbus-interfaces 變更已
[提出](https://gerrit.openbmc.org/c/openbmc/phosphor-dbus-interfaces/+/46806)，
但尚未合併，需等到對此設計達成共識為止。

本設計最初在
[Discord](https://discord.com/channels/775381525260664832/819741065531359263/964321666790477924)
上討論過，並在該處達成了初步共識。

## 需求

- 必須能表示來自機箱盤點物件的一對多關係，這些物件：
  - 連接纜線
  - 包含其他機箱，及/或被某個機箱包含
  - 包含儲存裝置
  - 由風扇散熱
  - 由電源供應器供電
  - 包含 CPU 等處理器
  - 包含 DIMM 等記憶體
- 必須支援預先定義、執行期偵測，或兩者兼有的關係
  - 執行期偵測可包含 I2C 匯流排掃描、USB 枚舉，及/或
    MCTP 探索

### 選定目標（初期實作之外）

- 非機箱的盤點物件也可能需要一對多關係
  - CPU 具有 CPU 核心及相關聯的 PCIe 插槽
  - CPU 核心具有執行緒

## 建議設計

本設計會影響 OpenBMC 架構的三層：phosphor-dbus-interfaces、
盤點管理器，以及盤點消費者（例如 bmcweb）。

### phosphor-dbus-interfaces

在 Inventory/Item 或更具體之子型別的介面定義中，我們為
上述每種關係類型新增一個 association 定義，並新增
對應的 association 定義以連結回該項目。

### 盤點管理器

#### phosphor-inventory-manager

phosphor-inventory-manager 已支援匯出自訂
Associations，因此此處無需變更。

#### entity-manager

對於 entity-manager，我們在 JSON 設定中為上游與下游
埠新增新的 `Exposes` 區塊。上游埠的類型以 `Port`
結尾（例如 `BackplanePort`、`PowerInputPort` 等）。下游埠的類型
為 `DownstreamPort`，並帶有一個 `ConnectsToType` 屬性，依類型
參照到上游埠。

entity-manager 中的新程式碼會比對這些屬性，並依據所涉及
盤點物件的類型在 D-Bus 上匯出 associations。兩個 Chassis 物件
之間會有 `containing` 與 `contained_by`，一個 Chassis 與一個 PowerSupply
之間則分別會有 `powering` 與 `powered_by`，以此類推。

JSON 設定範例：

superchassis.json

```json
{
  "Exposes": [
    {
      "Name": "MyPort",
      "Type": "BackplanePort"
    }
  ],
  "Name": "Superchassis",
  "Probe": "TRUE",
  "Type": "Chassis"
}
```

subchassis.json:

```json
{
  "Exposes": [
    {
      "ConnectsToType": "BackplanePort",
      "Name": "MyDownstreamPort",
      "Type": "DownstreamPort"
    }
  ],
  "Name": "Subchassis",
  "Probe": "TRUE",
  "Type": "Chassis"
}
```

#### 其他盤點管理器

如果系統上還有其他守護程式會匯出盤點物件，它們可以
選擇納入 phosphor-inventory-manager 與
entity-manager 所使用的相同 Associations。

### 盤點消費者

當 bmcweb 之類的守護程式想要判斷有哪些其他盤點項目
與特定項目存在關係時，它會向 object mapper 發出查詢，
後者會回傳所有相關聯項目及其彼此之間關係類型的清單。

`busctl` 呼叫範例：

```text
$ busctl get-property xyz.openbmc_project.ObjectMapper \
/xyz/openbmc_project/inventory/system/chassis/Superchassis/containing \
xyz.openbmc_project.Association endpoints

as 1 "/xyz/openbmc_project/inventory/system/chassis/Subchassis"

$ busctl get-property xyz.openbmc_project.ObjectMapper \
/xyz/openbmc_project/inventory/system/chassis/Subchassis/contained_by \
xyz.openbmc_project.Association endpoints

as 1 "/xyz/openbmc_project/inventory/system/chassis/Superchassis"
```

## 曾考量的替代方案

### 路徑階層

另一種替代提案是使用 D-Bus 路徑名稱來編碼
盤點項目之間的拓撲關係。舉例來說，位於另一個機箱下方的
機箱物件，即被該父機箱所包含。這適用於
符合樹狀結構的簡單關係，但當引入更複雜的關係時就會
失效，例如纜線、由多個物件共用的風扇與
電源供應器，或是焊接式的 CPU——它們是「屬於」機箱的一部分，而非被機箱
「包含」。引入各自擁有獨立拓撲的獨立托盤，
會進一步讓路徑階層的做法變得複雜。

一個可能的折衷方案是允許結合路徑階層與
associations 來傳達拓撲，但這會大幅增加
此資訊消費者的複雜度，因為它們必須同時支援
兩種做法，並設法解決互相衝突的資訊。

Associations 是唯一能符合所有使用情境的做法，因此我們應從
這個方法開始。如果未來確實需要路徑階層所帶來的額外複雜度，
屆時可以作為獨立設計另行加入。

為了改善人類檢視系統時的使用體驗，也可以有專門的
工具來查詢特定類型的 Associations，並呈現目前拓撲的
階層檢視。此外，
phosphor-inventory-manager 的設定可以依照其設定作者
認為合理的方式，來組織其 D-Bus 物件，但要讓盤點
消費者理解拓撲，仍舊必須具備
Association 屬性。

## 影響

這個新 API 會如前所述記錄在 phosphor-dbus-interfaces 中。
如果沒有為 entity-manager 或 phosphor-inventory-manager 的設定檔
新增任何拓撲資訊，則它們匯出的 D-Bus 介面將不會
改變。如果 bmcweb 之類的盤點資料消費者找不到新的
associations，則它們的輸出（例如 Redfish）也不會改變。

### 組織層面

此儲存庫是否需要新的儲存庫？不需要——所有變更都會放入
現有的儲存庫中。

## 測試

entity-manager 與 bmcweb 中的所有新程式碼都會使用現有的
框架與基礎設施進行單元測試。我們將在新 openbmc-test-automation 中
新增端對端測試，以確保 Redfish 輸出正確。

## 更新歷史

### 2023 年 2 月 14 日

- 將範例埠類型與名稱從 `Connector` 更新為 `Port`
- 根據 phosphor-dbus-interfaces 維護者的回饋，
  變更 association 名稱（chassisContains -> containing、chassisContainedBy ->
  contained_by 等）
