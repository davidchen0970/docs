# NC-SI core dump

作者：DelphineCCChiu (<Delphine_CC_Chiu@wiwynn.com>)

建立日期：03/12/2024

## 問題描述

NIC core-dump 對於偵錯 NIC 問題至關重要，可以透過帶內（in-band）與帶外（out of band）兩種方式取得。本設計提供的是從 BMC 透過 NC-SI 協定進行 NIC 帶外 dump 的解決方案。

## 背景與參考資料

用於取得 dump 的 NC-SI 指令：參考 NC-SI spec v1.2，章節 8.4.114
<https://www.dmtf.org/sites/default/files/standards/documents/DSP0222_1.2.0.pdf>

NC-SI over MCTP：
<https://www.dmtf.org/sites/default/files/standards/documents/DSP0261_1.3.0.pdf>

## 需求

此功能需要 Linux kernel 在 net/ncsi 模組中支援傳輸新的 NC-SI 指令（0x4D）
<https://github.com/torvalds/linux/commits/master/net/ncsi>

## 提案設計

### 介面

本設計將沿用現有的 phosphor-debug-collector 模組：
<https://github.com/openbmc/phosphor-debug-collector>，並擴展 dump 建立介面，新增一個「NC-SICoreDump」dump 類型。

用於建立 dump 的 D-Bus 介面為："xyz.openbmc_project.Dump.Manager
/xyz/openbmc_project/dump/bmc xyz.openbmc_project.Dump.Create"

為了指出要以哪一條 NC-SI link 為目標，CreateDump 方法需要一個額外的輸入參數："NICTarget"。有效的值可以是 EID 或網路介面，例如 eth0。

### Dump 取得

使用標準 NC-SI 指令：Retrieve Data From NC(0x4D)，透過 NC-SI over RBT 或 NC-SI over MCTP 協定來取得 dump。所有 NC-SI dump 程序都會在 phosphor-networkd 的 ncsi-netlink 與 ncsi-mctp 工具程式中實作：
<https://github.com/openbmc/phosphor-networkd/blob/master/src/ncsi_netlink_main.cpp>

### 與 phosphor-debug-collector 整合

由於 phosphor-debug-collector 使用 shell 腳本進行資料蒐集，因此會新增一個名為 "ncsicoredump" 的 collector 腳本。這個腳本會協助根據不同的 NICTarget 呼叫 ncsi-netlink 或 ncsi-mctp，並在特定資料夾下產生 dump 檔案。

下方的區塊圖說明了整體 dump 程序以及各模組之間的關係：

```text

                           +----------------+           +-----------+
                           |                |           |           |
             ------------->  Dump manager  +-----------> DumpEntry |
              CreateDump  |                |           |           |
                           +--------+-------+           +-----------+
                                    |
                                    |
                                    |
                           +--------v-------+
                           |                |
                           |    Dreport     |
                           |                |
                           +--------+-------+
                                    |
                                    |
                                    |
                           +--------v-------+
                           |                |
                           |  Plugin:       +------------------+
                           |  ncsicoredump  |                  |
                           |                |                  |
                           +--------+-------+                  |
                                    |                          |
                                    |                          |
                                    |                          |
                                    |                          |
       +------------+       +--------v-------+          +-------v------+        +------------+
       |            |       |                |          |              |        |            |
       |  DumpFile  <-------+  NCSI-NetLink  |          |  NCSI-MCTP   +-------->  DumpFile  |
       |            |       |                |          |              |        |            |
       +------------+       +--------^-------+          +-------^------+        +------------+
                                    |                          |
     --------------------------------+--------------------------+-----------------------------
         Kernel                      |Netlink                   |MCTP
                            +--------v-------+         +--------v-------+
                            |                |         |                |
                            |Net/NC-SI module|         |   I2C driver   |
                            |                |         |                |
                            +--------^-------+         +--------^-------+
                                    |                          |
                                    |NC-SI                     |SMBUS
                                    |                          |
                           +--------v--------------------------v-------+
                           |                                           |
                           |                     NIC                   |
                           |                                           |
                           +-------------------------------------------+

```

## 曾考量的替代方案

由於 NC-SI 指令（0x4d）的接收順序必須維持一致，我們應以此來阻擋重複的 dump 程序。因為 core dump 內部最多會包含兩份 crash dump，我們目前僅支援 core dump，對現階段的使用需求而言已經足夠。

## 影響

無。

## 測試

與 NIC 供應商（Broadcom）合作，驗證 dump 程序與 dump 檔案。
