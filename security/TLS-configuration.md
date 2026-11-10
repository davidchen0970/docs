# 如何設定伺服器 TLS 憑證以進行驗證

作者：Zbigniew Kurzynski <zbigniew.kurzynski@intel.com>

建立日期：2020 年 5 月 8 日

相關文件：

- [Redfish TLS 使用者驗證](https://github.com/openbmc/docs/blob/master/designs/redfish-tls-user-authentication.md)

## 簡介

透過本指南，你應該能夠建立由 CA 簽署的用戶端與伺服器憑證，用來驗證送往 OpenBMC 伺服器的使用者要求。你也會學到如何啟用並測試 OpenBMC 的 TLS 驗證。

## 憑證

若要讓憑證被標記為有效，它（以及鏈中的每一張憑證）都必須符合以下條件：

- `KeyUsage` 包含所需用途 `digitalSignature` 與 `keyAgreement`
  （請參閱 rfc 3280 4.2.1.3）
- `ExtendedKeyUsage` 對用戶端憑證包含所需用途 `clientAuth`，對伺服器憑證包含
  `serverAuth`（請參閱 rfc 3280 4.2.1.13）
- 公鑰符合最小位元長度要求
- 憑證必須處於其有效期限內
- `notBefore` 與 `notAfter` 欄位必須包含有效的時間
- 必須由憑證授權中心（certificate authority）正確簽署
- 憑證依 X.509 而言格式正確
- 對用戶端憑證而言，發行者名稱必須與 CA 的主體名稱相符
- 發行者名稱必須與你 OpenBMC 主機的完整網域名稱相符

如果你已經有憑證，可以直接跳到
[啟用 TLS 驗證](#enable-tls-authentication)，或前往
[驗證憑證](#verify-certificates)並檢查它們是否符合上述要求。

### 準備設定檔

若要產生具備所需參數的憑證，必須對預設的 openssl 設定檔做一些修改。

首先建立一個名為 `ca` 的新資料夾，並以預設設定檔作為範本來建立設定檔
（我們不想更動原本的那一份）。設定檔的位置會依作業系統而異。
在 Ubuntu 上通常是 `/usr/lib/ssl/openssl.cnf`，但也可能在 `/etc/ssl/openssl.cnf`。
在 Cygwin 上可能是 `/etc/defaults/etc/pki/tls/openssl.cnf`
或 `/etc/pki/tls/openssl.cnf`。

```bash
mkdir ~/ca
cd ~/ca
cp /usr/lib/ssl/openssl.cnf openssl-client.cnf
```

接著用你最喜歡的編輯器（例如 `vi`）開啟用戶端設定檔
`~/ca/openssl-client.cnf`。

```bash
vi ~/ca/openssl-client.cnf
```

找到下列各區塊，並加入或選擇所示的值。

```bash
[ req ]
req_extensions = v3_req

[ usr_cert ]
extendedKeyUsage = clientAuth

[ v3_req ]
extendedKeyUsage = clientAuth
keyUsage = digitalSignature, keyAgreement
```

現在透過複製用戶端檔案來建立伺服器設定檔 `openssl-server.cnf`

```bash
cp ~/ca/openssl-client.cnf openssl-server.cnf
```

並修改下列各區塊中所示的值。

```bash
[ usr_cert ]
extendedKeyUsage = serverAuth

[ v3_req ]
extendedKeyUsage = serverAuth
```

分別為用戶端與伺服器憑證建立另外兩個設定檔 `myext-client.cnf`
與 `myext-server.cnf`。若沒有這些檔案，就不會有任何擴充屬性被加入憑證中。

```bash
cat << END > myext-client.cnf
[ my_ext_section ]
keyUsage = digitalSignature, keyAgreement
extendedKeyUsage = clientAuth
authorityKeyIdentifier = keyid
END
```

```bash
cat << END > myext-server.cnf
[ my_ext_section ]
keyUsage = digitalSignature, keyAgreement
extendedKeyUsage = serverAuth
authorityKeyIdentifier = keyid
END
```

### 建立新的 CA 憑證

首先我們需要建立一個私密金鑰來簽署 CA 憑證。

```bash
openssl genrsa -out CA-key.pem 2048
```

現在我們可以使用先前產生的金鑰來建立 CA 憑證。系統會提示你輸入一些會被
納入憑證的資訊，例如國家、城市、公司名稱等。

```bash
openssl req -new -config openssl-client.cnf -key CA-key.pem -x509 -days 1000 -out CA-cert.pem
```

### 建立由指定 CA 憑證簽署的用戶端憑證

要建立用戶端憑證，必須先建立簽署要求（signing request）。
為此需要另一個私密金鑰。

產生一個將用來簽署憑證簽署要求的新金鑰：

```bash
openssl genrsa -out client-key.pem 2048
```

產生憑證簽署要求。

系統會提示你輸入與產生 CA 時相同的資訊，但請為此憑證的 `CommonName`
屬性提供 **OpenBMC 系統使用者名稱**。在本範例中使用 **root**。

```bash
openssl req -new -config openssl-client.cnf -key client-key.pem -out signingReqClient.csr
```

使用你的 `CA-cert.pem` 憑證，並以下列命令簽署憑證：

```bash
openssl x509 -req -extensions my_ext_section -extfile myext-client.cnf -days 365 -in signingReqClient.csr -CA CA-cert.pem -CAkey CA-key.pem -CAcreateserial -out client-cert.pem
```

檔案 `client-cert.pem` 現在包含一張已簽署的用戶端憑證。

### 建立由指定 CA 憑證簽署的伺服器憑證

為了方便起見，我們將使用在
[建立新的 CA 憑證](#create-a-new-ca-certificate)一節中所產生的同一個 CA，
不過也可以使用另一個不同的 CA。

產生一個將用來簽署伺服器憑證簽署要求的新金鑰：

```bash
openssl genrsa -out server-key.pem 2048
```

產生憑證簽署要求。系統會提示你輸入與產生 CA 時相同的資訊，但請為此憑證的
`CommonName` 屬性提供 **你 OpenBMC 伺服器的完整網域名稱**。在本範例中它會是
`bmc.example.com`。可以使用萬用字元（wildcard）來保護多部主機，例如一張設定為
`*.example.com` 的憑證將能保護 `www.example.com`、`mail.example.com`、
blog.example.com 以及其他主機。

```bash
openssl req -new -config openssl-server.cnf -key server-key.pem -out signingReqServer.csr
```

使用你的 `CA-cert.pem` 憑證，並以下列命令簽署憑證：

```bash
openssl x509 -req -extensions my_ext_section -extfile myext-server.cnf -days 365 -in signingReqServer.csr -CA CA-cert.pem -CAkey CA-key.pem -CAcreateserial -out server-cert.pem
```

檔案 `server-cert.pem` 現在包含一張已簽署的伺服器憑證。

### 驗證憑證

若要驗證簽署要求與兩張憑證，你可以使用下列命令。

```bash
openssl x509 -in CA-cert.pem -text -noout
openssl x509 -in client-cert.pem -text -noout
openssl x509 -in server-cert.pem -text -noout
openssl req -in signingReqClient.csr -noout -text
openssl req -in signingReqServer.csr -noout -text
```

以下是範例清單，你可以拿來與自己的結果比較。請特別注意以下屬性：

- 兩張憑證中的 Validity（有效期限），
- `client-cert.pem` 中的 `Issuer`，它必須與 `CA-cert.pem` 中的 `Subject` 相符，
- `client-cert.pem` 中的 _X509v3 extensions_ 區塊，它應該包含正確的值，
- `Public-Key` 長度，不可小於 2048 位元。
- `client-cert.pem` 中 `Subject` 的 CN，它應該與既有的 OpenBMC 使用者名稱相符。
  在本範例中是 **root**。
- `server-cert.pem` 中 `Subject` 的 CN，它應該與 OpenBMC 主機名稱相符。在本
  範例中是 **bmc.example.com**。（關於名稱限制，請參閱 rfc 3280 4.2.1.11）

以下是可供比較的憑證產生片段。

```text
CA-cert.pem
    Data:
        Version: 3 (0x2)
        Serial Number: 16242916899984461675 (0xe16a6edca3c34f6b)
    Signature Algorithm: sha256WithRSAEncryption
        Issuer: C=US, ST=California, L=San Francisco, O=Intel, CN=Test CA
        Validity
            Not Before: May 11 11:40:48 2020 GMT
            Not After : Feb  5 11:40:48 2023 GMT
        Subject: C=US, ST=California, L=San Francisco, O=Intel, CN=Test CA
        Subject Public Key Info:
            Public Key Algorithm: rsaEncryption
                Public-Key: (2048 bit)
                Modulus:
                    00:d4:24:c1:1d:ac:85:8c:5b:42:e4:f8:a8:d8:7c:
                    ...
                    55:83:8b:aa:ac:ac:6e:e3:01:2b:ce:f7:ee:87:21:
                    f9:2b
                Exponent: 65537 (0x10001)
        X509v3 extensions:
            X509v3 Subject Key Identifier:
                ED:FF:80:A7:F8:DA:99:7F:94:35:95:F0:92:74:1A:55:CD:DF:BA:FE
            X509v3 Authority Key Identifier:
                keyid:ED:FF:80:A7:F8:DA:99:7F:94:35:95:F0:92:74:1A:55:CD:DF:BA:FE

            X509v3 Basic Constraints:
                CA:TRUE
    Signature Algorithm: sha256WithRSAEncryption
         cc:8b:61:6a:55:60:2b:26:55:9f:a6:0c:42:b0:47:d4:ec:e0:
         ...
         45:47:91:62:10:bd:3e:a8:da:98:33:65:cc:11:23:95:06:1b:
         ee:d3:78:84
```

```text
client-cert.pem
    Data:
        Version: 3 (0x2)
        Serial Number: 10150871893861973895 (0x8cdf2434b223bf87)
    Signature Algorithm: sha256WithRSAEncryption
        Issuer: C=US, ST=California, L=San Francisco, O=Intel, CN=Test CA
        Validity
            Not Before: May 11 11:42:58 2020 GMT
            Not After : May 11 11:42:58 2021 GMT
        Subject: C=US, ST=California, L=San Francisco, O=Intel, CN=root
        Subject Public Key Info:
            Public Key Algorithm: rsaEncryption
                Public-Key: (2048 bit)
                Modulus:
                    00:cf:d6:d0:a2:09:62:df:e9:a9:b1:e1:3d:7f:2f:
                    ...
                    30:7b:48:dc:c5:2c:3f:a9:c0:d1:b6:04:d4:1a:c8:
                    8a:51
                Exponent: 65537 (0x10001)
        X509v3 extensions:
            X509v3 Key Usage:
                Digital Signature, Key Agreement
            X509v3 Extended Key Usage:
                TLS Web Client Authentication
            X509v3 Authority Key Identifier:
                keyid:ED:FF:80:A7:F8:DA:99:7F:94:35:95:F0:92:74:1A:55:CD:DF:BA:FE

    Signature Algorithm: sha256WithRSAEncryption
         7f:a4:57:f5:97:48:2a:c4:8e:d3:ef:d8:a1:c9:65:1b:20:fd:
         ...
         25:cb:5e:0a:37:fb:a1:ab:b0:c4:62:fe:51:d3:1c:1b:fb:11:
         56:57:4c:6a
```

```text
server-cert.pem
    Data:
        Version: 3 (0x2)
        Serial Number: 10622848005881387807 (0x936beffaa586db1f)
    Signature Algorithm: sha256WithRSAEncryption
        Issuer: C=US, ST=z, L=z, O=z, OU=z, CN=bmc.example.com
        Validity
            Not Before: May 22 13:46:02 2020 GMT
            Not After : May 22 13:46:02 2021 GMT
        Subject: C=US, ST=z, L=z, O=z, OU=z, CN=bmc.example.com
        Subject Public Key Info:
            Public Key Algorithm: rsaEncryption
                Public-Key: (2048 bit)
                Modulus:
                    00:d9:34:9c:da:83:c6:eb:af:8f:e8:11:56:2a:59:
                    ...
                    92:60:09:fc:f9:66:82:d0:27:03:44:2f:9d:6d:c0:
                    a5:6d
                Exponent: 65537 (0x10001)
        X509v3 extensions:
            X509v3 Key Usage:
                Digital Signature, Key Agreement
            X509v3 Extended Key Usage:
                TLS Web Server Authentication
            X509v3 Authority Key Identifier:
                keyid:5B:1D:0E:76:CC:54:B8:BF:AE:46:10:43:6F:79:0B:CA:14:5C:E0:90

    Signature Algorithm: sha256WithRSAEncryption
         bf:41:e2:2f:87:44:25:d8:54:9c:4e:dc:cc:b3:f9:af:5a:a3:
         ...
         ef:0f:90:a6

```

## 在 OpenBMC 上安裝 CA 憑證

CA 憑證可以透過 Redfish Service 安裝。檔案 `CA-cert.pem` 無法直接上傳，
必須內嵌在有效的 JSON 字串中送出，這需要跳脫 `\`、`"` 以及控制字元。
這意味著所有內容必須以將換行符號編碼為 `\n` 的方式，放置於單一行中的單一
字串內。下面的命令會準備好整個 POST 主體，並將其放入名為
`install_ca.json` 的檔案。

```bash
cat << END > install_ca.json
{
  "CertificateString":"$(cat CA-cert.pem | sed -n -e '1h;1!H;${x;s/\n/\\n/g;p;}')",
  "CertificateType": "PEM"
}
END
```

若要在 OpenBMC 伺服器上安裝 CA 憑證，請用此命令送出
`install_ca.json` 的內容：

其中 `${bmc}` 應該是 `bmc.example.com`。將它匯出為環境變數會比較方便。

```bash
curl --user root:0penBmc -d @install_ca.json -k -H "Content-Type: application/json" -X POST https://${bmc}/redfish/v1/Managers/bmc/Truststore/Certificates

```

憑證 `root:0penBmc` 可以替換成任何你選用的系統使用者名稱與密碼，
但必須具備此處所用資源的適當存取權限。

憑證成功安裝後，你應該會收到成功的 HTTP 回應，並且在這個資源集合下應該
可以取得一張新的憑證。

```bash
curl --user root:0penBmc -k https://${bmc}/redfish/v1/Managers/bmc/Truststore/Certificates

```

OpenBMC 預設已經存在一張自動產生的自簽伺服器憑證。若要使用由我們自己的
CA 簽署的憑證，就必須將它替換掉。此外我們還必須將用來簽署伺服器憑證的
私密金鑰上傳到 OpenBMC。可以使用這個命令準備好適當的訊息主體：

```bash
cat << END > replace_cert.json
{
  "CertificateString":"$(cat server-key.pem server-cert.pem | sed -n -e '1h;1!H;${x;s/\n/\\n/g;p;}')",
   "CertificateUri":
   {
      "@odata.id": "/redfish/v1/Managers/bmc/NetworkProtocol/HTTPS/Certificates/1"
   },
  "CertificateType": "PEM"
}
END
```

若要替換 OpenBMC 伺服器上的伺服器憑證，請用此命令送出
`replace_cert.json` 的內容：

```bash
curl --user root:0penBmc -d @replace_cert.json -k -H "Content-Type: application/json" -X POST https://${bmc}/redfish/v1/CertificateService/Actions/CertificateService.ReplaceCertificate/

```

## 啟用 TLS 驗證

若要檢查 TLS 驗證方式的目前狀態，請使用這個命令：

```bash
curl --user root:0penBmc -k https://${bmc}/redfish/v1/AccountService
```

並確認屬性 `Oem->OpenBMC->AuthMethods->TLS` 設為 true。

若要啟用 TLS 驗證，請使用這個命令：

```bash
curl --user root:0penBmc  -k -X PATCH -H "Content-Type: application/json" --data '{"Oem": {"OpenBMC": {"AuthMethods": { "TLS": true} } } }' https://${bmc}/redfish/v1/AccountService
```

若要停用 TLS 驗證，請使用這個命令：

```bash
curl --user root:0penBmc  -k -X PATCH -H "Content-Type: application/json" --data '{"Oem": {"OpenBMC": {"AuthMethods": { "TLS": false} } } }' https://${bmc}/redfish/v1/AccountService
```

其他驗證方式（例如 basic authentication）也可以使用相同的機制啟用或停用。
所有支援的驗證方式都可以在 `/redfish/v1/AccountService` 資源的屬性
`Oem->OpenBMC->AuthMethods` 下取得。

## 使用 TLS 存取 OpenBMC 資源

如果 TLS 已啟用、有效的 CA 憑證已上傳且伺服器憑證已替換，那麼應該就能只
使用用戶端憑證、金鑰與 CA 來執行 curl 要求，如下所示。

```bash
curl --cert client-cert.pem --key client-key.pem -vvv --cacert CA-cert.pem https://${bmc}/redfish/v1/SessionService/Sessions
```

## TLS 設定過程中的常見錯誤的常見錯誤

- OpenBMC 上的日期與時間無效，

- 測試 Redfish 資源（例如 `https://${bmc}/redfish/v1`），這類資源無需任何
  驗證即可存取，因此不論 TLS 是否停用或憑證是否無效，結果永遠都會成功。

- 憑證不符合要求。請參閱
  [驗證憑證](#verify-certificates)一節。

- 嘗試載入同一張憑證兩次會導致錯誤。

- 建置中未包含 phosphor-bmcweb-cert-config。
