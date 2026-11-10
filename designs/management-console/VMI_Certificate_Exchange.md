# VMI 憑證交換

作者：Raviteja Bailapudi

其他貢獻者：Ratan Gupta

建立日期：07/10/2019

## 字彙表

- HMC - Hardware Management Console：IBM 企業級伺服器的管理主控台。
- PHYP - Power Hypervisor：負責協調與管理系統虛擬化的軟體。
- VMI - Virtual Management Interface：促成 HMC 與 PHYP 內嵌 Linux
  虛擬機器之間通訊的介面。
- KVM - Kernel Virtual Machine：開放原始碼虛擬化軟體

## 問題描述

在企業級 POWER 系統上，硬體管理主控台（HMC）必須與虛擬化管理介面（VMI）建立
安全連線，以進行虛擬化管理。

VMI 是在 PHYP 上建立並執行、提供虛擬化功能的內嵌 Linux VM。

HMC 需要 client key、client.crt 及 CA.crt 才能與 VMI 建立安全連線。

BMC 必須提供憑證交換功能給管理主控台，原因如下：

- 主機韌體（PHYP）沒有驗證機制。
- 由於 HMC 與 BMC 之間已存在經過驗證的安全連線，VMI 信任 BMC 已經對任何連線
  的客戶端完成驗證並確認其真實性。

管理主控台需要一個 API，讓它能夠將 CSR 傳給 VMI（CA），並從 VMI 取得簽章後的
憑證以及 CA 憑證。本設計文件將說明管理主控台與 VMI 之間如何交換憑證。

IBM 系統可以執行 IBM 專用的主機韌體（PHYP）以及 Linux KVM。這個 API 僅供
基於 PHYP 的機器使用。

這個 API 的啟用與停用，會由建置時期（build time）可設定的變數來控制。

## 背景與參考

- VMI 將在 PHYP 上建立並執行，以提供虛擬化功能。
- 當 VMI 開機時，會產生一組公私鑰金鑰對，並使用該金鑰對建立一張自行簽章的
  root 憑證。
- VMI 僅作為 VMI 端點的 root CA，它並非正式的 CA，會使用其自行簽章的憑證來
  簽署來自客戶端的 CSR。
- HMC 必須與 VMI 建立安全連線，以執行虛擬化管理。

## 需求

BMC 將提供一個介面，讓管理主控台得以從 VMI 交換憑證資訊，以便 HMC 能與 VMI
建立安全連線。

## 擬議設計

管理主控台可以透過擬議的 BMC 介面，將 CSR 字串傳給 VMI（CA），並取得簽章後的
憑證及 Root CA 憑證。

以這個介面的角度來看，HTTP 錯誤碼可能是 4XX/5XX，將依據 PLDM 的錯誤回應來
對應。

HMC 可以查詢 BMC 狀態，並使用這個 API 來啟動憑證交換。如果 HMC 在 PHYP 開機
之前就執行這個指令，PLDM 指令會回傳錯誤；若 PLDM 指令丟出錯誤，將會對應到
Internal server Error（500）。

### 設計流程

```ascii
    +------------+        +--------+            +--------+
    |    HMC     |        |  BMC   |            |  VMI   |
    |  (client)  |        |        |            |  (CA)  |
    +-----+------+        +----+---+            +---+----+
          |                    |                    |
          |                    |                    |
          +------------------->+                    |
          | VMI Network info   |                    |
          +<-------------------+                    |
          |                    |                    |
 client.key|                    |                    |
 client.csr     SignCSR()       | pldm call to host  |
          +------------------->+------------------->|
          |                    |                    |  Sign CSR
          | SignCSR() response | pldm response from host
          +<-------------------+<-------------------|
          |                    |                    |
   Client.crt                   |                    |
   CA.crt                       |                    |
          |                    |                    |
          |                    |                    |
          |                    |                    |
          |                    |                    |
          +                    +                    +

```

### VMI 憑證交換

管理主控台應使用以下的 REST 指令與 VMI 交換憑證

#### 取得簽章憑證

用於從 VMI 取得簽章後客戶端憑證的 REST 指令

Request：

```bash
curl -k -H "X-Auth-Token:  <token>" -X POST "Content-Type: application/json" -d
  '{"CsrString":"<CSR string>"}' https://{BMC_IP}/ibm/v1/Host/Actions/SignCSR
```

Response：這會回傳包含簽章後客戶端憑證的憑證字串

```json
{
  "Certificate": "<certificate string>"
}
```

#### 取得 Root 憑證

用於取得 VMI root 憑證的 REST 指令

Request：

```bash
curl -k -H "X-Auth-Token:  <token>" -X GET http://{BMC_IP}/ibm/v1/Host/Certificate/root
```

Response：這會回傳包含 root CA 憑證的憑證字串。

```json
{
  "Certificate": "<certificate string>"
}
```

這個介面在失敗的情況下會回傳 HTTP 錯誤碼 5XX/4XX

## 已考慮的替代方案

我們已檢視現有的 BMC 憑證管理基礎設施，看是否能為這個使用案例加以延伸。

### 目前產生與安裝憑證的流程（基於 CSR）

- Certificate Signing Request（CSR）是由申請端傳給憑證授權中心的一則訊息，用以
  申請一張數位身分憑證。
- 使用者呼叫 CSR 介面，BMC 會建立新的私鑰以及 CSR 憑證檔。
- 將 CSR 憑證交給 CA 來簽署憑證，接著上傳經 CSR 簽章的憑證並安裝該憑證。

### 注意事項

- 我們現有的 BMC 憑證管理器／服務具有產生 CSR、上傳憑證，以及其他管理憑證的
  介面（replace、delete 等等）。
- 在 VMI 憑證交換中，BMC 的需求是提供一個介面，讓管理主控台得以取得經 VMI
  （CA）簽章的 CSR 憑證。
- 我們沒有任何現有的憑證管理器介面可以將 CSR 請求轉給 CA 請其簽章。
- 這裡的提案是提供一個 SignCSR() 介面，接受 CSR 字串並回傳簽章後的憑證及
  Root CA 憑證。
- 這個需求超出現有憑證管理器的範圍，因此提案將 SignCSR 介面作為管理主控台
  專用的介面。

### 替代設計

```ascii
    +------------+        +--------+            +--------+
    |    HMC     |        |  BMC   |            |  VMI   |
    |  (client)  |        |        |            |  PHYP  |
    +-----+------+        +----+---+            +---+----+
          |                    |                    |
          |                    |                    |
          +------------------->+                    |
          | VMI Network info   |                    |
          +<-------------------+                    |
          |                                         |
          |                SSL tunnel               |
          +---------------------------------------->|
          |              Verify Password            |Nets
          +---------------------------------------->|
          |                                         |
          |                  pldm                   |pldm call to authenticate
          +<-------------------+<-------------------|
          |                    |                    |
          |                   pam                   |
          |              authentication             |
          |                    +------------------->|
          |                                         |
          |        session established              |
          |<--------------------------------------->|

```

- 在這個替代設計中，管理主控台與 VMI 建立連線，並傳送 Verify Password 指令來
  驗證使用者，以建立安全工作階段。
- VMI 沒有驗證方法，因此 VMI 需要透過 PLDM 使用 BMC 的驗證方法。
- 若將原始密碼以明文透過 LPC 上的 PLDM 傳送，會有安全疑慮，因此這個設計被
  排除。

## 影響

- 在 webserver 中建立新的介面 GetRootCertificate，從 '/var/lib/bmcweb/RootCert'
  檔讀取 root 憑證。這個 API 可以同時處理多個請求。
- PLDM 在 VMI 開機後立即取得 root 憑證，並寫入
  '/var/lib/bmcweb/RootCert'。
- 實作 D-Bus 介面，為每個 signCSR 建立 dbus 物件，以便多個請求可以同時運作。
  D-bus service：
  xyz.openbmc_project.Certs.ca.authority.Manager Object path：
  /xyz/openbmc_project/certs/ca Interface：xyz.openbmc_project.Certs.Authority
  Method：SignCSR
- Dbus 物件包含 CSR、ClientCertificate 及 Status 屬性。
- PLDM 會為每個建立的物件聆聽 interface added 訊號，讀取 CSR 屬性取得 CSR
  字串，並將這個 CSR 字串轉給 VMI 以簽署此 CSR。
- 一旦 BMC 上的 PLDM 從 VMI 取得客戶端憑證，它會更新 ClientCertificate 的
  D-bus 屬性，並將 Dbus 物件中的 Status 屬性更新為 Complete。
- 在 webserver 中建立新的介面 SignCSR，接受 CSR 字串作為輸入並回傳憑證字串。
  這個介面會呼叫 SignCSR 的 dbus 方法，並聆聽 Status 屬性變更訊號來驗證狀態，
  讀取 ClientCertificate 屬性的內容並回傳憑證字串。
- 在完成處理 sign CSR 請求後，會在將憑證字串回傳給客戶端之前刪除對應的
  dbus 物件。
- BMC 僅作為 passthrough，讓 VMI 與 HMC 之間得以進行憑證交換。BMC 不會儲存或
  解析這些憑證。
- 已定義建置時期可設定變數，用以控制 webserver 中這個 API 的啟用與停用。它僅
  適用於搭配 IBM 專用主機韌體（PHYP）的 IBM 系統。

## 測試

- 從管理主控台測試介面指令，驗證憑證交換是否如預期運作，並驗證管理主控台是否
  能與 VMI 建立安全連線。

- 憑證交換在以下情境會失敗
  - 若 PHYP 未上線
  - 若 PHYP 因憑證驗證丟出錯誤。這個介面會依據錯誤類型回傳適當的 HTTP
    錯誤碼（4XX/5XX）。

- 若有憑證過期、憑證遭撤銷、日期／時間不正確或憑證不正確等問題，則 HMC 將
  無法與 VMI 建立連線。
