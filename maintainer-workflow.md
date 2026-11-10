# OpenBMC 維護者／CLA 工作流程

OpenBMC 的貢獻者在他們的貢獻被接受之前，必須先簽訂 OpenBMC CLA（貢獻者授權協議，Contributor License
Agreement）。本頁面是供子專案維護者在核准補丁前參照的檢查清單。

- 手動確認貢獻者已簽署 ICLA（個人），或已列名於既有的 CCLA（企業）之中。
  - 已簽訂的 CLA 可以在 [CLA 存放庫][1]找到。
  - 如果你尚未被加入適當的 CLA 存放庫 ACL，請發信到
    `openbmc@lists.ozlabs.org` 提出要求加入。
  - 若找到該貢獻者的 CLA，即接受該補丁(1)。
- 若找不到 CLA，請要求貢獻者簽訂一份並寄送到
  `manager@lfprojects.org`。
  - 在已簽署的 CLA（個人 _或_ 企業）上傳到 CLA 存放庫之前，請勿接受該補丁(1)。
  - CCLA 表單請見 [CCLA 範本][2]。
  - ICLA 表單請見 [ICLA 範本][3]。

接受對 OpenBMC 之中上游專案分岔（fork）的貢獻，例如 Linux 核心或 U-Boot，_不_ 需要已簽訂的 OpenBMC CLA。

請參閱 [貢獻指南](./CONTRIBUTING.md)中維護者的職責。維護者最終要負責處理開放原始碼授權問題、
使用擷取自網路的程式碼所產生的問題，以及維持程式碼的品質。

存放庫維護者應該具備以下經同儕共識所認可的特質：

- 盡責（responsible）：持續致力於確保只有高品質的程式碼進入了存放庫
- 領導力（leadership）：培育具備開放原始碼意識的實務，例如 [FOSS][4]
- 專業知識（expertise）：通常以對程式碼的重大貢獻或程式碼審查來展現

(1) 接受補丁的意涵取決於該子專案的貢獻流程。

- GitHub pull request —— 合併該 pull request。
- Gerrit —— +2。
- email —— 合併該補丁。

請確保已接受的變更確實合併進了 OpenBMC 存放庫。

[1]: https://drive.google.com/drive/folders/1Ooi0RdTcaOWF1DWFJUAJDdN7tRKde7Nl
[2]: https://drive.google.com/file/d/1d-2M8ng_Dl2j1odsvZ8o1QHAdHB-pNSH
[3]: https://drive.google.com/file/d/1k3fc7JPgzKdItEfyIoLxMCVbPUhTwooY
[4]: https://en.wikipedia.org/wiki/Free_and_open-source_software
