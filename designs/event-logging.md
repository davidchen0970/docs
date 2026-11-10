# 錯誤與事件記錄

作者：[Patrick Williams][patrick-email] `<stwcx>`

[patrick-email]: mailto:patrick@stwcx.xyz

其他貢獻者：

- 建立日期：2024 年 5 月 16 日
- 更新日期：2026 年 8 月 7 日

## 問題描述

目前 OpenBMC 程式碼堆疊沒有一套一致、端對端的錯誤與事件回報設計。現有兩套不同的實作，一套主要使用 phosphor-logging，另一套使用 rsyslog，兩者都存在完整解決方案應該處理的缺口。本提案旨在提供一套端對端的設計，同時處理錯誤與追蹤事件，以利以自動化且可維護的方式對系統進行外部管理。

## 背景與參考資料

### Redfish LogEntry 與 Message Registry

在 Redfish 中，[`LogEntry` schema][LogEntry] 用於各種可被視為「記錄」的項目，但在 OpenBMC 中的一種用途是用來對應 IPMI 的「System Event Log（SEL）」。

IPMI SEL 是 BMC 可以收集錯誤與事件的地方，這些錯誤與事件有時來自其他實體，例如 BIOS。這類記錄的例子可能是「DIMM-A0 發生不可更正的 ECC 錯誤」或「系統開機成功」。這些 SEL 記錄以人類可讀的字串呈現，無論是透過 OEM SEL 設計原生提供，或是透過 `ipmitool` 之類的工具，這些字串通常因各系統或各製造商而異，並且可能隨著 BMC 或韌體更新而改變，因此很難圍繞它們建立自動化工具。兩家不同的供應商可能會使用不同的字串來表示超過溫度臨界值，例如 ["temperature threshold exceeded"][HPE-Example] 與 ["Temperature #0x30 Upper Critical going high"][Oracle-Example]。此外，IPMI 也沒有任何機制可以詢問機器「你可能會產生哪些 SEL」。

為了同時解決這個問題的兩個層面，也就是可列舉的事件清單與版本控管，Redfish 提供了 Message Registries。一個 message registry 是一套經過版本控管的集合，包含系統可能產生的所有錯誤事件，以及關於這些事件如何被解析並呈現給使用者的提示。DMTF 的一份[參考資料][Registry-Example]給出以下範例：

```json
{
  "@odata.type": "#MessageRegistry.v1_0_0.MessageRegistry",
  "Id": "Alert.1.0.0",
  "RegistryPrefix": "Alert",
  "RegistryVersion": "1.0.0",
  "Messages": {
    "LanDisconnect": {
      "Description": "A LAN Disconnect on %1 was detected on system %2.",
      "Message": "A LAN Disconnect on %1 was detected on system %2.",
      "Severity": "Warning",
      "NumberOfArgs": 2,
      "Resolution": "None"
    }
  }
}
```

這個範例定義了一個事件 `Alert.1.0.LanDisconnect`，它可以記錄網路裝置的連線中斷狀態，並包含受影響裝置與系統的佔位符。當此事件發生時，可能會記錄下一筆 `LogEntry`，其中包含類似以下的內容：

```json
{
  "Message": "A LAN Disconnect on EthernetInterface 1 was detected on system /redfish/v1/Systems/1.",
  "MessageId": "Alert.1.0.LanDisconnect",
  "MessageArgs": ["EthernetInterface 1", "/redfish/v1/Systems/1"]
}
```

`Message` 包含一個人類可讀的字串，這個字串是將 `MessageArgs` 套用到 registry 中 `Message` 欄位的佔位符上所產生。系統管理軟體可以依賴 message registry（可透過 `LogEntry` 中的 `MessageId` 欄位引用）與 `MessageArgs`，而不必為了反應該事件而執行字串處理。

在 OpenBMC 內部，目前針對此 Redfish 功能存在一個[有限的設計][existing-design]，它要求在所有想要記錄這些事件的應用程式中插入特別構造的 Redfish 專用記錄訊息，這使得所有應用程式都與 Redfish 實作緊密耦合。此外也觀察到，這些[字串][app-example]在使用時，往往與 `bmcweb` 所公告的 [message registry][obmc-registry-example] 不同步。部分維護者已拒絕將新的 Redfish 專用記錄訊息加入他們的應用程式。

[LogEntry]:
  https://github.com/openbmc/bmcweb/blob/de0c960c4262169ea92a4b852dd5ebbe3810bf00/redfish-core/schema/dmtf/json-schema/LogEntry.v1_16_0.json
[HPE-Example]:
  https://support.hpe.com/hpesc/public/docDisplay?docId=sd00002092en_us&docLocale=en_US&page=GUID-D7147C7F-2016-0901-06CE-000000000422.html
[Oracle-Example]:
  https://docs.oracle.com/cd/E19464-01/820-6850-11/IPMItool.html#50602039_63068
[Registry-Example]:
  https://www.dmtf.org/sites/default/files/Redfish%20School%20-%20Events_0.pdf
[existing-design]:
  https://github.com/openbmc/docs/blob/master/architecture/redfish-logging-in-bmcweb.md
[app-example]:
  https://github.com/openbmc/phosphor-post-code-manager/blob/f2da78deb3a105c7270f74d9d747c77f0feaae2c/src/post_code.cpp#L143
[obmc-registry-example]:
  https://github.com/openbmc/bmcweb/blob/4ba5be51e3fcbeed49a6a312b4e6b2f1ea7447ba/redfish-core/include/registries/openbmc.json#L5

### 現有的 phosphor-logging 實作

**注意**：雖然本節使用了「exception（例外）」一詞，但現有（以及提案中）的型別可以被停用例外功能的應用程式與執行環境使用。它們被稱為「例外」是因為它們確實繼承自 `std::exception`，而且 `sdbusplus` 綁定支援將它們用於例外處理。

`sdbusplus` 綁定具備定義新的 C++ 例外型別的能力，這些型別可由 DBus 伺服器拋出，並轉換為給客戶端的錯誤回應。`phosphor-logging` 擴充了這個功能，額外加入了與記錄類型相關的中繼資料。請參考以下錯誤定義與使用的範例。

`sdbusplus` 錯誤綁定定義（位於 `xyz/openbmc_project/Certs.errors.yaml`）：

```yaml
- name: InvalidCertificate
  description: Invalid certificate file.
```

`phosphor-logging` 中繼資料定義（位於 `xyz/openbmc_project/Certs.metadata.yaml`）：

```yaml
- name: InvalidCertificate
  meta:
    - str: "REASON=%s"
      type: string
```

回報錯誤的應用程式程式碼：

```cpp
elog<InvalidCertificate>(Reason("Invalid certificate file format"));
// or
report<InvalidCertificate>(Reason("Existing certificate file is corrupted"));
```

在這個範例中，定義了一個名為 `xyz.openbmc_project.Certs.Error.InvalidCertificate` 的錯誤，它可以作為 DBus 回應在應用程式之間傳送。`InvalidCertificate` 預期會帶有一個名為 `REASON` 的額外中繼資料，型別為字串。兩個 API `elog` 與 `report` 的行為略有不同：`elog` 會拋出一個例外，可能導致錯誤的 DBus 結果，或在應用程式的其他地方被處理；而 `report` 會直接將事件傳送至 `phosphor-logging` 的守護程序進行記錄。這兩種呼叫的副作用是，中繼資料會被插入 `systemd` journal。

當錯誤被傳送至 `phosphor-logging` 守護程序時，它會：

1. 回溯搜尋 journal 中與該事件相關、已記錄的中繼資料（這是一個相對緩慢的操作）。
2. 建立一個 [`xyz.openbmc_project.Logging.Entry`][Logging-Entry] DBus 物件，帶有從 journal 擷取出來的相關資料。
3. 持久化儲存該物件的序列化版本。

在 `bmcweb` 內部，支援將 `phosphor-logging` 公告的 `xyz.openbmc_project.Logging.Entry` 物件轉換為 Redfish `LogEntries`，但這項支援並未引用 Message Registry。這使得這些事件對於系統管理軟體的消費實用性有限，因為它無法得知所有的事件類型，只能依靠（手寫程式碼的）正規表示式從 `LogEntry` 的 `Message` 欄位擷取任何資訊。此外，隨著內部 OpenBMC 錯誤回報結構、中繼資料或訊息字串的演進，這些正規表示式很可能會隨著時間過時。

[Logging-Entry]:
  https://github.com/openbmc/phosphor-dbus-interfaces/blob/9012243e543abdc5851b7e878c17c991b2a2a8b7/yaml/xyz/openbmc_project/Logging/Entry.interface.yaml#L1

### 現況的問題

- 有兩套不同的錯誤記錄實作，兩者都不完整，也未完全為維護者所接受。這些實作也未涵蓋追蹤事件。

- `REDFISH_MESSAGE_ID` 記錄方式會導致 Redfish Message Registry 與回報應用程式之間出現差異。它也要求每個應用程式都必須「知道 Redfish」，這限制了應用程式與外部管理介面之間的解耦。這也為在不同管理介面（例如同帶內 IPMI 與 PLDM）中回報錯誤留下了缺口。此做法也未在編譯期確保適當收集中繼資料，可能導致產生程式碼與 message registry 定義不同步。

- `phosphor-logging` 做法未在編譯期確保適當收集中繼資料，並且在每次回報錯誤時都需要守護程序對 `systemd` journal 進行昂貴的處理，這限制了可擴展性。

- 用於錯誤回報的 `sdbusplus` 綁定目前無法處理錯誤在 DBus 伺服器與客戶端之間的無損傳輸。

- 類似的應用程式對於相同的錯誤情境可能產生不同的 Redfish `LogEntry`。這已在 `dbus-sensors`、`phosphor-hwmon`、`phosphor-virtual-sensor` 與 `phosphor-health-monitor` 之間的感測器臨界值超過事件中觀察到。其中一個原因是存在兩種不同的錯誤回報方式，以及維護者之間對於偏好方式存在分歧。

## 需求

- 在 BMC 上執行的應用程式必須能夠回報錯誤與失敗，這些錯誤與失敗會被持久化，並可透過 Redfish 等標準供外部系統管理使用。
  - 這些錯誤必須是結構化的、經過版本控管的，並且 BMC 所能建立的完整錯誤集合應在 BMC 映像檔的建置期即可取得。
  - BMC 所能建立的錯誤集合，必須能夠被轉換為相關的資料集，例如 Redfish Message Registries。
    - 對於 Redfish，此轉換必須符合 Redfish 標準需求，例如符合語意化版本控管的期待。
    - 對於 Redfish，此轉換應允許將內部定義的事件對應到既有的 Redfish Message Registries，以獲得更廣泛的相容性。
    - 對於 Redfish，實作也必須支援 EventService 的推送回報機制。
  - BMC 回報的錯誤應包含足夠的資訊，以便為這些失敗對系統進行服務，無論是透過人工或自動化（取決於各系統的個別需求）。

- 在 BMC 上執行的應用程式應能夠回報與系統管理及／或除錯相關的重要追蹤事件，例如系統成功達到執行中狀態。
  - 所有與錯誤相關的需求同樣適用於追蹤事件。
  - 實作必須具備一套機制，讓供應商能夠停用特定的追蹤事件，以符合其自身的系統設計需求。

- 在 BMC 上執行的應用程式應能夠判斷先前回報的錯誤何時不再相關，並將其標記為「已解決」，同時為未來的用途（例如除錯）維護持久化的記錄。

- BMC 應提供一套機制，讓伺服器內受管理的實體能夠回報它們自己的錯誤與事件。受管理實體的例子包括韌體（例如 BIOS）以及衛星管理控制器。

- BMC 上的實作應能擴展至至少 [10,000][error-discussion] 筆錯誤與事件，且不影響 BMC 或受管理系統的效能。

- 實作應提供一套機制，允許對錯誤與事件定義（以及所產生的產出物，例如 Redfish Message Registry）進行 OEM 或供應商擴充，供閉源或未上游化的程式碼使用。這些擴充在所有介面中都必須被清楚標示為供應商專屬，且不得與 OpenBMC 專案綁定。

- 用於實作錯誤與事件回報的 API 應具有良好的易用性。這些 API 必須為適當的程式語言在編譯期辨識出不符合 BMC 錯誤與事件規範的呼叫點。
  - 所產生的錯誤類別與 API 不應要求使用例外，但也應與確實利用例外的 `sdbusplus` 客戶端與伺服器綁定整合。

[error-discussion]:
  https://discord.com/channels/775381525260664832/855566794994221117/867794201897992213

## 提案設計

提案設計有以下幾個高階設計元素：

- 整合 `sdbusplus` 與 `phosphor-logging` 的錯誤回報實作；將其擴展以涵蓋追蹤事件；改善相關 API 的易用性，並加入對缺少中繼資料的編譯期檢查。

- 在 `phosphor-logging` 中加入 API，讓守護程序能夠輕鬆查詢它們自己先前回報的事件（用以標記為已解決）。

- 在 `phosphor-logging` 中加入一套編譯期機制，停用特定追蹤事件的記錄，以進行供應商層級的客製化。

- 使用來自 `sdbusplus` 的綁定產生器，為 `phosphor-dbus-interfaces` 中定義的所有錯誤與事件產生一個 Redfish Message Registry。增強 `bmcweb` 中 `Logging.Entry` 到 `LogEvent` 的轉換實作，以涵蓋 Redfish Message Registry 與 `phosphor-logging` 的增強功能；利用 Redfish `LogEntry.DiagnosticData` 欄位提供整個 `Logging.Entry` 經 Base64 編碼的 JSON 表示，以利額外的診斷[[這是否需要設為選用？]]。在 `bmcweb` EventService 實作中加入支援，以支援由 `phosphor-logging` 托管的事件。

### `sdbusplus`

`Foo.errors.yaml` 的內容將與原本位於 `phosphor-logging` 所指定的 `Foo.metadata.yaml` 檔案中的內容合併，並由新的檔案類型 `Foo.events.yaml` 來指定。這個 `Foo.events.yaml` 格式將涵蓋目前 `error` 與 `metadata` 的資訊，並補充產生對外資料集（例如 Redfish Message Registries）所需的額外資訊。目前的 `Foo.errors.yaml` 與 `Foo.metadata.yaml` 檔案將被棄用，因為它們的用途將被新格式取代。

`sdbusplus` 函式庫將被增強以提供以下功能：

- 對所產生的例外型別及其指派的中繼資料進行 JSON 序列化與反序列化；當錯誤從 DBus 伺服器呼叫回傳時，將 JSON 序列化結果指派給 `sd_bus_error_set` 呼叫的 `message` 欄位。

- 一套機制，在函式庫載入時向 `sdbusplus` 函式庫註冊例外型別，以便在 DBus 客戶端中自動轉換回 C++ 例外型別。

綁定產生器將被擴展以執行以下工作：

- 為錯誤與事件產生完整的 C++ 例外型別，具備對缺少中繼資料的編譯期檢查以及 JSON 序列化。中繼資料可以是以下型別之一：
  - size 型別與有號整數
  - 浮點數
  - 字串
  - DBus 物件路徑

- 產生一種 `bmcweb` 可用來建立並填充 Redfish Message Registry 的格式，並針對一組錯誤與事件將 `phosphor-logging` 轉換為 Redfish `LogEntry`

對於 `sdbusplus` 的一般使用者而言，這些變更應不造成影響，除了新增可用的產生例外型別，以及在 DBus 客戶端中將可使用特定化實例的 `sdbusplus::exception::generated_exception`。

### `phosphor-dbus-interfaces`

將進行重構，隨著應用程式逐步遷移，把現有的 `Foo.metadata.yaml` 與 `Foo.errors.yaml` 內容遷移至 `Foo.events.yaml`。將做一些小改變以使用來自 `sdbusplus` 的新綁定產生器。也將進行一項小型函式庫增強，向 `sdbusplus` 註冊所有產生的例外型別。未來的貢獻者將能夠貢獻新的錯誤與追蹤事件定義。

### `phosphor-logging`

`Logging.Entry` 介面的 `AdditionalData` 屬性應變更為 `dict[string, variant[string,int64_t,size_t,object_path]]`。

`Logging.Create` 介面將新增一個新方法：

```yaml
- name: CreateEntry
  parameters:
    - name: Message
      type: string
    - name: Severity
      type: enum[Logging.Entry.Level]
    - name: AdditionalData
      type: dict[string, variant[string,int64_t,size_t,object_path]]
  returns:
    - name: Entry
      type: object_path
```

將新增一個 `lg2::commit` API 以支援新的 `sdbusplus` 產生例外型別，呼叫前述新提出的 `Logging.Create.CreateEntry` 方法。這個新 API 將支援 `sdbusplus::bus_t` 進行同步 DBus 操作，以及 `sdbusplus::async::context_t` 與 `sdbusplus::asio::connection` 進行非同步 DBus 操作。

`phosphor-logging` 實作仍存在尚未解決的效能顧慮，可能會影響擴展至 10,000 筆事件記錄的能力。這個問題預期是自-contained 於 `phosphor-logging` 之內，除了未來可能對 `bmcweb` 所使用的記錄擷取介面所做的變更。為了將此設計的過渡（由記錄 API 的呼叫者進行）與 `phosphor-logging` 中的實驗與改進解耦，我們將新增一個編譯選項與一個 Yocto `DISTRO_FEATURE`，可將 `lg2::commit` 行為轉換為 journal 中的一筆 `OPENBMC_MESSAGE_ID` 記錄，採用與先前 `REDFISH_MESSAGE_ID` 相同的方式，並提供相對應的 `rsyslog` 設定與 `bmcweb` 支援以直接使用這些記錄。這將讓那些明知會擴展至大量事件記錄、使用 `rsyslog` 機制的系統，享有相同層級的效能。此支援的一項注意事項是：當啟用該選項時，hint 與 resolution 行為將不存在。

### `bmcweb`

`bmcweb` 已支援在建置期將以 JSON 編寫的 Redfish Message Registry 轉換為它用來提供 registry 的標頭檔；這將被擴展以支援由 `sdbusplus` 產生的 Redfish Message Registries。`bmcweb` 將加入一個 Meson 選項，用於額外的 message registries，這些 registry 由 bitbake 從 `phosphor-dbus-interfaces` 提供，以及供應商專屬的事件定義，作為指向一個 Message Registry JSON 目錄的路徑。也將加入支援，將 `phosphor-dbus-interfaces` 作為 Meson 子專案加入，以便進行獨立測試。

我們希望 `sdbusplus` 能直接產生一個 Redfish Message Registry，利用現有腳本與 `bmcweb` 整合。作為此工作的一部分，我們希望支援將 `Logging.Entry` 事件對應到一個既有的標準化 Redfish 事件（例如 Base registry 中的那些）。所產生的資訊必須包含 `Logging.Entry::Message` 識別碼、`AdditionalData` 到 `MessageArgs` 的對應，以及從 `Message` 識別碼到 Redfish Message ID 的轉換（當 Message ID 並非來自「此」registry 時）。為了促成這一點，我們將需要在 Redfish Message Registry JSON 中加入 OEM 欄位，這些欄位僅供 `bmcweb` 處理腳本使用，以產生此项額外對應所需的資訊。

`xyz.openbmc_project.Logging.Entry` 到 `LogEvent` 的轉換需要加以增強，以利用這些 Message Registries，共四種方式：

1. `Logging.Entry` 經 Base64 編碼的 JSON 表示將被指派給 `DiagnosticData` 屬性。

2. 如果 `Logging.Entry::Message` 包含一個對應到某 Registry 條目的識別碼，則 `MessageId` 屬性將被設為相對應的 Redfish Message ID。否則，將直接使用 `Logging.Entry::Message`，不再進一步轉換（如同目前的作法）。

3. 如果 `Logging.Entry::Message` 包含一個對應到某 Registry 條目的識別碼，則 `MessageArgs` 屬性將透過從 `AdditionalData` 字典取得相對應的值來填充，而 `Message` 欄位將由將這些值與來自 Registry 的 `Message` 字串組合而產生。

4. 應實作一套機制，將 DBus `object_path` 引用轉換為 Redfish Resource URIs。當某個 `object_path` 無法被轉換時，`bmcweb` 將在 `MessageArgs` 值中使用類似 `object_path:` 的前綴。

`EventService` 的實作應加以增強，以支援由 `phosphor-logging` 托管的事件。`LogService` 的實作應加以增強，以支援針對 `phosphor-logging` 托管事件的記錄分頁。

### `phosphor-sel-logger`

`phosphor-sel-logger` 有一個 meson 選項 `send-to-logger`，用來切換使用 `phosphor-logging` 或是 [`REDFISH_MESSAGE_ID` 機制][existing-design]。使用 `phosphor-logging` 的路徑將被更新，以使用 `phosphor-dbus-interfaces` 所指定的錯誤與事件。

### YAML 格式

考慮 `phosphor-dbus-interfaces` 中一個範例檔案 `yaml/xyz/openbmc_project/Software/Update.events.yaml`，其中含有假設的錯誤與事件：

```yaml
version: 1.3.1

errors:
  - name: UpdateFailure
    severity: critical
    metadata:
      - name: TARGET
        type: string
        primary: true
      - name: ERRNO
        type: int64
      - name: CALLOUT_HARDWARE
        type: object_path
        primary: true
    en:
      description: While updating the firmware on a device, the update failed.
      message: A failure occurred updating {TARGET} on {CALLOUT_HARDWARE}.
      resolution: Retry update.

  - name: BMCUpdateFailure
    severity: critical
    deprecated: 1.0.0
    en:
      description: Failed to update the BMC
    redfish-mapping: OpenBMC.FirmwareUpdateFailed

events:
  - name: UpdateProgress
    metadata:
      - name: TARGET
        type: string
        primary: true
      - name: COMPLETION
        type: double
        primary: true
    en:
      description: An update is in progress and has reached a checkpoint.
      message: Updating of {TARGET} is {COMPLETION}% complete.
```

每個 `foo.events.yaml` 檔案將用於同時產生 C++ 類別（透過 `sdbusplus`）以供例外處理與事件回報，以及一個經過版本控管的 Redfish Message Registry，用於錯誤與事件。[YAML schema][yaml-schema] 位於 sdbusplus 倉儲中。

上述範例 YAML 將會產生類似以下的 C++ 類別：

```cpp
namespace sdbusplus::errors::xyz::openbmc_project::software::update
{

class UpdateFailure
{

    template <typename... Args>
    UpdateFailure(Args&&... args);
};

}

namespace sdbusplus::events::xyz::openbmc_project::software::update
{

class UpdateProgress
{
    template <typename... Args>
    UpdateProgress(Args&&... args);
};

}
```

這裡的建構子是可變參數範本，因為所產生的建構子實作將在編譯期確保所有中繼資料欄位都已被填充（順序不拘）。若要拋出一個 `UpdateFailure`，開發者可能會做類似以下的事情：

```cpp
// Immediately report the event:
lg2::commit(UpdateFailure("TARGET", "BMC Flash A", "ERRNO", rc, "CALLOUT_HARDWARE", bmc_object_path));
// or send it in a dbus response (when using sdbusplus generated binding):
throw UpdateFailure("TARGET", "BMC Flash A", "ERRNO", rc, "CALLOUT_HARDWARE", bmc_object_path);
```

如果其中一個欄位（例如 `ERRNO`）被省略，將引發一個編譯失敗，指出第一個缺少的欄位。

[yaml-schema]:
  https://github.com/openbmc/sdbusplus/blob/master/tools/sdbusplus/schemas/events.schema.yaml

### 版本控管政策

假設版本遵循語意化版本控管 `MAJOR.MINOR.PATCH` 慣例。

- 調整描述或訊息應導致 `PATCH` 遞增。
- 新增錯誤或事件，或為既有的錯誤或事件新增中繼資料，應導致 `MINOR` 遞增。
- 棄用一個錯誤或事件應導致 `MAJOR` 遞增。

有關 OpenBMC Message Registry 的[維護指引][registry-guidance]。我們將把該指引納入 `phosphor-dbus-interfaces` 中相對應的政策。

[registry-guidance]:
  https://github.com/openbmc/bmcweb/blob/master/redfish-core/include/registries/openbmc_message_registry.readmefirst.md

### 產生的 Redfish Message Registry

[DSP0266][dsp0266]，即 Redfish 規範，規定了 Redfish Message Registries 的需求，並制訂了識別碼的準則。

上述定義的假設事件將建立一個類似以下的 message registry：

```json
{
  "Id": "OpenBMC_Base_Xyz_OpenbmcProject_Software_Update.1.3.1",
  "Language": "en",
  "Messages": {
    "UpdateFailure": {
      "Description": "While updating the firmware on a device, the update failed.",
      "Message": "A failure occurred updating %1 on %2.",
      "Resolution": "Retry update."
      "NumberOfArgs": 2,
      "ParamTypes": ["string", "string"],
      "Severity": "Critical",
    },
    "UpdateProgress" : {
      "Description": "An update is in progress and has reached a checkpoint."
      "Message": "Updating of %1 is %2\% complete.",
      "Resolution": "None",
      "NumberOfArgs": 2,
      "ParamTypes": ["string", "number"],
      "Severity": "OK",
    }
  }
}
```

前綴 `OpenBMC_Base` 應專屬保留給來自 `phosphor-logging` 的事件使用。在其他倉儲中定義的事件預期將使用某種其他前綴。供應商定義的倉儲應依 [DSP0266][dsp0266] 的指示使用供應商拥有的前綴。

[dsp0266]:
  https://www.dmtf.org/sites/default/files/standards/documents/DSP0266_1.20.0.pdf

### 對供應商的影響

如上所述，供應商必須使用它們自己的識別碼，以符合 Redfish 規範（關於識別碼命名的需求請見 [DSP0266][dsp0266]）。`sdbusplus`（以及 `phosphor-logging` 與 `bmcweb`）的實作將讓供應商能夠為下游程式碼建立它們自己的事件，並建立 Registries 以與 Redfish 整合，方法是建立下游的錯誤定義倉儲。供應商有責任確保它們自己的版本控管與識別碼符合 [Redfish 規範][dsp0266]中的期待。

供應商可能出現的一種不良行為是分岔並修改 `phosphor-dbus-interfaces` 定義的事件。供應商不得在下游實作中將它們自己的事件加入 `phosphor-dbus-interfaces`，因為這會導致它們的實作在某個 OpenBMC 拥有的 Registry 中公告支援某個訊息，而事實卻並非如此；它們應該把這些事件加入自己的倉儲，並使用一個獨立的識別碼。類似地，如果供應商將上游變更「回溯移植」到它們的分岔中，它們需要確保該版本的 `foo.events.yaml` 檔案與上游實作完全一致。

### 事件擴充

事件在其 `foo.events.yaml` 中定義的中繼資料，專門承載了建立其 Redfish Message Args 所需的資訊，這些訊息引數透過 message registries 標準化。此外，某些事件還有可由 Redfish 承載的額外資料，例如特定硬體失敗的 CPER 屬性。另外，一些供應商對於 OEM 資料使用非標準的 schema。為了建立一個可擴充的框架來封裝這些資料，我們允許事件以額外資料進行擴充，這些資料由一個沒有方法的 dbus 介面表示。可以使用以下方式填充這些自訂資料：

```cpp
    lg2::commit(PlatformError(...).extend(CPER::properties_t{...});
```

透過這種方式，我們可以利用 dbus 介面為這些對標準事件的擴充進行 schema 化，讓 `phosphor-logging` 能夠將它們與 `Logging.Entry` 一起儲存，並讓其他守護程序能夠消費它們（例如 `bmcweb` 用於 CPER 資料）。當 `phosphor-logging` 在 `/xyz/openbmc_project/logging/entry/<N>` 建立一個 `Logging.Entry` 時，如果提供了擴充，它也會在同一個物件路徑上建立任何已知的擴充介面。

目前預期的擴充有：

- Search Hints
- Processed CPER data
- Raw CPER data

#### Search Hints

`SearchHint` 供守護程序查詢它們先前記錄的錯誤使用，用於例如將它們標記為已解決。這些字串需要是全域唯一的，建議格式為 `"<service_name>:<key>"`。將定義一個 `Logging.SearchHint` 介面，並允許其作為 `phosphor-logging` 支援的擴充：

```yaml
- property: Hint
  type: string
```

將在 `/xyz/openbmc_project/logging` 加入一個帶有單一方法的 `Logging.Manager` 介面：

```yaml
- name: FindEntry
  parameters:
    - name: Hint
      type: string
  returns:
    - name: Entry
      type: object_path
  errors:
    - xyz.openbmc_project.Common.Error.ResourceNotFound
```

## 考慮過的替代方案

先前的工作已探索並引用了許多替代方案。在本提案中，也已評估了許多小型替代方案。

### 例外繼承

原本的 `phosphor-logging` 錯誤描述允許兩個錯誤之間的繼承。本提案不支援這一點，原因有二：

- 這會在 Redfish Message Registry 的版本控管中引入複雜性，因為某個檔案中的變更應導致所有相依檔案的版本變更。

- 這會讓開發者難以在不 traversing 多個檔案的情況下清楚辨識他們預期需要填充的所有欄位。

### sdbusplus 例外 API

我想出了幾種可能的語法來構造所產生的例外型別。重要的是這些語法要具有良好的易用性、易於理解，並能對缺少的中繼資料欄位提供編譯期的感知。

```cpp
    using Example = sdbusplus::error::xyz::openbmc_project::Example;

    // 1)
    throw Example().fru("Motherboard").value(42);

    // 2)
    throw Example(Example::fru_{}, "Motherboard", Example::value_{}, 42);

    // 3)
    throw Example("FRU", "Motherboard", "VALUE", 42);

    // 4)
    throw Example([](auto e) { return e.fru("Motherboard").value(42); });

    // 5)
    throw Example({.fru = "Motherboard", .value = 42});
```

**注意**：這些範例全部以 `throw` 語法展示，但它們也可以被儲存在區域變數中、從函式回傳，或直接傳給 `lg2::commit`。

1. 這會是我在易用性與清晰度上的偏好，因為它會讓支援 LSP 的編輯器提供中繼資料欄位的補全，但可惜 C++ 中沒有任何機制可以定義一個可被構造但不可被拋出的型別，這意味著我們無法取得對所有中繼資料欄位的編譯期檢查。

2. 這個語法使用標籤分派（tag-dispatch），能夠對所有中繼資料欄位進行編譯期檢查，並可能對標籤型別進行 LSP 補全，但比選項 3 更冗長。

3. 這個語法比 (2) 更不冗長，並遵循 `phosphor-logging` 的 `lg2` API 中已使用的慣例，但不允許對中繼資料標籤進行 LSP 補全。

4. 這個語法類似選項 (1)，但使用 lambda 的間接層，以啟用編譯期檢查，確認所有中繼資料欄位都已被 lambda 填充。由於使用了 `auto`，LSP 補全可能不如選項 (1) 強大，而 lambda 的必要性對於不熟悉的開發者而言可能是一個障礙。

5. 這個語法具有與選項 (1) 類似的特性，但同樣無法提供對所有欄位都已被填充的編譯期確認。

因此，本提案建議選項 (3) 最為適合。

### Redfish 翻譯支援

提案的 YAML 格式允許未來加入翻譯，但目前尚未啟用。未來的開發可以在 `message:language` 存在於那些語言時，讓 Redfish Message Registry 以多種語言產生。

### Redfish Registry 版本控管

Redfish Message Registries 必須經過版本控管，並有 3 個數位欄位（例如 `XX.YY.ZZ`），但 Message ID 中只應使用前 2 個欄位。我們可以不使用手動指定的版本，而採取其他幾種做法：

- 使用日期碼（例如 `2024.17.x`）表示 registry 建置時的 ISO 8601 週。
  - 這無法涵蓋可能為穩定目的而選擇分岔的供應商，因此我們可能最終出現兩台機器具有相同 OpenBMC 版本控管的 message registry，但內容卻不同。

- 使用最近的 `openbmc/openbmc` 標籤作為版本。
  - 這無法涵蓋從 HEAD 建置並可能在兩次 OpenBMC 發佈之間部署多個映像檔的供應商。

- 根據 git 歷史記錄產生版本。
  - 這要求 `phosphor-dbus-interfaces` 必須從一個 git 倉儲建置，而這對 Yocto 來源鏡像而言未必總是成立，且需要非簡單的處理，並會隨著時間持續擴展。

### 現有的 OpenBMC Redfish Registry

目前版本為 `OpenBMC.0.4.0` 的現有 Redfish Message Registry 中定義了 191 則訊息。在這些訊息中，程式碼庫裡沒有一則是以正確的版本發出的。其中 96 則僅由 Intel 專屬的程式碼發出，而這些程式碼未被納入任何上游化的機器；39 則由潛在通用的程式碼發出；另有 56 則甚至在 bmcweb registry 之外完全未被程式碼庫引用。在這 39 則通用訊息中，有一半在標準 registries 之一中存在一個應被利用的對應訊息，而許多其他訊息不具備能促進多主機設定的屬性，因此這個 registry 至少需要更新。目前的實作完全沒有處理 Redfish Resource URIs 的能力。

因此，本提案是要棄用現有的 registry，並以新產生的 registries 取代它。對於目前以既有格式發出事件的倉儲，我們可以在 1 到 2 年的期間內維持那些呼叫點。

如果本提案的這一方面被否決，YAML 格式允許從 `phosphor-dbus-interfaces` 定義的事件對應到目前 `OpenBMC.0.4.0` registry 的 `MessageIds`。

潛在通用：

- phosphor-post-code-manager
  - BIOSPOSTCode（unique）
- dbus-sensors
  - ChassisIntrusionDetected（unique）
  - ChassisIntrusionReset（unique）
  - FanInserted
  - FanRedundancyLost（unique）
  - FanRedudancyRegained（unique）
  - FanRemoved
  - LanLost
  - LanRegained
  - PowerSupplyConfigurationError（unique）
  - PowerSupplyConfigurationErrorRecovered（unique）
  - PowerSupplyFailed
  - PowerSupplyFailurePredicted（unique）
  - PowerSupplyFanFailed
  - PowerSupplyFanRecovered
  - PowerSupplyPowerLost
  - PowerSupplyPowerRestored
  - PowerSupplyPredictiedFailureRecovered（unique）
  - PowerSupplyRecovered
- phosphor-sel-logger
  - IPMIWatchdog（unique）
  - `SensorThreshold*`：8 個不同的事件
- phosphor-net-ipmid
  - InvalidLoginAttempted（unique）
- entity-manager
  - InventoryAdded（unique）
  - InventoryRemoved（unique）
- estoraged
  - ServiceStarted
- x86-power-control
  - NMIButtonPressed（unique）
  - NMIDiagnosticInterrupt（unique）
  - PowerButtonPressed（unique）
  - PowerRestorePolicyApplied（unique）
  - PowerSupplyPowerGoodFailed（unique）
  - ResetButtonPressed（unique）
  - SystemPowerGoodFailed（unique）

Intel 專屬實作：

- intel-ipmi-oem
  - ADDDCCorrectable
  - BIOSPostERROR
  - BIOSRecoveryComplete
  - BIOSRecoveryStart
  - FirmwareUpdateCompleted
  - IntelUPILinkWidthReducedToHalf
  - IntelUPILinkWidthReducedToQuarter
  - LegacyPCIPERR
  - LegacyPCISERR
  - `ME*`：29 個不同的事件
  - `Memory*`：9 個不同的事件
  - MirroringRedundancyDegraded
  - MirroringRedundancyFull
  - `PCIeCorrectable*`、`PCIeFatal`：29 個不同的事件
  - SELEntryAdded
  - SparingRedundancyDegraded
- pfr-manager
  - BIOSFirmwareRecoveryReason
  - BIOSFirmwarePanicReason
  - BMCFirmwarePanicReason
  - BMCFirmwareRecoveryReason
  - BMCFirmwareResiliencyError
  - CPLDFirmwarePanicReason
  - CPLDFirmwareResilencyError
  - FirmwareResiliencyError
- host-error-monitor
  - CPUError
  - CPUMismatch
  - CPUThermalTrip
  - ComponentOverTemperature
  - SsbThermalTrip
  - VoltageRegulatorOverheated
- s2600wf-misc
  - DriveError
  - InventoryAdded

## 衝擊

- 為錯誤與事件記錄定義了新的 API。這將棄用現有的 `phosphor-logging` API（用於錯誤回報），並留有遷移時間。

- 此設計應能透過消除對 `systemd` journal 的常態性解析來改善效能。此設計也可能因為允許錯誤與事件記錄的數量大幅增加而降低效能，這會影響檔案系統的使用率，並有可能對某些服務（例如 `ObjectMapper`）造成 DBus 衝擊。

- 透過自動產生對應所有錯誤與事件回報的 Redfish Message Registry，應能改善向後相容性與文件。

### 組織層面

- **這個倉儲是否需要一個新的倉儲？**
  - 否
- **誰會是這個倉儲的初始維護者？**
  - 不適用
- **預期需要修改哪些倉儲以執行此設計？**
  - `sdbusplus`
  - `phosphor-dbus-interfaces`
  - `phosphor-logging`
  - `bmcweb`
  - 任何建立錯誤或事件的倉儲。

## 測試

- 將在 `sdbusplus` 與 `phosphor-logging` 中撰寫單元測試，針對錯誤與事件產生、建立 API，並為對 `Logging.Entry` 物件管理的任何變更提供涵蓋率。

- 將為 `bmcweb` 撰寫單元測試，針對基本的 `Logging.Entry` 轉換與 Message Registry 產生。

- 應利用（並視需要增強）來自 `openbmc-test-automation` 的整合測試，以涵蓋端對端的錯誤建立與 Redfish 回報。
