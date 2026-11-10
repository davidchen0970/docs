# OpenBMC 中的 Yocto

Yocto Project 是一個開放原始碼的協作專案，提供範本、工具與方法，協助你
為嵌入式產品建立客製化的 Linux 系統，不論其硬體架構為何。

OpenBMC 使用 Yocto 工具來管理 BMC 映像檔的配置與建立。

## 使用 Yocto 開發

Yocto 在 OpenBMC 中有兩種主要的使用情境：

1. 從 master 或既有標籤（tags）建置
2. 開發變更以提交至 master

第一種是較簡單的情況，主要是在呼叫 `bitbake` 之前先選擇要建置的系統
設定。關於 [Palmetto](cheatsheet.md#building-for-palmetto) 和
[Zaius](cheatsheet.md#building-for-zaius) 的範例請參閱
[cheatsheet](cheatsheet.md)。

第二種情況可以透過 Yocto 的 `devtool` 來輔助。執行完
`. setup <machine>` 之後，你的 path 中就會有一個名為 `devtool` 的工具，
可以有多種應用方式。

如果你已經有一個想要整合的來源碼樹（source tree），先執行
`devtool modify -n ${PACKAGE} ${SRCTREE}`，它會在你的建置目錄中建立一個
新的 Yocto layer，devtool 會將 recipe 的修改存放在這裡。接著它會為該
套件 recipe 建立一個 `.bbappend`，並使用 `externalsource` class 將下載、
抓取（fetch）與套用補丁（patch）的步驟取代為無動作（no-ops）。結果就是
當你建置該套件時，它會直接使用本地的來源碼目錄。請注意，套件 recipe
可能不會執行 clean，視你進行的操作而定，你可能需要在 `${SRCTREE}` 中執行
`${PACKAGE}` 建置系統的 clean 指令來清除任何已建置的物件。此外，如果你
修改了來源碼，可能需要執行 `bitbake -c cleansstate ${PACKAGE}` 來清除
BitBake 的快取。

或者，如果你還沒有本地的來源碼樹，但仍想修改該套件，呼叫
`devtool modify ${PACKAGE}` 會代你處理抓取、解包（unpack）與套用補丁的階段，
並在預設的工作區位置放置一份來源碼樹。

全部完成後，執行 `devtool reset ${PACKAGE}` 以從 devtool 的 Yocto layer
中移除 `.bbappend`。

關於 [devtool][0] 的更多資訊可以在 [Yocto Mega Manual][1] 中找到。

### 將檔案加入你的映像檔

關於 BitBake 的實際操作有很多範例。OpenEmbedded 的[ recipe 範例][2]
就是一個很棒的例子，也是這個 OpenBMC 專屬章節所依據的前提。

假設你寫了一些程式碼。你已經用 scp 把編譯好的二進位檔丟到 OpenBMC
系統上好一陣子了，而且你知道有更好的做法。讓它成為你 flash 映像檔的一部分
一起被建置出來吧。

執行 devtool 指令將你的 repo 加入工作區。在我的範例中，我在 GitHub 上有
一個包含我程式碼的 repo。

```bash
devtool add welcome https://github.com/causten/hello.git
```

現在編輯它幫你建立的 bb 檔。你可以直接用 `vim`，但 `devtool` 也可以編輯
recipe：`devtool edit-recipe welcome`，不必輸入完整的檔案路徑。

加入／修改這幾行。

```bash
do_install() {
        install -m 0755 -d ${D}${bindir} ${D}${docdir}/${PN}
        install -m 0644 ${S}/hello ${D}${bindir}
        install -m 0644 ${S}/README.md ${D}${docdir}/${PN}/
}
```

install 指令會建立目錄，然後把檔案複製進去。接著 BitBake 就會從傳統的
`/usr/bin` 和 `/usr/share/doc/welcome/README.md` 位置把它們取用。

最後一步是告訴 BitBake 你需要 `welcome` 這個 recipe

```bash
vim conf/local.conf
IMAGE_INSTALL_append = " welcome"
```

就這樣，重新編譯並開機你的系統，二進位檔 `hello` 就會在 `/usr/bin`，
而 `README.md` 就會在 `/usr/share/doc/welcome`。

### 了解你的映像檔裡有什麼

你當然可以刷機並開機來確認你的檔案有沒有成功放進去，但有更快的方法。
位於 `build/tmp` 深處目錄中的 `rootfs` 目錄，就是放置檔案以供打包的
暫存區（staging area）。

在我的範例中，要檢查 README.md 會不會被加入，只要……

```bash
ls build/tmp/work/${MACHINE}-openbmc-linux-gnueabi/obmc-phosphor-image/1.0-r0/rootfs/usr/share/doc/welcome/README.md
```

NXP 寫了一些 BitBake 的[實用](https://community.nxp.com/docs/DOC-94953)
指令範例，也能找出檔案

```bash
bitbake -g obmc-phosphor-image && cat pn-depends.dot |grep welcome
```

[0]:
  https://www.yoctoproject.org/docs/latest/mega-manual/mega-manual.html#using-devtool-in-your-sdk-workflow
  "devtool"
[1]:
  http://www.yoctoproject.org/docs/latest/mega-manual/mega-manual.html
  "Yocto Mega Manual"
[2]:
  http://www.embeddedlinux.org.cn/OEManual/recipes_examples.html
  "Recipe Example"
