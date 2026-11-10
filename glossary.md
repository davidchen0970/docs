# 術語表

準則：

- 收錄 OpenBMC 專案、OpenBMC 參考平台（例如 OpenPower 或 ASpeed）、BMC 或平台管理
  專有的術語。
- 收錄需要釐清歧義的術語。例如，"image" 可能指視覺影像（例如 JPEG），也可能指韌體
  映像檔（例如 UBIFS）。
- 縮寫詞與術語同等對待。

BMC - Baseboard management controller（主機板管理控制器）。一種設計用於讓主機（通常
是電腦系統）可透過帶外（out of band）方式進行遠端管理的裝置。

D-Bus - 提供 OpenBMC 系統內程序間通訊的主要機制。OpenBMC 的 D-Bus API 文件位於
`https://github.com/openbmc/phosphor-dbus-interfaces`。例如，請參閱
`/xyz/openbmc_project/Led` 下的樹狀結構。

IPMI - Intelligent Platform Management Interface（智慧平台管理介面）。OpenBMC 實作
了 IPMI 規範的一個子集。

ODM - Original design manufacturer（原廠設計製造商）。在 OpenBMC 中，ODM 通常指
主機系統的製造商。

OEM - Original equipment manufacturer（原廠設備製造商）。在 OpenBMC 中，OEM 通常指
主機系統的製造商。

Phosphor - 非正式地用來指稱適用於多種系統的 OpenBMC 軟體或 API，以區別於平台或
特定廠商專有的元素。

Redfish - Distributed Management Task Force (DTMF) 的 Redfish 規範。OpenBMC 提供
Redfish REST API 以進行平台管理。

REST - Representational State Transfer（表徵狀態轉移）。OpenBMC 提供 REST API。

SDR - IPMI Sensor Data Record（感測器資料記錄）。

SEL - IMPI System Event Log（系統事件記錄）。BMC 會儲存 SEL 事件。

Server - 運算裝置，通常指由 BMC 所管理的那一台。
