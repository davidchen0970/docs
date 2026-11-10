# OpenBMC 核心內 MCTP

作者：Jeremy Kerr <jk@codeconstruct.com.au>

關於一般性的 MCTP 設計說明、背景與需求，請參考 [MCTP 總覽](mctp.md)文件。

本文件描述一個以核心為基礎的 MCTP 基礎設施實作，為以 OpenBMC 為基礎的平台
提供一套基於 socket 的 MCTP 通訊 API。

## 核心實作的需求

- MCTP 訊息 API 應為既有 POSIX socket 介面的直覺應用

- 對於一個單純的 MCTP 端點而言，設定應該很簡單：單一網路搭配單一的本機
  端點 ID（EID）。

- 基礎設施應具備足夠的彈性，以容納更複雜的 MCTP 網路，並允許：
  - 每個 MCTP 網路（如 DSP0236 第 3.2.31 節所定義）可由多個本機實體介面
    及／或多個 EID 組成；

  - 多個彼此獨立（亦即非橋接）的網路，且各網路之間可能存在重複的
    EID；

  - 單一介面上可有多個本機 EID，以及

  - 網路內可自訂路由／橋接設定。

## 建議設計

本設計包含數個元件：

- 供使用者空間應用程式收發 MCTP 訊息的介面：將 sockets API 對應至 MCTP
  用途的一組映射

- 用於控制與設定 MCTP 網路的基礎設施，包含一個設定工具，以及供該工具
  使用的核心訊息傳遞機制。

- 供實體介面綁定使用的核心驅動程式。

一般來說，核心元件涵蓋 MCTP 的傳輸功能，例如訊息的組裝／拆解、封包轉發，
以及實體介面實作。

較高階的協定（例如 PLDM）則透過所引入的 socket API 在使用者空間實作。
這也涵蓋了 MCTP 控制協定（MCTP Control Protocol，DSP0236 第 11 節）實作的
大部分內容——MCTP 端點通常會有一個專屬程序來要求並回應控制協定訊息。
不過，核心會納入一小部分控制協定程式碼，好讓那些僅具靜態 EID 分配、
功能極為簡單的端點，無需此程序即可執行。需要超出單一端點功能（匯流排擁有者、
橋接器等）及／或動態 EID 分配的 MCTP 端點，才會納入控制訊息協定程序。

每個實體介面綁定都會引入一個新的驅動程式來處理。這些驅動程式會暴露相應的
`struct net_device`，以處理其關聯硬體通道上 MCTP 封包的傳送與接收。在 Linux
下，這些介面的命名空間有別於其他網路介面——例如乙太網路的介面。

### 結構：介面與網路

核心透過兩個項目來建模本機 MCTP 拓撲：介面（interfaces）與網路（networks）。

介面（或稱「連結」）是 MCTP 實體傳輸綁定（如 DSP0236 第 3.2.47 節所定義）
的一個實例，通常連接至某個特定硬體裝置。它以 `struct netdevice` 表示，
並有一個使用者可見的名稱與索引（`ifindex`）。系統也允許非實體附加的介面，
以支援本機迴環及／或虛擬介面。

網路以端點 ID 為 MCTP 端點定義一個唯一的位址空間（如 DSP0236 第 3.2.31 節
所述）。網路有一個使用者可見的識別碼，以便從使用者空間參照它。路由定義
為單一網路所專屬。

介面與單一網路相關聯。一個網路可與一個或多個介面相關聯。

若存在多個網路，每個網路都可能包含在其他網路中也存在的 EID。

### Sockets API

#### 協定定義

我們為 MCTP 定義一個新的位址家族（以及相對應的協定家族）：

```c
    #define AF_MCTP /* TBD */
    #define PF_MCTP AF_MCTP
```

MCTP socket 以 `socket()` 系統呼叫建立，並將 `AF_MCTP` 指定為領域。
目前僅定義了 `SOCK_DGRAM` 此種 socket 類型。

```c
    int sd = socket(AF_MCTP, SOCK_DGRAM, 0);
```

`protocol` 引數唯一的（目前）值為 0。日後可能會新增其他協定實作。

以通訊協定值 0 開啟的 MCTP Socket 會直接在傳輸層進行通訊；應用程式收到的
訊息緩衝區，將由已重新組裝的 MCTP 封包中的訊息資料組成，並包含完整訊息，
含訊息類型位元組與選用的訊息完整性檢查碼（IC）。個別封包的標頭不包含在內；
它們未來或許可透過 `SOCK_RAW` socket 類型存取。

與所有 socket 位址家族一樣，來源與目的位址以一個新的 `sockaddr` 類型指定：

```c
    struct sockaddr_mctp {
            sa_family_t         smctp_family; /* = AF_MCTP */
            int                 smctp_network;
            struct mctp_addr    smctp_addr;
            uint8_t             smctp_type;
            uint8_t             smctp_tag;
    };

    struct mctp_addr {
            uint8_t             s_addr;
    };

    /* MCTP network values */
    #define MCTP_NET_ANY        0

    /* MCTP EID values */
    #define MCTP_ADDR_ANY       0xff
    #define MCTP_ADDR_BCAST     0xff

    /* MCTP type values. Only the least-significant 7 bits of
     * smctp_type are used for tag matches; the specification defines
     * the type to be 7 bits.
     */
    #define MCTP_TYPE_MASK      0x7f

    /* MCTP tag definitions; used for smcp_tag field of sockaddr_mctp */
    /* MCTP-spec-defined fields */
    #define MCTP_TAG_MASK    0x07
    #define MCTP_TAG_OWNER   0x08
    /* Others: reserved */

    /* Helpers */
    #define MCTP_TAG_RSP(x) (x & MCTP_TAG_MASK) /* response to a request: clear TO, keep value */
```

#### 系統呼叫行為

以下各節描述標準 socket 系統呼叫中與 MCTP 相關的行為。這些行為的選擇旨在
與既有 sockets API 緊密對應。

##### `bind()`：設定本機 socket 位址

會接收傳入要求封包的 socket，會使用 `bind()` 系統呼叫繫結至一個本機位址。

```c
    struct sockaddr_mctp addr;

    addr.smctp_family = AF_MCTP;
    addr.smctp_network = MCTP_NET_ANY;
    addr.smctp_addr.s_addr = MCTP_ADDR_ANY;
    addr.smctp_type = MCTP_TYPE_PLDM;
    addr.smctp_tag = MCTP_TAG_OWNER;

    int rc = bind(sd, (struct sockaddr *)&addr, sizeof(addr));
```

這會確立 socket 的本機位址。符合該網路、位址與訊息類型的傳入 MCTP 訊息，
將由這個 socket 接收。這裡「傳入」一詞相當重要；已繫結的 socket 只會接收
帶有 TO 位元設定為 1 的訊息，以表示這是傳入的要求訊息，而非回應。

`smctp_tag` 的值將設定此 socket 從遠端接受哪些 tag。基於上述設定，唯一有效
的值是 `MCTP_TAG_OWNER`，這會讓遠端「擁有」的 tag 被路由至這個 socket。
由於設定了 `MCTP_TAG_OWNER`，`smctp_tag` 最低有效的 3 個位元並不被使用；
呼叫端必須將它們設為零。詳情請參閱
[傳送訊息的 Tag 行為](#傳送訊息的-tag-行為)一節。若未設定 `MCTP_TAG_OWNER`
位元，`bind()` 將失敗，並帶有 `EINVAL` 的 errno。

`smctp_network` 值為 `MCTP_NET_ANY` 時，會將 socket 設定為接收來自任何
本機連接網路的傳入封包。指定特定網路值時，socket 將只會接收來自該網路的
傳入訊息。

`smctp_addr` 欄位指定要繫結的本機位址。值為 `MCTP_ADDR_ANY` 時，會將 socket
設定為接收送往任何本機目的 EID 的訊息。

`smctp_type` 欄位指定要接收哪些訊息類型。對於傳入訊息，僅比對類型的低 7
個位元（亦即最高有效的 IC 位元不屬於比對的一部分）。這會讓 socket 同時接收
帶有與不帶有訊息完整性檢查尾註的封包。

##### `connect()`：設定遠端繫結位址

可以在 `bind()` 之前先使用 `connect()` 系統呼叫，將 `bind()` 的範圍限制到
單一遠端位址：

```c
    struct sockaddr_mctp peer_addr = { 0 }, local_addr = { 0 };
    int rc;

    peer_addr.smctp_family = AF_MCTP;
    peer_addr.smctp_network = MCTP_NET_ANY;
    peer_addr.smctp_addr.s_addr = 8; // remote peer EID
    peer_addr.smctp_type = MCTP_TYPE_PLDM;

    rc = connect(sd, (struct sockaddr *)&addr, sizeof(addr));
    if (rc) {
        /* handle error */
    }

    local_addr.smctp_family = AF_MCTP;
    local_addr.smctp_network = MCTP_NET_ANY;
    local_addr.smctp_addr.s_addr = MCTP_ADDR_ANY; // local EID
    local_addr.smctp_type = MCTP_TYPE_PLDM;

    rc = bind(sd, (struct sockaddr *)&local_addr, sizeof(local_addr));
```

與未限制範圍的 `bind()` 不同，此時 socket 只會接收來自某個特定對等端、
TO=1 的傳入訊息。這讓多個獨立的 socket 能接收相同類型、但來自不同對等端
的傳入訊息。

##### `sendto()` 與 `sendmsg()`：傳送一封 MCTP 訊息

MCTP 訊息透過 `sendto()` 或 `sendmsg()` 系統呼叫之一來傳送。以下以 `sendto()`
為主要範例：

```c
    struct sockaddr_mctp addr;
    char buf[13];
    ssize_t len;

    /* set message destination */
    addr.smctp_family = AF_MCTP;
    addr.smctp_network = 0;
    addr.smctp_addr.s_addr = 8;
    addr.smctp_tag = MCTP_TAG_OWNER;
    addr.smctp_type = MCTP_TYPE_ECHO;

    /* arbitrary message to send */
    memcpy(buf, "hello, world!", sizeof(buf) - 1);

    len = sendto(sd, buf, sizeof(buf), 0,
                    (struct sockaddr_mctp *)&addr, sizeof(addr));
```

位址引數指定對等端位址：網路與位址欄位定義要送往的遠端位址。若 `smctp_tag`
帶有 `MCTP_TAG_OWNER`，核心會忽略 `MCTP_TAG_VALUE` 中設定的任何位元，並為
目的 EID 產生一個合適的 tag 值。若未設定 `MCTP_TAG_OWNER`，訊息將以指定的
tag 值傳送。若無法分配 tag 值，系統呼叫將回報 `EAGAIN` 的 errno。

訊息緩衝區不包含訊息類型位元組；該位元組由 `smctp_type` 值指定。若要在
傳送的訊息中納入訊息完整性檢查碼，必須將它提供在訊息緩衝區中，且
`smctp_type` 的最高有效位元必須為 1。

`sendmsg()` 系統呼叫允許更精簡的引數介面，並可將訊息緩衝區指定為分散聚集
清單（scatter-gather list）。目前尚未定義任何輔助訊息類型（用於傳給
`sendmsg()` 的 `msg_control` 資料）。

在指定了 `MCTP_TAG_OWNER` 的 socket 上傳送訊息，會導致分配一個 tag。
（來源 EID、目的 EID、tag）這個元組充當一個隱含的本機 socket 位址，讓
socket 能接收對此送出訊息的回應。

Socket 只會接收它自己送出（TO=1）的要求的回應，且只能回應（TO=0）它自己
收到的要求。

##### `recvfrom()` 與 `recvmsg()`：接收一封 MCTP 訊息

應用程式可透過 `recvfrom()` 或 `recvmsg()` 系統呼叫之一來接收 MCTP 訊息。
以下以 `recvfrom()` 為主要範例：

```c
    struct sockaddr_mctp addr;
    socklen_t addrlen;
    char buf[13];
    ssize_t len;

    addrlen = sizeof(addr);

    len = recvfrom(sd, buf, sizeof(buf), 0,
                    (struct sockaddr_mctp *)&addr, &addrlen);

    /* We can expect addr to describe an MCTP address */
    assert(addrlen >= sizeof(buf));
    assert(addr.smctp_family == AF_MCTP);

    printf("received %zd bytes from remote EID %d, type %d\n", rc,
        addr.smctp_addr, addr.smctp_type);
```

`recvfrom` 與 `recvmsg` 的位址引數會被填入傳入訊息的遠端位址，包含 tag 值
（這是回覆該訊息時所需要的）。

傳入位址引數的 `smctp_type` 欄位會被填入所收到的 MCTP 訊息類型，包含 IC
位元。類型位元組不會出現在回傳的緩衝區資料中。若設定了 IC，完整性檢查碼會
跟在訊息之後，並會被包含在收到的緩衝區中。

與傳送呼叫一樣，socket 只會接收它自己送出（TO=1）的要求的回應，且只能
回應（TO=0）它自己收到的要求。

##### Socket 選項

以下 socket 選項為 MCTP socket 所定義：

###### `MCTP_ADDR_EXT`：在 sendmsg/recvmsg 中使用延伸位址資訊

啟用此 socket 選項可讓應用程式在傳送的封包上指定延伸位址資訊，並在收到的
封包上存取相同的資訊。

啟用 `MCTP_ADDR_EXT` socket 選項後，應用程式可向 `recvfrom()` 與 `sendto()`
系統呼叫指定一個擴充的 `struct sockaddr`。其定義如下：

```c
    struct sockaddr_mctp_ext {
            /* fields exactly match struct sockaddr_mctp */
            sa_family_t         smctp_family; /* = AF_MCTP */
            int                 smctp_network;
            struct mctp_addr    smctp_addr;
            uint8_t             smcp_tag;
            /* extended addressing */
            int                 smctp_ifindex;
            uint8_t             smctp_halen;
            unsigned char       smctp_haddr[/* TBD */];
    }
```

若指定給 `sendto()` 或 `recvfrom()` 的 `addrlen` 足以容納這個較大的結構，
則會各自消費／填入這些延伸位址欄位。

##### `ioctl`

以下 `ioctl` 為 MCTP socket 所定義：

###### `SIOCMCTPALLOCTAG2` / `SIOCMCTPDROPTAG2`：明確的 tag 控制

若某個 MCTP 應用層協定需要進階的 tag 控制（例如，採用非單純 1:1 要求／
回應模式的機制，或是對逾時有精確控制），應用程式可以執行明確的 tag 分配
與釋放。這兩個 `ioctl` 都接受一個 `struct mctp_ioc_tag_ctl2` 作為引數：

```c
struct mctp_ioc_tag_ctl2 {
    /* Peer details: network ID, peer EID, local EID. All set by the
     * caller.
     *
     * Local EID must be MCTP_ADDR_NULL or MCTP_ADDR_ANY in current
     * kernels.
     */
    unsigned int  net;
    mctp_eid_t    peer_addr;
    mctp_eid_t    local_addr;

    /* Set by caller, but no flags defined currently. Must be 0 */
    __u16         flags;

    /* For SIOCMCTPALLOCTAG2: must be passed as zero, kernel will
     * populate with the allocated tag value. Returned tag value will
     * always have TO and PREALLOC set.
     *
     * For SIOCMCTPDROPTAG2: userspace provides tag value to drop, from
     * a prior SIOCMCTPALLOCTAG2 call (and so must have TO and PREALLOC set).
     */
    __u8          tag;
};
```

tag 可以使用 `SIOCMCTPALLOCTAG2` `ioctl` 預先分配。這些 tag 會設定
`MCTP_TAG_PREALLOC` 位元。

分配 tag 會取得一個持續性的 TO=1 tag，供傳送送出訊息時使用。當我們收到
回應時，該 tag 不會被隱含地釋放，也不會受 tag 逾時影響。

呼叫端完成後應使用 `SIOCMCTPDROPTAG2` 釋放該 tag。在 socket 關閉時，tag
也會被釋放。

```c
    struct mctp_ioc_tag_ctl2 ctl = {
        .peer_addr = 8;
    };

    /* allocate a tag */
    rc = ioctl(sd, SIOCMCTPALLOCTAG2, &ctl);

    /* tag will be populated in ctl.tag, which we can pass directly as
     * smctp_tag value to sendto(): */
    addr.smctp_family = AF_MCTP;
    addr.smctp_addr.s_addr = 8;
    addr.smctp_tag = ctl.tag;

    rc = sendto(sd, buf, len, 0, &addr, sizeof(addr));

    /* we may do other operations using the same tag, as required */

    /* release tag */
    rc = ioctl(sd, SIOCMCTPDROPTAG2, &ctl);
```

##### 未實作的系統呼叫

以下系統呼叫並未為 MCTP 實作，主要因為它們並不用於 `SOCK_DGRAM` 類型的
socket：

- `listen()`
- `accept()`
- `shutdown()`
- `mmap()`

#### 使用者空間範例

這些範例涵蓋三種一般使用情境：

- **請求端（requester）**：向特定的（EID、類型）目標送出要求，並接收這些
  封包的回應

  這類似於典型的 UDP 用戶端

- **回應端（responder）**：接收所有以本機為目的、特定訊息類型的訊息，並立即
  回覆請求端。

  這類似於典型的 UDP 伺服器

- **控制器（controller）**：匯流排擁有者的專屬服務；可能送出廣播訊息、管理
  EID 分配、更新本機 MCTP 堆疊狀態。需要低階封包資料。

  這類似於 DHCP 伺服器。

##### 請求端

「用戶端」端的實作，向回應端送出要求並接收回應。它使用一個（虛構的）訊息
類型 `MCTP_TYPE_ECHO`。

```c
    int main() {
            struct sockaddr_mctp addr;
            socklen_t addrlen;
            struct {
                uint8_t type;
                uint8_t data[14];
            } msg;
            int sd, rc;

            sd = socket(AF_MCTP, SOCK_DGRAM, 0);

            addr.sa_family = AF_MCTP;
            addr.smctp_network = MCTP_NET_ANY; /* any network */
            addr.smctp_addr.s_addr = 9;    /* remote eid 9 */
            addr.smctp_tag = MCTP_TAG_OWNER; /* kernel will allocate an owned tag */
            addr.smctp_type = MCTP_TYPE_ECHO; /* fictitious message type */
            addrlen = sizeof(addr);

            /* set message type and payload */
            msg.type = MCTP_TYPE_ECHO;
            strncpy(msg.data, "hello, world!", sizeof(msg.data));

            /* send message */
            rc = sendto(sd, &msg, sizeof(msg), 0,
                            (struct sockaddr *)&addr, addrlen);

            if (rc < 0)
                    err(EXIT_FAILURE, "sendto");

            /* Receive reply. This will block until a reply arrives,
             * which may never happen. Actual code would need a timeout
             * here. */
            rc = recvfrom(sd, &msg, sizeof(msg), 0,
                        (struct sockaddr *)&addr, &addrlen);
            if (rc < 0)
                    err(EXIT_FAILURE, "recvfrom");

            assert(msg.type == MCTP_TYPE_ECHO);
            /* ensure we're nul-terminated */
            msg.data[sizeof(msg.data)-1] = '\0';

            printf("reply: %s\n", msg.data);

            return EXIT_SUCCESS;
    }
```

##### 回應端

「伺服器」端的實作，接收要求並回應。與用戶端一樣，它在 `struct sockaddr_mctp`
中使用一個（虛構的）訊息類型 `MCTP_TYPE_ECHO`；只有符合此類型的訊息才會被
接收。

```c
    int main() {
            struct sockaddr_mctp addr;
            socklen_t addrlen;
            int sd, rc;

            sd = socket(AF_MCTP, SOCK_DGRAM, 0);

            addr.sa_family = AF_MCTP;
            addr.smctp_network = MCTP_NET_ANY; /* any network */
            addr.smctp_addr.s_addr = MCTP_EID_ANY;
            addr.smctp_type = MCTP_TYPE_ECHO;
            addr.smctp_tag = MCTP_TAG_OWNER;
            addrlen = sizeof(addr);

            rc = bind(sd, (struct sockaddr *)&addr, addrlen);
            if (rc)
                    err(EXIT_FAILURE, "bind");

            for (;;) {
                    struct {
                        uint8_t type;
                        uint8_t data[14];
                    } msg;

                    rc = recvfrom(sd, &msg, sizeof(msg), 0,
                                    (struct sockaddr *)&addr, &addrlen);
                    if (rc < 0)
                            err(EXIT_FAILURE, "recvfrom");
                    if (rc < 1)
                            warnx("not enough data for a message type");

                    assert(addrlen == sizeof(addr));
                    assert(msg.type == MCTP_TYPE_ECHO);

                    printf("%zd bytes from EID %d\n", rc, addr.smctp_addr);

                    /* Reply to requester; this macro just clears the TO-bit.
                     * Other addr fields will describe the remote endpoint,
                     * so use those as-is.
                     */
                    addr.smctp_tag = MCTP_TAG_RSP(addr.smctp_tag);

                    rc = sendto(sd, &msg, rc, 0,
                                (struct sockaddr *)&addr, addrlen);
                    if (rc < 0)
                            err(EXIT_FAILURE, "sendto");
            }

            return EXIT_SUCCESS;
    }
```

##### 廣播要求

向廣播 EID 送出要求，並接收（單播）回應。典型的控制協定模式。

```c
    int main() {
            struct sockaddr_mctp txaddr, rxaddr;
            struct timespec start, cur;
            struct pollfd pollfds[1];
            socklen_t addrlen;
            uint8_t buf[2];
            int timeout;

            sd = socket(AF_MCTP, SOCK_DGRAM, 0);

            /* destination address setup */
            txaddr.sa_family = AF_MCTP;
            txaddr.smctp_network = 1; /* specific network required for broadcast */
            txaddr.smctp_addr.s_addr = MCTP_TAG_BCAST; /* broadcast dest */
            txaddr.smctp_type = MCTP_TYPE_CONTROL;
            txaddr.smctp_tag = MCTP_TAG_OWNER;

            buf[0] = MCTP_TYPE_CONTROL;
            buf[1] = 'a';

            /* We're doing a sendto() to a broadcast address here. If we were
             * sending more than one broadcast message, we'd be better off
             * doing connect(); sendto();, in order to retain the tag
             * reservation across all transmitted messages. However, since this
             * is a single transmit, that makes no difference in this
             * particular case.
             */
            rc = sendto(sd, buf, 2, 0, (struct sockaddr *)&txaddr,
                            sizeof(txaddr));
            if (rc < 0)
                    err(EXIT_FAILURE, "sendto");

            /* Set up poll behaviour, and record our starting time for
             * reply timeouts */
            pollfds[0].fd = sd;
            pollfds[0].events = POLLIN;
            clock_gettime(CLOCK_MONOTONIC, &start);

            for (;;) {
                    /* Calculate the amount of time left for replies */
                    clock_gettime(CLOCK_MONOTONIC, &cur);
                    timeout = calculate_timeout(&start, &cur, 1000);

                    rc = poll(pollfds, 1, timeout)
                    if (rc < 0)
                        err(EXIT_FAILURE, "poll");

                    /* timeout receiving a reply? */
                    if (rc == 0)
                        break;

                    /* sanity check that we have a message to receive */
                    if (!(pollfds[0].revents & POLLIN))
                        break;

                    addrlen = sizeof(rxaddr);

                    rc = recvfrom(sd, &buf, 2, 0, (struct sockaddr *)&rxaddr,
                            &addrlen);
                    if (rc < 0)
                            err(EXIT_FAILURE, "recvfrom");

                    assert(addrlen >= sizeof(rxaddr));
                    assert(rxaddr.smctp_family == AF_MCTP);

                    printf("response from EID %d\n", rxaddr.smctp_addr);
            }

            return EXIT_SUCCESS;
    }
```

#### 實作說明

##### 位址

傳送的訊息（透過 `sendto()` 及相關系統呼叫）透過 `struct sockaddr_mctp` 的
`smctp_network` 與 `smctp_addr` 欄位指定其目的。

`smctp_addr` 欄位直接對應到目的端點的 EID。

`smctp_network` 欄位指定一個本機定義的網路識別碼。為簡化僅定義單一網路的
情況，允許使用特殊值 `MCTP_NET_ANY`。這會讓核心為傳送選取一個特定網路。

這個選取完全由使用者設定；某個特定網路可被定義為系統預設值，在這種情況下，
凡是以 `MCTP_NET_ANY` 作為目的網路的訊息傳送，都會使用該網路。

特別的是，目的 EID 絕不會被用來選取目的網路。

MCTP 回應端應使用傳入要求的 EID 與網路值，來指定任何回應的目的。

##### 橋接／路由

網路與介面結構允許多個介面共用一個共同網路。預設情況下，封包不會在介面之間
轉發。

網路可被設定為「轉發」模式。在此模式下，若封包的目的 EID 非本機，且符合同一
網路中另一個介面的路由，則封包可被轉發。

依 DSP0236 所述，轉發過程中不會進行封包重組。若封包大於目的介面／路由的 MTU，
則該封包會被丟棄。

##### 傳送訊息的 Tag 行為

每當以 tag-擁有者位元設定為 1（在 DSP0236 中稱為「TO」）送出訊息時，核心都
必須分配一個 tag，以在（目的 EID、來源 EID、tag-擁有者、tag）這個元組上唯一
識別回應。tag 值的大小為 3 個位元。

為了允許這一點，在 `smctp_tag` 欄位中設定 `MCTP_TAG_OWNER` 位元的 `sendto()`，
會使核心為來自該特定遠端 EID 的後續回應分配一個唯一的 tag。

當下列任一情況發生時，此分配將逾期失效：

- socket 被關閉
- 有新訊息送往一個新的目的 EID
- 一個實作定義的逾時時間到期

由於「tag 空間」有限，核心可能無法為送出訊息分配一個唯一的 tag。在這種情況下，
`sendto()` 呼叫將失敗並帶有 `EAGAIN` 的 errno。這類似於 UDP 無法為送出訊息
分配本機埠號時的行為。

實作定義的逾時值應選擇能合理涵蓋標準回應逾時的數值。如有必要，可透過
`MCTP_TAG_CONTROL` socket 選項來修改此逾時。

對於預期要與某個特定目的位址進行持續訊息交換的應用程式，它們可使用 `connect()`
呼叫來設定一個持續性的遠端位址。在這種情況下，tag 會在 connect() 期間分配，
並為這個 socket 保留，直到下列任一情況發生：

- socket 被關閉
- 透過另一次 `connect()` 呼叫變更了遠端位址。

特別的是，以不同的位址呼叫 `sendto()` 並不會釋放 tag 保留。

廣播訊息對 tag 保留而言特別沉重。當訊息以 TO=1 且 dest=0xff（廣播 EID）
傳送時，核心必須在整個可能的 EID 範圍上保留該 tag。因此，必須某個 tag 值
目前對所有 EID 都未被使用，才能允許對廣播位址進行 `sendto()`。此外，此保留
在收到回應時並不會被清除，因為一個廣播可能有多個回應。

基於這個原因，想要傳送至廣播位址的應用程式應使用 `connect()` 系統呼叫來保留
一個 tag，並保證它在未來的訊息傳送中可用。請注意，這將會使該 tag 值從 _任何
其他 EID_ 的使用中移除。應盡量避免傳送至廣播位址；我們預期很少有應用程式需要
此功能。

##### MCTP 控制協定實作

除「Resolve endpoint EID」訊息外，MCTP 控制協定實作會以使用者空間程序
`mctpd` 的形式存在。此程序負責回應傳入的控制協定訊息、任何動態 EID 分配
（給匯流排擁有者裝置），以及維護 MCTP 路由表（給橋接裝置）。

此程序會建立一個繫結至類型 `MCTP_TYPE_CONTROL` 的 socket，並啟用
`MCTP_ADDR_EXT` socket 選項，以便在傳入的控制協定要求上存取實體位址資料。
它會透過 netlink 介面與核心的路由表互動——與為
[工具與設定介面](#工具與設定介面)所實作的相同。

#### 鄰居與路由實作

MCTP 基礎設施的封包傳送行為依賴單一路由表，來查詢路由與鄰居兩者資訊。此表
中的項目格式如下：

| EID 範圍 | 介面 | 實體位址 | metric | MTU | 旗標 | 逾期 |
| --------- | --------- | ---------------- | ------ | --- | ----- | ------ |

此表可從兩個來源更新：

- 從使用者空間，透過 netlink 介面（請見
  [工具與設定介面](#工具與設定介面)一節）。

- 直接在核心內，當基本的鄰居資訊被發現時。核心發起的路由會在旗標欄位中如此
  標記，並有一個由逾期欄位所示的最大有效存續期。

核心發現的路由資訊可來自兩個來源：

- 透過收到的封包所發現的實體對 EID 映射

- 明確的端點實體位址解析要求

當某封包將被傳送至一個在路由表中沒有項目的 EID 時，核心可嘗試使用 MCTP 控制
協定的 Resolve Endpoint ID 指令（DSP0236 第 12.9 節）來解析該端點的實體位址。
回應訊息將被用來向路由表新增一筆核心發起的路由。

這是 MCTP 控制協定訊息唯一的核心內部用途。

### 工具與設定介面

將開發一個小型工具來控制核心 MCTP 堆疊的狀態。它在設計上將類似 'iproute2'
工具，後者為 IPv4 與 IPv6 協定執行類似的功能。

該工具將以 `mctp` 呼叫，並提供子命令來管理核心堆疊的各個不同層面。

#### `mctp link`：管理介面

```sh
    mctp link set <link> <up|down>
    mctp link set <link> network <network-id>
    mctp link set <link> mtu <mtu>
    mctp link set <link> bus-owner <hwaddr>
```

#### `mctp network`：管理網路

```sh
    mctp network create <network-id>
    mctp network set <network-id> forwarding <on|off>
    mctp network set <network-id> default [<true|false>]
    mctp network delete <network-id>
```

#### `mctp address`：管理本機 EID 分配

```sh
    mctp address add <eid> dev <link>
    mctp address del <eid> dev <link>
```

#### `mctp route`：管理路由表

```sh
    mctp route add net <network-id> eid <eid|eid-range> via <link> [hwaddr <addr>] [mtu <mtu>] [metric <metric>]
    mctp route del net <network-id> eid <eid|eid-range> via <link> [hwaddr <addr>] [mtu <mtu>] [metric <metric>]
    mctp route show [net <network-id>]
```

#### `mctp stat`：查詢 socket 狀態

```sh
    mctp stat
```

將定義一組 netlink 訊息格式來支援這些控制功能。

## 設計重點與已考慮的替代方案

### 在傳送／接收緩衝區中省略訊息類型位元組

本設計規定：在傳送系統呼叫中傳給核心、以及在接收系統呼叫中由核心傳回的
訊息緩衝區，將不會在緩衝區中納入訊息類型位元組。這與 DSP0236 中 MCTP 訊息
主體（message body）的組成略有不同，但避免了在兩個不同地方重複同一個值。

我們需要 `smctp_type` 欄位以便 `bind()`，因此位址表示法勝出。

### MCTP 訊息類型的指定：使用 `sockaddr_mctp.smctp_type` 而非 protocol

本設計規定訊息類型透過 `struct sockaddr_mctp` 的 `smctp_type` 欄位傳遞。
另一種替代方案是透過 `socket()` 系統呼叫的 `protocol` 引數來傳遞：

```c
    int socket(int domain /* = AF_MCTP */, int type /* = SOCK_DGRAM */, int protocol);
```

選擇 `smctp_type` 實作，是因為它更貼合訊息類型的「位址」模型；socket 繫結至
一個傳入的訊息類型，類似 IP 協定將 UDP socket 繫結至本機埠號的模型。

沒有任何核心行為取決於特定類型（尤其考量上述設計選擇），因此在此使用
protocol 引數並不合適。

未來那些執行協定專屬訊息處理、因而會改變傳送／接收緩衝區格式的增補，
可以使用一個新的 protocol 引數。

### 網路以索引而非 UUID 參照

本設計提議以整數索引來參照網路。MCTP 標準確實可选择性地為網路關聯一個
RFC4122 UUID；在我們傳遞網路識別碼的地方，使用這個 UUID 是可行的。

此做法不會在核心內納入網路 UUID 的知識。鑑於 MCTP 控制協定中的 Get Network ID
訊息完全由使用者空間實作，它無需知道網路 UUID，而要求網路參照（例如
`struct sockaddr_mctp` 的 `smctp_network` 欄位，型別為 `uuid_t`）會讓分配變得更複雜。

取而代之的是使用整數索引，方式類似於網路堆疊中其他地方用來參照
`struct netdevice` 的整數索引。

### Tag 行為的替代方案

我們為 tag 處理行為考慮了 _好幾種_ 不同的設計。以下簡要概述其中較可行的幾個，
以及它們被否決的原因：

#### 每個 socket 在建立時分配一個唯一的 tag 值

我們可以在建立時為每個 socket 分配一個 tag，並在需要 tag 時使用該值。然而，
這樣做：

- 會在非 tag-擁有者的 socket（亦即以 TO=0 送出的 socket——回應端）上浪費一個
  tag；以及

- 使我們受限於每個網路最多 8 個 socket。

#### Tag 僅用於訊息封包化／重組

另一種替代方案是完全將 tag 分配與 socket 脫鉤；僅為封包化一封訊息並送出那些
封包這個（短暫的）任務分配一個 tag。tag 會在最後一個封包送出後被釋放。

然而，這消除了將回應與正確 socket 關聯起來的任何機制，而那正是 DSP0236 中
TO 位元的用途。為了讓送出的應用程式能收到回應，我們要麼需要：

- 將系統限制為每種訊息類型只有一個 socket（例如，這就排除了同時執行相同類型
  的請求端與回應端）；或

- 將特定訊息類型的所有傳入訊息都轉給所有監聽該類型的 socket，使得竊聽其他
  應用程式的 MCTP 資料變得易如反掌

#### 為一組要求／回應分配一個 tag

另一種替代方案是在每個送出的 TO=1 訊息上分配一個 tag，然後在觀察到對該 tag
（TO=0）的傳入回應之後，釋放該分配。

然而，存在著要求與回應並非 1:1 對應的 MCTP 協定——對給定的要求訊息，可能有多個
回應是有效的。例如，作為對某要求的回應，NVMe-MI 實作可能會在最終回應之前先送出一
個處理中（in-progress）的回覆。在這種情況下，我們會在收到第一個回應後釋放 tag，
接著便無從將第二則訊息與該 socket 關聯起來。

廣播 MCTP 要求訊息可能有多個端點的多個回應，這意味著我們無法在第一個回應時
就釋放 tag 分配。
