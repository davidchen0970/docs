# bmcweb 中的 Redfish 事件記錄

本指南旨在協助開發人員為 bmcweb 的 Redfish 事件記錄新增訊息。

Redfish Message Objects 可以有多種不同的呈現方式。在 bmcweb 中，我們選擇使用
Message Registries，並以 MessageId 與 MessageArgs 欄位來引用其中的 Message
Objects。

更多詳細資訊請參考
[Redfish Specification](http://redfish.dmtf.org/schemas/DSP0266_1.6.1.html)。

## Message Registries

要為 Redfish 事件記錄新增訊息，第一步是在某個 Message Registry 中尋找或新增一則
合適的訊息。

bmcweb 的 Message Registries 位於原始碼中的
"\redfish-core\include\registries"，也可以透過 Redfish 的
"/redfish/v1/Registries" 來檢視。

若已存在合適的訊息，請記錄下：

1. Message Object 的標題（必須作為 MessageId 中的 MessageKey）。
2. 「Message」欄位中標註為 "%x" 的 Args（MessageArgs 需要用到）。

若不存在合適的訊息，可以依照下列方式新增訊息：

1. 為新訊息選擇一個 Message Registry
2. 選擇一個 MessageKey 作為新 Message 條目的標題
3. 插入一個新的 Message 條目（標題最好依字母順序排列）
4. 加入 `description` 與 `resolution`
5. 將 `severity` 設為 "Critical"、"Warning" 或 "OK"
6. 定義 `message` 並加入所需的 args
7. 設定 `numberOfArgs` 與 `paramTypes` 以符合訊息的 args

## 記錄訊息

記錄訊息的方式是提供一組 Redfish MessageId 以及對應的 MessageArgs 給 bmcweb。

Redfish MessageId 的格式如下：

`RegistryName.MajorVersion.MinorVersion.MessageKey`

bmcweb 會在指定的 Message Registry 中搜尋該 MessageKey，使用 MessageArgs 組出
最終的訊息，並顯示在事件記錄中。

### 基於 journal 的 Redfish 記錄

journal 是目前用來記錄 Redfish Messages 的機制。bmcweb 會在 journal 的中繼資料
中尋找兩個欄位：

- `REDFISH_MESSAGE_ID`：一個存放 MessageId 的字串
- `REDFISH_MESSAGE_ARGS`：一個存放以逗號分隔的 args 清單的字串

這些欄位可以使用 `phosphor::logging::entry()` 指令加入 journal 條目中，也可以
直接使用 `sd_journal_send()` 指令加入。

### 範例

#### 記錄 ResourceCreated 事件

[Resource Event Message Registry](https://redfish.dmtf.org/registries/ResourceEvent.1.0.0.json)
中含有 ResourceCreated 訊息：

```json
{
    "ResourceCreated": {
        "Description": "Indicates that all conditions of a successful
        creation operation have been met.",
        "Message": "The resource has been created successfully.",
        "NumberOfArgs": 0,
        "Resolution": "None",
        "Severity": "OK"
    }
},
```

由於沒有任何參數，因此不需要 MessageArgs，這則訊息可以依照下列方式記錄到
journal 中：

```cpp
phosphor::logging::log<log::level>(
        "journal text",
        phosphor::logging::entry("REDFISH_MESSAGE_ID=%s",
        "ResourceEvent.1.0.ResourceCreated"));
```

或

```cpp
sd_journal_send("MESSAGE=%s", "journal text", "PRIORITY=%i", <LOG_LEVEL>,
                "REDFISH_MESSAGE_ID=%s",
                "ResourceEvent.1.0.ResourceCreated", NULL);
```

#### 記錄 ResourceErrorThresholdExceeded 事件

[Resource Event Message Registry](https://redfish.dmtf.org/registries/ResourceEvent.1.0.0.json)
中含有 ResourceErrorThresholdExceeded 訊息：

```json
{
    "ResourceErrorThresholdExceeded": {
        "Description": "The resource property %1 has exceeded error
        threshold of value %2. Examples would be drive IO errors, or
        network link errors.",
        "Message": "The resource property %1 has exceeded error
        threshold of value %2.",
        "NumberOfArgs": 2,
        "ParamTypes": [
            "string",
            "value"
        ],
        "Resolution": "None.",
        "Severity": "Critical"
    }
},
```

這則訊息有兩個參數，`%1` 與 `%2`，由 `"NumberOfArgs"` 欄位可以看出。這些參數
的意義與型別來自 Message Registry 中的 `"Message"` 與 `"ParamTypes"` 欄位。

在這個範例中，`%1` 是一個存放屬性名稱的字串，`%2` 則是一個存放閾值的數字。

參數會以逗號分隔的 MessageArgs 清單來填入，因此這則訊息可以依照下列方式記錄
到 journal 中：

```cpp
phosphor::logging::log<log::level>(
        "journal text",
        phosphor::logging::entry("REDFISH_MESSAGE_ID=%s",
        "ResourceEvent.1.0.ResourceErrorThresholdExceeded"),
        phosphor::logging::entry("REDFISH_MESSAGE_ARGS=%s,%d",
        "Property Name", propertyValue));
```

或

```cpp
sd_journal_send("MESSAGE=%s", "journal text", "PRIORITY=%i", <LOG_LEVEL>,
                "REDFISH_MESSAGE_ID=%s",
                "ResourceEvent.1.0.ResourceErrorThresholdExceeded",
                "REDFISH_MESSAGE_ARGS=%s,%d", "Property Name",
                propertyValue, NULL);
```
