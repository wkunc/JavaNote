# Git repository(仓库)
## 初始化本地仓库
初始化本地仓库非常的简单, 只要在你希望由 Git 管理的项目中执行 git init 命令
```
git init
```
这个命令会在当前位置创建一个名为 .git 的隐藏目录

到这里我们就成功的将项目变成一个 Git repository 但是这里我们只是初始化了我们的仓库, 
项目里面的文件并没有被 track(追踪)
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

# 撤销操作
## 
有时候我们提交完了才发现漏掉了几个文件没有添加, 或者提交信息写错了
此时可以运行带有 **--amend** 选项的提交命令
```
git commit --amend
```
这个命令会将暂存区的文件提交, 如果自上次提交以来你还未做出任何修改, 
那么快照会保持不变而你所修改的只是提交信息

文本编辑器启动后你可以看到之前的提交信息. 编辑后保存就会覆盖原来的提交信息

例如: 你提交之后发现忘记暂存某些需要的修改, 可以像下面这样操作
```
git commit -m "initial commit"
git add forgotten_file
git commit --amend
```

最终你将会只有一个提交, 第二个提交替代第一个提交

### 取消暂存文件
比如你已经修改了两个文件并且想将它们作为连个独立的修改提交, 但是却不小心暂存了两个文件.
如何取消暂存中的一个呢? **git status** 命令会提示你
```
$ git add *
$ git status
On branch master
# 这里就是提示
Changes to be committed:
  (use "git reset HEAD <file>..." to unstage)

    renamed:    README.md -> README
    modified:   CONTRIBUTING.md
```
使用 **git reset HEAD<file>...** 来取消暂存, 所以我们这样来取消暂存 CONTRIBUTING.md
```
$ git reset HEAD CONTRIBUTING.md
Unstaged changes after reset:
M	CONTRIBUTING.md
$ git status
On branch master
Changes to be committed:
  (use "git reset HEAD <file>..." to unstage)

    renamed:    README.md -> README

Changes not staged for commit:
  (use "git add <file>..." to update what will be committed)
  (use "git checkout -- <file>..." to discard changes in working directory)

    modified:   CONTRIBUTING.md
```

> 虽然在调用时加上 --hard 选项可以令 git reset 
> 成为一个危险的命令译注: (可能导致工作目录中所有当前进度丢失!), 但本例中工作目录内的文件并不会被修改.
> 不加选项地调用 git reset 并不危险 — 它只会修改暂存区域

## 撤销对文件的修改
如果你并不想保留对 CONTRIBUTING.md 文件的修改怎么办? 
你该如何方便地撤消修改 - 将它还原成上次提交时的样子(或者刚克隆完的样子, 或者刚把它放入工作目录时的样子)?
幸运的是, git status 也告诉了你应该如何做 在最后一个例子中，未暂存区域是这样：

使用 git checkout -- <file> 可以做到
```
Changes not staged for commit:
  (use "git add <file>..." to update what will be committed)
  (use "git checkout -- <file>..." to discard changes in working directory)

    modified:   CONTRIBUTING.md
它非常清楚地告诉了你如何撤消之前所做的修改。 让我们来按照提示执行：
$ git checkout -- CONTRIBUTING.md
$ git status
On branch master
Changes to be committed:
  (use "git reset HEAD <file>..." to unstage)

    renamed:    README.md -> README
```
可以看到那些修改已经被撤消了。

### 注意
你需要知道 git checkout -- [file] 是一个危险的命令,这很重要 你对那个文件做的任何修改都会消失 
- 你只是拷贝了另一个文件来覆盖它 除非你确实清楚不想要那个文件了, 否则不要使用这个命令
