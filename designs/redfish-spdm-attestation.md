# Redfish SPDM 認證（Attestation）支援

作者：Zhichuang Sun

其他貢獻者：Jerome Glisse、Ed Tanous、Manojkiran Eda

建立日期：2023 年 6 月 27 日 最後更新：2023 年 10 月 30 日

## 問題描述

Redfish 新增了
[ComponentIntegrity](https://redfish.dmtf.org/redfish/schema_index)
的 schema，讓使用者可以使用 [SPDM](https://www.dmtf.org/standards/spdm) 或
[TPM](https://trustedcomputinggroup.org/resource/trusted-platform-module-tpm-summary/)
來驗證裝置身分、硬體設定與韌體完整性。在 BMCWeb 中加入 SPDM
認證支援會很有幫助，這能為資料中心的裝置安全認證提供統一的介面，並為 SPDM
D-Bus Daemon 提供一套通用的實作。

本設計聚焦於 SPDM。

## 背景與參考資料

SPDM（Security Protocols and Data Models）是由
[DMTF](https://www.dmtf.org/sites/default/files/standards/documents/DSP0274_1.3.0.pdf)
發布的規範。它是為了裝置的安全認證而設計。GitHub 上的
[libspdm](https://github.com/DMTF/libspdm)
儲存庫提供了 SPDM 協定的開源實作。Redfish Schema
[ComponentIntegrity](https://redfish.dmtf.org/schemas/v1/ComponentIntegrity.v1_2_1.json)
新增了透過 Redfish API 進行 SPDM 裝置認證的支援。

## 需求

此功能目標在於透過 Redfish API 支援 SPDM 認證，並為系統管理人員與維運人員提供一種簡便的方式， remotely（遠端）驗證裝置的身分與完整性。

本設計包含：

- 為 Redfish 資源 `ComponentIntegrity` 與 `TrustedComponent` 新增的 D-Bus
  介面。
- 為支援上述 Redfish 資源而進行的 BMCWeb 變更。
- SPDM Attestation D-Bus Daemon 的設計，示範如何透過 D-Bus 取得認證結果。

## 建議設計

### 認證相關的 D-Bus 介面

在 D-Bus 上，我們需要從認證 daemon 取得三類資訊：

1. 基本資訊，例如認證協定、啟用狀態、更新時間戳記等。
2. 身分資訊，例如裝置身分憑證。
3. 量測（Measurements）資訊，例如元件韌體、硬體設定等的量測值。

到目前為止，phosphor-dbus-interfaces 缺乏為認證用途定義的介面。因此我們提出三個新介面：

- `Attestation.ComponentIntegrity`
- `Attestation.IdentityAuthentication`
- `Attestation.MeasurementSet`

`Attestation.ComponentIntegrity`
提供基本的元件完整性資訊，包括用以量測完整性的協定、最後更新時間、認證啟用狀態等。這個介面也提出了幾項關聯（associations），包括指向該元件完整性物件所回報之受信任元件（trusted component）的連結，以及指向該元件完整性物件所保護之系統（systems）的連結。請注意，本文件中的「受信任元件（trusted component）」指的是一台受信任的裝置，例如 TPM、整合式的信任根（Root of Trust，ROT）。

`Attestation.IdentityAuthentication`
提供身分驗證資訊。我們提出了兩項關聯，將它連結到要求端（requester）與回應端（responder）的憑證。

`Attestation.MeasurementSet`
定義了取得 SPDM 量測值的 D-Bus 方法。未來它可以延伸支援使用其他協定取得的量測值。

這裡是建議的 Phosphor D-Bus Interfaces：
[component-integrity](https://gerrit.openbmc.org/c/openbmc/phosphor-dbus-interfaces/+/64354)。

### TrustedComponent 相關的 D-Bus 介面

我們也提議為 `TrustedComponent` 新增一個 Inventory 介面，也就是
`Inventory.Item.TrustedComponent`。`TrustedComponent`
代表一個受信任的硬體元件，通常稱為信任根（Root of Trust，ROT）。它可以是一顆外接的安全晶片，例如 TPM，或是整合進裝置中的硬體
IP。它能安全地量測裝置的完整性資訊。

`Inventory.Item.TrustedComponent` 介面定義了以下屬性

- `AttachmentType`，提供這個受信任元件是整合在裝置內部、還是與裝置分離（discrete）的資訊。

一個 `TrustedComponent`
通常會與它所保護的其他硬體元件產生關聯。它也應該與這個 `TrustedComponent`
所回報的元件完整性物件（component integrity object）產生關聯。

這裡是為 `TrustedComponent` 建議的 Phosphor D-Bus Interfaces：
[trusted-component](https://gerrit.openbmc.org/c/openbmc/phosphor-dbus-interfaces/+/64355)。

### SPDM Attestation D-Bus Daemon

[`bmcweb` 已有實驗性的 SPDM
支援](https://github.com/openbmc/bmcweb/compare/master...sunzc:bmcweb:spdm)，
它在 BMCWeb 中為 `ComponentIntegrity` 與 `TrustedComponent`
新增了路由（routes）來支援此功能。但 BMCWeb
是從 D-Bus 收集資訊，實際的工作由 SPDM Attestation D-Bus Daemon 負責。

SPDM 協定需要綁定到一個傳輸層協定，由該協定在 BMC 與裝置之間傳送 SPDM
訊息。傳輸層協定可以是 MCTP、PCIe-DOE，甚至 TCP socket。若是 MCTP，下層的實體層可以是 PCI-VDM、SMBus/I2C
等。請注意，[`libspdm` 已提供傳輸層協定綁定](https://github.com/DMTF/libspdm/blob/main/include/internal/libspdm_common_lib.h#L445-L446)，
並具備訊息編碼／解碼的支援。裝置的傳送／接收函式則留給 SPDM daemon 去實作。若傳輸層使用標準的
MCTP、PCIe-DOE 或 TCP，設定傳輸層連線可能會很簡單。在本設計中，我們只考慮 SPDM over 標準
MCTP、TCP 與 PCIe-DOE 連線。

對於 SPDM-over-MCTP，SPDM daemon 可以向 mctpd 查詢 MCTP
端點（endpoint）的資訊，包括端點 id（eid）與上層回應端（upper layer responder），並且只為那些以上層回應端為
SPDM 的端點建立連線。

對於 SPDM-over-TCP，SPDM daemon 可以向
[entity-manager][entity-source] 查詢遠端 TCP 端點的資訊，包括來自
`SpdmTcpResponder` 介面的 `hostname / ip-address` 與 `port`
詳細資料，並依照 [SPDM over TCP 規範][spdm-tcp-spec]
所述的 reach out 模型或 reach down 模型來建立連線。

[entity-source]: https://github.com/openbmc/entity-manager
[spdm-tcp-spec]:
  https://www.dmtf.org/sites/default/files/standards/documents/DSP0287_1.0.0.pdf

對於 SPDM-over-PCIe-DOE，SPDM daemon 需要 PCIe 裝置的 BDF 來處理 DOE
mailbox 的探索（discovery）。由於並非所有支援 DOE 的 PCIe 裝置都支援
SPDM，我們無法查詢一個具備 DOE 能力的裝置是否支援 SPDM。因此我們需要某種方式將裝置資訊傳給
daemon。然而，PCIe 裝置的 BDF（Bus:Device:Function）資訊是在系統開機期間動態指派的。同一個裝置在不同的機器上可能會被指派不同的
BDF。daemon 所需要的應該是 PCIe 裝置 ID，它以 `VendorId:DeviceId`
來識別。為了設定的便利性，我們應該把 PCIe 裝置 ID 傳給 daemon，讓 daemon
可以列舉所有 PCIe 裝置，並依裝置 ID 找到符合的裝置。將 PCIe 裝置 ID 資訊傳給 daemon 的方式有很多種，例如設定檔、命令列參數。

對於 PCIe DOE 裝置，SPDM daemon 可以在 sysfs 下列舉所有 PCI
裝置，找出所有 `VendorId:DeviceId` 相符的 BDF。SPDM daemon
也需要查詢 `InventoryManager` 以取得所有 PCI 裝置的 inventory 路徑。透過查詢
`InventoryManager`
管理的物件並檢查物件介面 `xyz.openbmc_project.Inventory.Item.PCIeDevice`，它有屬性
`FunctionXVendorId` 與 `FunctionXDeviceId`（X 代表 0 到 7 的數字），我們就能找出所有
`VendorId:DeviceId`
相符的 PCI 裝置 inventory 路徑。到目前為止，尚沒有一種通用的方式可以將某個裝置的 BDF 對應到它的 inventory 路徑。這部分的關聯要由供應商自行處理。

關於 MCTP 裝置偵測，社群一直在 dbus-sensors 中開發 `mctpreactor`
daemon 來處理設定。相關的[實作](https://gerrit.openbmc.org/c/openbmc/dbus-sensors/+/69111)；以及相關的
[PR 討論](https://github.com/CodeConstruct/mctp/pull/17)。SPDM daemon
會監聽 `mctpd` 提供 `xyz.openbmc_project.MCTP.Endpoint`
介面的 `InterfacesAdded` 訊號，該介面在 `SupportedMessageTypes`
成員中公開端點所支援的訊息類型。SPDM daemon
會與具備 SPDM 能力的端點建立連線，以取得憑證與量測值。對於在 SPDM daemon
啟動之前就已發送的訊號，SPDM daemon 應在自己啟動後向 `mctpd`
查詢任何已偵測到的端點。

關於網路型裝置偵測，SPDM daemon
會監聽由`xyz.openbmc_project.Configuration.SpdmTcpResponder`
介面提供的 `InterfacesAdded` 訊號，該介面公開遠端實體的 `Hostname` 與
`Port`。對於在 SPDM daemon 啟動之前就已發送的訊號，SPDM daemon
應在自己啟動後向 `entity-manager` 查詢任何已偵測到的端點。

下圖是一個高階示意圖，展示各元件之間的關係。

```ascii
    +------+            +---------+
    |Client|            |Inventory|
    +--+---+            |Manager  |
       |                +---^-----+           +-------+
       |                    |                 |PCIe   |
     +--v---+            +---+---+------------>|Device |
     |BMCWeb+----------->|SPDM   |             +-------+
     +------+            |Daemon |
                         +---+---+------------>+-------+
                             |                 |MCTP   |
                         +---v---+             |Device |
                         |mctpd  |             +-------+
                         +-------+
```

一個參考的 D-Bus Daemon 工作流程如下：

0. （探索階段，Probing phase）Entity Manager 會解析受信任元件的設定檔；mctpd
   完成 MCTP 端點的探索。這些是 SPDM daemon 的前置條件。
1. 檢查傳輸層協定。若是 MCTP，它向 mctpd 查詢以蒐集所有支援 SPDM 的 eid；若是
   PCIe-DOE，它依 PCIe 裝置 ID 執行 DOE mailbox 探索；若是
   TCP，它向 entity-manager 查詢以蒐集遠端 IP 位址與 Port。
2. 對每個端點（可能是 MCTP、PCIe-DOE 或 TCP），SPDM daemon 向 Entity
   Manager 查詢符合的受信任元件設定。接著它建立並初始化對應的 D-Bus
   物件，用於 `TrustedComponent` 與 `ComponentIntegrity`，並帶入裝置專屬的資訊。
3. 在上述物件之間建立關聯，並與其他物件建立關聯，例如受保護的元件、作用中的軟體映像（active software images）；
4. 在 BMC 與每個具備 SPDM 能力的裝置之間建立連線；
5. 在連線之上初始化 SPDM context。
6. 交換 SPDM 訊息以取得裝置憑證。
7. 建立裝置憑證物件，並為受信任元件物件與元件完整性物件建立憑證關聯。
8. 在 D-Bus 上等待，並處理任何執行期的 `SPDMGetSignedMeasurements` 請求。

### 裝置憑證

在 OpenBMC 中，有一個
[certificate manager](https://github.com/openbmc/phosphor-certificate-manager)，
它讓使用者可以安裝或取代伺服器／用戶端憑證。然而，現有的憑證管理器是為管理 HTTPS/LDAP 服務的伺服器／用戶端憑證而設計的，並不適用於裝置憑證。現有的憑證管理器有幾項限制：

- 每個管理器只能管理一個憑證。
- 每個管理器都假設同時能取得私密金鑰（private key）與公開金鑰（public key）（例如為了完成 CSR 請求）。

裝置憑證有不同的需求：

- 裝置憑證管理器會為一組裝置管理多個憑證，例如四顆 GPU 就會有四張憑證。
- 裝置憑證不假設能取得私密金鑰。它只用於身分驗證。它不提供 CSR 簽署服務。私密金鑰不應離開裝置的信任根（Root of Trust，RoT）元件。

對於裝置憑證，我們只需要建立／取代憑證物件，不需要一個「管理」裝置憑證的全域憑證管理器。SPDM D-Bus daemon 可以直接與裝置溝通、向它們取得憑證，並為這些憑證建立 D-Bus 物件。

在 Redfish 中，裝置憑證位於 Chassis 之下，可透過 `/redfish/v1/Chassis/{ChassisId}/Certificates/` 存取。現有的憑證管理器依模式 `"/xyz/openbmc_project/certs/{type}/{endpoint}"` 來建構憑證路徑。為了符合這個模式，我們提議把裝置憑證放在 `/xyz/openbmc_project/certs/chassis/{ChassisId}/{certsId}`。如此一來，對於某個機箱（chassis）上的所有裝置憑證，我們就能在 D-Bus 上依其 chassisId 找到這些憑證。

### BMCWeb 支援

在 BMCWeb 中，我們需要為 `ComponentIntegrity`、`TrustedComponent` 與 `Certificates` 新增路由處理常式（routes handler）。根據 Redfish 規範，對應的 URI 定義如下：

- `/redfish/v1/ComponentIntegrity/`
- `/redfish/v1/Chassis/{ChassisId}/TrustedComponents/`

TrustedComponent 代表一個 ROT 或 TPM，與它關聯的憑證應以附屬資源（subordinate resources）的形式呈現在 TrustedComponent 之下。關於憑證正確的 URI 結構，請參考 [Redfish Data Model Specification][1]。在規範中尋找 CertificatesCollection 那一列。

- `/redfish/v1/Chassis/{ChassisId}/TrustedComponents/{TrustedComponentId}/`
  `Certificates`
- `/redfish/v1/Chassis/{ChassisId}/TrustedComponents/{TrustedComponentId}/`
  `Certificates/{CertificateId}`

在 D-Bus Daemon 端，我們建議 D-Bus 物件以下列方式組織：

- `/xyz/openbmc_project/component_integrity/{component_integrity_id}`
- `/xyz/openbmc_project/trusted_components/{trusted_component_id}`
- `/xyz/openbmc_project/certs/devices/{chassis_id}/{cert_id}`

在 BMCWeb 中，我們可以透過從受信任元件查詢其關聯的 Chassis，重建出以下的 Redfish URI：

- `/redfish/v1/Chassis/{ChassisId}/TrustedComponents/{TrustedComponentId}`

## 曾考慮的替代方案

管理裝置憑證的另一種方式是修改現有的
[phosphor-certificate-manager](https://github.com/openbmc/phosphor-certificate-manager)。

裝置憑證管理有兩個步驟：

- 步驟 1：透過與裝置交換 SPDM 訊息來取得裝置憑證。
- 步驟 2：建立或更新一個 D-Bus 憑證物件。

步驟 1 只能由 SPDM daemon 處理。步驟 2 足夠簡單，也可以由 D-Bus daemon 處理。僅為了這個單一目的去修改現有的 phosphor-certificate-manager 會是過度工程（over-kill）。

## 影響

此變更要：

- 建立一個 SPDM daemon，能為具備 SPDM 能力的裝置透過 PCIe DOE、MCTP 或 TCP 執行 SPDM 認證。
- 在 phosphor-dbus-interfaces 中新增 `ComponentIntegrity` 與 `TrustedComponent` 相關的 D-Bus 介面。
- 延伸 BMCWeb 中現有的憑證服務。
- 在 BMCWeb 中新增 SPDM 支援及新路由。

### 組織層面

此功能需要為 SPDM daemon 建立一個新的儲存庫。此外，預期會修改以下儲存庫以執行本設計：

- <https://github.com/openbmc/bmcweb>
- <https://github.com/openbmc/phosphor-dbus-interfaces>

## 測試

### 單元測試

對於 BMCWeb 的變更，可以使用 Redfish Service Validator 進行單元測試。

對於 SPDM Attestation D-Bus Daemon，單元測試應涵蓋以下情境：

- 與裝置建立傳輸層連線。
- 應實作單元測試以確保基礎設施函式可靠運作，無論底層的傳輸機制為何。
- SPDM 連線的建立，包括取得能力（get capabilities）、協商演算法（negotiate algorithms）。
- 從裝置取得裝置憑證並建立 D-Bus 物件。
- `SPDMGetSignedMeasurements` 方法測試。
- 列舉受信任元件的 D-Bus 物件並檢查屬性與關聯。
- 列舉元件完整性的 D-Bus 物件並檢查屬性與關聯。

### 整合測試

BMCWeb/D-Bus Daemon 整合測試應涵蓋以下幾類請求：

- 取得一組 `ComponentIntegrity` 資源（collection）。
- 取得一組 `TrustedComponent` 資源（collection）。
- 取得某個 `ComponentIntegrity` 資源的屬性。
- 取得某個 `TrustedComponent` 資源的屬性。
- 沿著資源連結（resources link）取得裝置憑證，並驗證它們是否正確地以附屬資源的形式呈現在 TrustedComponent 之下。
- 取得 `Certificate` 資源的屬性，並驗證回應與憑證鏈（certificate chain）中的末端憑證（leaf certificate）相符。
- 對 `ComponentIntegrity` 資源呼叫 Action（動作）以取得量測值。

[1]:
  https://www.dmtf.org/sites/default/files/standards/documents/DSP0268_2024.4.html#resource-collection-uris-in-redfish-v16-and-later
