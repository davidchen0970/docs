# OpenBMC 主機主控台支援

本文件說明如何從 OpenBMC 管理系統連線到主機 UART 主控台。

主控台基礎架構允許對單一主機 UART 建立多個共用連線。來自主機的
UART 資料會輸出到所有連線，而來自任一連線的輸入則會送到主機。

## 遠端主控台連線

若要從遠端連線到 OpenBMC 主控台工作階段，只要透過 ssh 連到你 BMC
的 2200 埠即可。使用與一般 ssh 工作階段相同的登入憑證：

```bash
ssh -p 2200 [user]@[bmc-hostname]
```

## 本機主控台連線

如果你已經登入一台 OpenBMC 機器，可以直接啟動主控台工作階段，使用：

```bash
obmc-console-client
```

若要離開主控台，請輸入：

```text
    return ~ .
```

請注意，若你是在 ssh 連線上，需要「跳脫」~ 字元，也就是輸入兩次：

```text
    return ~ ~ .
```

這是因為 obmc-console-client 本身就是一個 ssh 工作階段，因此需要
兩個 `~` 來跳脫「內層」（obmc-console-client）的 ssh 工作階段。

## 記錄

主控台記錄保留在：`/var/log/obmc-console.log`

此記錄的大小受限，達到上限後會循環覆蓋（目前設定為 16kB）。
