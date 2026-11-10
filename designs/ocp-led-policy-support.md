# OCP LED Policy 支援

作者：Alexander Hansen
[alexander.hansen@9elements.com](mailto:alexander.hansen@9elements.com)

其他貢獻者：無

建立日期：2024 年 7 月 22 日

## 問題描述

OpenBMC 必須支援 [OCP LED Policy](#background-and-references)。目前有一些有問題的情況，現有的服務（例如 phosphor-led-manager）並不支援該 Policy。

### 範例（[規格](#background-and-references) 6.2. System General Status）

#### 已定義的狀態

| STS（藍色） | FAULT（琥珀色） | NAME |
| ---------- | ------------- | ---- |
| 1          | 0             | "All OK" |
| 0          | 1             | "Module missing / fault" |
| 1          | 1             | LED Priority（無法表示 fault 狀態） |

#### 範例序列

| STS | FAULT | 動作 |
| --- | ----- | ---- |
| 0   | 0     | 初始狀態 |
| 1   | 0     | 設定 ok |
| 1   | 1     | 設定 fault |
| 1   | 1     | 最終狀態 |

例如，當這兩組同時被設定（assert）時，LED 狀態的變化將由 LED priority 決定。這樣的結果在[規格](#background-and-references)的狀態圖中是不被允許的。

在這個簡單的範例中，我們只要把 led-blue 的 Priority 指定為 "Off" 就能修正。但在撰寫本文件時（commit [bdbfcde](https://github.com/openbmc/phosphor-led-manager/commit/bdbfcde10520fc841b44d1e777c353a698b774ff)）這還做不到。

還有其他擁有更多 LED 的範例，光是這樣做無法解決問題。

### 範例（[規格](#background-and-references) 6.4 PSU Status）

這裡我們假設 LED Priority "Off" 是被允許的，並發現這樣仍不足以解決問題。

#### 已定義的狀態

| AC OK（藍色） | FAULT（琥珀色） | LOW V（琥珀色） | BACK UP（琥珀色） | 狀態 |
| ------------ | ------------- | ------------- | --------------- | ---- |
| 1            | 0             | 0             | 0               | "AC OK" |
| 0            | 1             | 0             | 0               | "AC Fault" |
| 0            | 1             | 1             | 0               | "AC Under Voltage" |
| 0            | 0             | 0             | 1               | "Backup due to AC Outage" |

#### LED Priority

| AC OK（藍色） | FAULT（琥珀色） | LOW V（琥珀色） | BACK UP（琥珀色） | 狀態 |
| ------------ | ------------- | ------------- | --------------- | ---- |
| 0            | 0             | 0             | 1               | "Backup due to AC Outage" |

#### 範例序列

| AC OK（藍色） | FAULT（琥珀色） | LOW V（琥珀色） | BACK UP（琥珀色） | 狀態 |
| ------------ | ------------- | ------------- | --------------- | ---- |
| 0            | 0             | 0             | 1               | LED Priority（僅供參考） |
| 0            | 0             | 0             | 0               | 初始狀態 |
| 1            | 0             | 0             | 0               | 設定 "AC OK" |
| 0            | 0             | 0             | 0               | 設定 "AC Fault" |
| 0            | 0             | 0             | 0               | 最終狀態 |

根據[規格](#background-and-references) 6.4，最終狀態是未定義的。

這裡各種可能的 LED priority 都不重要，因為 "Backup due to AC Outage" 很可能就是最高優先權的狀態。而選擇任何其他 LED Priority，都會在它與其他狀態同時被設定時，使得它完全無法被套用。

到此已經很清楚，我們需要的是一種以群組（而非單一 LED）來定義優先權的方式。

## 背景與參考資料

- [OCP Panel Indicator Specification rev1.0 (pdf)](http://files.opencompute.org/oc/public.php?service=files&t=65c02b1c6d59188351357cfb232cbfaa&download)
- [OCP Panel Indicator Specification rev1.0 (presentation)](https://www.opencompute.org/files/OCP18-EngWorkshop-Indicator-Specification-Proposal.pdf)

以下是對我們而言重要內容的摘要：

### 允許的指示燈顏色

- 藍色
- 琥珀色
- 在狹小空間中可使用藍/琥珀結合色

### 允許的指示燈狀態

- OFF
- ON
- BLINK（亮 500ms，滅 500ms）

### 元件上允許的指示燈數量

- 1 個藍色 LED
- 1 到 n 個琥珀色 LED，用以表達多種故障情況，或
- 1 個藍/琥珀色 LED

## 需求

- 能夠為不同的 LED 群組以及由此可能產生的指示燈狀態排列優先順序。
- 指示燈不應處於不一致的狀態。它應能顯示[規格](#background-and-references)中所定義的狀態。
- 無法信任其他服務會讓 LED 群組的設定（assert）保持一致。即使有互相衝突的 LED 群組被設定，指示燈狀態仍必須保持一致。

## 建議設計

在 phosphor-led-manager 中將「Priority」的概念延伸到群組。群組優先權（group priority）將是一個整數值。數字越大代表該群組的優先權越高。這讓使用者可以有像這樣的設定：

1. OK
2. Fault 1
3. Fault 2
4. Locate

其中 locate 與 fault 狀態具有較高的優先權。LED 將永遠處於一致的狀態。這個變更可以以向後相容的方式完成。

### 設定範例

```json
{
  "leds": [
    {
      "group": "enclosure_identify",
      "Priority": 2,
      "members": [
        {
          "Name": "sys_id",
          "Action": "Blink"
        },
        {
          "Name": "rear_id",
          "Action": "Blink"
        }
      ]
    },
    {
      "group": "fault",
      "Priority": 1,
      "members": [
        {
          "Name": "sys_id",
          "Action": "On"
        },
        {
          "Name": "fault",
          "Action": "On"
        }
      ]
    },
    {
      "group": "unrelated",
      "members": [
        {
          "Name": "rear_id",
          "Action": "On"
        }
      ]
    }
  ]
}
```

### 混合使用

在設定中，群組優先權與 LED 優先權可以互斥，以避免混合使用所帶來的任何問題。因此既有的設定仍可使用個別 LED Priority 繼續運作；而當指定了群組優先權時，我們就假設使用者想要使用它來設定。

## 曾考慮過的替代方案

- 延伸 phosphor-led-sysfs，將 2 個 LED（藍/琥珀）作為單一 LED 對外暴露。
  - 這行不通，因為根據[規格](#background-and-references)，可能會有多達 n 個不同的故障 LED。
  - 而且這本質上是在自欺欺人，並讓其他軟體難以再從 dbus 取得任何關於 LED 的有意義資訊。

- 在 LED 設定中允許 Priority "Off"。
  - 這只能解決非常簡單設定的問題。
  - 當多個 LED 組成一個指示燈時，個別 LED 優先權很難思考。

- 允許群組與個別 LED 優先權混合使用。
  - 這需要考慮混合使用所產生的更多邊際情況。
  - 不清楚有什麼使用情境會從混合使用獲益。

- 讓每個 LED 可以設定它所代表的群組優先權，而不只是一個狀態。
  - 例如 "Priority": ["enclosure_identify", "fault", "power"]
  - 這個設定必須在 LED 的每個實例上重複
  - 或者假設由第一個實例來定義？
  - 需要檢查某個 LED 上這些優先權清單是否全都相等
  - 這無法解決群組被不完整表示的問題
    - 例如，屬於同一群組的 2 個 LED 可能在其優先權清單中對這些群組有不同的優先順序。

- 允許設定一個由多個 LED 組成的「indicator」，然後只為該 indicator 定義狀態。
  - 無论如何都需要為了相容於現有的內部資料結構，而將這些狀態轉換為群組。
  - 需處理該 indicator 的成員 LED 同時也是其他群組成員的情況。
  - 這本質上與群組優先權相同，只是在設定解析上增加了額外負擔。

- 在群組互相衝突時，只顯示最後被設定的那個群組。
  - 這是不可取的，因為有些群組在顯示上更為重要。

## 影響

需要

- 延伸文件，說明如何設定 LED 群組優先權
- 實作 LED 群組優先權
- 為 LED 群組優先權撰寫單元測試
- 也許修改一些設定以使用這個新功能
  - 這是選擇性的，因為此變更是向後相容的

除非我們選擇對外暴露 led group priority，否則對 d-bus 不應有任何影響。

### 組織方面

- 這個 repository 是否需要新的 repository？
  - 不需要
- 誰會是這個 repository 的初始維護者？
- 預計需要修改哪些 repository 來執行這個設計？
  - phosphor-led-manager
  - 選擇性地，openbmc 以升級既有設定
- 列出一份清單，並將所列 repository 的維護者加入 gerrit review。

## 測試

將如何進行測試？這個功能會如何影響 CI 測試？

這個功能可以輕鬆地透過單元測試來驗證。
