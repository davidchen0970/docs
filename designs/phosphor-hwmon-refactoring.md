# Phosphor-hwmon 重構

作者：Kun Yi <kunyi!>

建立日期：2019/09/05

## 問題描述

phosphor-hwmon 中盤根錯節的程式碼，正在削弱我們新增功能以及
找出／修復錯誤的能力。

## 背景與參考資料

程式碼各處存在若干程式碼異味（smelly code）或不良設計決策的案例：

- 單元測試涵蓋率有限。目前 repo [1] 中的單元測試多半只是驗證
  sensor 能否被正確建構，但並沒有針對各種 sensor 讀值或失敗條件下的
  正確行為進行測試。
- 臃腫的主迴圈（mainloop）。mainloop::read() 高達 146 行 [2]，其中交雜著
  複雜的條件判斷、巨集（macro）與 try/catch 區塊。
- 缺乏抽象化與依賴注入（dependency injection）。這使得程式碼難以測試、
  也難以閱讀。舉例來說，[3] 處的條件判斷可以改為針對不同 sensor 型別
  實作不同的 sensor::readValue() 方法來取代。

[1](https://github.com/openbmc/phosphor-hwmon/tree/master/test)
[2](https://github.com/openbmc/phosphor-hwmon/blob/94a04c4e9162800af7b2823cd52292e3aa189dc3/mainloop.cpp#L390)
[3](https://github.com/openbmc/phosphor-hwmon/blob/94a04c4e9162800af7b2823cd52292e3aa189dc3/mainloop.cpp#L408)

## 目標

1. 提升單元測試涵蓋率
2. 運用 OOD 原則改善整體設計
3. 改善錯誤處理與韌性
4. 提升可設定性
   - 透過提供一個共用的設定類別，我們將能新增其他的設定方式，
     例如 JSON，同時維持與現有 env 檔做法的向下相容性。

## 提案設計

重構步驟的大略拆解：

1. Sensor 設定
   - 新增一個 SensorConfig struct，將所有 std::getenv() 呼叫集中於一處
   - DI：讓 sensor struct 以 SensorConfig 作為依賴
   - 為 SensorConfig 撰寫單元測試
   - 針對使用各種 SensorConfig 建立 sensor 的情境新增單元測試
2. 抽象化 sensors
   - 精煉 sensor 物件介面並使其成為抽象介面
   - 定義繼承自共用介面的 sensor 型別
   - 為 sensor 介面新增單元測試
   - DI：讓 sensor map 以 sensor 介面作為依賴
   - 新增一個可展現可控行為的 fake sensor
3. 重構 sensor 管理邏輯
   - 重構 sensor map 以便於查詢／插入
   - 為 sensor map 新增單元測試
   - DI：讓所有其他函式以 sensor map 作為依賴
4. 抽象化錯誤處理
   - 在 sensor 介面中新增可覆寫的錯誤處理程序
   - 定義不同的錯誤處理程序
   - 使用 fake sensor 為各錯誤處理程序新增單元測試

## 曾考慮的替代方案

不適用。

## 影響

此次重構不應對外部的 API 造成影響。效能影響則應加以量測分析。

## 測試

目標之一便是擁有更好的單元測試涵蓋率。整體功能將透過功能測試與
迴歸測試來驗證。
