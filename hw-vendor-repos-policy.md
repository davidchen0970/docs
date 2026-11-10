# 硬體廠商 Repository 政策

作者：Andrew Geissler @geissonator

其他貢獻者：

建立日期：2024 年 5 月 13 日

## 簡介

本文件說明當有人請求在 OpenBMC Github 專案下託管一個新的 repository 或 bitbake recipe 時，TOF（Technical Oversight Forum）所採用的評審標準。文件先列出各項需求，接著說明這些需求是因應哪些問題而建立的。

## 需求

請注意，本文件假設讀者已經熟悉 OpenBMC 架構（bitbake、d-bus、phosphor-dbus-interfaces……）。

為任何事物訂定硬性規則都很困難，若過於死板反而違背了原本的意圖，因此這些是供 OpenBMC TOF 在決定是否核准新 recipe 或 repo 時參考的指引。

無論是硬體專屬（hardware-specific）或廠商專屬（vendor-specific）的 repository，都應符合以下需求，才有資格在 OpenBMC 下建立 repository：

1. 與 Apache 2.0 授權條款相容
2. 若是既有的 repository，必須有活躍的維護者
3. 沒有任何外部相依性會抵觸本文件所定義的需求
4. 新 repository 不會破壞其他專案（例如提供新的 API 給某個通用 repository 使用，卻沒有用編譯旗標等機制加以保護）
5. 它是一項被認為對 OpenBMC 組織普遍有用的功能
   - 由 TOF 進行判斷
   - 參考："Create a new repository" TOF [流程][tof]
6. 有一位活躍的 OpenBMC 社群成員願意擔任維護者

請注意，首選的方案應該永遠是把 repository 直接託管在 OpenBMC 之下（而不是在 OpenBMC 中放一個指向外部 repository 的 recipe）。

若符合以下情況，repository 就必須託管在 OpenBMC 之下：

1. 該 repository 相依於其他託管在 OpenBMC 的 repository

如果有人在 OpenBMC 中建立一個 recipe 來指向外部 repository，以下可能是合理的理由：

1. 若該 repository 也被 OpenBMC 之外的其他專案使用，那麼放在 OpenBMC 之外、透過 bitbake recipe 來使用，可能會更合適

如果在 OpenBMC 中建立了一個 recipe（無論在哪個 meta-X layer）來指向外部 repository，則該 repository 必須滿足以下需求：

1. 它應該符合在 OpenBMC 內託管 repository 的所有品質相關需求
2. 絕不得重寫該外部 repository 的 git 歷史
3. 該 repository 的維護者必須承諾對關鍵修復（例如新的編譯器更新）提供快速的回應處理

請注意，當某個 recipe 不是 OpenBMC 專屬、但對 OpenBMC 有用時，應該把它推送到上游的 meta-openembedded。

### 硬體專屬功能

硬體廠商功能定義為以下其中之一：

1. 特定於廠商硬體的功能
   - 例如 PECI、FSI、硬體診斷（特定於處理器）
2. 一個連接到 BMC（MCTP/PLDM/I2C……）的裝置，該裝置為廠商專屬，且 BMC 必須與之互動

若要將硬體專屬的 repository 加入 OpenBMC，還應符合以下額外標準：

1. 若需要特定的 kernel API，這些 API 必須已取得，或正依照[上游][upstream]指引進行開發

有些硬體廠商功能與 OpenBMC 沒有真正的協同效益，例如只與某個特定廠商系統相關的 CPLD code base，可能更適合託管在 OpenBMC 之外。

### 廠商專屬功能

廠商專屬功能定義為以下其中之一：

1. 廠商專屬的資料格式或通訊協定
   - 例如主機韌體的 dump 收集，或特殊格式的 SPD/VPD
2. 需要特定硬體廠商功能的功能

若要將廠商專屬的 repository 加入 OpenBMC，還應符合以下額外標準：

1. 它使用 OpenBMC 的 D-Bus API
   - 這代表該功能與 OpenBMC 之間存在協同效益

## 背景與參考資料

隨著 OpenBMC 不斷成長，與特定硬體系列綁定的軟體功能需求也隨之增加。例如，[OpenPOWER][openpower] 是最早採用 OpenBMC 的用戶之一，在 openbmc github 組織中搜尋 "openpower" 會看到許多名稱含該字的 repository。搜尋 "intel" 也會得到類似的結果。

OpenBMC 整體上應對其所管理的硬體保持無關（agnostic），但由於某些功能或特性只有特定硬體才提供，這並非總能做到。

舉例來說，OpenPOWER 系統在處理器內有一個獨立的電源管理裝置，稱為 OCC。這個 OCC 有它自己的通訊協定與功能。這就是 OpenBMC 設有 openpower-occ-control repository 的原因。

另一個例子，Intel 有一種稱為 PECI 的通訊介面，它能提供有用的資訊（如溫度），但只在 Intel 架構的系統上可用。這就是 OpenBMC 設有 peci-pcie repository 的原因。

本文件的目標是清楚說明在 openbmc github 組織內託管硬體／廠商專屬 repository 的需求與期望。本文件旨在回答以下問題：

1. OpenBMC 社群在什麼樣的情況下會劃分界線，決定哪些東西要放入 OpenBMC 組織、哪些不該放入？

2. 如果某個 repository 不屬於 OpenBMC 組織，那麼是什麼決定了一個 bitbake recipe 是否可以加入 OpenBMC，以指向 OpenBMC 組織之外、廠商的 repository？

本文件是根據 2024 年 1 月 9 日的 TOF 會議所撰寫。

## 理念

OpenBMC 希望盡可能多地啟用各種機器，並讓各機器之間盡可能共用共同的路徑，期許在各系統之間維持實作上的穩定性。

在 OpenBMC 中有兩個「廠商專屬」repository 的領域：

- 硬體專屬功能（驅動程式的實作、硬體支援，或其他僅存在於該平台上的東西）。
- 廠商專屬功能的支援（OEM 擴充、debug API、Redfish 等）。

Yocto 以 "machine" 與 "distro" 的觀念來區分上述兩者。我們需要在 OpenBMC 中訂定一套政策，說明我們如何處理這些不同類型的功能。

OpenBMC 專案有一套相當完整的 CI 流程、許多優秀的 code reviewer，並追蹤上游 yocto master 的最新版本。這一切確保了堅實的 code base 供他人使用，也為託管在 OpenBMC 內的 repository 樹立了某種標準。將 source 置於 OpenBMC 的保護傘下，能確保它獲得全域性的更新（例如上游 poky/bitbake 的變更），以及其他像是最新編譯器等更新。

有時候 yocto 會帶來大型更新，導致 OpenBMC 所使用的多個 repository 出問題。對於託管在 OpenBMC 內的 repository，要協調這類重大變更總是比較容易（比起試圖與外部託管的 repository 協調）。

OpenBMC 已經託管了 100 多個 repository，對剛加入專案的新人而言可能相當令人卻步。新 repository 應該在能夠長期維護、且不會排擠其他專案工作資源的情況下才加入。為這個流程建立一套指引，有助於加速專案內的新程式碼開發。這個功能有多實用？還有多少人能用得上它？即將維護新 repository 的人在社群中有多活躍？這些只是我們需要納入審查流程中釐清的幾個問題。

在理想情況下，只有硬體專屬功能才會放在 meta machine layers 中，而廠商專屬功能則應放在 distro features 中。OpenBMC 尚未達到這個目標，但這是社群應該努力的方向。

## 結論

撰寫本文件是為了提供一份參考，讓人知道該把廠商專屬功能放在哪裡，同時也是 TOF 團隊在評估 OpenBMC 內廠商專屬 repository 的請求時可以參照的依據。

[openpower]: https://openpowerfoundation.org/
[upstream]: https://github.com/openbmc/docs/blob/master/kernel-development.md
[tof]: https://github.com/openbmc/technical-oversight-forum/issues
