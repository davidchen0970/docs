# OpenBMC 速查表

本文件旨在提供一系列常用 OpenBMC 客製化任務的操作步驟，讓你在不需要了解完整
yocto 建置流程的情況下也能完成。

## 使用本機 kernel 建置

kernel recipe 位於：

```text
 meta-phosphor/common/recipes-kernel/linux/linux-obmc_X.Y.bb
```

若要使用本機的 git tree，請將 `SRC_URI` 改為不含 hostnames 的 git:// URL，並移除
`protocol=git` 參數。例如：

```bitbake
SRC_URI = "git:///home/jk/devel/linux;branch=${KBRANCH}"
```

`SRCREV` 變數可用於指定明確的 git commit，或設為 `"${AUTOREV}"` 以使用
`KBRANCH` 中最新的 commit。

## 為 Palmetto 建置

Palmetto 的目標為 `palmetto`。

```bash
cd openbmc
. setup palmetto
bitbake obmc-phosphor-image
```

## 為 Zaius 建置

Zaius 的目標為 `zaius`。

```bash
cd openbmc
. setup zaius
bitbake obmc-phosphor-image
```

## 建置特定的 machine 設定

如果你要建置的系統包含不同的 machine 設定：

```text
meta-<layer>/meta-<system>/conf/machine/machineA.conf
meta-<layer>/meta-<system>/conf/machine/machineB.conf
```

你可以將名稱傳給 `setup` 來指定你想建置的 machine 設定。

```bash
cd openbmc
. setup machineB
bitbake obmc-phosphor-image
```

## 建置 OpenBMC SDK

想找辦法在 'ARM' 上編譯你的程式，但你剛好跑在 'PPC' 或 'x86' 系統上？你可以
建置 sdk 並取得一個 fakeroot 環境。

```bash
bitbake -c populate_sdk obmc-phosphor-image
./tmp/deploy/sdk/openbmc-phosphor-glibc-x86_64-obmc-phosphor-image-armv5e-toolchain-2.1.sh
```

依照提示操作。安裝完成後，設定環境的預設方式大致類似這條指令

```bash
. /opt/openbmc-phosphor/2.1/environment-setup-armv5e-openbmc-linux-gnueabi
```

## 重新建置與重新設定

你可以移除 build/conf 目錄來重新設定你的建置：

```bash
rm -rf build/conf
```

然後再次執行 `setup`。

## 實用的 D-Bus CLI 工具

## `busctl`

<http://www.freedesktop.org/software/systemd/man/busctl.html>

一個很棒的工具，可透過 cli 發出 D-Bus 指令。這樣你就不必等待程式碼在系統上
走到該路徑。也很適合搭配 QEMU 執行指令！

執行方式：

```bash
busctl call <path> <interface> <object> <method> <parameters>
```

- \<parameters\> 範例：sssay "t1" "t2" "t3" 2 2 3

## 使用 QEMU

QEMU 有一個 palmetto-bmc machine（自 v2.6.0 起）實化了開機 Linux kernel 所需的
核心裝置。OpenBMC 也
[維護了一個 tree](https://github.com/openbmc/qemu)，其中的補丁正送往 upstream，
或是暫時的替代方案，在適當的地方擴充 QEMU 的功能。

```bash
qemu-system-arm -m 256 -M palmetto-bmc -nographic \
    -drive file=<path>/flash-palmetto,format=raw,if=mtd \
    -net nic \
    -net user,hostfwd=:127.0.0.1:2222-:22,hostfwd=:127.0.0.1:2443-:443,hostname=qemu
```

如果遇到錯誤，你很可能需要先建置 QEMU（見本文件中的對應章節）。若沒有錯誤且
QEMU 順利啟動，只需在與 BMC 互動時更改連接埠即可⋯⋯

```bash
curl -k -H "Content-Type: application/json" -X POST -D headers.txt \
    https://${bmc}/redfish/v1/SessionService/Sessions -d \
    '{"UserName":"root", "Password":"0penBmc"}'
```

或

```bash
ssh -p 2222 root@localhost
```

若要結束，輸入 `Ctrl-a c` 切換到 QEMU monitor，然後輸入 `quit` 退出。

## 建置 QEMU

```bash
git clone https://github.com/openbmc/qemu.git
cd qemu
git submodule update --init dtc
mkdir build
cd build
../configure --target-list=arm-softmmu
make
```

建置產出的檔案會位於：`arm-softmmu/qemu-system-arm`

### 使用 bridge 裝置

使用 bridge 裝置需要一些 root 權限來設定。好處是你的 qemu session 執行在
bridge 的子網路中，因此不需要連接埠轉發。需要安裝一些套件才能取得 virbr0，
例如⋯⋯

```bash
apt-get install libvirt libvirt-bin bridge-utils uml-utilities qemu-system-common

qemu-system-arm -m 256 -M palmetto-bmc -nographic \
    -drive file=<path>/flash-palmetto,format=raw,if=mtd \
    -net nic,macaddr=C0:FF:EE:00:00:02,model=ftgmac100  \
    -net bridge,id=net0,helper=/usr/lib/qemu-bridge-helper,br=virbr0
```

還有一些其他實用的參數，可以將 console 重新導向到另一個視窗。這樣就能輕鬆
存取 qemu 指令 session。
`-monitor stdio -serial pty -nodefaults`

## 啟動主機

登入：

```bash
curl --insecure -H "Content-Type: application/json" -X POST -D headers.txt \
    https://${bmc}/redfish/v1/SessionService/Sessions -d  \
    '{"UserName":"root", "Password":"0penBmc"}'
```

會建立一個 headers.txt 檔案。在該檔案中找到 "X-Auth-Token"。

```bash
export bmc_token=<token>
```

連線到主機 console：

```bash
ssh -p 2200 root@bmc
```

開機：

```bash
curl -k -H "X-Auth-Token: $token" -H "Content-Type: application/json" -X POST \
    https://${bmc}/redfish/v1/Systems/system/Actions/ComputerSystem.Reset \
    -d '{"ResetType": "On"}'
```

## GDB

[SDK 建置](#building-the-openbmc-sdk) 提供 GDB 與除錯符號：

- 設定好 SDK 環境後即可使用 `$GDB`
- 除錯符號位於每個可執行檔的 `.debug/` 目錄中

使用 GDB 的方式：

1. 設定 SDK 環境；
2. 執行以下 GDB 指令：

```bash
cd <sysroot_of_sdk_build>
$GDB <relative_path_to_executable> <path_to_core_file>
```

## Coredump

OpenBMC 預設停用 coredump。要啟用 coredump：

```bash
echo '/tmp/core_%e.%p' | tee /proc/sys/kernel/core_pattern
ulimit -c unlimited
```

## 清理讀寫檔案系統的變更

你可能想查明哪些檔案會透過 overlay rwfs 持續保留。為此，你可以列出此路徑，
然後移除那些你希望恢復原狀的檔案，或移除刪除 overlay 以還原檔案。

```bash
/run/initramfs/rw/cow/
```

## 建置

### 尋找建置記錄

當某個 recipe 在編譯或安裝階段失敗時，這些 task 記錄通常是找到真正錯誤最快的
方式，不必捲動檢視完整的 BitBake console 輸出。

- 編譯記錄：`build/tmp/work/<arch>/<recipe>/<version>/temp/log.do_compile`
- 安裝記錄：`build/tmp/work/<arch>/<recipe>/<version>/temp/log.do_install`

### 共用 downloads 目錄

OpenBMC 第一次建置要花很長的時間。它會從網路上下載各種 repo。

檢視 `build/downloads` 即可看到所有已下載的 repo。

- 如果某個 repo 是單一壓縮檔，通常看起來像這樣：
  - `zlib-1.2.11.tar.xz` - repo 本身
  - `zlib-1.2.11.tar.xz.done` - 表示該 repo 已下載的旗標
- 如果某個 repo 由 git 管理，通常看起來像這樣：
  - `git2/github.com.openbmc.linux` - git 的 bare clone
  - `git2/github.com.openbmc.linux.done` - 表示該 repo 已下載的旗標

Bitbake 會在建置期間將程式碼解壓到工作目錄，因此同一個系統上不同的建置可以
共用 `downloads` 目錄：

- 編輯 `build/conf/local.conf` 檔案，將 `DL_DIR` Bitbake 環境變數設為你的共用
  downloads 目錄的位置：

```bitbake
DL_DIR ?= "<path>/<to>/<existing>/downloads"
```

- 或建立一個符號連結：

```bash
ln -sf <path>/<to>/<existing>/downloads build/downloads
```

然後進行建置。這會省下大量下載程式碼的時間。

## 使用 git proxy

如果你在下拉取程式碼時遇到極慢的下載速度（例如你在中國），可以使用 git proxy
來加速程式碼拉取。

搜尋 `git-proxy-wrapper` 會找到各種為 git 協定設定 proxy 的方式。

以下是一個位於 `~/bin` 的 wrapper 範例，假設在 port 9054 有一個 socks5 proxy：

```bash
#!/bin/sh
## Use connect-proxy as git proxy wrapper which supports SOCKS5
## Install with `apt-get install connect-proxy`
## Use with `export GIT_PROXY_COMMAND=~/bin/git-proxy-wrapper`
/usr/bin/connect -S localhost:9054 "$@"
```

接著你可以執行 `export GIT_PROXY_COMMAND=~/bin/git-proxy-wrapper`，現在你就能
透過 proxy 下載 git 程式碼。

## devtool

`devtool` 是 Yocto 中一個方便的公用程式，可在本機目錄中進行修改。典型的用法是：

```bash
# To create a local copy of recipe's code and build with it:
devtool modify <recipe>
cd build/workspace/sources/<recipe>  # And make changes
bitbake obmc-phosphor-image  # Build with local changes

# After you have finished, reset the recipe to ignore local changes:
devtool reset <recipe>
```

要使用這個工具，你需要建置環境，例如 `. oe-init-build-env`。上面的腳本會將
`<WORKDIR>/scripts/` 加入你的 `PATH` 環境變數，而 `devtool` 就在該路徑中。

以下為實際範例。

### 在 ipmi 上使用 devtool

如果你想除錯或在 ipmi 中新增一個新功能，你可能需要修改
[phosphor-host-ipmid][ipmid-repo] 中的程式碼。檢視 recipes 後，你會知道這個 repo
位於 [phosphor-ipmi-host.bb][ipmid-recipe]。以下是使用 devtool 在本機修改程式碼、
建置並測試它的步驟。

1. 使用 devtool 建立一個本機 repo：

   ```bash
   devtool modify phosphor-ipmi-host
   ```

   devtool 會將 repo 複製到 `build/workspace/sources/phosphor-ipmi-host`，
   建立並簽出 `devtool` 分支。

2. 在 repo 中進行修改，例如新增處理新 ipmi 指令的程式碼，或只是加入追蹤記錄。

3. 現在你可以建置整個映像或 ipmi recipe 本身：

   ```bash
   bitbake obmc-phosphor-image  # Build the whole image
   bitbake phosphor-ipmi-host  # Build the recipe
   ```

4. 要測試你的變更，可以燒錄整個映像或取代已變更的 binary。請注意，變更的
   程式碼會建置進 `libapphandler.so`，而它同時被 host 與 net ipmi daemon 使用。
   建議你將已變更的 binary 複製到 BMC，因為這樣更容易測試：

   ```bash
   # Replace libapphandler.so.0.0.0
   scp build/workspace/sources/phosphor-ipmi-host/oe-workdir/package/usr/lib/ipmid-providers/libapphandler.so.0.0.0 root@bmc:/usr/lib/ipmid-providers/
   systemctl restart phosphor-ipmi-host.service  # Restart the inband ipmi daemon
   # Or restart phosphor-ipmi-net.service if you want to test net ipmi.
   ```

5. 現在你可以測試你的變更了。

[ipmid-repo]: https://github.com/openbmc/phosphor-host-ipmid
[ipmid-recipe]:
  https://github.com/openbmc/openbmc/blob/c53f375a0f92f847d2aa50e19de54840e8472c8e/meta-phosphor/recipes-phosphor/ipmi/phosphor-ipmi-host_git.bb

## 開發 linux kernel

### 在 linux kernel 上使用 devtool

如果你想處理 linux kernel，也可以使用 devtool，只是一些地方與一般的 repo 不同。

**注意**：自 [ac72846][kernel-recipe-commit] 起，對於以 Aspeed 為基礎的 OpenBMC
建置，linux kernel recipe 的名稱已改為 `linux-aspeed`。在以下範例中，如果你用的
版本晚於 [ac72846][kernel-recipe-commit]，請將 `linux-obmc` 替換為 `linux-aspeed`。

1. devtool 不會建立 'devtool' 分支。相反地，它會簽出 recipe 中指定的分支。
   例如，在 OpenBMC v2.2 tag 上，`linux-obmc_4.13.bb` 指定 `dev-4.13` 分支。
2. 如果有補丁，`devtool` 會直接將它們套用到該分支上。
3. devtool 會將 defconfig 與 machine 專屬的設定複製到 `oe-workdir`。
4. devtool 會根據上述設定產生 `.config` 檔案。

你可以像往常一樣修改程式碼並建置 kernel，方式如下：

```bash
bitbake linux-obmc -c build
```

### 修改 config

如果你需要修改 config 並將其儲存為 defconfig 以供後續使用：

```bash
bitbake linux-obmc -c menuconfig
# Edit the configs and after save it generates
# .config.new as the new kernel config

bitbake linux-obmc -c savedefconfig
# It will save the new defconfig at oe-workdir/linux-obmc-<version>/defconfig
```

### 測試 linux kernel

建置完成後，你可以燒錄映像來測試新的 kernel。不過把映像燒錄到晶片上總是
很慢。

有一種更快的方式透過網路載入 kernel，讓你輕鬆測試 kernel 建置。

OpenBMC kernel 建置會產生 `fit` 映像，包含 `kernel`、`dtb` 與 `initramfs`。
通常我們可以透過 tftp 載入它，以 Romulus 為例：

1. 將
   `build/tmp/deploy/images/romulus/fitImage-obmc-phosphor-initramfs-romulus.bin`
   放到一個 tftp server 上，命名為 `fitImage`
2. 重新啟動 BMC 並按鍵進入 uboot shell；
3. 在 uboot 中：

```bash
setenv ethaddr <mac:addr>  # Set mac address if there it is unavailable
setenv ipaddr 192.168.0.80  # Set BMC IP
setenv serverip 192.168.0.11  # Set tftp server IP
tftp 0x83000000 fitImage  # Load fit image to ram. Use 0x43000000 on AST2400
bootm 0x83000000  # Boot from fit image
```

如此你便執行了一個帶有更新後 kernel 的 OpenBMC。

[kernel-recipe-commit]:
  https://github.com/openbmc/openbmc/commit/ac7284629ea572cf27d69949dc4014b3b226f14f

## 取得 Yocto 變數的值

要取得某個特定 recipe 的 Yocto 變數值，請使用：

```bash
bitbake-getvar <VARIABLE> -r <recipe>
```

例如，要取得 obmc-phosphor-image 的 MACHINE 變數值：

```bash
bitbake-getvar MACHINE -r obmc-phosphor-image
```

這對於除錯很有用，因為 Yocto 變數可能在多個 layer 與設定檔案中被指派或修改。
`bitbake-getvar` 有助於取得經過所有 layer 指派後最終解析出的值，準確顯示建置
期間將會使用的值。
