# 使用 phosphor-hwmon 為 OpenBMC 提供感測器支援

本文件說明由 [phosphor-hwmon][15] 所提供的感測器。另一種做法是使用
[dbus-sensors][16] 所提供的一組應用程式。雖然兩者的設定細節不同，
但 D-Bus 的表示方式大致相同。

OpenBMC 能讓你輕鬆地為硬體新增感測器，並符合傳統 Linux HWMon 感測器格式。
OpenBMC 感測器的架構是將感測器對應到 [D-Bus][1] 物件。當感測器數值或閾值
改變時，該 D-Bus 物件會廣播 `PropertiesChanged` 訊號。判斷此訊號對系統的影響，
是其他應用程式的責任。

## D-Bus

```text
Service     xyz.openbmc_project.Hwmon-<hash>.Hwmon1
Path        /xyz/openbmc_project/sensors/<type>/<label>
Interfaces  xyz.openbmc_project.Sensor.[*], others (see below)

Signals: All properties for an interface will broadcast signal changed
```

### 路徑定義

- **<type\>** ：[HWMon 類別][2]名稱的小寫形式。
  - 範例包括 `temperature, fan_tach, voltage`。

- **<label\>** ：使用者自訂的感測器名稱。
  - 範例包括 `ambient, cpu0, fan5`

**注意**：label 必須符合 [D-Bus 規格][3]中的「Valid Object Paths」，
也就是只能包含 ASCII 字元 `[A-Z][a-z][0-9]\_`。

### Hash 定義

服務名稱中的 hash 值用於讓服務擁有一個唯一且穩定的名稱。它是一個十進位數字，
由 std::hash() 對被監控裝置的特性進行 hash 運算而得到。

## Redfish

[BMCWeb Redfish][10] 支援會回傳感測器的相關資訊。此支援依賴兩類
[ObjectMapper 關聯（associations）][11]，以便在 D-Bus 上找到所需的感測器資訊。

### 關聯類型 #1：將機殼（chassis）連結到該機殼內的所有感測器

在 Redfish 中，感測器依機殼分組。系統使用一個 ObjectMapper 關聯，將機殼連結到
該機殼內的所有感測器。這包含了所有被視為位於該機殼內的硬體感測器。
舉例來說，一個機殼可能包含兩個風扇感測器、一個環境溫度感測器，
以及一個 VRM 輸出電壓感測器。

#### D-Bus 物件路徑

此關聯將下列 D-Bus 物件路徑連結在一起：

- 機殼清單項目（inventory item）的物件路徑
- 該機殼內各感測器的感測器物件路徑清單

#### 關聯名稱

- "all_sensors"
  - 包含該機殼所有感測器的清單
- "chassis"
  - 包含與此感測器相關聯的機殼

#### 關聯範例

- /xyz/openbmc_project/inventory/system/chassis/all_sensors
  - "endpoints" 屬性包含
    - /xyz/openbmc_project/sensors/fan_tach/fan0_0
    - /xyz/openbmc_project/sensors/fan_tach/fan0_1
    - /xyz/openbmc_project/sensors/temperature/ambient
    - /xyz/openbmc_project/sensors/voltage/p0_vdn_voltage
- /xyz/openbmc_project/sensors/fan_tach/fan0_0/chassis
  - "endpoints" 屬性包含
    - /xyz/openbmc_project/inventory/system/chassis

### 關聯類型 #2：將低階硬體項目連結到它的感測器

一個感測器通常與某個低階硬體項目相關，例如風扇、電源供應器、VRM 或 CPU。
Redfish 感測器支援可以從相關聯的硬體項目取得下列資訊：

- 存在狀態（Presence，[Inventory.Item 介面][12]）
- 功能狀態（Functional state，[OperationalStatus 介面][13]）
- 製造商、型號、料號、序號 Manufacturer, Model, PartNumber, SerialNumber
  （[Decorator.Asset 介面][14]）

基於此原因，系統使用一個 ObjectMapper 關聯，將低階硬體項目連結到它的感測器。
舉例來說，一個處理器 VRM 可能具有溫度與輸出電壓感測器，或一個雙轉子風扇可能
有兩個轉速（tach）感測器。

#### D-Bus 物件路徑

此關聯將下列 D-Bus 物件路徑連結在一起：

- 低階硬體清單項目（inventory item）的物件路徑
- 與該硬體項目相關的各感測器的感測器物件路徑清單

#### 關聯名稱

- "sensors"
  - 包含此低階硬體項目感測器的清單
- "inventory"
  - 包含此感測器對應的低階硬體清單項目

#### 關聯範例

- /xyz/openbmc_project/inventory/system/chassis/motherboard/fan0/sensors
  - "endpoints" 屬性包含
    - /xyz/openbmc_project/sensors/fan_tach/fan0_0
    - /xyz/openbmc_project/sensors/fan_tach/fan0_1
- /xyz/openbmc_project/sensors/fan_tach/fan0_0/inventory
  - "endpoints" 屬性包含
    - /xyz/openbmc_project/inventory/system/chassis/motherboard/fan0

## 開發細節

感測器屬性依感測器類型標準化。閾值（Threshold）感測器包含與感測器數值
上升、下降相關的特殊屬性。[Sensor 介面][4]在它們各自的 YAML 檔案中說明。
在來源樹中的路徑位置，與 [phosphor-dbus-interfaces][5] 上層目錄下所說明的
介面完全相同。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;範例：
[openbmc/phosphor-dbus-interfaces/xyz/openbmc_project/Sensor/Threshold/Warning.yaml][6]

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;對應到 D-Bus 介面
`xyz.openbmc_project.Sensor.Threshold.Warning`

YAML 檔案中的每個 'name' 屬性都直接對應到 D-Bus 屬性。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;範例：

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;[Warning.interface.yaml][6]

```yaml
properties:
  - name: WarningHigh
    type: int64
  - name: WarningLow
    type: int64
  - name: WarningAlarmHigh
    type: boolean
  - name: WarningAlarmLow
    type: boolean
```

對應到

```text
busctl --system introspect xyz.openbmc_project.Hwmon-3301914901.Hwmon1 \
 /xyz/openbmc_project/Sensors/temperature/ambient \
 xyz.openbmc_project.Sensor.Threshold.Warning | grep property

.WarningAlarmHigh                            property  b         false        emits-change writable
.WarningAlarmLow                             property  b         false        emits-change writable
.WarningHigh                                 property  x         40000        emits-change writable
.WarningLow                                  property  x         10000        emits-change writable

```

### REST

```json
{
  "/xyz/openbmc_project/Sensors/temperature/ambient": {
    "Scale": -3,
    "Unit": "xyz.openbmc_project.Sensor.Value.Unit.DegreesC",
    "Value": 30125,
    "WarningAlarmHigh": 0,
    "WarningAlarmLow": 0,
    "WarningHigh": 40000,
    "WarningLow": 10000
  }
}
```

### 其他介面

除了 `xyz.openbmc_project.Sensor` 介面之外，感測器的 D-Bus 物件
也可能揭露下列介面：

1. `xyz.openbmc_project.Control.FanSpeed`
   - 提供一個 Target 屬性以設定風扇 RPM 值
2. `xyz.openbmc_project.Control.FanPwm`
   - 提供一個 Target 屬性以設定風扇 PWM 值
3. `xyz.openbmc_project.State.Decorator.OperationalStatus`
   - 提供一個 Functional 屬性，追蹤任何故障檔案（fault files）的狀態

### 訊號

任何屬性值改變都會在 D-Bus 上廣播一個訊號。當某個數值越過閾值時，
會額外傳送一個 D-Bus 訊號。

範例，若 WarningLow 的值為 5...

| 來源 | 目標 | propertyChanged 訊號                                                                                                  |
| ---- | --- | ------------------------------------------------------------------------------------------------------------------------ |
| 1    | 5   | "xyz.openbmc_project.Sensor.Value" : value = 5                                                                           |
| 1    | 6   | "xyz.openbmc_project.Sensor.Value" : value = 6 ,<br>"xyz.openbmc_project.Sensor.Threshold.Warning" : WarningAlarmLow = 0 |
| 5    | 6   | "xyz.openbmc_project.Sensor.Value" : value = 6                                                                           |
| 6    | 1   | "xyz.openbmc_project.Sensor.Value" : value = 1 ,<br>"xyz.openbmc_project.Sensor.Threshold.Warning" : WarningAlarmLow = 1 |

### 系統設定

在 BMC 上，每個感測器的設定都存放在一個檔案中。這些檔案可以在
`/etc/default/obmc/hwmon` 路徑下的子目錄中找到。

## 建立感測器

HWMon 感測器定義在[機器設定][7]中的 `recipes-phosphor/sensor/phosphor-hwmon%`
路徑內。`obmc/hwmon` 目錄下的子項應遵循下列其中一種路徑：

1. 系統上 `devicetree/base` 目錄路徑的子項，由核心定義。程式碼從
   OF_FULLNAME udev 環境變數取得此資訊。

2. 若裝置不在裝置樹（device tree）中，則可使用裝置路徑。

作為範例，以下是 Palmetto 環境溫度感測器的[設定][8]檔案。

```text
recipes-phosphor/sensors/phosphor-hwmon/obmc/hwmon/ahb/apb/bus@1e78a000/i2c-bus@c0/tmp423@4c.conf

```

它對應到系統上這個特定的感測器與 conf 檔案...

```text
/sys/firmware/devicetree/base/ahb/apb/bus@1e78a000/i2c-bus@c0/tmp423@4c
/etc/default/obmc/hwmon/ahb/apb/bus@1e78a000/i2c@c0/tmp423@4c.conf

```

下一個範例示範在 OpenPOWER 系統的 OCC 裝置上使用裝置路徑，而非裝置樹路徑。
請注意 conf 檔案的目錄名稱中如何以 '--' 取代 ':'。

```text
recipes-phosphor/sensors/phosphor-hwmon%/obmc/hwmon/devices/platform/gpio-fsi/fsi0/slave@00--00/00--00--00--06/sbefifo1-dev0/occ-hwmon.1.conf
```

它對應到系統上這個特定的感測器與 conf 檔案...

```text
/sys/devices/platform/gpio-fsi/fsi0/slave@00:00/00:00:00:06/sbefifo1-dev0/occ-hwmon.1
/etc/default/obmc/hwmon/devices/platform/gpio-fsi/fsi0/slave@00--00/00--00--00--06/sbefifo1-dev0/occ-hwmon.1.conf
```

為了讓感測器能揭露到 D-Bus 上，設定檔案必須描述感測器的屬性。
屬性遵循一定的格式。

```text
xxx_yyy#=value

xxx = Attribute

# = Association number (i.e. 1-n)

yyy = HWMon sensor type (i.e. temp, pwm)

```

| 屬性           | 新增的介面                             |
| -------------- | -------------------------------------- |
| LABEL          | xyz.openbmc_project.Sensor.Value       |
| WARNHI, WARNLO | xyz.openbmc_project.Threshold.Warning  |
| CRITHI, CRITLO | xyz.openbmc_project.Threshold.Critical |

HWMon 感測器類型

| [HWMon 感測器類型][2] | 類型                       |
| ---------------------- | ---------------------------- |
| temp                   | temperature                  |
| in                     | voltage                      |
| \*                     | 其他名稱皆直接對應         |

更多定義與關鍵字細節請參閱 [HWMon 介面][2]定義。

在這個 conf 範例中，tmp423 晶片連接了兩個溫度感測器。
數值必須以 10<sup>-3</sup> 攝氏度為單位描述。

```text
LABEL_temp1=ambient WARNLO_temp1=10000 WARNHI_temp1=40000

LABEL_temp2=cpu WARNLO_temp2=10000 WARNHI_temp2=80000
```

### 其他設定檔案項目

phosphor-hwmon 程式碼支援這些額外的設定檔案項目：

#### INTERVAL

讀取感測器的間隔時間，單位為微秒。若未指定，間隔為 1000000us（1 秒）。

```text
INTERVAL=1000000
```

**GAIN**、**OFFSET**

用於支援縮放後的感測器讀值，其中 value = (raw sensor reading) \* gain + offset

```text
GAIN_in3 = 5.0 #GAIN is a double OFFSET_in3 = 6 #OFFSET is an integer
```

#### MINVALUE / MAXVALUE

若找到，將用於設定 `xyz.openbmc_project.Sensor.Value` 介面上的
MinValue/MaxValue 屬性。

```text
MINVALUE_temp1 = 1

```

#### MODE

某些裝置驅動程式需要此項目，特別是 OpenPOWER OCC 驅動程式，
其實例編號（tempN_input 中的 N）是動態的，而是由另一個檔案包含已知的 ID。

例如

```text
temp26_input:29000 temp26_label:171
```

其中 26 只是 hwmon 所指派的值，而 171 對應到像是 DIMM 溫度的 IPMI 感測器值。

因此設定檔案中會有：

```text
MODE_temp26 = "label" #Tells the code to look in temp26_label LABEL_temp171 =
"dimm3_temp" #Says that temp26_input holds dimm3_temp
```

#### REMOVERCS

包含一列裝置驅動程式的 errno 值。若讀取硬體時取得這些值，
對應的感測器物件將從 D-Bus 上移除，直到能再次成功讀取為止。

```text
REMOVERCS = "5,6" #If any sensor on the device returns a 5 or 6, remove it.
REMOVERCS_temp1 = "42" #If reading temp1_input returns a 42, remove it.
```

#### TARGET_MODE

當裝置驅動程式同時揭露兩種方法時，可讓使用者選擇風扇目標模式，
即 RPM 或 PWM。

```text
TARGET_MODE = "RPM"
```

#### PWM_TARGET

對於以 PWM 控制的風扇，可用來將 pwmN 檔案對應到風扇 M。

```text
PWM_TARGET_fan0 = 1 #Use the pwm1 file to control fan 0
```

#### ENABLE

若 pwmN_enable 檔案存在，會在啟動時寫入一個值到該檔案。

```text
ENABLE_fan1 = 2 #Write a 2 to pwm1_enable
```

### 在 IPMI YAML 設定檔案中定義感測器

關於感測器項目如何定義的範例，請參考
[範例 YAML](https://github.com/openbmc/phosphor-host-ipmid/blob/master/scripts/sensor-example.yaml)

#### 如何選擇最適合的係數

根據 IPMI 規格，感測器讀值的計算方式為：

```text
y = L[(Mx + B * 10^(bExp)) * 10^(rExp)]
```

- y：IPMItool 所回報的「最終值」
- x：8 位元，無號數，編碼於 IPMI 回應封包中的讀值資料
- M：10 位元，有號整數乘數，YAML 中的 `multiplierM`
- B：10 位元，有號加法偏移量，YAML 中的 `offsetB`
- bExp：4 位元，有號數，YAML 中的 `bExp`
- rExp：4 位元，有號數，YAML 中的 `rExp`

此外，phosphor-ipmi-host 設定也支援 `scale` 屬性，適用於類比感測器，
意思是透過 DBus 讀取到的值應以 10^S 進行縮放。

由此可知，應根據可能的感測器讀值範圍與所需的解析度來選擇係數。
通常 B=0，則會有：

- 支援範圍：[0, 255 \* M \* 10^(scale - rExp)]
- 解析度：M \* 10^(scale - rExp)

舉一個具體的例子，假設一個電壓感測器回報介於 0 到 5.0V 之間。
hwmon sysfs 會將該值縮放 1000 倍，因此透過 DBus 讀取到的感測器值介於
0 到 5000 之間。一個可能的設定為：

```text
multiplierM: 20
offsetB: 0
bExp: 0
rExp: -3
scale: -3
```

因此，當 DBus 感測器值為 4986（代表 4.986V）時，phosphor-ipmi-host 會將其編碼為

```text
    x: 4986 / 20 = 249
    M: 20
    rExp: -3
```

當呼叫 ipmitool sensor list 時，該工具會取得感測器係數並計算值為：

```text
y = 20 * 249 * 10^-3 = 4.98 (V)
```

## 延伸閱讀

郵件列表 [關於感測器設計的評論][9]

[1]: https://dbus.freedesktop.org/doc/dbus-tutorial.html
[2]: https://www.kernel.org/doc/Documentation/hwmon/sysfs-interface
[3]: https://dbus.freedesktop.org/doc/dbus-specification.html#basic-types
[4]:
  https://github.com/openbmc/phosphor-dbus-interfaces/tree/master/yaml/xyz/openbmc_project/Sensor
[5]: https://github.com/openbmc/phosphor-dbus-interfaces
[6]:
  https://github.com/openbmc/phosphor-dbus-interfaces/blob/master/yaml/xyz/openbmc_project/Sensor/Threshold/Warning.interface.yaml
[7]: https://github.com/openbmc/openbmc/tree/master/meta-openbmc-machines
[8]:
  https://github.com/openbmc/openbmc/blob/master/meta-ibm/meta-palmetto/recipes-phosphor/sensors/phosphor-hwmon/obmc/hwmon/ahb/apb/bus@1e78a000/i2c-bus@c0/tmp423@4c.conf
[9]: https://lists.ozlabs.org/pipermail/openbmc/2016-November/005309.html
[10]: https://github.com/openbmc/bmcweb/blob/master/DEVELOPING.md#redfish
[11]:
  https://github.com/openbmc/docs/blob/master/architecture/object-mapper.md#associations
[12]:
  https://github.com/openbmc/phosphor-dbus-interfaces/blob/master/yaml/xyz/openbmc_project/Inventory/Item.interface.yaml
[13]:
  https://github.com/openbmc/phosphor-dbus-interfaces/blob/master/yaml/xyz/openbmc_project/State/Decorator/OperationalStatus.interface.yaml
[14]:
  https://github.com/openbmc/phosphor-dbus-interfaces/blob/master/yaml/xyz/openbmc_project/Inventory/Decorator/Asset.interface.yaml
[15]: https://github.com/openbmc/phosphor-hwmon
[16]: https://github.com/openbmc/dbus-sensors
