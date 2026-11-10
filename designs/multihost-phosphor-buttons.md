# 多主機前面板 phosphor buttons 介面

作者：Velumani T(velu), [velumanit@hcl](mailto:velumanit@hcl.com) Naveen Moses
S(naveen.moses), [naveen.mosess@hcl.com](mailto:naveen.mosess@hcl.com)

其他貢獻者：

建立日期：2021 年 8 月 3 日

## 問題描述

Phosphor buttons 模組目前提供選項，可監控 power 與 reset 按鈕的 gpio
事件並觸發電源事件處理常式。

phosphor-buttons 目前僅支援按壓式（push type）按鈕。某些硬體需要支援
非按壓式的不同形式輸入，例如開關（switch）／多工器（MUX）。如果我們能
在 phosphor button 介面與事件處理中加入選項，以支援額外的輸入類型，
將會非常有幫助。

目前處理常式事件僅以監控 gpio 事件（power 與 reset）作為輸入。可能會有
這種情況：我們需要建立一個按鈕介面，它監控的是非 gpio 事件，並觸發按鈕
動作，例如基於 dbus 屬性變更的事件。

## 需求

此功能是為了支援額外的 phosphor button 介面，對應到現有 power 與 reset
按鈕介面之外、前面板上可用的平台專屬硬體按鈕／MUX／開關。

## 背景與參考

BMC 的前面板一般會有 power button、reset button 等按鈕，用來監控這些
按鈕並觸發動作的程式碼已有支援。

```text
   +----------------------------------------------+
   |                                              |
   |       Front panel                            |
   |                                              |
   |   +--------------+      +--------------+     |
   |   |              |      |              |     |
   |   | power button |      | reset button |     |
   |   |              |      |              |     |
   |   +--------------+      +--------------+     |
   |                                              |
   |                                              |
   |                                              |
   |                                              |
   +----------------------------------------------+
```

平台專屬的前面板可能包含額外的硬體按鈕或 MUX 開關。

以多主機平台 yosemite-V2 為例，它的前面板上有一個主機選擇器開關
（host selector switch）MUX 作為額外的按鈕。

```text
+---------------------------+                                      +------------+
|                           |                                      |            |
|       yv2 front panel     |           power/           ---------+|  host 1    |
|                           |           reset/serial mux |         |            |
|     +----------------+    |           control          |         +------------+
|     |                |    |                            |
|     |  power button  |-+  |                            |         +------------+
|     |                | |  |                            |         |            |
|     +----------------+ |  |                            |--------+|  host 2    |
|                        |  |                            |         |            |
|                        |  |                            |         +------------+
|                        |  |     +---------------+      |
|     +----------------+ |  |     |    host       |      |         +-------------+
|     |                | |  |     |  selector     |------+         |             |
|     |  reset button  --|--------+   switch      |      |---------+  host 3     |
|     |                |    |     +---------------+      |         |             |
|     +----------------+    |                            |         +-------------+
|                           |                            |
|                           |                            |          +------------+
|                           |                            |          |            |
|     +----------------+    |                            |----------+ host 4     |
|     | serial console +---------------------------------|          |            |
|     |    mux switch  |    |                            |          +------------+
|     |                |    |                            |
|     +----------------+    |                            |           +-----------+
|                           |                            +-----------|   BMC     |
|                           |                                        |           |
+---------------------------+                                        +-----------+

```

yv2 的前面板上有兩個額外的硬體按鈕／開關。

## 1.主機選擇器開關（selector switch）

這個主機選擇器開關／選擇器開關有 4 個 gpio 作為輸入連接，並根據 gpio
狀態值推導出主機選擇器開關的值。可能的開關值範圍為 0 到 10。gpio 值與
主機選擇器值之間的特定對應關係，可以存放於 gpio json 設定中。

yosemiteV2 平台有 1 個 bmc 與 4 個主機，且只有一個 power button 與一個
reset button。

舉例來說，當按下 power button 時，主機選擇器開關位置的值（1 到 10）決定
了 power button 的動作是要對 bmc 觸發，還是對其中某個主機觸發。

這個選項是為了讓多個主機共用同一個 power button 與 reset button。

power／reset 按鈕的按下事件會根據主機選擇器開關位置的值來觸發。

### 範例

host selector position value = 0,bmc is mapped to power and reset buttons host
selector position value = 1,host 1 is mapped to power and reset buttons host
selector position value = 2,host 2 is mapped to power and reset buttons host
selector position value = 3,host 3 is mapped to power and reset buttons host
selector position value = 4,host 4 is mapped to power and reset buttons

## 2.Serial console MUX 開關

每個主機和 bmc 也都有一個 uart console，透過一個 MUX 開關連接。

在任何時刻，只能透過 console 存取其中一個裝置（bmc 或某個主機），這取決
於 UART MUX 開關的設定。

根據主機選擇器開關的位置，一個獨立的處理常式介面可以設定選取相對應的
serial console 裝置。

## 3.OCP debug card 主機選擇器按鈕

OCP debug card 可以透過 usb 連接到 yosemitev2 以進行除錯。

這是一個延伸介面，有三個按鈕：1.Power button 2.reset button 3.host
selector button

Power 與 reset 按鈕對應到前面板相同的 gpio，因此不需要個別的事件處理。

主機選擇器按鈕的用途與 yosemitev2 前面板上的主機選擇器開關相同。debug
card 的主機選擇器按鈕（按壓式按鈕）所使用的 gpio 線，不同於前面板主機
選擇器開關的 gpio。每當按下 debug 主機選擇器按鈕時，主機選擇器開關位置
的值應遞增，直到 MaxPosition 為止。若主機選擇器開關的值超過 MaxPosition，
則應重設為零。

## 建議設計

phosphor button 介面類別需要三個高階層級的變更。

1. 加入支援，可新增監控多個 gpio 事件並以單一事件處理常式處理的按鈕
   介面。

2. 加入支援，可基於 dbus 屬性變更來監控按鈕／開關介面事件（serial
   console MUX 需要此功能）。

3. 每個按鈕介面類別可以延伸以下可覆寫的方法，以便在需要時於衍生類別中
   實作平台專屬邏輯。

- init() - 此方法可能包含 gpio 設定與事件處理常式的初始化。

- handleEvent() - 此方法可能包含對應按鈕介面的自訂事件處理常式。應從
  主要事件處理常式函式呼叫此方法。

## 主機選擇器開關介面

主機選擇器開關關聯著 4 個 gpio。主機選擇器開關按鈕介面以 sd-event 為
基礎的事件迴圈監控這 4 個 gpio io 事件，並為所有 4 個 gpio 事件呼叫單一
事件處理常式函式。

根據 4 個 gpio 狀態值推導出主機選擇器開關位置的值，並以名稱「Position」
存放為 dbus 屬性。

### 主機選擇器 dbus 介面細節

1. Position(property) - 此屬性根據 GPIO 狀態保存主機選擇器開關的目前值。

2. MaxPosition(property) - 此屬性保存 Position 屬性可容納的最大 Position
   數量。（此值取決於平台上可用的主機數量）

## OCP Debug card 主機選擇器按鈕介面

為 debug card 主機選擇器按鈕建立一個獨立的介面。

此按鈕介面透過 sd-event 為基礎的事件迴圈，監控對應的 gpio 線，以取得
debug card 主機選擇按鈕的按下與放開事件。

### OCP Debug card 主機選擇器按鈕 dbus 介面細節

1. Released(signal) - 當 ocp debug card 按鈕被按下並放開時，此訊號會在
   ocp debug card 事件處理常式中觸發。

當按鈕放開後事件處理常式被呼叫時，主機選擇器開關的 dbus 屬性「Position」
會遞增 1。當 Position 值超過 MaxPosition 值之後，主機選擇器開關的 dbus
屬性值會循環歸零。

如此一來，當處理 power 與 reset 按鈕按下事件時，會參考 Host selector
Position 屬性，並根據 Position 呼叫相對應的電源事件。

## serial console MUX 介面

此按鈕介面監控主機選擇器開關 Position dbus 屬性的變更，並根據 dbus 屬性
值變更選取相對應的 serial console。MUX 值可以透過設定專用於 serial
console mux 的 4 個 gpio 線來設定。

### 範例

0 - bmc serial console 1 - host 1 serial console 2 - host 2 serial console 3 -
host 3 serial console 4 - host 4 serial console

### power button 與 reset button 介面事件處理常式的變更

若有 selector button 實例可用，則 power button 與 reset button 的按下事件
必須路由到適當的 multihost dbus 服務名稱。

## 影響

變更的影響範圍在 phosphor buttons repo 之下。現有的 power 與 reset 按鈕
處理常式會呼叫 dbus 呼叫來處理單一主機的電源事件。這些呼叫應修改為配合
對應的多主機電源事件，而相應的主機 dbus 物件則取決於主機選擇器位置。

## 測試

建議的設計可以在連接了多個主機的平台進行測試。
