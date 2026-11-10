# Subtree 架構

過去，meta-layer 存放庫是以 subtree 的方式加入本 repo。現在已經不再是這樣了。
雖然各個 meta-\* tree 為了保留完整的歷史紀錄，仍然存在于 gerrit 和 github 上，
但已經不應該再使用它們，新的 meta layer 只會直接簽入（check in）到
openbmc/openbmc 程式碼 tree 當中。
