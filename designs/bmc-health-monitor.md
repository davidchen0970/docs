# BMC 健康監控

作者：Vijay Khemka <vijaykhemka@fb.com>、Sui Chen <suichen@google.com>、Jagpal
Singh Gill <paligill@gmail.com>

建立日期：2020-05-04

## 問題描述

問題在於如何監控一台配有 BMC 的系統健康狀態，讓我們有機會確認 BMC 是否正常運作。
使用者可以依照設定立即取得所需的指標資料。被監控的指標可能包括 CPU 與記憶體
使用率、上線時間（uptime）、剩餘磁碟空間、I2C 匯流排統計資料等等。可以根據監控
資料採取動作來修正 BMC 的狀態。

為此目的，可能會有一個指標產生端（本文件討論的主體）與一個指標消費端（一個使用
健康監控資料的程序，可執行在 BMC 上或主機上）。它們執行下列工作：

1. 設定（Configuration），由使用者指定要蒐集什麼、如何蒐集、閾值等等。
2. 指標蒐集（Metric collection），類似 phosphor-hwmon-readd 中的 read 常式所做的
   事情。
3. 指標暫存（Metric staging）。當指標被蒐集後，它們會以可存取的形式（例如 DBus
   物件或原始檔案）隨時可就緒供消費端程式讀取。因為有這個暫存步驟，消費端不需要
   輪詢等待。
4. 資料傳輸（Data transfer），消費端程式透過帶內（in-band）或帶外（out-of-band）
   方式從 BMC 取得指標。
5. 消費端程式可以根據蒐集到的指標採取某些動作。

在上述工作中，1)、2)、3) 是產生端的職責。4) 由產生端與消費端共同完成。5)
則由消費端自行決定。

我們了解感測器（sensors）與健康監控在設計理念與既有基礎設施上有一些重疊。但
感測器與指標之間也有不少差異：

1. 感測器資料來自硬體，而多數指標可能透過軟體取得。基於這個原因，各種 BMC 上的
   指標彼此之間的共通性，可能比 BMC 上的感測器還要多，而且我們對多數健康指標
   可能不需要硬體探索程序，或建置時期與硬體相關的設定。
2. 多數感測器是即時讀值，而指標可能隨時間累積，例如「uptime」。對於這類指標，
   我們可能想做一些不適用於感測器讀值的運算。
3. 指標可以代表不會變更的裝置屬性，例如固定不變的系統總記憶體。相反地，感測器的
   主要用意是偵測屬性的變化並呈現該變化性。
4. 指標以原生單位表示，例如記憶體以 byte 為單位。感測器基礎設施並不支援這一點，
   社群也已否決了為感測器單位新增 byte 的提案。

根據以上所述，使用感測器來代表指標資料似乎並不合理。

## 背景與參考資料

參考資料：dbus-monitor

## 需求

指標產生端應該提供

- 一個常駐程式（daemon），定期蒐集各種健康指標並將其暴露（expose）在 DBus 上。
- 一個 dbus 介面，讓其他服務（例如 redfish 與 IPMI）可以存取其資料。
- 能夠為各種各樣的指標設定健康監控，例如記憶體使用率、CPU 使用率、重新啟動
  統計資料等等。
- 能夠為各種指標類型提供細緻的明細，例如 -
  - 記憶體使用率 - 可用記憶體（Free Memory）、共用記憶體（Shared Memory）、
    Buffered&CachedMemory 等等。
  - CPU 使用率 - 使用者模式 CPU 使用率、核心模式 CPU 使用率等等。
  - 重新啟動統計資料 - 正常重新啟動次數、伴隨失敗的重新啟動次數等等。
- 能夠在數值超過閾值時，依設定採取動作。
- （選擇性）維持一定數量的歷史資料。
- （選擇性）記錄 critical / warning 訊息。

指標消費端可以用多種不同方式撰寫。不論消費端如何取得，它都應該能夠透過一組介面
從產生端取得健康指標。

指標消費端不在本文件的範圍之內。

## 建議設計

指標產生端是一個執行在 BMC 上的常駐程式，負責執行必要的工作並滿足上述需求。如前
所述，它負責：

1. 設定（Configuration）
2. 指標蒐集（Metric collection）與
3. 指標暫存與散布（Metric staging & disperse）工作

關於 1) 設定，常駐程式在程式碼中會有一份預設設定。平台如果想要覆寫（over-ride）
特定的預設屬性，可以提供一份設定檔。JSON 設定檔的格式如下 -

```json
  "kernel" : {
    "Frequency" : 1,
    "Window_size": 120,
    "Type": "CPU",
    "Threshold":
    {
        "Critical":
        {
            "Value": 90.0,
            "Log": true,
            "Target": "reboot.target"
        },
        "Warning":
        {
          "Value": 80.0,
          "Log": false,
          "Target": "systemd unit file"
        }
    }
  },
  "available" : {
    "Frequency" : 1,
    "Window_size": 120,
    "Type": "Memory",
    "Threshold":
    {
        "Critical":
        {
            "Value": 90.0,
            "Log": true,
            "Target": "reboot.target"
        }
    }
  }
```

Frequency：這是每隔幾秒以固定間隔蒐集這些資料。Window_size：這個值代表取樣次數，
用來平均化系統使用量，而不是取用某個使用量的尖峰資料。Log：一個布林值，決定是否
記錄一則告警。這個欄位是選擇性的，其預設值在 critical 為 'true'，在 warning 為
'false'。Target：這是一個 systemd target 單元檔案，當數值超過其閾值時會被呼叫，
它是選擇性的。Type：這表示設定項目的類型。可能值為 Memory、CPU、Reboot、Storage。

關於 2) 指標蒐集，這會是透過在常駐程式內部執行某些函式來完成，而不是啟動外部
程式與 shell 腳本。這是出於效能與安全性考量。

關於 3) 指標暫存與散布，常駐程式會建立一個名為
"xyz.openbmc_project.HealthManager" 的 D-Bus 服務。本設計提出新的
[Metrics Dbus 介面](https://gerrit.openbmc.org/c/openbmc/phosphor-dbus-interfaces/+/64914)。

| 介面名稱                             | 用途                                                                            | 必要／選擇性     |
| :----------------------------------- | :----------------------------------------------------------------------------- | :---------------- |
| xyz.openbmc_project.Metric.Value     | 代表指標數值的介面。                                                            | Required          |
| xyz.openbmc_project.Metric.Reset     | 重設持續性指標計數器的介面。                                                    | Optional          |
| xyz.openbmc_project.Common.Threshold | 代表指標閾值以及閾值違規訊號的介面。                                            | Optional          |
| xyz.openbmc_project.Time.EpochTime   | 指出指標何時被蒐集的介面。                                                      | Optional          |

每個指標會被暴露在特定的物件路徑上，上述介面會在這些路徑上實作。

```text
/xyz/openbmc_project
    |- /xyz/openbmc_project/metric/bmc/memory/total
    |- /xyz/openbmc_project/metric/bmc/memory/free
    |- /xyz/openbmc_project/metric/bmc/memory/available
    |- /xyz/openbmc_project/metric/bmc/memory/shared
    |- /xyz/openbmc_project/metric/bmc/memory/buffered_and_cached
    |- /xyz/openbmc_project/metric/bmc/cpu/user
    |- /xyz/openbmc_project/metric/bmc/cpu/kernel
    |- /xyz/openbmc_project/metric/bmc/reboot/count
    |- /xyz/openbmc_project/metric/bmc/reboot/count_with_failure
```

指標資料的伺服端

| 介面名稱           | 介面伺服端              | 資訊來源                                                |
| :----------------- | :---------------------- | :----------------------------------------------------- |
| Memory Utilization | phosphor-health-manager | /proc/meminfo                                          |
| CPU Utilization    | phosphor-health-manager | /proc/stat                                             |
| Reboot Statistics  | phosphor-state-manager  | 依重新啟動狀態遞增的持續性計數器 |

同類型的多個裝置 -

如果同類型有多個裝置，D-Bus 路徑可以延伸以加入關於 **"是哪一個裝置"** 的脈絡。
例如 -

```text
/xyz/openbmc_project/metric/device-0/memory/total
/xyz/openbmc_project/metric/device-1/memory/total
...
```

這些路徑可以由不同的常駐程式託管，例如，當主 BMC 使用 PLDM 與 BIC 通訊時，pldmd
可以託管 BIC 的 DBus 路徑。每個指標的 Value 介面需要與適當的系統庫存（system
inventory）項目建立關聯。

## 曾考慮的替代方案

我們曾嘗試完全在 IPMI Blob 架構內做到健康監控。相較之下，把指標蒐集部分做成獨立的
常駐程式，對於支援更多介面來說是較好的做法。

我們也曾嘗試透過執行外部二進位檔以及 shell 腳本來做指標蒐集的工作。結果發現執行
shell 腳本太慢，而執行外部程式則可能有安全疑慮（因為必須驗證第三方程式是安全的）。

Collected：Collectd 提供多種外掛（plugin），可以從各種來源蒐集各種各樣的指標，並
提供多種機制以不同方式儲存它們。要將這些指標暴露在 DBus 上，可以撰寫一個 Collectd
C 外掛。

優點：

- 現成（off the shelf）工具，支援大量指標。

缺點：

- 由於要支援多種系統（Linux、Solaris、OpenBSD、MacOSX、AIX 等等）與應用程式，
  每個 Collected 外掛的程式碼量都相當可觀。以 openBMC 所需的功能量來看，Collectd
  顯得笨重。phosphor-health-monitor 大部分的程式碼會圍繞著將指標暴露在 Dbus 上，
  而 Collectd 外掛同樣也需要這部分。因此，直接從 `/proc/<fileX>` 讀取顯得輕量，
  因為相關程式碼已經存在。
- Collected 對閾值監控的支援很有限，且不允許在閾值違規時啟動 systemd 服務。

## 未來增強

延伸 Metrics Dbus 介面以支援 -

- Storage
- Inodes
- 連接埠／網路統計資料
- BMC 常駐程式統計資料

## 衝擊

健康監控常駐程式所做的大部分工作是指標蒐集與更新 DBus 物件。常駐程式本身的衝擊
應該很小。

本建議設計將 DBus 介面從 Sensors 改為 Metrics，因此下列常駐程式需要重構／更新以
因應介面變更 -

- [BMCWeb](https://github.com/openbmc/bmcweb/blob/master/redfish-core/lib/manager_diagnostic_data.hpp)
- [phosphor-host-ipmid](https://grok.openbmc.org/xref/openbmc/openbmc/meta-quanta/meta-s6q/recipes-phosphor/configuration/s6q-yaml-config/ipmi-sensors.yaml?r=e4f3792f#82)

## 組織層面

### 這個設計需要新的 repository 嗎？

不需要，變更會進入 phosphor-health-monitor。

### 預期需要修改哪些 repository 來執行這個設計？

- phosphor-health-monitor
- phosphor-state-manager
- BMCWeb
- phosphor-host-ipmid

## 測試

### 單元測試

要驗證常駐程式運作正常，請監控由常駐程式產生的 DBus 流量，以及來自常駐程式 DBus
物件的指標數值。自動化單元測試將透過 GTest 涵蓋。

### 整合測試

手動端對端測試可以透過對 ManagerDiagnosticData 進行 Redfish GET 來執行。端對端
自動化測試將使用 openbmc-test-automation 來涵蓋。為了驗證效能層面，我們可以對
常駐程式的 DBus 介面做壓力測試，以確保這些介面不會造成過高的負擔。
