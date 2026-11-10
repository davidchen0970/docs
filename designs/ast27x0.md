# AST27x0

作者：Vince Chang <vince_chang@aspeedtech.com>

建立日期：2025 年 12 月 10 日

## 預先編譯二進位檔說明

AST27x0 `image-u-boot` 包含用於 AST27x0 初始啟動（bring-up）的預先編譯韌體二進位檔。

### 預先編譯二進位檔位置

預先編譯的二進位檔存放於 bmc-pb 程式庫中：

- <https://github.com/AspeedTech-BMC/bmc-pb>

### 預先編譯二進位檔摘要

| 檔案名稱          | 類別           | 說明                                                                                      |
| ----------------- | -------------- | ------------------------------------------------------------------------------------------- |
| caliptra-fw\*.bin | Caliptra       | Caliptra 二進位檔包含 Firmware Manifest、Caliptra FMC 二進位檔以及 Caliptra RT 二進位檔 |
| ddr\*.bin         | 供應商 IP      | DDR PHY 韌體（DDR4 / DDR5）                                                              |
| dp_fw.bin         | 專屬 IP        | Display Port 控制器韌體                                                            |
| uefi_ast2700.bin  | VBIOS          | AST2700 UEFI 開機韌體                                                                  |

#### Caliptra 二進位檔說明

Caliptra 二進位檔包含 Firmware Manifest、Caliptra FMC 二進位檔以及 Caliptra RT 二進位檔。Caliptra FMC 與 RT 二進位檔係由 caliptra-sw 專案建置而成
（<https://github.com/chipsalliance/caliptra-sw/commit/51ff0a89f169bbf8e06acb49b31db555e99fefb6>）。

Firmware Manifest 含有 ASPEED 的供應商金鑰雜湊值（key hash）與簽章，這是韌體身分驗證所必需的。

在 MP（量產，Mass Production）元件上，供應商金鑰雜湊值會作為硬體信任根（Root of Trust）的一部分，永久熔入（fused）OTP 中。開機期間，Caliptra ROM 會透過以熔入金鑰雜湊值驗證簽章的方式來驗證韌體。

因此，任何未使用 ASPEED 私鑰簽章、直接從原始碼建置的 Caliptra 韌體映像都無法通過身分驗證。在這種情況下，金鑰雜湊值驗證將會失敗，韌體將無法開機。

#### DDR 二進位檔說明

DDR 二進位檔用於 DDR4/DDR5 PHY 訓練（training）。這些二進位檔由 DDR PHY IP 供應商的工具鏈所提供／產生。因此，我們僅提供預先編譯的二進位檔。

#### Display Port 二進位檔說明

由於 IP 與授權限制，Display Port 控制器韌體以預先編譯的二進位檔形式提供，並不會在 OpenBMC 建置流程中產生。

#### UEFI 二進位檔說明

AST2700 UEFI 二進位檔以預先編譯的產出物（artifact）形式交付，因為它必須經過由供應商控管的 EV（Extended Validation）簽章流程，以滿足主機安全開機（secure boot）的需求。該二進位檔在交付之前，會於獨立的受控環境中建置並簽章。
