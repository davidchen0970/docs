# 本地端 CI 建置

以下的說明適用於在本地端執行上游 OpenBMC CI。

## 安裝 Docker

請先安裝並設定 Docker。Docker CE（社群版）的安裝方式會依平台而異，在你所屬的組織中也可能不盡相同，不過 [Docker 文件](https://docs.docker.com/install/)是一個很好的入門參考。

執行 `sudo docker run hello-world` 以確認安裝是否成功。

每個 repository 都會在 CI 中以 bootstrap.sh 與 automake 工具鏈在本地端建置。

## 下載 CI 映像檔

首先，建立一個存放 CI repository 的位置，並先複製（clone）CI 腳本：

```shell
mkdir ci_test_area
cd ci_test_area
git clone https://github.com/openbmc/openbmc-build-scripts.git
```

## 新增唯讀 Repo

如果你只想驗證某個專案在上游的狀態、完全不加入自己的任何修改，可以直接把它複製（clone）到 Docker 目錄中。這個範例會直接複製並建置 phosphor-hwmon。上游 CI 會嘗試建置所有內容，其中也包含在有 `make check` 可用時執行它。此外，若目標 repository 中存在 `.clang-format` 檔案，或 repository 中有名為 `format-code.sh` 的腳本，CI 也會執行 `format-code.sh` 並檢查程式碼是否已正確格式化。

```shell
cd /path/to/ci_test_area
git clone https://github.com/openbmc/phosphor-hwmon
WORKSPACE=$(pwd) UNIT_TEST_PKG=phosphor-hwmon \
./openbmc-build-scripts/run-unit-test-docker.sh
```

注意：執行 'run-unit-test-docker.sh' 時，請確認沒有任何未提交的變更。該腳本會在執行 'format-code.sh' 之後執行 git diff，因此任何未提交的變更都會出現在輸出中，接著腳本便會認為程式碼格式化不正確而失敗。

注意：如果你只是偶爾對某個專案做一點快速修改，這個做法沒問題；但如果你想改用一個頻繁開發中的分支，就會比較難使用。若你計畫在某個 repo 上定期開發，請改用下一個做法。

## 對本地端已修改的 Repo 執行 CI

如果你 repo 中已有本地端變更、且暫時還不想提交，也可以透過在執行腳本前設定 `NO_FORMAT_CODE=1` 來執行本地端 CI。

例如：

```shell
WORKSPACE=$(pwd) UNIT_TEST_PKG=phosphor-hwmon NO_FORMAT_CODE=1 \
./openbmc-build-scripts/run-unit-test-docker.sh
```

`NO_FORMAT_CODE=1` 會告訴腳本跳過 `format-code.sh`，因此不會格式化你的程式碼，如此你的 repo 就可以包含未提交的變更。

## 僅為測試目的執行 CI

若只想呼叫測試套件（test suite）、而不想被其他可能會執行的各種分析步驟絆住，請設定 `TEST_ONLY=1`。

例如：

```shell
WORKSPACE=$(pwd) UNIT_TEST_PKG=phosphor-hwmon TEST_ONLY=1 \
./openbmc-build-scripts/run-unit-test-docker.sh
```

## 使用 `git worktree` 引用現有的 Repo

如果你已經在本機某份 repo 的複本上積極開發，或想要開始這麼做，你應該改用 `git worktree`，讓 Docker 容器可以引用你現有本機 repo 的特定分支。若有興趣，可以執行 `git help worktree` 進一步了解 worktree。

這個範例示範如何在你的 Docker 容器中建立 `phosphor-host-ipmid` 的 worktree。

```shell
cd /my/dir/for/phosphor-host-ipmid
git worktree add /path/to/ci_test_area/phosphor-host-ipmid
```

現在，如果你 `cd /path/to/ci_test_area`，應該會看到一個 `phosphor-host-ipmid/` 目錄；進入其中並執行 `git status`，你會發現自己很可能位於一個名為 `phosphor-host-ipmid` 的新分支上。這只是為了方便起見，因為你無法在 worktree 中簽出（check out）一個已在別處簽出的分支；之後你可以安全地忽略或刪除那個分支。

不過，Git 將無法找到如何存取你的主 worktree（`/my/dir/for/phosphor-host-ipmid`），所以我們需要在執行時掛載它。開啟 `/path/to/ci_test_area/openbmc-build-scripts/run-unit-test-docker.sh`，找到我們在最底部呼叫 `docker run` 的地方。新增一個額外的引數，記得要跳脫換行字元（'\')：

```bash
PHOSPHOR_IPMI_HOST_PATH="/my/dir/for/phosphor-host-ipmid"

docker run --blah-blah-existing-flags \
  -v ${PHOSPHOR_IPMI_HOST_PATH}:${PHOSPHOR_IPMI_HOST_PATH} \
  -other \
  -args
```

然後提交這個變更，如此當你更新腳本 repository 時，才能確保不會失去它：

```shell
cd openbmc-build-scripts
git add run-unit-test-docker.sh
git commit -m "mount phosphor-host-ipmid"
```

在這種情況下，最簡單的工作流程現在大致如下：

```shell
# Hack in the main worktree.
cd /my/dir/for/phosphor-host-ipmid
git checkout -b add-foo
# Make a change.
touch foo
# Make sure to commit it.
git add foo
git commit -sm "change foo"
# Update the Docker worktree to agree with it.
cd /path/to/ci_test_area/phosphor-host-ipmid
git checkout --detach add-foo
# Now run the Docker container normally
```

### 互動式 Docker Session

若要使用互動式 session，你可以傳入 `INTERACTIVE=true` 旗標給 `run-unit-test-docker.sh`，它會讓你進入一個帶有預設 D-Bus instance 的 bash shell。當你需要在工作站上對 core dump 執行 gdb，或執行某個 daemon 時，這會很方便。
