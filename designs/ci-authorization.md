# OpenBMC 的持續整合與授權

作者：Brad Bishop !radsquirrel

其他貢獻者：無

建立日期：2019-01-30

## 問題描述

OpenBMC 專案維護著許多 Jenkins CI job，以確保進入專案原始碼的貢獻內容達到一定的品質水準。貢獻可以由大眾提出——任何擁有 GitHub 帳號的人都可以。雖然可能性不大，但有心人士仍有可能提交試圖危害專案資源（例如建置系統）的程式碼，因此必須對貢獻者進行某種程度的授權，以提供針對潛在有心人士的一定程度保護。

專案目前已有用於 CI 的貢獻者授權機制。本提案旨在說明現有做法的缺點，並提出一個能解決這些缺點的替代方案。

## 背景與參考資料

現有的授權做法會檢查使用者是否為 openbmc/general-developers GitHub team 的成員。如果貢獻者是該 team（或 general-developers 下屬 sub-team）的成員，自動化 CI 流程就會在無需人工介入的情況下觸發。如果貢獻者不是 general-developers team 的成員，則需要專案維護者進行人工介入（ok-to-test）才能觸發自動化 CI 流程。

延伸閱讀：

- <https://en.wikipedia.org/wiki/Continuous_integration>
- <https://jenkins.io/>
- <https://help.github.com/articles/about-organizations/>

## 需求

現有的授權方式有一個核心問題——GitHub organization owner 角色。若要將貢獻者加入 openbmc/general-developers GitHub team，貢獻者必須先是 openbmc GitHub organization 的成員。只有 organization owner 才能邀請 GitHub 使用者成為 organization 的成員。Organization owner 對專案的各個層面都擁有不受限制的存取權限——僅為了賦予管理 openbmc/general-developers 群組成員的能力就授予 organization 擁有權，是不明智的做法。

一套替代的 CI 授權方式應該要：

- 不需要 GitHub organization owner 角色就能管理獲准使用 CI 的使用者清單。
- 支援階層式的信任模型來進行使用者授權（群組可巢狀於群組之中）。

## 建議設計

提案內容是把目前的 openbmc/general-developers GitHub team 及其所有下屬 team 單純地遷移至 Gerrit group：

group: `openbmc/ci-authorized`

group: `xyzcorp/ci-authorized`

group: `abccorp/ci-authorized`

openbmc/ci-authorized 這個 group 除了可以包含與任何特定組織無關的使用者之外，也可以包含組織群組：

group: `openbmc/ci-authorized` contains ->

group `xyzcorp/ci-authorized`

group `abccorp/ci-authorized`

user `nancy`

user `joe`

本提案另外為組織群組的管理指定了一項慣例：

group: `xyzcorp/ci-authorized-owners` administers -> `xyzcorp/ci-authorized`

group: `abccorp/ci-authorized-owners` administers -> `abccorp/ci-authorized`

group: `openbmc/ci-authorized` administers -> `openbmc/ci-authorized`

最後，所有 Jenkins CI job 都必須更新為改以檢查是否為 Gerrit group 的成員，而非 GitHub team 的成員。

當 CCLA 簽署並獲專案接受後，就會建立新的組織群組（以及相對應的擁有者群組）。

## 曾考量的替代方案

曾考慮過將 GitHub organization owner 角色指派給組織群組的管理員，但這嚴重違反最小權限原則。

## 影響

GitHub 在負載平衡與備份能力上遠為優越，因此可能會降低服務可用性並有資料遺失的風險。

## 測試

部署到正式的生產伺服器上 😀
