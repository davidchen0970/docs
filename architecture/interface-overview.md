# OpenBMC 介面

目的：本文介紹 BMC 主要介面的簡化觀點，希望能提供適合廣大讀者參考的內容：

- 工程師在特定領域提供專業知識，並了解其介面所帶來的使用案例與威脅。
- 為 BMC 管理員與系統整合商提供 BMC 系統介面的簡化觀點，例如了解哪些介面可以被停用。
- 管理與資安人員需要一切協同運作良好，例如了解 BMC 的攻擊面。

## 介面與服務簡介

本節說明 BMC 的主要介面以及它們彼此的關聯。內容從 BMC 的實體介面開始，逐步過渡到網路服務等抽象概念。目的是把 OpenBMC 專案中必要的介面放在一個架構框架中呈現，讓讀者能據以推論有哪些介面存在、它們彼此如何關聯。這提供了基礎，用以推論哪些介面可以被停用、它們如何受到保護等等。附錄提供文中每個介面與服務的詳細資訊。

OpenBMC 的服務及其提供的介面由 `systemd` 控制。本文會引用 OpenBMC 的 `systemd` 單元名稱，協助將觀念連結到原始碼。請讀者事先熟悉 [systemd concepts][]。為求清晰，範本化單元（`unit@.service`）可能會省略。單元檔案中的相關細節可能會顯示出來，例如實作某個服務的程式。

OpenBMC 的 [Service Management][] 介面可以控制 `systemd` 服務。例如，停用某個 BMC 服務也會停用對應的外部介面。

[systemd concepts]:
  https://www.freedesktop.org/software/systemd/man/systemd.html#Concepts
[service management]:
  https://github.com/openbmc/phosphor-dbus-interfaces/blob/master/yaml/xyz/openbmc_project/Control/Service/README.md

文中附上圖示以協助視覺化各種關聯。圖示左側為管理代理端，中間為 BMC，右側為主機元素。這些圖示經過簡化，並非要完整呈現全貌。

### 實體介面

這裡顯示 BMC 的實體連線，包含網路、USB、UART 序列，以及與其主機平台的連線。這裡對主機採用簡化觀點，只顯示直接連線到 BMC 的主機介面。典型的主機還會有用於控制台、網路等的其他連線。

BMC 與其主機平台之間的介面，會因 BMC 與主機平台的實作不同而有極大差異。本節及其小節所提供的資訊，目的是說明常見的元素，而非代表某個特定的系統。本節預計供其他文件引用，那些文件會提供特定 BMC 與主機實作的細節。

```ascii
        +----------------+         +----------------+
        | BMC            |         | Host           |
        |                |         |                |
        | Network       -+- LPC ---+-               |
       -+- eth0         -+--PCIe --+-               |
       -+- eth1         -+--UART --+-               |
        |  lo           -+- I2C ---+-               |
        |               -+--I3C ---+-               |
        | USB           -+- SPI ---+-               |
       -+- usb0         -+- PECI --+-               |
        |               -+- GPIOs -+-               |
        | Serial        -+- UTMI --+-               |
       -+- tty0          |         |                |
        |                |         |                |
        +----------------+         +----------------+
```

#### 主機-BMC 實體介面傳輸協定

這裡列出運作於 BMC-主機實體介面之上的協定：

- 主機 IPMI。
- [MCTP][]。OpenBMC 提供 LPC、PCIe、UART 上的 MCTP。
- 自訂 OEM 解決方案。
- SMBus。

[mctp]:
  https://www.dmtf.org/sites/default/files/standards/documents/DSP0236_1.3.0.pdf

#### 主機-BMC 資料模型

這裡列出在 BMC-主機傳輸協定之上流動的資料規格：

- 主機 IPMI。
- PLDM（DMTF 文件 DSP0240）。
- 自訂 OEM 解決方案。

### 提供的網路服務

OpenBMC 透過其管理網路提供服務。以下依埠號列出預設服務。每個服務的更多資訊見下方各節或附錄。

```ascii
        +----------------------------------+
        | BMC                              |
        |                                  |
       -+-+ Network services               |
        | |                                |
        | +-+ TCP ports                    |
        | | +- 22 ssh - shell              |
        | | +- 80 HTTP (no connection)     |
        | | +- 443 HTTPS                   |
        | | +- 2200 ssh - host console     |
        | | +- 5355 mDNS service discovery |
        | |                                |
        | +-+ UDP ports                    |
        |   +- 427 SLP                     |
        |   +- 623 RMCP+ IPMI              |
        |   +- 5355 mDNS service discovery |
        |                                  |
        +----------------------------------+
```

提供給已連線用戶端的服務，可能會使用埠來處理：

- 進行中的 SSH 工作階段。
- 進行中的 KVM-IP 工作階段。
- 進行中的虛擬媒體工作階段。

### 使用的網路服務

本節列出 OpenBMC 系統所使用的網路服務。OpenBMC 以一般方式使用了通常的服務，例如 NTP、DNS 與 DHCP。此外，OpenBMC 還會使用：

- TFTP（預設停用，由 BMC 操作員呼叫時啟用）- 用於抓取韌體映像以進行 [code update][] 的簡易 FTP 用戶端。
- SNMP 管理者，用於接收 [SNMP traps][]（啟用時）。

[code update]:
  https://github.com/openbmc/docs/blob/master/code-update/code-update.md
[snmp traps]:
  https://github.com/openbmc/phosphor-snmp/blob/master/docs/snmp-configuration.md

### 主機控制台

OpenBMC 以多種方式提供對其主機序列控制台的存取：

- 用戶端透過網路 IPMI 存取。
- 用戶端透過 ssh 埠 2200 存取。
- hostlogger 機制。

```ascii
                +---------------------------+    +-----------------+
                | BMC                       |    | Host            |
 ipmitool sol   |                           |    |                 |
 activate       |                           |    |                 |
 UDP port 623 .... netipmid ------------}   |    |                 |
                |                       }   |    |                 |
 ssh -p 2200   ... obmc-console-client -}---+----+- serial UART    |
 TCP port 2200  |                       }   |    |  console        |
                |  hostlogger ----------}   |    |                 |
                |                           |    |                 |
                +---------------------------+    +-----------------+
```

[obmc-console][] 說明主機 UART 連線如何在 BMC 內部被抽象化為 Unix domain socket。

[obmc-console]: https://github.com/openbmc/obmc-console/blob/master/README.md

### Web 服務

OpenBMC 提供一個名為 BMCWeb 的客製 HTTP／Web 伺服器。

```ascii
        +--------------------------------------------------+
        | BMC                                              |
        |                                                  |
       -+-+ Network services                               |
        | ++ TCP                                           |
        |  +- 443 HTTPS - BMCWeb -> { static content       |
        |  |                        {   Web app (webui)    |
        |  +- (other ports) <---+   {   Redfish schema     |
        |       |               |   { /login               |
        |       V               |   { Redfish REST APIs    |
       -+- Websockets -+        |   { Phosphor REST APIs   |
        |              |        +<--{-- can set up:        |
        |              |            {     KVM-IP, USB-IP,  |
        |           various         {     Virtual Media    |
        |                                                  |
        +--------------------------------------------------+
```

在圖中，箭頭代表控制流程從 web 代理端流向 BMCWeb API，其中一些會建立 Websockets，讓網路代理端能直接與所需的介面通訊（不經過 BMCWeb）。

請注意 [BMCWeb is configurable][]，可在編譯期間設定。本節描述預設設定（在 HTTPS 傳輸協定上、TCP 埠 443 提供 HTTP 應用程式協定）。

[bmcweb is configurable]: https://github.com/openbmc/bmcweb#configuration

提供的服務：

- Web 應用程式（webui-vue）及其他靜態內容
- REST API，包含自訂的 phosphor-rest 與 Redfish API
- KVM-IP（Keyboard, Video, Mouse over IP）
- 透過 USB-IP（Universal Serial Bus over IP）提供的虛擬媒體
- 其他

### 主機 IPMI 服務

OpenBMC 提供主機 IPMI 服務。

```ascii
    +---------------+    +-----------------+
    | BMC           |    | Host            |
    |               |    |                 |
    |        ipmid -+----+-                |
    |               |    |                 |
    +---------------+    +-----------------+
```

IPMI 韌體防火牆（旨在控制可使用哪些主機命令與通道）並未在 OpenBMC 中實作。有支援 [Phosphor host IPMI whitelist][] 機制。

[phosphor host ipmi whitelist]:
  https://github.com/openbmc/openbmc/blob/master/meta-phosphor/classes/phosphor-ipmi-host-whitelist.bbclass

### D-Bus 介面

OpenBMC 使用 D-Bus 介面作為 OpenBMC 應用程式之間通訊（程序間通訊）的主要方式。請注意也會使用其他方法，例如 Unix domain socket。

```ascii
        +--------------------------------------------------+
        | BMC                                              |
        |                                                  |
        | +-------+                                        |
        | | D-Bus |                                        |
        | |      -+- bmcweb                                |
        | |      -+- ipmid                                 |
        | |      -+- ...                                   |
        | |      -+- many more (not shown here)            |
        | |      -+- ...                                   |
        | |       |                                        |
        | +-------+                                        |
        |                                                  |
        +--------------------------------------------------+
```

若要深入了解，請閱讀 [Phosphor D-Bus interface docs][]，並在 xyz/openbmc_project 路徑下的各子目錄中搜尋 README 檔案。

[phosphor d-bus interface docs]:
  https://github.com/openbmc/phosphor-dbus-interfaces

## 介面與服務

本節列出本文中顯示的每個介面與服務。目的是說明每一項的相關性，以及如何在原始碼中找到詳細資訊。

### BMC 網路

本節顯示 BMC 管理網路在運作環境上的各種變化。

BMC 可能連線到一個用於管理 BMC 的網路。這個網路被稱為「管理網路」，以區別於主機系統所連線的承載網路（payload network）。這些通常是各自獨立的網路。

```ascii
             +-----------+      +----------------+
             | BMC       |      | Host           |
 management   |           |      |                |
 network   ---+- Network  |      |       Network -+- payload
             |           |      |                |  network
             +-----------+      +----------------+
```

BMC 可能由 Network Controller Sideband Interface（NC-SI）提供服務，它與主機維持一個邏輯上獨立的網路，如以下圖示：

```ascii
             +-----------+      +----------------+
             | BMC       |      | Host           |
 management   |           |      |                |
 network    +-+- Network  |      |       Network -+-+
            | |           |      |                | |
            | +-----------+      +----------------+ |
            |                                       |
            |      +------------------+             |
            |      | NIC              |             |
            |      |.........+       -+-------------+
            +------+- side-  :        |
 management -------+- band   :       -+- payload
 network           |.........+        |  network
                   +------------------+
```

BMC 的管理網路可能由其主機系統提供，且沒有任何直接連線到主機外部，如以下圖示：

```ascii
             +-----------+      +----------------+
             | BMC       |      | Host           |
             |           |      |                |
          +--+- Network  |      |       Network -+- payload
          |  |           |      |                |  network
          |  |           |   +--+- management    |
          |  |           |   |  |  network       |
          |  +-----------+   |  +----------------+
          |                  |
          +------------------+
```

BMC 的管理網路可能連線到 USB（LAN over USB）：

```ascii
             +-----------+      +----------------+
             | BMC       |      | Host           |
        +-+  |           |      |                |
   USB --+---+- Network  |      |       Network -+- payload
        +-+  |           |      |                |  network
             |           |      |                |
             +-----------+      +----------------+
```

### BMC 序列

這提供對 BMC 控制台的存取，其功能包括控制 BMC 的 U-Boot，接著提供對 BMC shell 的存取。請與主機序列控制台存取相互對照。

### 網路介面

這指的是 BMC 上標準的 NIC 與 Linux 網路服務。

### Secure Shell (SSH)

這指的是 SSH 協定，它同時提供對 BMC 的安全 shell（ssh）與安全複製（scp）存取。OpenBMC 使用 Dropbear SSH 實作。請注意埠 22 連線到 BMC 的 shell，而埠 2200 連線到主機控制台。

### HTTP 與 HTTPS

OpenBMC 支援在 HTTPS 之上的 HTTP 應用程式協定，兩者都由 BMCWeb 伺服器處理。"http" URI scheme 預設停用，但可透過 BMCWeb 的編譯設定選項啟用。

### 主機序列控制台

指的是 BMC 對其主機序列連線的存取，通常用於存取主機系統的控制台。另見 `obmc-console-server`，它對各種 BMC 內部服務提供主機序列存取。請與對 BMC 序列連線的存取相互對照，後者提供對 BMC 控制台的存取。

### 服務探索

指的是多播探索服務（mDNS）。例如，你可以用 `avahi-browse -rt _obmc_rest._tcp` 命令找到 BMC。

### Service Location Protocol (SLP)

指的是由 `slpd` 提供的單播服務探索協定。例如，你可以用
`slptool -u ${ip} findsrvtypes or findsrvs` 命令找到 BMC。

### RMCP+、IPMI 與 ipmitool

指的是由 `netipmid` 提供的 RMCP+ 協定與 IPMI 實作，原始碼在此：`https://github.com/openbmc/phosphor-net-ipmid`，部分細節由 [IPMI Session management][] 提供。網路 IPMI 提供對許多資源的存取，包含主機 IPMI 存取、SOL（對主機控制台的存取）等等。也稱為帶外（out of band）IPMI。請與與主機互動的 host-IPMI，以及提供替代功能的 Redfish 相互對照。

BMC 的 RMCP+ IPMI 介面設計為由 [ipmitool][] 外部命令操作。

[ipmi session management]:
  https://github.com/openbmc/phosphor-dbus-interfaces/blob/master/yaml/xyz/openbmc_project/Ipmi/SESSION_README.md
[ipmitool]: https://github.com/ipmitool/ipmitool

### 主機 IPMI

指的是由 `ipmid` 程式提供、面向主機的 IPMI 服務，原始碼在此：`https://github.com/openbmc/phosphor-host-ipmid`。systemd 服務為 `phosphor-ipmi-host`，由 `ipmid` 程式實作。也稱為帶內（in-band）IPMI。請與面向網路的 RMCP+，以及提供替代功能的 PLDM 相互對照。

### BMC shell

這指的是 BMC 的命令列介面，預設為 `bash` 程式，透過 BMC 檔案系統上的 `/bin/sh` 路徑提供。請注意 shell（連同其公用程式）提供對 BMC 許多內部與外部介面的存取。

### obmc-console

這指的是在 <https://github.com/openbmc/obmc-console> 中對多重獨立控制台的支援，以及兩個應用程式：

- `obmc-console-server` 將主機控制台（UART）連線抽象化為 Unix domain socket。
- `obmc-console-client` 可將控制台連線到 SSH 工作階段。

其他應用程式也會使用 console server。

### hostlogger

指的是由 `hostlogger` 程式提供的 BMC 服務，位於：
<https://github.com/openbmc/phosphor-hostlogger>，它監聽 `obmc-console-server`，並將主機控制台訊息記錄到 BMC 的檔案系統中。

### BMCWeb web 伺服器

指的是客製的 HTTP／Web 伺服器，原始碼在此：
<https://github.com/openbmc/bmcweb>。請注意 BMCWeb 可依照 <https://github.com/openbmc/bmcweb#configuration> 進行設定，透過建置期間的選項來控制它提供哪些介面。例如，有以下設定選項可以：

- 啟用從 TFTP 伺服器下載韌體映像
- 啟用 "http" URI scheme
- 其他

此 web 伺服器也會為 KVM-IP、Virtual-USB 等服務建立安全的 Websockets。

### Redfish

指的是由 BMCWeb web 伺服器提供的一組 Redfish REST API。詳細資訊見：<https://github.com/openbmc/bmcweb/blob/master/Redfish.md>，文件見：<https://github.com/openbmc/docs/blob/master/REDFISH-cheatsheet.md>

### phosphor-dbus-rest

指的是由 BMCWeb 伺服器選擇性提供的傳統 REST API。文件：
<https://github.com/openbmc/docs/blob/master/REST-cheatsheet.md>

### KVM-IP

指的是 OpenBMC 對 Remote Frame Buffer（RFB，又稱 VNC）協定的實作，讓你能遠端操作主機系統的鍵盤、影像與滑鼠（KVM）。見 <https://github.com/openbmc/obmc-ikvm/blob/master/README.md>。也稱為 IPKvm。請勿與 Kernel Virtual Machine（另一種 KVM）混淆。

### 虛擬媒體

- 也稱為：remote media 與 USB-over-IP。
- 設計：<https://github.com/openbmc/docs/blob/master/designs/VirtualMedia.md>
- 請與 LAN-over-USB 相互對照。

### 虛擬 USB

也稱為 USB-over-IP，有助於實作虛擬媒體。請與 BMC 及主機的實體 USB 埠相互對照。
