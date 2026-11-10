# Entity-Manager HW ID：透過 Device-Tree 屬性進行 VPD 探索

Author: Christopher Sides (citysides)

其他貢獻者：

建立日期：2023 年 9 月 14 日

## 問題說明

BMC 需要一套流程，用於辨識並處理 HPE GXP 基底板（baseboard），這類硬體透過
非 I2C 通道提供 HW ID 資料，且採用的是專有格式，並不在多數平台所依賴的
Entity-Manager『fru-device』daemon 涵蓋範圍之內。

本提案描述一種處理方式：針對 HW ID 資料是從 device tree 檔案路徑收集而來
的硬體，供 Entity-Manager 使用。

## 背景與參考資料

典型平台會以 FRU 儲存 blob 的形式提供基底板的 HW ID 資料——通常稱為
『vital product data』（VPD）——存放於實體 EEPROM 中。

[如 Entity-Manager 文件所述](https://github.com/openbmc/entity-manager/blob/master/docs/my_first_sensors.md)，
這個 blob 會透過 I2C 被探索並讀取，接著解碼，然後由 Entity-Manager 的
fru-device daemon 將資料複製到 D-Bus 上，成為 `xyz.openbmc_project.FruDevice`
介面的屬性。目前的 FRU-device daemon 能夠解碼 IPMI-FRU 儲存格式的 blob，
以及 Tyan 的資料格式。

此外，另一個用於處理 OpenPower 格式 VPD 的獨立 daemon 託管在
『openpower-vpd-parser』子專案中。某些 IBM 支援硬體的辨識即依賴此 daemon。

HPE 硬體使用不同於上述的通道與資料格式，依賴對 device-tree 的修改，將屬性
以每個屬性對應一個檔案 handles 的方式輸出。

一旦 VPD 在 D-Bus 上可用，Entity-Manager 設定中的『probe』敘述即可引用它，
這些敘述被用作硬體／設定偵測的觸發條件。

HPE 平台為基底板 VPD 提供以下屬性：

- server_id（平台型號 id）
- pca_part_number（基底板料號 part no）
- pca_serial_number（基底板序號 serial no）
- part_number（平台產品料號，於出廠組裝時設定）
- serial_number（系統序號，於出廠組裝時指派）
- mac0（若存在：專用 BMC 網路介面的 MAC address）
- mac1（若存在：次要網路介面的 MAC address，可能與 host 共用也可能不共用）

Gen 10 及更早世代的 HPE 平台透過標準可由 I2C 存取的 EEPROM 提供 VPD
（但仍是專有資料格式）。在較新的 HPE 系統上，這條途徑已不再可用。

對於以 GXP 為基礎的 Gen 11 HPE 平台，由 HPE 控制的專有 bootblock 韌體與
安全元件（secure element）通訊以提供必要的 VPD。此流程使用一個實作受到嚴格
授權控管、無法公開釋出的 driver。開源部分的啟動鏈必須配合 bootblock 為從
安全元件提供 VPD 所保留的記憶體 application binary interface（ABI）。

在啟動流程早期，會先驗證 HPE 專有的 bootblock，接著區塊中的 HPE 專屬
bootloader 透過第三方 driver 從安全元件抓取 VPD blob。資料 blob 接著被複製
到 RAM 中預先設定的位置，再由 u-Boot 中的 HPE 專屬程式碼取出並解析
（該 u-Boot 為開源，可於 <https://github.com/HewlettPackard/gxp-uboot>
取得）。未來我們將盡可能把所需的『HPE-critical』u-boot 程式碼 upstream，
使得啟動 HPE 硬體時不再需要 HPE 專屬的 u-boot fork。

此外，u-Boot 中的程式碼會收集 BMC 網路介面的 MAC address，並從一個內嵌的
CPLD 記憶體裝置取得『server_id』值。此值對每種 HPE 平台型號都是唯一的，
將用於平台層級的識別。相對地，前述 blob 中的資料可用於辨識更細項的資訊，
例如特定平台的具體板子修訂版本。

上述 HPE 流程依賴共享記憶體而非 u-boot env 變數，因為 HPE bootloader 並不
知道接下來會執行 u-boot，也不知道 u-boot 的變數儲存區在哪裡——如果它存在的
話——因為客戶可能決定使用 u-boot 以外的東西，或將他們的環境變數存放在不同
的位置或格式。

之後，u-Boot 透過修改 flattened device tree 將相關資料提供給 Linux，設定
['/model' 與 '/serial-number,' 兩者的值，它們在 device tree 根節點中都有
知名路徑][dt-spec-chap3-root-node]，以及 ['local-mac-address,' 一個由網路
binding 提供的屬性][dt-spec-chap4-device-bindings]。

[dt-spec-chap3-root-node]:
  https://github.com/devicetree-org/devicetree-specification/blob/main/source/chapter3-devicenodes.rst#root-node
[dt-spec-chap4-device-bindings]:
  https://github.com/devicetree-org/devicetree-specification/blob/main/source/chapter4-device-bindings.rst#local-mac-address-property

接著，必須將 GXP 產品資料發佈到 D-Bus，否則 Entity-Manager 在用於偵測支援
硬體設定的 probe 評估過程中將無從引用它。只要讓 'device-tree/model' 的內容
在 D-Bus 上可用，就足以讓 Entity-Manager 對 HPE 基底板硬體進行基本辨識。

未來，HPE 在此流程中對 u-Boot 的幕後依賴可能會減少甚至完全移除，但這些實作
細節不應影響一個 service 如何從 u-Boot 傳給 kernel 的 device-tree 收集資料。
修改 flat device-tree 的另一種替代做法，是在映像檔建置時就在平台的
device-tree 來源中設定 'model' 之類的值。

本文件討論在 Entity-Manager 中利用一個『device-tree -> D-Bus』daemon，讓
BMC 能偵測並處理 HPE Gen 11 硬體（也可能適用於其他硬體）。

## 需求

- 必須解析 vital product data 並發佈到 D-Bus

- 路徑 `xyz/openbmc_project/MachineContext` 處介面的 D-Bus 屬性，可適當地
  使用取回到的 VPD 值填入，以提供一個 D-Bus 路徑來存放常見的特定型號識別
  細節。

- Entity-Manager 設定的『probe』敘述應引用 vital product data 屬性，以便
  據以決定設定的啟用。

## 建議設計

高階流程：

- 開機時，先驗證專有的 HPE bootblock，接著區塊中包含的 HPE GXP-bootloader
  程式碼（專有）從安全元件抓取資料。特定的硬體識別資訊被搬到 RAM 中預先
  設定好的記憶體位置以供撈取。

- u-Boot 程式碼（透過一個
  [開源但 HPE 專屬的流程](https://github.com/HewlettPackard/gxp-uboot)）
  收集 MAC address，並從 RAM 位置以及一個 CPLD 記憶體裝置搬移產品資料，
  並在 Linux 下揭露於 /proc/device-tree/

- DeviceTree-VPD-Parser 中的一個 daemon 從已知的 device tree 路徑取回產品
  資料，然後解析並發佈到 D-Bus

- 引用 D-Bus 上 `xyz.openbmc_project.Inventory.Decorator.Asset`（.Model 與
  .SerialNumber 屬性）介面屬性的 Entity-Manager probe，可據 GXP 硬體資料
  作為鍵值並相應地作出反應。

## 曾考慮的替代方案

### 延長 Entity-Manager probe 以處理從任意 device-tree 節點讀取資料

#### 優點

1. 不需要新的 daemon

#### 缺點

1. 可能鼓勵不想要的 device-tree 利用行為

### 使用 device-tree 中的一個 reserved-memory 節點來傳輸 'serial-number' 與 'model' ID 資料

#### 優點

1. Reserved-memory 常被用於從韌體傳遞資料

#### 缺點

1. 不像 device-tree 那樣對 /model 或 /serial-number 等常見屬性有知名
   handles；更適合保存任意資料而非『well-known』屬性
2. HPE 想依賴作為平台識別的關鍵識別項（/model）與計畫透過 reserved-memory
   輸出的其餘資料來自不同的幕後來源。用 reserved-memory 來放 'model' 而非
   知名的 device-tree /model handle，會增加額外複雜度。
3. 由於 /model 在 device-tree 文件中有完整描述，非 HPE 平台也有可能輕鬆
   加以利用。將 'model' 當作 reserved-memory 中的一個任意屬性來依賴，在
   HPE 專屬應用之外的實用潛力較低。

### 所有 VPD 都以 devicetree 屬性揭露於 /chosen/ 之下

#### 優點

1. 是一個行之已久的 device-tree 節點
2. 未被其他 BMC 平台使用，因此易於根據 'chosen' 路徑是否存在來啟用／停用
   service
3. 值可由 u-Boot 透過修改它可取得的 flattened device tree 來設定（適用於
   任意 device-tree 路徑）

#### 缺點

1. 不是傳遞這種綁定硬體資料的『慣用』位置。其他單位幾乎不可能用它來搬移
   產品資料，因此這樣的 daemon 實質上會是 HPE 專屬程式碼。
2. 在此呈現硬體專屬資料似乎違反 'chosen' 節點的原意

### 將 vital product data 發佈到 u-Boot 環境變數，再轉移到 D-Bus

#### 優點

1. 會提升程式碼重用與測試

#### 缺點

1. 儲存 u-boot env 變數的記憶體可能有寫入次數有限的潛在問題。
2. 從 HPE 安全元件取回 VPD 的 HPE 元件，並不知道也無法改變 u-boot 環境變數

### 使用 HPE 專屬 daemon，而非一般的『[channel]-> D-Bus』daemon

#### 優點

1. 程式碼更簡單（稍微？）
2. 只會對 HPE 平台觸發
3. 更好的抽象化

#### 缺點

1. 相較於一般、非 HPE 專屬的解法，省不了多少工
2. 較難為未來可能的情況利用程式碼

### Service 託管於 Entity-Manager 中 [經 TOF 重新檢視後接受]

#### 優點

1. 將與既有的 Fru_Device daemon 平行運作

#### 缺點

1. 社群不願意在 Entity-Manager repo 中託管 VPD -> DBus daemon

後續：一場 Technical Oversight Forum 討論已達成共識，將這個 device-tree VPD
daemon 託管在 Entity-Manager 中、與既有 fru-device daemon 並列會較佳，而不是
為一個如此簡單的 daemon 建立專屬 repo，或硬塞進另一個更不相關的 repo。

<https://github.com/openbmc/technical-oversight-forum/issues/38>

### Service 託管於 Phosphor-u-boot-env-mgr

#### 優點

1. 該 repo 已與 u-boot 操作相關聯

#### 缺點

1. 目前 repo 的維護者認為此 service 不適合這個 repo，因為該 repo 的宗旨
   明確是處理 u-boot env 變數的操作，而此 service 與之無關。

   <https://gerrit.openbmc.org/c/openbmc/phosphor-u-boot-env-mgr/+/71512>

### 補述：為待定的 device-tree -> D-Bus daemon 所遭拒的 D-Bus 介面

#### 既有介面（遭拒）

1. 提案：xyz.openbmc_project.Decorator.\*

   拒絕理由：「沒有反映實體板子的任何資訊」-Ed Tanous

   後續：Phosphor-dbus-interfaces 的維護者（Patrick Williams）拒絕託管一個
   帶有 'Model' 與 'SerialNumber' 等屬性、而這些屬性已存在於其他介面中的
   介面，並明確建議依賴 xyz.openbmc_project.Inventory.Decorator.Asset
   （用於 .model 與 .serialnumber 屬性），以及
   xyz.openbmc_project.Inventory.Item.NetworkInterface（用於 .MACAddress
   屬性）。

   討論 @
   <https://gerrit.openbmc.org/c/openbmc/phosphor-dbus-interfaces/+/73260/comment/e0226ae4_d7e514dd/>

   後續 2：沿用 Inventory.Decorator.Asset，但移除 MAC address 屬性以降低
   不必要的複雜度。

2. 提案：xyz.openbmc_project.FRUDevice

   拒絕理由：資料並非來自 FRU 裝置

#### 提案介面（遭拒）

1. 提案：xyz.openbmc_project.DeviceTree
2. 提案：xyz.openbmc_project.uBootHWID
3. 提案：xyz.openbmc_project.GXPFRU

   拒絕理由：上述介面未能反映底層硬體

4. 提案：xyz.openbmc_project.BMCPrebootContext

   拒絕理由：可說並未代表一個硬體概念。

5. 提案：xyz.openbmc_project.UbootDeviceModel

   拒絕理由：目前已有持續進行的工作要啟用 u-boot 之後的動態 device-tree
   修改。此 daemon 讀取的節點不太可能受影響，但對此並無保證。

6. 提案：xyz.openbmc_project.Platform

   拒絕理由：某些屬性如 serial-number 或 MAC 並不適用於整個平台

7. 提案：xyz.openbmc_project.Machine

   拒絕理由：看似有希望，但名稱對其用途而言顯得含糊。已提出
   xyz.openbmc_project.MachineContext 作為替代。

8. 提案：xyz.openbmc_project.MachineContext

   拒絕理由：「我們已經定義了這一切[.Model、.SerialNumber 等屬性]。我不會
   接受一個把『HPe [今天] 認為重要的一堆隨機屬性』塞進新介面的介面」
   - Patrick Williams，Phosphor-dbus-interfaces 維護者。

   反提案：「重用既有的 dbus 介面，並把它們放在一個明確定義的位置？
   Inventory.Decorator.[Asset] = Model
   Inventory.Decorator.Asset = SerialNumber Inventory.Item.NetworkInterace =
   MACAddress..」 - Patrick Williams

   討論 @
   <https://gerrit.openbmc.org/c/openbmc/phosphor-dbus-interfaces/+/73260/comment/e0226ae4_d7e514dd/>

   後續：最終定案介面為 xyz.openbmc_project.Inventory.Decorator.Asset，
   位於路徑 xyz/openbmc_project/MachineContext

## 影響

上述流程僅打算在 service 初次啟動時，掃描選定的路徑（如果它們存在的話）
一次。

如果所涉路徑（device-tree/model 或 device-tree/serial-number）不存在，就
不會再做任何工作，因此預期對不依賴此 service 進行 Entity-Manager HW／設定
偵測的平台，不會有明顯的效能或功能影響。

## 組織面

### 這個設計需要新的 repository 嗎？

不需要。經過一些討論，Technical Oversight Forum 達成共識，將 device-tree
VPD parser daemon 託管在 Entity-Manager repo 中、與既有 fru-device daemon
並列。

### 預期會修改哪些 repository 以執行這個設計？

Technical Oversight Forums 已達成共識，這個 device-tree VPD daemon 將託管在
Entity-Manager 中，與既有 FRU-device VPD daemon 並列。

Entity-Manager 也將為 HPE 硬體新增設定檔，透過 probe 敘述使用新的介面屬性。

## 測試

任何在 Device-Tree 中有 'model' 或 'serial-number' 節點被填入值的平台（若
需要，可在建置映像檔前將值硬編碼進 DTS 檔來達成），都可透過確認
device-tree/model 及／或 serial-number 節點的內容，是否符合 D-Bus 路徑
'xyz/openbmc_project/MachineContext' 處 D-Bus 介面的 'model' 與
'serial-number' 屬性內容來檢查。

實務上，可在 HPE 系統上驗證這個新 D-Bus 路徑的整個 service 堆疊（包含
Entity-Manager probe 驗證）：方法是將某個 Entity-Manager 設定的 probe 設定為
在與一個已知 HPE 硬體識別屬性相符時觸發，然後確認
`busctl tree  xyz.openbmc_project.EntityManager` 顯示的物件清單與修改後 EM
設定檔中所述相符。

目前 HPE 硬體需要 fork（公開可取）的 OBMC 元件才能完整啟動，因此針對
'mainstream' OBMC 映像檔對 HPE 硬體進行自動化測試尚非可行選項。啟用這類測試
是一項較長期的目標。
