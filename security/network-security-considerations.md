# 網路安全性注意事項

本文說明基於 OpenBMC 的系統所提供的網路服務、BMC 從其網路介面所面臨的某些
威脅，以及 OpenBMC 為因應這些威脅所採取的措施。

本文僅作為指南之用；安全性最終仍是那些選擇將 OpenBMC 納入其專案的項目所應承擔
的責任。若您發現安全性弱點，請參考[如何回報安全性弱點][]。

[how to report a security vulnerability]:
  https://github.com/openbmc/docs/blob/master/security/how-to-report-a-security-vulnerability.md

對 BMC 的威脅以 [CIA 三要素][] 進行分類。所有威脅類型都相當重要；以下為各類型
的範例：

- 機密性（Confidentiality）：若攻擊者能從 BMC 取得資料，他們或許能串連其他弱點，
  建立隱密資訊通道，從主機取得敏感資訊。
- 完整性（Integrity）：若攻擊者能修改 BMC 的設定或資料，他們或許能取得額外存取權，
  並發動更多攻擊。
- 可用性（Availability）：若某代理（agent）能耗盡 BMC 的資源，無論是出於意外或刻意，
  BMC 都將無法為其主機提供服務（阻斷服務攻擊）。

[cia triad]: https://en.wikipedia.org/wiki/Information_security#Key_concepts

本文依照 OpenBMC 服務連接到網路的方式來組織。一般的流程為：

- 假設 BMC 具備網路轉接卡。NIC 的安全性注意事項對 BMC 安全性相當重要，但不在本文
  討論範圍之內。
- 網路流量接著會通過核心，詳見下文。
- 最後，連線會流向各項 OpenBMC 服務。

OpenBMC 在 TCP 與 UDP 埠上提供服務。例如，443 埠上的 HTTPS 協定用於提供 REST API
以及提供網頁應用程式服務。這些服務詳見下文。隱含的前提是所有其他埠皆未啟用。

OpenBMC 也會主動發起網路通訊，例如 NTP、LDAP 等。這些會在其相關功能中一併說明。

## 核心與 ICMP 訊息

網路流量由 Linux 核心處理。所使用的核心與裝置驅動程式有其安全性注意事項，對 BMC
安全性相當重要，但較適合由 Linux 核心社群來處理。您可以從機板支援套件（board
support packages）中的核心配方（recipes）得知所使用的核心與修補程式，這些套件由
您機器的設定所參照的 BMC 使用。例如，在 `https://github.com/openbmc/meta-aspeed`
儲存庫的 `recipes-kernel/linux/linux-aspeed_git.bb` 之下。

根據 [CVE 1999-0524][]，回應某些 ICMP 封包可能讓攻擊者取得更多關於 BMC 時鐘或子網路
的資訊，有助於其後續攻擊。OpenBMC 會回應所有的 ICMP 要求。

[cve 1999-0524]: https://nvd.nist.gov/vuln/detail/CVE-1999-0524

適用於 ICMP 訊息的一般注意事項亦適用，例如封包碎片化（packet fragmentation）與
封包洪氾（packet flooding）弱點。

為了除錯及其他診斷目的，有時會需要過濾並記錄網路訊息。OpenBMC 並未提供這方面的
支援。

## 服務的一般注意事項

有幾項服務會執行使用者辨識與驗證：

- Phosphor REST API
- Redfish REST API SessionService
- 網路 IPMI
- SSH 安全外殼（secure shell）

OpenBMC 的 [phosphor-user-manager][] 提供底層的驗證與授權功能，並與 IPMI、Linux PAM、
LDAP 及記錄整合。部分 OpenBMC 服務會使用 phosphor-user-manager。

[phosphor-user-manager]:
  https://github.com/openbmc/docs/blob/master/architecture/user-management.md

傳輸層安全性（TLS）協定在編譯時期為每項服務進行設定，成為映像檔的一部分，且無法
動態變更。使用 TLS 的協定包括：

- IPMI 的 RAKP。
- ssh 與 scp 使用的 SSH。
- 網頁與 REST API 使用的 HTTPS。

自動化網路代理（例如硬體管理主控台）可能以某種方式失靈，導致 BMC 持續發生驗證失敗，
進而可能造成阻斷服務。例如，在回報失敗之前加入短暫延遲，例如一秒，或許有助於防止
這個問題或減輕其嚴重性。請參閱 [OWASP Blocking Brute Force Attacks][]。

[owasp blocking brute force attacks]:
  https://www.owasp.org/index.php/Blocking_Brute_Force_Attacks

網路代理可能無法正確結束工作階段，導致服務耗用資源來追蹤孤兒工作階段（orphaned
sessions）。為協助防止這種情況，服務可以限制同時連線工作階段的數量上限，或設定
工作階段閒置逾時。

對於不需要的服務，應予以停用，以限制 BMC 的攻擊面。例如，大規模資料中心可能不需要
網頁介面。可以透過幾種方式停用服務：

1. 設定 OpenBMC 配方，將不需要的功能從 BMC 的韌體映像檔中建置移除。這能讓 BMC
   享有較小攻擊面的優勢。
2. 實作類似 [Redfish ManagerNetworkProtocol][] 屬性的機制，用於 IPMI、SSH 及其他
   BMC 服務，可能透過使用類似 'systemctl disable ipmid' 與 'systemctl stop ipmid'
   的 shell 指令來達成。

[redfish managernetworkprotocol]:
  https://redfish.dmtf.org/schemas/ManagerNetworkProtocol.v1_4_0.json

網路服務應該記錄所有驗證嘗試及其結果，以滿足基本的監控與鑑識分析需求。例如，作為
即時監控服務的一部分，或用來回答何人在何時存取了哪些服務。

OpenBMC 沒有防火牆。

法規可能要求基於 OpenBMC 打造的產品必須內建合理的安全性，例如不具備預設密碼。
請參閱 [CA Law SB-327]。

[ca law sb-327]:
  https://leginfo.legislature.ca.gov/faces/billTextClient.xhtml?bill_id=201720180SB327

## 在 TCP 與 UDP 埠上提供的服務

### TCP 埠 22 - 透過安全外殼（SSH）存取 BMC

提供安全外殼（SSH）協定，包含透過安全外殼（ssh 指令）存取 BMC 的 shell，以及透過
安全複製（scp 指令）存取 BMC 的檔案系統。

預設的 SSH 伺服器實作由 Dropbear 提供。所有設定都在編譯時期完成，其預設值涵蓋：

- 由 Linux PAM 提供的驗證，方法包含使用者名稱與密碼，以及 SSH 憑證（`ssh-keygen`
  指令）。
- 所提供的傳輸層安全性（TLS）協定。

透過 SSH 存取 BMC 的 shell 並非操作 BMC 的預期方式，會賦予操作員超過所需的權限，
且在服務處理敏感資料之主機的 BMC 上，可能不被允許。不過，在佈署（provision）BMC
或在 BMC 運作期間協助診斷問題時，可能需要 BMC 的 shell 存取。

### TCP 埠 443 - HTTPS REST API 與網頁應用程式

BMCWeb 是以下項目的網頁伺服器：

- Redfish REST API。
- webui-vue 網頁介面。
- Phosphor D-Bus REST 介面。並為以下項目發起 WebSocket：
- 主機 KVM。
- 虛擬媒體。
- 主機序列主控台。

[BMCWeb 設定][] 控制提供哪些服務。

適用於 HTTP 伺服器的一般安全性注意事項亦適用，例如 [OWASP Application Security][]
所提供的內容。

BMCWeb 透過 <https://github.com/openbmc/bmcweb> 儲存庫中編譯時期的標頭檔
`include/ssl_key_handler.hpp` 來控制其提供的 HTTPS 傳輸層安全性（TLS）加密算法。
實作由 OpenSSL 提供。

BMCWeb 提供適當的 HTTP 回應標頭，例如在 <https://github.com/openbmc/bmcweb>
儲存庫中的標頭檔 `include/security_headers_middleware.hpp` 與
`crow/include/crow/websocket.h`。

[bmcweb configuration]: https://github.com/openbmc/bmcweb#configuration
[owasp application security]:
  https://www.owasp.org/index.php/Category:OWASP_Application_Security_Verification_Standard_Project

#### REST API

BMCWeb 提供三種驗證方法：

1. Redfish SessionService，接受使用者名稱與密碼並提供 X-Auth 權杖。
2. Phosphor D-Bus REST 介面的 '/login' URI，接受使用者名稱與密碼並提供工作階段
   cookie。此方法在 OpenBMC 中已被棄用。
3. 基本存取驗證（Basic Access Authentication），在 "Authorization" 要求標頭中接受
   使用者名稱與密碼（常以 URL 編碼，例如 `https://user:pass@host/...`），且不回傳
   任何憑證。此方法已被 RFC 3986 棄用。

使用者名稱與密碼會提交給 phosphor-user-manager 進行驗證。

前兩種方法會建立相同類型的工作階段，但回傳不同的憑證。例如，您可以建立一個 Redfish
工作階段，並使用您的憑證來呼叫 Phosphor D-Bus REST API。不過請注意，使用 POST、PUT、
PATCH 或 DELETE 方法時需要 X-Auth 權杖。

適用於 REST API 的一般安全性注意事項亦適用：
<https://github.com/OWASP/CheatSheetSeries/blob/master/cheatsheets/REST_Security_Cheat_Sheet.md>

Redfish 在可於 <https://www.dmtf.org/standards/redfish> 取得的《Redfish Specification》
（文件 ID DSP0266）的 "Security Detail" 章節中提供安全性注意事項。

#### webui-vue 網頁應用程式

適用於網頁應用程式的一般注意事項，例如 [OWASP Web Application Security Guidance][]
所提供的內容，適用於 OpenBMC。webui-vue 使用基於使用者名稱與密碼的驗證，並使用
REST API 進行後續存取。

[owasp web application security guidance]:
  https://www.owasp.org/index.php/Web_Application_Security_Guidance

此網頁應用程式也提供介面以使用主機序列主控台、虛擬媒體與主機 KVM。

### TCP 埠 2200

透過 2200 埠上的 SSH 協定，提供對 BMC [主機序列主控台][] 的存取。

[host serial console]: https://github.com/openbmc/docs/blob/master/console.md

這使用與 22 埠相同的伺服器實作，包括相同的 TLS 機制。

主機如何保護其主控台（例如使用者名稱與密碼提示）不在本文討論範圍之內。

### TCP 與 UDP 埠 5355 - mDNS 服務探索

適用於服務探索的一般安全性注意事項亦適用。例如，此處所述：
<https://attack.mitre.org/techniques/T1046/>

### UDP 埠 427 - SLP、Avahi

適用於服務探索的一般安全性注意事項亦適用。

### UDP 埠 623 - IPMI RCMP

IPMI 的網路導向設計說明於此：
<https://github.com/openbmc/docs/blob/master/architecture/ipmi-architecture.md>，
其實作說明於此：<https://github.com/openbmc/phosphor-net-ipmid>。請注意，主機 IPMI
不在本文討論範圍之內。

適用於 IPMI 的一般安全性注意事項亦適用。例如，此處所述：
<https://www.us-cert.gov/ncas/alerts/TA13-207A>

OpenBMC 實作了 RCMP+ 與 IPMI 2.0。phosphor-user-manager 提供底層的驗證機制。

支援的 IPMI 加密算法可在程式碼中找到，例如搜尋 `isAlgorithmSupported` 函式，或從
`ipmitool` 指令取得，例如 `ipmitool channel getciphers ipmi`。

OpenBMC 支援 IPMI 的「序列線上」（serial over LAN，SOL）連線（透過
`impitool sol activate`），會共用主機序列主控台的 socket。
