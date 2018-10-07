# Git 分支
Git 分支其实是个指针指向 commit object, 当你在分支上提交时,就会创建新的提交对象并且使分支指针指向它,
当然 提交对象 会保存它的 父亲位置, 就像链表一样

HEAD 指针代表你当前所在分支, 所以切换分支只是简单的切换HEAD指针指向的分支指针

## 创建分支
使用 git branch [branchname] 可以创建分支

-b 参数可以创建分支并且切换到该分支
```
git branch newBranch
git branch -b otherBranch
```
不带名字使用 git branch 可以查看当前所有分支

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
当我们完成了上面的分支合并时, 我们就不再需要之前的分支了, 所以可以使用
git branch -d branchname 来删除分支, (-d 代表delete 吧)