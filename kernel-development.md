# OpenBMC kernel 與 u-boot 開發

> [!IMPORTANT]
>
> 以下所述的 kernel 開發理念同樣適用於 OpenBMC 的
> u-boot fork，另需遵循 [U-Boot](#u-boot) 中說明的額外規則。

OpenBMC 專案維護了一個供專案本身使用的 kernel tree。此 tree 的
整體開發政策是「程式碼必須先進入 upstream」。這點強烈建議但並非
硬性要求，可視個別情況作出例外處理。若有疑問，請在郵件列表上
發起討論。

OpenBMC kernel tree 託管於 <https://github.com/openbmc/linux>，
其中包含了我們所攜帶（carry）的 patch 集合。理想情況下不應攜帶
任何 patch，因為所有內容都應該已經進入 upstream。

你的程式碼會以以下幾種方式進入 OpenBMC tree，由最理想到最不理想
排列：

1. 當 OpenBMC kernel 跟進到新的 upstream 發行版時
2. 透過將較新 kernel 版本的 upstream commit 反向移植（backport）到
   OpenBMC kernel
3. 暫時收錄在 OpenBMC tree 中的 patch

## 快速摘要（TL;DR）

若你需要將某個 patch 加入 OpenBMC tree，請依循以下步驟：

1. 將你的 patch 提交到 upstream。它不一定要已經在 upstream 中，但應該
   正在進行的路上，且沒有任何尚未解決的設計疑慮
2. 若該 patch 已經被套用（applied）到 upstream，請使用
   `git cherry-pick -x -s ${COMMIT_HASH}` 將該 commit cherry pick 到
   OpenBMC tree。這樣做的話，原始的 commit hash 會被加入 commit message
   中
3. 使用
   `git format-patch --subject-prefix="PATCH linux ${BRANCH}" --to=openbmc@lists.ozlabs.org --to=andrew@codeconstruct.com.au --to=tan.siewert@9elements.com --cc=joel@jms.id.au`
   來產生格式化過的 patch。若是 u-boot，請使用 `PATCH u-boot ${BRANCH}`
   作為 subject prefix

## 開發新的驅動程式

在開發新的驅動程式時，你的目標是讓程式碼能被 upstream 接受。第一步
應該是確認目前還沒有支援你想支援的硬體之現有驅動程式。請檢查 OpenBMC
的 `dev-` 分支、檢查 upstream，如果你在這些地方都找不到支援，就在郵件
列表上詢問。

一旦你確定必須撰寫某個驅動程式，就應該先開發並測試該驅動程式，再將它
送往 upstream 給相關的維護者（maintainers）。在送往 upstream 時將 OpenBMC
列表加入 cc 是十分歡迎的，這樣其他 kernel 開發者就能在適當時機提供意見。
務必遵循〔upstream 開發流程〕(https://www.kernel.org/doc/Documentation/process/submitting-patches.rst)。

過去 patch 會在 OpenBMC 郵件列表上進行「預先審查（pre-review）」。雖然
這對於不熟悉撰寫 kernel 程式碼的開發者很有幫助，但卻造成大家對 upstream
流程的混淆，因此現在我們所有的開發都在社群中進行。

一旦驅動程式被 upstream 接受，請將這個好消息連同對 upstream tree 的參考
一併發送到 OpenBMC 列表。這可能是 Linus 的 tree，也可能是某位子系統維護者
的 tree。之後 OpenBMC kernel 團隊就能決定將你的程式碼納入 OpenBMC tree
的最佳方式。

### 例外情況

有些情況下，等待 upstream 接受會延遲新系統的 bring-up。這應該透過謹慎的
規劃以及提早將功能在 upstream 開發來避免；但若情況並非如此，我們可以選擇
在 upstream 開發持續進行的期間，於 OpenBMC tree 中攜帶這些 patch。

「先進入 upstream」規則的另一個例外是 patch 修改了並非 upstream 的檔案。
這目前包含 aspeed 的板級檔案（board file）
`arch/arm/mach-aspeed/aspeed.c`，以及 device tree source 檔案 `dts`。當我們
為所有功能都寫好驅動程式後，這個 board file 就應該消失；目前它包含了一些
與 LPC 及早期初始化（early init）相關的 hack。

如果你發現自己正在修改 `arch/arm/mach-aspeed/aspeed.c`，請先發一封 email
到 OpenBMC 列表以徵詢 kernel 開發者的意見。由於一旦我們為所有 Aspeed
週邊都寫好驅動程式後這個檔案就會被移除，因此針對 `aspeed.c` 的 patch
會帶有些許成見地被審查。

## U-Boot

OpenBMC 在 <https://github.com/openbmc/u-boot> 維護了自己的 u-boot fork。

`v2019.04-aspeed-openbmc` 分支與 upstream v2019.04 差異很大，主要是因為它
攜帶了 ASPEED 的支援，且與最新的 upstream u-boot 不相容。因此，現有
upstream u-boot 中的功能可能無法取得，也可能無法被反向移植。

某個 patch 應該屬於哪個 tree，取決於 SoC：

- ASPEED AST2600 及更早的版本：patch 送到差異化的 v2019.04 tree。
  同時將變更送到 upstream 也很歡迎，但我們無法保證 upstream 的程式碼
  能在這些平台上運作。
- ASPEED AST2700 及之後的版本：patch **必須**在 upstream 維護。它們
  不會被反向移植到 v2019.04。
- 非 ASPEED 的 SoC：與 AST2700 相同的規則，patch **必須**在 upstream
  維護。SoC 支援不會被反向移植到 v2019.04。

### Device trees

u-boot 中的 device tree 是一個特殊情況。與 kernel 不同，OpenBMC 上的
u-boot 通常只需要初始化足夠的硬體以載入 kernel，而預設的 EVB device tree
通常就足夠應付這種需求。

只有在確實需要時，才為 u-boot 送出特定機器的 device tree，例如當 u-boot
本身要用於網路開機（netbooting）而需要一個描述正確的網路或儲存路徑時，
或當你的平台需要 ECC 或類似功能時。

## 測試

在提交 patch 之前，建議你至少在 Qemu 平台上以及任何你手上可取得的硬體上
進行開機測試。

## 小技巧與訣竅

一些 kernel 開發的通用提示

### 樹外建置（Out-of-tree builds）

你可以在 yocto 環境之外建置 kernel，方法是直接使用 initramfs
（來自既有的 yocto 建置）：

```sh
make ARCH=arm \
    O=obj \
    CROSS_COMPILE=arm-linux-gnueabihf- \
    CONFIG_INITRAMFS_SOURCE=.../obmc-phosphor-image-palmetto.cpio.gz
```

（請適度調整 `O` 與 `CROSS_COMPILE` 參數）。

你需要使用一個相關的 BMC defconfig（例如 `aspeed_g4_defconfig` 或
`aspeed_g5_defconfig`）作為你的基礎 kernel 設定。

這個 cpio 可以在相關機器目錄下的以下 yocto 輸出目錄中找到：

```sh
 build/tmp/deploy/images/
```

### 建置 uImage

若要建置 uImage（例如用於網路開機）：

```sh
# build a zImage using the obmc rootfs
make ARCH=arm \
    O=obj \
    CROSS_COMPILE=arm-linux-gnueabihf- \
    CONFIG_INITRAMFS_SOURCE=/path/tp/obmc-phosphor-image-palmetto.cpio.gz

# create a combined zImage + DTB image
cat obj/arch/arm/boot/zImage \
    obj/arch/arm/boot/dts/aspeed-bmc-opp-palmetto.dtb \
        > obj/aspeed-zimage

# create a uImage
./scripts/mkuboot.sh -A arm -O linux -C none -T kernel \
    -a 0x40008000 -e 0x40008000 -n $USER-`date +%Y%m%d%H%M` \
    -d obj/aspeed-zimage obj/uImage
```

請注意，有些系統可能已升級到以 FIT 為基礎的 u-boot，在那種情況下舊的
uImage 格式將不再被接受。
