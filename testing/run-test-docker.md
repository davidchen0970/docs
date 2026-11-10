# 使用 Docker 執行 OpenBMC 自動化測試

使用 Docker 執行 OpenBMC 自動化，包含建立 Docker 映像檔，接著在
Docker 容器內部執行自動化測試。

## 建置 Docker 映像檔

`Note: Prerequisite is to have Docker installed.`

1. 建立一個工作區目錄。

   `mkdir ${HOME}/OpenBMC_Automation`

2. 切換至所建立的工作區目錄。

   `cd ${HOME}/OpenBMC_Automation`

3. 複製 openbmc-build-scripts 倉儲。

   `git clone https://github.com/openbmc/openbmc-build-scripts`

4. 切換至 openbmc-build-scripts 目錄。

   `cd openbmc-build-scripts`

5. 建置執行 robot 測試所需的 Docker 映像檔（首次建置可能需要將近
   15 分鐘）。預設的 Docker 映像檔名稱為
   "openbmc/ubuntu-robot-qemu"。你可以使用 "docker images"
   指令來檢視映像檔。

   `./scripts/build-qemu-robot-docker.sh`

**Note:** 當你的 Docker 位於代理伺服器之後時，請在建置指令中加入以下參數
（請使用正確的 IP 與 PORT 值。）

```bash
   --build-arg http_proxy=<IP>:<PORT> --build-arg https_proxy=<IP>:<PORT>
```

## 使用 robot 測試程式碼進行程式碼更新

1. 切換至 HOME 工作區目錄。

   `cd ${HOME}/OpenBMC_Automation`

2. 複製 openbmc-test-automation 倉儲。

   `git clone https://github.com/openbmc/openbmc-test-automation`

3. 執行 docker run 以啟動 BMC 程式碼更新。

   **Note:** 請先下載 BMC 韌體映像檔 (`_.all.tar`) 再執行此步驟。
   下方的 `BMC\*IMG_PATH` 指向這個已下載的檔案。

   ```bash
   docker run --user root \
             --env HOME=${HOME} \
             --workdir ${HOME} \
             --volume ${HOME}/OpenBMC_Automation:${HOME} \
             --tty openbmc/ubuntu-robot-qemu python -m robot \
                 -v OPENBMC_HOST:<BMC IP> \
                 -v FILE_PATH:<BMC_IMG_PATH> \
                 -i Initiate_Code_Update_BMC \
                 ${HOME}/openbmc-test-automation/extended/code_update/update_bmc.robot
   ```

   以下範例使用 witherspoon-20170614071422.all.tar 映像檔，
   該檔案位於執行 docker run 指令之系統的 HOME 目錄中：

   ```bash
   docker run --user root \
             --env HOME=${HOME} \
             --workdir ${HOME} \
             --volume ${HOME}/OpenBMC_Automation:${HOME} \
             --tty openbmc/ubuntu-robot-qemu python -m robot \
                 -v OPENBMC_HOST:1.11.222.333 \
                 -v FILE_PATH:/home/witherspoon-20170614071422.all.tar \
                 -i Initiate_Code_Update_BMC \
                 ${HOME}/openbmc-test-automation/extended/code_update/update_bmc.robot
   ```

4. 程式碼更新完成後，robot framework 執行所產生的記錄檔將存放於以下位置：

   ```bash
   ${HOME}/OpenBMC_Automation/log.html
   ${HOME}/OpenBMC_Automation/report.html
   ${HOME}/OpenBMC_Automation/output.xml
   ```

## 執行自動化測試

1. 執行 docker run 以執行 OpenBMC 自動化測試案例。

   **Note:** 這會使用 openbmc/ubuntu-robot-qemu 映像檔來啟動一個 Docker 容器。

   Robot 測試程式碼會透過 run-robot.sh
   腳本擷取並在此容器上執行。

   ```bash
   docker run --user root \
             --env HOME=${HOME} \
             --env IP_ADDR=<BMC IP> \
             --env SSH_PORT=22 \
             --env HTTPS_PORT=443 \
             --env ROBOT_TEST_CMD="tox -e <System Type> -- <Robot Cmd>" \
             --workdir ${HOME} \
             --volume ${WORKSPACE}:${HOME} \
             --tty openbmc/ubuntu-robot-qemu \
                 ${HOME}/openbmc-build-scripts/scripts/run-robot.sh
   ```

   執行整個測試套件（test suite）的範例：

   ```bash
   docker run --user root \
             --env HOME=${HOME} \
             --env IP_ADDR=1.11.222.333 \
             --env SSH_PORT=22 \
             --env HTTPS_PORT=443 \
             --env ROBOT_TEST_CMD="tox -e witherspoon -- tests" \
             --workdir ${HOME} \
             --volume ${HOME}/OpenBMC_Automation:${HOME} \
             --tty openbmc/ubuntu-robot-qemu \
                 ${HOME}/openbmc-build-scripts/scripts/run-robot.sh
   ```

2. 執行完成後，測試結果將存放於以下檔案中。

   ```bash
   ${HOME}/OpenBMC_Automation/log.html
   ${HOME}/OpenBMC_Automation/report.html
   ${HOME}/OpenBMC_Automation/output.xml`
   ```
