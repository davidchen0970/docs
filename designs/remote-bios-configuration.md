# 透過 BMC 進行遠端 BIOS 設定（RBC）

作者：Suryakanth Sekar! ssekar

其他貢獻者：Jia Chunhui Deepak Kodihalli Patrick Williams

建立日期：19-Nov-2019

## 問題描述

目前的 OpenBMC 並未提供遠端設定 BIOS 的機制。在資料中心中，需要在相同設定下維繫許多的系統，因此需要這個功能。

遠端 BIOS 設定（Remote BIOS Configuration）讓使用者能夠在任何主機狀態下，透過 BMC 遠端檢視與修改 BIOS 設定參數。新的 BIOS 設定參數會立即生效，或在下次系統重新啟動時生效，端視主機韌體的支援模型而定。

## 背景與參考資料

1. <https://www.dmtf.org/sites/default/files/standards/documents/DSP0247_1.0.0.pdf>
2. <https://redfish.dmtf.org/schemas/v1/Bios.v1_1_0.json>
3. <https://redfish.dmtf.org/schemas/v1/AttributeRegistry.v1_3_2.json>

## 需求

1. 提供可透過網路介面遠端設定 BIOS 設定的機制。
2. BMC 應支援將所有 BIOS 變數的值設回原廠預設狀態的能力。
3. 根據主機韌體的支援模型，BMC 應同時支援立即更新（Immediate Update）或延遲更新（Deferred update）。
4. 在延遲模型中，當系統處於 S0、S3、S4 或 S5 狀態時，BMC 只會在下次 BIOS 啟動時送出已更新的 BIOS 變數，而不會立即啟動 BIOS。
5. 在立即模型中，每當設定被修改時，BMC 都應向系統韌體（BIOS）發送訊息。
6. BMC 應在 redfish schema 中支援 BIOS attribute registry，以進行 BIOS 設定。
7. BMC 應提供安全的方式來更新 BIOS 設定密碼的設定。詳細的密碼處理設計為 TBD（將使用 ARM TrustZone 解決）。例如：更新 BIOS 密碼應該只在 POST 結束之前受到支援。
8. 遠端 BIOS 設定 daemon 應獨立於介面專屬的資料格式。
9. BMC 應能夠從主機取得預設／目前的設定並儲存，再將其暴露出來以供帶外（out of band）更新使用。
10. BMC 應將新的值提供給主機。

## 建議設計

```text
+----------------------------------------------------------------------------------------------------------------+
| Remote BIOS configuration (RBC) via BMC                                                                        |
|                                                                                                                |
|                                                                                                                |
| +-------------+       +-------------+       +--------------------------------+      +-------+                  |
| |             |       |             |       |   RBC daemon                   |      |       |    +----------+  |
| | NET/ Tools  +<-LAN->+ LAN-IPMID/  +<Dbus->+                                |      |       |    |Web client|  |
| |             |       | REDFISH     |       |  Provide following Methods     |      |       |    |          |  |
| +-------------+       +-------------+       |     -SetAttribute()            |      |       |    +----^-----+  |
|                                             |     -GetAttribute()            |      |       |         |        |
| +-------------+       +-------------+       |     -VerifyPassword()          |      |       |        LAN       |
| |             |       |             |       |     -ChangePassword()          |      |       |         |        |
| | HOST/ BIOS  +<-KCS->+  HOST-IPMID +<Dbus->+                                |      |Redfish|    +----V-----+  |
| |             |       |             |       | Properties                     +<Dbus>+  API  |    |Redfish & |  |
| +-----+-------+       +-------------+       |     -BaseBIOSTable             |      |       +<-->+BMCWeb    |  |
|       |                                     |     -PendingAttributes         |      |       |    +----^-----+  |
|       |                                     |     -ResetBIOSSettings         |      |       |         |        |
|       |                                     |     -IsPasswordInitDone        |      |       |         |        |
|       |                                     |                                |      |       |    +----V-----+  |
|       |                                     |                                |      |       |    | Redfish  |  |
|       |                                     |                                |      |       |    |  Host    |  |
|       |                                     |                                |      |       |    | Interface|  |
|       |                                     +----^-----------------+---------+      +-------+    +----------+  |
|       |                                          |       PendingAttributesUpdated                              |
|       |                                          |                 |    Signal                                 |
|       |                                          |                 |                                           |
|       |                                     +----V-----------------V---------+                                 |
|       |                                     | PLDM daemon                    |                                 |
|       |                                     | Collect the BIOS data & convert|                                 |
|       +---------MCTP----------------------->| into native D-bus format and   |                                 |
|                                             | send to the RBC                |                                 |
|                                             +--------------------------------+                                 |
+----------------------------------------------------------------------------------------------------------------+
```

## Intel 使用以下的邏輯

BIOS 透過 IPMI 介面以專有（Proprietary）格式將資料傳送至 BMC。BIOS 設定中有兩種專有的 XML 格式檔案。Type-0 以 XML 格式包含完整的 BIOS 變數（由 BIOS 產生）。Type-1 以 XML 格式包含已修改的 BIOS 變數（由 BMC 產生）。

BIOS 必須在早期啟動階段透過 KCS 介面提供 BIOS 能力。BIOS 必須透過 IPMI 命令將壓縮過的專有 XML type 0 檔案傳送至 BMC。

IPMI 介面應解壓縮並驗證所收到的 XML Type 0。IPMI 介面應將 XML Type 0 轉換為 D-bus 的原生格式並傳送至 RBC daemon。

在 BIOS 的 BDS 階段中，BIOS 必須向 BMC 取得現有的 XML 資訊。如果 XML 版本／校驗和不符，或 BMC 中不存在 XML Type 0，則 BIOS 必須將 XML type 0 傳送至 BMC。如果 XML 版本與 XML 校驗和相符，且 BMC 中存在待處理（pending）的 BIOS 屬性清單（XML Type 1），則 BIOS 必須向 BMC 取得待處理的 BIOS 設定（XML Type 1）並在 BIOS 區域中更新，然後將已更新的 XML Type 0 傳回 BMC 以再次保持一致，接著 BIOS 重設系統，以便在 BIOS 啟動時反映已更新的值。

BIOS 在 BIOS 非揮發性記憶體中保有預設的 BIOS 設定。BIOS 可以根據 BIOS 啟動期間發出的 OEM IPMI 命令中的旗標設定，來還原預設的 BIOS 設定。因此，可以透過這個機制來還原預設 BIOS 設定。

RBC daemon 應在非揮發性儲存中保留 AllBiosTables 與 PendingAttributes 清單。每當收到新的屬性資料時，待處理屬性清單都會被清除。

### Intel 用於 BIOS 首次啟動的邏輯

```text
   +---------------------------------------------------------------------------------------------------------------------+
   |                                                                                                                     |
   | +-----------------------+             +----------------------------------------------------------------------------+|
   | | BIOS                  |             |   BMC                                                                      ||
   | |                       |             |  +-------------------------------+        +------------------------------+ ||
   | |                       |             |  |IPMI Interface (kcs)           |        |RBC daemon Manager            | ||
   | |                       |             |  | -Responsible for send /recv   |        |-Responsible for handling BIOS| ||
   | |                       |             |  |  data between BIOS and BMC    |        | configuration attributes.    | ||
   | |                       |             |  |                               |        | -AllBaseAttributes           | ||
   | |                       |             |  |                               |        | -Pending Attributes          | ||
   | |                       |             |  |                               |        | -FactoryDefaultSetting       | ||
   | |                       |             |  |                               |        | -BIOSPwdHashData             | ||
   | |                       |             |  +-------------------------------+        +------------------------------+ ||
   | +-----------------------+             |  +-------------------------------+         +-----------------------------+ ||
   | | Set BIOS capability   |<---Req-/Res--> | Set the BIOS capability       |         |                             | ||
   | |                       |             |  |                               |         |                             | ||
   | |Set BIOS Pwd hash/Seed |<---Req-/Res--> | Forward to the RBC daemon     |--dbus-->| Set BIOSPwdHash data.       | ||
   | |                       |             |  |                               |         |                             | ||
   | |                       |             |  |                               |         |                             | ||
   | |Check factory settings |<---Req-/Res--> |  Get the FactoryDefault prop  |<-dbus-- | Send FactoryDefaultSettings | ||
   | |Init the BIOS config   |             |  |                               |         |                             | ||
   | |Based on value.        |             |  |                               |         |                             | ||
   | |Get the XML Type0 info |<---Req-/Res--> | Provide the XML Type 0 Info   |         |                             | ||
   | |Generate & compress    |             |  |                               |         |                             | ||
   | |XML type 0 file        |             |  | Validate the XML Type 0       |         |Collect the Attributes Info  | ||
   | |                       |             |  | Unzip the XML Type 0          |--dbus-->| AllBaseAttributes           | ||
   | | Send the XML Type 0   |<---Req-/Res--->| Convert into D-bus format     |                                       | ||
   | | via SetPayload command|             |  | and send to the RBC.          |         |                             | ||
   | |                       |             |  |                               |         |                             | ||
   | | Continue the BIOS boot|             |  |                               |         |                             | ||
   | +-----------------------+             |  +-------------------------------+         +-----------------------------+ ||
   |                                       +----------------------------------------------------------------------------+|
   +---------------------------------------+-----------------------------------------------------------------------------+
```

### Intel 用於 BIOS 重設的邏輯

```text
   +---------------------------------------------------------------------------------------------------------------------+
   |                                                                                                                     |
   | +-----------------------+             +----------------------------------------------------------------------------+|
   | | BIOS                  |             |   BMC                                                                      ||
   | |                       |             |  +-------------------------------+        +------------------------------+ ||
   | |                       |             |  |IPMI Interface (kcs)           |        |RBC daemon Manager            | ||
   | |                       |             |  | -Responsible for send /recv   |        |-Responsible for handling BIOS| ||
   | |                       |             |  |  data between BIOS and BMC    |        | configuration attributes.    | ||
   | |                       |             |  |                               |        | -AllBaseAttributes           | ||
   | |                       |             |  |                               |        | -Pending Attributes          | ||
   | |                       |             |  |                               |        | -FactoryDefaultSetting       | ||
   | |                       |             |  |                               |        | -BIOSPwdHashData             | ||
   + |                       |             |  +-------------------------------+        +------------------------------+ ||
   | +-----------------------+             |  +-------------------------------+         +-----------------------------+ ||
   | |Set BIOS capability    |<---Req-/Res--> | Set the BIOS capability       |         |                             | ||
   | |                       |             |  |                               |         |                             | ||
   | |Set BIOS Pwd hash/Seed |<---Req-/Res--> | Forward to the RBC daemon     |--dbus-->| Set BIOSPwdHash data        | ||
   | |                       |             |  |                               |         |                             | ||
   | |                       |             |  |                               |         |                             | ||
   | |Check factory settings |<---Req-/Res--> | Get the FactoryDefault prop   |<-dbus-- | Send FactoryDefaultSettings | ||
   | |Init the BIOS config   |             |  |                               |         |                             | ||
   | |Based on value.        |             |  |                               |         |                             | ||
   | |Get the XML Type 0 info|<---Req-/Res--> | Provide the XML Type 0 Info   |         |                             | ||
   | |Generate & compress    |             |  |                               |         |                             | ||
   | |XML type 0 file.       |             |  |                               |         |Collect the Attributes Info  | ||
   | |Check XML file Chksum. |             |  | Validate the XML Type 0       |--dbus-->| BaseBIOSTable               | ||
   | |If Chksum mismatch     |<---Req-/Res--->| Unzip XML Type 0 & convert    |         |                             | ||
   | |then send the payload  |             |  | into D-bus format and send to |         |                             | ||
   | |via SetPayload.        |             |  | the RBC                       |         |                             | ||
   | |Continue the BIOS boot |             |  |                               |         |                             | ||
   | +-----------------------+             |  +-------------------------------+         +-----------------------------+ ||
   |                                       +----------------------------------------------------------------------------+|
   +---------------------------------------+-----------------------------------------------------------------------------+
```

### 當 BMC 有待處理設定時的 Intel BIOS 重設邏輯

```text
   +---------------------------------------------------------------------------------------------------------------------+
   |                                                                                                                     |
   | +-----------------------+             +----------------------------------------------------------------------------+|
   | | BIOS                  |             |   BMC                                                                      ||
   | |                       |             |  +-------------------------------+        +------------------------------+ ||
   | |                       |             |  |                               |        |                              | ||
   | |                       |             |  |IPMI Interface (kcs)           |        |RBC daemon Manager            | ||
   | |                       |             |  | -Responsible for send /recv   |        |-Responsible for handling BIOS| ||
   | |                       |             |  |  data between BIOS and BMC    |        | configuration attributes.    | ||
   | |                       |             |  |                               |        | -AllBaseAttributes           | ||
   | |                       |             |  |                               |        | -Pending Attributes          | ||
   | |                       |             |  |                               |        | -FactoryDefaultSetting       | ||
   | |                       |             |  |                               |        | -BIOSPwdHashData             | ||
   + |                       |             |  +-------------------------------+        +------------------------------+ ||
   | +-----------------------+             |  +-------------------------------+         +-----------------------------+ ||
   | |Set BIOS capability    |<---Req-/Res--> | Set the BIOS capability       |         |                             | ||
   | |                       |             |  |                               |         |                             | ||
   | |Set BIOS Pwd hash/Seed |<---Req-/Res--> | Forward to the RBC daemon     |---dbus->| Set BIOSPwdHash data        | ||
   | |                       |             |  |                               |         |                             | ||
   | |                       |             |  |                               |         |                             | ||
   | |Check factory settings |<---Req-/Res--> | Get the FactoryDefault prop   |<--dbus--| Send FactoryDefaultSettings | ||
   | |Init the BIOS config   |             |  |                               |         |                             | ||
   | |Based on value.        |             |  |                               |         |                             | ||
   | |Get the XML Type 0 info|<---Req-/Res--> | Provide the XML Type 0 Info   |         |                             | ||
   | |Generate & compress    |             |  |                               |         |                             | ||
   | |XML Type 0 file.       |             |  |                               |         |                             | ||
   | |Check XML file Chksum. |             |  |                               |         |                             | ||
   | |If Chksum   match      |<---Req-/Res--->| Provide the XML Type 0        |         |                             | ||
   | |then get the XML Type 1|             |  |                               |         |                             | ||
   | |via GetPayload.        |             |  |                               |         |                             | ||
   | |Get  the XML Type 1    |             |  |                               |         |                             | ||
   | |via GetPayload command.|<---Req-/Res--->| Provide the XML Type 1        |         |                             | ||
   | |                       |             |  |                               |         |                             | ||
   | |Update the new value   |             |  |                               |         |                             | ||
   | |and new chksum in      |<---Req-/Res--->| Get new XML Type 0 & convert  |--dbus-->| Collect the Attributes info | ||
   | |BIOS.                  |             |  | into native to D-bus format   |         | AllBaseAttributes           | ||
   | |Reset the system.      |             |  |                               |         |                             | ||
   | +-----------------------+             |  +-------------------------------+         +-----------------------------+ ||
   |                                       +----------------------------------------------------------------------------+|
   +---------------------------------------+-----------------------------------------------------------------------------+
```

## BIOS 透過 MCTP 以 BIOS 設定 PLDM 傳送資料

BIOS 應透過 Set BIOS table PLDM 命令，經由 MCTP 更新 BIOS 設定 —— BIOS 字串表（string table）、屬性名稱表（Attribute name table）、屬性值表（Attribute value table）。

RBC daemon 應根據使用者輸入的設定建立待處理（pending）BIOS 屬性清單，並發送 PendingAttributesUpdated 訊號。

PLDM daemon 應向 PendingAttributesUpdated 訊號註冊，而 RBC daemon 應在屬性值被 SetPendingAttributes、set attribute D-bus 呼叫修改時產生訊號。PLDM 應等待 PendingAttributesUpdated 訊號，並從 RBC daemon 取得待處理屬性的值。一旦待處理屬性值表不再有效或已在 BIOS 中更新，PLDM 就應刪除該表。每當收到新的屬性資料時，RBC 應清除待處理屬性清單。

RBC daemon 應在非揮發性儲存中保留 AllBaseAttributes 與 PendingAttributes 清單。PLDM daemon 應在非揮發性儲存中保留 BIOS 表。每當 BMC 重設時，RBC 與 PLDM 都應還原資料。

### BIOS 首次啟動

```text

   +--------------------------------------------------------------------------------------------------------------------+
   | +-----------------------+             +---------------------------------------------------------------------------+|
   | | BIOS                  |             |   BMC                                                                     ||
   | |                       |             |  +-------------------------------+         +-----------------------------+||
   | |                       |             |  |PLDM Interface (MCTP)          |         |RBC Daemon Manager           |||
   | |                       |             |  | -Responsible for send /recv   |         |-Responsible for handle BIOS |||
   | |                       |             |  |  data between BIOS and BMC    |         | configuration attributes.   |||
   | |                       |             |  |                               |         | -AllBaseAttributes          |||
   | |                       |             |  |                               |         | -Pending Attributes         |||
   | |                       |             |  |                               |         | -FactoryDefaultSetting      |||
   | |                       |             |  |                               |         | -BIOSPwdHashData            |||
   + +-----------------------+             |  +-------------------------------+         +-----------------------------+||
   | |Get the table info     |<---Req-/Res--> | Provide the table information |         |                             |||
   | | & Check table Tag     |             |  |                               |         |                             |||
   | |using GetBIOS table    |             |  |                               |         |                             |||
   | |                       |             |  |                               |         |                             |||
   | |Set BIOS string table  |<---Req-/Res--> | Get the string table          |         |                             |||
   | | via Set BIOS table    |             |  |                               |         |                             |||
   | |SetBIOSAttribute table |<---Req-/Res--> | Get the attributes table      |         |                             |||
   | |SetBIOSAttributeValue  |<---Req-/Res--> | Get the attributes value table|         |                             |||
   | |table via SetBIOStable |             |  |                               |         |                             |||
   | |table via SetBIOStable |             |  |                               |         |                             |||
   | |Init the BIOS config   |             |  | Send the attributes data in   |         |                             |||
   | |Based on value.        |             |  | native to D-bus format        |--dbus-->| AllBaseAttributes           |||
   | |via Set BIOS table     |             |  | by Setting AllBaseAttributes  |         |                             |||
   | |                       |             |  |                               |         |                             |||
   | | Continue the BIOS boot|             |  |                               |         |                             |||
   | +-----------------------+             |  +-------------------------------+         +-----------------------------+||
   |                                       +---------------------------------------------------------------------------+|
   +---------------------------------------+----------------------------------------------------------------------------+
```

### BIOS 重設

```text
   +--------------------------------------------------------------------------------------------------------------------+
   | +-----------------------+             +---------------------------------------------------------------------------+|
   | | BIOS                  |             |   BMC                                                                     ||
   | |                       |             |  +-------------------------------+         +-----------------------------+||
   | |                       |             |  |PLDM Interface (MCTP)          |         |RBC Daemon Manager           |||
   | |                       |             |  | -Responsible for send /recv   |         |-Responsible for handle BIOS |||
   | |                       |             |  |  data between BIOS and BMC    |         | configuration attributes.   |||
   | |                       |             |  |                               |         | -AllBaseAttributes          |||
   | |                       |             |  |                               |         | -Pending Attributes         |||
   | |                       |             |  |                               |         | -FactoryDefaultSetting      |||
   | |                       |             |  |                               |         | -BIOSPwdHashData            |||
   | +-----------------------+             |  +-------------------------------+         +-----------------------------+||
   | |Get the table info     |<---Req-/Res--> | Provide the table information |         |                             |||
   | | & Check table Tag     |             |  |                               |         |                             |||
   | |using GetBIOS table    |             |  |                               |         |                             |||
   | |                       |             |  |                               |         |                             |||
   | | Get the Pending table |<---Req-/Res--> | Provide the Pending attributes|         |                             |||
   | | using Get BIOS table  |             |  | table                         |         |                             |||
   | |                       |             |  |                               |         |                             |||
   | |SetBIOSAttributeValue  |<---Req-/Res--> | Get the attributes table      |         |                             |||
   | |table via SetBIOStable.|             |  | Delete the Pending attributes |         |                             |||
   | |Init the BIOS config   |             |  | table                         |         |                             |||
   | |Based on value.        |             |  |                               |         |                             |||
   | |                       |             |  | Send the attributes data in   |         |                             |||
   | |                       |             |  | native to D-bus format by     |--dbus-->| AllBaseAttributes           |||
   | | Reset the BIOS boot   |             |  | setting AllBaseAttributes     |         |                             |||
   | +-----------------------+             |  +-------------------------------+         +-----------------------------+||
   |                                       +---------------------------------------------------------------------------+|
   +---------------------------------------+----------------------------------------------------------------------------+

```

### 延遲更新模型中，BIOS 設定完整的 BIOS BMC 流程

```text
+----------------------------------------+                    +----------------------------------------+
|                BIOS                    |                    |                  BMC                   |
|                                        |                    |                                        |
|  +----------------------------------+  |                    | +------------------------------------+ |
|  | Send the BIOS capability  Support|  |--------KCS-------->| |1.Get the complete attributes data. | |
|  | Send the compressed BIOS file(or)|  |-MCTP/KCS/Redfish-->| |2.Validate and convert into         | |
|  | Send PLDM data via MCTP  (or)    |  |                    | |  native to D-bus format.           | |
|  | Send the Redfish host interface  |  |                    | |3.Expose the D-bus interface        | |
|  +----------------------------------+  |                    | +------------------------------------+ |
|                                        |                    |                                        |
|  +----------------------------------+  |                    |                                        |
|  | Get the file info & config status|   <-Get config status-|                                        |
|  | - Any config changed or not      |  |                    |                                        |
|  | - File checksum in BMC           |  |                    |                                        |
|  | - New attribute values exist     |  |                    |                                        |
|  +----------------------------------+  |                    |                                        |
|                                        |                    |  +-----------------------------------+ |
|  +----------------------------------+  |                    |  |                                   | |
|  | If new attribute value exist     |<-|-----------------------|  Send the new value attributes    | |
|  |           then                   |  |                    |  |  (Pending Attributes list)        | |
|  | Get & Update the BIOS variables  | -| -----+             |  |                                   | |
|  |                                  |  |      |             |  +-----------------------------------+ |
|  +---------------+------------------+  |      |             |                                        |
|                  |                     |      |             |                                        |
|                  YES                   |      |             |                                        |
|                  |                     |      |             |  +----------------------------------+  |
|   +--------------V------------------+  |      |             |  |                                  |  |
|   |  Send the updated data to BMC   |  |      |             |  | Update the BIOS attributes       |  |
|   |                                 |------------------------->| (BaseBIOSTable)                  |  |
|   +---------------------------------+  |      |             |  +----------------------------------+  |
|                                        |      |             |                                        |
|                                        |      |             |                                        |
|   +---------------------------------+  |      |             |                                        |
|   | Reset the BIOS for BIOS conf    |  |     NO             |                                        |
|   | update                          |  |      |             |                                        |
|   +---------------------------------+  |      |             |                                        |
|                                        |      |             |                                        |
|  +----------------------------------+  |      |             |                                        |
|  |  Continue the BIOS boot          | <-------+             |                                        |
|  +----------------------------------+  |                    |                                        |
+----------------------------------------+                    +----------------------------------------+
```

### 立即更新模型中，BIOS 設定完整的 BIOS BMC 流程

```text
+----------------------------------------+                    +----------------------------------------+
|                BIOS                    |                    |                  BMC                   |
|                                        |                    |                                        |
|  +----------------------------------+  |                    | +------------------------------------+ |
|  | Send the BIOS capability  Support|  |--------KCS-------->| |1.Get the complete attributes data  | |
|  | Send the compressed BIOS file(or)|  |-MCTP/KCS/Redfish-->| |2.Validate and convert into         | |
|  | Send PLDM data via MCTP  (or)    |  |                    | |  native to D-bus format.           | |
|  | Send the Redfish host interface  |  |                    | |3.Expose the D-bus interface        | |
|  +----------------------------------+  |                    | +------------------------------------+ |
|                                        |                    |                                        |
|                                        |                    | +------------------------------------+ |
|                                        |                    | |PLDM can send complete all tables   | |
|                                        |                    | | & send the native to D-bus format  | |
|                                        |                    | | by Setting AllBaseAttributes       | |
|                                        |                    | +------------------------------------+ |
|                                        |                    | +------------------------------------+ |
|                                        |                    | | If attribute  value changed then   | |
|                                        |                    | | Biosconfig send signal to PLDM.    | |
|                                        |                    | | PLDM read Pending attributes table | |
|                                        |                    | | PLDM send the message to the host  | |
|                                        |                    | | PLDM send the data to the host     | |
|                                        |                    | +------------------------------------+ |
|  +----------------------------------+  |                    | +------------------------------------+ |
|  | Read the new Attributes value    |<-|----------------------| Send new Pending attributes table  | |
|  |           and                    |  |                    | |                                    | |
|  | Update in the BIOS structure     |  |                    | +------------------------------------+ |
|  +---------------+------------------+  |                    |                                        |
|                  |                     |                    |                                        |
|                  YES                   |                    |                                        |
|                  |                     |                    | +-----------------------------------+  |
|   +--------------V------------------+  |                    | | Update the BIOS attributes data.  |  |
|   |  Send the updated data to BMC   |  |                    | | Discard the Pending attributes    |  |
|   |                                 |------------------------>|                        table      |  |
|   +---------------------------------+  |                    | +-----------------------------------+  |
|                                        |                    |                                        |
+----------------------------------------+                    +----------------------------------------+
```

每當 SetPendingAttributes 與 SetAttribute 方法被呼叫時，RBC 都應建立待處理（Pending）屬性清單並提出 PendingAttributesUpdated 訊號。RBC 應在 BMC 重設之後仍保留待處理屬性清單，而每當從 BIOS 收到新的 AllBaseBIOSTables 時，RBC 應清除待處理屬性清單。

### 用於遠端 BIOS 設定的 Redfish 介面

```text
 +-----------------------------------------------------------------------------------------------------------+
 | +-------------------------+             +----------------------------------------------------------------+|
 | | RBC Web tool - POSTMAN  |             |   BMC                                                          ||
 | |  (Please refer the      |             |  +-----------------------+       +---------------------------+ ||
 | |   below redfish example |             |  |Redfish Daemon         |       |RBC Daemon Manager         | ||
 | |   for each request)     |             |  |-Responsible for handle|       |-Parse Bios Data,convert to| ||
 | |                         |             |  |all Redfish request    |       | required format & return  | ||
 | |                         |             |  +-----------------------+       +---------------------------+ ||
 | +-------------------------+             |  +-----------------------+       +---------------------------+ ||
 | |                         |             |  |                       |       |                           | ||
 | |1.Get Current attributes |<---Req-/Res--> | Read BaseBIOSTable    |<-dbus-| BaseBIOSTable             | ||
 | |   name & value list     |             |  |                       |       |                           | ||
 | |                         |             |  |                       |       |                           | ||
 | |2.Get Attribute Registry |<---Req-/Res--> | Read BaseBIOSTable    |<-dbus-| BaseBIOSTable             | ||
 | |                         |             |  |                       |       |                           | ||
 | |3.Change BIOS Password   |<---Req-/Res--> | Call RBC D-bus Method |-dbus->| ChangePassword()          | ||
 | |                         |             |  |                       |       |                           | ||
 | |4.Reset To default       |<---Req-/Res--> | Set ResetBIOSSettings |-dbus->| ResetBiosSettings         | ||
 | |            settings     |             |  |                       |       |     -ResetFlag            | ||
 | |5.Update new BIOS setting|<---Req-/Res--->| Call RBC D-bus Method |-dbus->| SetAttribute()            | ||
 | |  (For single attribute) |             |  |                       |       |                           | ||
 | |                         |             |  |                       |       |                           | ||
 | |6.Get Pending attributes |<---Req-/Res--->| Get PendingAttributes |<-dbus-| PendingAttributes         | ||
 | |           list          |             |  |                       |       |                           | ||
 | |7.Update new BIOS setting|<---Req-/Res--->| Set PendingAttributes |<-dbus-| PendingAttributes         | ||
 | |           list          |             |  |                       |       |                           | ||
 | |  For multiple attributes|             |  |                       |       |                           | ||
 | +-------------------------+             |  +-----------------------+       +---------------------------+ ||
 |                                         +---------------------------------------------------------------+||
 +-----------------------------------------+-----------------------------------------------------------------+
```

1. 取得目前屬性的名稱與值清單：取得目前 BIOS 設定的屬性名稱與值配對清單。GET Method -
   `https://<BMC>/redfish/v1/Systems/system/Bios`

2. 取得 Attribute Registry：取得 BIOS 屬性的詳細資訊，例如目前值、支援的值、描述、Menupath、預設值。
   GET Method - `https://<BMC>/redfish/v1/Registries/Bios`

3. 變更 BIOS 密碼：ACTION -
   `https://<BMC>/redfish/v1/Systems/system/Bios/Actions/Bios.ChangePassword`

4. 重設為預設設定：ACTION -
   `https://<BMC>/redfish/v1/Systems/system/Bios/Actions/Bios.ResetBios`

5. 更新新的 BIOS 設定（單一屬性）：用於傳送特定屬性或屬性清單的新值。PATCH Method -
   `https://<BMC>/redfish/v1/Systems/system/Bios/Settings` 例如：屬性名稱與新值：`{ "DdrFreqLimit" : 2400 }`

6. 取得新的待處理值清單：用於取得新的待處理屬性清單。GET Method - `https://<BMC>/redfish/v1/Systems/system/Bios/Settings` - 僅在延遲模型中有效。對於立即更新模型，它會是空的。例如：屬性名稱與新值：`{ "DdrFreqLimit" : 2400, "QuietBoot", 0x1 }`

7. 更新新的 BIOS 設定（多個屬性）：用於傳送特定屬性或屬性清單的新值。PATCH Method -
   `https://<BMC>/redfish/v1/Systems/system/Bios/Settings` 例如：屬性名稱與新值清單：`{ "DdrFreqLimit" : 2400, "QuietBoot", 0x1 }`

## 曾考慮的替代方案

Redfish Host specification 的定義尚未完成，而 BIOS 支援也尚不可用。BIOS 變數超過 1000 個，將它們存放在 phosphor-settingsd 中並非最佳做法。

## 影響

BIOS 必須支援並遵循 RBC BIOS 設定流程。

## 測試

能夠透過 BMC 經由 LAN 變更 BIOS 設定。能夠透過 BMC 變更 BIOS 設定密碼。將使用 Redfish Service Validator 測試是否符合 Redfish 規範。
