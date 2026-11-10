# 多主機 IPMI 設計

作者：Velumani T(velu)，[velumanit@hcl](mailto:velumanit@hcl.com)、Kumar
T(kumar_t)，[thangavel.k@hcl.com](mailto:thangavel.k@hcl.com)

其他貢獻者：

建立日期：2020 年 6 月 26 日

## 問題描述

目前版本的 OpenBMC 在 IPMI 指令處理上並不支援多主機實作。我們有一套多主機
系統，並提出此設計來支援多主機。

如下所述，各主機透過 IPMB 介面連接，所有與主機相關的通訊都以 IPMB 為基礎。
OpenBMC 使用 ipmbbridged 來管理 IPMB 匯流排，IPMB 訊息會被路由到 ipmid。

問題 1：ipmbridged 不會傳送通道編號（也就是 HostId）。問題 2：ipmid
並不知道請求是從哪一個 IPMB 通道傳來的。ipmid
的處理常式需要主機詳細資訊，才能取得各主機專屬的回應。

## 背景與參考資料

IPMI 與 IPMB 系統架構：

```ascii
 +------------------------------------+
 |               BMC                  |
 | +-----------+       +------------+ |      +--------+
 | |           |       |            | | IPMB1|        |
 | |           |       |            |-|------| Host-1 |
 | |           |       |            | |      |        |
 | |           |       |            | |      +--------+
 | |           |       |            | |
 | |           |       |            | |
 | |           | D-Bus |            | |      +--------+
 | | ipmid     |-------| ipmbbridged| | IPMB2|        |
 | |           |       |            |-|------| Host-2 |
 | |           |       |            | |      |        |
 | |           |       |            | |      +--------+
 | |           |       |            | |
 | |           |       |            | |
 | |           |       |            | |      +--------+
 | |           |       |            | | IPMBn|        |
 | |           |       |            |-|------| Host-N |
 | |           |       |            | |      |        |
 | +-----------+       +------------+ |      +--------+
 +------------------------------------+
```

各主機以 IPMB 介面連接，主機數量可為 1 到 N。來自主機的 IPMB
請求會由 ipmbbridged 路由到 ipmid。IPMB
請求會由 ipmid 或任何服務透過 D-Bus 介面路由傳出，而送出的 IPMB
請求則由 ipmbbridged 路由到 IPMB 介面。

## 需求

目前版本的 OpenBMC 在 IPMI 指令處理上並不支援多主機實作。我們有一套多主機
系統，並提出此設計來支援多主機。

## 建議設計

為了處理問題 1 與問題 2，我們針對 ipmbbridged 與 ipmid
提出以下的設計變更。為了處理來自網路端的帶外（out-of-band）IPMI
指令，相關提案收錄在「netipmid 的變更」一節。

## 問題 1：ipmbbridged 的變更

ipmbbridged 需傳送請求是從哪一個通道收到的詳細資訊。

### 變更：將主機詳細資訊作為額外參數傳送

在路由來自主機通道的 IPMB 請求時，我們會在 json
設定檔中新增一個主機 ID 的項目 `'"devIndex": 0'`。ipmbbridged
會在 D-Bus 介面中將 `'"devIndex": 0'`
作為選擇性參數（options）傳給 ipmid，這可以用來取得訊息是從哪一條 IPMB
匯流排傳來的資訊。

json 檔案如下所示。每個 devIndex 可以有一個 "me" 與一個 "ipmb"
通道。為了確保既有平台不受影響，如果檔案中沒有 "devIndex"
這個鍵，ipmbbridged 會使用預設的 "devIndex" 值 0。

```json
[
  {
    "type": "me",
    "slave-path": "/dev/ipmb-1",
    "bmc-addr": 32,
    "remote-addr": 64,
    "devIndex": 0
  },
  {
    "type": "ipmb",
    "slave-path": "/dev/ipmb-2",
    "bmc-addr": 32,
    "remote-addr": 64,
    "devIndex": 0
  },
  {
    "type": "me",
    "slave-path": "/dev/ipmb-3",
    "bmc-addr": 32,
    "remote-addr": 64,
    "devIndex": 1
  },
  {
    "type": "ipmb",
    "slave-path": "/dev/ipmb-4",
    "bmc-addr": 32,
    "remote-addr": 64,
    "devIndex": 1
  }
]
```

## 問題 2：ipmid 的變更

將 ipmbbridged 傳送的選擇性參數作為主機詳細資訊接收。在 executionEntry
方法呼叫中收到該參數後，會將其傳遞給 common 與 oem
處理常式中的指令處理常式。指令處理常式可以利用主機資訊來取得各主機專屬的資料。

舉例來說，host1 傳送一個請求向 BMC 取得開機順序，BMC
會為每一台主機分別維護資料。當這個指令傳到 ipmid 時，指令處理常式會取得接收該
指令的主機。處理常式會取得 host1 的開機順序詳細資料，並由指令處理常式回應。這
適用於 common 與 oem 處理常式。

## netipmid 的變更

可以使用 "options" 參數從 netipmid
傳送主機資訊。針對 ipmbbridged 提出的變更也可以應用在 netipmid
上。netipmid 會傳送 "devIndex" 來表示請求是從哪一個通道傳來的。ipmid
不需要再做其他變更。netipmid
可以有多種方式來處理多主機，以下列出部分做法，但不限於這些。

1. Virtual Ethernet interfaces（虛擬乙太網路介面）- 每台主機一個虛擬介面。
2. Different port numbers（不同連接埠號碼）- 每台主機可以使用不同的連接埠號碼。
3. VLAN Ids（VLAN ID）- 可以使用 VLAN ID 來支援多主機。netipmid
   需要有一個設定檔，可在其中為每台主機設定介面。也可以為每台主機設定一個或多個
   介面，介面可以是虛擬的或實體的。以下是設定檔範例：

{"Host":1, "Interface-1":"eth0", "Interface-2":"eth1", "Interface-3":"veth4",
"Interface-4":"veth5" }, {"Host":2, "Interface-1":"eth2", "Interface-2":"eth3",
"Interface-3":"veth1", "Interface-4":"veth2" },

做法 1 的實作範例：虛擬乙太網路介面。

```ascii
+--------------------------------------------+
|           BMC                              |
| +--------+       +-----------+   +------+  |      +--------+
| |        | D-Bus |           |   |      |  |      |        |
| |        |-------| netipmid1 |---|veth1 |---------| Host-1 |
| |        |       |           |   |      |  |      |        |
| |        |       +-----------+   +------+  |      +--------+
| |        |                                 |
| |        |       +-----------+   +------+  |      +--------+
| | ipmid  | D-Bus |           |   |      |  |      |        |
| |        |-------| netipmid2 |---|veth2 |---------| Host-2 |
| |        |       |           |   |      |  |      |        |
| |        |       +-----------+   +------+  |      +--------+
| |        |                                 |
| |        |       +-----------+   +------+  |      +--------+
| |        | D-Bus |           |   |      |  |      |        |
| |        |-------| netipmidN |---|vethN |---------| Host-N |
| |        |       |           |   |      |  |      |        |
| +--------+       +-----------+   +------+  |      +--------+
+--------------------------------------------+
```

在上圖中，每台主機執行一個 netipmid 實例。每個實例綁定到一個虛擬乙太網路介面，
虛擬介面 ID 可以用來建立 devIndex，它代表 HostId。

## 曾考慮的替代方案

## 做法 1：ipmbbridged 在 payload 中傳送 host-id

ipmbbridged 將被修改為在資料 payload 中傳送主機
ID。這看起來是個簡單的變更，但會影響到已在使用它的既有平台，因此可能不是合適的
做法。

## 做法 2：建立多個 ipmid 來處理多主機，每台主機一個 ipmid 程序

這是多服務的做法，會為每台主機衍生一個 ipmid
服務實例來回應。變更看起來簡單，且相對於現有設計沒有重大的設計變更。但許多
common 處理常式會在多個實例中重複執行。

## 做法 3：使用不同的 IPMI 通道來處理多台主機

使用不同的 IPMI 通道來處理多台主機，在 ipmbbridged
中可以使用通道 ID 來辨識主機。在這個做法中我們會有多個 ipmbbridged
實例，每個實例會註冊一個通道編號。依規格，最大通道編號受限於
8，這限制了可支援的最大主機數。

## 影響

ipmid 的指令處理常式函式可能不會受到影響。此設計不會影響現有功能。

## 測試

建議的設計可以在連接了多台主機的平台上進行測試。指令請求會從所有主機接收，回應
則是各主機專屬的資料。當請求以 IPMB 指令的形式從主機傳來時，ipmbbridged
會附加 devIndex，ipmid 接收到相對應的 devIndex。ipmid
會依據收到的 devIndex（Host Id）回應，回應一路傳回到主機。資料可以在主機端進行
驗證。
