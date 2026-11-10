# Google 專屬 API - 一個供 Google 使用的新服務根

作者：Feras Aldahlawi (faldahlawi)

其他貢獻者：無

建立日期：2021 年 3 月 30 日

## 問題描述

Redfish API 沒有任何資源類型與 Google 的信任根（RoT）晶片類似。Google 需要一些尚未納入 Redfish 標準的 API。目前已有專門的工作群組致力於將 RoT 晶片支援帶入 Redfish 標準。因此，在 Google 命名空間下新增此支援可以避免與那些工作群組產生衝突。本文件提供 Google 對其新服務根所需內容的結構描述（schema）。

## 背景與參考

在 Google，我們依賴各種傳輸機制與 RoT 晶片溝通。Google 希望延伸支援範圍，納入基於 REST 的 API。Google 未來 RoT 裝置的發展將採用
[SPDM 協定](https://www.dmtf.org/sites/default/files/PMCI_Security-Architecture_12-11-2019.pdf)。
不過，本設計文件針對的是一群永遠無法支援標準化介面的 RoT 裝置。

## 需求

- 建立一個 Google 專屬 API 的新服務根。
- 為 RootOfTrust 資源建立一個結構描述（schema）。
- 能夠透過 API 執行 RoT 動作（證明（attestation）等）。

## 建議設計

在 `google/v1` 下建立一個新的服務根。這個服務根將包含一組 `RootOfTrust` 實體，這些實體具有以下屬性與動作（Actions）：

- 晶片類型字串
- 唯一硬體 ID 字串
- 韌體版本對應（字串對字串的 map）
- Mailbox API

這個新 API 的設計目的是將呼叫轉傳給 RoT 裝置，並避免對資料做任何檢查。一個範例呼叫如下：

```json
{
  "#RootOfTrust.Mailbox": {
      "target": "/redfish/v1/RootsOfTrust/0/Actions.Mailbox",
      "@Redfish.ActionInfo": "/redfish/v1/RootsOfTrust/0/Actions.Mailbox"
    }
  "RawRequest": "some_bytes_to_be_parsed_by_receiver"
}
```

這個新的服務根與 `/ibm/v1` 非常類似。這將需要一個新的 dbus 介面来服務此 API：

```yaml
description: >
  Forward bytes to Google RoT devices.
methods:
  - name: Mailbox
    description: >
      A method to forward bytes to RoT device.
    parameters:
      - name: rawRequest
        type: array[byte]
        description: >
          Value to be updated for the keyword.
    errors:
      - xyz.openbmc_project.Common.Error.InvalidArgument
      - xyz.openbmc_project.Common.Error.InternalFailure
```

## 曾考慮的替代方案

曾考慮將這些新 API 作為 TPM 資源的 OEM 延伸新增。然而，這是一種不自然的延伸。以下是對 Google 的 RoT 而言使用 TPM 某種程度上不夠自然的原因：

- FirmwareVersion1/2
  - 某種程度上與 TPM 的設計綁定得很緊。TPM 1.2 使用 32 位元的韌體版本，而 TPM 2.0 只是笨拙地再追加另一個 32 位元的韌體版本欄位来延伸它。
  - TPM 的 "Firmware 1" 與 "Firmware 2" 合起来指的是一個 64 位元的韌體版本號。多數 TPM 2.0 供應商將其劃分為 4 個欄位，每個欄位寬 2 個位元組：（大端序，每個字元是一個位元組：）0xMMmmrrbb（M 為 major，m 為 minor，r 為 rev，b 為 build）。Infineon 對韌體版本號採用的慣例與其他 TPM 供應商不同，它保留某些位元，僅以 0xMMmm_rrb 表示寬 1 個位元組的 "build number"。
  - 將這些值以字串形式暴露給 Redfish 介面，對 Google 而言是可行的。但我只是想提供目前 TPM 生態圈中這件事有多不統一（其實並不統一）的資訊。
- InterfaceType
  - 目前與 TPM 各種變體的生態圈綁定得很緊。
  - 實作的是哪一種 TPM 介面風格。TCM 是 TPM 1.2 的「中國版本」。在中國 TPM 轉用 TPM 2.0 之前，該版本規範已可使用。
  - TPM 1.2 與 2.0 是完全不同的 API 介面範圍，這類似於任何 TPM 與 Google 的 RoT 晶片之間的差異。
- InterfaceTypeSelection
  - 目前與 TPM 各種變體的生態圈綁定得很緊。
  - 某些 TPM 可以在 TPM 1.2 與 2.0 之間切換。除非 Google 開始出貨一款可以透過韌體更新切換到 TPM 模式的開放原始碼 RoT（那會是不錯的做法），否則 Google 大概可以忽略這點。

雖然我們可以將所有內容都放在 TPM 的 OEM 之下（例如版本號與其他功能），但多數欄位對 Google 的 RoT 而言都無法使用。

## 影響

新的 daemon，新的 Rest API。

## 測試

測試將在 Google 內部進行。此功能不應該影響 CI 測試。我們會嘗試正常路徑與異常路徑。我們也會實作模糊測試（fuzz tests）。
