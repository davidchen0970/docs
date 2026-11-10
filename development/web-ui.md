# OpenBMC Web 使用者介面開發

**文件目的：** 如何客製化、建置與執行 OpenBMC Web UI

**目標讀者：** 熟悉 HTML、CSS 與 JS 的開發者

**前置條件：** 目前的 Linux、Mac 或 Windows 系統

## Webui-vue

[webui-vue](https://github.com/openbmc/webui-vue) 儲存庫將在 phosphor-webui
被淘汰後取代它。Webui-vue 使用 [Vue.js](https://vuejs.org/) 框架，透過 Redfish
API 與 BMC 互動。

請造訪
[README.md](https://github.com/openbmc/webui-vue/blob/master/README.md)，進一步了解
為何要建立這個 Vue.js 應用程式、要達到功能對等所需的功能，以及為何它要取代
Angular.JS 應用程式。

請造訪
[CONTRIBUTING.md](https://github.com/openbmc/webui-vue/blob/master/CONTRIBUTING.md)，
尋找專案設定、設計資訊與貢獻流程的相關資訊。

請造訪 [OpenBMC Web UI 樣式指南](https://openbmc.github.io/webui-vue/)，尋找以下資訊：

- 程式碼規範
- 指導方針
- 單元測試
- 元件使用方式
- 快速上手參考

請造訪
[OpenBMC Web UI 佈景主題指南 - 如何客製化](https://openbmc.github.io/webui-vue/customization/theme.html)，
了解如何建立自訂建置，以滿足你在以下方面的品牌與客製化需求：

- 路由
- 導覽
- 狀態儲存
- 佈景主題

### 針對 QEMU 載入 Web UI

連線到 QEMU 中的 Web UI

1. 你需要按照
   [dev-environment](https://github.com/openbmc/docs/blob/master/development/dev-environment.md)
   中「Download and Start QEMU Session」一節的說明，執行一個 QEMU 工作階段。

2. 假設你在 QEMU 工作階段中對 HTTPS 埠使用了預設值 2443，則請將瀏覽器指向
   <https://localhost:2443>。

3. 以預設的使用者名稱與密碼登入，並確認基本的 Web UI 功能如預期般運作。

**注意** 你需要在瀏覽器中核准安全性例外狀況才能連線。OpenBMC 是以自簽憑證
（self-signed SSL certificate）執行。

## Phosphor-webui

[phosphor-webui](https://github.com/openbmc/phosphor-webui) 儲存庫為 OpenBMC
提供了網頁式的介面。它使用 [AngularJS](https://angularjs.org/) 框架，透過 REST
API 呼叫與 BMC 互動。它讓使用者能夠檢視硬體資訊、更新韌體、設定網路設定等等。

請參考下方的說明，了解
[如何客製化 phosphor-webui](#customize-phosphor-webui)。

Phosphor-webui 是以 AngularJS 建置，而 AngularJS 將於 2021 年 6 月 30 日終止支援
（End of Life）。

### 客製化 Phosphor-webui

1. 複製儲存庫

   ```bash
   git clone https://github.com/openbmc/phosphor-webui.git
   ```

2. 安裝適當的套件並啟動 Web UI

   請遵循 phosphor-webui
   [README](https://github.com/openbmc/phosphor-webui/blob/master/README.md)
   中的說明，安裝所需的套件並啟動 Web UI。你可以使用在
   [dev-environment](https://github.com/openbmc/docs/blob/master/development/dev-environment.md)
   中建立開發環境，或使用了你自己的系統——前提是你要安裝 README 中提到的所需套件。

3. 客製化 Web UI 登入畫面並驗證

   使用 Ctrl^C 終止上一步驟的 npm run。取得一個 png，你將用它來代表你客製化
   版本的 OpenBMC。這一步驟你可以隨意使用任何你想要的 .png。

   ```bash
   wget http://www.pngmart.com/files/3/Free-PNG-Transparent-Image.png
   ```

   將你的新 .png 複製到適當的目錄

   ```bash
   cp Free-PNG-Transparent-Image.png app/assets/images/
   ```

   在 Web UI 的 HTML 中指向該新圖片

   ```bash
   vi app/login/controllers/login-controller.html
   # Replace the logo.svg near the top with Free-PNG-Transparent-Image.png
   <img src="../../assets/images/Free-PNG-Transparent-Image.png" class="login__logo" alt="OpenBMC logo" role="img"/>
   ```

   帶著你的變更啟動伺服器

   ```bash
   npm run-script server
   ```

   在瀏覽器中載入 <https://localhost:8080>，並確認登入畫面上有你的新圖片。

   使用 Ctrl^C 終止你的 npm run。

4. 以新圖片客製化頁首（header）並驗證。頁首出現在 Web UI 的每個頁面上。它含有
   一個較小尺寸的標誌，我們在這一步驟中要將它換掉。

   與上一步驟類似，修改頁首對應的 HTML：

   ```bash
   vi app/common/directives/app-header.html
   # Replace logo.svg with Free-PNG-Transparent-Image.png again
   <div class="logo__wrapper"><img src="../../assets/images/Free-PNG-Transparent-Image.png" class="header__logo" alt="company logo"/></div>
   ```

   帶著你的變更啟動伺服器

   ```bash
   npm run-script server
   ```

   瀏覽至 `https://localhost:8080`，並確認頁首上有你的新圖片。

   請注意，你需要登入才能看到頁首。在「BMC HOST OR BMC IP ADDRESS」欄位中輸入
   你的 QEMU 工作階段（例如 localhost:2443），將 Web UI 指向你的 QEMU 工作階段。

   請注意，在你替換圖片的 HTML 中，還有與圖片相對應的文字。將文字改成與你的
   標誌相符，也是你能輕鬆做到、用以客製化你所建立的 OpenBMC 系統的一件事。

就這樣！你已經下載、客製化並執行了 OpenBMC phosphor-webui 程式碼！
