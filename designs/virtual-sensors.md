# 虛擬感測器

作者：Vijay Khemka <vijaykhemka@fb.com>; <vijay!>

建立日期：2020-05-13

## 問題描述

系統中有些感測器的數值是從實際感測器資料，加上其他因素（例如溫度感測器所參考的風扇轉速）、某些複雜且非標準的比例因子，或其他演算法計算而得。虛擬感測器的範例包括：

1. 氣流：以風扇 RPM 為函數計算，單位為立方英尺每秒。
2. 兩個 ADC 輸入之間的電壓差。當實際差值不應超過某個閾值時。
3. 當我們有 Energy/time 時的功耗。

本設計將允許產生新的虛擬感測器，這些感測器被定義為既有感測器的特定操作／函數。

## 背景與參考

無。

## 需求

針對每個給定的感測器，都應該實作一個新的虛擬感測器，並依既有感測器的變化更新它們。此實作應提供

- 一個 daemon，用於建立並監控設定檔中定義的每個感測器，並根據每個給定感測器的數值變化來更新它們。
- 每個虛擬感測器都會等待各感測器參數在 dbus 數值上的變更事件，並更新此感測器。
- 感測器參數可以是任何感測器的 dbus 路徑，且支援虛擬感測器的鏈結（chaining）。
- 像其他感測器一樣的 dbus 介面，讓其他服務可以存取其資料
- 從設定檔讀取參數詳細資訊的能力
- 從設定檔讀取演算法字串的能力，並在計算新虛擬感測器的數值時使用同一個演算法

## 建議設計

建立一個 D-bus 服務 "xyz.openbmc_project.VirtualSensor"，為每個感測器提供物件路徑："/xyz/openbmc_project/sensor/temperature/sensor_name"

有一個 JSON 設定檔用於名稱、舊路徑與演算法。例如，

```json
  {
    "Desc" :
    {
        "Name" : "Virtual_Inlet_Temp",
        "SensorType" : "temperature"
    },
    "Params":
    {
      "ConstParam" :
      [
        {
          "ParamName" : "P1",
          "Value" : 1.1
        }
      ],
      "DbusParam" :
      [
        {
          "ParamName" : "P2",
          "Desc" :
          {
              "Name" : "MB_INLET_TEMP",
              "SensorType" : "temperature"
          }
        },
        {
          "ParamName" : "P3",
          "Desc" :
          {
            "Name" : "MB_FAN0_TACH",
            "SensorType" : "fan_tach"
          }
        }
      ]
    },
    "Expression": "P1 + P2 + 5 - P3 * 0.1",
    "Thresholds":
    {
      "CriticalHigh": 90,
      "CriticalLow": 20,
      "WarningHigh": 70,
      "WarningLow": 30
    }
  }

Desc:       It defines name and unit of a sensor. Main Desc object defines
            details about new virtual sensor and inside DbusParam object,
            it defines details about existing dbus sensors.
Name:       Name of virtual sensor
SensorType: Unit type of sensors and supported
Expression: An algorithm to be used to calculate sensor value
Params:     There are currently 2 types of parameter supported: ConstParam
            and DbusParam. Every parameter should have a name to be referred to
            in an Expression
ConstParam: This is a parameter which has a constant value defined here
DbusParam:  This is an existing/virtual sensor which is listed in dbus and
            it's value can be read from dbus path
Thresholds: It has critical and warning high/low value to monitor sensor
            value and used as per sensor interface defined in
            phosphor-dbus-interfaces.

```

## 曾考慮的替代方案

無

## 影響

這個應用程式會監控既有感測器，只有在它們的數值變更時才會更新此感測器，因此影響應該非常小。

## 測試

可以透過在一段時間內監控從 dbus 介面讀取的資料來進行測試，也可以與給定的感測器比較，觀察這些資料是否有所變化。
