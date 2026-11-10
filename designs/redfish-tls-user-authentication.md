# Redfish TLS 使用者認證

作者：Kamil Kowalski <kamil.kowalski@intel.com>

其他貢獻者：無

建立日期：2019 年 6 月 7 日

## 問題描述

由 [BMCWeb](https://github.com/openbmc/bmcweb) 提供的 Redfish API 允許使用者透過多種方法進行認證，例如 BasicAuth、Sessions 等。除了這些方式之外，使用者還可以在協商 HTTPS 連線時提供憑證以供識別，從而取得節點的存取權。此解決方案的設計與背後的原理說明如下。

## 背景與參考資料

Redfish 目前缺乏對現代認證方式的支援。基於憑證的認證能夠帶來更安全、更可控制的使用者存取控制。使用 SSL 憑證可以提供有效期限、從 CA 層級撤銷存取權的能力，以及許多其他安全性功能。

參考文件：

- [Certificate Schema Definition](https://redfish.dmtf.org/schemas/v1/Certificate_v1.xml)
- [CertificateLocations Schema Definition](https://redfish.dmtf.org/schemas/v1/CertificateLocations_v1.xml)
- [CertificateService Schema Definition](https://redfish.dmtf.org/schemas/v1/CertificateService_v1.xml)
- [DSP-IS0008 DMTF 的 Redfish 憑證管理文件](https://www.dmtf.org/dsp/DSP-IS0008)
- [RFC 5246 - TLS 1.2 規格](https://tools.ietf.org/html/rfc5246)
- [RFC 8446 - TLS 1.3 規格](https://tools.ietf.org/html/rfc8446)

### 詞彙表

**Redfish API** - 由 DMTF 定義的 Redfish API **Redfish** - Redfish API 在 BMCWeb 中的實作

## 需求

加入此功能將大幅提升 WebUI 與 Redfish API 的安全性，並使其朝符合現代安全標準的方向邁進。

## 建議設計

### 流程總覽

每當 `CA` 的憑證變更時，`User` 就必須將其提供給 `Redfish`。完成後，使用者應向 `Redfish` 請求一份 **CSR**（**C**ertificate **S**igning **R**equest，憑證簽署請求），以取得一份可據以從 `CA` 產生正確 `user` 憑證的請求。取得此憑證後，`User` 便可以在初始化 HTTPS 工作階段時使用此憑證。

```text
┌──┐                           ┌────┐                                 ┌───────┐
│CA│                           │User│                                 │Redfish│
└┬─┘                           └─┬──┘                                 └───┬───┘
 │    Request CA's certificate   │                                        │
 │ <──────────────────────────────                                        │
 │                               │                                        │
 │    Return CA's certificate    │                                        │
 │  ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─>                                        │
 │                               │                                        │
 │                               │          Upload CA Certificate         │
 │                               │ ───────────────────────────────────────>
 │                               │                                        │
 │                               ──────────┐                              │
 │                                         │ Generate CSR                 │
 │                               <─────────┘                              │
 │                               │                                        │
 │ Request certificate using CSR │                                        │
 │ <──────────────────────────────                                        │
 │                               │                                        │
 │   Return User's certificate   │                                        │
 │  ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─>                                        │
 │                               │                                        │
 │                               │                                        │
 │                   ╔═══════╤═══╪════════════════════════════════════════╪════╗
 │                   ║ LOOP  │  Typical runtime                           │    ║
 │                   ╟───────┘   │                                        │    ║
 │                   ║           │         Initiate HTTPS Session         │    ║
 │                   ║           │ ───────────────────────────────────────>    ║
 │                   ║           │                                        │    ║
 │                   ║           │   Request TLS client authentication    │    ║
 │                   ║           │ <─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─    ║
 │                   ║           │                                        │    ║
 │                   ║           │          Provide certificate           │    ║
 │                   ║           │ ───────────────────────────────────────>    ║
 │                   ║           │                                        │    ║
 │                   ║           │          Return requested data         │    ║
 │                   ║           │ <─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─    ║
 │                   ╚═══════════╪════════════════════════════════════════╪════╝
┌┴─┐                           ┌─┴──┐                                 ┌───┴───┐
│CA│                           │User│                                 │Redfish│
└──┘                           └────┘                                 └───────┘
```

### BMCWeb / Redfish API

#### 上傳 CA 憑證

用於使用者認證的 CA 憑證存放於
`/redfish/v1/AccountService/TLSAuth/Certificates`。此處可以存放多個憑證，
因此使用者必須使用由**任何一個**在此處存有有效憑證的 CA 所簽署的憑證。新的憑證可以透過在 CertificateCollection 上以 *POST* 方式送出新的憑證物件來上傳。

POST 內容範例：

```json
{
  "CertificateString": "... <Certificate String> ...",
  "CertificateType": "PEM"
}
```

若 CA 憑證變為無效（遭洩露、過期等），建議使用位於
`/redfish/v1/CertificateService` 的 `#CertificateService.ReplaceCertificate` 動作，以避免為處理無效憑證而不必要地耗費空間與效能。

在
`/redfish/v1/CertificateService/Actions/CertificateService.ReplaceCertificate` 上執行的 `#CertificateService.ReplaceCertificate` 動作內容範例：

```json
{
  "CertificateUri": "/redfish/v1/AccountService/TLSAuth/Certificates/1",
  "CertificateString": "... <Certificate String> ...",
  "CertificateType": "PEM"
}
```

#### 產生 CSR

使用者可以使用任何對其而言方便的方式來產生 CSR。

#### 認證流程

```text
                                    +-+
                                    +++
                                     |
                                     V
                        +------------+------------+
                  Yes   |                         |
              +---------+  Is certificate valid   |
              |         | and signed by known CA? |
              |         |                         |
              |         +------------+------------+
              |                      |
              |                      | No
              |                      V
              |          +-----------+-----------+
              |    Yes   |                       |
              +----------+  Is URI whitelisted?  |
              |          |                       |
              |          +-----------+-----------+
              |                      |
              |                      | No
              |                      V
              |          +-----------+-----------+
              |    Yes   |                       |
              +----------+ Is X-Token provided?  |
              |          |                       |
              |          +-----------+-----------+
              |                      |
              |                      | No
              |                      V
              |          +-----------+-----------+
              |    Yes   |                       |
              +----------+  Is cookie provided?  |
              |          |                       |
              |          +-----------+-----------+
              |                      |
              |                      | No
              |                      V
              |          +-----------+-----------+
              |    Yes   |                       |
              +----------+   Is Token provided?  |
              |          |                       |
              |          +-----------+-----------+
              |                      |
              |                      | No
              |                      V
              |      +---------------+--------------+
              | Yes  |                              |  No
              +------+ Is Basic auth data provided? +------+
              |      |                              |      |
              |      +------------------------------+      |
              V                                            V
 +-------------+--------------+               +-------------+--------------+
 |                            |               |                            |
 |       Create session       |               | Return authorization error |
 |                            |               |                            |
 +-------------+--------------+               +-------------+--------------+
              |                                            |
              |                     +-+                    |
              +--------------------->*<--------------------+
                                    +-+
```

基於憑證的認證擁有最高的優先順序，這是因為
`Boost.Beast/Boost.ASIO/OpenSSL` 的設計，憑證驗證是在 HTTPS 請求處理的一開始就進行。`OpenSSL` 函式庫負責判定憑證是否有效。要讓憑證被標記為有效，該憑證（以及憑證鏈中的每一張憑證）都必須滿足下列條件：

- KeyUsage 是否包含所需的資料（"digitalSignature" 與 "keyAgreement"）
- ExtendedKeyUsage 是否包含所需的資料（包含 "clientAuth"）
- 公鑰是否符合最小位元長度需求
- 憑證必須處於其有效期限內
- notBefore 與 notAfter 欄位必須包含有效的時間
- 必須已由憑證授權單位正確簽署
- 憑證不得已被撤銷
- 憑證必須依照 X.509 正確格式編製
- 憑證不得為自我簽署
- 發行者名稱必須與 CA 的主體名稱相符

完成這些檢查後，會呼叫一個回呼函式，提供 `user<->CA`
的比對結果。若比對成功，Redfish 會從 `CommonName` 擷取出使用者名稱，並驗證該使用者是否存在於系統中。

如流程圖所示，Redfish 會按照處理順序使用**第一個有效**的憑證。建議使用者在單一請求中只使用一組憑證/認證資料，以確保知道將使用哪一組，否則將無法確定操作過程中使用的是哪一組憑證。

使用者可以在 `/redfish/v1/AccountService` 的 OEM schema 中設定可使用哪些方式。無論如何設定，憑證驗證的順序都保持不變。白名單驗證永遠開啟，這是因為 Redfish 規格及其他可存取性的需求。

使用者憑證不一定要由那些憑證已存放的確切 CA 所簽署，而是可以透過憑證鏈來完成（Redfish 保證支援最深 5 層的憑證鏈，但更深的鏈也可能運作）。建議至少使用 2048 位元的 RSA 或 256/384 位元的橢圓曲線金鑰。憑證在初始化工作階段的當下必須處於其有效期限內。

#### 授權

以任何上述方式識別出的使用者，會經過一個檢查流程，確認該使用者確實存在，以及應該提供哪些權限、群組等。目前的基礎是 BasicAuth，因為它應用於建立可在後續連線中使用的工作階段，並由 BMCWeb 透過使用 PAM 函式庫來執行。其他認證方式只能取得使用者的登入憑證而不含密碼，因此無法透過傳統 PAM 流程直接完成使用者是否存在於系統的驗證，必須以其他方式進行。基於憑證的認證也適用這一點，因此所有非 BasicAuth 的方式都應在每次登入嘗試時驗證使用者是否存在且未被系統鎖定。

## 其他考慮過的替代方案

無。

## 影響

現有的認證方式不會受到影響。此提案是以本地儲存的 CA 憑證為基礎，因此無法針對憑證已被撤銷、而使用者/管理員尚未在 BMC 上更新憑證的情況提供自動化的防護措施。

## 測試

應針對目前支援的 TLS 以外認證方式進行測試，以確認它們的行為沒有改變、也沒有出現任何效能或功能衰退。

至於 TLS 認證本身：

1. 應測試在 [流程總覽](#process-overview) 中所描述的流程，
   以確認完成該流程後一切都能如預期般運作。
2. 有效期限測試——確認尚未生效與已過期的憑證不會被接受，方法包括直接修改憑證本身的有效期限，以及修改 BMC 本身的時間。
3. 移除 CA 憑證，並確認使用者在此之後使用移除前原本可用的憑證時將不會被授權存取。
4. 憑證鏈驗證——確認憑證鏈中的憑證能如需求被接受。
5. 針對使用者憑證進行破壞性的反向測試——無效的使用者名稱、無效的有效期限、無效的 CA、損毀的二進位憑證等。
