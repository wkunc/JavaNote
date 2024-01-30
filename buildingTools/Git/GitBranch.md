# 分支简介
为了真正理解Git处理分支的方式, 我们需要回顾一下 Git 是如何保存数据的.

Git 保存的不是文件的变化或者差异, 而是一系列不同时刻的 snapshot (快照)

在进行 commit 操作时, Git 会保存一个提交对象(commit object).
该提交对象会包含一个指向 `暂存内容快照` 的指针.
不仅如此提交对象还包含了作者的姓名和邮箱, 提交时输入的信息以及指向父对象的指针.
首次提交的提交对象没有父对象, 普通提交操作产生的提交对象有一个父对象,
而由多个分支合并产生的提交对象有多个父对象.

# Git 分支
Git 的分支, 本质上仅仅是指向提交对象的可变指针.

当你在分支上提交时, 就会创建新的提交对象并且使分支指针指向它,
当然 commit object 会保存它的 父提交 的位置, 就像链表一样.

HEAD 指针代表你当前所在分支, 所以切换分支只是简单的切换HEAD指针指向的分支指针

## 创建分支
使用 git branch [branchname] 可以创建分支

-b 参数可以创建分支并且切换到该分支
```
git branch newBranch // 创建一个新分支
git branch -b otherBranch //创建并切换到新分支上
```
不带参数使用 git branch 可以查看当前所有分支

```
git log --oneline --decorate // 查看提交日志, 并且标明每个分支
git log --oneline --decorate --graph
```

由于Git的分支实质上仅是包含所指对象校验和(长度为40的SHA-1值字符串)的文件,
所以它的创建和销毁都异常高效

## 切换分支
git checkout branchname 可以切换到对应名字的分支.
如果你在当前分支做出的修改并且没有提交的文件和想要切换的分支存在冲突的话, Git 会阻止你切换分支

## 分支合并
使用 git merg branchname 可以将当前分支和指定分支进行合并

Git 的分支合并情况有两种 fast-forward 和 recursive

如果想要合并的分支的祖先是当前分支,那么合并分支就是简单的将 当前分支指针移动到对应分支
这称为 fast-forward

如果想要合并的分支的祖先不是当前分支, 情况会更复杂一些
Git 会自动的计算出两个分支的最近的共同祖先, 然后让来一个三方合并(将共同祖先,当前分支,指定分支一起合并),
这会产生一个合并提交

如果在三方合并的过程中出现了冲突, 就是多个分支中的中的文件的同一个位置有不同的改动
这时 Git 做了合并但是没有自动的创建出一个新的合并提交, Git 会暂停下来, 
等待你解决合并产生的冲突. 你可以使用 git status 来查看哪些因包含冲突而处于未合并状态(unmerged)的文件
```
$ git status
On branch master
You have unmerged paths.
  (fix conflicts and run "git commit")

Unmerged paths:
  (use "git add <file>..." to mark resolution)

    both modified:      index.html

no changes added to commit (use "git add" and/or "git commit -a")
```

Git 会在有冲突的文件中加入标准的冲突解决标记, 这样你可以打开这些冲突文件并手动解决冲突
用 ====== 来分割两个不同分支中的不同版本, 上面也标识了哪个版本是当前分支的样子
你可以选择其中一个将另一个删除,当然也可以使用手动改变成任何你想要的样子
(ps:记得要把 Git 产生的标识删除)
```
<<<<<<< HEAD:index.html
<div id="footer">contact : email.support@github.com</div>
=======
<div id="footer">
 please contact us at support@github.com
</div>
>>>>>>> iss53:index.html
```
然后, 对这些文件使用 git add 命令来将其标记未冲突已解决, 一旦暂存这些冲突文件 Git 就会将它标记为冲突已解决

最后, 当你对结果感到满意, 并且确定之前的冲突的文件都已经暂存了,
这时就可以使用 git commit 命令来完成合并提交了

## 分支删除
当我们完成了上面的分支合并时, 我们就不再需要之前的分支了,
所以可以使用 -d(delete) 选项来执行删除指定分支.
git branch -d branchname 来删除分支, (-d 代表delete 吧)

## 常用分支命令参数

git branch --merged

参数 --merged 会展示所有已经合并的分支. 
在这个列表中没有 * 标注的分支通常可以使用 git branch -d 删除.
因为已经将它们的工作整合到了另一个分支中, 所以不会失去任何东西.

git branch --no-merged

和上面选项相反, 显示所有没有合并的分支.


# 远程分支

远程引用是对远程仓库的引用, 包括分支,标签等等.
可以通过 git ls-remote <remote> 来显示地获取远程引用的完整列表.
