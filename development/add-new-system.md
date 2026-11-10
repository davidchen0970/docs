# 在 OpenBMC 中新增系統

**文件目的：** 說明如何將一個新系統加入 OpenBMC 發行版

**讀者對象：** 熟悉 OpenBMC 的程式設計師

**前置條件：** 已完成開發環境設定[文件][32]

## 總覽

**請注意：** 這份文件已不再獲得 OpenBMC
團隊的官方支援。由於其中仍包含許多有用的資訊，因此保留在此。
理想上，這份指南會成為一份獨立文件（位於
development 樹之外），並涵蓋新增新系統時必須更新的所有
不同領域。

本文件將說明以下內容：

- 回顧 Yocto 與 BitBake 的背景知識
- 建立一個新的系統層
- 填充這個新的層
- 建置新的系統並在 QEMU 中測試
- 新增感測器、LED、庫存等設定

## 背景

OpenBMC 發行版以 [Yocto](https://www.yoctoproject.org/) 為基礎。
Yocto 是一個讓開發者建立自訂 Linux 發行版的專案。
OpenBMC 使用 Yocto 來建立其嵌入式 Linux 發行版，以便執行在
各種裝置上。

Yocto 具有層級式層（hierarchical layers）的概念。當你要建置一個以
Yocto 為基礎的發行版時，你會為該發行版定義一組層。OpenBMC 使用
許多來自 Yocto 的常見層，以及一些自有的層。在 OpenBMC
[GitHub](https://github.com/openbmc/openbmc) 中，可以透過 meta-\* 目錄
找到 OpenBMC 內定義的層。

Yocto 的層是由不同檔案組合而成，這些檔案定義了該層中要納入的套件。
這些層中使用到的重要檔案類型之一是
[BitBake](https://github.com/openembedded/bitbake/blob/master/README) 配方（recipes）。

BitBake 本身就是一門功能完整的語言。在本課程中，我們只會著重於
BitBake 中理解新增系統流程所需的部分。

## 啟動初始的 BitBake

進行這項工作，你需要為開發環境配置至少 100GB 的空間，以及盡可能
多的記憶體與 CPU。OpenBMC 發行版的首次建置可能需要數小時。
不過一旦完成第一次建置，後續的建置會使用第一次建置所快取的
資料，使流程速度提升數個數量級。

所以在做其他任何事情之前，讓我們先把那次建置啟動。

請依照 OpenBMC GitHub
[頁面](https://github.com/openbmc/openbmc/blob/master/README.md#2-download-the-source)
上的指示，針對 Romulus 系統進行（步驟 2-4）。

## 建立新系統

在上述 BitBake 執行的同時，我們開始建立新的系統。
與先前的課程類似，我們會以 Romulus 作為參考。我們的
新系統將命名為 romulus-prime。

從你上方複製（clone）的 openbmc 儲存庫中，Romulus 層定義在
`meta-ibm/meta-romulus/` 內。Romulus 層的定義位於 `conf`
子目錄中。在 `conf` 下你會看到像這樣的結構：

```bash
meta-ibm/meta-romulus/conf/
├── bblayers.conf.sample
├── conf-notes.txt
├── layer.conf
├── local.conf.sample
└── machine
    └── romulus.conf
```

要建立我們新的 romulus-prime 系統，我們會從複製 romulus 層
開始。

```bash
cp -R meta-ibm/meta-romulus meta-ibm/meta-romulus-prime
```

讓我們檢視並修改新層中所需的每個檔案

1. meta-ibm/meta-romulus-prime/conf/bblayers.conf.sample

   這個檔案定義要納入 meta-romulus-prime
   發行版中的各個層。你可以看到其中有多種 Yocto 層（meta、meta-poky、
   meta-openembedded/meta-oe、……）。它也有 OpenBMC 的層，例如
   meta-phosphor、meta-openpower、meta-ibm，以及 meta-ibm/meta-romulus。

   在這個檔案中你唯一需要做的修改，就是把兩處的 meta-romulus
   改為 meta-romulus-prime。這將確保在建置你的新系統時
   會使用你的新層。

2. meta-ibm/meta-romulus-prime/conf/conf-notes.txt

   這個檔案只說明使用者在使用你的新層時預設要建置的目標。
   由於這對所有 OpenBMC 系統而言是共通的，因此保持不變。

3. meta-ibm/meta-romulus-prime/conf/layer.conf

   這個檔案的主要用途是告訴 BitBake 要到哪裡尋找配方
   （\*.bb 檔案）。配方檔案以 `.bb` 副檔名結尾，其中包含了
   各層的封裝邏輯。`.bbappend` 檔案也是配方檔案，但提供了一種
   附加到 `.bb` 檔案之上的方式。`.bbappend`
   檔案常用於對另一個層中對應的 `.bb`
   檔案新增或移除某些內容。

   你在這裡唯一需要做的修改，就是把「romulus-layer」
   尋找並取代（find/replace）為「romulus-prime-layer」

4. meta-ibm/meta-romulus-prime/conf/local.conf.sample

   這個檔案是你這個層所有本機設定值放置的地方。
   其中的文件說明寫得很好，值得一讀。

   這裡唯一需要的修改就是把 `MACHINE` 改為
   `romulus-prime`。

5. meta-ibm/meta-romulus-prime/conf/machine/romulus.conf

   這個檔案描述你機器的各項細節。你會在此定義要使用的
   核心裝置樹（device tree）、對你將納入的特定功能所做的任何覆寫
   （overrides），以及其他系統專屬的指標。這個檔案是一份很好的參考，
   讓你了解建立新系統時需要修改的各種項目（核心裝置
   樹、MRW、LED 設定、庫存存取、……）

   你要做的第一件事是把檔案重新命名為 `romulus-prime.conf`。

   **注意** 如果我們的新系統真的只是 Romulus 的一個變體，
   硬體設定完全相同，那麼我們其實只需在 Romulus 層中建立一個新的
   machine 即可。該系統的任何自訂都可以寫在新 machine 對應的
   .conf 檔案中。為了本次練習，我們假設 romulus-prime 系統至少有
   一些硬體變更，需要我們建立這個新的層。

## 建置新系統

起初它還無法編譯，但最好先確認初始設定中有幾件事
都做對了。

請務必等到本課程開頭啟動的那次建置完成後，再開始
這個步驟。

1. 修改你目前建置所用的 conf

   在你執行初始「bitbake」操作的那個 shell 中，你需要重新設定
   建置用的 conf 檔案。你可以手動複製新的檔案進去，或乾脆把它
   移除並讓 BitBake 幫你重新產生：

   ```bash
   cd ..
   rm -r ./build/conf
   . setup romulus-prime
   ```

   執行你的「bitbake」指令。

2. Nothing RPROVIDES 'romulus-prime-config'

   這會是你針對新系統執行「bitbake obmc-phosphor-image」後
   看到的第一個錯誤。

   openbmc/skeleton 儲存庫曾用於 OpenBMC 的初期原型開發。
   這個儲存庫中有一個
   [configs](https://github.com/openbmc/skeleton/tree/master/configs) 目錄。

   這類設定資料大部分已不再使用，但在完全移除之前，
   你仍需要提供它。

   由於這個儲存庫和檔案即將被淘汰，我們只需針對這個問題做個
   快速的權宜處理。

   建立設定檔案如下：

   ```bash
   cp meta-ibm/meta-romulus-prime/recipes-phosphor/workbook/romulus-config.bb meta-ibm/meta-romulus-prime/recipes-phosphor/workbook/romulus-prime-config.bb

   vi meta-ibm/meta-romulus-prime/recipes-phosphor/workbook/romulus-prime-config.bb

   SUMMARY = "Romulus board wiring"
   DESCRIPTION = "Board wiring information for the Romulus OpenPOWER system."
   PR = "r1"

   inherit config-in-skeleton

   #Use Romulus config
   do_make_setup() {
          cp ${S}/Romulus.py \
                  ${S}/obmc_system_config.py
          cat <<EOF > ${S}/setup.py
   from distutils.core import setup

   setup(name='${BPN}',
      version='${PR}',
      py_modules=['obmc_system_config'],
      )
   EOF
   }

   ```

   重新執行你的「bitbake」指令。

3. Fetcher failure for URL: file://romulus.cfg

   這是核心所需的設定檔案。你可以在這裡放入一些
   額外的核心設定參數。為了此處的目的，只要修改
   romulus-prime 使其使用 romulus.cfg 檔案即可。我們只需要在
   prepend 路徑中加入 `-prime`。

   ```bash
   vi ./meta-ibm/meta-romulus-prime/recipes-kernel/linux/linux-aspeed_%.bbappend

   FILESEXTRAPATHS_prepend_romulus-prime := "${THISDIR}/${PN}:"
   SRC_URI += "file://romulus.cfg"
   ```

   重新執行你的「bitbake」指令。

4. No rule to make target arch/arm/boot/dts/aspeed-bmc-opp-romulus-prime.dtb

   .dtb 檔案是一種裝置樹二進位 blob 檔案。它會在 Linux
   核心建置過程中，根據其對應的 .dts 檔案產生。當你引入一個新的
   OpenBMC 系統時，你需要把這些核心更新送交上游（upstream）。所連結的
   郵件
   [討論串](https://lists.ozlabs.org/pipermail/openbmc/2018-September/013260.html)
   就是這個流程的一個範例。將東西送交核心上游本身就是一門課程。
   在本課程中，我們只會直接使用 Romulus 的核心設定檔案。

   ```bash
   vi ./meta-ibm/meta-romulus-prime/conf/machine/romulus-prime.conf
   # Replace the ${MACHINE} variable in the KERNEL_DEVICETREE

   # Use romulus device tree
   KERNEL_DEVICETREE = "${KMACHINE}-bmc-opp-romulus.dtb"
   ```

   重新執行你的「bitbake」指令。

## 啟動新系統

你終於建置好新系統的映像檔了！還有更多自訂要做，但讓我們先確認
你目前有的東西能啟動。

你的新映像檔會位於你執行「bitbake」指令的位置底下的以下路徑：

```bash
./tmp/deploy/images/romulus-prime/obmc-phosphor-image-romulus-prime.static.mtd
```

把這個映像檔複製到你設定 QEMU 工作階段的地方，並重新執行啟動
QEMU 的指令（來自 [dev-environment.md][32] 的 `qemu-system-arm`
指令），將你的新檔案作為輸入。

啟動後，你應該會看到以下的登入提示：

```text
romulus-prime login:
```

成功了！你已經完成了建立、啟動與建置新系統的
基本步驟。這絕不是一個完整的系統，但你現在已經有了基礎，
可以針對新系統進行所需的各項自訂。

## 進一步的自訂

建立新系統時還有許多其他需要自訂的領域。

### 核心變更

本節說明你可以如何修改核心，以便將 OpenBMC 移植到
新的機器。裝置樹位於
<https://github.com/openbmc/linux/tree/dev-4.13/arch/arm/boot/dts>。
範例請見 [aspeed-bmc-opp-romulus.dts][1] 或類似的機器。完成核心變更
請按照以下步驟：

1. 新增新機器的裝置樹：
   - 描述 GPIO，例如 LED、FSI、gpio-keys 等。你應該從電路圖
     （schematic）取得這類資訊。
   - 描述 i2c 匯流排與裝置，通常包含各種 hwmon
     感測器。
   - 描述其他裝置，例如 uarts、mac。
   - 通常快閃記憶體的佈局（flash layout）不需要變動。只要引入
     `openbmc-flash-layout.dtsi` 即可。
2. 修改 Makefile 以建置裝置樹。
3. 關於送交修補程式到郵件列表，請參考
   [openbmc 核心文件][31]。

注意：

- 在 `dev-4.10` 中，`arch/arm/mach-aspeed/aspeed.c` 有共通且機器專屬的
  初始化程式碼，用於進行共通的初始化，並針對每台機器執行特定的設定。
  從 `dev-4.13` 分支開始，已不再有這類初始化程式碼。大部分
  的初始化都由上游的 clock 與 reset 驅動程式完成。
- 如果機器需要特定的設定（例如 uart 路由），請發信到
  [郵件列表][2] 進行討論。

### Workbook

在舊版 OpenBMC 中，有一個「workbook」用來描述機器的服務、
感測器、FRU 等。這個 workbook 是一個 python 設定檔案，會被
[skeleton][3] 中的其他服務使用。在最新版的 OpenBMC 中，skeleton 服務
大多已被 phosphor-xxx 服務取代，因此 skeleton 已被棄用。
但目前仍需 workbook 才能完成建置。

[meta-quanta][4] 是一個在 OpenBMC 樹中定義自身設定的例子，
使其不依賴 skeleton 儲存庫，雖然它的內容有點像是佔位用（dummy）。

在 [e0e69be][26] 之前，或 v2.4 標籤之前，OpenPOWER 機器使用數個
與 GPIO 相關的設定。例如，在 [Romulus.py][5] 中，
設定內容如下：

```python
GPIO_CONFIG['BMC_POWER_UP'] = \
        {'gpio_pin': 'D1', 'direction': 'out'}
GPIO_CONFIG['SYS_PWROK_BUFF'] = \
        {'gpio_pin': 'D2', 'direction': 'in'}

GPIO_CONFIGS = {
    'power_config' : {
        'power_good_in' : 'SYS_PWROK_BUFF',
        'power_up_outs' : [
            ('BMC_POWER_UP', True),
        ],
        'reset_outs' : [
        ],
    },
}
```

PowerUp 與 PowerOK 這些 GPIO 是建置所需，用以開啟機殼
電源並檢查電源狀態。

在那之後，GPIO 相關的設定從 workbook 中移除，並由
`gpio_defs.json` 取代，例如 [2a80da2][27] 為 Romulus 引入了 GPIO json 設定。

```json
{
  "gpio_configs": {
    "power_config": {
      "power_good_in": "SYS_PWROK_BUFF",
      "power_up_outs": [
        { "name": "SOFTWARE_PGOOD", "polarity": true },
        { "name": "BMC_POWER_UP", "polarity": true }
      ],
      "reset_outs": []
    }
  },

  "gpio_definitions": [
    {
      "name": "SOFTWARE_PGOOD",
      "pin": "R1",
      "direction": "out"
    },
    {
      "name": "BMC_POWER_UP",
      "pin": "D1",
      "direction": "out"
    }
  ]
}
```

每台機器都應定義類似的 json 設定來描述 GPIO
配置。

### Hwmon 感測器

Hwmon 感測器包含板載感測器（例如溫度感測器、風扇）以及 OCC
感測器。設定檔案的路徑與名稱應與裝置樹中的裝置相符。

openbmc [doc/architecture/sensor-architecture][6] 中有詳細的文件。

這裡我們以 Romulus 為例。設定檔案位於
[meta-romulus/recipes-phosphor/sensors][7]，其中包含板載感測器與
OCC 感測器，板載感測器透過 i2c，OCC 感測器則透過 FSI。

- [w83773g@4c.conf][8] 定義了 `w83773` 溫度感測器，包含 3 個
  溫度：

```bash
  LABEL_temp1 = "outlet"
  ...
  LABEL_temp2 = "inlet_cpu"
  ...
  LABEL_temp3 = "inlet_io"
```

這個裝置在其裝置樹中定義為 [w83773g@4c][9]。當 BMC 啟動時，
udev 規則會啟動 `phosphor-hwmon`，它會根據其 sysfs 屬性，在下列 DBus
物件上建立溫度感測器。

```bash
  /xyz/openbmc_project/sensors/temperature/outlet
  /xyz/openbmc_project/sensors/temperature/inlet_cpu
  /xyz/openbmc_project/sensors/temperature/inlet_io
```

- [pwm-tacho-controller@1e786000.conf][10] 定義風扇，設定與上述類似，
  差別在於它會建立 `fan_tach` 感測器。
- [occ-hwmon.1.conf][11] 定義主 CPU 的 occ hwmon 感測器。這個
  設定稍有不同，它要告訴 `phosphor-hwmon` 去讀取 label，而不是直接取得
  感測器的索引，因為 CPU 核心與 DIMM 可能是動態變化的，例如 CPU 核心
  可能被停用、DIMM 可能被拔出。

```bash
  MODE_temp1 = "label"
  MODE_temp2 = "label"
  ...
  MODE_temp31 = "label"
  MODE_temp32 = "label"
  LABEL_temp91 = "p0_core0_temp"
  LABEL_temp92 = "p0_core1_temp"
  ...
  LABEL_temp33 = "dimm6_temp"
  LABEL_temp34 = "dimm7_temp"
  LABEL_power2 = "p0_power"
  ...
```

- `MODE_temp* = "label"` 表示如果它看到 `tempX`，就應讀取 label，
  而 label 就是感測器 id。
- 而 `LABEL_temp* = "xxx"` 則告訴對應感測器 id 的感測器名稱。
- 例如，如果在 sysfs 中 `temp1_input` 是 37000 且 `temp1_label` 是 91，
  `phosphor-hwmon` 就知道 `temp1_input` 是感測器 id 91，也就是
  `p0_core0_temp`，因此它會建立
  `/xyz/openbmc_project/sensors/temperature/p0_core0_temp`，感測值為
  37000。
- 對 Romulus 而言，電源感測器不需要讀取 label，因為系統上所有的
  電源都可取得。
- 對 Witherspoon 而言，電源感測器與溫度感測器類似，它要告訴 hwmon
  去讀取 `function_id`，而不是直接取得感測器的索引。

### LED

LED 牽涉到幾個部分。

1. 在核心 dts 中，應描述 LED，例如 [romulus dts][12] 描述了 3 個
   LED：`fault`、`identify` 和 `power`。

   ```dts
    leds {
      compatible = "gpio-leds";

      fault {
        gpios = <&gpio ASPEED_GPIO(N, 2) GPIO_ACTIVE_LOW>;
      };

      identify {
        gpios = <&gpio ASPEED_GPIO(N, 4) GPIO_ACTIVE_HIGH>;
      };

      power {
        gpios = <&gpio ASPEED_GPIO(R, 5) GPIO_ACTIVE_LOW>;
      };
    };
   ```

2. 在機器層中，應透過 yaml 設定 LED 來描述它的運作方式，例如
   [Romulus led yaml][28]：

   ```yaml
   bmc_booted:
     power:
       Action: "Blink"
       DutyOn: 50
       Period: 1000
       Priority: "On"
   power_on:
     power:
       Action: "On"
       DutyOn: 50
       Period: 0
       Priority: "On"
   ```

   它告訴 LED 管理器在 BMC 準備好並完成啟動時將 `power` LED 設為閃爍，
   並在主機通電時將它設為恆亮。3. 在執行期，LED 管理器會根據上述
   yaml 設定自動將 LED 設為開/關/閃爍。4. LED 可以透過
   /xyz/openbmc_project/led/ 手動存取，例如

3. 取得 identify LED 狀態：

   ```bash
    curl -b cjar -k https://$bmc/xyz/openbmc_project/led/physical/identify
   ```

4. 將 identify LED 設為閃爍：

   ```bash
    curl -b cjar -k -X PUT -H "Content-Type: application/json" -d '{"data": "xyz.openbmc_project.Led.Physical.Action.Blink" }' https://$bmc/xyz/openbmc_project/led/physical/identify/attr/State
   ```

5. 當發生與某個 FRU 相關的錯誤時，會在 logging 中建立一筆事件紀錄，
   帶有 CALLOUT 路徑。[phosphor-fru-fault-monitor][29] 會監控這些紀錄：
   - 當產生帶有 CALLOUT 路徑的紀錄時，assert 相關的 fault LED 群組；
   - 當該紀錄被標記為「Resolved」或被刪除時，de-assert 相關的 fault
     LED 群組。

**注意**：這份 yaml 設定可以由
[phosphor-mrw-tools][13] 從其 MRW 自動產生，請見
[Witherspoon 範例][14]。

### 庫存與其他感測器

庫存、其他感測器（例如 CPU/DIMM 溫度）與 FRU 都定義在
ipmi 的 yaml 設定檔案中。

例如 [meta-romulus/recipes-phosphor/ipmi][15]

- `romulus-ipmi-inventory-map` 定義一般的庫存，例如 CPU、記憶體、
  主機板。
- `phosphor-ipmi-fru-properties` 定義庫存的額外屬性。
- `phosphor-ipmi-sensor-inventory` 定義來自 IPMI 的感測器。
- `romulus-ipmi-inventory-sel` 定義用於 IPMI SEL 的庫存。

對於 inventory map 與 fru-properties，不同系統之間是類似的，
你可以參考這個範例為你的系統建立一份。

對於 ipmi-sensor-inventory，來自 IPMI 的感測器在各系統之間不同，
所以你需要定義自己的感測器，例如

```yaml
0x08:
  sensorType: 0x07
  path: /org/open_power/control/occ0
  ...
0x1e:
  sensorType: 0x0C
  path: /system/chassis/motherboard/dimm0
  ...
0x22:
  sensorType: 0x07
  path: /system/chassis/motherboard/cpu0/core0
```

第一個值 `0x08`、`0x1e` 和 `0x22` 是 IPMI 的感測器 id，這是在
MRW 中定義的。你應該依照系統的 MRW 來定義上述設定。

**注意**：這些 yaml 設定可以由
[phosphor-mrw-tools][13] 從其 MRW 自動產生，請見
[Witherspoon 範例][14]。

### 風扇

[phosphor-fan-presence][16] 管理所有與風扇相關的服務：

- `phosphor-fan-presence` 檢查風扇是否存在，在庫存中建立風扇的 DBus
  物件並更新 `Present` 屬性。
- `phosphor-fan-monitor` 檢查風扇是否正常運作，並更新風扇 DBus
  物件的 `Functional` 屬性。
- `phosphor-fan-control` 透過根據條件（例如溫度）設定風扇轉速目標
  來控制風扇速度。
- `phosphor-cooling-type` 透過設定
  `/xyz/openbmc_project/inventory/system/chassis` 物件的屬性，
  檢查並設定系統是氣冷還是水冷。

上述所有服務都是可設定的，例如透過 yaml 設定。所以在將
OpenBMC 移植到新機器時，應撰寫機器專屬的設定。

以 Romulus 為例，它是氣冷，有 3 個風扇，且沒有 GPIO
存在偵測。

#### 風扇存在偵測

Romulus 沒有風扇的 GPIO 偵測，所以它會檢查風扇 tach 感測器：

```yaml
- name: fan0
  path: /system/chassis/motherboard/fan0
  methods:
    - type: tach
      sensors:
        - fan0
```

這份 yaml 設定表示

- 它應在庫存中建立 `/system/chassis/motherboard/fan0` 物件。
- 它應檢查 fan0 tach 感測器（`/sensors/fan_tach/fan0`）來設定
  fan0 物件上的 `Present` 屬性。

#### 風扇監控

Romulus 風扇使用 pwm 來控制風扇轉速，pwm 範圍為 0 到 255，
而風扇轉速範圍為 0 到約 7000。所以它需要一個因子（factor）與偏移
（offset）來將 pwm 對應到風扇轉速：

```yaml
- inventory: /system/chassis/motherboard/fan0
  allowed_out_of_range_time: 30
  deviation: 15
  num_sensors_nonfunc_for_fan_nonfunc: 1
  sensors:
    - name: fan0
      has_target: true
      target_interface: xyz.openbmc_project.Control.FanPwm
      factor: 21
      offset: 1600
```

這份 yaml 設定表示：

1. 它應使用 `FanPwm` 作為 tach 感測器的目標介面。
2. 它應將預期風扇轉速計算為 `target * 21 + 1600`。
3. 偏差（deviation）為 `15%`，所以如果風扇轉速超出預期範圍超過
   30 秒，fan0 就應被設為無法運作（non-functional）。

#### 風扇控制

風扇控制服務需要 4 份 yaml 設定檔案：

- `zone-condition` 定義散熱區（cooling zone）的條件。Romulus
  永遠是氣冷，所以這份設定很簡單，只是根據 cooling type 屬性定義一個
  `air_cooled_chassis` 條件。

```yaml
  - name: air_cooled_chassis
   type: getProperty
   properties:
     - property: WaterCooled
       interface: xyz.openbmc_project.Inventory.Decorator.CoolingType
       path: /xyz/openbmc_project/inventory/system/chassis
       type: bool
       value: false
```

- `zone-config` 定義散熱區。Romulus 只有一個區：

```yaml
zones:
  - zone: 0
    full_speed: 255
    default_floor: 195
    increase_delay: 5
    decrease_interval: 30
```

它定義了風扇的該區全速與預設下限轉速，所以當處於全速時，
風扇 pwm 會被設為 255，當風扇處於預設下限轉速時則設為 195。

- `fan-config` 定義哪些風扇在哪個散熱區中被控制，以及應使用哪個
  目標介面，例如下方的 yaml 設定定義 fan0 應在 zone0 中被控制，且它
  應使用 `FanPwm` 介面。

```yaml
  - inventory: /system/chassis/motherboard/fan0
   cooling_zone: 0
   sensors:
     - fan0
   target_interface: xyz.openbmc_project.Control.FanPwm
   ...
```

- `events-config` 定義各種事件及其處理程式（handler），例如在哪個
  溫度下應設定哪些風扇目標。這份設定較為複雜，
  [範例 event yaml][17] 提供了文件與範例。Romulus 範例：

```yaml
- name: set_air_cooled_speed_boundaries_based_on_ambient
  groups:
    - name: zone0_ambient
      interface: xyz.openbmc_project.Sensor.Value
      property:
        name: Value
        type: int64_t
  matches:
    - name: propertiesChanged
  actions:
    - name: set_floor_from_average_sensor_value
      map:
        value:
          - 27000: 85
          - 32000: 112
          - 37000: 126
          - 40000: 141
        type: std::map<int64_t, uint64_t>
    - name: set_ceiling_from_average_sensor_value
      map:
        value:
          - 25000: 175
          - 27000: 255
        type: std::map<int64_t, uint64_t>
```

上述 yaml 設定了在 `zone0_ambient` 不同溫度下的風扇下限與上限轉速。
例如：

1. 當溫度低於 27 度 C 時，下限轉速（pwm）應被設為 85。
2. 當溫度介於 27 到 32 度 C 之間時，下限轉速（pwm）應被設為 112，依此類推。

有了上述設定，phosphor-fan 就會針對該機器以特別設定的方式
執行風扇的存在/監控/控制邏輯。

**注意**：Romulus 的風扇很簡單。要了解更複雜的範例，請參考
[Witherspoon 風扇設定][18]。以下是 Witherspoon 風扇設定的
額外功能：

- 它會檢查 GPIO 來偵測風扇存在與否。
- 它會檢查 GPIO 來判斷系統是氣冷還是水冷。
- 它的風扇控制中有更多感測器與更多事件。

### GPIO

本節主要著重於裝置樹中應被監控的 GPIO。例如：

- 某個 GPIO 可能代表主機 checkstop 的信號。
- 某個 GPIO 可能代表按鈕按下。
- 某個 GPIO 可能代表某個裝置是否連接。

它們被歸類為用於檢查裝置存在與否的 `phosphor-gpio-presence`，以及
用於監控某個 GPIO 的 `phosphor-gpio-monitor`。

#### 裝置樹中的 GPIO

所有要監控的 GPIO 都應在裝置樹中描述。例如

```dts
  gpio-keys {
    compatible = "gpio-keys";

    checkstop {
      label = "checkstop";
      gpios = <&gpio ASPEED_GPIO(J, 2) GPIO_ACTIVE_LOW>;
      linux,code = <ASPEED_GPIO(J, 2)>;
    };
    id-button {
      label = "id-button";
      gpios = <&gpio ASPEED_GPIO(Q, 7) GPIO_ACTIVE_LOW>;
      linux,code = <ASPEED_GPIO(Q, 7)>;
    };
  };
```

以下程式碼描述了兩個 GPIO 按鈕，一個是 `checkstop`，另一個是
`id-button`，其中按鈕碼（key code）由 [aspeed-gpio.h][24] 計算而來：

```bash
#define ASPEED_GPIO_PORT_A 0
#define ASPEED_GPIO_PORT_B 1
...
#define ASPEED_GPIO_PORT_Y 24
#define ASPEED_GPIO_PORT_Z 25
#define ASPEED_GPIO_PORT_AA 26
...

#define ASPEED_GPIO(port, offset) \
  ((ASPEED_GPIO_PORT_##port * 8) + offset)
```

#### GPIO 存在偵測

Witherspoon 與 Zaius 都有 gpio 存在偵測的範例。

- [Witherspoon][19]：

```bash
INVENTORY=/system/chassis/motherboard/powersupply0
DEVPATH=/dev/input/by-path/platform-gpio-keys-event
KEY=104
NAME=powersupply0
DRIVERS=/sys/bus/i2c/drivers/ibm-cffps,3-0069
```

它檢查 `powersupply0` 存在與否的 GPIO 按鈕碼 104，建立庫存
物件並綁定或解除綁定驅動程式。

- [Zaius][20]：

```bash
INVENTORY=/system/chassis/pcie_card_e2b
DEVPATH=/dev/input/by-path/platform-gpio-keys-event
KEY=39
NAME=pcie_card_e2b
```

它檢查 `pcie_card_e2b` 存在與否的 GPIO 按鈕碼 39，並建立庫存
物件。

#### GPIO 監控

GPIO 監控的典型用途是監控來自主機的 checkstop 事件，
或按鈕按下。

- [checkstop monitor][21] 是 OpenPOWER 機器的共通服務。

```bash
DEVPATH=/dev/input/by-path/platform-gpio-keys-event
KEY=74
POLARITY=1
TARGET=obmc-host-crash@0.target
```

預設它監控 GPIO 按鈕碼 74，如果被觸發，它會告訴 systemd
啟動 `obmc-host-crash@0.target`。對於使用不同 GPIO 腳位作為
checkstop 的系統，只需在 meta-machine 層中指定自己的設定檔案，
即可覆寫預設值。例如 [Zaius 的 checkstop 設定][22]。
**注意**：當按鈕被按下時，`phosphor-gpio-monitor` 會啟動目標
單元（unit）然後結束。

- [id-button monitor][23] 是 Romulus 上一個監控 ID 按鈕按下的
  範例服務。

```bash
DEVPATH=/dev/input/by-path/platform-gpio-keys-event
KEY=135
POLARITY=1
TARGET=id-button-pressed.service
EXTRA_ARGS=--continue
```

它監控按鈕按下的 GPIO 按鈕碼 135，並啟動
`id-button-pressed.service`，該服務透過設定 identify LED 群組的
`Assert` 屬性來處理這個事件。**注意**：它有一個額外的參數
`--continue`，告訴 `phosphor-gpio-monitor` 在按鈕按下時不要結束，
而是繼續執行。

[1]:
  https://github.com/openbmc/linux/blob/dev-4.13/arch/arm/boot/dts/aspeed-bmc-opp-romulus.dts
[2]: https://lists.ozlabs.org/listinfo/openbmc
[3]: https://github.com/openbmc/skeleton
[4]:
  https://github.com/openbmc/openbmc/tree/master/meta-quanta/meta-q71l/recipes-phosphor/workbook
[5]: https://github.com/openbmc/skeleton/blob/master/configs/Romulus.py
[6]:
  https://github.com/openbmc/docs/blob/master/architecture/sensor-architecture.md
[7]:
  https://github.com/openbmc/openbmc/tree/master/meta-ibm/meta-romulus/recipes-phosphor/sensors
[8]:
  https://github.com/openbmc/openbmc/blob/298c4328fd20fcd7645da1565c143b1b668ef541/meta-ibm/meta-romulus/recipes-phosphor/sensors/phosphor-hwmon/obmc/hwmon/ahb/apb/i2c%401e78a000/i2c-bus%40440/w83773g%404c.conf
[9]:
  https://github.com/openbmc/linux/blob/aca92be80c008bceeb6fb62fd1d450b5be5d0a4f/arch/arm/boot/dts/aspeed-bmc-opp-romulus.dts#L208
[10]:
  https://github.com/openbmc/openbmc/blob/298c4328fd20fcd7645da1565c143b1b668ef541/meta-ibm/meta-romulus/recipes-phosphor/sensors/phosphor-hwmon/obmc/hwmon/ahb/apb/pwm-tacho-controller%401e786000.conf
[11]:
  https://github.com/openbmc/openbmc/blob/298c4328fd20fcd7645da1565c143b1b668ef541/meta-ibm/meta-romulus/recipes-phosphor/sensors/phosphor-hwmon/obmc/hwmon/devices/platform/gpio-fsi/fsi0/slave%4000--00/00--00--00--06/sbefifo1-dev0/occ-hwmon.1.conf
[12]:
  https://github.com/openbmc/linux/blob/aca92be80c008bceeb6fb62fd1d450b5be5d0a4f/arch/arm/boot/dts/aspeed-bmc-opp-romulus.dts#L42
[13]: https://github.com/openbmc/phosphor-mrw-tools
[14]:
  https://github.com/openbmc/openbmc/blob/764b88f4056cc98082e233216704e94613499e64/meta-ibm/meta-witherspoon/conf/distro/openbmc-witherspoon.conf#L4
[15]:
  https://github.com/openbmc/openbmc/tree/master/meta-ibm/meta-romulus/recipes-phosphor/ipmi
[16]: https://github.com/openbmc/phosphor-fan-presence
[17]:
  https://github.com/openbmc/phosphor-fan-presence/blob/master/control/example/events.yaml
[18]:
  https://github.com/openbmc/openbmc/tree/master/meta-ibm/meta-witherspoon/recipes-phosphor/fans
[19]:
  https://github.com/openbmc/openbmc/blob/master/meta-ibm/meta-witherspoon/recipes-phosphor/gpio/phosphor-gpio-monitor/obmc/gpio/phosphor-power-supply-0.conf
[20]:
  https://github.com/openbmc/openbmc/blob/master/meta-ingrasys/meta-zaius/recipes-phosphor/gpio/phosphor-gpio-monitor/obmc/gpio/phosphor-pcie-card-e2b.conf
[21]:
  https://github.com/openbmc/openbmc/blob/master/meta-openpower/recipes-phosphor/host/checkstop-monitor.bb
[22]:
  https://github.com/openbmc/openbmc/blob/master/meta-ingrasys/meta-zaius/recipes-phosphor/host/checkstop-monitor/obmc/gpio/checkstop
[23]:
  https://github.com/openbmc/openbmc/tree/master/meta-ibm/meta-romulus/recipes-phosphor/gpio
[24]:
  https://github.com/openbmc/linux/blob/dev-4.13/include/dt-bindings/gpio/aspeed-gpio.h
[26]:
  https://github.com/openbmc/openbmc/commit/e0e69beab7c268e4ad98972016c78b0d7d5769ac
[27]:
  https://github.com/openbmc/openbmc/commit/2a80da2262bf13aa1ddb589cf3f2b672d26b0975
[28]:
  https://github.com/openbmc/openbmc/blob/3cce45a96f0416b4c3d8f2b698cb830662a29227/meta-ibm/meta-romulus/recipes-phosphor/leds/romulus-led-manager-config/led.yaml
[29]: https://github.com/openbmc/phosphor-led-manager/tree/master/fault-monitor
[31]: https://github.com/openbmc/docs/blob/master/kernel-development.md
[32]: https://github.com/openbmc/docs/blob/master/development/dev-environment.md
