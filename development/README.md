# OpenBMC 開發者文件

這個目錄專注於為 OpenBMC 開發者提供開始進行 OpenBMC 開發所需的說明。
你可以依任何順序閱讀，但建議的流程如下。

1. [開發環境設定](dev-environment.md)

   從這裡開始。這份文件示範如何使用 bitbake 建置流程設定 OpenBMC 開發環境，
   以及如何啟動軟體模擬器 QEMU。

2. [Hello World](devtool-hello-world.md)

   這份文件示範如何使用 yocto 工具 devtool 來擷取 OpenBMC 原始碼儲存庫、
   建置，並在 QEMU 中測試你的變更。

3. [Web UI 開發](web-ui.md)

   這份文件示範如何修改 phosphor-webui 網頁應用程式，並在 QEMU 中測試你的
   變更。

4. [使用 Gerrit 進行程式碼審查](gerrit-setup.md)

   這份文件示範如何設定你的環境，以便利用 Gerrit 提交你的變更進行程式碼審查。
