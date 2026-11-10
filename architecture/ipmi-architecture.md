# IPMI 架構

IPMI 由命令（commands）與通道（channels）組成。命令由提供者（providers）提供，
通道則是稱為橋接器（bridges）的程序。命令會經由橋接器從外部來源（系統管理員
使用者）接收，再經由主要的 IPMI 主控程序路由到各提供者的命令處理程式。

## 高階概觀

IPMI 的重心全在命令與回應。通道提供了一種傳輸資料的機制，每個通道都有略微
不同的協定與傳輸層，但最終而言，最上層的資料就是一筆原始 IPMI 命令，包含
NetFn/LUN、Command 以及選用的資料。每筆回應同樣由 Completion Code 與選用資料
組成。因此第一步就是把通道與 IPMI 佇列拆開。

```ascii
                                                     /------------------\
    /----------------------------\                   |                  |
    |      KCS/BT - Host         | <-All IPMI cmds-> |                  |
    |                            |                   |                  |
    \----------------------------/                   |   IPMI Daemon    |
                                                     |    (ipmid)       |
                                                     |                  |
   /-----------------------------\                   |                  |
   |            LAN - RMCP+      |                   |                  |
   | /--------------------------\|                   |                  |
   | |*Process the Session and  || <-All IPMI cmds-> |                  |
   | | SOL commands.            ||  except session   |                  |
   | |*Create Session Objs      ||  and SOL cmds     |                  |
   | \--------------------------/|                   |                  |
   \-----------------------------/                   \------------------/
                :                                             ^     ^
                :                                             |     |
                :                                             |     |
     /-------------------------\                              |     |
     | Active session/SOL Objs | <---------Query the session-/      |
     | - Properties            |           and SOL data via Dbus    |
     \-------------------------/                                    |
                                                                    V
                                                     ---------------------\
                                                     | D-Bus services     |
                                                     | ------------------ |
                                                     | Do work on behalf  |
                                                     | of IPMI commands   |
                                                     ---------------------/
```

傳給 IPMI 主控程序（ipmid）以及從其中傳出的 IPMI 訊息，大致上相當於 ipmitool
的「raw」命令，只是多了一些關於該訊息的資訊。

```less
Message Data:
  byte:LUN - LUN from netFn/LUN pair (0-3, as per the IPMI spec)
  byte:netFn - netFn from netFn/LUN pair (as per the IPMI spec)
  byte:cmd - IPMI command ID (as per the IPMI spec)
  array<byte>:data - optional command data (as per the IPMI spec)
  dict<string:variant>:options - optional additional meta-data
    "userId":int - IPMI user ID of caller (0 for session-less channels)
    "privilege": enum:privilege - ADMIN, USER, OPERATOR, CALLBACK;
      must be less than or equal to the privilege of the user and less
      than or equal to the max privilege of this channel
```

```less
Response Data:
  byte:CC - IPMI completion code
  array<byte>:data - optional response data
```

通道橋接器在收到新的 IPMI 命令時，會從各種傳輸格式（RMCP+、IPMB、KCS 等）中
擷取出必要的欄位。對於以工作階段為基礎的通道（RMCP+），橋接器負責使用憑證
建立工作階段，並決定此工作階段可用的最大權限。接著橋接器會取得擷取出的命令、
資料，以及可能的使用者與權限資訊，將它們編碼成一個 D-Bus 方法呼叫，傳送給
IPMI 主控程序 ipmid。主控程序接收該訊息後，會嘗試在其處理程式表中找到適當的
處理程式。若找到處理程式，它會嘗試擷取該處理程式所需的參數並傳過去。處理程式
會回傳一個回應參數的 tuple，這些參數會被重新打包成 D-Bus 回應訊息，並傳回給
呼叫端通道的橋接器。橋接器接著會把回應重新封裝成其傳輸協定並送出。

接下來的部分，是要提供一個更高階、強型別、現代化 C++ 的機制來註冊處理程式。
每個處理程式會明確指出該命令需要哪些引數，以及回應中會回傳哪些型別。如此
一來，ipmid 佇列就能以安全的方式解包請求並打包回應。由於處理程式的打包與
解包程式碼是以樣板（template）寫成，因此大部分都寫在標頭檔中。

## 詳細資訊與實作

對於無工作階段的通道（如 BT、KCS 與 IPMB），唯一的權限檢查是確認請求的權限
小於或等於通道的最大權限。若通道設有工作階段且會對使用者驗證，則權限必須
小於或等於通道的最大權限，同時也要小於或等於使用者的最大權限。

Ipmid 取得 LUN/netFN/Cmd 這個 tuple，並查尋對應的處理程式函式。如果請求的權限
小於或等於該已註冊命令所需的權限，請求才可以繼續進行。若上述任何一項檢查失敗，
ipmid 會以 _Insufficient Privilege_ 回傳。

到了這一步，IPMI 命令會經過過濾器掛鉤（filter hooks）。預設的掛鉤是 ACCEPT，
此時命令直接進入執行階段。但 OEM 與提供者可以註冊掛鉤，最終用以阻擋 IPMI 命令
執行，情況類似 IPMI 2.0 規格中的 Firmware Firewall。掛鉤會接收 IPMI 呼叫的
脈絡（context）以及呼叫的原始內容，並有機會回傳任何有效的 IPMI completion
code。任何非零的 completion code 都會阻止命令執行，並會回傳給呼叫端。

網路通道橋接器（netipmid）每個介面執行一個程序，負責處理工作階段（RMCP+）與
SOL 命令以及這些命令的回應。Get/Set SOL 設定、Get session info、close session
等命令也可以經由 KCS/BT 介面請求，因此 ipmid 主控程序同樣需要關於工作階段與
SOL 的詳細資訊。為了在 ipmid 與 netipmid 主控程序之間維持同步，工作階段與 SOL
會暴露在 D-Bus 上，ipmid 可以查詢它們，並回應經由主機介面（KCS/BT）發出的命令。

下一個階段是參數解包與驗證。這是透過在註冊處理程式時，由編譯器以可變引數樣板
（variadic templates）產生的程式碼來完成。註冊函式是一個樣板函式，允許傳入任何
型別的處理程式，如此便能擷取並解包處理程式的型別。

核心 IPMI 函式庫（從高階角度來看）是這樣實作的：

```cpp
template <typename Handler>
class IpmiHandler
{
  public:
    explicit IpmiHandler(Handler&& handler) :
        handler_(std::forward<Handler>(handler))
    {
    }
    message::Response::ptr call(message::Request::ptr request)
    {
        message::Response::ptr response = request->makeResponse();
        // < code to deduce UnpackArgsType and ResultType from Handler >
        UnpackArgsType unpackArgs;
        ipmi::Cc unpackError = request->unpack(unpackArgs);
        if (unpackError != ipmi::ccSuccess)
        {
            response->cc = unpackError;
            return response;
        }
        ResultType result = std::apply(handler_, unpackArgs);
        response->cc = std::get<0>(result);
        auto payload = std::get<1>(result);
        response->pack(*payload);
        return response;
    }
  private:
    Handler handler_;
};
 ...
 namespace ipmi {
   constexpr NetFn netFnApp = 0x06;
 namespace app {
   constexpr Cmd cmdSetUserAccessCommand = 0x43;
 }
  class Context {
     NetFn netFn;
     Cmd cmd;
     int channel;
     int userId;
     Privilege priv;
     boost::asio::yield_context* yield; // for async operations
   };
 }
```

而某些 IPMI 處理程式看起來會像這樣：

```cpp
ipmi::RspType<> setUserAccess(
    std::shared_ptr<ipmi::Context> context,
    uint4_t channelNumber,
    bool ipmiEnable,
    bool linkAuth,
    bool callbackRestricted,
    bool changeBit,
    uint6_t userId,
    uint2_t reserved1,
    uint4_t privLimit,
    uint4_t reserved2,
    std::optional<uint4_t> userSessionLimit,
    std::optional<uint4_t> reserved3) {
  ...
  return ipmi::ResponseSuccess();
}

ipmi::RspType<uint8_t, // max user IDs
              uint6_t, // count of enabled user IDs
              uint2_t, // user ID enable status
              uint8_t, // count of fixed user IDs
              uint4_t  // user privilege for given channel
              bool,    // ipmi messaging enabled
              bool,    // link authentication enabled
              bool,    // callback access
              bool,    // reserved bit
              >
    getUserAccess(std::shared_ptr<ipmi::Context> context, uint8_t channelNumber,
                  uint8_t userId)
{
    if (<some error condition>)
    {
        return ipmi::response(ipmi::ccParmOutOfRange);
    }
    // code to get
    const auto& [usersMax, status, usersEnabled, usersFixed, access, priv] =
        getSdBus()->yield_method_call(*context->yield, ...);
    return ipmi::responseSuccess(usersMax, usersEnabled, status, usersFixed,
                                 priv, access.messaging, access.linkAuth,
                                 access.callback, false);
}

void providerInitFn()
{
    ipmi::registerHandler(ipmi::prioOpenBmcBase, ipmi::netFnApp,
                          ipmi::app::cmdSetUserAccessCommand,
                          ipmi::Privilege::Admin,
                          setUserAccess);
    ipmi::registerHandler(ipmi::prioOpenBmcBase, ipmi::netFnApp,
                          ipmi::app::cmdGetUserAccessCommand,
                          ipmi::Privilege::Operator,
                          getUserAccess);
}
```

Ipmid 的提供者全部都是以 boost::asio::coroutines 執行。這代表它們可以利用任何
boost::asio 的非同步方法呼叫，寫起來看起來像同步程式碼，但實際上會經由
yield_context 物件將處理器的控制權讓給執行迴圈，以非同步方式執行。使用
yield_context 物件的方式是把它當作「callback」傳入，這樣就會讓呼叫中的
coroutine 讓出處理器，直到其 IO 準備好為止，屆時它會「返回」，情況就像同步呼叫
一樣。

理想上，所有處理程式都應該透過 boost::asio::yield_context 善用 ipmid 的非同步
能力。這代表佇列可以同時有多個進行中的呼叫正在等待另一個 D-Bus 呼叫返回。有了
非同步呼叫，這就不會阻擋佇列其餘部分的運作。所有提供者都可以透過 getSdBus()
函式取得的 sdbusplus::asio::connection，提供了 yield_method_call()，這是一種
非同步的 D-Bus 呼叫機制，「看起來」像同步呼叫。有一點很重要：任何非同步處理程式
所使用的全域資料都必須加以保護，就如同該處理程式是多執行緒一樣。由於 IPMI 處理
程式中使用的許多資源實際上都是 D-Bus 物件，因為 D-Bus 呼叫會帶來序列化，所以
這不太可能是常見的問題。

使用樣板，可以擷取處理程式的回傳型別與引數型別，並用它來從傳入的請求中解包
（並驗證）引數，然後把結果重新打包成一個位元組向量傳回給呼叫端。反序列化器會
追蹤它已解包的位元數，然後與該方法所要求的總位元數做比較。在這個範例中，我們
假設非完整位元組的整數在訊息中是以最高有效位元先（most-significant-bit first）
的順序打包的位元（與規格描述它們的順序相同）。搭配 C++17 的 std::optional，
可以輕鬆地使用選用引數。

支援的某些型別如下：

- 標準整數型別（`int8_t`、`uint16_t`、`uint32_t`、`int`、`long` 等）
- `bool`（擷取單一一位元，同 `uint1_t`）
- 多精度整數（`uint<N>`）
- `std::bitset<N>`
- `std::optional<T>` - 若有足夠的剩餘位元/位元組則擷取 T
- `std::array<T, N>` - 若可能則擷取 N 個 T 元素
- `std::vector<T>` - 從剩餘的位元組中擷取 T 元素
- 任何額外的型別都可以透過製作 pack/unpack 樣板來建立

對於部分位元組型別，會先擷取下一個完整位元組的最低有效位元（least-significant
bits）。雖然這與 IPMI 規格的書寫順序相反，但它讓程式碼更為簡潔。多位元組欄位
以 LSByte 先（小端序，little-endian）的方式擷取，以符合 IPMI 規格。

在回傳部分位元組型別時，它們同樣以最低有效位元先的順序打包進回應中。隨著每個
位元組填滿，完整的位元組會被推入位元組串流中。關於打包與解包如何使用的範例，
請參見 test/message/pack.cpp 與 test/message/unpack.cpp 中的單元測試案例。

作為範例，以下是 bitset 的解包方式：

```cpp
/** @brief Specialization of UnpackSingle for std::bitset<N>
 */
template <size_t N>
struct UnpackSingle<std::bitset<N>>
{
    static int op(Payload& p, std::bitset<N>& t)
    {
        static_assert(N <= (details::bitStreamSize - CHAR_BIT));
        size_t count = N;
        // acquire enough bits in the stream to fulfill the Payload
        if (p.fillBits(count))
        {
            return -1;
        }
        fixed_uint_t<details::bitStreamSize> bitmask = ((1 << count) - 1);
        t |= (p.bitStream & bitmask).convert_to<unsigned long long>();
        p.bitStream >>= count;
        p.bitCount -= count;
        return 0;
    }
};
```

如果某個處理程式需要解包可變的載具（payload），可以請求 Payload 參數。當存在
Payload 參數時，任何剩餘、尚未解包的位元組都可以供檢查。使用之前用來解包其他
參數的相同解包呼叫，就可以手動解包剩餘的參數。這對於像 Set LAN Configuration
Parameters 這類多功能命令很有幫助，它的前兩個位元組是固定的，但剩餘的 3:N
個位元組會根據所選的參數而變動。
