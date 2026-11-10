# 實體 LED 設計支援

作者：Velumani T (Velu)，[velumanit@hcl](mailto:velumanit@hcl.com) Jayashree D
(Jayashree)，[jayashree-d@hcl](mailto:jayashree-d@hcl.com)

其他貢獻者：無

建立日期：2022 年 7 月 10 日

## 問題說明

現有的 phosphor-led-sysfs 設計會針對裝置樹（device tree）中的每個 LED 設定
公開一個服務。因此，當有多個 GPIO 腳位被設定為 LED 時，就會建立多個服務。

在某些情況下，裝置樹中會為每個 host 設定多個 LED，並需要在多 host 平台上
將它們配對成指定 host 的一組 LED。

舉例來說，電源 LED 與系統識別 LED 會合併為每個 host 一顆藍黃雙色 LED。
總共會有 4 顆 × LED 沿著電路板前緣以網格方式排列。此網格為 2 列 ×
每列 2 顆 × LED，以對應卡槽（card slot）的佈局。

根據每個 host 的狀態，藍色或黃色 LED 需要為 Blink、OFF 或 ON，
其他 LED 則需要處於 OFF 狀態。因此，雙色 LED 需要配對成一組，
並暴露到使用者空間（userspace）。

根據 phosphor-led-sysfs 應用程式目前的設計，由於它對每個 LED 公開一個服務，
要實作配對分組將會很困難。為了抽象化這個做法，同時也在單一服務下建立 LED，
因此提出一個新的應用程式。

## 背景與參考

下圖呈現目前實體 LED 設計的概觀。

```ascii
      KERNEL                 META-PHOSPHOR           PHOSPHOR-LED-SYSFS SERVICE

   +------------+  Event 1   +----------+  LED 1  +-----------------------------+
   |            |----------->|          |-------->|                             |
   |    LED 1   |  Event 2   |          |  LED 2  | Service 1 :                 |
   |            |----------->|   UDEV   |-------->|   xyz.openbmc_project.      |
   |    LED 2   |   .....    |  Events  |  ....   |        Led.Controller.LED1  |
   |            |   .....    |          |  ....   | Service 2 :                 |
   |    LED 3   |  Event N   |          |  LED N  |   xyz.openbmc_project.      |
   |            |----------->|          |-------->|        Led.Controller.LED2  |
   |    LED 4   |            +----------+         |          .......            |
   |            |                                 |          .......            |
   |  .......   |                                 | Service N :                 |
   |  .......   |                                 |   xyz.openbmc_project.      |
   |  .......   |                                 |        Led.Controller.LEDN  |
   |            |                                 +-----------------------------+
   |    LED N   |                                             |
   |            |                                             |
   +------------+                                             V
                                            +----------------------------------+
                                            |                                  |
                                            | Service 1 :                      |
                                            |    /xyz/openbmc_project/<led1>   |
                                            |                                  |
                                            | Service 2 :                      |
                                            |    /xyz/openbmc_project/<led2>   |
                                            |          .......                 |
                                            |          .......                 |
                                            |                                  |
                                            | Service N :                      |
                                            |    /xyz/openbmc_project/<ledN>   |
                                            |                                  |
                                            +----------------------------------+

                                                PHOSPHSOR-LED-SYSFS DAEMON

```

現有設計使用 sysfs 項目作為特定 LED 的 udev 事件，並觸發必要的動作，
以產生該指定服務的 dbus 物件路徑與介面。它對每個 LED 公開一個服務。

### 範例

```text
     busctl tree xyz.openbmc_project.LED.Controller.led1
     `-/xyz
       `-/xyz/openbmc_project
         `-/xyz/openbmc_project/led
           `-/xyz/openbmc_project/led/physical
             `-/xyz/openbmc_project/led/physical/led1

     busctl tree xyz.openbmc_project.LED.Controller.led2
     `-/xyz
       `-/xyz/openbmc_project
         `-/xyz/openbmc_project/led
           `-/xyz/openbmc_project/led/physical
             `-/xyz/openbmc_project/led/physical/led2

```

## 需求

- 支援 systemd 服務以觸發腳本。

- 為 phosphor-led-sysfs daemon 加入 DBUS API method。

## Proposed 設計

下圖呈現所提出實體 LED 設計的概觀。

```ascii
      KERNEL                               PHOSPHOR-LED-SYSFS DAEMON

   +------------+  Event 1   +--------+      +---------+      +--------------+
   |            |----------> |        |      |         |      |              |
   |   LED 1    |  Event 2   |  UDEV  |      |  SYSFS  |      |  EXECUTABLE  |
   |            |----------> | Events |----->| SERVICE |----->| (DBUS Method |
   |   LED 2    |  .....     |        |      |         |      |    call)     |
   |            |  Event N   |        |      |         |      |              |
   |   LED 3    |----------> +--------+      +---------+      +--------------+
   |            |                                                  |
   |            |                                                  V
   |   LED 4    |           +------------------------------------------------------+
   |            |           |  +-----------+   +---------------------------------+ |
   |  .......   |           |  |           |   | Service :                       | |
   |  .......   |           |  |  SYSTEMD  |   |                                 | |
   |  .......   |           |  |  SERVICE  |   |  /xyz/openbmc_project/led/<led1>| |
   |            |           |  |  ( LED.   |-->|  /xyz/openbmc_project/led/<led2>| |
   |   LED N    |           |  |Controller)|   |               .....             | |
   |            |           |  |           |   |               .....             | |
   |            |           |  |           |   |  /xyz/openbmc_project/led/<ledN>| |
   +------------+           |  +-----------+   +---------------------------------+ |
                             +------------------------------------------------------+

```

本文件提出一個用於實體 LED 實作的新設計。

- 實體 LED 在裝置樹的 "leds" 章節下定義，並設定對應的 GPIO 腳位。

```dts
        leds {
                compatible = "gpio-leds";

                ledName {
                        label = "devicename:color:function";
                        gpios = <&gpio ASPEED_GPIO(M, 0) GPIO_ACTIVE_HIGH>;
                };
        };
```

- 會在 phosphor-led-sysfs 儲存庫中建立 Dbus API method，介面名稱為
  **_xyz.openbmc_project.Led.Sysfs.Internal_**，用於新增或移除 LED，
  這些 LED 將來自各自的 udev LED 事件。

```text
    NAME                                   TYPE      SIGNATURE  RESULT/VALUE  FLAGS

    xyz.openbmc_project.Led.Sysfs.Internal interface -          -             -
    .AddLED                                method    s          -             -
    .RemoveLED                             method    s          -             -

```

- 會為每個 LED 觸發 udev 事件，並呼叫服務為每個 LED 事件執行
  對應的可執行檔。

- 可執行檔會呼叫 phosphor-led-sysfs 儲存庫中的 Dbus API method，
  在主服務（primary service）啟動後，將 LED 名稱作為引數從 udev 傳入。

- Phosphor-led-sysfs 將有一個單一的 systemd 服務（主服務）
  **_(xyz.openbmc_project.LED.Controller.service)_**，預設在系統啟動時執行。

- 一旦 Dbus API method 被呼叫，phosphor-led-sysfs daemon 會取得 LED 名稱
  作為參數，並將該名稱與硬體中 /sys/class/leds 路徑對應。接著，
  dbus 物件路徑與介面會在單一 systemd 服務
  (xyz.openbmc_project.LED.Controller) 下建立。

- Schema 會在 phosphor-led-sysfs daemon 中實作。會為每個 host 將一組 LED
  配對，schema 負責處理該組的顏色控制，以啟動其中一顆必須為
  Blink/ON/OFF 狀態的 LED，並將該組中剩餘的 LED 設為 OFF 狀態。

**_範例_**

```text
   SERVICE        xyz.openbmc_project.LED.Controller

   INTERFACE      xyz.openbmc_project.LED.Physical

   OBJECT PATH

     busctl tree xyz.openbmc_project.LED.Controller
     `-/xyz
       `-/xyz/openbmc_project
         `-/xyz/openbmc_project/led
           `-/xyz/openbmc_project/led/physical
             `-/xyz/openbmc_project/led/physical/led1
             `-/xyz/openbmc_project/led/physical/led2
                         ............
                         ............
             `-/xyz/openbmc_project/led/physical/ledN
```

## 已考慮的替代方案

### 做法 1

當為 LED 初始化 udev 事件時，會利用腳本將這些 LED 名稱儲存到檔案中。
Phosphor-led-sysfs 會使用 inotify 監控該檔案以處理變更。透過讀取檔案，
可取得所有 LED 名稱，並在單一 systemd 服務下為所有 LED 建立 dbus 路徑。

inotify 會持續監控該檔案，且需要在 udev 事件與 phosphor-led-sysfs
應用程式之間進行通訊。由於 udev 事件可能在 systemd 服務之前就產生，
inotify 之後並不需要監控該檔案，而且它也不是一個常用的通訊機制。

在系統啟動後，用來呼叫 Dbus method 的可執行檔是與應用程式通訊、
並在單一服務下為 LED 建立 dbus 路徑的較佳方式。

## 影響

這些變更會對 API 造成影響，因為 dbus 物件會在單一服務中暴露多個 LED
路徑。systemd 服務將使用單一名稱
**_xyz.openbmc_project.LED.Controller_**，而不再是多個服務。

## 測試

所提出的設計已在多 host 平台上經過測試。

**_手動測試_** — 實體 LED 狀態會使用 busctl 指令在硬體上測試。

**_Robot 測試_** — 實體 LED 會在 robotframework 下，使用系統 LED
測試套件（system LEDs suite）中的 REST API 進行測試。

**_單元測試_** — 所提出的程式碼可由既有已存在的單元測試涵蓋。
