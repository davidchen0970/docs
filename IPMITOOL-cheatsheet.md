# IPMITOOL-cheatsheet.md

本文件旨在提供一組用於 OpenBMC 的 ipmitool 指令。

## 電源（Power）

### 1. 檢視伺服器狀態

```bash
ipmitool -C 17 -H "$BMC_IP" -I lanplus -U "$BMC_USER" -P "$BMC_PASSWD" power status
```

### 2. 伺服器開機

```bash
ipmitool -C 17 -H "$BMC_IP" -I lanplus -U "$BMC_USER" -P "$BMC_PASSWD" power on
```

### 3. 伺服器關機

```bash
ipmitool -C 17 -H "$BMC_IP" -I lanplus -U "$BMC_USER" -P "$BMC_PASSWD" power off
```

### 4. 伺服器重設（reset）

```bash
ipmitool -C 17 -H "$BMC_IP" -I lanplus -U "$BMC_USER" -P "$BMC_PASSWD" power reset
```

### 5. 伺服器電源循環（power cycle）

```bash
ipmitool -C 17 -H "$BMC_IP" -I lanplus -U "$BMC_USER" -P "$BMC_PASSWD" power cycle
```

### 6. 伺服器軟性關機（power soft）

```bash
ipmitool -C 17 -H "$BMC_IP" -I lanplus -U "$BMC_USER" -P "$BMC_PASSWD" power soft
```

### 7. 直接對處理器送出一個診斷中斷（不支援）

```bash
ipmitool -C 17 -H "$BMC_IP" -I lanplus -U "$BMC_USER" -P "$BMC_PASSWD" power diag
```

## 使用者（Users）

### 1. 檢視所有使用者的資訊

```bash
ipmitool -C 17 -H "$BMC_IP" -I lanplus -U "$BMC_USER" -P "$BMC_PASSWD" user list
```

### 2. 顯示 BMC 使用者的簡要摘要

```bash
ipmitool -C 17 -H "$BMC_IP" -I lanplus -U "$BMC_USER" -P "$BMC_PASSWD" user summary
```

### 3. 以指定名稱建立一個 BMC 使用者

```bash
ipmitool -C 17 -H "$BMC_IP" -I lanplus -U "$BMC_USER" -P "$BMC_PASSWD" user set name <userid> <username>
```

### 4. 為指定使用者設定密碼

```bash
ipmitool -C 17 -H "$BMC_IP" -I lanplus -U "$BMC_USER" -P "$BMC_PASSWD" user set password <userid>[<password>]
```

### 5. 停用指定使用者存取 BMC 的權限

```bash
ipmitool -C 17 -H "$BMC_IP" -I lanplus -U "$BMC_USER" -P "$BMC_PASSWD" user disable <userid>
```

### 6. 啟用指定使用者存取 BMC 的權限

```bash
ipmitool -C 17 -H "$BMC_IP" -I lanplus -U "$BMC_USER" -P "$BMC_PASSWD" user enable <userid>
```

## 現場可更換單元（Field-replaceable Unit，FRU）

### 1. 檢視 FRU 資訊

```bash
ipmitool -C 17 -H "$BMC_IP" -I lanplus -U "$BMC_USER" -P "$BMC_PASSWD" fru list
```

## 感測器資料記錄（Sensor Data Record，SDR）

### 1. 檢視 SDR 資訊

```bash
ipmitool -C 17 -H "$BMC_IP" -I lanplus -U "$BMC_USER" -P "$BMC_PASSWD" sdr
```

### 2. 查詢 BMC 中相關的 SDR 資訊

```bash
ipmitool -C 17 -H "$BMC_IP" -I lanplus -U "$BMC_USER" -P "$BMC_PASSWD" sdr info
```

### 3. 檢視感測器資料記錄

```bash
ipmitool -C 17 -H "$BMC_IP" -I lanplus -U "$BMC_USER" -P "$BMC_PASSWD" sdr list [all|full|compact|event|mcloc|fru|generic]

##
all     : All SDR records (sensors and positioners)
full    : Complete sensor recording
compact : Simple sensor recording
event   : Event information recorded by the sensor
mcloc   : Manage controller locator records
fru     : FRU (Field Replaceable Unit) locator record
generic : General SDR records
```

## 感測器（Sensors）

### 1. 檢視感測器資訊

```bash
ipmitool -C 17 -H "$BMC_IP" -I lanplus -U "$BMC_USER" -P "$BMC_PASSWD" sensor list
```

## 管理控制器（Management Controller，MC）

### 1. 指示 BMC 執行冷開機重設（cold reset）

```bash
ipmitool -C 17 -H "$BMC_IP" -I lanplus -U "$BMC_USER" -P "$BMC_PASSWD" mc reset cold
```

### 2. 指示 BMC 執行熱重設（warm reset，不支援）

```bash
ipmitool -C 17 -H "$BMC_IP" -I lanplus -U "$BMC_USER" -P "$BMC_PASSWD" mc reset warm
```

### 3. 檢視 BMC 資訊

```bash
ipmitool -C 17 -H "$BMC_IP" -I lanplus -U "$BMC_USER" -P "$BMC_PASSWD" mc info
```

### 4. 檢視 BMC 目前可用的操作選項

```bash
ipmitool -C 17 -H "$BMC_IP" -I lanplus -U "$BMC_USER" -P "$BMC_PASSWD" mc getenables
```

## 通道（Channels）

### 1. 顯示所選資訊通道的驗證功能

```bash
ipmitool -C 17 -H "$BMC_IP" -I lanplus -U "$BMC_USER" -P "$BMC_PASSWD" channel authcap <channel number> <max privilege>
```

### 2. 顯示所選通道的資訊

```bash
ipmitool -C 17 -H "$BMC_IP" -I lanplus -U "$BMC_USER" -P "$BMC_PASSWD" channel info [<channel number>]
```

### 3. 檢視通道資訊

```bash
ipmitool -C 17 -H "$BMC_IP" -I lanplus -U "$BMC_USER" -P "$BMC_PASSWD" channel info
```

## 機殼（Chassis）

### 1. 顯示系統機架與電源子系統的高階狀態資訊

```bash
ipmitool -C 17 -H "$BMC_IP" -I lanplus -U "$BMC_USER" -P "$BMC_PASSWD" chassis status
```

### 2. 此指令會回傳開機時間（Power-On Hours）

```bash
ipmitool -C 17 -H "$BMC_IP" -I lanplus -U "$BMC_USER" -P "$BMC_PASSWD" chassis poh
```

### 3. 查詢上次系統重新啟動的原因（不支援）

```bash
ipmitool -C 17 -H "$BMC_IP" -I lanplus -U "$BMC_USER" -P "$BMC_PASSWD" chassis restart_cause
```

### 4. 設定斷電時的機架電源策略

```bash
ipmitool -C 17 -H "$BMC_IP" -I lanplus -U "$BMC_USER" -P "$BMC_PASSWD" chassis policy <state>

##
list        : return supported policies
always-on   : turn on when power is restored
previous    : return to previous state when power is restored
always-off  : stay off after power is restored
```

### 5. 檢視並變更電源狀態

```bash
ipmitool -C 17 -H "$BMC_IP" -I lanplus -U "$BMC_USER" -P "$BMC_PASSWD" chassis power

##
status : show current status
on     : power on
off    : power off
reset  : power reset
soft   : power soft
cycle  : power cycle
```

### 6. 設定下次系統重新啟動時的開機裝置

```bash
ipmitool -C 17 -H "$BMC_IP" -I lanplus -U "$BMC_USER" -P "$BMC_PASSWD" chassis bootdev <device>
```

##

目前支援的裝置：

| device | function                                                    |
| :----: | ----------------------------------------------------------- |
|  none  | 不變更開機裝置                                   |
|  pxe   | 從 pxe 開機                                               |
|  disk  | 從 BIOS 預設開機裝置開機                          |
|  safe  | 從 BIOS 預設開機裝置開機，但需要安全模式 |
|  diag  | 從診斷分割區開機                          |
| cdrom  | 從 CD/DVD 開機                                        |
|  bios  | 進入 bios 設定                                         |

若您希望讓此覆寫設定在重新開機後仍然生效，請使用 `persistent`
選項：

```bash
ipmitool -C 17 -H "$BMC_IP" -I lanplus -U "$BMC_USER" -P "$BMC_PASSWD" chassis bootdev <device> options=persistent
```

如果主機是基於 x86 CPU，您還需要注意 legacy/EFI 模式選擇器。
IPMI 預設會在啟用 legacy 模式的情況下覆寫開機來源。若要設定為
EFI 模式，請使用 `efiboot` 選項：

```bash
ipmitool -C 17 -H "$BMC_IP" -I lanplus -U "$BMC_USER" -P "$BMC_PASSWD" chassis bootdev <device> options=efiboot
```

您可以使用 `,` 來組合多個選項：

```bash
ipmitool -C 17 -H "$BMC_IP" -I lanplus -U "$BMC_USER" -P "$BMC_PASSWD" chassis bootdev <device> options=persistent,efiboot
```

若要讀取目前的開機來源覆寫設定：

```bash
ipmitool -C 17 -H "$BMC_IP" -I lanplus -U "$BMC_USER" -P "$BMC_PASSWD" chassis bootparam get 5
```

### 7. 控制面板指示燈（logo light）

```bash
ipmitool -C 17 -H "$BMC_IP" -I lanplus -U "$BMC_USER" -P "$BMC_PASSWD" chassis identify <interval>
```

## 區域網路（Local Area Network，LAN）

### 1. 輸出指定通道目前的設定資訊

```bash
ipmitool -C 17 -H "$BMC_IP" -I lanplus -U "$BMC_USER" -P "$BMC_PASSWD" lan print [<channel number>]
```

### 2. 為指定通道設定指定參數

```bash
ipmitool -C 17 -H "$BMC_IP" -I lanplus -U "$BMC_USER" -P "$BMC_PASSWD" lan set <channel> <command> <parameter>
##
Valid parameter
ipaddr <x.x.x.x>                 : Set ip for this channel
netmask <x.x.x.x>                : Set netmask for this channel
macaddr<xx:xx:xx:xx:xx:xx>       : Set the mac address for this channel
defgw ipaddr <x.x.x.x>           : Set the default gateway IP address
defgw macaddr<xx:xx:xx:xx:xx:xx> : Set the mac address of the default gateway
bakgw ipaddr <x.x.x.x>           : Set the IP address of the backup gateway
bakgw macaddr<xx:xx:xx:xx:xx:xx> : Set the IP address of the backup gateway
password <pass>                  : Set no user password
access <on|off>                  : Set the LAN channel access mode
```

## 系統事件記錄（System Event Log，SEL）

### 1. 查詢 BMC 中 SEL 及其內容的相關資訊

```bash
ipmitool -C 17 -H "$BMC_IP" -I lanplus -U "$BMC_USER" -P "$BMC_PASSWD" sel info
```

### 2. 清除 SEL 中的資訊，且此操作無法復原

```bash
ipmitool -C 17 -H "$BMC_IP" -I lanplus -U "$BMC_USER" -P "$BMC_PASSWD" sel clear
```

### 3. 刪除單一事件

```bash
ipmitool -C 17 -H "$BMC_IP" -I lanplus -U "$BMC_USER" -P "$BMC_PASSWD" sel delete <number>
```

### 4. 顯示 SEL 時鐘的目前時間

```bash
ipmitool -C 17 -H "$BMC_IP" -I lanplus -U "$BMC_USER" -P "$BMC_PASSWD" sel time get
```

## 工作階段（Session）

### 1. 顯示工作階段資訊

```bash
ipmitool -C 17 -H "$BMC_IP" -I lanplus -U "$BMC_USER" -P "$BMC_PASSWD" session info all
```

## 序列埠-over-LAN（Serial Over Lan，SOL）

### 1. 取得指定通道的 Serial-Over-LAN 設定資訊

```bash
ipmitool -C 17 -H "$BMC_IP" -I lanplus -U "$BMC_USER" -P "$BMC_PASSWD" sol info [<channel number>]
```

### 2. 將 ipmitool 切換至 Serial Over LAN 模式

```bash
ipmitool -C 17 -H "$BMC_IP" -I lanplus -U "$BMC_USER" -P "$BMC_PASSWD" sol activate
```

### 3. 在 BMC 模式下停用序列 LAN

```bash
ipmitool -C 17 -H "$BMC_IP" -I lanplus -U "$BMC_USER" -P "$BMC_PASSWD" sol deactivate
```
