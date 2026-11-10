# OpenBMC 的 LED 支援

本文件說明如何依據 OpenBMC 的 [LED 架構][led d-bus readme]文件，為你的機器
新增 LED 支援。對於支援使用 MRW 的機器，LED 群組管理會自動完成，不在本文件的
討論範圍之內。

## D-Bus

```text
Service     xyz.openbmc_project.LED.GroupManager
Path        /xyz/openbmc_project/led/groups/<label>
Interfaces  xyz.openbmc_project.Led.Group

Signals: none
Attribute: Asserted (boolean)
```

## REST

```text
PUT /xyz/openbmc_project/led/groups/<group>/attr/Asserted
```

LED 群組的狀態可以藉由將 Asserted 值設為布林 0 或 1 來變更。在下面的範例中，
lamp_test 群組正在被設定為啟動（assert）...

```bash
curl -b cjar -k -X PUT -H "Content-Type: application/json" -d '{"data":  1}' \
    https://${bmc}/xyz/openbmc_project/led/groups/lamp_test/attr/Asserted
```

## REDFISH

### Resource identify Operation

從 Redfish v2020.3 開始，Redfish 資源具有 **LocationIndicatorActive** 屬性，
可用於辨識特定的資源。這僅適用於 LED 辨識（Identify）操作。

所有適用的 Inventory D-Bus 物件都會有一個指向 LED Group D-Bus 物件的前向
association 對應，也就是：

```text
 - identify_led_group
```

所有適用的 LED Group D-Bus 物件都會有一個對應到 inventory D-Bus 物件的
association，也就是：

```text
 - identify_inventory_object
```

當對資源執行 `LocationIndicatorActive` 的 GET 操作時，bmcweb 會進行以下操作：

- 在 Inventory D-Bus 物件上尋找 `identify_led_group` association
- 若找到，則從 `identify_led_group` 所指向的 D-Bus 物件讀取 `asserted` 屬性

當對資源執行 `LocationIndicatorActive` 的 PATCH 操作時，bmcweb 會進行以下操作：

- 在 Inventory D-Bus 物件上尋找 `identify_led_group` association
- 若找到，則在 `identify_led_group` 所指向的 D-Bus 物件上設定 `asserted` 屬性

### Representing resource faults

若要表示資源的故障狀態，會使用 Redfish 的 `Health` 屬性。所有適用的 Inventory
D-Bus 物件都會有一個指向 LED Group D-Bus 物件的前向 association 對應，也就是：

```text
 - fault_led_group
```

所有適用的 LED Group D-Bus 物件都會有一個對應到 inventory D-Bus 物件的
association，也就是：

```text
 - fault_inventory_object
```

當偵測到元件故障時，負責的程式碼會在 Inventory D-Bus 物件上將
`xyz.openbmc_project.State.Decorator.OperationalStatus` 的 `Functional` 屬性
設為 `false`。

當 LED manager 收到 PropertyChanged 信號時，它會執行以下動作：

- 在 Inventory D-Bus 物件上尋找 `fault_led_group` association
- 根據 `Functional` 的值開啟/關閉 fault LED 群組；`false` 代表開啟 LED 群組，
  `true` 代表關閉 LED 群組

- bmcweb 已具有一項設計，會尋找 `Functional` 屬性並設定 `Health` 屬性。

## Development Details

LED 操作有兩個重要的層級：實體層（physical）與邏輯層（logical）。LED Group
Manager 會與實體的 LED Manager 溝通，以驅動實體的 LED。邏輯群組定義在機器的
led.yaml 檔案中。LED Group manager 會讀取此檔案，並為群組中的個別 LED 建立
D-Bus/REST 介面。

### Defining the physical LED

實體 LED 的接線定義在機器的 [device tree][kernel arm dts]的 `leds` 區塊中。
請參考 [Palmetto DTS][palmetto dts led]作為範例。

_在 device tree 中新增一個帶有對應 gpio 接腳的 fault LED..._

```dts
  leds {
    compatible = "gpio-leds";

    fault {
      gpios = <&gpio ASPEED_GPIO(N, 2) GPIO_ACTIVE_LOW>;
    };
  }
```

_核心接著會建立..._

```text
 ls -l /sys/class/leds/fault/
total 0
-rw-r--r--    1 root     root          4096 Jun 21 20:04 brightness
lrwxrwxrwx    1 root     root             0 Jun 21 20:29 device -> ../../../leds
-r--r--r--    1 root     root          4096 Jun 21 20:29 max_brightness
drwxr-xr-x    2 root     root             0 Jun 21 20:29 power
lrwxrwxrwx    1 root     root             0 Jun 21 20:04 subsystem -> ../../../../../class/leds
-rw-r--r--    1 root     root          4096 Jun 21 20:04 trigger
-rw-r--r--    1 root     root          4096 Jun 21 20:04 uevent
```

### Defining Groups

一個 LED Group 可以包含零個或多個 LED，並定義在機器的
[led.yaml][led yaml]中。預設的檔案很可能需要依照你機器的配置進行調整。自訂的
yaml 檔案會放置於該機器專屬的 Yocto 位置。舉例來說：

```text
meta-ibm/meta-palmetto/recipes-phosphor/leds/palmetto-led-manager-config/led.yaml
```

yaml 檔案中的父屬性會在 `/xyz/openbmc_project/led/groups/` 之下被建立。子屬性
則需要對應到 `/sys/class/leds` 中的一個 LED 名稱。

在下方的範例中，會建立兩個 URI：
`/xyz/openbmc_project/led/groups/enclosure_fault` 與
`/xyz/openbmc_project/led/groups/lamp_test`。兩者都作用於同一個實體 LED
`fault`，但作用方式不同。若已建立實體 `power` LED，lamp_test 也會對它送出
閃爍（blink）信號。

```yaml
EnclosureFault:
  fault:
    Action: "On"
    DutyOn: 50
    Period: 0
lamp_test:
  fault:
    Action: "Blink"
    DutyOn: 20
    Period: 100
  power:
    Action: "Blink"
    DutyOn: 20
    Period: 100
```

### Required Groups

OpenBMC 架構要求必須建立特定的 LED Group，並記錄在 [D-Bus
interface][led d-bus readme]中。

## Yocto packaging

1. 建立一份客製化的 LED manager 檔案

   例如：
   `meta-ibm/meta-romulus/recipes-phosphor/leds/romulus-led-manager-config-native.bb`

   ```bitbake
       SUMMARY = "Phosphor LED Group Management for Romulus"
       PR = "r1"

       inherit native
       inherit obmc-phosphor-utils
       inherit obmc-phosphor-license

       PROVIDES += "virtual/phosphor-led-manager-config-native"

       SRC_URI += "file://led.yaml"
       S = "${WORKDIR}"

       # Overwrites the default led.yaml
       do_install() {
           SRC=${S}
           DEST=${D}${datadir}/phosphor-led-manager
           install -D ${SRC}/led.yaml ${DEST}/led.yaml
       }
   ```

2. 在 conf 檔案中變更你機器所使用的 led-manager 偏好供應商（preferred provider）

   例如：`meta-ibm/meta-romulus/conf/machine/romulus.conf`

   `PREFERRED_PROVIDER_virtual/phosphor-led-manager-config-native = "romulus-led-manager-config-native"`

[led d-bus readme]:
  https://github.com/openbmc/phosphor-dbus-interfaces/blob/master/yaml/xyz/openbmc_project/Led/README.md
[led yaml]: https://github.com/openbmc/phosphor-led-manager/blob/master/led.yaml
[kernel arm dts]:
  https://github.com/openbmc/linux/tree/dev-4.19/arch/arm/boot/dts
[palmetto dts led]:
  https://github.com/openbmc/linux/blob/dev-4.19/arch/arm/boot/dts/aspeed-bmc-opp-palmetto.dts#L45
