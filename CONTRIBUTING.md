# 參與 OpenBMC 貢獻

首先，感謝您撥空貢獻！以下是一組貢獻由 GitHub 上
[OpenBMC Organization](https://github.com/openbmc) 託管的 OpenBMC 倉庫時
應遵循的準則。歡迎自由對本文件提出修改建議。

## 行為準則

我們熱忱地遵循我們的
[行為準則](https://github.com/openbmc/docs/blob/master/code-of-conduct.md)。
若您有任何疑慮，請查閱該文件以了解該由誰協助您解決。

## 包容性命名

OpenBMC 仰賴 Open Compute Project 提供包容性命名的準則。OCP 的準則可於此處找到：

<https://www.opencompute.org/documents/ocp-terminology-guidelines-for-inclusion-and-openness>

## 架構

OpenBMC 具有相當模組化的架構，由職責範圍有限的小型 daemon 所組成。這些
daemon 透過 D-Bus 與其他元件通訊，共同實作出完整的 BMC 系統。

BMC 對外界的介面通常透過個別的 daemon 來處理，再由這些 daemon 將那些請求
轉換為 D-Bus 訊息。

這些獨立的專案接著由整體的 'openbmc' 建置基礎設施編譯成最終的系統。

在未來的開發中，請牢記這個設計。各元件的功能應以邏輯方式分組，並在合理時
將相關的部分放在一起。

## 起步

在您做出貢獻之前，請先簽署一份 OpenBMC 貢獻者授權協議（Contributor License
Agreement），_僅需一次_：

- [個人 CLA](https://drive.google.com/file/d/1k3fc7JPgzKdItEfyIoLxMCVbPUhTwooY)
- [企業 CLA](https://drive.google.com/file/d/1d-2M8ng_Dl2j1odsvZ8o1QHAdHB-pNSH)

若您任職於下列任一類別的公司，則必須由您的雇主先簽署企業 CLA（CCLA），
您才可進行貢獻：

- 晶片、韌體或伺服器／硬體供應商
- 雲端服務供應商（CSP）——例如 Meta、Google、Microsoft、Amazon 或類似的
  超大規模資料中心業者（hyperscaler）
- 其他任何與 OpenBMC 處於相同技術領域、且可能合理對本專案主張智慧財產權
  的公司

CCLA 可保護本專案，避免公司日後主張某位員工未經授權貢獻了專有 IP。一旦
您的雇主簽署了 CCLA，該公司的其他貢獻者就能被加入既有的 CCLA Schedule A，
而不需要各自單獨簽署。

若您是個人貢獻者，且並非任職於上述任一類別的公司，則個人 CLA 可能就已足夠。

簽署 CLA 後，請將其寄送至 <manager@lfprojects.org>。

如果您在尋找入門 OpenBMC 的起點，可以看看標記為 'bitesize' 的 issue。這些
是不需要對 OpenBMC 程式碼庫有深入知識即可處理的修復或增強，對新手來說比較
容易上手。

請在此處查看該清單：

<https://github.com/issues?utf8=%E2%9C%93&q=is%3Aopen+is%3Aissue+user%3Aopenbmc+label%3Abitesize>

如果您需要其中任何 issue 的更多細節，歡迎隨時加上留言。

進行程式碼審查（code review）另一個很好的入門方式。前往
<https://gerrit.openbmc.org> 並點擊「all」與「open」選單項目，或者若您對
某個特定倉庫有興趣——例如 "bmcweb"——在搜尋列輸入
"status:open project:openbmc/bmcweb" 並按「search」按鈕。然後挑選一個審查。
記得保持正面態度，讓每則審查意見都能帶來價值。

## 程式碼風格

OpenBMC 原始碼的各部分應保持一致的風格。若原始碼來自其他專案，我們選擇遵循
該上游專案既有的風格。否則，慣例依程式語言而定。

### Python

所有 Python 原始碼都應符合 PEP8。此風格在 Python 社群中已廣為建立，並可透過
'pycodestyle' 工具驗證。

<https://www.python.org/dev/peps/pep-0008/>

### Python 格式化

若某個倉庫的根目錄中存在 setup.cfg 檔案，CI 就會自動驗證 Python 程式碼是否符合
[pycodestyle](http://pycodestyle.pycqa.org/en/latest/intro.html) 的要求。
這會對所有 Python 程式碼強制施行 PEP 8 標準。

OpenBMC 的 Python 標準與 PEP 8 一致，因此一般來說只需要一個空白的 setup.cfg
檔案即可。如有需要，將字元上限強制為 80（而非預設的 79）也是可以的：

```toml
[pycodestyle]
max-line-length = 80
```

預設情況下，pycodestyle 不會強制施行以下規則：E121、E123、E126、
E133、E226、E241、E242、E704、W503 以及 W504。這些規則被忽略是因為它們並非
共識，且 PEP 8 並未強制施行。是否強制施行前述規則由倉庫維護者自行決定。這些
規則可以透過在 setup.cfg 中加入以下內容來強制施行：

```toml
[pycodestyle]
ignore= NONE
```

### JavaScript

我們遵循
[Google JavaScript Style Guide](https://google.github.io/styleguide/jsguide.html)。

[.clang-format 範例](https://www.github.com/openbmc/docs/blob/master/style/javascript/.clang-format)

### HTML/CSS

我們遵循
[Google HTML/CSS Style Guide](https://google.github.io/styleguide/htmlcssguide.html)。

### C

對於 C 程式碼，我們通常使用 Linux 的程式碼風格，其文件記錄於：

<http://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/tree/Documentation/process/submitting-patches.rst>

簡而言之：

- 使用 tab 而非空格縮排，設定為 8 個欄位

- 每行 80 欄

- 左大括號放在行尾，函式除外

此風格大致可以透過 'astyle' 驗證，方式如下：

```bash
astyle --style=linux --indent=tab=8 --indent=force-tab=8
```

### C++

請參閱 [C++ 風格與慣例](./cpp-style-and-conventions.md)。

## 規劃變更

若您要進行的變更並非瑣碎之作，就應該規劃如何將其導入 OpenBMC 開發社群。

若您正在規劃新功能，應該先取得共識，確認您的變更會被接受。請盡早透過 OpenBMC
Discord 伺服器或電子郵件郵件列表引入這個變更以展開討論。您可能會發現執行您所
需要之事更好的做法。

若回饋看起來正面，或要求提供更多細節，請接著以
[design_template](https://github.com/openbmc/docs/blob/master/designs/design-template.md)
的複本為起點，將設計提交到 gerrit。

## 提交變更

我們幾乎所有事情都使用 git。OpenBMC 中的多數專案使用 Gerrit 來審查補丁。對上游
專案（例如 Yocto Project、systemd 或 Linux kernel）的變更可能需要以補丁或 git
pull request 的形式送出。

### 組織 Commit

一個好的 commit 只做一件事。我們偏好許多小型、原子性的 commit，而不是一個做出
大量功能變更的大型 commit。

- 太大：「將 foo 轉換為新 API；同時修復 bar 中的 CVE 1234」
- 太小：「把 abc.h 搬到 include 清單最上面」以及「把 xyz.h 搬到 include 清單最下面」
- 剛好：「將 foo 轉換為新 API」以及「將 foo 的 tab 轉換為空格」

其他 commit 小技巧：

- 若您的變更有一份規格書，先勾勒您的想法並設法讓它被接受，再完成細節。
- 最多工作幾天後就尋求審查。
- 在撰寫程式碼之前，先提交並審查標頭檔。
- 將每個實作模組分開提交並審查。

通常以這種方式建立小型 commit 會產生許多相依於先前 commit 的 commit；Gerrit
能很好地處理這種情況，所以請放心推送那些以仍在審查中的變更為基礎的 commit。
不過，在可能的情況下，您的 commit 應該能獨立位於 master 之上——「Fix whitespace
in bar()」不需要相依於「refactor foo()」。換句話說，請確保語意上彼此無關的主題，
在合併進 master 之前，在 Git 中也彼此無關聯。

當您推送一組補丁堆疊（stack of patches）時，這些 commit 會在 Gerrit 中以相同的
相依關係出現。這代表每個補丁都必須依該相依關係的順序合併。因此若中間某個補丁
需要修改，從該點之後的所有補丁都需要重新推送以維持該相依關係。這實際上會對未
變更的補丁重新變基（rebase），進而觸發新的 CI 建置。理想上，整個補丁集（patchset）
的變更可以一次全部完成，以減少不必要的變基。

當某個人在 Gerrit 上對您的 commit 留言時，請修改該 commit 並再次送到 Gerrit。
這通常會涉及 `git rebase --interactive` 或 `git commit --amend`，網路上有許多
相關指南。如同上一段所述，在可能的情況下，您應該在推送之前先對同一堆疊中的多個
補丁做出修改，以盡量減少變基作業帶來的 CI 與通知噪音。

包含可由單元測試驗證之變更的 commit，也應在同一個 commit 中包含一個單元測試來
驗證該變更。單元測試應該撰寫得清楚明瞭——相比正式程式碼，單元測試主要是讓人閱讀
的——並且應該同時測試正確與錯誤的行為。請參考
[測試文件](https://github.com/openbmc/docs/blob/master/testing/local-ci-build.md)
以取得撰寫測試的協助以及最佳實踐。

請確保您的補丁沒有改動無關的區域。務必留意不小心造成的空白字元變更——這會讓審查
變得不必要的困難。

### 格式化 Commit 訊息

您的 commit 訊息應該說明：

- 簡潔地，您正在做出 _什麼_ 變更？例如「libpldm: Add APIs to enable
  PLDM Requester」。您 commit 訊息的第一行就是主旨行（subject line）。
- 完整地，您 _為什麼_ 要做那個變更？在某些情況下，例如小型重構，原因相當明顯。
  但在例如納入新功能的情況下，您應該說明為什麼需要該功能。
- 簡潔地，您 _如何_ 測試？（見下文）。

設法在您的主旨行開頭納入您正在修改的元件；這通常採用您所修改的類別、模組、處理
常式或目錄的形式。例如「apphandler: refactor foo to new API」

Commit 訊息應遵循 50/72 規則：主旨行不應超過 50 個字元，內文不應超過 72 個字元。
這在許多使用 Git 的專案中是常見做法。

例外情況允許以連結的形式呈現，可以表示為：

```markdown
This implements [1]

....

[1]: https://openbmc.org/myverylongurl.
```

所有 commit 訊息都必須包含一行 "Signed-off-by"，表示您這位貢獻者已同意開發者出處
授權書（Developer Certificate of Origin，見下文）。這一行必須包含您慣用的全名，
通常是名字加上家族姓氏或姓。（可以：Sam Samuelsson、Robert A. Heinlein；不可以：
xXthorXx、Sam、RAH）

### 測試

每個 commit 都預期要經過測試。測試的預期因子專案而異，但通常會包括執行所有適用的
自動化測試並進行手動測試。每個 commit 都應該分開測試，即使它們是一起提交的（例外
情況是不同專案之間的 commit 彼此相依時）。

Commit 訊息應包含一個 "Tested" 欄位，描述您如何測試補丁中的程式碼變更。範例：

```text
    Tested: I ran unit tests with "make check" (added 2 new tests) and manually
    tested on Witherspoon that Foo daemon no longer crashes at boot.
```

若該變更需要手動測試，請描述您做了什麼以及發生了什麼事；若在您變更之前它的行為
不同，也請一併描述。若該變更可以完全透過執行單元測試來驗證，請在 "Tested:" 欄位
中如此說明。

### Linux Kernel

Linux kernel 中的準則非常有幫助：

- <https://www.kernel.org/doc/html/latest/process/submitting-patches.html>
- <https://www.kernel.org/doc/html/latest/process/submit-checklist.html>

您的貢獻通常需要經過審查才會被接受。

## 透過 Gerrit 伺服器將變更提交到 OpenBMC

OpenBMC Gerrit 伺服器支援 GitHub 憑證，其連結為：

<https://gerrit.openbmc.org/#/q/status:open>

_一次性設定_：使用您的 GitHub 憑證登入 WebUI，並在您的 Account Settings 中確認
您的 SSH 金鑰已被匯入：

<https://gerrit.openbmc.org/#/settings/>

多數倉庫都由 Gerrit 伺服器支援，目前的清單可於 Projects -> List 之下找到：

<https://gerrit.openbmc.org/#/admin/projects/>

如果您將經常使用 Gerrit，在 ~/.ssh/config 中建立一個 SSH host 區塊會很有用。例如：

```text
Host openbmc.gerrit
        Hostname gerrit.openbmc.org
        Port 29418
        User your_github_id
```

從您的 OpenBMC git 倉庫，新增一個指向 Gerrit 伺服器的 remote，其中
'openbmc_repo' 是您目前正在處理的 git 倉庫，例如 phosphor-state-manager，而
'openbmc.gerrit' 是您先前新增的 Host 名稱：

`git remote add gerrit ssh://openbmc.gerrit/openbmc/openbmc_repo`

或者，您可以將所有資料編碼在 URL 中：

`git remote add gerrit ssh://your_github_id@gerrit.openbmc.org:29418/openbmc/openbmc_repo`

接著為推送到這個 remote 新增預設分支：

`git config remote.gerrit.push HEAD:refs/for/master`

Gerrit 使用
[Change-Ids](https://gerrit-review.googlesource.com/Documentation/user-changeid.html)
來識別屬於同一次審查的 commit。請設定您的 git 倉庫，讓它自動在 commit 訊息中加入
Change-Id。步驟為：

```bash
gitdir=$(git rev-parse --git-dir)
curl https://gerrit.openbmc.org/tools/hooks/commit-msg -o ${gitdir}/hooks/commit-msg
chmod +x ${gitdir}/hooks/commit-msg
```

若要提交一組變更，請 commit 您的變更並推送到 Gerrit 伺服器，其中 'gerrit' 是用
git remote add 指令新增的 remote 名稱：

`git push gerrit`

有時您需要指定一個 topic（特別是當您處理橫跨數個專案的功能時），或（取消）標記
變更為進行中（Work-in-Progress）。為此請參考
[Gerrit 文件](https://gerrit-review.googlesource.com/Documentation/intro-user.html#topics)。

Gerrit 會顯示指向您審查的 URL 連結。您也可以從
[OpenBMC Gerrit 伺服器](https://gerrit.openbmc.org)的搜尋列或選單（All > Open，
或 My > Changes）找到您的審查。

邀請審查者來審查您的變更。每個 OpenBMC 倉庫都有一份 `OWNERS` 檔案，列出作為領域
專家的必要審查者。這些審查者可以再新增其他審查者。若要從 Gerrit 網頁新增審查者，
請點擊審查者清單旁的「add reviewers」圖示。

您應該回應所有意見。並記得使用「reply」按鈕發布您的回覆，讓其他人能看到。

審查的結果會是提案的變更為被接受、被駁回要求重做，或被放棄。當您重做變更時，記得
使用 `git commit --amend`，讓 Gerrit 將更新處理成同一次審查的新補丁。

每個倉庫都由一小群維護者治理，他們是在各自領域具備專業知識的領導者。他們負責審查
變更並維持程式碼品質。您需要一位維護者接受您的變更，而他們會參考其他審查者的意見。
一旦獲得接受，您的變更就會合併進 OpenBMC 專案。

## 引用非公開資源

程式碼與 commit 訊息不得引用公司的內部文件或系統（包括 bug 追蹤系統）。其他開發者
可能無法存取這些資源，這會讓未來的維護變得困難。

貢獻給 OpenBMC 的程式碼必須能從公開可取得的原始碼建置，且不得相依於非公開資源
（URL、倉庫等）。

## D-Bus 介面的最佳實踐

- 新的 D-Bus 介面應該可重複使用

- 型別簽章應該盡可能使用最簡單、且適合所傳遞資料的型別。例如，不要把數字當做
  ASCII 字串傳遞。

- D-Bus 介面定義於 `phosphor-dbus-interfaces` 倉庫中，位於：

<https://github.com/openbmc/phosphor-dbus-interfaces>

參閱：<http://dbus.freedesktop.org/doc/dbus-api-design.html>

## C 的最佳實踐

其他地方已有許多資源，以下是幾項與 OpenBMC 工作相關的要點：

- 不要使用 `system(<some shell pipeline>)`。讀寫檔案應該使用適當的系統呼叫。

  一般來說，若您確實需要產生（spawn）一個行程，尤其是需要提供變動參數時，使用
  `fork(); execve()` 好得多。

- 對需要特定大小的資料使用 `stdint` 型別（例如 `uint32_t`、`int8_t`）。如有需要，
  請使用 `PRIxx` 巨集供 printf 使用。

- 不要假設 `char` 是有號號或無號。

- 在讀取至 C 型別及從 C 型別寫出時，務必留意位元組序（endian）的考量

- 在可能的情況下，將僅內部使用的函式宣告為 `static`，唯讀資料宣告為 `const`。

- 確保您的程式碼可以在不產生警告的情況下編譯，對 kernel 的變更尤其如此。

## Systemd 使用的最佳實踐

- Systemd 服務應該包含在與之相關的 OpenBMC 倉庫內

- Systemd 服務應該使用 Wants/After 列出它們的相依關係，以確保必要的服務會被啟動

- 若您的倉庫提供一個共用函式庫，它應該妥善處理其所具有的任何 D-Bus 相依關係

  這可以是清楚文件化會回傳哪一種錯誤或例外，也可以是確保所需的 D-Bus 服務在被呼叫
  時會自動啟用。

- 請參閱這份
  [文件](https://github.com/openbmc/docs/blob/master/architecture/openbmc-systemd.md)
  以取得關於 OpenBMC 及其 Systemd 使用的更多資訊

## 審查的節奏

習慣由所屬團隊在公司內部進行程式碼審查的貢獻者，或完全不熟悉程式碼審查的貢獻者，
有時會對開源專案中程式碼審查的節奏感到意外。請設法牢記，審查您補丁的人可能是以
志工或業餘時間的方式參與 OpenBMC，可能位於與您相距遙遠的時區，而且可能已經有很深
的審查佇列，其中的補丁等待時間比您的更久。請盡一切努力讓審查者能輕鬆審查您的貢獻。

若您覺得您的補丁完全被忽略了，當然，寄信給維護者（位址可在 OWNERS 檔案中找到）或在
Discord 上提醒他們是可以的——但合理的時間範圍大約是一週，而不是幾小時。

維護者的工作是確保進來的補丁盡可能正確且易於維護。人員流動是開源本質的一部分——
貢獻者很容易來來去去——所以維護者通常不會相信諸如「我會在後續的補丁中加入單元測試」
或「我會在下個月結束前實作這個提案」這類承諾。這常表現為看起來嚴苛或苛求的審查；
請牢記社群是想與您合作，共同打造一個本身就能讓本專案受益的補丁。

## 開發者出處授權書 1.1

```text
    By making a contribution to this project, I certify that:

    (a) The contribution was created in whole or in part by me and I
        have the right to submit it under the open source license
        indicated in the file; or

    (b) The contribution is based upon previous work that, to the best
        of my knowledge, is covered under an appropriate open source
        license and I have the right under that license to submit that
        work with modifications, whether created in whole or in part
        by me, under the same open source license (unless I am
        permitted to submit under a different license), as indicated
        in the file; or

    (c) The contribution was provided directly to me by some other
        person who certified (a), (b) or (c) and I have not modified
        it.

    (d) I understand and agree that this project and the contribution
        are public and that a record of the contribution (including all
        personal information I submit with it, including my sign-off) is
        maintained indefinitely and may be redistributed consistent with
        this project or the open source license(s) involved.
```

## 著作權標示

對 OpenBMC 的貢獻不強制要求包含明確的著作權聲明，但某些公司可能有其公開貢獻必須
包含聲明的政策。對本專案的貢獻只能使用 [SPDX 檔案標籤][spdx]，且不得包含其他著作權
或授權區塊。遵循 Linux Foundation 關於著作權聲明的
[建議][lf-copyright]，貢獻應使用「Copyright OpenBMC Authors」這個措辭，而非逐一
列出各別專案成員。合在一起，SPDX 識別碼應為：

```cpp
// SPDX-License-Identifier: Apache-2.0
// SPDX-FileCopyrightText: Copyright OpenBMC Authors
```

若程式碼是重複使用自既有的開源專案，且貢獻者並非原始作者，則應保留既有的著作權及／或
授權區塊。請勿移除您無權移除的著作權聲明！此例外不適用於您現在貢獻給 OpenBMC 的、
先前非公開的程式碼。所有直接對 OpenBMC 的貢獻都應使用上述的 SPDX 記錄。

[spdx]:
  https://github.com/spdx/spdx-spec/blob/support/2.3.1/chapters/file-tags.md#annex-h-specifying-spdx-information-in-source-files-informative
[lf-copyright]:
  https://www.linuxfoundation.org/blog/blog/copyright-notices-in-open-source-software-projects
