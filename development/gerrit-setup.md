# OpenBMC Gerrit 設定／整合

**文件目的：** 逐步說明如何設定你的工作站及一個 Gerrit 帳號。若要參與
OpenBMC 基於 Gerrit 的程式碼審查，這是必要的。

**前置需求：** 現代的 Linux、Mac 或 Windows 系統。已安裝 Git 套件。
一個電子郵件帳號（建議 Gerrit 與 GitHub 使用相同的電子郵件地址）。

## 初始設定

### 更新 Git 身分資訊

- `git config --global --add user.name "Your name" (eg. John Smith)`
- `git config --global --add user.email "youremail@your-domain" (eg. jsmith@somedomain.com)`
- （選填）
  `git config --global --add diff.tool "preferred diff tool" (eg. gvimdiff or meld)`

### 設定 SSH 金鑰

建立金鑰：`ssh-keygen -t ed25519 -C "your_email@your-domain"`

- 建議使用預設值，而不要自行選取目錄／檔案名稱。

將金鑰新增到 GitHub：

- <https://help.github.com/articles/adding-a-new-ssh-key-to-your-github-account/>

將金鑰新增到 OpenBMC 的 Gerrit 伺服器：

- 使用你的 GitHub 帳號登入 [Gerrit](https://gerrit.openbmc.org/)。
- 前往 <https://gerrit.openbmc.org/plugins/github-plugin/static/account.html>
- 你的資訊應該會自動填入，因此點擊「Next」。
- 若成功，你應該會看到一個空白頁面，可以直接關閉離開。

**注意** 若在將金鑰新增到 Gerrit 時收到「Not found」錯誤，你可以手動新增：

- 登入 [Gerrit](https://gerrit.openbmc.org/)
- 在 Settings -> SSH Keys -> New SSH key 中輸入你先前建立的公開 SSH 金鑰
- 點擊「ADD NEW SSH KEY」
- 若成功，你應該會看到你的公開金鑰已新增，且狀態為「Valid」

### 將電子郵件新增到 Gerrit

- 登入 [Gerrit](https://gerrit.openbmc.org/)
- 在 Settings -> Contact Information -> Register New E-Mail 中輸入電子郵件
- 檢查你的電子郵件以取得確認信，並點擊連結進行確認

### 將全名新增到 Gerrit

- 在 Settings -> Profile -> Full name 中輸入你的全名

### 新增 SSH 設定項目

將以下內容新增到 `~/.ssh/config`：

```bash
Host openbmc.gerrit
    Hostname gerrit.openbmc.org
    Port 29418
    User <Your Gerrit Username>
```

- **注意**：AFS 有一個缺陷，若使用 AFS，需要在 SSH 項目中新增 `AFSTokenPassing no`。
- 你的 Gerrit 使用者名稱可以在 Gerrit 的 Settings -> Profile ->
  Username 中找到
- 確保你的 .ssh 目錄有正確的權限：`chmod 600 ~/.ssh/*`

### 確認設定成功

嘗試複製一個 repository 來測試與 Gerrit 的連線

- `git clone ssh://openbmc.gerrit/openbmc/docs`
- 若成功，你應該會看到類似以下的內容：
  `Checking out files: 100% (45/45), done.`

### 新增 Hooks

在你剛複製的 repository 內，輸入以下指令：

```bash
gitdir=$(git rev-parse --git-dir)
curl https://gerrit.openbmc.org/tools/hooks/commit-msg -o ${gitdir}/hooks/commit-msg
chmod +x ${gitdir}/hooks/commit-msg
```

這會增強 `git commit` 指令，在你的提交訊息中加入一個 `Change-Id`，
Gerrit 會用它來追蹤審查。

### 將程式碼變更推送到 Gerrit

現在你的工作站與 Gerrit 都已設定完成，你可以開始進行程式碼變更，
並將它們推送到 Gerrit 進行程式碼審查。基本的工作流程會如下所示。

- 進行你的程式碼變更
- 將那些檔案加入索引以便提交：
  `git add [file1 file2 ... fileN]`
- 提交你的變更，並在其中加入一行 Signed-off-by（更多關於
  [撰寫良好的提交訊息](https://github.com/openbmc/docs/blob/master/CONTRIBUTING.md#submitting-changes)）：
  `git commit --signoff`
  - 若要更新一個現有的審查（相同的 `Change-Id`），改用 amend：
    `git commit --amend`
- 將你的變更推送到 Gerrit 進行程式碼審查：
  `git push origin HEAD:refs/for/master`
- 前往 [Gerrit 網頁介面](https://gerrit.openbmc.org/)，點擊你的新審查，
  並根據 repository 中的 `OWNERS` 檔案加入審查者。

### 以 Topic 為基礎的跨 Repository 測試

當一個變更相依於其他 OpenBMC repository 中的變更時，使用一個 Gerrit
topic，讓 CI 能把相關的變更一起建置。請參閱
[郵件清單討論](https://lore.kernel.org/openbmc/Zv2oBkO0J6OEUvie@heinlein.vulture-banana.ts.net/)。

```bash
git push origin HEAD:refs/for/master -o topic=my-feature
```

#### 注意事項

- 對所有相關的變更使用相同的 topic。
- CI 的相依共同建置（co-requisite build）邏輯，對於給定的 topic，
  每個 repository 只支援一個有效的變更。
- 若同一個 repository 的多個變更共用一個 topic，CI 可能不會選取預期的
  相依項目。請只將 topic 保留在每個 repository 中最新的變更上。

## 結論

若你已順利完成上述所有步驟，就這樣！你現在已經完成 Gerrit 的設定，
也知道如何提交你的程式碼變更進行審查了！

提交變更進行審查只是貢獻流程中的眾多步驟之一。請參閱
[CONTRIBUTING](https://github.com/openbmc/docs/blob/master/CONTRIBUTING.md)
以了解最佳實務做法。
