# OpenBMC 功能

透過下列連結進一步了解 OpenBMC 的功能：

- [BMCWeb][] HTTP/網頁伺服器
- [WebUI Vue][] 網頁應用程式
- REST 管理：[BMCWeb Redfish][]、[Phosphor REST APIs][]，其中包含 [主機管理 REST
  APIs][host management rest apis]
- [D-Bus 介面][d-bus interfaces] 說明內部介面
- [D-Bus Object Mapper][]
- [遠端 KVM][remote kvm]
- [IPMI in band][] 與 [IPMI out of band][]
- 完整符合 IPMI 2.0 規範並支援 DCMI
- 基於 SSH 的 SOL：[如何使用][sol how to use]
- 電源與散熱管理：[Phosphor Fan Control][]
- [記錄][phosphor logging] 與 [Callouts][logging callouts]
- 透過 `systemd-networkd` 提供 Zeroconf 探索
- [感測器][sensors]
- 庫存管理：[Entity manager][]、[Phosphor inventory manager][] 及其 [MSL
  應用程式][msl application]
- [LED][leds]：另請參閱 [LED 群組][led groups]
- 主機看門狗（Watchdog）：[Phosphor Watchdog 實作][phosphor watchdog
  implementation]
- [電源狀態管理][power state management] 與 [機箱電源控制][chassis power
  control]
- [網路管理][network management]
- [出廠重設][factory reset]
- [使用者管理][phosphor user management]
- 時間（日期時間時鐘）管理
- [憑證管理][certificate management]：[Phosphor Certificate Manager][]
- [模擬][simulation]（透過 QEMU）
- [韌體更新支援][firmware update support]
- [自動化測試][automated testing]
- LDAP
- 遠端 syslog

## OpenPOWER 功能

- [POWER OCC 支援][power occ implementation]（On Chip Controller，晶片內控制器）
- [硬體診斷][hardware diagnostics]：用於 POWER 系統的致命硬體錯誤。

[automated testing]:
  https://github.com/openbmc/openbmc-test-automation/blob/master/README.md
[bmcweb]: https://github.com/openbmc/bmcweb/blob/master/README.md
[bmcweb redfish]:
  https://github.com/openbmc/bmcweb/blob/master/DEVELOPING.md#redfish
[certificate management]:
  https://github.com/openbmc/phosphor-dbus-interfaces/tree/master/yaml/xyz/openbmc_project/Certs/README.md
[chassis power control]:
  https://github.com/openbmc/phosphor-dbus-interfaces/blob/master/yaml/xyz/openbmc_project/Chassis/README.md
[d-bus interfaces]:
  https://github.com/openbmc/phosphor-dbus-interfaces/blob/master/README.md
[d-bus object mapper]:
  https://github.com/openbmc/docs/blob/master/architecture/object-mapper.md
[entity manager]:
  https://github.com/openbmc/entity-manager/blob/master/README.md
[factory reset]:
  https://github.com/openbmc/phosphor-dbus-interfaces/tree/master/yaml/xyz/openbmc_project/Common/FactoryReset/README.md
[firmware update support]:
  https://github.com/openbmc/docs/blob/master/architecture/code-update/code-update.md
[hardware diagnostics]:
  https://github.com/openbmc/openpower-hw-diags/blob/master/README.md
[host management rest apis]:
  https://github.com/openbmc/docs/blob/master/host-management.md
[ipmi in band]:
  https://github.com/openbmc/docs/blob/master/architecture/ipmi-architecture.md
[ipmi out of band]: https://github.com/openbmc/ipmitool/blob/master/README
[led groups]:
  https://github.com/openbmc/phosphor-dbus-interfaces/blob/master/yaml/xyz/openbmc_project/Led/README.md
[leds]:
  https://github.com/openbmc/docs/blob/master/architecture/LED-architecture.md
[logging callouts]:
  https://github.com/openbmc/phosphor-dbus-interfaces/tree/master/yaml/xyz/openbmc_project/Common/Callout/README.md
[msl application]:
  https://github.com/openbmc/phosphor-dbus-monitor/blob/master/mslverify/README.md
[network management]:
  https://github.com/openbmc/phosphor-dbus-interfaces/blob/master/yaml/xyz/openbmc_project/Network/README.md
[phosphor certificate manager]:
  https://github.com/openbmc/phosphor-certificate-manager/blob/master/README.md
[phosphor fan control]:
  https://github.com/openbmc/phosphor-fan-presence/blob/master/README.md
[phosphor inventory manager]:
  https://github.com/openbmc/phosphor-inventory-manager/blob/master/README.md
[phosphor logging]:
  https://github.com/openbmc/phosphor-logging/blob/master/README.md
[phosphor rest apis]: https://github.com/openbmc/docs/blob/master/rest-api.md
[phosphor user management]:
  https://github.com/openbmc/docs/blob/master/architecture/user-management.md
[phosphor watchdog implementation]: https://github.com/openbmc/phosphor-watchdog
[power occ implementation]: https://github.com/openbmc/openpower-occ-control
[remote kvm]: https://github.com/openbmc/obmc-ikvm/blob/master/README.md
[sensors]:
  https://github.com/openbmc/docs/blob/master/architecture/sensor-architecture.md
[simulation]:
  https://github.com/openbmc/docs/blob/master/development/dev-environment.md
[power state management]:
  https://github.com/openbmc/phosphor-dbus-interfaces/blob/master/yaml/xyz/openbmc_project/State/README.md
[sol how to use]: https://github.com/openbmc/docs/blob/master/console.md
[webui vue]: https://github.com/openbmc/webui-vue/blob/master/README.md
