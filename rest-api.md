# OpenBMC REST API

OpenBMC REST API 在 bmcweb 中預設是停用的。大部分功能都可以在 Redfish 上取得。
本文件提供 OpenBMC REST 介面的一些基本結構與使用範例。

REST 介面的 schema 直接由 OpenBMC 的 D-Bus 結構定義。因此，物件、屬性與方法
都與 D-Bus schema 中的對應項目密切對應。

關於 HTTP 動詞及其與 RESTful API 之間關係的簡要說明，請參考
<https://restapitutorial.com/introduction/httpmethods>。

## 驗證

驗證的詳細資訊請參考
[REST-cheatsheet](https://github.com/openbmc/docs/blob/master/REST-cheatsheet.md#establish-rest-connection-session)。

本教學使用 token based 驗證方式：

```bash
export bmc=xx.xx.xx.xx
export token=`curl -k -H "Content-Type: application/json" -X POST https://${bmc}/login -d '{"username" : "root", "password" : "0penBmc"}' | grep token | awk '{print $2;}' | tr -d '"'`
curl -k -H "X-Auth-Token: $token" https://${bmc}/xyz/openbmc_project/...
```

## HTTP GET 操作與 URL 結構

OpenBMC REST 介面的 URL 結構有幾項慣例，分別是：

- 若要查詢某個物件的屬性，請對該物件名稱發出 GET 請求，末尾不加斜線。例如：

```bash
curl -k -H "X-Auth-Token: $token" https://${bmc}/xyz/openbmc_project/inventory/system
```

```json
{
  "data": {
    "AssetTag": "",
    "BuildDate": "",
    "Cached": true,
    "FieldReplaceable": true,
    "Manufacturer": "",
    "Model": "",
    "PartNumber": "",
    "Present": true,
    "PrettyName": "",
    "SerialNumber": ""
  },
  "message": "200 OK",
  "status": "ok"
}
```

- 若要查詢單一屬性，請使用 `attr/<name>` 路徑。以上面的 `system` 物件為例，
  我們可以只查詢 `Name` 值：

```bash
curl -k -H "X-Auth-Token: $token" https://${bmc}/xyz/openbmc_project/inventory/system/attr/Cached
```

```json
{
  "data": true,
  "message": "200 OK",
  "status": "ok"
}
```

- 當路徑結尾帶有斜線時，回應會列出該 URL 的子物件。例如，使用與上面相同的
  物件路徑，但加上一個斜線：

```bash
curl -k -H "X-Auth-Token: $token" https://${bmc}/xyz/openbmc_project/
```

```json
{
  "data": [
    "/xyz/openbmc_project/Chassis",
    "/xyz/openbmc_project/Hiomapd",
    "/xyz/openbmc_project/Ipmi",
    "/xyz/openbmc_project/certs",
    "/xyz/openbmc_project/console",
    "/xyz/openbmc_project/control",
    "/xyz/openbmc_project/dump",
    "/xyz/openbmc_project/events",
    "/xyz/openbmc_project/inventory",
    "/xyz/openbmc_project/ipmi",
    "/xyz/openbmc_project/led",
    "/xyz/openbmc_project/logging",
    "/xyz/openbmc_project/network",
    "/xyz/openbmc_project/object_mapper",
    "/xyz/openbmc_project/sensors",
    "/xyz/openbmc_project/software",
    "/xyz/openbmc_project/state",
    "/xyz/openbmc_project/time",
    "/xyz/openbmc_project/user"
  ],
  "message": "200 OK",
  "status": "ok"
}
```

這顯示 `openbmc_project/` 物件共有 19 個子物件：`dump`、`software`、`control`、
`network`、`logging` 等等。將它與基礎 REST URL（亦即 `http://${bmc}/`）搭配使用，
即可探索整個階層中的所有物件。

- 對相同查詢加上 `/list`，會*遞迴地*列出子物件。

```bash
curl -k -H "X-Auth-Token: $token" https://${bmc}/xyz/openbmc_project/network/list
```

```json
{
  "data": [
    "/xyz/openbmc_project/network/config",
    "/xyz/openbmc_project/network/config/dhcp",
    "/xyz/openbmc_project/network/eth0",
    "/xyz/openbmc_project/network/eth0/ipv4",
    "/xyz/openbmc_project/network/eth0/ipv4/3b9faa36",
    "/xyz/openbmc_project/network/eth0/ipv6",
    "/xyz/openbmc_project/network/eth0/ipv6/ff81b6d6",
    "/xyz/openbmc_project/network/eth1",
    "/xyz/openbmc_project/network/eth1/ipv4",
    "/xyz/openbmc_project/network/eth1/ipv4/3b9faa36",
    "/xyz/openbmc_project/network/eth1/ipv4/66e63348",
    "/xyz/openbmc_project/network/eth1/ipv6",
    "/xyz/openbmc_project/network/eth1/ipv6/ff81b6d6",
    "/xyz/openbmc_project/network/host0",
    "/xyz/openbmc_project/network/host0/intf",
    "/xyz/openbmc_project/network/host0/intf/addr",
    "/xyz/openbmc_project/network/sit0",
    "/xyz/openbmc_project/network/snmp",
    "/xyz/openbmc_project/network/snmp/manager"
  ],
  "message": "200 OK",
  "status": "ok"
}
```

- 改用 `/enumerate` 而非 `/list`，還會一併包含所列物件的屬性。

```bash
curl -k -H "X-Auth-Token: $token" https://${bmc}/xyz/openbmc_project/time/enumerate
```

```json
{
  "data": {
    "/xyz/openbmc_project/time/bmc": {
      "Elapsed": 1563209492098739
    },
    "/xyz/openbmc_project/time/host": {
      "Elapsed": 1563209492101678
    },
    "/xyz/openbmc_project/time/owner": {
      "TimeOwner": "xyz.openbmc_project.Time.Owner.Owners.BMC"
    },
    "/xyz/openbmc_project/time/sync_method": {
      "TimeSyncMethod": "xyz.openbmc_project.Time.Synchronization.Method.NTP"
    }
  },
  "message": "200 OK",
  "status": "ok"
}
```

## HTTP PUT 操作

PUT 操作用於更新現有的資源（物件或屬性），或在用戶端已知放置位置時建立新的
資源。這些操作需要 json 格式的 payload。若要取得這類請求的範例，請參考：

```bash
curl -k -H "X-Auth-Token: $token" https://${bmc}/xyz/openbmc_project/state/host0 > host.json
cat host.json
```

```json
{
  "data": {
    "AttemptsLeft": 3,
    "BootProgress": "xyz.openbmc_project.State.Boot.Progress.ProgressStages.Unspecified",
    "CurrentHostState": "xyz.openbmc_project.State.Host.HostState.Off",
    "OperatingSystemState": "xyz.openbmc_project.State.OperatingSystem.Status.OSStatus.Inactive",
    "RequestedHostTransition": "xyz.openbmc_project.State.Host.Transition.Off"
  },
  "message": "200 OK",
  "status": "ok"
}
```

或

```bash
curl -k -H "X-Auth-Token: $token" https://${bmc}/xyz/openbmc_project/state/host0/attr/RequestedHostTransition > requested_host.json
cat requested_host.json
```

```json
{
  "data": "xyz.openbmc_project.State.Host.Transition.Off",
  "message": "200 OK",
  "status": "ok"
}
```

將這些結果轉換為請求送出時，請刪除 message 與 status 屬性。

要讓 curl 使用正確的 content type 標頭，請使用 -H 選項來指定我們傳送的是
JSON 資料：

```bash
curl -k -H "X-Auth-Token: $token" -H "Content-Type: application/json" -X PUT -d <json> <url>
```

對某個物件進行 PUT 操作需要完整的物件。若要進行部分更新，則有 PATCH，但目前
尚未實作。作為替代方案，個別屬性是可以 PUT 的。

例如，修改 requested_host.json 檔案後執行一次 PUT（上傳）：

```bash
$ cat requested_host.json
{"data": "xyz.openbmc_project.State.Host.Transition.Off"}
$ curl -k -H "X-Auth-Token: $token" -H "Content-Type: application/json" -X PUT -T requested_host.json https://${bmc}/xyz/openbmc_project/state/host0/attr/RequestedHostTransition
```

或者使用 -d 直接內嵌指定 json：

```bash
curl -k -H "X-Auth-Token: $token" -H "Content-Type: application/json" -X PUT -d '{"data": "xyz.openbmc_project.State.Host.Transition.On"}' https://${bmc}/xyz/openbmc_project/state/host0/attr/RequestedHostTransition
```

使用 '-d' 時請記得 json 需要加上引號。

## HTTP POST 操作

POST 操作用於呼叫方法，也在用戶端不知道放置位置時用於建立新的資源。OpenBMC
不支援透過 REST 建立新資源，因此任何嘗試建立新資源的行為都會導致 HTTP 403
(Forbidden)。這些操作同樣需要 json 格式的 payload。若要刪除記錄條目：

```bash
curl -k -H "X-Auth-Token: $token" -H 'Content-Type: application/json' -X POST -d '{"data":[]}' https://${bmc}/xyz/openbmc_project/logging/action/DeleteAll
```

若要呼叫一個不帶參數的方法（將 BMC 與主機重設為出廠設定）：

```bash
curl -k -H "X-Auth-Token: $token" -H 'Content-Type: application/json' -X POST -d '{"data":[]}' https://${bmc}/xyz/openbmc_project/software/action/Reset
```

## HTTP DELETE 操作

DELETE 操作用於移除實例。只有 D-Bus 物件（實例）可以被移除。如果底層的 D-Bus
物件實作了 `xyz.openbmc_project.Object.Delete` 介面，REST 伺服器就會呼叫它。
如果未實作 `xyz.openbmc_project.Object.Delete`，REST 伺服器會回傳 HTTP 403
(Forbidden) 錯誤。

例如，若要刪除一筆事件記錄：

顯示記錄條目：

```bash
curl -k -H "X-Auth-Token: $token" -H "Content-Type: application/json" -X GET https://${bmc}/xyz/openbmc_project/logging/entry/enumerate
```

然後刪除 ID 為 1 的事件記錄：

```bash
curl -k -H "X-Auth-Token: $token" -H "Content-Type: application/json" -X DELETE  https://${bmc}/xyz/openbmc_project/logging/entry/1
```

## 上傳映像

可以透過 REST 上傳軟體升級映像（例如升級 BMC 或主機軟體）。content-type 應設為
"application/octet-stream"。

例如，若要上傳一個映像：（`<file_to_upload>` 必須是一個 tar 封包）

```bash
curl -k -H "X-Auth-Token: $token" -H "Content-Type: application/octet-stream" -X POST -T <file_to_upload> https://${bmc}/upload/image
```

在上述範例中，BMC 上的檔名將由 REST 伺服器選取。

使用者也可以自行選擇上傳檔案在遠端的名稱：

```bash
curl -k -H "X-Auth-Token: $token" -H "Content-Type: application/octet-stream" -X PUT -T foo https://${bmc}/upload/image/bar
```

在上述範例中，檔案 foo 會在 BMC 上以名稱 bar 儲存。

此操作在成功時會回傳上傳檔案的版本 id（hash）：

```json
{
  "data": "afb92384",
  "message": "200 OK",
  "status": "ok"
}
```

或回傳錯誤訊息

```json
{
  "data": {
    "description": "Version already exists or failed to be extracted"
  },
  "message": "400 Bad Request",
  "status": "error"
}
```

關於上傳與更新軟體的更多詳細資訊，請參考：
<https://github.com/openbmc/docs/tree/master/architecture/code-update>

## 事件訂閱協定

可以訂閱 BMC 上所發生的、您有興趣的事件。BMC 上的實作為此使用 WebSockets，
讓用戶端不必採用輪詢。取而代之的是，BMC 上的 REST 伺服器可以透過 websocket
將資料推播給用戶端。BMC 可以推播與 D-Bus InterfacesAdded 及 PropertiesChanged
訊號相關的資訊。

以下說明事件訂閱協定，並附上代表用戶端端程式碼的 JS 程式碼片段範例。

a. 用戶端必須已登入 BMC。

b. 用戶端必須使用 URL `<BMC IP>/subscribe` 開啟一個安全的 websocket。

```javascript
var ws = new WebSocket("wss://<BMC IP>/subscribe");
```

c. 用戶端必須透過 websocket 傳送一個由鍵值對組成的 JSON 字典。這個字典作為
「事件過濾器」。所有的鍵都是選擇性的，因此如果不想要任何過濾，字典可以是空的。
每個鍵值對所代表的過濾器之間是 OR（或）的關係。

其中一個支援的鍵是 "paths"。其對應的值是一個 D-Bus 路徑陣列。只有從這些路徑
發出、而非從其他任何路徑發出的 InterfacesAdded 與 PropertiesChanged D-Bus 訊號，
才會被包含在 BMC 送出的事件訊息中。

另一個支援的鍵是 "interfaces"。其對應的值是一個 D-Bus 介面陣列。包含這些介面
中任一者的 InterfacesAdded 與 PropertiesChanged D-Bus 訊號訊息，會被包含在 BMC
送出的事件訊息中。

以下所有寫法都是有效的：

```javascript
var data = JSON.stringify({
  paths: ["/xyz/openbmc_project/logging", "/xyz/openbmc_project/sensors"],
  interfaces: [
    "xyz.openbmc_project.Logging.Entry",
    "xyz.openbmc_project.Sensor.Value",
  ],
});
ws.onopen = function () {
  ws.send(data);
};
```

```javascript
var data = JSON.stringify({
  paths: ["/xyz/openbmc_project/logging", "/xyz/openbmc_project/sensors"],
});
ws.onopen = function () {
  ws.send(data);
};
```

```javascript
var data = JSON.stringify({
  interfaces: [
    "xyz.openbmc_project.Logging.Entry",
    "xyz.openbmc_project.Sensor.Value",
  ],
});
ws.onopen = function () {
  ws.send(data);
};
```

```javascript
var data = JSON.stringify({});
ws.onopen = function () {
  ws.send(data);
};
```

d. 當發生 D-Bus 事件時，BMC 上的 REST 伺服器會根據用戶端提供的過濾器，透過
websocket 回應。REST 伺服器會通知 InterfacesAdded 與 PropertiesChanged 事件。
回應是一個 JSON 字典，如下所示：

InterfacesAdded

```text
"event": InterfacesAdded
"path": <string : new D-Bus path that was created>
"interfaces": <dict : a dictionary of interfaces> (similar to org.freedesktop.DBus.ObjectManager.InterfacesAdded )
```

PropertiesChanged

```text
"event": PropertiesChanged
"path": <string : D-Bus path whose property changed>
"interface": <string : D-Bus interface to which the changed property belongs>
"properties": <dict : a dictionary of properties> (similar to org.freedesktop.DBus.Properties.PropertiesChanged)
```
