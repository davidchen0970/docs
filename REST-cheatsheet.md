# OpenBMC REST 速查表

本文件旨在提供一組用於 OpenBMC 的 REST 用戶端指令。OpenBMC REST 在 bmcweb 中
預設為停用。更多資訊請參見
<https://github.com/openbmc/bmcweb/commit/47c9e106e0057dd70133d50e928e48cbc68e709a>。
OpenBMC REST 先前提供的大部分功能都可以在 Redfish 中找到。

## 使用 CURL 指令

### 關於認證的說明

原始的 REST 伺服器來自 phosphor-rest-server 資料庫（repository），使用由
curl cookie jar 檔案處理的認證。bmcweb REST 伺服器對於 GET 等唯讀 REST
方法可以使用相同的 cookie jar 檔案，但對於非唯讀方法，則需要在 URL 中傳入
認證權杖（authentication token）或使用者名稱與密碼。

自 2.7 OpenBMC 版本（2019 年 8 月）起，bmcweb 成為預設的 REST 伺服器。
phosphor-rest-server 資料庫已於 2022 年 10 月封存。

### 建立 REST 工作階段連線

- 僅對 phosphor-rest 伺服器使用 cookie jar 檔案：

```bash
 export bmc=xx.xx.xx.xx
 curl -c cjar -b cjar -k -H "Content-Type: application/json" -X POST https://${bmc}/login -d "{\"username\": \"root\", \"password\": \"0penBmc\" }"
```

- 若將使用者名稱/密碼作為 URL 的一部分傳入，則不需要單獨的登入呼叫。URL
  格式為：

```bash
  <username>:<password>@<hostname>/<path>...
```

例如：

```bash
 export bmc=xx.xx.xx.xx
 curl -k -X GET https://root:0penBmc@${bmc}/xyz/openbmc_project/list
```

- 基於權杖的認證。

```bash
 export bmc=xx.xx.xx.xx
 export token=`curl -k -H "Content-Type: application/json" -X POST https://${bmc}/login -d '{"username" :  "root", "password" :  "0penBmc"}' | grep token | awk '{print $2;}' | tr -d '"'`
 curl -k -H "X-Auth-Token: $token" https://${bmc}/xyz/openbmc_project/...
```

建議使用第三種方法。

### 指令

注意：為了讓下方的語法在上述 phosphor-rest 與 bmcweb 實作之間保持一致，
此處假設若使用 bmcweb，則採用上述的「基於權杖」登入方法：

```bash
export bmc=xx.xx.xx.xx
export token=`curl -k -H "Content-Type: application/json" -X POST https://${bmc}/login -d '{"username" :  "root", "password" :  "0penBmc"}' | grep token | awk '{print $2;}' | tr -d '"'`
curl -k -H "X-Auth-Token: $token" https://$bmc/xyz/openbmc_project/...
```

- 列出與列舉：

```bash
 curl -k -H "X-Auth-Token: $token" https://${bmc}/xyz/openbmc_project/list
 curl -k -H "X-Auth-Token: $token" https://${bmc}/xyz/openbmc_project/enumerate
```

- 列出子物件：

```bash
 curl -k -H "X-Auth-Token: $token" https://${bmc}/xyz/openbmc_project/
 curl -k -H "X-Auth-Token: $token" https://${bmc}/xyz/openbmc_project/state/
```

- 主機軟性關機（soft power off）：

```bash
 curl -k -H "X-Auth-Token: $token" -H "Content-Type: application/json" -d '{"data": "xyz.openbmc_project.State.Host.Transition.Off"}' -X PUT https://${bmc}/xyz/openbmc_project/state/host0/attr/RequestedHostTransition
```

- 主機硬性關機（hard power off）：

```bash
 curl -k -H "X-Auth-Token: $token" -H "Content-Type: application/json" -X PUT -d '{"data":"xyz.openbmc_project.State.Chassis.Transition.Off"}' https://${bmc}//xyz/openbmc_project/state/chassis0/attr/RequestedPowerTransition
```

- 主機開機：

```bash
 curl -k -H "X-Auth-Token: $token" -H "Content-Type: application/json" -d '{"data": "xyz.openbmc_project.State.Host.Transition.On"}' -X PUT https://${bmc}/xyz/openbmc_project/state/host0/attr/RequestedHostTransition
```

- 重新啟動主機：

```bash
 curl -k -H "X-Auth-Token: $token" -H "Content-Type: application/json" -X PUT -d '{"data":"xyz.openbmc_project.State.Host.Transition.Reboot"}' https://${bmc}/xyz/openbmc_project/state/host0/attr/RequestedHostTransition
```

- 重新啟動 BMC：

```bash
 curl -k -H "X-Auth-Token: $token" -H "Content-Type: application/json" -X PUT -d '{"data":"xyz.openbmc_project.State.BMC.Transition.Reboot"}' https://${bmc}/xyz/openbmc_project/state/bmc0/attr/RequestedBMCTransition
```

- 顯示紀錄（logging）項目：

```bash
 curl -k -H "X-Auth-Token: $token" -H "Content-Type: application/json" -X GET https://${bmc}/xyz/openbmc_project/logging/entry/enumerate
```

- 刪除紀錄項目：

```bash
 curl -k -H "X-Auth-Token: $token" -H "Content-Type: application/json" -X DELETE https://${bmc}/xyz/openbmc_project/logging/entry/<entry_id>
 curl -k -H "X-Auth-Token: $token" -H 'Content-Type: application/json' -X POST -d '{"data":[]}' https://${bmc}/xyz/openbmc_project/logging/action/DeleteAll
```

- 刪除轉傾印（dump）項目：

```bash
 curl -k -H "X-Auth-Token: $token" -H "Content-Type: application/json" -X DELETE https://${bmc}/xyz/openbmc_project/dump/entry/<entry_id>
 curl -k -H "X-Auth-Token: $token" -H 'Content-Type: application/json' -X POST -d '{"data":[]}' https://${bmc}/xyz/openbmc_project/dump/action/DeleteAll
```

- 從系統刪除映像：
  - 刪除映像：

```bash
 curl -k -H "X-Auth-Token: $token" -H "Content-Type: application/json" -X POST -d '{"data": []}' https://${bmc}/xyz/openbmc_project/software/<image id>/action/Delete
```

- 刪除所有未執行中的映像：

```bash
 curl -k -H "X-Auth-Token: $token" -H "Content-Type: application/json" -X POST -d '{"data": []}' https://${bmc}/xyz/openbmc_project/software/action/DeleteAll
```

- 清除 gard 紀錄：

```bash
 curl -k -H "X-Auth-Token: $token" -H "Content-Type: application/json" -X POST -d '{"data":[]}' https://${bmc}/org/open_power/control/gard/action/Reset
```

- 控制開機來源覆寫（boot source override）：
  - 讀取目前的開機來源覆寫設定：

```bash
 curl -k -H "X-Auth-Token: $token"  https://${bmc}/xyz/openbmc_project/control/host0/boot/enumerate
```

- 設定開機來源：

```bash
 curl -k -H "X-Auth-Token: $token" -H "Content-Type: application/json" -X PUT https://${bmc}/xyz/openbmc_project/control/host0/boot/attr/BootSource -d '{"data": "xyz.openbmc_project.Control.Boot.Source.Sources.Default"}'
```

- 設定開機模式：

```bash
 curl -k -H "X-Auth-Token: $token" -H "Content-Type: application/json" -X PUT https://${bmc}/xyz/openbmc_project/control/host0/boot/attr/BootMode -d '{"data": "xyz.openbmc_project.Control.Boot.Mode.Modes.Regular"}'
```

- 設定開機類型（僅適用於以 x86 CPU 為基礎的主機）：

```bash
 curl -k -H "X-Auth-Token: $token" -H "Content-Type: application/json" -X PUT https://${bmc}/xyz/openbmc_project/control/host0/boot/attr/BootType -d '{"data": "xyz.openbmc_project.Control.Boot.Type.Types.EFI"}'
```

- 將開機來源覆寫設為永久性：

```bash
 curl -k -H "X-Auth-Token: $token" -H "Content-Type: application/json" -X PUT https://${bmc}/xyz/openbmc_project/control/host0/boot/one_time/attr/Enabled -d '{"data": "false"}'
```

- 啟用開機來源覆寫：

```bash
 curl -k -H "X-Auth-Token: $token" -H "Content-Type: application/json" -X PUT https://${bmc}/xyz/openbmc_project/control/host0/boot/attr/Enabled -d '{"data": "true"}'
```

- 設定 NTP 與名稱伺服器（Nameserver）：

  使用公開伺服器的範例。
  - NTP 伺服器：

```bash
 curl -k -H "X-Auth-Token: $token" -H "Content-Type: application/json" -X PUT -d '{"data": ["pool.ntp.org"] }' https://${bmc}/xyz/openbmc_project/network/eth0/attr/NTPServers
```

- 名稱伺服器：

```bash
 curl -k -H "X-Auth-Token: $token" -H "Content-Type: application/json" -X PUT -d '{"data": ["time.google.com"] }' https://${bmc}/xyz/openbmc_project/network/eth0/attr/Nameservers
```

- 設定時間擁有者（time ownership）與時間同步方法：

  關於時間設定的說明請參見這裡：
  <https://github.com/openbmc/phosphor-time-manager>

  注意：自 OpenBMC 2.6（使用 systemd v239）起，systemd 的 timedated 引入了一種
  新行為：它在設定時間時會檢查 NTP 服務的狀態，而不是檢查 NTP 設定：

  -當 NTP 伺服器設為停用，且 NTP 服務正在停止但尚未停止時，設定時間會發生錯誤。

  在 OpenBMC 2.4 之前（使用 systemd v236），上述情況一定會成功。這導致了
  [openbmc/openbmc#3459](https://github.com/openbmc/openbmc/issues/3459)，
  相關的測試案例也已更新以配合此行為變更。
  - 讀取：

```bash
 curl -k -H "X-Auth-Token: $token" -X GET https://${bmc}/xyz/openbmc_project/time/owner/attr/TimeOwner
 curl -k -H "X-Auth-Token: $token" -X GET https://${bmc}/xyz/openbmc_project/time/sync_method/attr/TimeSyncMethod
```

- 寫入：

時間擁有者：

```bash
 curl -k -H "X-Auth-Token: $token" -H "Content-Type: application/json" -X  PUT -d '{"data": "xyz.openbmc_project.Time.Owner.Owners.BMC" }' https://${bmc}/xyz/openbmc_project/time/owner/attr/TimeOwner
 curl -k -H "X-Auth-Token: $token" -H "Content-Type: application/json" -X  PUT -d '{"data": "xyz.openbmc_project.Time.Owner.Owners.Host" }' https://${bmc}/xyz/openbmc_project/time/owner/attr/TimeOwner
 curl -k -H "X-Auth-Token: $token" -H "Content-Type: application/json" -X  PUT -d '{"data": "xyz.openbmc_project.Time.Owner.Owners.Split" }' https://${bmc}/xyz/openbmc_project/time/owner/attr/TimeOwner
 curl -k -H "X-Auth-Token: $token" -H "Content-Type: application/json" -X  PUT -d '{"data": "xyz.openbmc_project.Time.Owner.Owners.Both" }' https://${bmc}/xyz/openbmc_project/time/owner/attr/TimeOwner
```

時間同步方法：

```bash
 curl -k -H "X-Auth-Token: $token" -H "Content-Type: application/json" -X  PUT -d '{"data": "xyz.openbmc_project.Time.Synchronization.Method.NTP" }' https://${bmc}/xyz/openbmc_project/time/sync_method/attr/TimeSyncMethod
 curl -k -H "X-Auth-Token: $token" -H "Content-Type: application/json" -X  PUT -d '{"data": "xyz.openbmc_project.Time.Synchronization.Method.Manual" }' https://${bmc}/xyz/openbmc_project/time/sync_method/attr/TimeSyncMethod
```

- 電源供應器冗餘（Power Supply Redundancy）：
  - 讀取：

```bash
 curl -k -H "X-Auth-Token: $token" -X GET https://${bmc}/xyz/openbmc_project/control/power_supply_redundancy
```

- 寫入（啟用/停用）：

```bash
 curl -k -H "X-Auth-Token: $token" -H "Content-Type: application/json" -X PUT https://${bmc}/xyz/openbmc_project/control/power_supply_redundancy/attr/PowerSupplyRedundancyEnabled -d '{"data": 1}'
 curl -k -H "X-Auth-Token: $token" -H "Content-Type: application/json" -X PUT https://${bmc}/xyz/openbmc_project/control/power_supply_redundancy/attr/PowerSupplyRedundancyEnabled -d '{"data": 0}'
```

- 出廠重置（Factory Reset）：
  - 出廠重置主機與 BMC 軟體：

```bash
 curl -k -H "X-Auth-Token: $token" -H 'Content-Type: application/json' -X POST -d '{"data":[]}' https://${bmc}/xyz/openbmc_project/software/action/Reset
```

- 出廠重置網路設定：

```bash
 curl -k -H "X-Auth-Token: $token" -H 'Content-Type: application/json' -X POST -d '{"data":[]}' https://${bmc}/xyz/openbmc_project/network/action/Reset
```

- 啟用維修模式（field mode）：

```bash
 curl -k -H "X-Auth-Token: $token" -H 'Content-Type: application/json' -X PUT -d '{"data":1}' https://${bmc}/xyz/openbmc_project/software/attr/FieldModeEnabled
```

然後重新啟動 BMC。
