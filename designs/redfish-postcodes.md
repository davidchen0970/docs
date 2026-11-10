# 透過 Redfish 記錄 BIOS POST Codes

Author: Terry Duncan

其他貢獻者：Jason Bills、Zhikui Ren

建立日期：2019 年 12 月 23 日

## 問題描述

BIOS Power-On Self-Test (POST) codes 目前在 DBUS 上揭露，但尚未透過
Redfish 揭露。本文說明如何利用 logging service，透過 Redfish
介面揭露 BIOS POST codes。

## 背景與參考

將使用標準的 Redfish LogService 與 LogEntry schemas 來揭露 BIOS
POST codes。並會在 LogServiceCollection 中新增一個額外的 log service（PostCodes）。

[LogService](https://redfish.dmtf.org/schemas/LogService_v1.xml) 範例條目：

```json
https://obmc/redfish/v1/Systems/system/LogServices/PostCodes
{
    "@odata.context": "/redfish/v1/$metadata#LogService.LogService",
    "@odata.id": "/redfish/v1/Systems/system/LogServices/PostCodes",
    "@odata.type": "#LogService.v1_1_0.LogService",
    "Actions": {
        "#LogService.ClearLog": {
            "target": "/redfish/v1/Systems/system/LogServices/PostCodes/Actions/LogService.ClearLog"
        }
    },
    "Description": "POST Code Log Service",
    "Entries": {
        "@odata.id": "/redfish/v1/Systems/system/LogServices/PostCodes/Entries"
    },
    "Id": "BIOS POST Code Log",
    "Name": "POST Code Log Service",
    "OverWritePolicy": "WrapsWhenFull"
}
```

事件（Events）將使用
[LogEntry](https://redfish.dmtf.org/schemas/LogEntry_v1.xml) schema 來揭露。

```json
https://obmc/redfish/v1/Systems/system/LogServices/PostCodes/Entries
{
    "@odata.context": "/redfish/v1/$metadata#LogEntryCollection.LogEntryCollection",
    "@odata.id": "/redfish/v1/Systems/system/LogServices/PostCodes/Entries",
    "@odata.type": "#LogEntryCollection.LogEntryCollection",
    "Description": "Collection of POST Code Log Entries",
    "Members": [
        {
            "@odata.context": "/redfish/v1/$metadata#LogEntry.LogEntry",
            "@odata.id": "/redfish/v1/Systems/system/LogServices/PostCodes/Entries/B1-03",
            "@odata.type": "#LogEntry.v1_4_0.LogEntry",
            "Created": "2019-12-06T14:10:30+00:00",
            "EntryType": "Event",
            "Id": "B1-03",
            "Message": "Boot Count: 4: TS Offset: 0.0033; POST Code: 0x43",
            "MessageArgs": [
                4,
                0.0033,
                "0x43"
            ],
            "MessageId": "OpenBMC.0.1.BiosPostCode",
            "Name": "POST Code Log Entry",
            "Severity": "OK"
        },
        ...
    ],
    "Members@odata.count": 10000,
    "Name": "BIOS POST Code Log Entries"
}
```

新的 [MessageRegistry](https://redfish.dmtf.org/schemas/MessageRegistry_v1.xml)
schema 條目定義了訊息的格式。

```json
https://obmc/redfish/v1/Registries/OpenBMC/OpenBMC
{
    "@Redfish.Copyright": "Copyright 2018 OpenBMC. All rights reserved.",
    "@odata.type": "#MessageRegistry.v1_0_0.MessageRegistry",
    "Description": "This registry defines the base messages for OpenBMC.",
    "Id": "OpenBMC.0.1.0",
    "Language": "en",
    "Messages": {
        "BiosPostCode": {
            "Description": "BIOS Power-On Self-Test Code received.",
            "Message": "Boot Count: %1: TS Offset: %2; POST Code: %3",
            "NumberOfArgs": 3,
            "ParamTypes": [
                "number",
                "number",
                "string"
            ],
            "Resolution": "None.",
            "Severity": "OK"
        },
        ...
    }
    "Name": "OpenBMC Message Registry",
    "OwningEntity": "OpenBMC",
    "RegistryPrefix": "OpenBMC",
    "RegistryVersion": "0.1.0"
}
```

## 需求

Redfish 介面必須符合 Redfish 規範，並通過 Redfish 合規性
測試。

Redfish 介面必須揭露自上次 BMC 重置以來在 DBUS 上追蹤的 POST
codes。

## 擬定設計

目前 OpenBMC 使用 xyz.openbmc_project.State.Boot.PostCode 服務在
DBus 上揭露 BIOS POST codes。現有的介面會追蹤過去 100 次 host 開機事件
的 POST codes，以及目前的開機週期索引（boot cycle index）。

```json
xyz.openbmc_project.State.Boot.PostCode
    GetPostCodes(q undefined) → [uint64] undefined
    CurrentBootCycleIndex = 1
    MaxBootCycleIndex = 100
```

GetPostCodes 方法使用開機週期索引來呼叫，以取得該開機週期的
codes。

```json
{
  "call": "GetPostCodes",
  "interface": "xyz.openbmc_project.State.Boot.PostCode",
  "obj": "/xyz/openbmc_project/State/Boot/PostCode",
  "result": [1, 2, 3, 4, 5, 6, 17, 50, 4, 173],
  "status": "ok"
}
```

現有的 DBus GetPostCodes 方法將保留以維持向下相容性。將新增一個新方法
GetPostCodesTS，在每個 POST code 之外附帶一個 ISO 格式、具微秒解析度的時間戳記。

```json
{
  "call": "GetPostCodesTS",
  "interface": "xyz.openbmc_project.State.Boot.PostCode",
  "obj": "/xyz/openbmc_project/State/Boot/PostCode",
  "result": [
    {"20191223T143052.632591", 1},
    {"20191223T143052.634083", 2},
    {"20191223T143053.928719", 3},
    {"20191223T143053.930168", 4},
    {"20191223T143054.512488", 5},
    {"20191223T143054.513945", 6},
    {"20191223T143054.960246", 17},
    {"20191223T143054.961723", 50},
    {"20191223T143055.368219", 4},
    {"20191223T143055.369680", 173}
  ],
  "status": "ok"
}
```

將實作 DBus DeleteAll 介面以移除條目。Redfish 的 ClearLog
action 會呼叫 DBus DeleteAll 介面。

```json
{
  "call": "DeleteAll",
  "interface": "xyz.openbmc_project.Collection.DeleteAll",
  "obj": "/xyz/openbmc_project/State/Boot/PostCode",
  "result": "POST Codes Cleared"
  "status": "ok"
}
```

## 曾考慮的替代方案

曾考慮使用現有的 DBus 方法而不揭露相關的時間戳記。在這種情況下，每個開機
週期可以使用單一 log 條目，揭露一個字串，包含與該開機週期相關的所有 POST codes。
大家認為時間戳記資料很有價值，即使這些資料可能會因 DBUS 而產生偏差。

也曾考慮透過 OEM 擴充而非使用 LogEntry schema 來揭露 POST codes。
不建議在 Redfish 中使用 OEM 擴充。如果 DMTF 為 POST codes 新增
schema，可以再重新檢視此決定。

## 影響

與現有的 DBUS 介面方法維持向下相容。預期追蹤時間戳記只會造成極小的
效能影響。

## 測試

將使用 Redfish Service Validator 測試 Redfish 合規性。
