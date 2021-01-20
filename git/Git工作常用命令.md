# Git工作常用命令

## Git配置用户名和邮箱

公司的项目跟自己的项目用户git用户名和邮箱希望是不一样的，今天就学了一下git的全局配置和仓库配置

git用户名和邮箱有全局配置和仓库配置之分

**仓库配置**的优先级高于全局配置

查看全局配置

`git config --list`

设置全局配置

`git config --global user.name "Kakk22"`

`git config --global user.email "XXX@xx.com"`



创建了项目，提交代码到github上，**如果没有为该项目单独配置用户名邮箱，则会使用上面配置的全局的用户名邮箱**。因为本机和github是使用ssh来通信的，本地git的用户名邮箱和github的用户名邮箱不一样也行！

但如果现在你的项目要提交到公司的gitlab上，并且不使用ssh通信，选择了账号和密码通信，那么这个时候就需要配置用户名邮箱，和gitlab的用户名邮箱保持一致。

### 配置单个仓库的用户名和邮箱

在idea的项目 `Terimial`命令行 如下配置

`git config user.name "cyf"`

`git config user.email "xxx@xx.com"`

`git config --list`  

git config --list查看当前配置, 在当前项目下面查看的配置是全局配置+当前仓库的配置, 使用的时候会优先使用当前仓库的配置

## git本地仓库关联远程仓库

### 初始化

在本地需要关联到远程仓库的项目根目录下执行

```bash
git init 
```

然后关联远程仓库 [project]。你需要存在一个远程仓库，名字随意，然后执行下面的命令（去掉中括号）就可以关联到该仓库。

```bash
git remote add origin https://github.com/Kakk22
```

### 提交

```bash
git add .
git commit -m '初始化项目'
git branch --set-upstream-to=origin/master  master  ##设置本地master分支
git pull origin master --allow-unrelated-histories ##如果远程初始化仓库有文件没关联 则先拉取
git push -u origin master
```

