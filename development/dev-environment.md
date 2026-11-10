# OpenBMC 開發環境

**文件目的：** 如何設定一個 OpenBMC 開發環境

**目標讀者：** 熟悉 Linux 與 BMC 的程式設計師

**前置需求：** 現行版本的 Linux、Mac 或 Windows 系統

## 總覽

OpenBMC 採用 [Yocto](https://www.yoctoproject.org/) 專案作為其底層的建置與發行版產生框架。OpenBMC 主 [README](https://github.com/openbmc/openbmc/blob/master/README.md) 提供了關於如何開始使用 Yocto 與 OpenBMC 的資訊。本教學將帶領你了解如何使用 bitbake 建置 OpenBMC 韌體，並在 [QEMU](https://www.qemu.org/) 中啟動它。

Bitbake 是 Yocto 與 OpenBMC 用來為系統建置其自訂 Linux 發行版的建置引擎。QEMU 是一款軟體模擬器，可用來執行 OpenBMC 映像檔。

本文件逐步說明設定 OpenBMC 開發環境、建置 OpenBMC，以及在 QEMU 中執行該映像檔的建議步驟。

出於測試考量，本指南以 Romulus 系統作為預設，因為這是每個 CI 作業都會測試的系統，也就代表它是最穩定的。

## 安裝 Linux 環境

如果你使用的是 Linux，並且可以接受安裝一些額外的套件，那麼你可以直接跳到步驟 3。

建議的 OpenBMC 開發環境是最新的 Ubuntu LTS 發行版。其他版本的 Linux 也可能可用，但使用它們必須自負風險。如果你使用 Windows 或 Mac OS，則建議使用 VirtualBox 作為執行開發環境的虛擬化工具。

1. 安裝一個虛擬機器

   在你的電腦（Mac、Windows、Linux）上安裝 [VirtualBox](https://www.virtualbox.org/wiki/Downloads) 或 [VMware](https://www.vmware.com/products/workstation-player/workstation-player-evaluation.html)

   兩者都有可滿足你需求的免費版本。**注意：** 如果你想用這個 VM 透過 BitBake 建置完整的 OpenBMC 映像檔，你會希望盡量分配最多的資源。理想的最低資源為 8 執行緒、16GB 記憶體、200GB 硬碟。

   OpenBMC 的持續整合使用 docker 來建置其映像檔。這是另一種可行的做法，但不在本文件的涵蓋範圍內。請參考 [build-setup.sh](https://github.com/openbmc/openbmc-build-scripts/blob/master/build-setup.sh)，其中說明了 CI 用來建置合適 docker 容器並在其中啟動 bitbake 的腳本。這在 podman 環境中同樣適用。

2. 安裝最新的 Ubuntu LTS 發行版

   OpenBMC 開發社群的多數成員使用 Ubuntu。下述的 qemu 是在最新的 Ubuntu LTS 發行版上建置的，但任何最新版本_應該_都能運作。Fedora 等其他 Linux 發行版也類似，但同樣地，相對於 Ubuntu，OpenBMC 核心團隊對這些發行版的測試遠遠少了許多。

   **VirtualBox 提示** - 你會希望 VM 與主機之間的複製／貼上能夠運作。為此，一旦你的 VM 完成啟動並執行後：
   - Devices -> Insert Guest Additions CD Image（安裝）
   - Devices -> Shared Clipboard -> Bidirectional
   - 重新啟動（該 VM）

3. 安裝必要套件

   請參考 [Prerequisite](https://github.com/openbmc/openbmc/blob/master/README.md#1-prerequisite) 連結。

   **注意** - 在 Ubuntu 中，安裝套件之前大概需要先執行 "sudo apt-get update"。

   **注意** - Ubuntu 24.04 預設會透過 AppArmor 限制非特權使用者命名空間，這會導致 BitBake workers 在寫入 `/proc/self/uid_map` 時因 `PermissionError: [Errno 1] Operation not permitted` 而失敗。請在建置前執行以下指令來修正這個問題：

   ```bash
   echo 'kernel.apparmor_restrict_unprivileged_userns = 0' | \
       sudo tee /etc/sysctl.d/99-bitbake.conf
   sudo sysctl -p /etc/sysctl.d/99-bitbake.conf
   ```

## 建置 OpenBMC

請注意本節將帶領你完成建置 Romulus OpenBMC 映像檔的流程。未來的教學將在此基礎上進一步讓你自訂映像檔。如果你想跳過建置、只想試用 OpenBMC 與 QEMU，那麼你可以從 [OpenBMC Jenkins](https://jenkins.openbmc.org/job/latest-master/label=docker-builder,target=romulus/lastSuccessfulBuild/artifact/openbmc/build/tmp/deploy/images/romulus/obmc-phosphor-image-romulus.static.mtd) 下載最新的 Romulus 映像檔，並直接跳到[下載並啟動 QEMU 工作階段](#download-and-start-qemu-session)一節。

1. 複製 OpenBMC

   ```bash
   git clone https://github.com/openbmc/openbmc.git
   ```

2. 建置 Romulus OpenBMC 映像檔（注意，依你的硬體不同，這將耗時 30 至 120 分鐘）

   ```bash
   . setup romulus
   bitbake obmc-phosphor-image
   ```

Romulus 映像檔現在位於相對於你目前目錄的 `build/tmp/deploy/images/romulus/obmc-phosphor-image-romulus.static.mtd`。

## 下載並啟動 QEMU 工作階段

1. 下載 openbmc/qemu 最新分支的 QEMU 應用程式

   ```bash
   wget https://jenkins.openbmc.org/job/latest-qemu-x86/lastSuccessfulBuild/artifact/qemu/build/qemu-system-arm

   chmod u+x qemu-system-arm
   ```

2. 將建置產生的映像檔複製到你的目前目錄

   ```bash
   cp ./tmp/deploy/images/romulus/obmc-phosphor-image-romulus.static.mtd ./
   ```

3. 使用 Romulus 映像檔啟動 QEMU

   **注意** - 要讓 REST、SSH 與 IPMI 能夠連入你的 QEMU 工作階段，你必須將某些主機連接埠對應到 QEMU 工作階段中的 REST、SSH 與 IPMI 連接埠。在本例中僅使用 2222、2443、2623。你可以使用任何你偏好的連接埠。

   ```bash
   ./qemu-system-arm -m 256 -M romulus-bmc -nographic \
      -drive file=./obmc-phosphor-image-romulus.static.mtd,format=raw,if=mtd \
      -net nic \
      -net user,hostfwd=:127.0.0.1:2222-:22,hostfwd=:127.0.0.1:2443-:443,hostfwd=udp:127.0.0.1:2623-:623,hostname=qemu
   ```

   如果你是在一個可以使用真實連接埠的虛擬環境中執行，那麼你會以下列方式啟動 QEMU。

   ```bash

   ./qemu-system-arm -m 256 -machine romulus-bmc -nographic \
    -drive file=./obmc-phosphor-image-romulus.static.mtd,format=raw,if=mtd \
    -net nic \
    -net
   user,hostfwd=:127.0.0.1:22-:22,hostfwd=:127.0.0.1:443-:443,hostfwd=tcp:127.0.0.1:80-:80,hostfwd=tcp:127.0.0.1:2200-:2200,hostfwd=udp:127.0.0.1:623-:623,hostfwd=udp:127.0.0.1:664-:664,hostname=qemu

   ```

4. 等待你基於 QEMU 的 BMC 完成啟動

   使用預設的 root/0penBmc 帳號登入（注意其中的 0 是數字零）。

5. 檢查系統狀態

   你會在主控台看到大量服務正在啟動，你可以開始執行 obmcutil 工具來檢查 OpenBMC 狀態服務的狀況。當你看到下列內容時，就代表你已成功啟動至 "Ready" 狀態。

   ```bash
   root@openbmc:~# obmcutil state
   CurrentBMCState     : xyz.openbmc_project.State.BMC.BMCState.Ready
   CurrentPowerState   : xyz.openbmc_project.State.Chassis.PowerState.Off
   CurrentHostState    : xyz.openbmc_project.State.Host.HostState.Off
   ```

   **注意** 要退出（並終止）你的 QEMU 工作階段，請執行：`ctrl+a x`

6. 測試 ssh 網路介面

   在你啟動 QEMU 的那個系統上執行這些指令

   ```bash
   ssh root@localhost -p 2222
   ```

   登入方式與上述預設 QEMU 登入所用的相同。

你現在已經建置好一個 OpenBMC 發行版並在 QEMU 中啟動了它！

## 替代方案：yocto QEMU

yocto 提供了用於建置與執行 qemu 的工具。這些工具可以避免因下載預先建置好的映像檔及修改二元檔而產生的一些設定問題。使用 yocto qemu 也會採用 [TAP 介面](https://www.kernel.org/doc/Documentation/networking/tuntap.txt)，有些人認為它更為穩定。這在應用層級進行除錯時特別有用。

這是較為進階的一節，或許在你讓基本功能運作之後再回頭看會更有幫助。下一篇教學將聚焦於 devtool，以及如何運用它來自訂你的 OpenBMC 映像檔。

- 設定一個 bmc 建置環境

```bash
source setup romulus myBuild/build
```

- 加入用於測試的 qemu x86 open embedded machine

```bash
MACHINE ??= "qemux86"
```

- 對建置進行修改（例如 devtool modify bmcweb、devtool add gdb）

```bash
devtool modify bmcweb myNewLocalbmcweb/
```

- 為 qemu x86 machine 建置 open bmc

```bash
MACHINE=qemux86 bitbake obmc-phosphor-image
```

- 以 yocto 提供的方式執行 qemu

```bash
runqemu myBuild/build/tmp/deploy/images/qemux86/ nographic \
    qemuparams="-m 2048"
```

- 之後會新增一個 TAP 網路介面，且 ssh、scp、http 等協議都能順利運作。

## 共用下載與建置快取

BitBake 會同時快取抓取的來源碼（DL_DIR）與可重複使用的建置輸出（SSTATE_DIR）。讓多個建置指向共用的目錄，可以避免重新下載與重新建置，節省時間與磁碟空間。

1. 一次建立共用的快取目錄：

   ```bash
   mkdir -p "${XDG_CACHE_HOME}/bitbake/downloads" "${XDG_CACHE_HOME}/bitbake/sstate"
   ```

2. 加入至 build/conf/local.conf：

   ```bash
   DL_DIR = "${XDG_CACHE_HOME}/bitbake/downloads"
   SSTATE_DIR = "${XDG_CACHE_HOME}/bitbake/sstate"
   BB_HASHSERVE_DB_DIR = "${SSTATE_DIR}"
   ```

這樣就能讓每個建置重複使用先前已下載與已建置的內容，因此你能使用較少的磁碟空間並更快完成。
