# Redfish 快速參考

本文旨在提供一組用於 OpenBMC 的 [Redfish][1] 用戶端指令範例。本文使用
cURL。本文假設了若干 id，例如 ManagerId 為 "bmc"、ComputerSystemId 為
"system"。這種假設並不正確，任何使用 Redfish API 的軟體都不應該做出此類
假設。根據 Redfish 規範 DSP0266：「用戶端不應假設資源集合中各成員的
URI。」

## 查詢 Redfish Service Root

```bash
export bmc=xx.xx.xx.xx
curl -k https://${bmc}/redfish/v1
```

---

## 建立 Redfish 連線工作階段（session）

### 方法 1

```bash
export bmc=xx.xx.xx.xx
curl --insecure -H "Content-Type: application/json" -X POST -D headers.txt https://${bmc}/redfish/v1/SessionService/Sessions -d    '{"UserName":"root", "Password":"0penBmc"}'
```

會建立一個名為 headers.txt 的檔案。在該檔案中找到 `"X-Auth-Token"`。
將其儲存到環境變數中，如下所示：

```bash
export bmc_token=<token>
```

### 方法 2

```bash
export bmc=xx.xx.xx.xx
export token=`curl -k -H "Content-Type: application/json" -X POST https://${bmc}/login -d '{"username" :  "root", "password" :  "0penBmc"}' | grep token | awk '{print $2;}' | tr -d '"'`
curl -k -H "X-Auth-Token: $token" https://${bmc}/redfish/v1/...
```

注意：本文使用的方法 2。

---

## 檢視 Redfish 物件

```bash
curl -k -H "X-Auth-Token: $token" -X GET https://${bmc}/redfish/v1/Chassis
curl -k -H "X-Auth-Token: $token" -X GET https://${bmc}/redfish/v1/Managers
curl -k -H "X-Auth-Token: $token" -X GET https://${bmc}/redfish/v1/Systems
```

---

## 檢視工作階段（sessions）

```bash
curl -k -H "X-Auth-Token: $token" https://${bmc}/redfish/v1/SessionService/Sessions
```

---

## 主機電源

主機軟性關機（soft power off）：

```bash
curl -k -H "X-Auth-Token: $token" -H "Content-Type: application/json" -X POST https://${bmc}/redfish/v1/Systems/system/Actions/ComputerSystem.Reset -d '{"ResetType": "GracefulShutdown"}'
```

主機硬性關機（hard power off）：

```bash
curl -k -H "X-Auth-Token: $token" -H "Content-Type: application/json" -X POST https://${bmc}/redfish/v1/Systems/system/Actions/ComputerSystem.Reset -d '{"ResetType": "ForceOff"}'
```

主機開機（power on）：

```bash
curl -k -H "X-Auth-Token: $token" -H "Content-Type: application/json" -X POST https://${bmc}/redfish/v1/Systems/system/Actions/ComputerSystem.Reset -d '{"ResetType": "On"}'
```

重新開機主機（Reboot Host）：

```bash
curl -k -H "X-Auth-Token: $token" -H "Content-Type: application/json" -X POST https://${bmc}/redfish/v1/Systems/system/Actions/ComputerSystem.Reset -d '{"ResetType": "GracefulRestart"}'
```

---

## BMC 重新開機

```bash
curl -k -H "X-Auth-Token: $token" -H "Content-Type: application/json" -X POST https://${bmc}/redfish/v1/Managers/bmc/Actions/Manager.Reset -d '{"ResetType": "GracefulRestart"}'
```

---

## BMC 回復出廠設定（factory reset）

請謹慎操作：

```bash
curl -k -H "X-Auth-Token: $token" -H "Content-Type: application/json" -X POST https://${bmc}/redfish/v1/Managers/bmc/Actions/Manager.ResetToDefaults -d '{"ResetType": "ResetAll"}'
```

---

## 記錄項目（Log entry）

顯示記錄項目：

```bash
curl -k -H "X-Auth-Token: $token" -X GET https://${bmc}/redfish/v1/Systems/system/LogServices/EventLog/Entries
```

刪除記錄項目：

```bash
curl -k -H "X-Auth-Token: $token" -H "Content-Type: application/json" -X POST https://${bmc}/redfish/v1/Systems/system/LogServices/EventLog/Actions/LogService.Reset
```

---

## 韌體更新

韌體更新：注意 `<image file path>` 必須是一個 tarball。

```bash
uri=$(curl -k -H "X-Auth-Token: $token" https://${bmc}/redfish/v1/UpdateService | jq -r ' .HttpPushUri')

curl -k -H "X-Auth-Token: $token" -H "Content-Type: application/octet-stream" -X POST -T <image file path> https://${bmc}${uri}
```

使用多部分表單資料（multi-part form data）進行韌體更新：注意
`<apply time>` 可以是 `OnReset` 或 `Immediate`。

```bash
uri=$(curl -k -H "X-Auth-Token: $token" https://${bmc}/redfish/v1/UpdateService | jq -r ' .MultipartHttpPushUri')

curl -k -H "X-Auth-Token: $token" -H "Content-Type:multipart/form-data" -X POST -F UpdateParameters="{\"Targets\":[\"/redfish/v1/Managers/bmc\"],\"@Redfish.OperationApplyTime\":<apply time>};type=application/json" -F "UpdateFile=@<image file path>;type=application/octet-stream" https://${bmc}${uri}
```

使用 TransferProtocol 進行 TFTP 韌體更新：注意：`<image file path>`
包含 TFTP 服務的地址：
`xx.xx.xx.xx/obmc-phosphor-xxxxx-xxxxxxxxx.static.mtd.tar`

```bash
curl -k -H "X-Auth-Token: $token" -H "Content-Type: application/json" -X POST https://${bmc}/redfish/v1/UpdateService/Actions/UpdateService.SimpleUpdate -d '{"TransferProtocol":"TFTP","ImageURI":"<image file path>"}'
```

在 ImageURI 中帶有協定的 TFTP 韌體更新：

```bash
curl -k -H "X-Auth-Token: $token" -H "Content-Type: application/json" -X POST https://${bmc}/redfish/v1/UpdateService/Actions/UpdateService.SimpleUpdate -d '{"ImageURI":"tftp://<image file path>"}'
```

---

## 更新 "root" 密碼

將密碼變更為 "0penBmc1"：

```bash
curl -k -H "X-Auth-Token: $token" -H "Content-Type: application/json" -X PATCH -d '{"Password": "0penBmc1"}' https://${bmc}/redfish/v1/AccountService/Accounts/root
```

---

## BIOS 韌體開機控制

開機時進入 BIOS 設定（setup）

```bash
curl -k -H "X-Auth-Token: $token" -H "Content-Type: application/json" -X PATCH https://${bmc}/redfish/v1/Systems/system -d '{"Boot":{"BootSourceOverrideEnabled": "Continuous","BootSourceOverrideTarget": "BiosSetup"}}'
```

完整開機（Fully boot）

```bash
curl -k -H "X-Auth-Token: $token" -H "Content-Type: application/json" -X PATCH https://${bmc}/redfish/v1/Systems/system -d '{"Boot":{"BootSourceOverrideEnabled": "Disabled","BootSourceOverrideTarget": "None"}}'
```

變更 Legacy/EFI 選擇器（僅在主機以 x86 CPU 為基礎時有效）

```bash
curl -k -H "X-Auth-Token: $token" -H "Content-Type: application/json" -X PATCH https://${bmc}/redfish/v1/Systems/system -d '{"Boot":{"BootSourceOverrideEnabled": "Once","BootSourceOverrideTarget": "None","BootSourceOverrideMode": "UEFI"}}'
```

---

## 啟用 NTP

新增一個 NTP 伺服器

```bash
curl -k -H "X-Auth-Token: $token" -H "Content-Type: application/json" -X PATCH https://${bmc}/redfish/v1/Managers/bmc/NetworkProtocol -d '{"NTP":{"NTPServers":["time.nist.gov"]}}'
```

接著啟用 NTP

```bash
curl -k -H "X-Auth-Token: $token" -H "Content-Type: application/json" -X PATCH https://${bmc}/redfish/v1/Managers/bmc/NetworkProtocol -d '{"NTP":{"ProtocolEnabled": true}}'
```

---

## 停用 IPMI

```bash
curl -k -H "X-Auth-Token: $token" -H "Content-Type: application/json" -X PATCH https://${bmc}/redfish/v1/Managers/bmc/NetworkProtocol -d '{"IPMI":{"ProtocolEnabled": false}}'
```

## 管理訂閱（subscriptions）

列出已設定的訂閱：

```bash
curl -k -H "X-Auth-Token: $token" -H "Content-Type: application/json" https://${bmc}/redfish/v1/EventService/Subscriptions/
```

設定訂閱

```bash
curl -k -H "X-Auth-Token: $token" -H "Content-Type: application/json" -X POST https://${bmc}/redfish/v1/EventService/Subscriptions -d '{"Context": "My context", "DeliveryRetryPolicy": "RetryForever", "Destination": "http://yy.yy.yy.yy:8888/events", "EventFormatType": "Event", "Protocol": "Redfish", "SubscriptionType": "RedfishEvent"}"
```

將第一個訂閱的 ID 儲存到環境變數中：

```bash
export subscription_id=$(curl -k -H "X-Auth-Token: $token" -H "Content-Type: application/json" https://${bmc}/redfish/v1/EventService/Subscriptions/ |jq '.Members[0]."@odata.id"' | cut -d '/' -f 6| cut -d '"' -f 1)
```

顯示與所儲存 ID 相關聯之訂閱的資訊：

```bash
curl -k -H "X-Auth-Token: $token" -H "Content-Type: application/json" https://${bmc}/redfish/v1/EventService/Subscriptions/${subscription_id}
```

更新與所儲存 ID 相對應的訂閱：

```bash
curl -k -H "X-Auth-Token: $token" -H "Content-Type: application/json" https://${bmc}/redfish/v1/EventService/Subscriptions/${subscription_id} -X PATCH -d '{"VerifyCertificate": false}'
```

移除由所儲存 ID 識別的訂閱：

```bash
curl -k -H "X-Auth-Token: $token" -H "Content-Type: application/json" https://${bmc}/redfish/v1/EventService/Subscriptions/${subscription_id} -X DELETE
```

### 測試訂閱

使用內建於 Redfish Event Service 的功能來寄送測試事件：

```bash
curl -k -H "X-Auth-Token: $token" -H "Content-Type: application/json" https://${bmc}/redfish/v1/EventService/Actions/EventService.SubmitTestEvent -X POST -d '{"EventTimestamp":"2023-02-13T14:49:20Z","Severity":"Warning","Message":"This is a test event message 2","MessageId":"iLOResourceEvents.1.3.DrvArrLogDrvErasing","MessageArgs":["1","slot 3"],"OriginOfCondition":"/redfish/v1/Systems/1/Storage"}'
```

寄送一個測試用的 DBus 事件，它應該會被 bmcweb 取得並將事件寄給訂閱者
（此指令應在 BMC 上執行）：

```bash
busctl call xyz.openbmc_project.Logging \
/xyz/openbmc_project/logging \
xyz.openbmc_project.Logging.Create \
Create 'ssa{ss}' \
OpenBMC.0.1.PowerButtonPressed \
xyz.openbmc_project.Logging.Entry.Level.Error 0
```

[1]: https://www.dmtf.org/standards/redfish
