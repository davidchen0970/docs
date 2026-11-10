# Redfish PCIe 資源

作者：Jason Bills，jmbills

其他貢獻者：Ed Tanous

建立日期：2019 年 4 月 3 日

## 問題描述

Redfish 提供了可用於描述系統上 PCIe 裝置與功能的資源。若能透過 OpenBMC 以 Redfish 帶外（out-of-band）方式向使用者提供這些資源，將會很有幫助。

## 背景與參考

Redfish 的 PCIe 資源位於此處：

[PCIeSlots](https://redfish.dmtf.org/schemas/PCIeSlots_v1.xml)

[PCIeDevice](https://redfish.dmtf.org/schemas/PCIeDevice_v1.xml)

[PCIeFunction](https://redfish.dmtf.org/schemas/PCIeFunction_v1.xml)

## 需求

此功能旨在滿足上述 PCIe 資源的 Redfish 需求，以便為系統管理員與操作人員提供有用的系統設定資訊。

## 擬議設計

擬議的實作將遵循 OpenBMC 中使用的標準 D-Bus 生產者─消費者模型。生產者將提供從硬體讀取所需的 PCIe 值。消費者則會擷取並解析 D-Bus 資料，以提供 Redfish 的 PCIe 資源。

擬議的 D-Bus 介面可於此處找到：
<https://gerrit.openbmc.org/c/openbmc/phosphor-dbus-interfaces/+/19768>

擬議的生產者將是一個新的 D-Bus 守護進程（daemon），負責蒐集並快取 PCIe 硬體資料，以及維護 D-Bus 介面與屬性。實際用於蒐集 PCIe 硬體資料的硬體機制會有所不同。

舉例來說，在 BMC 能存取主機 PCI 設定空間（configuration space）的系統上，它可以直接讀取所需的暫存器。在無法存取主機 PCI 設定空間的系統上，則可由 BIOS 或 OS 之類的實體蒐集所需資料，並透過 IPMI 等方式傳送給 PCIe 守護進程。

直接讀取硬體時，PCIe 守護進程必須能察覺電源狀態的變更以及任何 BIOS 的時序需求，以便檢查硬體變更、更新其快取，並對 D-Bus 屬性做出必要的變更。這樣一來，即使系統已關機，使用者仍能取得截至上次系統啟動為止最新的 PCIe 資源資料。

bmcweb 將作為消費者。它負責從 D-Bus 屬性擷取 Redfish 的 PCIe 資源資料，並提供給使用者。

## 已考慮的替代方案

無。

## 影響

可能會對硬體掃描與 D-Bus 更新造成效能影響。實作硬體掃描的部分應使用快取硬體設定等機制，以盡可能縮短掃描時間並減少對 D-Bus 屬性的更新。

## 測試

可使用 Redfish Service Validator 進行測試。
