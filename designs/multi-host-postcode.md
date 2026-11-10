# 多主機 Postcode 支援

Author: Manikandan Elumalai, [manikandan.hcl.ers.epl@gmail.com]

其他貢獻者：無

建立日期：2020-07-02

## 問題說明

phosphor-host-postd 目前的實作僅支援透過 LPC 介面存取單一主機的
postcode。

隨著 open BMC 架構不斷演進，單一主機的支援變得更受限，因此需要實作
多主機的 post code 存取功能。

## 背景與參考

下方的元件圖以高階概覽的方式呈現單一主機 postcode 與歷史記錄的設計。
單一主機的設計相較於原本略有調整，以更好地符合社群慣例（在 D-Bus
物件上使用後綴 0）。

圖例說明：|Label|代表意義| |-----|---------| |`I:` |D-Bus 介面|
|`S:` |D-Bus 服務名稱（well-known bus name）| |`R:` |Repository 名稱| |`U:`
|Systemd 服務單元名稱| |`A:` |可執行檔名稱| |`H:` |硬體模組|

```text
                                         +-----------+
                       +------+          | 7 segment |
                       | Host |          |  display  |
                       +--+---+          +-----^-----+
                          |                    |
+-------------------------+--------------------+--------+
| BMC                     |                    |        |
|                      +--v----+         +-----+-----+  |
|                      | H:LPC +---------> H:(S)GPIO |  |
|                      +--+----+         +-----------+  |
|                         |                             |
| +-----------------------v--------------+              |
| | U:lpcsnoop                           |              |
| |                                      |              |
| | Current POST code:                   |              |
| | S:xyz.openbmc_project.State.Boot.Raw |              |
| | /xyz/openbmc_project/state/boot/raw0 |              |
| | I:xyz.openbmc_project.State.Boot.Raw >----+         |
| +--------------------------------------+    |         |
|                                             |         |
| +-------------------------------------------v---+     |
| | U:xyz.openbmc_project.State.Boot.PostCode     |     |
| | POST code history:                            |     |
| | +-------------------------------------------+ |     |
| | | S:xyz.openbmc_project.State.Boot.PostCode0| |     |
| | | /xyz/openbmc_project/State/Boot/PostCode0 | |     |
| | | I:xyz.openbmc_project.State.Boot.PostCode >----+  |
| | +-------------------------------------------+ |  |  |
| +-----------------------------------------------+  |  |
|                                                    |  |
|      +--------------------------------+            |  |
|      | Other consumers of POST codes: <------------+  |
|      | Redfish, etc...                |               |
|      +--------------------------------+               |
+-------------------------------------------------------+
```

由於多部主機無法一致地將它們的 POST code 寫入同一個位置，
因此需要一條額外的資料路徑來從每部主機接收 POST code。又因為這條
新的資料路徑在 ASPEED 硬體中沒有內建支援，所以還需要額外的邏輯
來驅動七段顯示器。

## 需求

- 從所有伺服器讀取 postcode。
- 根據主機位置選擇，將主機的 postcode 顯示在七段顯示器上。
- 提供命令介面讓使用者可以檢視任何伺服器（多主機）的目前
  postcode。
- 提供命令介面讓使用者可以檢視任何伺服器（多主機）的 postcode
  歷史記錄。
- 支援可熱插拔的主機。

## 建議設計

本文件提出一個新的設計，透過 IPMB 介面從多部主機讀取 port-80 的
post code。現有的單一主機 LPC 監聽（snooping）機制不受影響，且不會
顯示在下方圖表中。此設計同時支援主機探索，包含連線在插槽上
可熱插拔的主機。

為了此實作，以下模組將會更新

- phosphor-host-postd。
- phosphor-post-code-manager。
- 平台專屬的 OEM handler（fb-ipmi-oem）。
- bmcweb（redfish logging service）。

### 介面圖

以下提供 post code 介面圖及其流程順序

```text
+---------------------------------------------------+
| BMC                                               |
|                                                   |
|  +-----------------------+    +----------------+  |         +----+-------------+
|  | A:ipmid               |    | IPMB Bridge    <--+--IPMB-+-|BIC |             |
|  |                       |    | (R:ipmbbridge) |  |       | |    |     Host1   |
|  | +-------------------+ |    +--------+-------+  |       | +------------------+
|  | | OEM IPMI Handlers | |             |          |       |           .
|  | | (R:fb-ipmi-oem)   <-+-------------+          |       |           .
|  | --------------------+ |   +----------------+   |       |           .
|  +-+---------------------+   | Host selection |   |       | +------------------+
|    |                         | monitoring     <---+--+    +-|BIC |             |
|    |                         +---------+------+   |  |      |    |     HostN   |
|    |                                   |          |  |      +------------------+
|    | +---------------------------------v------+   |  |
|    | | R:phosphor-host-postd (1 process)      |   |  |      +---------------+
|    | | Per-host POST code object:             |   |  +-GPIO-| Button/switch |
|    | | /xyz/openbmc_project/state/boot/raw<N> |   |         | input device  |
|    +-> I:xyz.openbmc_project.State.Boot.Raw   >-+ |         +---------------+
|      +---------------------------------+------+ | |
|                                        |        | |         +-------------------+
|                                        +--------+-+----GPIO-> 7-segment display |
|                                                 | |         +-------------------+
|                                                 | +------+
|                                                 |        |
|  +----------------------------------------------v---+    |
|  | R:phosphor-post-code-manager (N processes)       |    |
|  | Per-host POST code history service:              |    |
|  | +----------------------------------------------+ |    |
|  | | S:xyz.openbmc_project.State.Boot.PostCode<N> | |    |
|  | | /xyz/openbmc_project/State/Boot/PostCode<N>  | |    |
|  | | I:xyz.openbmc_project.State.Boot.PostCode    >-+-+  |
|  | +----------------------------------------------+ | |  |
|  |                        ...                       | |  |
|  +--------------------------------------------------+ |  |
|                                                       |  |
|       +--------------------------------+              |  |
|       | Other consumers of POST codes: <--------------+  |
|       | Redfish, CLI, etc...           |                 |
|       +--------------------------------+                 |
|                                                          |
+----------------------------------------------------------+
```

### Postcode 流程

- BMC 開啟主機電源。
- 主機開始持續將 postcode IPMI 訊息傳送至 BMC。
- ipmbbridged（phosphor-ipmi-ipmb）將該訊息轉傳給 IPMI daemon。
- ipmid（phosphor-ipmi-host）將主機資訊附加在 postcode 上，並將
  值寫入由 phosphor-host-postd 託管的適當 D-Bus 物件。
- phosphor-host-postd 根據透過 D-Bus 介面讀取的主機位置，將 postcode
  顯示在七段顯示器上。
- phosphor-post-code-manager 透過 D-Bus 訊號接收新的 POST code，並將
  postcode 作為歷史記錄存放在 /var 目錄中。

## 平台專屬 OEM Handler（fb-ipmi-oem）

此函式庫是
[phosphor-ipmi-host](https://github.com/openbmc/phosphor-host-ipmid) 的一部分，並透過
[phosphor-ipmi-ipmb](https://github.com/openbmc/ipmbbridge) 從主機取得 postcode。

- 註冊 IPMI OEM postcode callback handler。
- 從 IPMI 訊息中提取 postcode（phosphor-ipmi-host/phosphor-ipmi-ipmb）。
- 在由 `phosphor-host-postd` 託管的適當 D-Bus `Raw` 物件上設定
  `Value` 屬性。其他程式（例如 `phosphor-post-code-manager`）可以
  訂閱此物件上的 `PropertiesChanged` 訊號以取得更新。

## phosphor-host-postd

此實作涉及 phosphor-host-postd 中的以下變更。

- `phosphor-host-postd` 處理 `xyz.openbmc_project.State.Boot.Raw`
  介面每個實例上 `Value` 屬性的設定事件。
- phosphor-host-postd 從 dbus 屬性讀取主機選擇。
- 將透過 D-Bus 讀取到的、所選主機最新的 postcode 顯示在
  七段顯示器上。

### D-Bus 介面

多主機 post-code 需要建立以下 D-Bus 名稱。

```text
Service name: xyz.openbmc_project.State.Boot.Raw
Obj path name: /xyz/openbmc_project/State/Boot/RawX(1,2..N)
Interface name: xyz.openbmc_project.State.Boot.Raw
```

## phosphor-post-code-manager

phosphor-post-code-manager 是針對多主機的 multi service 設計。單一
主機 postcode 處理的 D-bus 命名慣例將更新以符合社群的命名方案。

- 為單主機與多主機系統建立對應的 D-Bus 服務名稱。社群
  遵循的慣例是單主機使用 Host0，多主機使用 Host1 到 N。
- phosphor-post-code-manager 訂閱由 `phosphor-host-postd` 託管的
  `Raw<N>` 物件上的變更。
- 根據從 phosphor-host-postd 收到的事件，將 post-code 儲存/讀取至
  目錄（/var/lib/phosphor-post-code-manager/hostX(1,2,3..N)）。

### D-Bus 介面

多主機 post-code 需要建立以下 D-Bus 名稱。

```text
Service name: xyz.openbmc_project.State.Boot.PostCodeX(1,2..N)
Obj path name: /xyz/openbmc_project/State/Boot/PostCodeX(1,2..N)
Interface name: xyz.openbmc_project.State.Boot.PostCode
```

## bmcweb

需要透過 redfish logging service 來處理多主機的 postcode 歷史記錄。

## 替代設計

### phosphor-post-code-manager 單一程序方式

此實作考慮用單一服務來處理多主機 postcode。在這種方式下，所有
D-Bus 處理都由單一程序負責。

單一服務與現有的 x86-power-control 及 obmc-console 不同，後者
採用的是 multi-service 方式。

相較於單一服務，multi-service 方式在處理多主機時更具可擴展性。
