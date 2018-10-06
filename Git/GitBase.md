# Git repository(仓库)
## 初始化本地仓库
初始化本地仓库非常的简单, 只要在你希望由 Git 管理的项目中执行 git init 命令
```
git init
```
这个命令会在当前位置创建一个名为 .git 的隐藏目录

到这里我们就成功的将项目变成一个 Git repository 但是这里我们只是初始化了我们的仓库, 项目里面的文件并没有被 track(追踪)
## 克隆现有仓库
可以使用 git clone [url] 的命令来 clone 一个现有的仓库
```
# 比如克隆这个笔记仓库
git clone https://github.com/wkunc/JavaNote.git
```

# 文件状态
![](./imgs/lifecycle.png)
上面我们初始化了本地仓库但是项目文件并没有被 Track(跟踪)

## 追踪新文件
Untracked 的文件不会被 Git 记录, 如果想要这个文件被追踪可以使用
```
git add [file|path]
```
这样指定的 file | path 就加入了 Git 的控制之下, 并且文件就到了 staged (暂存区)
## 将修改写入暂存区(Stage)
使用 git add 命令也可以将 已修改未提交的文件加入 staged

## 检查文件状态
使用 git status 命令可以查看哪些文件处于什么状态
```
git status
```
## 提交更新
在合适的时候,我们就可以提交更新啦使用 git commit 命令来提交暂存区的内容
```
git commit
```
这种方式会启动你配置的文本编辑器, 让你输入这次提交的 message (信息)
也可以使用 参数 -m 来将提交信息与命令放在同一行
```
git commit -m "commit message"
```
记住普通的 git commit 指令永远只是提交 Stage (暂存区) 的内容
## 跳过使用暂存区
使用暂存区的方式可以精心准备要提交的细节, 但有时又会显得特别繁琐
Git 提供了一个跳过暂存区的使用方法 -a 参数
```
git commit -a -m "commit message"
```
-a 是 --all 简写 代表 commit all changed files

这里的changed file 只是那些已修改未提交的文件, 并不会提交哪些被创建的新文件(即没有加入 track 的文件)
## 查看提交历史
提交之后就可以用 git log 命令来查看提交历史啦
```
git log
commit c056469fe77e79c97daf17718ab6b72c83bb9e4d (HEAD -> master, origin/master)
Author: wkunc <w879587095@hotmail.com>
Date:   Fri Sep 28 19:59:24 2018 +0800

    commit message
```
