# Authorities 清單管理

Author: Nan Zhou <nanzhoumails@gmail.com>

Created: 12/01/2021

## 問題描述

有些使用案例會在系統上安裝多張根憑證（root certificate），用以驗證不同的
client。舉例來說，在 Google，會在 BMC 上安裝一個 trust bundle 檔（也就是
一組 authorities 清單）來進行 mTLS 驗證。

目前的 phosphor-certificate-manager 對於管理多張根憑證並沒有良好的支援：

1. 它在 dbus 上只允許取代單一的 Authority 物件；然而 Google 的使用案例
   需要批次取代（請見下方的 ReplaceAll 介面）

2. 當給定一個包含多張憑證的 PEM 編碼檔時，它只會擷取第一張憑證；然而 Google
   的 trust bundle 檔包含多張以 PEM 編碼的憑證

## 需求

Phosphor-certificate-manager（僅限 Authority Manager）與 BMCWeb 將支援
authorities 清單：

1. 批次安裝（Bulk Installation）：給定一個包含多張根憑證的 PEM 檔，它會驗證
   並安裝其中的全部憑證，並回傳一組建立出來的物件清單

2. 批次取代（Bulk Replacement）：給定一個包含多張根憑證的 PEM 檔，它會先刪除
   目前所有的根憑證，然後重新執行安裝

3. Redfish：BMCWeb 會將所有的 authorities 匯出為 Redfish Certificate

4. 開機時的復原（Recovery）：當 phosphor-certificate-manager 被實例化時，如果
   它在安裝路徑中找到一份 authorities 清單，它會透過批次安裝從該清單進行復原

5. 原子性（Atomic）：批次安裝與批次取代具備原子性；也就是說，如果清單中有一張
   無效的憑證，服務就不會安裝清單中的任何一張憑證

## 擬議設計

我們提出兩個新的介面：

1. InstallAll
2. ReplaceAll

### xyz.openbmc_project.Certs.InstallAll

當憑證類型為 Authority 時，我們不再只擷取第一張憑證，而是逐一遍歷每一張憑證、
驗證它、在 DBus 上建立對應的物件、將各別憑證以 PEM 檔匯出到安裝路徑、依據
subject hash 為每一張憑證建立 alias（這是來自 boost 的 `ssl_context` 之需求），
最後將 PEM 檔複製到安裝路徑（該 PEM 檔會有一個固定的檔名）

我們會以字串向量（vector of strings）的形式回傳所有建立出來的物件路徑。

對於其他類型的憑證（server 與 client），服務會拋出 NOT ALLOWED 錯誤。

### xyz.openbmc_project.Certs.ReplaceAll

這個新介面包含一個 ReplaceAll 方法，它接受一個指向輸入 PEM 檔的路徑。

憑證管理員（certificate manager）將實作這個新的 ReplaceAll 介面。當該方法被
呼叫時，它會刪除目前所有的 authority 物件、取得輸入的 PEM 檔，然後重新執行
安裝。

對於其他類型的憑證管理員（server 與 client），服務會拋出 NOT ALLOWED 錯誤。

### xyz.openbmc_project.Certs.Replace

沒有變更。各別的 authority 憑證仍然可以分別被取代。即使 PEM 檔中包含多張根憑證，
它也只會擷取第一張憑證。

## 影響

除了新增 API 之外，沒有其他影響

## 曾考慮的替代方案

我們也可以建立一個 trust bundle 介面（而不是使用多張 Certificates），並實作它
獨立的 manager 常駐程式（daemon）。這樣對既有程式碼的影響較小。然而，trust
bundle 並不存在於 BMCWeb 中，Redfish schema 中也沒有。

## 測試

強化 phosphor-certificates-manager 中現有的單元測試，以測試批次安裝與批次取代。
