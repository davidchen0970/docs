# 使用 devtool 開發 OpenBMC Hello World

**文件目的：** 帶領你完成在 QEMU 中編譯並執行一個 OpenBMC 應用程式。

**前置條件：** 已完成開發環境設定
[文件](https://github.com/openbmc/docs/blob/master/development/dev-environment.md)

## Clone 並 Build 一個 Repo

devtool 是 yocto 發行版所提供的一款命令列工具，讓你可以將目標 repo
的原始碼萃取到本地的 bitbake 環境中並加以修改。

今天我們要使用 devtool 來萃取
[phosphor-state-manager](https://github.com/openbmc/phosphor-state-manager.git)
repo 並對它進行修改。

這裡假設你已經依照第一篇教學完成操作，並且正處於 bitbake 環境中，也就是
才剛執行完 ". setup" 命令的狀態。

1. 使用 devtool 萃取原始碼

   ```bash
   devtool modify phosphor-state-manager
   ```

   上述命令會輸出存放原始碼的目錄。

2. 修改一個原始碼檔案並加入一個 cout

   ```bash
   vi workspace/sources/phosphor-state-manager/bmc_state_manager_main.cpp
   ```

   你的 diff 看起來應該類似這樣：

   ```bash
   +#include <iostream>

   int main(int argc, char**)
   {
   @@ -17,6 +18,8 @@ int main(int argc, char**)

       bus.request_name(BMC_BUSNAME);

   +    std::cout<<"Hello World" <<std::endl;
   +
       while (true)
       {
   ```

3. 重新 build 開機映像檔，它現在會包含你的修改

   這次的 build 會快很多，因為 bitbake 會充分利用你前一次 build
   的快取，只編譯新增的部分。

   ```bash
   bitbake obmc-phosphor-image
   ```

   依照第一篇教學的步驟，把你的新映像檔載入到 QEMU session
   並開機。

4. 確認你的 "Hello World" 已進到新映像檔中

   登入你的 QEMU session 後，確認 journal 中有這則訊息

   ```bash
   journalctl | grep "Hello World"
   ```

   你應該會看到類似這樣的內容：

   ```bash
   <date> romulus phosphor-bmc-state-manager[1089]: Hello World
   ```

你完成了一項修改，重新 build 出包含該修改的開機映像檔，接著把該映像檔開機並確認修改確實生效了！

## 直接把應用程式載入到執行中的 QEMU

在這一節中，我們會修改同樣的原始碼檔案，但這次不再完整地重新產生開機映像檔
並重新啟動 QEMU，而是只 build 出所需的 binary，把它複製到正在執行的 QEMU
session 中並啟動它。

1. 修改你的 hello world

   ```bash
   vi workspace/sources/phosphor-state-manager/bmc_state_manager_main.cpp
   ```

   把你的 cout 改成 "Hello World Again"

2. 只 Bitbake phosphor-state-manager 這個 repo

   在 bitbake 中，你可以只 build 你有興趣的那個 repo（以及它的相依套件）。
   在這裡，我們只重新 build phosphor-state-manager repo，以取得你新的 hello
   world 修改。

   ```bash
   bitbake phosphor-state-manager
   ```

   你的新 binary 會位於相對於你 bitbake 目錄的以下位置：
   `./workspace/sources/phosphor-state-manager/oe-workdir/package/usr/bin/phosphor-bmc-state-manager`

3. 為你的應用程式建立一個安全的檔案系統

   現在是把你的 Hello World 應用程式載入到 QEMU
   虛擬硬體的時候了。OpenBMC 會覆寫 PATH
   變數，讓它一律優先尋找 /usr/local/bin/，所以你可以乾脆把應用程式 scp
   進到 /usr/local/bin，執行它就結束了。對於命令列測試這樣做沒問題，但當你
   開始透過 systemd service 啟動應用程式時就會遇到問題，因為應用程式路徑是
   硬編碼在 service 檔案裡的。讓我們從專業人士的做法開始，建立一個 overlay
   檔案系統。Overlay 能幫你省下時間與麻煩：不必再重新命名並還原原始檔案，
   不必再忘記你在除錯時動過哪些檔案，當然，也不必再把系統搞掛。登入 QEMU
   實例並執行這些命令。

   ```bash
   mkdir -p /tmp/persist/usr
   mkdir -p /tmp/persist/work/usr
   mount -t overlay -o lowerdir=/usr,upperdir=/tmp/persist/usr,workdir=/tmp/persist/work/usr overlay /usr
   ```

4. 用 scp 把這個 binary 傳到你的 QEMU 實例上

   如果你啟動 QEMU 時使用的是預設連接埠，那麼以下是從你的
   phosphor-state-manager 目錄執行的 scp 命令。如果你選擇了自己的連接埠，
   就在這裡把 2222 換掉。

   ```bash
   scp -P 2222 ./workspace/sources/phosphor-state-manager/oe-workdir/package/usr/bin/phosphor-bmc-state-manager root@127.0.0.1:/usr/bin/
   ```

## 在 QEMU 中執行應用程式

1. 停止 BMC state manager service

   ```bash
   systemctl stop xyz.openbmc_project.State.BMC.service
   ```

2. 在你的 QEMU session 中執行應用程式：

   ```bash
   phosphor-bmc-state-manager
   ```

   你會看到你的 "Hello World Again" 訊息顯示出來。按 Ctrl^C
   結束該應用程式。一般來說，這並不是你測試新應用程式的方式。實際上你會使用
   systemd service。

3. 透過 systemd service 啟動應用程式

   OpenBMC 使用 systemd 來管理它的應用程式。之後會有相關的教學，但目前先執行
   以下命令來重新啟動 BMC state service，讓它載入你的新應用程式：

   ```bash
   systemctl restart xyz.openbmc_project.State.BMC.service
   ```

   由於是 systemd 啟動你的 service，"Hello World Again"
   不會輸出到控制台，但它會出現在 journal 中。後續的教學會討論 journal，但目前
   先執行：

   ```bash
   journalctl | tail
   ```

   你應該會在某一筆 journal 條目中看到類似這樣的內容：

   ```bash
   <date> romulus phosphor-bmc-state-manager[1089]: Hello World Again
   ```

完成了！你客製化了一個現有的 BMC 應用程式，使用 devtool 建置它，並且透過
重新產生整個開機映像檔以及用 scp 打補丁這兩種方式，都在 QEMU
中執行了它！

## 進階工作流程

當你已經熟練掌握上面的內容後，還有一些更進階的工具與包裝腳本可以利用，以最佳化
這個流程。請參閱
[工作流程文件](https://amboar.github.io/notes/2022/01/13/openbmc-development-workflow.html)
以取得更多資訊。
