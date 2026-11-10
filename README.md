# OpenBMC 文件

[OpenBMC 專案](https://www.openbmc.org/) 是 Linux Foundation 旗下的一個專案，
目標是產出一套可客製化的開源韌體堆疊，供主機板管理控制器（BMC）使用。本
repository 收錄了 OpenBMC 整體的文件。各元件的相關文件可能會存放在該元件
自己的 repository 中。

[features](features.md) 文件列出了專案的主要功能，並附有通往更多資訊的連結。

## 聯絡方式

- 郵件：`openbmc@lists.ozlabs.org` <https://lists.ozlabs.org/listinfo/openbmc>
- 郵件清單封存：<https://lore.kernel.org/openbmc/>
- Discord：<https://discord.gg/69Km47zH98>

## OpenBMC 開發

這些文件說明了開發 OpenBMC 程式碼本身的相關細節

- [cheatsheet.md](cheatsheet.md)：一些常見開發任務的快速參考

- [CONTRIBUTING.md](CONTRIBUTING.md)：對 OpenBMC 貢獻的準則

- [開發教學](development/README.md)：幫助你快速上手 OpenBMC 開發的教學

- [kernel-development.md](kernel-development.md)：常見核心（kernel）開發任務的參考

## OpenBMC 使用

這些文件說明了如何使用 OpenBMC，包括使用 OpenBMC 系統程式化介面的方式。

- [code-update](architecture/code-update)：更新 OpenBMC 與主機平台韌體

- [console.md](console.md)：使用主機主控台

- [host-management.md](host-management.md)：使用 OpenBMC 執行主機管理任務

- [rest-api.md](rest-api.md)：使用 OpenBMC REST API 的簡介

- [REDFISH-cheatsheet.md](REDFISH-cheatsheet.md)：一些常見 OpenBMC Redfish 指令的快速參考

- [REST-cheatsheet.md](REST-cheatsheet.md)：一些常見 OpenBMC REST API 指令的快速參考
