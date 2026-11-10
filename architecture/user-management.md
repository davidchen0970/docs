# 使用者管理 - OpenBMC - 設計文件

## 範圍

本文件涵蓋使用者管理元件所需之架構、介面與設計細節。實作層面的細節不在本文件
的範圍之內。

## 設計基本原則

1. 使用通用的使用者管理機制（例如 phosphor-user-manager），而非各應用程式自行
   實作的使用者管理。特別是要避免使用基於 IPMI 的使用者管理。
2. 基於安全性考量，避免透過任何 D-Bus API 傳輸密碼。即使是在建立、修改或驗證
   使用者時，也必須遵守此規則。
3. 讓應用程式使用 PAM 模組來驗證使用者，而非依賴基於 D-Bus API 的做法。
4. 只要可能，使用者的建立必須維持一般性的設計。
5. 由於 IPMI 需要明文密碼（詳情請參考 IPMI 2.0 規格書第 13.19 至 13.33 節），
   因此必須搭配一般的 PAM 密碼雜湊模組（例如 pam-unix），使用新的 PAM 模組
   （例如 pam-ipmi 模組），以加密形式儲存密碼。若 IPMI 守護程序並非發行版本的
   一部分，或所建立的使用者不屬於 'ipmi' 群組角色，實作方可選擇略過此步驟。
6. 使用者名稱、密碼、群組與權限角色均由通用的使用者管理機制（例如
   phosphor-user-manager）維護，而各應用程式的個別使用者相關設定則必須由該
   應用程式自行管理。換言之，除了使用者名稱、密碼、群組與權限角色之外，其餘
   所需的設定必須由對應的應用程式負責。（例如 IPMI 守護程序必須為對應的使用者
   管理通道型限制等設定。）本設計已涵蓋此情境。

## 支援的群組角色

群組角色的用途在於決定對應使用者的第一層授權。它用於在高階層級判斷使用者是否
被授權存取所需的介面。換言之，若使用者僅屬於 redfish 群組而不屬於 SSH 群組，
則不應允許其登入 SSH。此外，將群組角色置於通用的使用者管理機制中（例如
phosphor-user-manager），可讓不同的應用程式為彼此建立角色（例如管理員使用者
將能透過 webserver 建立一位新的使用者，而該使用者將能登入 webserver/REDFISH
與 IPMI 等）。

_註：群組名稱僅供呈現之用，可依需求修改或擴充_

| 編號 | 群組名稱      | 用途 / 備註                                                            |
| -----: | ----------- | ------------------------------------------------------------------- |
|      1 | ssh         | 僅允許此群組中的使用者透過 SSH 登入。                                       |
|      2 | ipmi        | 僅允許此群組中的使用者使用 IPMI 介面。                                       |
|      3 | redfish     | 僅允許此群組中的使用者使用 REDFISH 介面。                                    |
|      4 | hostconsole | 僅允許此群組中的使用者與主機主控台互動。                                       |

## 支援的權限角色

OpenBMC 支援跨所有受支援群組通用的權限角色（亦即使用者在 REDFISH / IPMI / SSH /
HostConsole 上擁有相同的權限）。使用者在任何時間點僅能屬於下列其中一種權限角色。

_註：權限僅供呈現之用，可依需求修改或擴充_

| 編號 | 權限角色          | 用途 / 備註                                                                                                                                                              |
| -----: | --------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
|      1 | admin           | 允許使用者設定所有 OpenBMC（包含使用者管理、網路及所有設定）。使用者將擁有完整的系統管理權限。                             |
|      2 | operator        | 允許使用者檢視並控制基本作業，包括主機重新啟動等。但不允許使用者變更其他設定，例如使用者、網路等。 |
|      3 | user            | 使用者僅擁有唯讀權限，無法變更系統的任何行為。                                                                                                        |

## 通用使用者管理 - D-Bus API（phosphor-user-manager）

User Manager 服務公開了供使用者管理作業使用的 D-Bus 方法。它以
`xyz.openbmc_project.User.Manager` 作為服務名稱，並透過
`org.freedesktop.DBus.ObjectManager` 處理物件。詳細的使用者管理 D-Bus API 與
介面請參考
<https://github.com/openbmc/phosphor-dbus-interfaces/tree/master/yaml/xyz/openbmc_project/User>。

## OpenBMC - 使用者管理 - 高階架構圖

```ascii
                                 WEBSERVER/REDFISH
     ========================================================================
    ||  ________________  |  ________________  |   |**********************| ||
    ||  |PAM for user   | |  |Create new user| |   | Redfish specific 1:1 | ||
    ||  |authentication | |  |or delete or   | |   | user settings storage| ||
    ||  |_|_____________| |  |update________|| |   |**********************| ||
    ||====|===============|=================|===========^===================||            Network
          |                                 |           |                          ||**********************||
          V   Storage                       |           |--------------------------|| MaxPrivilege - max   ||
    |**************|********************|   V           ^                          || allowed privilege on ||
    |  pam_unix -  | pam_ipmi- encrypted|   |           |                          || channel              ||
    |  /etc/shadow | password (only if  |   |           |              NET-IPMID   **************************
    |  (hashed) or | user in ipmi group)|   |           |    ===========================       |
    |  pam_ldap    |                    |   |           |    || _____________________  ||      |
    |***********************|***********|   |           |    || | RMCP+ login using  | ||      |
                            +---<-----------------<---------<---- clear text password| ||      |
                                            |           |    || |____________________| ||      |
                                            |           |    ||________________________||      |
                                D-Bus Call  |           |    || _____________________  ||      |
                                +------------+          ^    || | Create new user    | ||      |
                                |                       |    || | or delete or       | ||      |
           Common user manager  |      D-Bus Call       |    || | update             | ||      |
    ||==========================V==||<---------------------<----|(Note: Host-IPMID   | ||      |
    ||     phosphor-user-manager   ||                   |    || | must use same logic| ||      |
    ||                             ||                   |    || |____________________| ||      |
    ||======================|======||                   |    ||                        ||      |
                            V                           |    || |********************| ||      |
                            |                           ^    || | IPMI specific 1:1  | ||      |
                            |                           |    || | user mappings      | ||      |
                            +------>-------------->-----+------>| storage            |<--------|
                            PropertiesChanged /              || | Note: Either Host  | ||
                            InterfacesAdded /                || | / Net IPMID must   | ||
                            InterfacesRemoved /              || | implement signal   | ||
                            UserRenamed Signals              || | handler            | ||
                                                             || |********************| ||
                                                             ||========================||

```

## 使用者管理 - 總覽

```ascii
                             user management
          +---------------------------------------------------------+
          |                phosphor-user-manager                    |
          |    +---------------------------------------------+      |
          |    | Local user management:                      |      |
          |    | PATH: /xyz/openbmc_project/user             |      |
          |    | I: Manager                                  |      |
          |    | M: CreateGroup, CreateUser, CreateUser2,    |      |
          |    | DeleteGroup, GetUserInfo, RenameUser        |      |
          |    | P: AllPrivileges, AllGroups                 |      |
          |    | S: UserRenamed                              |      |
          |    |                                             |      |
          |    | I: AccountPolicy                            |      |
          |    | P: MaxLoginAttemptBeforeLockout,            |      |
          |    | AccountUnlockTimeout, MinPasswordLength,    |      |
          |    | RememberOldPasswordTimes                    |      |
          |    |                                             |      |
          |    | PATH: /xyz/openbmc_project/user/<name>      |      |
          |    | I: Attributes                               |      |
          |    | P: PasswordExpiration, RemoteUser,          |      |
          |    | TOTPSecretkeyRequired, UserEnabled,         |      |
          |    | UserGroups, UserLockedForFailedAttempt,     |      |
          |    | UserPasswordExpired, UserPrivilege, UserType|      |
          |    |                                             |      |
          |    | General API (Local/Remote)                  |      |
          |    | M: GetUserInfo()                            |      |
          |    |                                             |      |
          |    +---------------------------------------------+      |
          |                                                         |
          |        Remote User Management - Configuration           |
          |    +--------------------------+------------------+      |
          |    |  Provides interface for remote              |      |
          |    |  user management configuration              |      |
          |    |  (LDAP / NIS / KRB5)                        |      |
          |    +---------------------------------------------+      |
          |                                                         |
          +---------------------------------------------------------+

```

## OpenBMC - 使用者管理 - 自 webserver 建立使用者的流程 - 包含所有群組

```ascii
------------------------------------|---------------------------------- -|-----------------------------|
WEBSERVER                           | Common User Manager                |   IPMI & REDFISH(webserver) |
------------------------------------|------------------------------------|-----------------------------|
1.Request to add new user           |                                    |                             |
with 'ipmi, redfish' Group.         |                                    |                             |
Webserver sends D-Bus command       |                                    |                             |
to user-manager with User Name,     |                                    |                             |
Groups and Privilege (No Password)  |                                    |                             |
                             (REQ)--------->                             |                             |
                                    | 2. Validate inputs, including      |                             |
                                    | group restrictions.                |                             |
                                    | if successful go to step 4, else   |                             |
                                    | return a corresponding error, (e.g.|                             |
                                    | too many users, user name is too   |                             |
                                    | long (say, more than IPMI required |                             |
                                    | 16 bytes etc.)                     |                             |
                            <-------------(FAILURE)                      |                             |
3. Throw error to the user          |                                    |                             |
                                    |                                    |                             |
                                    | 4. Add User Name, Groups and       |                             |
                                    | Privileges accordingly. Send signal|                             |
                                    | for User Name created         ---(SIGNAL)->                      |
                                    | (InterfacesAdded)                  | 5. IPMI & REDFISH can       |
                                    |                                    | register for 'UserUpdate'   |
                                    |                                    | signal handler and use      |
                                    |                                    | the same to maintain 1:1    |
                                    |                                    | mapping, if required        |
                            <-------------(SUCCESS)                      |                             |
6. User created successfully but    |                                    |                             |
still can't login, as there is no   |                                    |                             |
password set.                       |                                    |                             |
                                    |                                    |                             |
7.Set the password for              |                                    |                             |
the user using pam_chauthtok()      |                                    |                             |
(which will store clear-text        |                                    |                             |
password in encrypted form, if      |                                    |                             |
user is part of 'ipmi' Group)       |                                    |                             |
                                    |                                    |                             |
8. User created successfully        |                                    |                             |
--------------------------------------------------------------------------------------------------------
```

## OpenBMC - 使用者管理 - 自 IPMI 建立使用者 - 僅限 'ipmi' 群組

```ascii
------------------------------------|---------------------------------- -|-----------------------------|
IPMI                                | Common User Manager                |   pam_unix/pam_ipmi storage |
------------------------------------|------------------------------------|-----------------------------|
1. User sends Set User Name command.|                                    |                             |
IPMI Sends D-Bus command to         |                                    |                             |
to user-manager with User Name      |                                    |                             |
Groups & privileges (No Password) (REQ)--------->                        |                             |
(Note: Configurations for other     |                                    |                             |
commands like Set User Access /     |                                    |                             |
Channel access has to be stored     |                                    |                             |
in IPMI NV along with User Name)    |                                    |                             |
                                    | 2. Validate inputs, including      |                             |
                                    | group restrictions.                |                             |
                                    | if success go to step 4, else      |                             |
                                    | return a corresponding error.      |                             |
                            <-------------(FAILURE)                      |                             |
3. Return error to the Set User     |                                    |                             |
Name command. User creation failed. |                                    |                             |
                                    | 4. Add User Name, Groups and       |                             |
                                    | Privileges accordingly. Send signal|                             |
                                    | for User Name created              |                             |
                            <-------------(SIGNAL) (InterfacesAdded)     |                             |
5. IPMI can register for            |                                    |                             |
'UserUpdate' signal handler and use |                                    |                             |
the same to maintain 1:1 mapping,   |                                    |                             |
if required                         |                                    |                             |
                            <-------------(SUCCESS)                      |                             |
6. User sends Set User Password     |                                    |                             |
command. IPMI uses pam-ipmi         |                                    |                             |
to set the password.                |                                    |                             |
(Note: can't accept password without|                                    |                             |
Set User Name command in first      |                                    |                             |
place as User Name has to exist     |                                    |                             |
before trying to store the password.|                                    |                             |
Implementation can elect to accept  |                                    |                             |
the password and set it through PAM |                                    |                             |
after Set User Name command)   (SET)----------------------------------------> pam-ipmi will store      |
                                    |                                    |clear text password in       |
                                    |                                    |encrypted form along with    |
                                    |                                    |hashed password by pam-unix, |
                                    |                                    |when the user belongs to IPMI|
                            <----------------------------------------------(SUCCESS)                   |
7. User created successfully        |                                    |                             |
but will allow only when user is    |                                    |                             |
enabled. IPMI shouldn't allow user  |                                    |                             |
to login if user is disabled        |                                    |                             |
(Note: stored in IPMI NV).          |                                    |                             |
--------------------------------------------------------------------------------------------------------
```

## OpenBMC - 使用者管理 - 自 webserver 刪除使用者的流程 - 包含所有群組

```ascii
------------------------------------|---------------------------------- -|-----------------------------|
WEBSERVER                           | Common User Manager                |   IPMI & REDFISH(webserver) |
------------------------------------|------------------------------------|-----------------------------|
1.Request to delete existing user   |                                    |                             |
with 'ipmi, redfish' group          |                                    |                             |
Webserver sends D-Bus command       |                                    |                             |
to user-manager with User Name      |                                    |                             |
to be deleted            (REQ)--------->                                 |                             |
                                    | 2. Validate inputs, including      |                             |
                                    | group restrictions.                |                             |
                                    | if successful go to step 4, else   |                             |
                                    | return corresponding error.        |                             |
                                    | Can be user does not exist etc.    |                             |
                            <-------------(FAILURE)                      |                             |
3. Throw error to the user          |                                    |                             |
                                    |                                    |                             |
                                    | 4. Delete User Name, Group and     |                             |
                                    | privileges along with password.    |                             |
                                    | User Name deleted             ---(SIGNAL)->                      |
                                    | (InterfacesRemoved)                | 5. IPMI & REDFISH can       |
                                    |                                    | register for 'UserUpdate'   |
                                    |                                    | signal handler and use      |
                                    |                                    | the same to maintain 1:1    |
                                    |                                    | mapping, if required        |
                                    |                                    | IPMI must delete stored     |
                                    |                                    | encrypted password.         |
                            <-------------(SUCCESS)                      |                             |
6. User deleted successfully,       |                                    |                             |
                                    |                                    |                             |
--------------------------------------------------------------------------------------------------------
```

## OpenBMC - 使用者管理 - 自 IPMI 刪除使用者 - 僅限 'ipmi' 群組

```ascii
------------------------------------|---------------------------------- -|
IPMI                                | Common User Manager                |
------------------------------------|------------------------------------|
1. User sends Set User Name command |                                    |
to clear user name. Send D-Bus API  |                                    |
to user-manager with User Name      |                                    |
to delete                      (REQ)--------->                           |
(Note: Configurations for other     |                                    |
commands like Set User Access /     |                                    |
Channel access has to be stored     |                                    |
in IPMI NV along with User Name)    |                                    |
                                    | 2. Validate inputs, including      |
                                    | group restrictions.                |
                                    | if successful go to step 4, else   |
                                    | return a corresponding error (e.g. |
                                    | User name doesn't exists etc.)     |
                            <-------------(FAILURE)                      |
3. Return error to the Set User     |                                    |
Name command. User deletion failed. |                                    |
                                    |                                    |
                            <-------------(SIGNAL) 4. User Name deleted  |
5. IPMI can register for            | (InterfacesRemoved)                |
'UserUpdate' signal handler and use |                                    |
the same to maintain 1:1 mapping,   |                                    |
if required                         |                                    |
                            <-------------(SUCCESS)                      |
6. User deleted successfully        |                                    |
                                    |                                    |
--------------------------------------------------------------------------
```

## 驗證流程

應用程式必須使用 `pam_authenticate()` API 來驗證使用者。系統採用堆疊式（stacked）
PAM 模組，使得 `pam_authenticate()` 可同時用於本機與遠端使用者。

```ascii
                +----------------------------------+
                |    Stacked PAM Authentication    |
                |     +-----------------------+    |
                |     | pam_faillock.so       |    |
                |     | user failed attempt   |    |
                |     | tracking module.      |    |
                |     +-----------------------+    |
                |                                  |
                |     +-----------------------+    |
                |     | pam_unix.so / local   |    |
                |     | user authentication   |    |
                |     | module.               |    |
                |     +-----------------------+    |
                |                                  |
                |     +-----------------------+    |
                |     | pam_ldap.so / ldap    |    |
                |     | authentication        |    |
                |     | pam module            |    |
                |     +-----------------------+    |
                +----------------------------------+
```

## 密碼更新

應用程式必須使用 `pam_chauthtok()` API 來設定 / 變更使用者密碼。堆疊式 PAM 模組
可讓所有 'ipmi' 群組使用者的密碼以加密形式儲存，供 IPMI 使用。此作業由載入於
堆疊式 PAM 模組中的 `pam_ipmicheck` 與 `pam_ipmisave` 模組執行。變更使用者的
'ipmi' 群組成員身分時，必須強制密碼到期，以使 IPMI 憑證得以更新。

```ascii
                +------------------+---------------+
                |      Stacked PAM - Password      |
                |                                  |
                |  +----------------------------+  |
                |  | pam_pwquality.so.          |  |
                |  | Strength checking the      |  |
                |  | password for acceptance    |  |
                |  +----------------------------+  |
                |                                  |
                |  +----------------------------+  |
                |  | pam_ipmicheck.so. Checks   |  |
                |  | password acceptance for    |  |
                |  | 'ipmi' group users         |  |
                |  +----------------------------+  |
                |                                  |
                |  +----------------------------+  |
                |  | pam_pwhistory.so. Checks   |  |
                |  | to avoid last used         |  |
                |  | passwords                  |  |
                |  +----------------------------+  |
                |                                  |
                |  +-------------+--------------+  |
                |  | pam_unix.so - to update    |  |
                |  | local user's password      |  |
                |  |                            |  |
                |  +----------------------------+  |
                |                                  |
                |  +-----------------+----------+  |
                |  | pam_ipmisave.so - stores   |  |
                |  | 'ipmi' group user's        |  |
                |  | password in encrypted form |  |
                |  +----------------------------+  |
                |                                  |
                +----------------------------------+
```

## 授權流程（IPMI 除外）

```ascii
                                +
                                |
                                |
                  +-------------v--------------+
                  |pam_authenticate() to       |
                  |authenticate the user       |
                  |(local / remote)            |
                  +-------------+--------------+
                                |
                                |
                  +-------------v--------------+
                  |Read user properties using  |
                  |GetUserInfo() (for local &  |
                  |remote users).              |
                  |Allow group access based on |
                  |group property              |
                  +-------------+--------------+
                                |
                                |
                  +-------------v--------------+
                  |Read Channel MaxPrivilege   |
                  |from /xyz/openbmc_project/  |
                  |network/ethX. Use the       |
                  |minimum of user & channel   |
                  |privilege as the privilege  |
                  |Note: Implementation can    |
                  |elect to skip the same, if  |
                  |authorization based on      |
                  |channel restriction is not  |
                  |needed.                     |
                  +----------------------------+


```

## LDAP

SSH、Redfish、Webserver 與 HostConsole 介面允許使用者對 LDAP 目錄進行驗證。IPMI
介面無法用於對 LDAP 進行驗證，因為 IPMI 在建立工作階段時需要明文密碼。

在 OpenBMC 中實作了基於 PAM 的驗證，因此無論是 LDAP 使用者還是本機使用者，其驗證
流程都相同。

對於 LDAP 使用者帳戶，並沒有對應到 OpenBMC 權限角色的 LDAP 屬性型別。建議的做法
是將 LDAP 使用者帳戶歸類到 LDAP 群組中。系統提供了 D-Bus API，讓使用者可將權限
角色指派給 LDAP 群組。

## 授權流程

本節說明 webserver 介面如何取用使用者帳戶的權限角色。對於本機使用者，權限角色是
使用者 D-Bus 物件的一項屬性。對於 LDAP 使用者帳戶，權限角色將以 LDAP 群組為依據。
在使用 LDAP 使用者帳戶進行驗證之前，必須先設定好 LDAP 群組與權限角色的對應關係。

1. 呼叫 PAM API，以使用者憑證進行驗證。若驗證成功則繼續進行。
2. 檢查該使用者是否為本機使用者帳戶。若為本機帳戶，則從 D-Bus 物件取得權限角色，
   並更新工作階段資訊。
3. 若該使用者帳戶並非本機帳戶，則讀取該使用者的群組名稱。
4. 取得對應於該群組名稱的權限角色，並以該權限角色更新工作階段資訊。
5. 若群組名稱與權限角色之間沒有對應關係，則拒絕建立工作階段。必須設定好 LDAP
   群組與權限角色的對應關係，使用者才能成功驗證。

## 建議實作方式

1. 應預設加入 IPMI_NULL_USER，並讓該使用者保持停用狀態。這是為了防止
   IPMI_NULL_USER 被建立為一位實際的使用者。由於從 Unix 使用者管理的觀點來看，
   IPMI 中帶有 NULL 密碼的 NULL 使用者無法作為一筆項目新增，因此有此需要。
2. IPMI 規格並不支援使用者管理的群組。因此可以透過 OEM 指令來實作，從而在 IPMI
   中建立具有不同群組角色的使用者。
3. 不要使用 'Set User Name' IPMI 指令，將已存在的非 ipmi 群組使用者擴充至 'ipmi'
   群組。'Set User Name' IPMI 指令無法區分這是新增使用者的請求，還是將既有使用者
   擴充至 'ipmi' 群組的請求。請使用 OEM 指令將既有使用者擴充至 'ipmi' 群組。
   註：若嘗試新增系統中已存在的使用者，'Set User Name' IPMI 指令會回傳完成碼
   CCh 'Invalid data field in Request'。

## 部署 - 出廠時

### 指導方針

根據
[SB-327 資訊隱私法案](https://leginfo.legislature.ca.gov/faces/billTextClient.xhtml?bill_id=201720180SB327)，
連網裝置必須避免以通用的使用者名稱與密碼出貨。所期望的合理安全性為：

1. 每個裝置各別唯一的預燒密碼
2. 在使用裝置之前，強制使用者產生新的驗證帳戶。

### 部署期間建立使用者

為了遵循上述指導方針並讓 OpenBMC 更為安全，本設計規範了在使用者透過任一系統帶內
介面（例如 KCS 等）進行部署時，強制終端使用者產生一個新的帳戶。IPMI 2.0 規格書
提供了 `SetUserName`、`SetUserPassword`、`SetUserAccess` 等指令，必須使用這些
指令來建立新的使用者帳戶，而非使用任何通用的預設使用者名稱與密碼。以此方式建立的
帳戶可存取 IPMI、REDFISH 與 Webserver，並可用於透過帶外界面建立更多帳戶。

### 特殊使用者 - root - user id 0

預設向終端使用者暴露 root 帳戶（user id 0）（除錯 / 開發情境除外）是一種安全性
風險。因此目前的架構建議預設不為終端使用者啟用 root 使用者。若要為除錯 / 開發
版本進行一般登入，可透過在 local.conf.sample 檔案中指定，來建立一位帶有密碼的
新預設使用者。這可用於預設建立工作階段（CI 系統等可使用此帳戶）。可從 OpenBMC
套件中加入使用者名稱 `openbmc`，密碼為 `0penBmc$`。

#### 除錯使用情境

在開發 / 除錯階段，需要 `root` 使用者 / sudo 權限存取。為此目的，可使用新的
IPMI OEM 指令（待定）/ REDFISH OEM 動作（待定）來為 root 使用者設定密碼，之後
即可使用 `root` 使用者登入序列主控台以進行進一步除錯（註：從使用者管理的觀點來看，
`root` 使用者不會在 IPMI / REDFISH 等任何介面中被列為使用者帳戶）。

### 無帶外界面的系統之部署

任何不具備帶內系統介面的裝置，可為每一台裝置產生唯一的密碼，或暴露一組預設的使用者
名稱與密碼，並在使用裝置之前強制終端使用者更新之（待定）。
