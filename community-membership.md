# 社群成員資格

本文件說明 OpenBMC 中各貢獻者角色的各項期望。OpenBMC 專案劃分為多個
子儲存庫（subrepositories）。多數角色的職責範圍都以這些子儲存庫為界。

| Role                | Expectations                                           | Requirements                                                                                           | Defined by                                                     |
| ------------------- | ------------------------------------------------------ | ------------------------------------------------------------------------------------------------------ | -------------------------------------------------------------- |
| Code Contributor    | 遵守行為準則                                           | 已簽署貢獻者授權協議（Contributor License Agreement）                                                   | 已合併的程式碼                                                  |
| Active Contributor  | 對專案有持續穩定的貢獻                                  | 由 [Technical Oversight Forum][TOF election cycle] 設定的資格條件                                        | 列入當期的 ["Roll-call"][tof-rollcall] 名冊。                    |
| Designated Reviewer | 審核其他成員的貢獻                                      | 在某個子專案中有審核紀錄                                                                                | [OWNERS] 檔案中的 reviewer 條目                                   |
| Platform Maintainer | 審核並維護單一平台的貢獻                                 | 對某個特定平台有測試與擁有權的紀錄                                                                       | 該平台 meta-\* 子資料夾中的 [OWNERS] 條目                         |
| Approver            | 審核並維護部分程式碼的貢獻                               | 跨平台對該部分程式碼展現責任感與卓越技術判斷力                                                           | 子專案 [OWNERS] 檔案中的 _owners_ 條目                            |
| Subproject owner    | 為子專案設定方向與優先順序                               | 跨平台對該子專案展現責任感與卓越技術判斷力                                                               | 子專案 [OWNERS] 檔案中的 _owners_ 條目                            |
| Security Response   | 回應並處理未揭露的安全性問題                             | 有參與安全性問題的紀錄                                                                                  | docs 儲存庫安全性文件中的條目                                     |
| TOF Member          | 為專案設定方向與優先順序                                 | 經由 [TOF election cycle] 當選                                                                          | docs 儲存庫 TOF 文件中的條目                                      |

## 新貢獻者

現有成員應當歡迎 [新貢獻者] 加入社群，協助他們熟悉審核流程，並引導他們
前往相關的說明文件與溝通管道。

## 既有社群成員

既有社群成員應當展現對本文件中所列原則的遵循、對專案組織、角色、政策、
程序、慣例等的熟悉程度，以及技術與（或）寫作能力。各角色特定的期望、
職責與要求列於下方。

## Code Contributor

成員即是社群中的貢獻者。他們可以被指派 issue 與審核、透過 Gerrit 參與
設計審查，並且他們的審核會自動執行提交前測試。

**Defined by:** 列於有效的貢獻者授權協議上。

### Expectations

- 遵守行為準則

## Active Contributor

一旦開發者根據 Technical Oversight Forum 設定的期望，持續對專案做出貢獻，
即可獲授予 "Active" 狀態。此狀態有效期為 6 個月，依據的是前 6 個月的貢獻。

**Defined by:** 列入 TOF 選舉的 roll-call 名冊。

### Expectations

- 已對專案或社群做出多項貢獻。貢獻的型態包含但不限於：
  - 在 Gerrit 上撰寫或審查 code review
  - 在 GitHub 上提出 issue 或留言
  - 參與設計審查、子專案或社群的討論（例如會議、Discord、郵件列表討論）
- 已訂閱 [mailing list]
- 已閱讀 [contributor guide]
- 積極對一個或多個子專案做出貢獻。
- **[Submit a CLA]**

## Designated Reviewer

Designated Reviewer 有能力針對子專案的某些部分，審核程式碼的品質與正確性。
他們同時精通程式碼庫與軟體工程原則。

**Defined by:** OpenBMC 專案所擁有的儲存庫中，OWNERS 檔案裡的 _reviewers_ 條目。

Reviewer 的權責範圍限定於程式碼庫的某個部分。

**Note:** 要接受程式碼貢獻，除了指定的 reviewer 之外，至少還需要一位
approver 核准。

### Expectations

以下適用於在 [OWNERS] 檔案中擔任 reviewer 的程式碼庫部分。

- 成為成員至少 3 個月
- 作為主要 reviewer 參與程式碼庫至少 5 個變更
- 審核或合併過程式碼庫中至少 5 個實質變更
- 精通程式碼庫
- 由一位子專案 approver 提攜（sponsor）
  - 且其他 approver 沒有反對意見
  - 透過 PR 更新 OWNERS 檔案來完成
- 可以自我提名，或由該子專案的一位 approver 提名。

### Responsibilities and privileges

以下適用於在 [OWNERS] 檔案中擔任 reviewer 的程式碼庫部分。

- code reviewer 狀態可能是接受大型程式碼貢獻的先決條件
- 透過 [code reviews] 負責專案的品質管制
  - 著重於程式碼品質與正確性，包含測試與重構（factoring）
  - 也可以針對更整體性的問題進行審核，但並非強制要求
- 依據 [community expectations]，被期望對審核請求保持回應
- 被指派審核與其所精通子專案相關的變更
- 被指派與其精通子專案相關的測試缺陷（test bugs）

## Platform Maintainer

Platform maintainer 有能力針對適用於單一平台的 meta 圖層與子系統，審核設定
變更的品質與正確性。他們了解特定平台的限制條件，並能取得該平台的一部實例
來進行測試。

**Defined by:** [main OpenBMC repository] 中，某個機器 meta 圖層的 OWNERS 檔案裡的
_owners_ 條目。

### Expectations

以下適用於在 [OWNERS] 檔案中擔任 platform owner 的程式碼庫部分。

- 成為成員至少 3 個月
- 作為主要 reviewer 參與程式碼庫至少 5 次審核
- 了解該平台的特定限制條件
- 能取得一部平台來測試並執行程式碼
- 具備可證明（demonstrated）的 bitbake 中繼資料（metadata）知識
- 由 openbmc/openbmc OWNERS 檔案中的一位根 approver 提攜
  - 且其他 approver 沒有反對意見
  - 透過 PR 更新 OWNERS 檔案來完成
- 可以自我提名，或由該子專案的一位 approver 提名。

### Responsibilities and privileges

以下適用於在 [OWNERS] 檔案中擔任 reviewer 的程式碼庫部分。

- 擁有一位具備 platform maintainer 狀態的 owner，可能是接受新平台的先決條件
- 負責平台的穩定性
  - 定期測試（基本期望為每季一次）
  - 就平台專屬的問題向社群提供結果與見解
- 依據 [community expectations]，被期望對審核請求保持回應
- 被指派審核並測試與該平台相關的變更
- 若將長期（超過 1 個月）請假或辭去這些職責，需透過修改 [OWNERS] 檔案，
  與 technical-oversight-forum 溝通。

## Approver

Code approver 既能審核也能核准程式碼貢獻。雖然 code review 著重於程式碼
品質與正確性，核准則著重於對貢獻的整體性接受與否，包含：向後／向前相容性、
遵循 API 慣例、細微的效能與正確性問題、與系統其他部分的互動等等。

所有 approver 都應當是 OpenBMC 的 active contributor，並以近期
[TOF roll-call][tof-rollcall] 確認。任何近期未符合資格的 approver，
Subproject Owner 或 TOF 成員可自行決定將其從 [OWNERS] 檔案中移除。

- 自 2026 年起，approver 應當在 2024H1 之後的某個時間點符合過資格。
- 自 2027 年起，approver 應當在 2026H1 之後的某個時間點符合過資格。
- 在 2028 年及之後，approver 在最近一期（half）中至少要有 "Active" 或
  "Emeritus" 狀態。

**Defined by:** OpenBMC 專案所擁有的儲存庫中，OWNERS 檔案裡的 _owners_ 條目。

Approver 的權責範圍限定於程式碼庫的某個部分。

### Expectations

以下適用於在 [OWNERS] 檔案中擔任 approver 的程式碼庫部分。

- 擔任程式碼庫的 reviewer 至少 9 個月
- 作為主要 reviewer 參與程式碼庫至少 10 個實質變更
- 審核或合併過程式碼庫中至少 30 個變更
- 能取得一個合適的平台環境
- 由兩位子專案或 TOF 根 owner 提攜（sponsor）提名
  - 且其他 subproject owner 沒有反對意見
  - 透過 PR 更新子專案最上層的 OWNERS 檔案來完成
  - **請注意以下對提攜者（sponsors）的期望**：
    - 提攜者必須與擬納入的成員有緊密的互動——例如程式碼／設計／提案審查、
      協同處理 issue 等。
    - 提攜者必須至少在一個子專案中擔任 approver。
    - 提攜者必須來自多家成員公司，以展現跨社群的整合。

### Responsibilities and privileges

以下適用於在 [OWNERS] 檔案中擔任 approver 的程式碼庫部分。

- approver 狀態可能是接受大型程式碼貢獻的先決條件
- 展現穩健的技術判斷力
- 透過 [code reviews] 負責專案的品質管制
  - 著重於對貢獻的整體性接受與否，例如與其他功能的相依性、向後／向前相容性、
    API 與旗標（flag）定義等
  - 維護不含多餘或未使用程式碼的程式碼庫，並移除不再使用和（或）維護的舊貢獻。
- 依據 [community expectations]，被期望對審核請求保持回應
- 指導（mentor）貢獻者與 reviewer
- 可核准程式碼貢獻以進行接受

## Subproject Owner

**Note:** 此處是該角色概括性的高階描述，subproject owner 角色的職責與相關
程序的細節，在個別子專案中可能更為具體，並以各自 OWNERS 檔案中的定義為準。

Subproject Owner 是 OpenBMC 專案中某個子專案的技術權威。他們 _MUST_ 已展現
對該子專案健康狀況的良好判斷力與責任感。Subproject Owner _MUST_ 設定技術
方向，並為其子專案做出或核准設計決策——可以直接進行，或透過委派這些職責來
完成。

所有 subproject Owner _MUST_ 是 OpenBMC 的 active contributor，並以近期
[TOF roll-call][tof-rollcall] 確認。任何近期未符合資格的 subproject Owner，
將由 TOF 成員從 [OWNERS] 檔案中移除。

- 自 2026 年起，subproject owner 必須在 2024H1 之後的某個時間點符合過資格。
- 自 2027 年起，subproject owner 必須在 2026H1 之後的某個時間點符合過資格。
- 在 2028 年及之後，subproject owner 在最近一期（half）中至少要有 "Active"
  或 "Emeritus" 狀態。

**Defined by:** 子專案 [OWNERS] 中的 _owners_ 條目

### Expectations

成為 subproject Owner 的每個 repository（per-repository）要求，應當在子專案的
OWNERS 檔案中定義。不同於上文所列的角色，子專案的 Owners 通常限於一組相對
少量的決策者，並依子專案的需要更新。

以下適用於你將擔任 owner 的子專案。

- 深入理解子專案的技術目標與方向
- 深入理解子專案所屬的技術領域
- 透過以下所有方式持續對設計與方向做出貢獻：
  - 撰寫並審查提案
  - 發起、參與並解決討論（郵件、GitHub issue、會議）
  - 在設計與實作審查中找出細微或複雜的問題
  - 透過測試確保子專案在一個或多個平台上功能正常
- 透過實作與（或）審核直接對子專案做出貢獻
- 符合 OWNERS 檔案中所列所有子專案專屬的要求

### Responsibilities and privileges

以下適用於你將擔任 owner 的子專案。

- 為子專案做出並核准技術設計決策。
- 為子專案設定技術方向與優先順序。
- 指導並引導子專案的 approver、reviewer 與貢獻者。
- 確保子專案的持續健康
  - 有足夠的測試涵蓋率以自信地發布
  - 測試存在且能穩定通過（即不會不穩定），失敗時會被修正
- 確保已建立健康的討論與決策流程。
- 與其他 subproject owner 合作，整體性地維護專案的總體健康與成功。
- 隨時掌握專案整體的技術變更，並自行維護或委派維護子專案，使其與整體專案
  保持一致。

## Security Response Team Member

安全性回應團隊負責 OpenBMC 專案中的協調揭露（coordinated disclosure）。成員
會獲准存取未公開的 embargoed 資訊。

**Defined by:** docs 儲存庫中的安全性回應團隊成員名單。

### Expectations

- 對專案的安全性架構有深入技術理解。
- 認識社群成員、專案內外部的技術專家，並在被納入前的 12 個月內有參與安全性
  問題的紀錄。
- 任何時候都維持 "Active" 成員狀態。
- 至少擁有一部代表性的 BMC 系統，可用來重現安全性問題。

### Responsibilities and privileges

- 維護專案中的安全性文件。
- 就安全性基礎設施與最佳實務，指導並引導專案的貢獻者。
- 依照安全性回應團隊的準則回應揭露事件（disclosures）。
- 可存取安全性郵件列表。
- 在發布前執行保密期限（embargoes）與安全性問題的不得揭露規定。
- 維護一個聯絡 email 位址，並在專案溝通管道上保持在線。

## Technical Oversight Forum Member

Technical Oversight Forum 是 OpenBMC 專案的技術權威。他們 _MUST_ 已展現對
OpenBMC 專案健康狀況的良好判斷力與責任感，並由社群選舉產生。TOF 成員
_MUST_ 為專案設定技術方向、做出或核准設計決策、確保遵循本文件及其他文件中的
規則，並促進與更廣泛開源社群的互動。

**Defined by:** docs 儲存庫中的 TOF 成員名單

### Expectations

- 深入理解專案的技術目標與方向
- 認識社群成員、專案內外部的技術專家，並有參與互動的紀錄。
- 對 OpenBMC 及 BMC 整體的技術領域有深入理解。
- 透過以下所有方式持續對設計與方向做出貢獻：
  - 對 technical-oversight-forum 儲存庫的提案進行撰寫、審查與投票
  - 發起、參與並解決涉及全專案範圍變更與期望的討論（郵件、GitHub issue、會議）
  - 找出跨專案發生的設計與實作上細微或複雜的問題。
  - 透過測試確保專案功能正常（盡可能地）適用於所有平台
  - 確保編碼標準、專案慣例與社群準則在整個專案中都有被文件化並遵循。
- 透過實作與（或）審核直接對專案做出貢獻

### Responsibilities and privileges

- 為 OpenBMC 做出並核准技術設計決策。
- 定義里程碑與發布版本。
- 指導並引導專案的 approver、reviewer 與貢獻者。
- 建立新的子專案，並確保其加入能持續促進專案的成長與健康。
- 定義並維護健康的討論與決策流程。
- 與 subproject owner 及 platform maintainer 合作，整體性地維護專案的總體
  健康與成功。

[new contributors]: /CONTRIBUTING.md#starting-out
[code reviews]: https://gerrit.openbmc.org/
[mailing list]: https://lists.ozlabs.org/listinfo/openbmc
[main OpenBMC repository]: https://github.com/openbmc/openbmc
[community expectations]: /code-of-conduct.md
[submit a cla]: /CONTRIBUTING.md#starting-out
[contributor guide]: /CONTRIBUTING.md
[tof election cycle]: /tof/membership-and-voting.md
[tof-rollcall]: https://github.com/openbmc/tof-election
