# CPER records - CPER

作者：Ed Tanous - edtanous

建立日期：2024-05-22

## 問題說明

伺服器 CPU 會揭露一個被稱為 CPER records 的可管理性介面。BMC 之外的使用者
希望能够以解碼後的狀態讀取這些 records，而不是一包包的原始資料。

## 背景與參考資料

CPER 是 Common Platform Error Record 的縮寫，作為產業標準定義於
[UEFI Specification][uefi_spec] 中，並針對 aarch64、x64、Itanium 與 IA32 提供
各 CPU ISA 專屬的定義。在本文件中包含數個關於 BMC 的架構定義，其中包含
C.2.2 RAS IPMI Message Format 章節，以及 IPMI based RAS Event Receiver。

在 Redfish specification 2021.3 版中，Redfish 新增了對 [LogEntry resource][logentry]
中 CPER records 的支持。這些資源會以逐 section 解碼的方式揭露 CPER instance。
除了 Redfish 之外，DMTF 也提出了一個介面提案，用於透過 [MCTP/PLDM][cperevent]
將 CPER log events 傳送到 BMC，並預計在未來的 DMTF [DSP0248] 版本中加入。

ARM 已經開發了一個用於解碼 CPER records 的參考函式庫，但它沒有貢獻機制、
版本發布或維護，而且他們已經[明確表示][cper_examples]希望 OpenBMC 成為
這個函式庫的長期維護者。

這個函式庫 hosting 了 meson-dev 分支，該分支是為了本設計而新增的，目前可以
通過 OpenBMC CI 測試。如果獲得批准，這將被推送到 openbmc/libcper。

## 需求

- BMC 應該能夠解碼源自 CPER 相容 CPU 的二進位 CPER records。

- BMC 應該能夠依照 [CPER 規格][cper_records]從 CPU 接收並解碼 CPER records。

- BIOS/EDK2 build 應該能夠與 OpenBMC 共用解碼程式碼，其目的是當新增 records
  時，不需要在每個 codebase 中手動實作。

- CPU 供應商應該能夠新增對 CPER extensions 的支持，讓 OpenBMC 也能夠解碼，
  且如 CPER 規格所承諾的那樣，不會影響其他供應商的用戶。

- CPER 解碼器應該允許解碼多個 CPU complexes。

## 提議的設計

雖然本設計屬於前述 ARM 文件中暗示的一個更完整設計的一部分，但本文件只要求
第一步，也就是在 OpenBMC 組織內建立一個共用的函式庫實作，未來可以在此基礎
上持續擴展，但此時可能不會實作完整的方案。預期 CPER records 在 BMC 生態系
中的普及程度，足以證明建立這個 repository 是合理的；即使最初的實作可能無法
滿足所有貢獻者的所有設計目標，擁有共用的貢獻模型、CI 測試與授權條款，整體
而言是有益的。

未來的設計文件（或對本設計的修訂）將會持續迭代，以實作此
[CPER 規格][cper_records]中引用的更多設計內容，用於常見的 ARM 平台；但取得
libcper repo 的保管權、將品質提升到標準之上，是本設計的初始目標。

## 曾考慮的替代方案

從全新的設計角度重寫 libcper 的解碼。考慮到目前 libcper repo 的規模相當小，
這當然是可行的，但這會讓既有的解碼實作產生分岔。

## 影響

將在組織內建立新的 repository。將在 OpenBMC 中新增新的 recipe。

### 組織面

- 這個 repository 是否需要一個新的 repository？是
- 誰將是這個 repository 的初始維護者？Ed Tanous
- 預期為了執行本設計，會修改哪些 repositories？
  - openbmc/openbmc
  - openbmc/libcper

#### 未來可能

- openbmc/phosphor-debug-collector
- openbmc/bmcweb
- openbmc/phosphor-logging

## 測試

repo 中已經存在單元測試，用於驗證基本功能。等到設計完成後，將使用 CPU
型號專屬的錯誤產生器來模擬完整的路徑。

[cper_records]:
  https://uefi.org/specs/UEFI/2.10_A/Apx_N_Common_Platform_Error_Record.html
[uefi_spec]: https://uefi.org/specifications
[logentry]:
  https://github.com/DMTF/Redfish-Publications/blob/5b217908b5378b24e4f390c063427d7a707cd308/csdl/LogEntry_v1.xml#L1403
[cperevent]:
  https://www.dmtf.org/sites/default/files/PMCI_CPEREvent_Proposal_v3.pdf
[DSP0248]: https://www.dmtf.org/dsp/DSP0248
[cper_examples]:
  https://gitlab.arm.com/server_management/libcper/-/blob/b8b687c2e05846afd37b60222a0b4253acda81fd/README.md#usage-examples
