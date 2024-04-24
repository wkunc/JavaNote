# Git的概念

# 获取Git仓库

## 在本地创建仓库

进入想要管理的文件夹 执行 git init

## Clone 现有仓库

指令是 git clone [url] [dir]

这个命令会从指定的 url 中获取你要拷贝的项目,并在当前的文件夹中创建, 你也可以通过 dir 参数来控制位置

git 支持多种协议 如: http,git,ssh 等等

# 更新到仓库

文件状态可以分为 已追踪 和 未追踪 , 已追踪的文件可以被修改 进入 modifiy ,但还是已追踪的文件, 然后添加到 暂存区,然后提交到仓库

## 查看文件状态

git status

```
$ git status
On branch master
nothing to commit, working directory clean
```

然后让我创建一个文件,然后使用 git status 就会看到新的容

```
$ echo 'My Project' > README
$ git status
On branch master
Untracked files:
  (use "git add <file>..." to include in what will be committed)

    README

nothing added to commit but untracked files present (use "git add" to track)
```

之前没有的文件会出现在 Untrancked file 栏中, Git 不会自动将其加入追踪范围, 除非我们明确告知它我们想要追踪该文件,
这样可你不比担心将生成的二进制文件加入追踪范围

## 追踪新文件

git add [file name | dir name]
如果是 dir 的话会递归追踪目录下的所有文件, 支持正则
通过提示我们知道可以通过 git add 命令可以开始追踪文件

```
$ git add README
$ git status
On branch master

No commited yet

Changes to be committed:
  (use "git rm --cached <file>..." to unstage)

    new file:   README
```

## 提交更新

git commit

这个指令可以提交当前你所有的在 **暂存区** 的改动
它会打开一个指定的编辑器 然后你可以输入提交说明

## 跳过使用暂存区域

使用暂存区是有必要的,但是在你没有新文件想让 Git 追踪的时候是繁琐的

Git 提供了一个跳过暂存区的方式 ,只要在 git commit -a ,这个参数会自动把所有已经跟踪过的文件添加到暂存区

##除
