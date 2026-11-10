# BMC 上的憑證撤銷清單

Author: Nan Zhou <nanzhoumails@gmail.com>

Created: 02/25/2022

## 問題描述

本設計旨在為 OpenBMC 新增憑證撤銷清單（certificate revocation list）的管理介面。

## 背景與參考資料

憑證撤銷清單（CRL）是一份數位憑證清單，其中所列的憑證在實際到期日或指定期限之前，就已由簽發的憑證授權中心（CA）撤銷。在 Google，有一些使用案例需要 BMC 將 CRL 安裝到 Redfish 伺服器，使得持有已撤銷憑證的用戶端在 TLS 交握時會被拒絕。多數應用也建議支援 CRL。

目前的 OpenBMC 憑證管理架構包含兩個主要元件。

1. [phosphor-certificate-manager](https://github.com/openbmc/phosphor-certificate-manager)
   負責憑證物件並實作管理介面；目前支援三種類型的憑證：client、server 與
   authority。

2. [BMCWeb](https://github.com/openbmc/bmcweb)：Redfish 前端，將憑證物件轉換為
   Redfish 資源。BMCWeb 同時也是這些憑證的使用者；它在其 TLS 交握中使用憑證。

DMTF 在 Redfish 規範中尚未支援 CRL，相關新增工作仍在進行中。請參閱
[這個討論](https://redfishforum.com/thread/618/resource-certificate-revocation-list?page=1&scrollTo=2173)。
Google 不打算使用 Redfish 介面來管理憑證與 CRL。取而代之的是，Google 有一個專用的憑證安裝常駐程式（daemon），而這個常駐程式透過 DBus API 與 OpenBMC 憑證管理架構互動。

## 需求

OpenBMC 支援 CRL 的管理介面：

1. 用戶端應能透過 DBus API 安裝/刪除/替換 CRL
2. 每當 CRL 變更時，憑證管理系統應通知仍使用舊 CRL 的使用者，以新提供的 CRL 重新整理
3. 其他常駐程式，例如 BMCWeb，應以與現有 authority/server/client 憑證相同的方式使用 CRL，也就是透過在編譯時決定的檔案路徑或目錄。

## 提議設計

### phosphor-dbus-interfaces

我們提議在
[Certs](https://github.com/openbmc/phosphor-dbus-interfaces/tree/master/yaml/xyz/openbmc_project/Certs)
中新增一個名為 `CRL` 的新介面。

由於目前尚無可用的 Redfish 規範，我們提議此介面唯一的屬性為 `CRLString`，其中包含 PEM 編碼的 CRL。未來可依需要新增更多屬性。

### phosphor-certificate-manager

我們提議在現有的三種 Manager 之外，新增一種新的憑證管理員（CRL-manager）。

CRL-manager 將實作下列通用介面：

1. [InstallAll](https://github.com/openbmc/phosphor-dbus-interfaces/blob/master/yaml/xyz/openbmc_project/Certs/InstallAll.interface.yaml)：
   安裝多個 CRL 並通知使用者。此通知流程即 phosphor-certificate-manager
   用來告知使用者重新載入新安裝憑證的既有行為。

2. [ReplaceAll](https://github.com/openbmc/phosphor-dbus-interfaces/blob/master/yaml/xyz/openbmc_project/Certs/ReplaceAll.interface.yaml)：
   以多個新的 CRL 取代所有現有的 CRL 並通知使用者

3. [DeleteAll](https://github.com/openbmc/phosphor-dbus-interfaces/blob/master/yaml/xyz/openbmc_project/Collection/DeleteAll.interface.yaml)：
   刪除所有現有的 CRL 並通知使用者

### BMCWeb

我們提議將 CRL 引入 BMCWeb 的 SSL Context。每當 BMCWeb 重新載入時，它不只重新整理 authority 與 server 憑證，也會重新整理 CRL。範例程式碼可在許多開源專案中找到，例如這段
[程式片段](https://github.com/Icinga/icinga2/blob/master/lib/base/tlsutility.cpp#L338)。

## 已考慮的替代方案

我們可以將整個 CRL 清單建模為單一物件，但這樣與現有的 authorities 清單設計不一致。

## 影響

1. 新增的 DBus 介面
2. 更完整的安全性支援

## 測試

在 phosphor-certificate-manager 中新增單元測試。

手動整合測試：安裝 CRL 並驗證用戶端已撤銷的憑證會被拒絕。
