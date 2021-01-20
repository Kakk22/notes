# Git在本地不提交前提下，拉去远程最新代码

今天在写代码时，同事叫拉取远程仓库最新代码，本地代码还未完成并不想提交。

那么如何在不提交代码前提下，拉去最新代码呢？

在git bash中使用如下命令
```bash
##显示Git栈内的所有备份，可以利用这个列表来决定从那个地方恢复。
git stash list
##暂存本地修改
git stash 
##拉取最新代码
git pull
##合并暂存的代码
git stash pop
##清空本地暂存栈信息
git stash clear
```
但是在 使用`git stash pop`合并时发现以下错误
```bash
$ git stash pop
error: Your local changes to the following files would be overwritten by merge:
        lib/generated/intl/messages_en.dart
        lib/generated/intl/messages_zh.dart
        lib/generated/l10n.dart
Please commit your changes or stash them before you merge.
Aborting
```

**原因是**：该文件的代码已被更改，而远程仓库的代码没有改动，`git pull` 拉下代码后，执行`git stash pop`发现该文件不一样，导致无法从缓冲区里提取代码。

**解决方法**：将该文件copy一份，然后从项目中删除，再执行`git stash pop`，之后在对文件进行修改即可。