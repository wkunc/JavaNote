# 初次运行前的配置
Git 自带一个 git config 的工具来帮助设置控制 Git 的外观和行为的配置变量

这些配置存储在 
* /etc/gitconfig :包含系统上每个用户的设置 --system 选项的 git config 时
* ~/.gitconfig 或 ~/.config/git/config :只是当前用户的设置 --global 选项 的 git config 
* 当前仓库的 .git/config :针对该仓库

每个级别覆盖上一级别的配置, 用户的 gitconfig 覆盖全局配置, 仓库配置覆盖上面连个配置

## 用户信息
配置用户信息
```
git config --global user.name "youname"
git config --global user.email johndoe@example.com
git config --global core.editor vim
```
检查配置信息可以使用 git config --list 命令来列出所有找到的配置
```
git config --list
user.name='wkunc'
user.email='w879587095@hotmail.com'
core.repositoryformatversion=0
core.filemode=false
core.bare=false
core.logallrefupdates=true
core.symlinks=false
core.ignorecase=true
```
也可以用 git config \<key> 来检查某一项配置
