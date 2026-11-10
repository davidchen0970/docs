# 從 AMI 固件轉換到 OpenBMC

從 AMI 固件轉換到 OpenBMC 通常需要多個步驟。以下針對各種機器記錄這些步驟。

## Palmetto

1. 將固件快閃記憶體晶片升級為 256Mb（32MB）的元件（MX25L25635E 或同等產品）
2. [建置](cheatsheet.md#building-for-palmetto)並將 `image-bmc` 輸出燒錄到 256Mb 的快閃記憶體晶片
3. 更換主機板上現有的快閃記憶體晶片（最靠近電池的插座）
4. 製作一條纜線連接到 COM2 排針以提供 BMC 主控台。COM2 採用 DB-9 式的接腳配置。
5. 將 `BMC_DEBUG1`（TX）和 `BMC_DEBUG2`（RX）兩組跳線重新設定為短路接腳 2-3。
6. 在你的 Palmetto 上啟動 OpenBMC。主控台的傳輸速率為 115200baud。
