# 虛擬媒體（Virtual Media，又稱遠端媒體 Remote Media）

Author: Pawel Rapkiewicz <pawel.rapkiewicz@intel.com> `pawelr`

其他貢獻者：

- Przemyslaw Czarnowski <przemyslaw.hawrylewicz.czarnowski@intel.com>
- Anna Platash <anna.platash@intel.com>

建立日期：6/4/2019

## 問題描述

虛擬媒體讓使用者可以透過 BMC 將指定的 ISO/IMG 磁碟映像檔遠端掛載到伺服器 HOST。
遠端磁碟在 HOST 上會以 USB 儲存裝置的形式呈現，並可運作於 RO（唯讀）模式或
RW（可讀寫）模式（需留意容器的限制，以及寫入保護開關）。這甚至可以用來在裸機
（bare metal）系統上安裝作業系統。本文件著重於幾種 redirection 選項，例如在瀏覽器
內掛載 ISO/IMG 映像檔，以及遠端 CIFS/HTTPS 映像檔掛載。

## 參考資料

- 虛擬媒體將使用 Network Block Device 作為主要的磁碟映像檔轉發器。
- 使用 NBDkit 透過 HTTPS/CIFS 從遠端儲存裝置提供映像檔服務。
  USBGadget 則作為將 Media Storage Device 呈現給 HOST 的方式。

## 需求

無

## 建議設計

虛擬區分為兩種運作模式，我們稱之為 Proxy 與 Legacy。

- Proxy 模式 - 直接從瀏覽器運作，使用 JavaScript/HTML5 透過 Secure WebSockets
  直接與 BMC 上由 bmcweb 託管的 HTTPS endpoint 通訊。
- Legacy 模式 - 由瀏覽器使用 Redfish 定義的 VirtualMedia schema 發起，接著
  BMC 程序會連線至初始化時所指派的對外 CIFS/HTTPS 映像檔。

兩種方法都繼承預設的 Redfish/BMCWeb 驗證與權限機制。

下方的元件圖顯示虛擬媒體的高階概觀

```ascii
+------------------+           +----------------------------------+     +-----------------------+
|Remote Device|    |           |BMC|              +------------+  |     |HOST|                  |
+-------------/    |           +---/ +--Dbus----->+VirtualMedia|  |     +----/                  |
|                  |           |     v            +------------+  |     |                       |
|  +------------+  |           |   +-+--------+                   |     |                       |
|  |WebBrowser  +<----HTTPS------->+BMCWeb    |      +---------+  |     |  +----------+         |
|  +------------+  |           |   +-+--------+      |USBGadget+<--------->+USB Device|         |
+------------------+           |     ^               +----+----+  |     |  +----------+         |
                               |     |                    ^       |     |                       |
                               |     |    +------+        v       |     |                       |
                               |     +--->+UNIX  |   +----+----+  |     |                       |
                               |          |SOCKET+<->+NBDClient|  |     |                       |
+------------------+           |     +--->+      |   +---------+  |     |                       |
|Remote Storage|   |           |     |    +------+                |     |                       |
+--------------/   |           |     |                            |     |                       |
|                  |           |     v                            |     |                       |
|   +-----------+  |           |   +-+-------+                    |     |                       |
|   |ISO/IMG    +<---CIFS/HTTPS+-->+NBDkit   |                    |     |                       |
|   +-----------+  |           |   +---------+                    |     +-----------------------+
|                  |           |                                  |
+------------------+           +----------------------------------+
```

虛擬媒體功能在兩種模式下都支援多個同時連線。

### 非同步性（Asynchronicity）

遠端裝置的掛載與卸載可能會很耗時。虛擬媒體應在 DBus 層級支援非同步，並在
Redfish API 層級選擇性支援。

DBus 的預設逾時（25 秒）與 Redfish 的預設逾時（60 秒），在某些情況下可能不足以
完成掛載與卸載。

非同步回應將於後續適當的章節中說明。

### Network Block Device（NBD）

讀者會注意到圖上多數連線都是基於 Network Block Device。根據 Sourceforge 專案說明：

> With this compiled into your kernel, Linux can use a remote server as one of
> its block devices. Every time the client computer wants to read /dev/nbd0, it
> will send a request to the server via TCP, which will reply with the data
> requested. This can be used for stations with low disk space (or even
> diskless - if you use an initrd) to borrow disk space from other computers.
> Unlike NFS, it is possible to put any file system on it. But (also unlike
> NFS), if someone has mounted NBD read/write, you must assure that no one else
> will have it mounted.
>
> -- [https://nbd.sourceforge.io/](https://nbd.sourceforge.io/)

在虛擬媒體的使用案例中，它被用來從遠端客戶端拉取資料，並呈現在非 BMC 掛載的
`/dev/nbdXX` 裝置上。接著該區塊裝置會透過 USB Gadget 提供給 Host。

### USB Gadget

Linux kernel 的一部分，若適當連線到 Host，可透過 USB "On-The-Go" 實作某些 USB
裝置類別的_模擬_。在虛擬媒體的案例中，它模擬一個連接到 HOST 的 USB 大量儲存裝置。
其來源或 redirection 是 nbd-client 建立的區塊裝置 `/dev/nbdXX`

### Proxy 模式

Proxy 模式使用瀏覽器的 JavaScript 與 WebSockets 支援來建立 JS NBD Server。
瀏覽器負責建立 HTTPS session、驗證使用者並取得指定的權限，然後透過
[RFC6455](HTTPS://tools.ietf.org/html/rfc6455) 所描述的機制將 HTTPS session
升級為 WSS。自 WSS 升級之後，JS 應用程式負責處理規格要求的所有 NBD Server
指令。

支援多個同時連線，每個連線在 HTTPS server 上開啟不同的 URI。可用的同時連線數量
定義在下一章節說明的設定檔中。

Proxy 的加密透過 HTTPS/WSS 通道支援，並直接繼承 HTTPS server 的加密機制，所有
資料交易都經過 bmcweb。

連線的初始化流程如圖所示：

```ascii
┌───────┐                  ┌──────┐       ┌────────────┐          ┌─────────┐   ┌────┐ ┌─────────┐
│Browser│                  │bmcweb│       │VirtualMedia│          │NBDClient│   │uDEV│ │USBGadget│
└───┬───┘                  └──┬───┘       └─────┬──────┘          └────┬────┘   └─┬──┘ └────┬────┘
    │ establish HTTPS session │                 │                      │          │         │
    │─────────────────────────>                 │                      │          │         │
    │                         │                 │                      │          │         │
    │   upgrade to WSS on     │                 │                      │          │         │
    │    /nbd/X endpoint      │  ╔══════════════╧════╗                 │          │         │
    │─────────────────────────>  ║* bmcweb creates  ░║                 │          │         │
    │                         │  ║  /tmp/nbd.X.sock  ║                 │          │         │
    │                         │  ║* bmcweb locks new ║                 │          │         │
    │                         │  ║  connections on   ║                 │          │         │
    │                         │  ║  endpoint /nbd/X  ║                 │          │         │
    │                         │  ╚══════════════╤════╝                 │          │         │
    │                         │    Mount from:  │                      │          │         │
    │                         │ /tmp/nbd.X.sock │                      │          │         │
    │                         │ ────────────────>                      │          │         │
    │                         │                 │                      │          │         │
    │                         │                 │ Spawn NBDClient from │          │         │
    │                         │                 │    /tmp/nbd.x.sock   │          │         │
    │                         │                 │      to /dev/nbdX   ┌┴┐         │         │
    │                         │                 │ ──────────────────> │ │         │         │
    │                         │                 │                     │ │         │         │
    │                         │                 │     Block Device    │ │         │         │
    │                         │                 │  properties changed │ │         │         │
    │                         │                 │ <───────────────────────────────│         │
    │                         │                 │                     │ │         │         │
    │                         │                 │      Configure USB mass         │         │
    │                         │                 │    storage from /dev/nbd/X      │         │
    │                         │                 │ ─────────────────────────────────────────>│
    │                         │                 │                     │ │         │         │
    │                         │    Completion   │                     | |         │         │
    │                         │      signal     │                     | |         │         │
    │                         │ <───────────────│                     │ │         │         │
    │                         │                 │                     │ │         │         │
    │                         │                 │         Data        │ │         │         │
    │<─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─>│
    │                         │                 │                     │ │         │         │
```

### Legacy 模式

Legacy 模式使用由 DMTF 定義的 VirtualMedia schema 來掛載對外的 CIFS/HTTPS
映像檔。目前實作僅支援串流（stream）掛載。在此情況下 Redfish 僅作為虛擬媒體
初始化的機制，並不負責資料傳輸。資料部分則由一個稱為 NBDkit 的獨立元件負責處理
CIFS/HTTPS 流量。

支援多個同時連線，方法是衍生（spawn）額外的 nbkit 執行個體，可用於 CIFS/HTTPS
的執行個體數量會經過設定，並在下一章節詳細說明。

加密以遠端儲存連線為基礎，並遵循 Intel 的最佳安全性實務作法，前提是遠端伺服器
支援（例如 HTTPS 需要 SSL，不支援純 HTTP；對 CIFS 而言，通訊協定版本 3.0 可啟用
加密，這將會被提供）。

流程如下圖所示：

```ascii
┌───────┐  ┌──────────┐  ┌──────┐  ┌────────────┐           ┌──────┐┌─────────┐┌────┐ ┌─────────┐
│Browser│  │CIFS/HTTPS│  │bmcweb│  │VirtualMedia│           │NBDkit││NBDClient││uDEV│ │USBGadget│
└───┬───┘  └────┬─────┘  └──┬───┘  └─────┬──────┘           └──┬───┘└────┬────┘└─┬──┘ └────┬────┘
    │establish HTTPS session│            │                     │         │       │         │
    │───────────────────────>            │                     │         │       │         │
    │           │           │            │                     │         │       │         │
    │Create new VirtualMedia│            │                     │         │       │         │
    │  mountpoint via POST  │            │                     │         │       │         │
    │───────────────────────>            │                     │         │       │         │
    │           │           │            │                     │         │       │         │
    │           │           │ Mount from │                     │         │       │         │
    │           │           │ CIFS/HTTPS │                     │         │       │         │
    │           │           │  location  │                     │         │       │         │
    │           │           │────────────>                     │         │       │         │
    │           │           │            │                     │         │       │         │
    │           │           │            │Spawn NBDKit mounting│         │       │         │
    │           │           │            │    given location   │         │       │         │
    │           │           │            │     appropriate     │         │       │         │
    │           │           │            │   /tmp/nbd.X.sock  ┌┴┐        │       │         │
    │           │           │            │ ──────────────────>│ │        │       │         │
    │           │           │            │                    │ │        │       │         │
    │           │           │            │      Spawn NBDClient from     │       │         │
    │           │           │            │ /tmp/nbd.X.sock to /dev/nbdX ┌┴┐      │         │
    │           │           │            │ ────────────────────────────>│ │      │         │
    │           │           │            │                    │ │       │ │      │         │
    │           │           │            │    Block Device properties changed    │         │
    │           │           │            │ <──────────────────────────────────────         │
    │           │           │            │                    │ │       │ │      │         │
    │           │           │            │    Configure USB mass storage from /dev/nbd/X   │
    │           │           │            │ ───────────────────────────────────────────────>│
    │           │           │            │                    │ │       │ │      │         │
    │           │           │ Completion │                    │ │       │ │      │         │
    │           │           │   signal   │                    │ │       │ │      │         │
    │           │           |<───────────│                    │ │       │ │      │         │
    │           │           │            │                    │ │       │ │      │         │
    │           │           │            │               Data │ │       │ │      │         │
    │           │ <─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─>│
```

### Redfish 支援

虛擬媒體服務將以 Redfish VirtualMedia endpoint 的形式暴露，如 DMTF 所定義。
以下是一些範例。

#### Virtual Media Collection schema

集合中的成員將根據後續章節說明的設定檔來定義。不論媒體是否已插入，都會可見。

```json
{
  "@odata.type": "#VirtualMediaCollection.VirtualMediaCollection",
  "Name": "Virtual Media Services",
  "Description": "Redfish-BMC Virtual Media Service Settings",
  "Members@odata.count": 2,
  "Members": [
    {
      "@odata.id": "/redfish/v1/Managers/BMC/VirtualMedia/ISO0"
    },
    {
      "@odata.id": "/redfish/v1/Managers/BMC/VirtualMedia/1"
    }
  ],
  "@odata.context": "/redfish/v1/$metadata#VirtualMediaCollection.VirtualMediaCollection",
  "@odata.id": "/redfish/v1/Managers/BMC/VirtualMedia"
}
```

#### Virtual Media schema

```json
{
  "@odata.type": "#VirtualMedia.v1_1_0.VirtualMedia",
  "Id": "ISO0",
  "Name": "Virtual Removable Media",
  "Actions": {
    "#VirtualMedia.InsertMedia": {
      "target": "/redfish/v1/Managers/bmc/VirtualMedia/ISO0/Actions/VirtualMedia.InsertMedia"
    },
    "#VirtualMedia.EjectMedia": {
      "target": "/redfish/v1/Managers/bmc/VirtualMedia/ISO0/Actions/VirtualMedia.EjectMedia"
    }
  },
  "MediaTypes": ["CD", "USBStick"],
  "Image": "https://192.168.0.1/Images/os.iso",
  "ImageName": "Os",
  "ConnectedVia": "URI",
  "Inserted": true,
  "WriteProtected": false,
  "@odata.context": "/redfish/v1/$metadata#VirtualMedia.VirtualMedia",
  "@odata.id": "/redfish/v1/Managers/BMC/VirtualMedia/ISO0",
  "Oem": {
    "OpenBMC": {
      "@odata.type": "#OemVirtualMedia.v1_0_0.VirtualMedia",
      "WebSocketEndpoint": "/nbd/0"
    }
  }
}
```

schema 在 Proxy 與 Legacy 模式下看起來會類似。一些主要差異如下：

| 欄位名稱             | Proxy 模式 | Legacy 模式                      | 備註                                                       |
| -------------------- | ---------- | -------------------------------- | ---------------------------------------------------------- |
| InsertMedia          | N/A        | 如 DMTF 規格所定義的 action      | 若程序耗時，Action 可回傳 Task 物件                        |
| Image                | N/A        | 映像檔位置                       |                                                            |
| ImageName            | N/A        | 映像檔名稱                       |                                                            |
| ConnectedVia         | "Applet"   | 如 DMTF 規格所定義               | 僅適用於已連線的媒體                                       |
| TransferMethod       | "Stream"   | "Stream"                         | 依設計不支援 "upload"                                      |
| TransferProtocolType | "OEM"      | 如 DMTF 規格所定義               |                                                            |

#### Virtual Media OEM Extension

Virtual Media schema 已針對 Legacy 模式進行調整，在此模式下映像檔由使用者透過
Redfish action 直接提供，且整個連線在服務與 web server 之間處理。

對於 [Proxy 模式](#proxy-mode)，nbd 資料由客戶端 web 瀏覽器提供。在有多個連線
的情況下，為了設定連線，客戶端需要關於 web server 所建立 websocket 位置的資訊。
這個值會以 OEM `WebSocketEndpoint` 屬性的形式針對每個項目暴露出來。

### 閒置逾時（Inactivity timeout）

虛擬媒體支援閒置逾時，在經過若干秒的閒置之後會中斷虛擬媒體連線。由於 nbdclient
有映像檔快取機制，kernel 也對區塊裝置有 home buffer 機制，因此構想是對 USBGadget
driver 準備一個補丁，將 USB gadget 的統計資訊寫入 /proc/USBGadget/lun.X 檔案。
虛擬媒體應用程式會觀察這些統計資訊。

### 虛擬媒體服務

虛擬媒體服務是一個獨立的應用程式，會在 DBus 上共存。它會從設定 json 檔初始化，
並暴露所有可供佈建（provisioning）的 VirtualMedia 物件。虛擬媒體負責：

- 將目前的虛擬媒體設定暴露給 DBus 使用者。
- 針對 Proxy 連線，衍生（spawn）nbdclient 並監控其生命週期。
- 針對 CIFS/HTTPS 連線，衍生 nbdkit 並監控其生命週期。
- 監控 uDEV 上所有與 NBD 相關的區塊裝置變化，並據以設定／解除設定 USB Gadget。
- 監控 NBD 裝置的閒置期間，以支援閒置逾時。

### 設定

在程序啟動時，虛擬媒體會讀取其設定檔，結構如下：

```json
"InactivityTimeout": 1800,              # Timeout of inactivity on device in seconds, that will lead to automatic disconnection
"MountPoints": {
    "ISO0": {
        "EndpointId": "/nbd/0",         # bmcweb endpoint (URL) configured for this type of connection
        "Mode": 0,                      # 0 - Proxy Mode, 1 - Legacy Mode
        "NBDDevice": "/dev/nbd0",       # nbd endpoint on device usually matches numeric value with EndpointId
        "UnixSocket": "/tmp/nbd.sock",  # defines which Unix socket will be occupied by connection
        "Timeout": 30,                  # timeout in seconds passed to nbdclient
        "BlockSize": 512,               # Block size passed to nbdclient
    }
},
```

### DBus Interface

虛擬媒體將暴露以下的物件結構。所有物件路徑都是上述設定檔的呈現

```ascii
/xyz/openbmc_project/VirtualMedia/Proxy/ISO0
/xyz/openbmc_project/VirtualMedia/Proxy/1
/xyz/openbmc_project/VirtualMedia/Legacy/0
/xyz/openbmc_project/VirtualMedia/Legacy/1
```

每個物件都會實作 `xyz.openbmc_project.VirtualMedia.Process` 介面，其定義如下：

| 名稱     | 型別     | 輸入  | 回傳    | 說明                                                                  |
| -------- | -------- | ----- | ------- | ------------------------------------------------------------------- |
| Active   | Property | -     | BOOLEAN | 若物件被使用中的程序佔用則為 `True`，否則為 `False`                  |
| ExitCode | Property | -     | INT32   | 若程序終止，此屬性將包含回傳的 exit code                             |

每個物件也會在 `xyz.openbmc_project.VirtualMedia.MountPoint` 下暴露其自身的設定
（所有屬性皆為 RO）

| 名稱                       | 型別     | 輸入  | 回傳    | 說明                                                                    |
| -------------------------- | -------- | ----- | ------- | ------------------------------------------------------------------------- |
| EndPointId                 | Property | -     | STRING  | 依設定                                                                    |
| Mode                       | Property | -     | BYTE    | 依設定                                                                    |
| Device                     | Property | -     | STRING  | 依設定                                                                    |
| Socket                     | Property | -     | STRING  | 依設定                                                                    |
| Timeout                    | Property | -     | UINT16  | 依設定                                                                    |
| BlockSize                  | Property | -     | UINT16  | 依設定                                                                    |
| RemainingInactivityTimeout | Property | -     | UINT16  | 針對已啟動的 endpoint，伺服器斷線連線前的剩餘秒數，否則為 0              |
| ImageURL                   | Property | -     | STRING  | 已掛載映像檔的 URL                                                        |
| WriteProtected             | Property | -     | BOOLEAN | 若映像檔以唯讀方式掛載則為 'True'，否則為 'False'                         |

每個物件暴露的另一個介面是 `xyz.openbmc_project.VirtualMedia.Stats` 下的統計資料
（所有屬性皆為 RO）：

| 名稱    | 型別     | 輸入  | 回傳   | 說明                                     |
| ------- | -------- | ----- | ------ | ---------------------------------------- |
| ReadIO  | Property | -     | UINT64 | 自映像檔掛載以來的讀取 IO 次數           |
| WriteIO | Property | -     | UINT64 | 自映像檔掛載以來的寫入 IO 次數           |

依物件路徑不同，物件會暴露不同的介面來掛載映像檔。

掛載可能是一項耗時的任務，因此必須引入事件導向的機制。Mount 與 Unmount 呼叫將
觸發非同步動作並立即結束，在背景作業完成時給出包含狀態的適當 signal。

對於 Proxy，其介面為 `xyz.openbmc_project.VirtualMedia.Proxy`，定義如下：

| 名稱       | 型別   | 輸入  | 回傳    | 說明                                                                       |
| ---------- | ------ | ----- | ------- | ------------------------------------------------------------------------------ |
| Mount      | Method | -     | BOOLEAN | 在指定物件上對 HOST 執行非同步掛載作業。                                      |
| Unmount    | Method | -     | BOOLEAN | 在指定物件上從 HOST 執行非同步卸載作業                                        |
| Completion | Signal | -     | INT32   | 背景作業完成後，成功回傳 0，失敗回傳 errno                                    |

對於 Legacy，其介面為 `xyz.openbmc_project.VirtualMedia.Legacy`，定義如下：

| 名稱       | 型別   | 輸入                                      | 回傳    | 說明                                                                                                                                                                                                                                                                                                                                                                                                                                                                      |
| ---------- | ------ | ----------------------------------------- | ------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Mount      | Method | STRING                                    | BOOLEAN | 在指定物件上對 HOST 執行非同步掛載作業，位置以 STRING 參數給定                                                                                                                                                                                                                                                                                                                                                                                                                |
| Mount      | Method | STRING<br>BOOLEAN<br>VARIANT<UNIX_FD,INT> | BOOLEAN | 在指定物件上對 HOST 執行非同步掛載作業，參數如下：<br><br>`STRING` : 映像檔的 url。應以 `smb://` 或 `https://` 前綴開頭<br>`BOOLEAN` : 已掛載 gadget 的 RW 旗標（應與遠端映像檔的能力一致）<br>`VARIANT<UNIX_FD,INT>` : 用於傳遞 null 字元結尾的機密資料（使用者名稱與密碼）之 named pipe 的檔案描述元。當沒有資料要傳遞時，應以 `INT` 傳遞 `-1` |
| Unmount    | Method | -                                         | BOOLEAN | 在指定物件上從 HOST 執行非同步卸載作業                                                                                                                                                                                                                                                                                                                                                                                                                                      |
| Completion | Signal | -                                         | INT32   | 背景作業完成後，成功回傳 0，失敗回傳 errno                                                                                                                                                                                                                                                                                                                                                                                                                                    |

Mount 與 unmount 作業若非同步作業已啟動則回傳 true，若初步檢查遇到錯誤則回傳
false。它們也可能指出適當的 dbus 錯誤。

## 曾考量的替代方案

OpenBMC 中現有的實作

## 影響

不應影響目前虛擬媒體實作的可用性

## 測試

TBD
