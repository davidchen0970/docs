# 透過 Avahi 探索服務處理器

作者：Ratan Gupta

其他貢獻者：Asmitha KR

建立日期：2019-07-12

## 背景與參考資料

1. <https://www.engineersgarage.com/Articles/IoT-Service-Discovery-Protocols>
2. <http://www.zeroconf.org/ZeroconfAndUPnP.html> ：mDns 與 UPnP 的比較。
3. <https://www.avahi.org/>
4. <https://github.com/lathiat/avahi/blob/master/avahi-daemon/example.service>

Apple 的 Bonjour 使用 mDNS 與 DNS-SD。Linux 的 Avahi 使用 IPv4LL、mDNS 與
DNS-SD。Linux 的 systemd 透過 DNS、mDNS 與 LMMNR 來解析網路上的主機名稱。

## 問題描述

在一個擁有數千台系統的網路中，管理主控台應能夠探索到伺服器。我們有一項需求，
是管理主控台想要探索出特定廠商的伺服器。

例如：管理主控台想要探索出製造商 XYZ 的低階伺服器。

## 建議設計

目前 OpenBMC 中使用 Avahi 作為服務探索機制。Avahi 是一個透過 mDNS/DNS-SD
通訊協定套件，在區域網路中提供服務探索的系統。BMC 使用 Avahi 服務將主機名稱
與介面詳細資訊發佈到網路上。

Avahi 所發佈的服務包含多種欄位，例如服務名稱、類型、連接埠、主機名稱、位址、
連接埠，以及一筆文字記錄（text record）。為解決上述問題，我們提出一個解決方案，
將特定廠商的資訊納入 Avahi 服務檔案的文字記錄欄位中。

為此，目前 OpenBMC 已具備一套基礎架構，可透過針對各服務的 systemd 服務專用
bb 檔案來傳遞服務相關資料。依據是否啟用對應的發行版功能（distro feature，即
Avahi），它會以給定的資料產生 Avahi 服務檔案。我們將強化這套基礎架構，在
avahi 服務檔案中加入特定廠商的資訊（置於 txt-record 之下）。

以下的 commit 實作了這個行為。

- <https://gerrit.openbmc.org/c/openbmc/meta-phosphor/+/22950>
- <https://gerrit.openbmc.org/c/openbmc/meta-ibm/+/22951>

作為 avahi 探索回應的一部分，客戶端會收到以下內容：

1. 主機名稱（Hostname）
2. IP 位址
3. 服務連接埠（Service Port）
4. 服務類型（Service Type）
5. 額外資料（Additional data）

### 如何透過 avahi 進行探索

可以使用以下指令來探索網路上的 BMC 系統。
`avahi-browse -rt <service type>`，例如，要探索服務類型
\_obmc_rest.\_tcp：avahi-browse -rt \_obmc_rest.\_tcp。上述指令的輸出如下
avahi-browse -rt \_obmc_rest.\_tcp

- eth0 IPv6 obmc_rest \_obmc_rest.\_tcp local
- eth0 IPv4 obmc_rest \_obmc_rest.\_tcp local = eth0 IPv6 obmc_rest
  \_obmc_rest.\_tcp local hostname =
  [witherspoon-cdcb785972fb41f082c7ca747e287fa6.local] address =
  [fe80::72e2:84ff:fe14:2390] port = [443] txt = ["Manufacturer=XYZ"] txt =
  ["Type=abc"] = eth0 IPv4 obmc_rest \_obmc_rest.\_tcp local hostname =
  [witherspoon-cdcb785972fb41f082c7ca747e287fa6.local] address = [9.5.180.233]
  port = [443] txt = ["Manufacturer=XYZ"] txt = ["Type=abc"]

## 已考慮的替代方案

openSLP

## 影響

無。

## 測試

路徑 /etc/avahi/services 包含 avahi 在啟動時發佈的所有服務。
