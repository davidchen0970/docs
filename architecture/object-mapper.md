# ObjectMapper

`xyz.openbmc_project.ObjectMapper` 服務通常被簡稱為 mapper，是一個 OpenBMC
應用程式，它透過提供協助探索與關聯其他 D-Bus 物件的 API，試圖降低使用
D-Bus 的痛苦。

mapper 有兩大功能：

- [方法](#methods) - 提供與 D-Bus 探索相關的功能。
- [關聯](#associations) - 將兩個不同的物件互相關聯。

## 方法

官方的 YAML 介面定義可以在 [phosphor-dbus-interfaces][1] 中找到。

### GetObject

使用這個方法來找出實作某個物件路徑的服務及其介面。輸出是一個從服務名稱
到其所實作介面的對映（map）。也可以傳入一組選擇性的介面清單，將輸出限制
為實作那些特定介面的服務。

輸入：

- path：物件路徑
- param：interfaces - 一組選擇性的介面清單，用於限制搜尋範圍

輸出：

- 從服務名稱到其介面的對映

```bash
dbus-send --system --print-reply \
--dest=xyz.openbmc_project.ObjectMapper \
/xyz/openbmc_project/object_mapper \
xyz.openbmc_project.ObjectMapper.GetObject \
string:"/xyz/openbmc_project/sensors/voltage/ps1_input_voltage" array:string:

   array [
      dict entry(
         string "xyz.openbmc_project.Hwmon-1025936882.Hwmon1"
         array [
            string "xyz.openbmc_project.Sensor.Threshold.Critical"
            string "xyz.openbmc_project.Sensor.Threshold.Warning"
            string "xyz.openbmc_project.Sensor.Value"
         ]
      )
   ]
```

#### 使用範例

找出擁有目標物件路徑的服務名稱，以便傳入 get property 呼叫中。

### GetSubTree

使用這個方法來找出指定子樹中實作某個介面的物件、服務與介面。也可以傳入
一組選擇性的介面清單，將輸出限制為實作那些特定介面的服務。若沒有傳入任何
介面，則會回傳子樹中所有的物件／服務／介面。

輸入：

- param：subtree - 樹的根節點。使用 "/" 會搜尋整棵樹
- param：depth - 超過根節點之後要搜尋的樹最大深度。使用 0 表示搜尋全部
- param：interfaces - 一組選擇性的介面清單，用於限制搜尋範圍

輸出：

- 從物件路徑對映到「從服務名稱到其介面」的對映

```bash
dbus-send --system --print-reply \
--dest=xyz.openbmc_project.ObjectMapper \
/xyz/openbmc_project/object_mapper \
xyz.openbmc_project.ObjectMapper.GetSubTree \
string:"/" int32:0 array:string:"xyz.openbmc_project.Sensor.Threshold.Warning"

   array [
      dict entry(
         string "/xyz/openbmc_project/sensors/current/ps0_output_current"
         array [
            dict entry(
               string "xyz.openbmc_project.Hwmon-1040041051.Hwmon1"
               array [
                  string "xyz.openbmc_project.Sensor.Threshold.Critical"
                  string "xyz.openbmc_project.Sensor.Threshold.Warning"
                  string "xyz.openbmc_project.Sensor.Value"
               ]
            )
         ]
      )
      dict entry(
         string "/xyz/openbmc_project/sensors/current/ps1_output_current"
         array [
            dict entry(
               string "xyz.openbmc_project.Hwmon-1025936882.Hwmon1"
               array [
                  string "xyz.openbmc_project.Sensor.Threshold.Critical"
                  string "xyz.openbmc_project.Sensor.Threshold.Warning"
                  string "xyz.openbmc_project.Sensor.Value"
               ]
            )
         ]
      )
...

```

#### 使用範例

找出實作某個特定介面所有的物件路徑與服務。

### GetAssociatedSubTree

使用這個方法來找出指定子樹中，實作某個介面且為輸入之 associationPath
某一端點的物件、服務與介面。也可以傳入一組選擇性的介面清單，將輸出限制為
實作那些特定介面的服務。若沒有傳入任何介面，則會回傳子樹及關聯端點中所有的
物件／服務／介面。

輸入：

- param：associationPath - 要用來尋找關聯端點的路徑。
- param：subtree - 樹的根節點。使用 "/" 會搜尋整棵樹
- param：depth - 超過根節點之後要搜尋的樹最大深度。使用 0 表示搜尋全部
- param：interfaces - 一組選擇性的介面清單，用於限制搜尋範圍

輸出：

- 位於關聯端點中，從物件路徑對映到「從服務名稱到其介面」的對映

```bash
dbus-send --system --print-reply \
--dest=xyz.openbmc_project.ObjectMapper \
/xyz/openbmc_project/object_mapper \
xyz.openbmc_project.ObjectMapper.GetAssociatedSubTree \
objpath:"/${ASSOCIATED_PATH}" \
objpath:"/" int32:0 array:string:"xyz.openbmc_project.Sensor.Threshold.Warning"

   array [
      dict entry(
         string "/xyz/openbmc_project/sensors/current/ps0_output_current"
         array [
            dict entry(
               string "xyz.openbmc_project.Hwmon-1040041051.Hwmon1"
               array [
                  string "xyz.openbmc_project.Sensor.Threshold.Critical"
                  string "xyz.openbmc_project.Sensor.Threshold.Warning"
                  string "xyz.openbmc_project.Sensor.Value"
               ]
            )
         ]
      )
      dict entry(
         string "/xyz/openbmc_project/sensors/current/ps1_output_current"
         array [
            dict entry(
               string "xyz.openbmc_project.Hwmon-1025936882.Hwmon1"
               array [
                  string "xyz.openbmc_project.Sensor.Threshold.Critical"
                  string "xyz.openbmc_project.Sensor.Threshold.Warning"
                  string "xyz.openbmc_project.Sensor.Value"
               ]
            )
         ]
      )
...


# All output must be in the association endpoints
busctl get-property  xyz.openbmc_project.ObjectMapper \
   /${ASSOCIATED_PATH} \
  xyz.openbmc_project.Association endpoints
as N "/xyz/openbmc_project/sensors/current/ps0_output_current" \
  "/xyz/openbmc_project/sensors/current/ps1_output_current" \
  ...
```

#### 使用範例

找出實作某個特定介面、且為輸入之 associationPath 端點的所有物件路徑與
服務。

### GetAssociatedSubTreeById

使用這個方法來找出指定子樹中，實作某些介面且端點以輸入的 `id` 結尾的
物件、服務與介面。也可以傳入一組選擇性的介面清單，將輸出限制為實作那些
特定介面的服務。若沒有傳入任何介面，則會回傳子樹及關聯端點中所有的
物件／服務／介面。

輸入：

- param：id - dbus 路徑的葉節點名稱，可在系統中唯一識別某個特定的元件或
  實體。
- param：objectPath - 要為其取得結果的物件路徑。
- param：subtreeInterfaces - 用於限制搜尋範圍的介面清單
- param：association - 端點關聯。
- param：endpointInterfaces - 用於篩選關聯端點路徑的介面陣列。

輸出：

- 位於以 `id` 結尾之關聯端點中，從物件路徑對映到「從服務名稱到其介面」的
  對映

```bash
ID="chassis"
ASSOCIATION="powered_by"
dbus-send --system --print-reply \
--dest=xyz.openbmc_project.ObjectMapper \
/xyz/openbmc_project/object_mapper \
xyz.openbmc_project.ObjectMapper.GetAssociatedSubTreeById \
string:"${ID}" string:"/xyz/openbmc_project/inventory" \
array:string:"xyz.openbmc_project.Inventory.Item.Chassis" \
string:"${ASSOCIATION}" \
array:string:"xyz.openbmc_project.Inventory.Item.PowerSupply"

   array [
      dict entry(
         string "/xyz/openbmc_project/inventory/system/chassis/motherboard/powersupply0"
         array [
            dict entry(
               string "xyz.openbmc_project.Inventory.Manager"
               array [
                   ...
                   string "xyz.openbmc_project.Inventory.Item"
                   string "xyz.openbmc_project.Inventory.Item.PowerSupply"
                   ...
               ]
            )
         ]
      )
      dict entry(
         string "/xyz/openbmc_project/inventory/system/chassis/motherboard/powersupply1"
         array [
            dict entry(
               string "xyz.openbmc_project.Inventory.Manager"
               array [
                   ...
                   string "xyz.openbmc_project.Inventory.Item"
                   string "xyz.openbmc_project.Inventory.Item.PowerSupply"
                   ...
               ]
            )
         ]
      )
      ....
...

# All output must be in the association endpoints that ends with the given `id`
CHASSIS_PATH=/xyz/openbmc_project/inventory/system/chassis
busctl get-property  xyz.openbmc_project.ObjectMapper \
   /xyz/openbmc_project/inventory/system/chassis/${ASSOCIATION} \
  xyz.openbmc_project.Association endpoints
as N "/xyz/openbmc_project/inventory/system/chassis/motherboard/powersupply0" \
"/xyz/openbmc_project/inventory/system/chassis/motherboard/powersupply1" \
 ...
```

#### 使用範例

找出實作某個特定介面、且為輸入之 associationPath 端點的所有物件路徑與
服務。

### GetSubTreePaths

這與 GetSubTree 相同，但只回傳物件路徑

輸入：

- param：subtree - 樹的根節點。使用 "/" 會搜尋整棵樹
- param：depth - 超過根節點之後要搜尋的樹最大深度。使用 0 表示搜尋全部
- param：interfaces - 一組選擇性的介面清單，用於限制搜尋範圍

輸出：

- 該子樹中物件路徑的陣列

```bash
dbus-send --system --print-reply \
--dest=xyz.openbmc_project.ObjectMapper \
/xyz/openbmc_project/object_mapper \
xyz.openbmc_project.ObjectMapper.GetSubTreePaths \
string:"/" int32:0 array:string:"xyz.openbmc_project.Sensor.Threshold.Warning"

   array [
      string "/xyz/openbmc_project/sensors/current/ps0_output_current"
      string "/xyz/openbmc_project/sensors/current/ps1_output_current"
      string "/xyz/openbmc_project/sensors/power/ps0_input_power"
...
   ]
```

#### 使用範例

找出實作某個特定介面所有的物件路徑。

### GetAssociatedSubTreePaths

這與 GetAssociatedSubTreePaths 相同，但只回傳物件路徑

輸入：

- param：associationPath - 要用來尋找關聯端點的路徑。
- param：subtree - 樹的根節點。使用 "/" 會搜尋整棵樹
- param：depth - 超過根節點之後要搜尋的樹最大深度。使用 0 表示搜尋全部
- param：interfaces - 一組選擇性的介面清單，用於限制搜尋範圍

輸出：

- 該子樹中、位於關聯端點內之物件路徑的陣列

```bash
dbus-send --system --print-reply \
--dest=xyz.openbmc_project.ObjectMapper \
/xyz/openbmc_project/object_mapper \
xyz.openbmc_project.ObjectMapper.GetAssociatedSubTreePaths \
objpath:"/${ASSOCIATED_PATH}" objpath:"/" int32:0 array:string:"xyz.openbmc_project.Sensor.Threshold.Warning"

   array [
      string "/xyz/openbmc_project/sensors/current/ps0_output_current"
      string "/xyz/openbmc_project/sensors/current/ps1_output_current"
      string "/xyz/openbmc_project/sensors/power/ps0_input_power"
...
   ]

# All output must be in the association endpoints
busctl get-property  xyz.openbmc_project.ObjectMapper \
   /${ASSOCIATED_PATH} \
  xyz.openbmc_project.Association endpoints
as N "/xyz/openbmc_project/sensors/current/ps0_output_current" \
  "/xyz/openbmc_project/sensors/current/ps1_output_current" \
  "/xyz/openbmc_project/sensors/power/ps0_input_power" \
  ...
```

#### 使用範例

找出實作某個特定介面、且為輸入之 associationPath 端點的所有物件路徑。

### GetAssociatedSubTreePathsById

這與 GetAssociatedSubTreePathsById 相同，但只回傳物件路徑

輸入：

- param：id - dbus 路徑的葉節點名稱，可在系統中唯一識別某個特定的元件或
  實體。
- param：objectPath - 要為其取得結果的物件路徑。
- param：subtreeInterfaces - 用於限制搜尋範圍的介面清單
- param：association - 端點關聯。
- param：endpointInterfaces - 用於篩選關聯端點路徑的介面陣列。

輸出：

- 位於以 `id` 結尾之關聯端點中，從物件路徑對映到「從服務名稱到其介面」的
  對映

```bash
ID="chassis"
ASSOCIATION="powered_by"
dbus-send --system --print-reply \
--dest=xyz.openbmc_project.ObjectMapper \
/xyz/openbmc_project/object_mapper \
xyz.openbmc_project.ObjectMapper.GetAssociatedSubTreePathsById \
string:"${ID}" string:"/xyz/openbmc_project/inventory" \
array:string:"xyz.openbmc_project.Inventory.Item.Chassis" \
string:"${ASSOCIATION}" \
array:string:"xyz.openbmc_project.Inventory.Item.PowerSupply"

   array [
      string "/xyz/openbmc_project/inventory/system/chassis/motherboard/powersupply0"
      string "/xyz/openbmc_project/inventory/system/chassis/motherboard/powersupply1"
       ...
   ]
...

# All output must be in the association endpoints that ends with the given `id`
CHASSIS_PATH=/xyz/openbmc_project/inventory/system/chassis
busctl get-property  xyz.openbmc_project.ObjectMapper \
   /xyz/openbmc_project/inventory/system/chassis/${ASSOCIATION} \
  xyz.openbmc_project.Association endpoints
as N "/xyz/openbmc_project/inventory/system/chassis/motherboard/powersupply0" \
"/xyz/openbmc_project/inventory/system/chassis/motherboard/powersupply1" \
 ...
```

#### 使用範例

找出實作某個特定介面、且為輸入之 associationPath 端點的所有物件路徑。

### GetAncestors

使用這個方法來找出某個物件的所有祖先中，實作某個特定介面的祖先。若沒有
傳入任何介面，則會回傳所有祖先的路徑／服務／介面。

輸入：

- param：path - 要尋找其祖先的物件路徑
- param：interfaces - 一組選擇性的介面清單，用於限制搜尋範圍

輸出：

- 從物件路徑對映到「從服務名稱到其介面」的對映

```bash

dbus-send --system --print-reply \
--dest=xyz.openbmc_project.ObjectMapper \
/xyz/openbmc_project/object_mapper \
xyz.openbmc_project.ObjectMapper.GetAncestors \
string:"/xyz/openbmc_project/inventory/system" array:string:

   array [
      dict entry(
         string "/xyz/openbmc_project"
         array [
            dict entry(
               string "xyz.openbmc_project.ObjectMapper"
               array [
                  string "org.freedesktop.DBus.ObjectManager"
               ]
            )
         ]
      )
      dict entry(
         string "/xyz/openbmc_project/inventory"
         array [
            dict entry(
               string "xyz.openbmc_project.Inventory.Manager"
               array [
                  string "xyz.openbmc_project.Inventory.Manager"
                  string "org.freedesktop.DBus.ObjectManager"
               ]
            )
         ]
      )
      dict entry(
         string "/"
         array [
            dict entry(
               string "xyz.openbmc_project.Settings"
               array [
                  string "org.freedesktop.DBus.ObjectManager"
               ]
            )
         ]
      )
   ]
```

#### 使用範例

找出實作某個特定介面的父物件。

## 關聯

關聯是由 mapper 建立的特殊 D-Bus 物件，用來讓兩個物件互相關聯。要發生
這種關聯，必須由某個應用程式實作 `xyz.openbmc_project.Association.Definitions`
介面，然後當需要建立關聯時，需要寫入該介面上一個名為 `Associations` 的
屬性。

這個 `Associations` 屬性是一個由元組（tuple）組成的陣列，其形式為：

```text
[forward, reverse, object path]
```

- forward：這是正向關聯物件的名稱
- reverse：這是反向關聯物件的名稱
- object path：這是要與之關聯的另一個物件

當某個物件（例如物件路徑為 `pathA`）使用下列值時：

```json
["foo", "bar", "pathB"]
```

mapper 會建立 2 個新物件：

1. `pathA/foo`
2. `pathB/bar`

在每一個這樣的物件上，都會實作 `xyz.openbmc_project.Association` 介面，
它有一個名為 `endpoints` 的單一屬性。這個屬性是一個陣列，保存了指向關聯
另一端的物件路徑。

所以 `pathA/foo->endpoints` 會包含 `pathB`，而 `pathB/bar->endpoints`
會包含 `pathA`。

如果另一個物件（假設是 `pathC`）也與 `pathB` 有關聯，那麼 `pathB` 的
endpoints 屬性中就會新增第二個條目 `pathC`。

這些新物件的生命週期會與被關聯的物件一致。舉例來說，如果 `pathA` 被刪除，
那麼 `pathA/foo` 也會被刪除，而 `pathA` 會從 `pathB/bar` 的 endpoints
屬性中被移除。如果那是該屬性中最後一個條目，那麼 `pathB/bar` 也會被刪除。
此外，如果端點路徑（在此例中是 `pathB`）從 D-Bus 上被移除，mapper 會先
移除這 2 個關聯路徑，直到 `pathB` 再次出現為止。

注意：關聯定義介面原本的名稱是 `org.openbmc.Associations`。雖然 mapper
暫時仍支援這個介面，但新程式碼應該使用 `xyz` 版本。

可以使用 mappertool 在系統上即時檢視關聯。它可以全部傾印，或依據名稱或
類型進行篩選：

```bash
mappertool assocs -n Altitude
/xyz/openbmc_project/inventory/system/chassis  | all_sensors
/xyz/openbmc_project/sensors/altitude/Altitude | chassis
```

### 使用範例

將一筆錯誤日誌與造成它的庫存項目（inventory item）關聯起來。

```text
# Error log
"/xyz/openbmc_project/logging/entry/3": {
...
 "associations": [
   [
     "callout",
     "fault",
     "/xyz/openbmc_project/inventory/system/chassis/motherboard/powersupply0"
   ]
 ]
}

# Newly created forward association object
"/xyz/openbmc_project/logging/entry/3/callout": {
    "endpoints": [
    "/xyz/openbmc_project/inventory/system/chassis/motherboard/powersupply0"
    ]
}

# Newly created reverse association object
"/xyz/openbmc_project/inventory/system/chassis/motherboard/powersupply0/fault": {
    "endpoints": [
    "/xyz/openbmc_project/logging/entry/3"
    ]
}

```

[1]:
  https://github.com/openbmc/phosphor-dbus-interfaces/blob/master/yaml/xyz/openbmc_project/ObjectMapper.interface.yaml
