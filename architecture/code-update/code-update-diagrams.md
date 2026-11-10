# Code Update 圖示

1. [高階概述](#高階概述)

## 高階概述

```ascii
┌──────────────┐            ┌─────────────┐       ┌────────────┐
│User Interface│            │Image Manager│       │Item Updater│
└──────┬───────┘            └──────┬──────┘       └──────┬─────┘
       │          Upload           │                     │
       │         Firmware          │                     │
       │       Image to BMC        │                     │
       ├──────────────────────────▶│                     │
       │                           │                     │
       │                           │  Extract            │
       │                           │   image             │
       │                           │ contents            │
       │                           │     │               │
       │                           ├─────┘               │
       │                           ▼                     │
       │                           │                     │
       │                           │    Create           │
       │                           │Software D-Bus       │
       │                           │  object[1]          │
       │                           │       │             │
       │                           ├───────┘             │
       │                           ▼                     │
       │                           │                     │
       │         Request to        ●                     │
       │          Activate                               │
       │          Software                               │
       │        D-Bus Object                             │
       ├────────────────────────────────────────────────▶│
       │                                                 │   Verify
       │                                                 │  digital
       │                                                 │ signatures
       │                                                 │      │
       │                                                 ├──────┘
       │                                                 ▼
       │                                                 │
       │                                                 │   Write
       │                                                 │ image to
       │                                                 │  flash[*]
       │                                                 │     │
       │                                                 ├─────┘
       │                                                 ▼
       │                                                 │
       │                                     Success     │
       │◀────────────────────────────────────────────────┤
       │                                                 │
       │                                                 ●
       ▼
    BMC Reboot is
  required to boot
  from the updated
        image
 
```

- [1]
  [Software D-Bus Object](https://github.com/openbmc/phosphor-dbus-interfaces/tree/master/yaml/xyz/openbmc_project/Software)
- [*] 在靜態佈局（static layout）設定中，映像檔會儲存在 RAM 中，並在 BMC 重新開機時才將內容寫入 flash。請參考
  [initrdscripts](https://github.com/openbmc/openbmc/tree/master/meta-phosphor/recipes-phosphor/initrdscripts)
  所提供的更新（update）與關機（shutdown）腳本。
