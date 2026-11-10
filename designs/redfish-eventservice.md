# Redfish EventService 設計

作者：AppaRao Puli

其他貢獻者：無

建立日期：2019-07-24

## 問題描述

OpenBMC 中的 Redfish 目前僅支援針對要求資料的 client 回應要求。目前沒有任何服務可
將事件、錯誤或狀態更新等非同步訊息傳送給 client。

本設計文件的目的，是在提供 redfish event service 的資源概觀，以及其在 OpenBMC 中的
實作細節。

## 背景與參考資料

- [Redfish 規格](https://www.dmtf.org/sites/default/files/standards/documents/DSP0266_1.7.0.pdf)
- [DMTF Redfish 教學 - Events](https://www.dmtf.org/sites/default/files/Redfish_School-Events_2018.2.pdf)
- [Redfish 事件記錄](https://github.com/openbmc/docs/blob/master/redfish-logging-in-bmcweb.md)
- [Telemetry 服務](https://gerrit.openbmc.org/#/c/openbmc/docs/+/24357/)
- [Beast 非同步 client](https://www.boost.org/doc/libs/develop/libs/beast/example/http/client/async/)
- [Beast 非同步 ssl client](https://www.boost.org/doc/libs/develop/libs/beast/example/http/client/async-ssl/)
- [EventService schema](https://redfish.dmtf.org/schemas/v1/EventService_v1.xml)
- [EventDestinationCollection schema](https://redfish.dmtf.org/schemas/v1/EventDestinationCollection_v1.xml)
- [EventDestination schema](https://redfish.dmtf.org/schemas/v1/EventDestination_v1.xml)
- [升級連線以支援 SSE](https://gerrit.openbmc.org/#/c/openbmc/bmcweb/+/13948/)

## 需求

高階需求：

- BMC 應支援透過網路使用 redfish(http/https) 通訊協定，對已訂閱的外部 client 傳送
  非同步事件通知，取代一般的查詢（querying）機制。
- BMC 應提供機制讓使用者能以特定篩選條件設定事件服務，並據此將事件通知傳送至所有
  已註冊的 client。
- BMC 應在重新啟動後仍保留事件服務的設定，並在使用出廠預設值時將其重置。
- BMC 應具備可停用 EventService 的選項。
- BMC 的 Redfish 介面應支援 EventService、EventDestination 與
  EventDestinationCollection schema。
- BMC 可提供送出測試事件(SubmitTestEvent) 的功能。
- 必須僅允許具備 'ConfigureManager' 權限的使用者建立、刪除或更新事件服務設定與
  訂閱，並限制其他使用者。必須允許具備 'Login' 權限的使用者檢視 EventService 設定
  與訂閱詳細資訊。
- EventService 僅限 Redfish 介面使用。也就是說，其他服務皆不得新增／修改／刪除
  設定或訂閱。
- client 或服務皆可隨時終止事件串流。當遞送錯誤次數超過預先設定的門檻時，服務
  可終止訂閱。
- 對於 Push 樣式的事件訂閱建立，BMC 應以 HTTP 狀態碼 201 回應成功的訂閱，並將
  HTTP Location 標頭設為新訂閱資源的位址。訂閱為持續性的，且應在事件服務重新
  啟動後仍然保留。
- 對於 Push 樣式的事件訂閱建立，BMC 應以下列 http 狀態碼回應 client：
  - 400：如果 body 中的參數不支援。
  - 404：如果要求 body 同時包含 RegistryPrefixes 與 MessageIds。
- SSE：BMC 應在 EventService schema 中 "ServerSentEventUri" 所指定的 URI 上回應
  GET 方法，以 201(created) 連同訂閱資訊回應。BMC 應開啟新的 https 連線，並應在
  訂閱有效期間保持連線存活。如果篩選參數不支援，BMC 應回應 https 碼 400；如果篩選
  參數包含無效資料，則回應 404。此外，SSE 訂閱資料無法在服務重設／重新啟動後持續
  保存。
- client 應透過對訂閱資源的 URI 傳送 DELETE 訊息來終止訂閱。
- BMC 可透過傳送一則特殊的 "subscription terminated" 事件作為最後一則訊息來終止
  訂閱。之後對相關訂閱資源的要求將回應 HTTP 狀態碼 404。
- BMC 應接受來自 client 的 "Last-Event-ID" 標頭，以允許 client 在連線中斷時重新
  開始事件／指標回報串流。
- 為避免連線資源耗盡，BMC 應實作最多 20 個連線（最多 10 個 SSE 連線）的限制。
  BMC 應以 503 Service Unavailable 伺服器錯誤回應碼回應，表示伺服器尚未準備好
  處理該要求。
- BMC 應針對訂閱的新增、刪除與修改記錄一筆 redfish 事件訊息。
- 服務不得傳送大小超過 1 MB 的 "push" 事件 payload。

## 建議設計

Redfish event service 提供了一種訂閱特定種類 Events、MetricReport 的方法，並能在
訂閱類型的 event 或回報發生時，非同步地將它們傳送至 client。

OpenBMC EventService 支援兩種事件樣式。

1. Push 樣式事件 (https)：當服務偵測到需要傳送事件時，會使用 HTTP POST 將事件訊息
   推播給 client。client 可以透過在 EventService 中建立訂閱項目來啟用事件接收。

2. Server-Sent Events (SSE)：client 透過對 Event Service 中 "ServerSentEventUri"
   所指定的 URI 執行 GET，來向服務開啟 SSE 連線。

本文件主要著重在 OpenBMC 的實作觀點。EventService 的更多相關細節可於
["Redfish Specification"](https://www.dmtf.org/sites/default/files/standards/documents/DSP0266_1.7.0.pdf)
的下列章節找到：

- Eventing（Section 12.1）
- Server Sent-Events（Section 12.5）
- Security（Section 12 與 13.2.8）

```text
+------------------------------------------------------------------------------------+
|                                 CLIENT                                             |
|                             EVENT LISTENER                                         |
|                                                                                    |
+-----------------------------------------------------------------^---------^--------+
CONFIGURATION|         |SSE(GET)        NETWORK        PUSH STYLE |         |SSE
SUBSCRIPTION |         |                                 EVENTS   |  POST   |
+------------------------------------------------------------------------------------+
|            |         |                                          |         |        |
|   +--------+---------+---+  +---------------------------------------------------+  |
|   |     REDFISH          |  |           EVENT SERVICE MANAGER   |         |     |  |
|   |     SCHEMA           |  |   +-------------------------------+---------+--+  |  |
|   |    RESOURCES         |  |   |          EVENT SERVICE                     |  |  |
|   |                      |  |   |            SENDER                          |  |  |
|   |                      |  |   +------^----------------------------+--------+  |  |
|   |  +-------------+     |  |          |                            |           |  |
|   |  |EVENT SERVICE|     |  |   +------+--------+            +------+--------+  |  |
|   |  +-------------+     |  |   | METRICSREPORT |            |  EVENT LOG    |  |  |
|   |                      |  |   | FORMATTER     |            |  FORMATTER    |  |  |
|   |                      |  |   +------^--------+            +------^--------+  |  |
|   |     +-------------+  |  |          |                            |           |  |
|   |   + |SUBSCRIPTIONS+--------------------+------------------------------+     |  |
|   | + | +-------------+  |  |          |   |                        |     |     |  |
|   | | +--------------+   |  |   +------+---+----+            +------+-----+--+  |  |
|   | +--------------|     |  |   | METRCISREPORT |            |  EVENT LOG    |  |  |
|   |                      |  |   | MONITOR/AGENT |            |  MONITOR      |  |  |
|   |                      |  |   +---------------+            +---------------+  |  |
|   +----+-----------------+  +---------------------------------------------------+  |
|        |                              |                             |              |
+------------------------------------------------------------------------------------+
         |                              |                             |
         |                              |                             |INOTIFY
         |                              |                             |
+--------+----------+         +---------+----------+        +-----------------------+
|                   |         |                    |        |  +-----------------+  |
|    REDFISH        |         |     TELEMETRY      |        |  |     REDFISH     |  |
|  EVENT SERVICE    |         |      SERVICE       |        |  |    EVENT LOGS   |  |
|    CONFIG         |         |                    |        |  +-----------------+  |
| PERSISTENT STORE  |         |                    |        |         |RSYSLOG      |
|                   |         |                    |        |  +-----------------+  |
|                   |         |                    |        |  |     JOURNAL     |  |
|                   |         |                    |        |  |  CONTROL LOGS   |  |
|                   |         |                    |        |  +-----------------+  |
+-------------------+         +--------------------+        +-----------------------+

```

Push 樣式事件(HTTPS) 流程圖：

```text
+-------------+   +------------+               +-------------------------------------+ +-----------------+
|             |   |  CLIENT    |               |              BMCWEB                 | |   EVENT LOG/    |
|   CLIENT    |   | LISTENERS  |               |RESOURCE SENDER  FORMATTER  MONITOR  | |TELEMETRY SERVICE|
+-----+-------+   +-----+------+               +--+--------+--------+---------+------+ +-----+-----------+
      |                 |                         |        |        |         |              |
+----------------------------------------------------------------------------------------------------
|SUBSCRIBE FOR EVENT|   |                         |        |        |         |              |
+-------------------+   |                         |        |        |         |              |
      |                 |                         |        |        |         |              |
      |                 |                         |        |        |         |              |
      |                 |    EVENT SUBSCRIPTION   |        |        |         |              |
      +------------------------------------------>+        |        |         |              |
      |                 |     PUSH STYLE EVENT    |        |        |         |              |
      |                 |                         |-------------------------> |              |
      |                 |                         |    SEND SUBSCRIPTION INFO |              |
      |                 |                         |----------------------------------------->|
      |                 |                         |    LOG REDFISH EVENT LOG  |              |
      |                 |                         |        |        |         |              |
+----------------------------------------------------------------------------------------------------
|SEND EVENTS/METRICS WHEN GENERATED |             |        |        |         |              |
+-----------------------------------+             |        |        |         |              |
      |                 |                         |        |        |         |              |
      |                 |                         |        |        |         |              +----+
      |                 |                         |        |        |         |              |LOOP|
      |                 |                         |        |        |         |              +<---+
      |                 |                         |        |        |         +<------------>+
      |                 |                         |        |        |         | NOTIFY EVENT/TELEMETRY
      |                 |                         |        |        |         |       DATA   |
      |                 |                         |        |        |         +-------------->
      |                 |                         |        |        |         | READ DATA    |
      |                 |                         |        |        |         +<-------------+
      |       +-----+--------------------------------------------------------------------------+
      |       |LOOP |   |                         |        |        |         |              | |
      |       +-----+   |                         |        |        |         |              | |
      |       |         |                         |        |        |         +-----------+  | |
      |       |         |                         |        |        |         | MATCH     |  | |
      |       |         |                         |        |        |         |SUBSCRIPTION  | |
      |       |         |                         |        |        |         |WITH DATA  |  | |
      |       |         |                         |        |        |         +-----------+  | |
      |       |         |                         |        |        |         |              | |
      |       |         |                         |        |        +<--------+              | |
      |       |         |                         |        |        | SUBSCRIPTION           | |
      |       |         |                         |        |        |  AND  DATA             | |
      |       |         |                         |        |        |         |              | |
      |       |         |                         |        +<-------+         |              | |
      |       |         |                         |        |SUBSCRIPTION AND  |              | |
      |       |         |                         |        |FORMATTED DATA    |              | |
      |       |         |                         |        |        |         |              | |
      |       |         +--------------------------------------------------------------------+-+
      |       |         |FIRST TIME, CREATE CONNECTION |   |        |         |              | |
      |       |         +------------------------------+   |        |         |              | |
      |       |         <---------------------------------->        |         |              | |
      |       |         |  ESTABLISH HTTP/HTTPS   |        +-----+  |         |              | |
      |       |         |     CONNECTION          |        |STORE|  |         |              | |
      |       |         |                         |        |CONN-SUBS         |              | |
      |       |         |                         |        |MAP  |  |         |              | |
      |       |         |                         |        |<----+  |         |              | |
      |       |         |                         |        |        |         |              | |
      |       |         |                         |        +----+|  |         |              | |
      |       |         |                         |        |PUSH QUEUE        |              | |
      |       |         |                         |        |INCOMING|         |              | |
      |       |         |                         |        |DATA |  |         |              | |
      |       |         |                         |        |<---+|  |         |              | |
      |       |         |                         |        |        |         |              | |
      |       | +------+-----------------------------------------+  |         |              | |
      |       | |RETRY ||<---------------------------------+     |  |         |              | |
      |       | |LOOP  ||   SEND FORMATTED DATA   |        |     |  |         |              | |
      |       | |-+----+|                         |        |     |  |         |              | |
      |       | | +-----------------------------------------------------------------------------
      |       | | |SEND FAILED |                  |        |     |  |         |              | |
      |       | | +------------+                  |        |     |  |         |              | |
      |       | | |     |                         |        |     |  |         |              | |
      |       | | |     <---------------------------------->     |  |         |              | |
      |       | | |     |  ESTABLISH HTTP/HTTPS   |        +----+|  |         |              | |
      |       | | |     |     CONNECTION          |        |REOPEN  |         |              | |
      |       | | |     |                         |        |CONN&|  |         |              | |
      |       | | |     |                         |        |UPDATE STORE      |              | |
      |       | | |     |                         |        |<---+|  |         |              | |
      |       | | +------------REPEAT SEND LOOP------------------|  |         |              | |
      |       | | +-----------------------------------------------------------------------------
      |       | | |SEND SUCCESS |                 |        |     |  |         |              | |
      |       | | +-------------+                 |        |     |  |         |              | |
      |       | |       |                         |        |---+ |  |         |              | |
      |       | |       |                         |        |POP| |  |         |              | |
      |       | |       |                         |        |QUEUE|  |         |              | |
      |       | |       |                         |        |<--+ |  |         |              | |
      |       | |       |                         |        |     |  |         |              | |
      |       | |       |                         |        +----+|  |         |              | |
      |       | |       |                         |        |SEND NEXT         |              | |
      |       | |       |                         |        |QUEUEED |         |              | |
      |       | |       |                         |        |DATA||  |         |              | |
      |       | |       |                         |        |<---+|  |         |              | |
      |       | |       |                         |        |     |  |         |              | |
      |       | | +-----------------------------------------------------------------------------
      |       | | |REACHED THRESHOLD LIMIT |      |        |     |  |         |              | |
      |       | | +------------------------+      |        |     |  |         |              | |
      |       | |       |                         |        |----+|  |         |              | |
      |       | |       |                         |        |DELETE  |         |              | |
      |       | |       |                         |        |STORE|  |         |              | |
      |       | |       |                         |        |&QUEUE  |         |              | |
      |       | |       |                         |        |<--+ |  |         |              | |
      |       | |       |                         |<-------|     |  |         |              | |
      |       | |       |                         |REMOVE RESOURCE  |         |              | |
      |       | |       |                         +------+ |     |  |         |              | |
      |       | |       |                         |REMOVE| |     |  |         |              | |
      |       | |       |                         |SUBSCRIPTION  |  |         |              | |
      |       | |       |                         |& CONNECTION  |  |         |              | |
      |       | |       |                         |<-----+ |     |  |         |              | |
      |       | |       |                         |-------------------------->|              | |
      |       | |       |                         |  UPDATE SUBSCRIPTION      |              | |
      |       | |       |                         |----------------------------------------->| |
      |       | |       |                         |    LOG REDFISH EVENT LOG  |              | |
      |       | |       |                         |        |     |  |         |              | |
      |       | +-------+----------------------------------------+  |         |              | |
      |       |         |                         |        |        |         |              | |
      |       +--------------------------------------------------------------------------------+
      |                 |                         |        |        |         |              |
+--------------------------------------------------------------------------------------------------
|DELETE EVENT SUBSCRIPTIONS |                     |        |        |         |              |
+---------------------------+                     |        |        |         |              |
      |                 |                         |        |        |         |              |
      +-----------------+------------------------>+        |        |         |              |
      |        DELETE SUBSCRIPTION                |        |        |         |              |
      |                 |                         +------+ |        |         |              |
      |                 |                         |REMOVE| |        |         |              |
      |                 |                         |SUBSCRIPTION     |         |              |
      |                 |                         <------+ |        |         |              |
      |                 +<------------------------|        |        |         |              |
      |                 |    CLOSE CONNECTION     |        |        |         |              |
      |                 |                         |--------------------------->              |
      |                 |                         |    UPDATE SUBSCRIPTION    |              |
      |                 |                         |----------------------------------------->|
      |                 |                         |    LOG REDFISH EVENT LOG  |              |
      |                 |                         |        |        |         |              |

```

SSE 流程圖：

```text
+-------------+   +------------+               +-------------------------------------+ +-----------------+
|             |   |  CLIENT    |               |              BMCWEB                 | |   EVENT LOG/    |
|   CLIENT    |   | LISTENERS  |               |RESOURCE SENDER  FORMATTER  MONITOR  | |TELEMETRY SERVICE|
+-----+-------+   +-----+------+               +--+--------+--------+---------+------+ +-----+-----------+
      |                 |                         |        |        |         |              |
+--------------------------------------------------------------------------------------------------------
|SUBSCRIBE FOR EVENT|   |                         |        |        |         |              |
+-------------------+   |                         |        |        |         |              |
      |                 |                         |        |        |         |              |
      |                 |                         |        |        |         |              |
      |                 |    EVENT SUBSCRIPTION   |        |        |         |              |
      |                 |-------------------------|        |        |         |              |
      |                 |     SSE(ON ESTABLISHED CONNECTION)        |         |              |
      |                 |                         |        |        |         |              |
      |                 |                         +-----+  |        |         |              |
      |                 |                         |KEEP |  |        |         |              |
      |                 |                         |CONN |  |        |         |              |
      |                 |                         |HANDLE  |        |         |              |
      |                 |                         +<----+  |        |         |              |
      |                 |                         |        |        |         |              |
      |                 |                         |--------------------------->              |
      |                 |                         |    SEND SUBSCRIPTION INFO |              |
      |                 |                         |        |        |         |              |
+-----------------------------------------------------------------------------------------------------
|SEND EVENTS/METRICS WHEN GENERATED |             |        |        |         |              |
+-----------------------------------+             |        |        |         |              |
      |                 |                         |        |        |         |              |
      |                 |                         |        |        |         |              +----+
      |                 |                         |        |        |         |              |LOOP|
      |                 |                         |        |        |         |              +<---+
      |                 |                         |        |        |         +<------------>+
      |                 |                         |        |        |         | NOTIFY EVENT/TELEMETRY
      |                 |                         |        |        |         |       DATA   |
      |                 |                         |        |        |         +-------------->
      |                 |                         |        |        |         | READ DATA    |
      |                 |                         |        |        |         +<-------------+
      |                 |                         |        |        |         |              |
      |                 |                         |        |        |         +-----------+  |
      |                 |                         |        |        |         | MATCH     |  |
      |                 |                         |        |        |         |SUBSCRIPTION  |
      |                 |                         |        |        |         |WITH DATA  |  |
      |                 |                         |        |        |         +-----------+  |
      |       +-----+--------------------------------------------------------------------------+
      |       |LOOP |   |                         |        |        +<--------+              | |
      |       +-----+   |                         |        |        | SUBSCRIPTION           | |
      |       |         |                         |        |        |  AND  DATA             | |
      |       |         |                         |        |        |         |              | |
      |       |         |                         |        +<-------+         |              | |
      |       |         |                         |        |SUBSCRIPTION AND  |              | |
      |       |         |                         |        |FORMATTED DATA    |              | |
      |       |         |                         |        |        |         |              | |
      |       |         |                         |        +----+|  |         |              | |
      |       |         |                         |        |PUSH QUEUE        |              | |
      |       |         |                         |        |INCOMING|         |              | |
      |       |         |                         |        |DATA |  |         |              | |
      |       |         |                         |        |<---+|  |         |              | |
      |       |         |                         |        |        |         |              | |
      |       | +------+-----------------------------------------+  |         |              | |
      |       | |RETRY ||<---------------------------------+     |  |         |              | |
      |       | |LOOP  ||   SEND FORMATTED DATA   |        |     |  |         |              | |
      |       | |+-----+|                         |        |     |  |         |              | |
      |       | |+-------------------------------------------------------------------------------
      |       | ||SEND FAILED |                   |        |     |  |         |              | |
      |       | |+------------+                   |        |     |  |         |              | |
      |       | ||      |                         |        |     |  |         |              | |
      |       | |+-------------REPEAT SEND LOOP------------------|  |         |              | |
      |       | |       |                         |        |     |  |         |              | |
      |       | |+-------------------------------------------------------------------------------
      |       | ||SEND SUCCESS |                  |        |     |  |         |              | |
      |       | |+-------------+                  |        |     |  |         |              | |
      |       | |       |                         |        |---+ |  |         |              | |
      |       | |       |                         |        |POP| |  |         |              | |
      |       | |       |                         |        |QUEUE|  |         |              | |
      |       | |       |                         |        |<--+ |  |         |              | |
      |       | |       |                         |        +----+|  |         |              | |
      |       | |       |                         |        |SEND NEXT         |              | |
      |       | |       |                         |        |QUEUEED |         |              | |
      |       | |       |                         |        |DATA ON |         |              | |
      |       | |       |                         |        |<---+|  |         |              | |
      |       | |       |                         |        |     |  |         |              | |
      |       | |+-------------------------------------------------------------------------------
      |       | ||REACHED THRESHOLD LIMIT |       |        |     |  |         |              | |
      |       | |+------------------------+       |        |     |  |         |              | |
      |       | |       |                         |        |----+|  |         |              | |
      |       | |       |                         |        |DELETE  |         |              | |
      |       | |       |                         |        |STORE|  |         |              | |
      |       | |       |                         |        |& QUEUE |         |              | |
      |       | |       |                         |        |<---+|  |         |              | |
      |       | |       |                         |<-------|     |  |         |              | |
      |       | |       |                         |REMOVE RESOURCE  |         |              | |
      |       | |       |                         +------+ |     |  |         |              | |
      |       | |       |                         |REMOVE| |     |  |         |              | |
      |       | |       |                         |SUBSCRIPTION  |  |         |              | |
      |       | |       |                         |& CONNECTION  |  |         |              | |
      |       | |       |                         |<-----+ |     |  |         |              | |
      |       | |       |                         |-------------------------->|              | |
      |       | |       |                         |  UPDATE SUBSCRIPTION      |              | |
      |       | |       |                         |----------------------------------------->| |
      |       | |       |                         |    LOG REDFISH EVENT LOG  |              | |
      |       | |       |                         |        |     |  |         |              | |
      |       | +-------+----------------------------------------+  |         |              | |
      |       |         |                         |        |        |         |              | |
      |       +--------------------------------------------------------------------------------+
      |                 |                         |        |        |         |              |
+--------------------------------------------------------------------------------------------------
|DELETE EVENT SUBSCRIPTIONS |                     |        |        |         |              |
+---------------------------+                     |        |        |         |              |
      |                 |                         |        |        |         |              |
      +-----------------+------------------------>+        |        |         |              |
      |        DELETE SUBSCRIPTION                |        |        |         |              |
      |                 |                         +------+ |        |         |              |
      |                 |                         |REMOVE| |        |         |              |
      |                 |                         |SUBSCRIPTION     |         |              |
      |                 |                         <------+ |        |         |              |
      |                 +<------------------------|        |        |         |              |
      |                 |    CLOSE CONNECTION     |        |        |         |              |
      |                 |                         |--------------------------->              |
      |                 |                         |    UPDATE SUBSCRIPTION    |              |
      |                 |                         |----------------------------------------->|
      |                 |                         |    LOG REDFISH EVENT LOG  |              |
      |                 |                         |        |        |         |              |
+--------------------------------------------------------------------------------------------------
|CLIENT CLOSE CONNECTION |                        |        |        |         |              |
+------------------------+                        |        |        |         |              |
      |                 |                         |        |        |         |              |
      |                 |    CLIENT CLOSED        |        |        |         |              |
      |                 |------------------------>|        |        |         |              |
      |                 |                         +-------+|        |         |              |
      |                 |                         |REMOVE ||        |         |              |
      |                 |                         |SUBSCRIPTION     |         |              |
      |                 |                         +<------+|        |         |              |
      |                 |                         |--------------------------->              |
      |                 |                         |    UPDATE SUBSCRIPTION    |              |
      |                 |                         |----------------------------------------->|
      |                 |                         |    LOG REDFISH EVENT LOG  |              |
      |                 |                         |        |        |         |              |
      v                 v                         v        v        v         v              v
```

EventService 設計包含下列主要區塊。

1. Redfish 資源 (Schema based)
   1. EventService
   2. Subscription Collections (EventDestination collection)
   3. Subscriptions (EventDestination)
2. Event Service Manager
   1. Event log monitor
   2. Event log formatter
   3. MetricReport monitor/agent
   4. MetricReport formatter
   5. Event service Sender (Common)
3. EventService config store ( Persistent)

本文件亦涵蓋下列模組的說明。

1. Server-Sent Event(SSE)：說明 SSE 是什麼，以及在不同情況下連線的開啟與關閉如何
   處理。
2. Push style events(HTTP/HTTPS)：說明如何訂閱 Push 樣式事件，以及 bmcweb 中如何
   處理 http/https 連線。
3. SubmitTestEvent：submit test event 動作的詳細資訊。

本文件不涵蓋下列元件的設計。

1. Redfish Event Logs：這是既有的實作。在 OpenBMC 中，所有事件都會記錄到 Journal
   control logs，而 RSyslog 服務會監控與 "REDFISH_MESSAGE_ID" 相關的事件記錄，並將
   所需的資訊寫入 "/var/log/redfish" 以持續保存事件記錄。

   因此 EventService 設計之所以與 Redfish Event Logs 串接，主要基於下列兩個原因。
   - 每當有新事件記錄到 Redfish event Logs 時取得通知。
   - 讀取 Redfish 事件記錄資訊的方法。

   本文件涵蓋一個 Redfish Event Log 實作範例，包含在下述設計中。作為其中一部分，
   它使用 inotify 取得新事件記錄的通知，並透過存取 redfish 事件記錄檔來讀取記錄
   資訊。

   更多相關資訊請參閱下方連結。
   [Redfish Logging](https://github.com/openbmc/docs/blob/master/redfish-logging-in-bmcweb.md)

   若其他 OEM 對 Redfish EventLogs 有不同實作，只要滿足上述兩項需求，即可將其
   串接至此處說明的 EventService 設計。舉例來說，若 xyz OEM 使用以 D-Bus 為基礎的
   Redfish Event logs，他們可使用 'signal' 來通知新的 redfish 事件記錄與事件記錄
   資訊。他們可將 'inotify' 替換為 'signal'，並將其 OEM 專屬的 Redfish 事件記錄
   串接至本 Redfish EventService 設計。

2. Telemetry service：Redfish telemetry service 用於收集與不同資源相關的所有指標
   回報，例如 Power metrics、thermal metrics、memory metrics、processor metrics
   等。Telemetry 中的 Monitoring service 支援不同模式，並支援彙總運算
   (Single、average、max、min、sum 等)。

   Telemetry 服務的設計文件請參閱下方連結。
   [Telemetry Service](https://gerrit.openbmc.org/#/c/openbmc/docs/+/24357/)

### Redfish 資源 schema

Redfish 將有三個主要資源來支援 EventService。

1. Event Service
2. Subscription Collections
3. Subscriptions

所有設定與訂閱資訊資料都會快取在 bmcweb webserver 上，並與設定 json 檔同步，以還原
並持續保存這些資料。

**EventService** 包含服務的各項屬性，例如 Service Enabled、狀態、遞送重試次數、
Event Types、Subscriptions、Actions URI 等。

- URI : `/redfish/v1/EventService`
- SCHEMA :
  [EventService](https://redfish.dmtf.org/schemas/v1/EventService_v1.xml)
- METHODS : `GET, PATCH`

支援的屬性與範例請參閱 Testing 章節。

**Subscription Collections** 此資源用於取得所有訂閱的集合，也用於設定新的訂閱以
取得自動事件。

- URI : `/redfish/v1/EventService/Subscriptions`
- SCHEMA :
  [EventDestinationCollection](https://redfish.dmtf.org/schemas/v1/EventDestinationCollection_v1.xml)
- METHODS : `GET, POST`

範例請參閱 Testing 章節。

**Subscriptions** 此資源包含訂閱詳細資訊。用於取得各別訂閱資訊、更新與刪除既有
訂閱資訊。

- URI : `/redfish/v1/EventService/Subscriptions/<ID>`
- SCHEMA :
  [EventDestination](https://redfish.dmtf.org/schemas/v1/EventDestination_v1.xml)
- METHODS : `GET, PATCH, DELETE`

支援的屬性與範例請參閱 Testing 章節。

### Event service manager

Event Manager 是主要區塊，負責監控事件、格式化要求 body，並將其傳送至已訂閱的
目的地。它應支援 redfish 規格中定義的兩種事件格式。

1. Event：當訂閱詳細資訊符合那些事件記錄時，資源類型 'Event' 會以 json body 的
   形式將警示傳送至已訂閱的 client。

2. MetricReport：當訂閱詳細資訊符合 telemetry 服務所收集的 telemetry 資料時，它會
   以 json body 的形式將警示傳送至已訂閱的 client。

"Event service sender" 是兩種資源類型的共同實體。formatter 與 monitor 實體則依每種
資源類型而不同。

#### Event log monitor

在 OpenBMC 中，所有 redfish 事件記錄目前都會記錄到 BMC 檔案系統的持續性區域
("/var/log/redfish")。"Event service manager" 下的 "Event log monitor" 服務會使用
**inotify** 監控 redfish 事件記錄檔以識別新事件。一旦識別到新事件，它會從 redfish
事件檔讀取所有已記錄的事件資訊，並傳給 'Event log formatter' 做進一步處理。最後一次
處理的事件資料時間戳記會快取在 bmcweb 中，用以識別 redfish 事件記錄檔中的新事件。

它監控即時事件以非同步傳送，且不會傳送在 bmcweb 服務啟動之前就已記錄的事件。

#### Event log formatter

它用於讀取從 'Event log monitor' 傳入的事件記錄來格式化要求 body，並讀取訂閱資訊以
篩選資料。它會從 "Redfish event service store" 取得所有訂閱資訊，並透過比對使用者
訂閱資訊中的 severity、event type 來篩選事件。它會連同 CustomText、HttpHeaders 一起
格式化 http "request body"，並針對篩選後的訂閱者使用 http 通訊協定透過網路傳送
Events。

要求 body 範例請參閱 Tested 章節。

#### MetricReport monitor

Telemetry monitoring service（本文件未涵蓋）負責從 D-Bus 感測器收集指標，並將其
揭露為 D-Bus 物件。Monitoring service 支援不同的監控模式（periodic、on change、
on demand），並支援彙總運算，例如 single、average、max、min、sum 等。此服務會依
Telemetry 服務設定（例如 'Metric Definitions'、'Metric Report Definitions' 與
'Metric Triggers'）定期收集指標回報。

telemetry monitor 應在有回報收集時發出 signal。MetricReport monitor/agent 應實作
telemetry signal 的 listener，並使用訂閱資訊來比對、篩選出所需的指標回報，再傳給
'MetricReport formatter'。由於 'MetricReport monitor' 與已執行監控迴圈的
'Telemetry monitor' 緊密耦合，因此比對訂閱規則並提供給 MetricReport formatter 是
'Telemetry monitor' 的責任。EventService 應提供一種方式，在使用者建立、更新或刪除
訂閱時溝通訂閱資料。

#### MetricReport formatter

'MetricReport formatter' 負責格式化來自 'MetricReport monitor/agent' 的資料，方法
是檢查訂閱資訊，例如 HttpHeaders、Custom text 字串，連同 Telemetry 回報資料。

#### Event service sender

"Event service sender" 用於將從 'Event log formatter' 或 'Telemetry formatter'
傳入的已格式化要求 body 傳送至所有篩選後的訂閱者。它使用 beast 函式庫支援傳送
https 與 http 兩種要求。http/https 連線的處理（開啟／關閉）會因 'Push style events'
與 'SSE' 兩種訂閱類型而異，更詳細的內容於各自章節說明。

Beast 是一個僅含標頭檔的 C++ 函式庫，透過使用 Boost.Asio 一致的非同步模型提供低階
HTTP/1、WebSocket 與網路通訊協定的詞彙型別與演算法，作為撰寫可互通網路函式庫的基礎。
關於 beast 的更多資訊請參閱
[Beast](https://www.boost.org/doc/libs/develop/libs/beast/doc/html/beast/introduction.html)

1. PUSH 樣式事件傳遞：BMC 使用 boost::beast 非同步地將事件傳送至已訂閱的 client。
   以下為在加密與未加密通道上傳送同步與非同步要求的 Beast client 範例。
   [Beast Async](https://www.boost.org/doc/libs/develop/libs/beast/example/http/client/async/)
   [Beast AsyncSSL](https://www.boost.org/doc/libs/develop/libs/beast/example/http/client/async-ssl/)

2. Server-Sent Events (SSE)：訂閱資料的 EventDestination 資源應包含已開啟的 client
   SSE 串流連線的 opaque 值。在收到與特定訂閱資訊關聯的已格式化事件後，EventSender
   會將已格式化的事件資料寫入具有特定 ID 的開啟 SSE 連線，讓 client 能收到該事件。

在 push 樣式事件傳遞的情況下，https 連線會在第一個事件期間建立。若已使用訂閱 ID
建立連線，則會使用同一個連線 handle 來傳送事件。這個連線 handle 與訂閱對應表會儲存
在 event service sender 中。使用者可從單一目的地擁有多個訂閱，而 BMC 會為每個訂閱
關聯一個獨立的連線 handle。Event listener（web server）有能力接受來自同一個
client(BMC) 的多個連線，並依路由規則在 event listener 端解析收到的事件。

在 SSE 的情況下，連線 handle 會從資源讀取，並由 sender 用來傳送事件。

收到的已格式化資料會被推入 sender 維護、對應 connection-subscription map 的佇列中。
成功傳送事件資料後，會從佇列 pop 出該已格式化資料項目，接著嘗試傳送下一筆已格式化
資料，依此類推。

若傳送事件（記錄或指標回報）失敗，則會依照 Event Service 中 "DeliveryRetryAttempts"
設定的次數進行重試，間隔為 "DeliveryRetryIntervalSeconds"。在 Push Style Events 的
情況下，若傳送因故失敗，將重新建立 https 連線並重新嘗試第一次傳送。若在設定次數的
重試之後失敗仍然存在，訂閱資訊將被刪除，並同步更新給消費者（Telemetry 服務或
event log 服務）。在 SSE 連線中，bmcweb 僅在網路失敗／逾時／壅塞的情況下才會重試
傳送事件。若發生 connreset／badf 之類的錯誤，bmcweb 會如上所述刪除訂閱。在這些
情況下，使用者可重新訂閱事件。依目前的 redfish 規格，這是必要行為。

若事件通知期間發生暫時性網路失敗，且失敗持續到門檻上限，訂閱項目將被移除。這可能是
個安全疑慮，因為網路恢復後 client 可能並不知道訂閱已終止。注意：Redfish 規格指出，
Event destination 訂閱在達到門檻上限後應被終止。這是已知的安全疑慮。TBD：需與
Redfish forum 合作尋求解決方法。

### Redfish event service store

Redfish event service store 用於儲存所有與 EventService 相關的設定與訂閱資訊。
它會執行下列操作：

- 在 bmcweb 服務啟動期間，它會從設定 json 檔取得所有設定與訂閱資訊，並填補內部
  結構資料。

- 當有任何經由 redfish 的使用者操作（例如 Get／Add／Update／Delete 訂閱等）時，
  Redfish EventService 資源會取得／更新此 event store。

- 當有更新時，它會將 "event config store" 與設定 Json 檔同步以持續保存資訊。
  **注意**：設定 json 檔僅會針對 Push 樣式事件的訂閱資訊更新。SSE 事件的訂閱資訊
  是暫時的（volatile），不會更新到 json 檔。

- 它會用於讓 "Event service manager" 取得所有訂閱資訊，以篩選事件資料。

bmcweb 服務會將 config store 資料（包含 EventService 設定與訂閱資訊）更新到 json
檔（位置：/var/lib/bmcweb/eventservice_data.json）。這會在 bmcweb 服務重新啟動或
bmc 重新開機後持續保存，並在初始化 bmcweb EventService 類別時載入。

注意：

1. 在 SSE 的情況下，client 是連線的發起者，因此當 bmcweb 服務或 BMC 重新開機時，
   SSE 串流連線無法重新建立。所以與 SSE 相關的訂閱資訊不會持續保存，且每當 bmc
   離開 reset 後，client 必須使用 "ServerSentEventUri" 重新訂閱 SSE 事件。

   限制：若 bmcweb 伺服器因 watchdog 觸發等原因突然關閉，則 bmcweb 無法傳送
   close connection 事件給 SSE client（以通知 SSE client 重新建立連線）。

快取實作包含下列主要區塊：

- EventServiceStore Class：它保存所有訂閱的 map（具有唯一 key），以及
  Create／Delete／update 訂閱資訊的方法。它也保存與 EventService schema 相關的高階
  設定資訊。

- Configuration structure：它保存與 EventService schema 屬性相關的設定。

- Subscription structure：它保存與各 EventDestination schema 屬性相關的資料，以及
  已建立的連線 handle。

### Server-Sent Events (SSE)

Server-Sent Events (SSE) 是一種伺服器推播技術，讓瀏覽器能透過 HTTP 連線自動接收來自
伺服器的更新。Server-Sent Events 的 EventSource API 已由 W3C 標準化為 HTML5 的一部分。

Server-Sent Events 是一項標準，描述在初始的 client 連線建立之後，伺服器如何朝向
client 主動發起資料傳輸。它常用於將訊息更新或連續資料串流傳送至瀏覽器 client，並旨在
透過名為 EventSource 的 JavaScript API 強化原生、跨瀏覽器的串流；client 透過它要求特定的
URL 以接收事件串流。

"EventService" 資源包含一個名為 "ServerSentEventUri" 的屬性。若 client 對
"ServerSentEventUri" 所指定的 URI 執行 GET，bmcweb 伺服器會保持連線開啟，並遵循
HTML5 規格，直到 client 關閉 socket 為止。bmcweb 服務應對 GET URI 送出帶有下列標頭的
回應。

```text
'Content-Type': 'text/event-stream',
'Cache-Control': 'no-cache',
'Connection': 'keep-alive'
```

OpenBMC 所產生的事件會使用開啟的連線傳送至 client。事件訊息的回應在每一行應包含
"data: " 前綴與 "\n" 後綴。為表示資料結束，應以 "\n\n" 結尾。傳送 json 回應的
範例看起來如下：

```text
data: {\n
data: "msg": "hello world",\n
data: "id": 12345\n
data: }\n\n
```

開啟連線：當 client 使用所需的篩選條件對 'ServerSentEventUri' 執行 get 時，bmcweb
會在 subscriptions 集合中建立一個 EventDestination 實例。一旦 bmcweb 收到 SSE 串流
要求，會透過檢查要求 Content-Type 標頭（符合 'text/event-stream'）將開啟的連線升級為
WebSocket，並保持連線存活，直到 client／伺服器關閉連線。關於如何為 SSE 升級連線的
詳細資訊，請參閱
[code review](https://gerrit.openbmc.org/#/c/openbmc/bmcweb/+/13948/)。EventDestination
資源中的 "Context" 屬性應為由服務產生的 opaque 字串。

注意：由於這是對既有已開啟連線（在正常操作下已通過驗證與授權）進行的 GET 操作，因此
不會帶來特殊的安全漏洞。所有 bmcweb 相關的安全性威脅模型皆適用於此。

關閉連線：client 或伺服器皆可隨時終止事件串流並關閉連線。當連線關閉時，bmcweb 伺服器
可刪除對應的 EventDestination 實例。

連線可透過以下方式關閉：

- Client 對已訂閱 id 的 URI 傳送 DELETE 方法。
- 伺服器（bmcweb）可關閉連線，並使用 'event: close' 要求 body 向 client 發送通知。

由於 SSE 是單向的，若 client 突然關閉連線／瀏覽器，bmcweb 服務並不知道 client 的連線
狀態。這是 SSE 連線的一項限制。在這種情況下，若遞送錯誤次數超過預先設定的門檻，
bmcweb 服務可刪除訂閱。

注意：

- SSE 訂閱資料本質上不是持續性的。當 bmcweb 重新啟動或 BMC 重新開機時，client 應重新
  建立對 bmcweb 的 SSE 連線以聆聽事件。
- 在 SSE 的情況下，client 可將 "Last-Event-ID" 作為標頭傳送。Monitor 服務會取得在
  該指定 "Last-Event-ID" 之後擷取的所有事件或指標回報串流，並在格式化後傳送至
  client。若找不到指定的事件 ID，它將僅開始串流即時事件／指標回報。

服務應支援使用 client 在 SSE 串流的 URI 中提供的 $filter 查詢參數，以減少回傳給
client 的資料量。

支援的篩選條件為：

- RegistryPrefix
- ResourceType
- EventFormatType
- MessageId
- OriginResource

使用篩選條件訂閱 SSE 的範例：

```text
MessageId based filter:
METHOD: GET
URI: https://<BMCIP>/redfish/v1/EventService/SSE?MessageId='Alert.1.0.LanDisconnect'

RegistryPrefix based filter:
https://sseuri?$filter=(RegistryPrefix eq Resource) or (RegistryPrefix eq
Task)

ResourceType based filter:
https://sseuri?$filter=(ResourceType eq 'Power') or (ResourceType eq
'Thermal')
```

### Push style events (HTTP/HTTPS)

push 樣式事件的運作方式與 SSE（bmcweb 扮演伺服器）不同。在此情況下，BMC 扮演 HTTP
client，將資料傳送至在外部系統上執行的 webserver。在 SSE 的情況下，連線會保持存活直到
串流關閉；但在 push 樣式事件中，連線依需求開啟，並在推送事件／回報資料後立即關閉。

使用者可使用 POST 方法於 '/redfish/v1/EventService/Subscriptions' URI，搭配所需的
要求 body（必要欄位：Destination、Protocol）來訂閱事件。當有事件記錄或 Telemetry 資料
可用時，monitor 與 formatter 資源（Event log 與 MetricReport 兩者）會使用訂閱資料
作為比對規則。'Event service sender' 使用訂閱資料中的 destination 欄位來開啟連線，
並將資料推送至外部執行的 webserver。

它應支援 http 與 https（Redfish 規格未定義 https）兩種連線，這些連線將使用
boost::beast 函式庫建立。對於 https 連線，它使用載入於 certificate store 中的自簽
憑證。若建立連線失敗，它會依設定的重試次數（DeliveryRetryAttempts）重試，並在達到
門檻上限時刪除訂閱。若 SSL 協商失敗，則不會重試建立連線。使用者可上傳新憑證。更多
關於憑證的細節可於
[TLS design](https://github.com/openbmc/docs/blob/master/designs/redfish-tls-user-authentication.md)
找到。

建立 Push 樣式事件的範例記錄於本文件的 testing 章節。

### SubmitTestEvent

此動作會以動作參數中指定的事件資料，將一筆測試事件新增到事件服務。根據訂閱資料，
這則訊息接著應傳送至所有已訂閱的 Listener 目的地目標。

- URI: `/redfish/v1/EventService/Actions/EventService.SubmitTestEvent`
- SCHEMA:
  [EventService](https://redfish.dmtf.org/schemas/v1/EventService_v1.xml)
- METHODS : `POST`

注意：EventService.v1_3_0 已更新，棄用 SubmitTestEvent 中的 EventType 參數，並新增
SubmitTestEvent 中的 EventGroupId 參數。

POST 要求 body 範例：

```json
{
  "Message": "Test Event for validation",
  "MessageArgs": [],
  "EventId": "OpenBMC.0.1.TestEvent",
  "EventGroupId": "",
  "Severity": "OK"
}
```

## 曾考慮的替代方案

### HTTP POLLING

輪詢（polling）是一種傳統技術，由 client 反覆地向伺服器輪詢資料。client 提出要求並
等待伺服器回應資料。BMC 提供了一種輪詢事件記錄的方式（使用 Events schema）。client 可
持續對 Events LogService 執行 http GET，並從 BMC 取得所有事件。但在 HTTP 通訊協定中，
抓取資料繞著 request/response 進行，造成較高的 HTTP 開銷。

Push 樣式(https) 事件傳遞或 SSE 可避免持續輪詢的 http 開銷，並在伺服器端發生時將
資料／事件傳送給已訂閱的 client。

### WEBSOCKET

WebSocket 提供了更豐富的協定，以進行雙向、全雙工通訊。擁有雙向通道對某些情境更具
吸引力，但在某些情境下，資料不需要從 client 送出（client 只需要在發生時收到來自伺服器的
錯誤／事件更新）。

SSE 提供單向連線，並在傳統 HTTP 上運作。SSE 不需要特殊的協定或伺服器實作即可運作。
另一方面，WebSocket 需要全雙工連線以及新的 Web Socket 伺服器來處理該協定。

### IPMI PEF

Platform Event Filtering (PEF) 提供了一種機制，可設定 BMC 對其收到的事件訊息採取選取的
動作。此機制以 IPMI 為中心，與 Redfish 相比，在安全性、擴展性等方面有其自身的限制。

## 實作設計替代方案

- 直接讀取 journal logs，對比使用 inotify 讀取 Redfish 事件記錄檔

曾考慮使用 `sd*journal*<xyz>` api 直接讀取 journald logs，但這似乎效率不佳，因為

1. sdjournal 目前同時包含多種記錄（不只是事件記錄），因此要監控並處理資料、僅辨識出
   redfish 事件記錄會非常沒有效率。

2. 需要在 sdjournal 中加入一些 hook，以在有任何訊息記錄到 sdjournal 時發出 signal。

為克服上述挑戰，目前會使用 rsyslog 將 redfish 事件記錄篩選後記錄到獨立的檔案。
EventService 同樣使用該檔案，透過 inotify 監控它。

- D-Bus 服務，對比 Event Service config store

曾考慮建立獨立的 D-Bus 服務，以儲存 EventService 設定與訂閱資料。也曾考慮使用
settings D-Bus 服務在重新開機之間持續保存資料。但由於 EventService 僅限
redfish/bmcweb 使用，且不會被系統中其他任何程序消費／更新，因此擁有 D-Bus 服務並沒有
實質的好處。

## 影響

Push 樣式事件的安全性問題：

push 樣式事件也支援 HTTP 通訊協定，其並非安全，易受中間人攻擊（man-in-the-middle）與
竊聽影響，可能導致病毒注入，並向攻擊者洩露敏感資訊。push 樣式事件的訂閱或對 BMC 的變更，
會透過安全的 HTTPS 通道，並以完整的驗證與授權進行。因此 BMC 端的安全性並未被破壞。
在中間人攻擊的情況下，client 端的 Event Listener 可能收到遭竄改的資料。為避免這類攻擊，
建議使用 SSE 事件機制或基於 https 的 push 樣式事件。

SSE 運作於 HTTPS 通訊協定之上，其為加密且安全的。SSE EventService 訂閱可透過 HTTPS
通訊協定建立或更新，並搭配對 BMC 使用者的適當驗證與授權。更多資訊請參閱 DSP0266 中的
安全性細節（section 13）。

SSE 連線：

當 client 開啟 SSE HTTPS 連線後，它會保持開啟狀態，直到 client 或伺服器關閉連線。
在 Linux 上，每個檔案描述元都有上限（當然這是一個非常大的數字 > 8K）。bmcweb 上有多個
開啟的 http 連線可能會拖慢效能。

由於已開啟的 SSE client 連線受到 BMC 端多層保護的代理（例如需要驗證、使用者必須具備
適當的權限等級才能建立 SSE 連線等），這些影響已獲得緩解。此外，BMC 將支援最多 10 個
SSE 訂閱，這將限制 http 連線耗盡的情況。這可以作為一個編譯期選項。

## 測試

使用者應涵蓋下列功能。

- 使用 GET 與 PATCH 方法檢視並更新 EventService 設定，並驗證其功能。
- 如下所述訂閱 Events（「Push style event」與「SSE」兩者），並針對正面與負面用例
  （body 參數無效、無效要求、權限不當 - forbidden 等）驗證 http 狀態碼。
- 驗證所有訂閱集合以及每一個個別訂閱。
- 對特定 ID 的訂閱 URI 使用 delete 方法，觀察 client 端連線是否正常關閉、server 端
  訂閱資料是否清除。
- 啟用與停用 EventService，產生事件，並驗證在啟用時 Events 是否正常傳送至所有訂閱、
  停用時是否停止傳送。
- 使用 SubmitTestEvent 產生測試事件。也涵蓋帶有無效 body 資料的負面用例。
- 同時針對正面與負面用例，驗證重試次數與重試逾時的功能。
- client 連線關閉：關閉正在聆聽 SSE 的 client 連線，檢查在設定次數的重試之後 server 端
  連線會關閉，並清除訂閱資訊。
- 重新開機 BMC 或重設 bmcweb 服務，並透過產生與檢查事件，確認所有「push style events」
  訂閱皆被保留且功能正常。
- 重新開機 BMC 或重設 bmcweb 服務，並確認所有「SSE」訂閱資料皆未被保留，且 client 端
  連線正常關閉。
- 建立具有多重權限的使用者，檢查建立／刪除／更新訂閱與設定資料僅對具備
  'ConfiguredManager' 權限的使用者開放，且檢視 EventService 資訊僅對 'Login' 開放。
- 建立新訂閱，檢查事件訊息是否記錄在 redfish 事件記錄中。也修改、刪除訂閱，並觀察
  Redfish 事件是否被記錄。
- 驗證在具有適當篩選資料的訂閱上，傳送至 client 的事件之正確性（例：若 RegistryPrefixes
  設為 "OpenBMC.0.1"，則不應傳送具有 "OpenBMC.0.2" 前綴的事件）。
- 就安全性而言，以未授權使用者驗證對 "ServerSentEventUri" 指定 URI 的 GET，觀察它以
  適當的 http 狀態碼回應，且不應建立任何訂閱或開啟任何 http 連線。
- 上限檢查：建立最多支援的事件訂閱並驗證。

### Push 樣式事件

1. EventListener：client 應設定 web 服務以聆聽事件（範例：`http://192.168.1.2/Events`）。

2. Subscription：client 可使用以下方式向 BMC 訂閱事件

   ```text
   URI: /redfish/v1/EventService/Subscriptions
   METHOD: POST
   REQUEST BODY:
   {
   "Destination":"http://192.168.1.2/Events",
   "Context":"CustomText",
   "Protocol":"Redfish",
   }
   ```

   事件訂閱成功後，client 會收到 201(Created) http 狀態碼，連同建立的訂閱 ID。從此刻
   起，client 會收到與訂閱資料相關的事件。

   client 可使用 "SubmitTestEvent"（上方文件已說明）來測試運作狀態。

3. ViewSubscription：client 可使用以下方式取得訂閱詳細資訊

   ```text
   URI: /redfish/v1/EventService/Subscriptions/<ID>
   METHOD: GET
   ```

4. UpdateSubscription：client 可隨時更新訂閱資訊。

   ```text
   URI: /redfish/v1/EventService/Subscriptions/<ID>
   METHOD: PATCH
   REQUEST BODY:
   {
     "Destination":"http://192.168.1.2/Events",
     "Context":"Changed Custom Text",
     "RegistryPrefix": "OpenBMC.0.2"
   }
   ```

5. DeleteSubscription：client 可取消訂閱，並透過刪除訂閱來停止接收事件。

   ```text
   URI: /redfish/v1/EventService/Subscriptions/<ID>
   METHOD: DELETE
   ```

### Server-Sent events 訂閱

1. EventListener and Subscription：client 可開啟最新版的瀏覽器（chrome），並在
   "ServerSentEventUri" 上搭配篩選條件對其發出 GET URI。

   ```text
   URI: /redfish/v1/EventService/SSE?RegistryPrefix='OpenBMC.0.2'&MessageId='OpenBMC.0.2.xyz"
   METHOD: GET
   ```

   事件訂閱成功後，client 會收到 200(Created) http 狀態碼、訂閱 id，連同下列用於串流的
   回應標頭。

   ```text
   'Content-Type': 'text/event-stream',
   'Cache-Control': 'no-cache',
   'Connection': 'keep-alive'
   ```

   從此刻起，client 瀏覽器會收到與訂閱資料相關的事件。

   client 可使用 "SubmitTestEvent"（上方文件已說明）來測試運作狀態。

2. ViewSubscription：client 可使用以下方式取得訂閱詳細資訊

   ```text
   URI: /redfish/v1/EventService/Subscriptions/<ID>
   METHOD: GET
   ```

3. UpdateSubscription：client 可隨時更新訂閱資訊。注意，client 無法變更 destination，
   因為它是與已開啟 SSE 連線相關聯的 opaque 值。

   ```text
   URI: /redfish/v1/EventService/Subscriptions/<ID>
   METHOD: PATCH
   REQUEST BODY:
   {
     "Context":"Changed Custom Text",
     "RegistryPrefix": "OpenBMC.0.2"
   }
   ```

4. DeleteSubscription：從 client 端有兩種方式可關閉連線。
   - client 可直接關閉瀏覽器，這將關閉 SSE http 連線，因此 bmcweb 服務可關閉並刪除
     訂閱資料。
   - client 也可透過刪除訂閱來取消訂閱並停止接收事件。

   ```text
   URI: /redfish/v1/EventService/Subscriptions/<ID>
   METHOD: DELETE
   ```

### EventListener

Push 樣式 Events：

要聆聽 push 樣式事件，client 可在任何外部系統上執行 web server，並搭配 destination URI
與訂閱詳細資訊向 BMC 訂閱。

存在多種 EventListener，以下是一個簡單的 node js 伺服器，可使用 "node server.js" 執行。

```js
// server.js
const http = require("http");
var displayResponse;

http
  .createServer((request, response) => {
    console.log("Requested url: " + request.url);
    const { headers, method, url } = request;

    if (method.toLowerCase() == "post" && url.toLowerCase() === "/events") {
      let body = [];
      request
        .on("error", (err) => {
          console.error(err);
        })
        .on("data", (chunk) => {
          body.push(chunk);
        })
        .on("end", () => {
          body = Buffer.concat(body).toString();

          console.log(body);
          displayResponse.write(body);
          displayResponse.write("\n\n");

          response.statusCode = 200;
          response.end();
        });
    } else if (
      method.toLowerCase() == "get" &&
      url.toLowerCase() === "/display_events"
    ) {
      response.writeHead(200, {
        Connection: "keep-alive",
        "Content-Type": "text/event-stream",
        "Cache-Control": "no-cache",
        "Access-Control-Allow-Origin": "*",
      });

      displayResponse = response;
    } else {
      response.writeHead(404);
      response.end();
    }
  })
  .listen(5000, () => {
    console.log(
      "Use 'http://<IP>:5000/events' as destination to subscribe for BMC events",
    );
    console.log("Open 'http://<IP>:5000/display_events' URL on browser\n");
  });
```

Server-Sent Events (SSE)：

client 可透過在 Event Service 中對 "ServerSentEventUri" 所指定的 URI，搭配篩選條件執行
GET，來向服務開啟 SSE 連線。瀏覽器會自動提示輸入憑證，在驗證與授權成功完成後，即會
建立 client 連線以聆聽事件。

您也可以參考下方連結以取得關於 Server-Sent events 的詳細資訊。
<https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events/Using_server-sent_events>

另一個簡單的 python 腳本，使用 EventSource 測試 SSE。使用 "python sse-client.py" 執行。

```python
// sse-client.py
import requests, json, sys, time, re, os
import sseclient
from requests.packages.urllib3.exceptions import InsecureRequestWarning
requests.packages.urllib3.disable_warnings(InsecureRequestWarning)
from sseclient import SSEClient

try:
  bmc_ip = sys.argv[1]
  username = sys.argv[2]
  password = sys.argv[3]

except:
  print "FAIL: Usage: <ScriptName> <BMC_IP> <Username> <Password>"
  print "Example: python sseClient.py 10.223.244.145 root root\n"
  sys.exit()


url = 'https://%s/redfish/v1/EventService/SSE?<filters>' %bmc_ip

try:
  messages = SSEClient(url, verify=False, auth=(username, password))

  for msg in messages:
    print(json.dumps(msg.data))

except Exception as exc:
  print("\nERROR: Error in subscribing the SSE events\n")
```

#### EventService

支援的屬性：

| PROPERTY NAME                | TYPE          | DESCRIPTION                                                                                                                                                                                                                                                                                                 |
| ---------------------------- | ------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| ServiceEnabled               | Boolean       | 這表示事件服務是否已啟用。預設：True                                                                                                                                                                                                                                              |
| DeliveryRetryAttempts        | Integer       | 這是在訂閱被終止之前，事件遞送會重試的次數。預設：3                                                                                                                                                                                                |
| DeliveryRetryIntervalSeconds | Integer       | 這代表傳送任何給定 Event 時，兩次重試之間相隔的秒數。預設：30                                                                                                                                                                                                       |
| EventFormatTypes             | Array(string) | 表示此服務可傳送至事件目的地的訊息內容類型。**Event：** 將 'Event' 的資源類型傳送至已訂閱的目的地。若未指定任何內容，此為預設值。**MetricReport：** 僅將 'MetricReport' 的資源類型傳送至已訂閱的目的地。 |
| RegistryPrefixes             | Array(string) | 可用於訂閱上 RegistryPrefix 屬性的 Message Registries 之前綴清單。若此屬性不存在或包含空陣列，則服務不支援以 RegistryPrefix 為基礎的訂閱。                                                                |
| ResourceTypes                | Array(string) | 可用於訂閱上 RegistryPrefix 屬性的 message registries 之前綴清單。若此屬性不存在或包含空陣列，則服務不支援以 RegistryPrefix 為基礎的訂閱                                                                 |
| ServerSentEventUri           | String        | 此屬性的值應為一個 URI，指定一個符合 HTML5 Server-Sent Event 的端點。URI：/redfish/v1/EventService/SSE                                                                                                                                                                  |
| Subscriptions                | Object        | 此屬性的值應包含一個連結，指向類型為 EventDestinationCollection 的集合。                                                                                                                                                                                                       |
| Actions                      | Object        | 此動作會以動作參數中指定的事件資料，將一筆測試事件新增到事件服務。關於 TestEvents 的更多細節於下方記錄。                                                                                                                                                  |

注意：建立 EventService.v1_3_0 是為了棄用 EventTypesForSubscription 與
SSEFilterPropertiesSupported\EventType 屬性。

GET 回應範例：

```json
{
  "@odata.context": "/redfish/v1/$metadata#EventService.EventService",
  "@odata.id": "/redfish/v1/EventService",
  "@odata.type": "#EventService.v1_3_0.EventService",
  "Actions": {
    "#EventService.SubmitTestEvent": {
      "target": "/redfish/v1/EventService/Actions/EventService.SubmitTestEvent"
    }
  },
  "DeliveryRetryAttempts": 3,
  "DeliveryRetryIntervalSeconds": 30,
  "EventFormatTypes": ["Event"],
  "Id": "EventService",
  "Name": "Event Service",
  "RegistryPrefixes": ["Base", "OpenBMC"],
  "ServerSentEventUri": "/redfish/v1/EventService/SSE",
  "ServiceEnabled": true,
  "Subscriptions": {
    "@odata.id": "/redfish/v1/EventService/Subscriptions"
  }
}
```

#### Subscription Collections

GET 輸出範例：

```json
{
  "@odata.type":"#EventDestinationCollection.EventDestinationCollection",
  "@odata.context":"/redfish/v1/$metadata#EventDestinationCollection.EventDestinationCollection",
  "@odata.id":"/redfish/v1/EventService/Subscriptions"
  "Id":"Event Subscription Collection",
  "Name":"Event Subscriptions Collection",
  "Members@odata.count": 2,
  "Members": [
    {
      "@odata.id":"/redfish/v1/EventService/Subscriptions/1",
      "@odata.id":"/redfish/v1/EventService/Subscriptions/2"
    }
  ],
}
```

#### Subscriptions

支援的屬性：

| PROPERTY NAME    | TYPE          | DESCRIPTION                                                                                                                                                   |
| ---------------- | ------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Destination      | String        | 目的地 Event Service 的 URI。                                                                                                                     |
| Context          | String        | 由 client 提供、會與事件目的地訂閱一起儲存的字串。預設：null                                                                |
| Protocol         | Enum          | 事件連線的通訊協定類型。預設：redfish                                                                                                   |
| HttpHeaders      | array(string) | 用於設定 HTTP 標頭，例如授權資訊。預設：null                                                                            |
| RegistryPrefixes | String        | 包含將傳送至本事件目的地之 MessageIds 所屬 Message Registries 的前綴清單。預設：null                      |
| ResourceTypes    | Array(String) | 與 OriginOfCondition 對應的 Resource Type 值（Schema 名稱）清單。不應指定版本與完整 namespace。預設：null |
| EventFormatType  | String        | 表示將傳送至 EventDestination 的訊息內容類型。支援的類型為 'Event' 與 'MetricReport'。預設：Event          |

GET 要求範例：

```json
{
  "@odata.context": "/redfish/v1/$metadata#EventDestination.EventDestination",
  "@odata.id": "/redfish/v1/EventService/Subscriptions/1",
  "@odata.type": "#EventDestination.v1_5_0.EventDestination",
  "Context": "143TestString",
  "Destination": "http://test3.domain.com/EventListener",
  "EventFormatTypes": ["Event"],
  "HttpHeaders": ["X-Auth-Token:XYZABCDEDF"],
  "Id": "1",
  "MessageIds": ["InventoryAdded", "InventoryRemoved"],
  "Name": "EventSubscription 1",
  "Protocol": "Redfish",
  "RegistryPrefixes": ["Base", "OpenBMC"],
  "SubscriptionType": "Event"
}
```

- Event log 格式化後的回應 body 看起來像：

```text
id: 1
data:{
data: "@odata.type": "#Event.v1_1_0.Event",
data: "Id": "1",
data: "Name": "Event Array",
data: "Context": "ABCDEFGH",
data: "Events": [
data: {
data:  "Created": "2019-08-13T06:25:31+00:00",
data:  "EntryType": "Event",
data:  "Id": "1565677531",
data:  "Message": "F1UL16RISER3 Board with serial number BQWK64100593 was installed.",
data:  "MessageArgs": [
data:    "F1UL16RISER3",
data:    "Board",
data:    "BQWK64100593"
data:  ],
data:  "MessageId": "OpenBMC.0.1.InventoryAdded",
data:  "Name": "System Event Log Entry",
data:  "Severity": "OK"
data:  "Context": "ABCDEFGH"
data:  }
data: ]
data:}
```

- 設定結構的偽程式碼：

```cpp
struct EventSrvConfig
{
    bool enabled;
    uint32_t retryAttempts;
    uint32_t retryTimeoutInterval;
};
```

- 訂閱結構的偽程式碼：

```cpp
struct EventSrvSubscription
{
    std::string destinationUri;
    std::string customText;
    std::string subscriptionType;
    std::string protocol;
    std::vector<std::string> httpHeaders; // key-value pair
    std::vector<std::string> registryMsgIds;
    std::vector<std::string> registryPrefixes;
    std::vector<std::string> eventFormatTypes;
};
```
