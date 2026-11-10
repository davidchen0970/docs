# OpenBMC meta layer 指引

雖然這些內容大多可以被稱為「規則」，但在特定情境下，遵循它們可能反而明顯違背原本
想要的目標。這也是為什麼每一項指引都附有一個很長的「為什麼」章節，以確保背後的意圖能夠
被滿足，並且當規則確實存在例外時，專案社群能夠理解並加以管理。一般來說，如果有任何疑問，
而且社群也達成共識，這些指引可以視個案情況予以豁免。

## Meta layer 不應該 patch 存在於 openBMC tree 中的專案

**為什麼？**

總體而言，讓 codebase 長期維持可建置的狀態本來就不容易。一旦允許存在 repo 維護者未知的
patch，就會大幅提高單一機器壞掉，或是我們出現兩個 repo 之間行為差異的機率。

此外，一般來說維護者的職責是確保整體社群、功能與 codebase 的優先級高於任何單一 patch，
而他們通常是在 code review 中達成這件事。如果 patch 被直接 check in 到 meta layer 裡，
維護者通常就不是 review 的參與者，等於大致上破壞了維護者這個角色的存在意義。

**那我應該怎麼做？**

與專案維護者及社群討論，確認你要開發的功能是否需要可設定，還是可以套用到所有專案上。
如果可以套用到所有專案，直接透過 gerrit review check in 到 master branch，並遵循該
repository 既定的流程即可。如果它需要依專案或依機器做設定，請在維護者的建議下，以一個
編譯期選項的形式 check in，並新增一個可以設定的 PACKAGECONFIG entry
來啟用該功能。

## Meta layer 不應該 patch Yocto recipe 與專案

**為什麼？**

Yocto 本身就是一個接受貢獻的開放原始碼專案。OpenBMC 針對 Yocto recipe 堆疊的修改越多，
就越難以維護，要 rebase 到新版 Yocto 所需的時間也越長。一般來說，Yocto 社群在處理
pull request 時的回應速度，與 OpenBMC 社群相比一樣快（有時候甚至快得多）。

**那我應該怎麼做？**

將所有需要的修改，依照 Yocto 上游 repository 的流程送上去。如果 Yocto 的流程已經好幾個
週都無人回應，就把該 commit cherry-pick 進 OpenBMC tree，並在 commit message
與文件中附上該 review 的指標。patch 的 commit message 應遵循 Yocto 對 patch 註解
的要求，包括說明此 patch 的上游狀態。
[yocto patch requirements](https://wiki.yoctoproject.org/wiki/Best_Known_Methods_%28BKMs%29_for_Package_Updating#Patch_Comments)

## Meta layer 應避免使用 EXTRA_OEMAKE 與 EXTRA_OEMESON

**為什麼？**

有些 OpenBMC 專案會被用在 OpenBMC 之外。因此，存在一些並非設計給 OpenBMC 使用的設定項，
或是會帶來安全風險的設定項。另外，當選項變動或被棄用時，專案需要有一個單一的地方
來更新可用的設定項與相依項目。

此外，子專案可能會變更其建置工具，例如為了追求其他目標而從 autotools 改用 meson。使用與
特定工具綁定的設定，會讓這種變更困難得多。

**那我應該怎麼做？**

在根 recipe 中，為該功能新增一個 PACKAGECONFIG entry，然後在你的 meta layer
中使用它來啟用該功能。

## Meta layer 不應該有指向專有授權程式碼的 recipe

**為什麼？**

OpenBMC 是一個開放原始碼專案，其設計是要從原始碼建置，並搭配適當的散布授權，使其能夠被
重複使用。指向商業授權的 repository 與這個目標背道而馳。

**那我應該怎麼做？**

找一個能滿足需求的等效開放原始碼專案，或是請求該專案的擁有者將其重新授權。

## Meta layer 的 recipe 應該只指向維護良好的開放原始碼專案

**為什麼？**

若沒有這條指引，就會存在一個漏洞，讓 OpenBMC 開發者得以繞過 code review：方法是把
上游 recipe 指向一個由他們自己掌控、但 OpenBMC 對其內容毫無話語權的公開 repository。
這會讓討論園地以毫無生產力的方式分裂，並使 OpenBMC 內其他所有良好的流程，例如 bug
tracking 與 continuous integration，都無法發揮作用。

**那我應該怎麼做？**

建議多半要視個案情況而定，但如果這段程式碼只打算在 OpenBMC 上使用，那麼請先推出設計文件，
並把程式碼推到 openbmc gerrit 上 openbmc/openbmc repository 底下，讓它能被
review，同時附上一個 OWNERS 檔，表明你願意維護這個專案。接著，一旦社群審閱過你的
設計之後，就會建立一個 repository 供你推送程式碼。如果你是要引入一個已停止維護專案的程式
碼，請透過 mailing list 或 discord 向社群詢問，OpenBMC 社群是否願意接手該專案的支援
與維護。

## Meta layer 不應該指向 <https://github.com/openbmc> 之外、屬於 OpenBMC 專屬的 repository

**什麼定義了一個 OpenBMC 專屬的 repository？**

一般來說，一個 OpenBMC 專屬的 repository 是指符合下列任一條件的東西：

- 依賴或實作由 OpenBMC 定義的 Dbus 介面。
- 並非以「可透過設定在 OpenBMC 之外使用」為設計考量
- 依賴 OpenBMC 專屬的佈局與設定資訊來完成其主要功能。

**為什麼？**

OpenBMC 作為一個 codebase，需要長期維持可建置的狀態。公司、維護者與貢獻者會在專案的歷程中
來來去去。把所有需要長期支援的程式碼都保留在 OpenBMC github 中，能確保這個專案持續
有用。此外，把大量 OpenBMC 專屬的程式碼放在 github 之外，會限制專案進行重構的能力，
而為了跟上最新的最佳實踐、Yocto 更新與 package 改版，這類重構可能是必要
的，因為當 repository 不在 OpenBMC github 專案之內時，並不一定有一條清楚的
路徑可以讓 patch 被接受

**那我應該怎麼做？**

透過 mailing list 及／或 discord，與社群討論你想撰寫的程式碼。如果這個設計需要一個新的
repository，請使用 OpenBMC technical oversight committee 的流程來提出申請。

## 不要在 recipe 中使用 SRCREV="${AUTOREV}"

**為什麼？**

Repository 的 branch 隨時可能變動。指向 autorev revision 會提高建置壞掉的機率，並讓
建置大幅失去可重現性。

此外，能精確掌握建置內容包含了哪些東西，可以避免一種錯誤：在開發過程中 repo
被悄悄更新，內容突然發生重大變化。

**那我應該怎麼做？**

把 SRCREV 指向 repository 中某個特定的 commit，並提升 revision——方法是透過 CI
中的 autobump script（可在 mailing list 上提出申請），或是在有新 revision 出現時手動
提升。

## Meta-layer 不應包含已編譯的程式碼

**為什麼？**

此專案的架構是將程式碼 repository 與 Yocto meta-layer repository 分開，而且每一種
repository 型別都各自有明確的 continuous-integration 式測試。把程式碼放在 meta-layer
中，等於放棄了這套 CI 基礎設施的可用之處，也排除了我們既有的許多類型測試與 linting。

Meta-layer 通常被設定成由公司內部的開發者擁有 merge 權限。把程式碼放進這些 meta-layer，
會讓不符合整體期望的做法在缺乏監督的情況下發生，並導致不鼓勵協作的公司專屬實作與方法。

再者，歷史上 Yocto 對放在 meta-layer 中的程式碼，其支援程度不如放在獨立 git repository
或 archive 中的程式碼。已知這會在下游的 OpenBMC fork 中造成編譯問題。

**那我應該怎麼做？**

找一個與你目的相符的現有程式碼 repository，或是提出建立一個新的。
