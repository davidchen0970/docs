# 動態 Redfish 授權

作者：Nan Zhou <nanzhoumails@gmail.com>

建立日期：2022/08/08

## 問題描述

Redfish 授權子系統負責控管哪些已通過驗證的使用者可以存取資源，以及
使用者擁有哪種類型的存取權限。

DMTF 已經定義了授權模型與 schema。本設計旨在強化 BMCWeb Redfish
介面目前的實作，讓 OpenBMC 系統能在不重新啟動 Redfish 服務的情況下，
於執行期變更授權行為。

## 背景與參考資料

### Redfish 授權模型

Redfish 授權模型由權限模型（privilege model）與操作對權限對應
（operation-to-privilege mapping）兩部分組成。

在權限模型中，有一組固定的標準 Redfish 角色，每個角色都被指派了一組
固定的標準權限（例如 `Login`、`ConfigureManager` 等）。服務可以自訂
OEM 角色（唯讀）。服務甚至可以允許用戶端自訂的角色透過對
`RoleCollection` 進行 REST 操作來建立、修改與刪除。

操作對權限對應是針對每種資源類型所定義的，並套用於服務針對該適用資源
類型所實作的每一個資源。它用來判斷已驗證 Redfish 角色的身分權限是否
足以完成請求中的操作。Redfish Forum 在其官方 registry 集合中提供了
Privilege Registry 定義，作為基礎的操作對權限對應。它同時允許透過
對應覆寫（Property override、Subordinate override 以及 Resource URI
override）來自訂對應。

舉例來說，一個具有 `Login`、`ConfigureComponents` 與 `ConfigureSelf`
權限的 `Operator` 同儕（peer），有權對 `ChassisCollection` 發送 `GET`
請求，因為對此資源的 `GET` 操作只需要 `Login` 權限。另一方面，同一個
同儕若對 `CertificateService` 發送 POST 請求則會被拒絕存取，因為對
憑證的 POST 操作需要同儕所缺乏的 `ConfigureManager` 權限。

**注意**，在 Redfish 規格中，OEM 角色可以透過對 `AccountService` 中的
`RoleCollection` 發送 POST 來新增；`PrivilegesUsed`、`OEMPrivilegesUsed`
以及 `Mappings` 的各項屬性皆為唯讀。

參考資料：

1. <https://redfish.dmtf.org/schemas/DSP0266_1.15.1.html#privilege-model>
2. <https://redfish.dmtf.org/schemas/DSP0266_1.15.1.html#redfish-service-operation-to-privilege-mapping>
3. <https://redfish.dmtf.org/schemas/DSP0266_1.15.1.html#roles>
4. <https://redfish.dmtf.org/registries/v1/Redfish_1.3.0_PrivilegeRegistry.json>

### Phosphor-user-manager

Phosphor-user-manager 是一個負責使用者管理的 OpenBMC 常駐程式（daemon）。

它暴露 DBus API 以動態新增使用者、管理使用者的屬性（例如群組、權限、
狀態與帳戶原則）。它有一份寫死的（hardcoded）使用者群組清單（SSH、
IPMI、Redfish、Web）以及一份寫死的權限清單（"priv-admin"、
"priv-operator"、"priv-user"、"priv-noaccess"）。這些權限是以 Linux
次要群組（secondary groups）的形式實作。

它也整合了 LDAP（支援 ActiveDirectory 或 OpenLDAP）以進行遠端使用者
管理，其中 BMC 作為 LDAP 用戶端，並使用 nslcd 自動將 Linux 系統呼叫
轉換為 LDAP 查詢。

參考資料：

1. <https://github.com/openbmc/docs/blob/master/architecture/user-management.md>
2. <https://github.com/openbmc/phosphor-user-manager>
3. <https://github.com/openbmc/phosphor-dbus-interfaces/tree/master/yaml/xyz/openbmc_project/User>
4. <https://linux.die.net/man/8/nslcd>
5. <https://linux.die.net/man/8/nscd>

### BMCWeb

BMCWeb 是一個實作 Redfish 服務的 OpenBMC 常駐程式（除了 Redfish
之外，它也實作了其他管理介面）。

BMCWeb 支援多種「驗證（authentication）」選項，但在底層，為了驗證
使用者確實是其宣稱的身分，主要有兩種驗證方法：

1. 基於 PAM：使用 Linux-PAM 進行使用者名稱/密碼式的驗證
2. 基於 TLS：使用公開金鑰基礎架構（Public Key infrastructure）來驗證
   同儕憑證的簽章；接著使用身分識別（在 X509 憑證中，這些是 Common
   Name 或 Subject Alternative Name）作為使用者名稱。

取得同儕的使用者名稱後，BMCWeb 會對 phosphor-user-manager 發送 DBus
查詢以查詢該使用者的權限，並使用一份寫死的對應表將權限轉換為 Redfish
角色。這份寫死的對應表列於下方：

| Phosphor-user-manager 權限（以群組實作） | BMCWeb Redfish 角色 |
| ---------------------------------------- | ------------------- |
| priv-admin                               | Administrator       |
| priv-operator                            | Operator            |
| priv-user                                | ReadOnly            |
| priv-noaccess                            | NoAccess            |

為了將 Redfish 角色對應到它們被指派的 Redfish 權限，BMCWeb 實作了
以下寫死的對應表：

| BMCWeb Redfish 角色 | 被指派的 Redfish 權限                                                                  |
| ------------------- | -------------------------------------------------------------------------------------- |
| Administrator       | "Login", "ConfigureManager", "ConfigureUsers", "ConfigureSelf", "ConfigureComponents"} |
| Operator            | "Login", "ConfigureSelf", "ConfigureComponents"                                        |
| ReadOnly            | "Login", "ConfigureSelf"                                                               |
| NoAccess            | NA                                                                                     |

在編譯時期，BMCWeb 會為每個實體的每個操作指派一組所需的 Redfish 權限。
當執行某個 BMCWeb 路由回呼（route callback）時，會執行一次授權檢查：
檢查被指派的 Redfish 權限是否為所需 Redfish 權限的超集合（superset）。

在後續章節中，我們將 BMCWeb 路由（route）稱之為某個給定資源 URI 上
某項操作的處理程式（handler），也就是 BMCWEB_ROUTE 巨集所定義的內容。

參考資料：

1. <https://github.com/openbmc/bmcweb/blob/d9f6c621036162e9071ce3c3a333b4544c6db870/include/authentication.hpp>
2. <https://github.com/openbmc/bmcweb/blob/d9f6c621036162e9071ce3c3a333b4544c6db870/http/http_connection.hpp>
3. <https://github.com/openbmc/bmcweb/blob/d9f6c621036162e9071ce3c3a333b4544c6db870/redfish-core/lib/roles.hpp>

### 缺口

如上所述，目前大多數的 Redfish 授權設定都是在編譯時期配置的，包括：

1. Phosphor-user-manager 權限的集合
2. 從 Phosphor-user-manager 權限到 Redfish 角色的對應
3. Redfish 角色的集合
4. 從 Redfish 角色到 Redfish 權限的對應
5. 操作對權限對應

然而，現代系統有一些使用案例，需要在系統持續運行的情況下變更 Redfish
角色、Redfish 權限以及操作對權限對應。例如，引入了一個新的微服務，
需要與叢集中的既有 BMC 通訊，我們需要推送一份設定，讓這個新的同儕
獲得適當的 Redfish 角色，並被授權存取某些資源，而不需要推送新的 BMC
韌體。

另一個缺口是，目前的 Redfish 角色與操作對權限對應導致非常粗粒度的
存取控制，並未實作最小權限原則（principle of least privilege）。雖然
這些設定是由 DMTF 定義的，但標準中明確指出，當「標準權限過於寬鬆」時，
實作端可以自行定義它們自己的 OEM 角色與權限。

對於有些系統要求某個給定使用者只能存取它所需的資源。舉例來說，一個
負責遠端電源控制的微服務只能對 Chassis 與 ComputerSystems 發送 GET，
並對對應的 ResetActions 發送 POST。以現有的實作而言，這樣的微服務
至少需要 ConfigureComponents 這個 Redfish 權限，而這將導致它能夠
修改（patch）一個 EthernetInterface 資源。

## 需求

BMC 實作一套動態 Redfish 授權系統：

1. 用戶端應能在不需重新編譯的情況下新增新的 OEM Redfish 權限
2. 用戶端應能在不需重新編譯的情況下新增新的 OEM Redfish 角色，並為其
   指派任何既有的 Redfish 權限
3. 用戶端應能在不需重新編譯的情況下修改既有的操作對權限對應
4. 上述系統上的變更應為原子性的；也就是說，一旦變更完成，所有新的
   查詢都會使用最新的設定
5. BMC 應對上述修改執行合理性檢查（sanity check）；也就是
   - 它會拒絕格式不良的修改
   - 它會拒絕修改非 OEM 的權限
   - 當有任何使用者（本地端或遠端）對應到某個 OEM Redfish 角色時，
     它會拒絕刪除該 OEM Redfish 角色
   - 當有任何 OEM Redfish 角色被指派了某個 OEM Redfish 權限時，它會
     拒絕刪除該 OEM Redfish 權限
6. BMC 應持久化所有上述修改，並能從當機中復原
7. 採用靜態授權的既有系統應如同此功能未被引入般運作，包含非 Redfish
   路由（例如 KVM websocket）
8. 預設的 OEM 角色與 Redfish 權限必須能在編譯時期以每個系統為單位
   進行選擇；預設的 Redfish PrivilegeRegistry 必須能在編譯時期以每個
   系統為單位進行設定
9. 此功能所使用的總儲存空間應有所限制；這是指
   - 在啟用了動態授權功能的系統上，rwfs 磁碟使用量的總增加應少於
     100 KB
   - 在啟用了動態授權功能的系統上，執行期記憶體使用量的增加應少於
     1 MB
   - 在所有系統上，被修改的常駐程式的二進位檔大小增加應少於 100 KB
10. BMC 實作一套完整的權限 registry；也就是
    - 它應在編譯時期實作 Redfish 基礎 Privilege registry 中的所有覆寫；
      它應支援在執行期設定覆寫，但實作可以先從靜態設定開始，並拒絕
      執行期的修改
    - BMC 暴露 PrivilegeRegistry，代表目前的設定並反映執行期的變更
    - 對資源實體的變更應自動傳遞到目前的權限 registry
11. 新的 Redfish 資源可以在不修改自訂 PrivilegeRegistry 的情況下實作
12. 上述所有內容都應可測試；CI 必須能夠找出問題

## 建議設計

### 對應：使用者、Redfish 角色與 Redfish 權限

如背景章節所述，既有的 Redfish 角色是以帶 "priv" 前綴的 Linux 次要
群組儲存（包含 "priv-admin"、"priv-operator" 與 "priv-user"）。而一個
Linux 使用者會位於其中一個群組中，代表該使用者是某個特定的 Redfish
角色。BMCWeb 接著使用一份寫死的表格將 Redfish 角色對應到 Redfish
權限。

然而，僅將角色建模為靜態的 Linux 次要群組，並不適用於 OEM Redfish
角色，因為一個 Redfish 角色會對應到一組動態的 OEM 權限：次要群組無法
表示關聯性（associations）。

為了解決這個問題，我們提出以下方案：

**將 Redfish 角色儲存為 Linux 使用者與次要群組** 我們提議將 Redfish
角色同時儲存為 Linux 使用者與次要群組。儲存為次要群組是為了將使用者
與 Redfish 角色關聯起來。另一方面，儲存為使用者是為了將 Redfish 角色
與 Redfish 權限關聯起來。這些對應詳見下方章節。

用於 Redfish 角色的使用者不會屬於任何預先定義的群組（web、redfish、
ipmi）。我們可以新增額外屬性，將它們與真正的本地端及 LDAP 使用者區分
開來。用於 Redfish 角色的使用者也不會擁有 SSH 權限。

Redfish 角色會有一個固定的前綴 "openbmc-rfr-"。"rfr" 指的是 Redfish
角色。OEM redfish 角色會取得前綴 "openbmc-orfr-"。"orfr" 指的是 OEM
Redfish 角色。舉例來說，基礎 Redfish 角色 "Administrator" 會產生一個
Linux 次要群組 "openbmc-rfr-administrator" 以及一個本地端使用者
"openbmc-rfr-administrator"。我們也可以讓廠商名稱（"openbmc"）在編譯
時期可設定。使用縮寫是為了節省字元，因為 Linux 使用者名稱有 31 個字元
的限制。

**將 Redfish 權限儲存為次要群組** Redfish 權限會以帶固定前綴
"openbmc-rfp" 的 Linux 次要群組儲存。rfr" 指的是 Redfish 權限。OEM
權限會有固定前綴 "openbmc-orfp"。"orfr" 指的是 OEM Redfish 權限。

**使用者名稱到 Redfish 角色的對應** 將使用者名稱對應到 Redfish 角色，
變成搜尋該使用者所屬、以 "openbmc-rfr" 開頭的群組。

**Redfish 角色到 Redfish 權限的對應** 將 Redfish 角色對應到 Redfish
權限，變成搜尋該使用者所有以 "openbmc-rfp" 或 "openbmc-orfp" 開頭的
群組。

一個使用者可能屬於多個 Linux 次要群組，代表他們被指派了某些權限；
舉例來說，使用者 "PowerService" 位於 "openbmc-orfr-power" 群組中，
代表它的 Redfish 角色是 "openbmc-orfr-power"；本地端使用者
"openbmc-orfr-power" 位於次要群組 "openbmc-rfp-configureself" 與
"openbmc-orfp-power" 中，代表 "openbmc-orfr-power" 被指派了 Redfish
權限 "ConfigureSelf" 與 "OemPrivPower"。

下圖說明了一個具有身分識別（在 PAM 中為使用者名稱，或在 TLS 中為
CN/SAN）"power-service" 的電源控制服務，其被指派的權限是如何被解析的。

```ascii
 +-----------------+
 |  power service  |
 |                 |
 +--------|--------+
          |
          |
 +-----------------+
 |  authentication |
 |   (PAM or TLS)  |
 |      BMCWeb     |
 +--------|--------+
          |
          |username:power-service
          |
 +-----------------+                        +-----------------+            +----------------------------+
 |      BMCWeb     |DBus: privileges        |  phosphor-      |            |     Linux                  |
 |                 ------------------------>|   user-manager  |            |user: power-service         |
 |                 |<-----------------------|                 |            |group:                      |
 |                 | privileges:            |                 <----------->|  openbmc-orfr-power        |
 |                 |  Login, ConfigureSelf  |                 |            |                            |
 |                 |  OemPrivPower          |                 |            |user: openbmc-orfr-power    |
 +-----------------+                        +-----------------+            |group:                      |
                                                                            |  openbmc-rfp-configureself |
                                                                            |  openbmc-orfp-power        |
                                                                            |  openbmc-rfp-login         |
                                                                            +----------------------------+
```

上圖對 LDAP 使用者同樣適用。一個遠端角色或群組可以透過 RemoteRoleMapping
對應到基礎 Redfish 角色或 OEM Redfish 角色：一個 LDAP 群組對應到一個
以本地端使用者儲存的單一 Redfish 角色。

我們提議擴充既有的 phosphor-user-manager：

1. 它透過查詢目前的群組來動態回傳 AllPrivileges
2. Phosphor-user-manager 提供 DBus API 以查詢某個給定使用者的權限

傳統群組（例如 `priv-user`）仍一如往常地運作。舉例來說，一個同時位於
`priv-user` 與 `openbmc-orfp-power` 的使用者將擁有下列 Redfish 權限：
`Login`、`ConfigureSelf`、`OemPrivPower`。

### 建立/刪除：使用者、Redfish 角色與 Redfish 權限

基礎權限與角色將不允許在執行期變更。

#### OEM Redfish 權限

對 PrivilegeRegistry 中的 OEMPrivilegesUsed 進行 PATCH，以建立/刪除
OEM 權限，從而在執行期建立或刪除 OEM Privileges。

我們提議擴充既有的 phosphor-user-manager：

1. Phosphor-user-manager 提供 DBus API 以建立/刪除 OEM Redfish 權限；
   在底層，它將權限儲存為 Linux 群組，就像目前基礎 Redfish 角色的
   儲存方式一樣
2. Phosphor-user-manager 維護一個 Redfish 權限的最大數量；考慮到快速的
   位元運算，我們在第一版提議為 32
3. Phosphor-user-manager 執行驗證：
   - OEM Redfish 權限的名稱必須唯一且有效；例如，以 "openbmc-orfp-"
     開頭
   - 拒絕刪除目前仍在使用的權限（已被指派給任何有使用者關聯的 Redfish
     角色）

系統也可以選擇性地透過 useradd class 中 Yocto 的 GROUPADD_PARAM，在
編譯時期新增 OEM Privileges。

#### OEM Redfish 角色

對 AccountService 中的 RoleCollection 進行 POST 以建立 OEM 角色，並在
Role 資源的 AssignedPrivileges 與 OemPrivileges 屬性中帶入被指派的
Privileges。

對 RoleCollection 中特定的 Role 進行 DELETE 以刪除 OEM 角色。預先定義
的角色不允許被刪除。

我們提議擴充既有的 phosphor-user-manager：

1. Phosphor-user-manager 提供 DBus API 以建立 Redfish 角色
2. Phosphor-user-manager 維護一個 Redfish 角色的最大數量；考慮到快速的
   位元運算，我們在第一版提議為 32
3. Phosphor-user-manager 執行驗證：
   - OEM Redfish 角色的名稱必須唯一且有效；例如，以 "openbmc-orfr-"
     開頭
   - 拒絕刪除目前仍在使用的 RedfishRole（已與使用者關聯）

#### 使用者

對 ManagerAccountCollection 進行 POST 以建立使用者，並帶一個指向被指派
Redfish 角色的 RoleId。

對 ManagerAccountCollection 中特定的使用者進行 DELETE 以刪除使用者。

注意：對 ManagerAccountCollection 的修改同樣需要授權。舉例來說，一個
僅具有 "ConfigureSelf" 權限的使用者不允許刪除其他使用者。

#### 典型工作流程

總結來說，下方描繪了一個典型的工作流程：建立一個新的本地端使用者，
其具有一個新的 OEM Redfish 角色，而該角色被指派了一組新的 OEM Redfish
Privileges。

```ascii
         Root User                    BMCWeb                  Phosphor-User-Manager             Linux
              |   PATCH PrivilegeRegistry |                              |                        |
              |-------------------------->|   DBus: createGroup          |                        |
              |   Add OemAddPowerService  |----------------------------->|     groupadd           |
  Create      |                           |                              |----------------------->|
  OemPrivilege|                           |            Okay              |<-----------------------|
              |           Okay            |<-----------------------------|        Okay            |
              |<--------------------------|                              |                        |
              |                           |                              |                        |
              |                           |                              |                        |
              | POST RoleCollection       |                              |                        |
              |-------------------------->|  DBus: createUser            |                        |
              |                           |----------------------------->|  useradd + groupadd    |
  Create      |                           |                              |----------------------->|
  OemRole     |                           |            Okay              |<-----------------------|
              |          Okay             |<-----------------------------|         Okay           |
              |<--------------------------|                              |                        |
              |                           |                              |                        |
              |                           |                              |                        |
              |POST                       |                              |                        |
              | ManagerAccountCollection  |                              |                        |
              |-------------------------->| DBus: createUser             |                        |
              |                           |----------------------------->|       useradd          |
  Create      |                           |                              |----------------------->|
  User        |                           |                              |<-----------------------|
              |                           |<-----------------------------|        Okay            |
              |<--------------------------|            Okay              |                        |
              |           Okay            |                              |                        |
  Time        |                           |                              |                        |
              v                           v                              v                        v
```

### 非 Redfish 路由或 OEM 資源

我們仍保留目前的 `privileges` C++ API，以透過 `BMCWEB_ROUTE` 為非
redfish 路由新增明確的 Redfish 權限。

### Redfish 路由

我們提議建立一個新的巨集 `REDFISH_ROUTE`，讓既有 `REDFISH_ROUTE` 對於
非 redfish 路由維持不變。

對於每個 `REDFISH_ROUTE`，本設計提議建立下列額外的 C++ API，而非接收
一個權限陣列（或對權限陣列的參照）：

1. `entity`：它接收一個代表資源名稱的字串（與 PrivilegeRegistry 中的
   定義相同）；例如，"/redfish/v1/Managers/${ManagerId}/EthernetInterfaces/"
   會提供字串 "EthernetInterfaceCollection" 作為實體
2. `subordinateTo`：它接收一個字串陣列，代表此路由器隸屬於（subordinate
   to）哪些資源（與 PrivilegeRegistry 中的定義相同）；例如，一個 URL 為
   "/redfish/v1/Managers/${ManagerId}/EthernetInterfaces/" 的路由會提供
   一個 {"Manager", "EthernetInterfaceCollection"} 的陣列作為
   `subordinateTo` 的值。

任何 Redfish 路由都必須提供這些屬性。非 Redfish 路由不應提供它們，
而是直接指定 `privileges`。這些屬性的值可以在編譯時期從 schema 產生。
為了在 C++ 程式碼中保證這一點，我們可以將它們設為建構子中的必要參數。

下方說明我們如何提議透過使用建議的 `entity` 與 `subordinateTo`、既有的
`methods`，以及來自請求的 URL，來取得某個給定資源上某個給定方法所需的
Redfish 權限。

關於我們如何能免去手動設定這些屬性，請參閱替代方案章節。

### 記憶體中的操作對權限對應資料結構

BMCWeb 在記憶體中維護一個類似 JSON 的資料結構，以代表整體的操作對權限
對應。對於一個已知實體名稱、HTTP 方法與資源 URL 的給定路由，可以有效地
取得所需的權限集合。

用來檢查某個使用者是否有權執行某項 Redfish 操作的操作，仍然是既有的
位元運算 `isSupersetOf`，比較的是某個給定資源上某項給定操作所需的權限
與某個角色被指派的權限。

### 在編譯時期產生操作對權限對應資料結構

BMCWeb 有一項需求：它不傾向在服務啟動時載入一個大型檔案，因為這會拖慢
服務以及任何依賴它的服務。

因此，我們提議在編譯時期產生這個資料結構，它接收一個 PrivilegeRegistry
JSON，並產生必要的標頭檔與來源檔，以持有一個代表整體操作對權限對應的
變數。輸入的 JSON 可以因系統而異。

這可以實作為一個自訂的 Meson generator。

我們將刪除目前的靜態權限標頭檔，但產生的標頭檔會增加二進位檔的大小。
我們應將此增加限制在 <= 100KB。`Redfish_1.3.0_PrivilegeRegistry.json`
的大小約為 275 KB；它的壓縮版（minified，無空白字元）約為 62 KB。當
解析這個 JSON 並將其轉換為 C++ 程式碼時，我們不應讓二進位檔的大小增加
太多，否則我們乾脆把整個 registry 儲存為一個 Nlohmann JSON 物件即可。

### 操作對權限對應覆寫

在路由程式碼中，我們可以解析操作對權限對應資料結構，尋找
SubordinateOverrides 與 ResourceURIOverrides，將它們與來自路由及請求的
屬性結合，並執行授權。

授權程式碼的大多數部分會在資源處理程式之前執行。然而，讀取操作的
PropertyOverrides 只能在回應準備好時才能執行，因為我們需要檢查回應的
屬性。寫入操作的 PropertyOverrides 仍應在處理程式碼之前執行：授權程式碼
檢查請求的載荷（payload）與對應的屬性，並在記憶體中的操作對權限對應
資料結構中查詢它們。

下方描繪了一個讀取操作的範例執行流程。

```ascii
    +---------------+
    |   BMCWeb      | Get
    |    routing    | /redfish/v1/Managers/${ManagerId}/EthernetInterfaces/
    +-------|-------+
            | Known information:
            |  Request.URL
            |  Request.method
            |  Route.entity
            |  Route.subordinateTo
            |
 +-----------------------+       +--------------------------------------------------------------+
 | Any                   |       |Operation-to-Privilege Mapping in-memory Data Structure       |
 | ResourceURIOverrides? <------>|                                                              |
 |                       |       |{                                                             |
 +-----------|-----------+       |   "Entity": "EthernetInterface",                             |
            |Updated            |   "OperationMap": {                                          |
            |RequiredPrivileges |      "Get": [{                                               |
 +-----------------------+       |         "Privilege": ["ConfigureComponents"]                 |
 | Any                   |       |      }]                                                      |
 | SubordinateOverrides? |<----->|   },                                                         |
 |                       |       |   "SubordinateOverrides": [{                                 |
 +-----------------------+       |      "Targets": ["Manager", "EthernetInterfaceCollection"],  |
            |Updated            |      "OperationMap": {                                       |
            |RequiredPrivileges |         "Get": [{                                            |
            v                   |            "Privilege": ["ConfigureManager"]                 |
 +-----------------------+       |         }]                                                   |
 | Authorization         |       |      }                                                       |
 +-----------|-----------+       |   }]                                                         |
            |Okay               |}                                                             |
            |Got Response       |                                                              |
            |                   +--------------------------------------------------------------+
 +-----------------------+                       ^
 | Any                   |                       |
 | PropertyOverrides?    |<----------------------+
 +-----------|-----------+
            |
            |
            v
        Final Response
```

實作可以先從僅在編譯時期實作 DMTF 基礎 PrivilegeRegistry 中所規定的
覆寫開始，並拒絕對覆寫屬性進行任何 PATCH。

### PrivilegeRegistry 資源

BMCWeb 將在一個新的路由中實作 PrivilegeRegistry。該路由會反映目前使用中
的 PrivilegeRegistry，包含動態變更。

有了建議的動態操作對權限對應資料結構，以及 phosphor-user-manager 提供的
DBus API，實作就非常簡單：將 JSON 資料結構轉換成 JSON 回應。

### 對 PrivilegeRegistry 進行 PATCH

雖然 Redfish 規格宣告 PrivilegeRegistry 為唯讀，本設計仍提議實作對
PrivilegeRegistry 的 PATCH：使用者可以直接 PATCH 任何屬性，例如，修改
Entity EthernetInterface 的 OperationMap 的 POST 權限陣列，讓具有
"OemEthernetManager" 的使用者也能對 EthernetInterface 發送 GET。

```json
{
    Entity": "EthernetInterface",
    "OperationMap": {
        "GET": [
            {
                "Privilege": [
                    "Login"
                ]
            },
            {
                "Privilege": [
                    "OemEthernetManager"
                ]
            }
        ]
    }
}
```

當對應的實作尚未就緒時，實作可能會拒絕對某些屬性的修改。例如，當服務
尚未實作 SubordinateOverrides 時，它會拒絕修改 SubordinateOverrides。

針對基礎 PrivilegeRegistry 的變更將被拒絕，例如，從某個 OperationMap
刪除 ConfigureSelf。本設計是針對 OEMPrivileges 與 OEMRoles。

### 持久化資料

OEM Redfish 角色、Redfish 權限與使用者都由 Linux 持久化。當角色與權限的
最大數量已設定後，總持久化資料會非常小（約幾 KB）。

為了將持久化資料的大小降到最低，我們提議 BMCWeb 只儲存對 PrivilegeRegistry
的 PATCH 請求序列。這份資料可以儲存在既有的 `bmcweb_persistent_data.json`
中。當從當機或重設重新啟動時，BMCWeb 將能夠執行這份 PATCH 請求序列以復原
PrivilegeRegistry。對既有 `bmcweb_persistent_data.json` 的一項變更在於：
現在 BMCWeb 會在對客戶端於 PrivilegeRegistry 上的 PATCH 回傳 okay 之前，
先將變更寫入磁碟（commit）。鑑於對 PrivilegeRegistry 的操作遠比其他管理與
監控資源的操作稀少，將一小段資料寫入磁碟是可以接受的。

我們可以設定 BMCWeb 允許的 PATCH 請求最大數量，以限制磁碟使用量。大多數
上游系統也配置了幾 MB 的讀寫分割區。例如，在本設計撰寫時，`romulus` 的
rwfs 為 4194304 bytes。我們提議從允許 1000 個請求開始。

未啟用動態授權功能的系統不會新增任何新的持久化資料。

## 已考慮的替代方案

### 從 URL 推論 Entity 與 SubordinateTo

我們可以透過建立一個類似 Trie 的資料結構，從 URL 推論實體。然而，為
一個路由寫死一個實體並不是什麼大問題，因為實體與 SubordinateTo 在執行期
從不變更。

## 影響

1. phosphor-user-manager 上新的 DBus 介面
2. 將在 BMC 上新增由 BMCWeb 管理的持久化資料
3. BMCWeb 的二進位檔大小將增加
4. 採用靜態授權的既有系統應如同此功能未被引入般運作

## 組織面

不需要新的 repository。將修改 Phosphor-user-manager 與 BMCWeb 以實作本設計。

## 測試

既有測試應如同此功能未被引入般仍然通過。可以新增新的 Robot Framework
測試，以測試 PrivilegeRegistry 的執行期修改。

測試案例應包含：

1. 驗證在未進行執行期修改時，基礎 Redfish 角色、權限與基礎操作對權限
   對應都遵循 Redfish 規格
2. 驗證 Redfish OemPrivilege 可以透過對 PrivilegeRegistry 的 PATCH 新增，
   並反映在 PrivilegeRegistry 資源中
3. 驗證 Redfish OemRole 可以透過對 ManagerAccountCollection 的 POST 新增
   並帶被指派的 OemPrivilege，且反映在 ManagerAccountCollection 中
4. 驗證操作對權限對應可以透過對 PrivilegeRegistry 的 PATCH 修改；可以用
   上述 OemPrivilege 新增某資源上某項動作的對應，且最終該 OemRole 的使用者
   現在可以存取該資源
5. 驗證這 3 種動態覆寫如預期運作；例如，可以新增一個新的覆寫到
   PrivilegeRegistry，且應反映在新的請求中
