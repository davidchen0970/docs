# 在 BMC 機器首次開機時設定唯一的 hostname

作者：Asmitha KR

其他貢獻者：無

建立日期：2019-05-07

## 問題描述

在 OpenBMC 中，hostname 的探索是由 avahi Dbus 服務在啟動時完成的。在一個存在多台 OpenBMC 機器的網路環境中，avahi 會不斷遇到 hostname 衝突與服務名稱衝突。因此，問題在於要找到一個能夠解決這些衝突的方案。

## 背景與參考資料

關於 hostname 與服務名稱衝突的詳細說明，請參考以下連結。

- <https://github.com/openbmc/openbmc/issues/1741.>
- <https://lists.freedesktop.org/archives/avahi/2018-January/002492.html>
- <https://github.com/lathiat/avahi/issues/117>

## 需求

無。

## 建議設計

為了解決這個問題，我們提出一個服務，它會在第一次開機時執行，並為 BMC 指派一個唯一的 hostname。產生唯一 hostname 的方法之一，是將從 Inventory Manager 取得的身分號碼（Serial Number）附加到現有的預設 hostname 之後。

## 曾評估的替代方案

無。

## 影響

無。

## 測試

無。
