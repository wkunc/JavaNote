# SELinux (Security Enhanced Linux)


## 问题发现
IT运维项目的自动安装脚本是特殊的, 没有安装linux目录规范进行Beat服务安装. 在环境项目上自动安装Beat时碰到了问题.
> 安装位置可以在application.yml中指定, 默认为 /home/metricbeat 等目录下, 主配置文件就在 /home/metricbeat/metricbeat.yml
> 标准的安装应该放在 /usr/share/metricbeat/bin 目录下放可执行程序, /etc/metricbeat 下放配置文件

具体现象
1. `systemctl status metricbeat` 时提示, service not found.
用`systemctl list-unit-files --type=service` 命令可以看到 state 栏为bad
但是使用 `systemd-analyze verify` 未提示unit文件存在错误.
最后误打误撞用vim复制了一遍文件内容解决(ps: 原因应该还是selinux导致的)

2. `systemctl start metricbeat`启动失败, 但是手动命令行可以启动成功

## 排查过程
由于SELinux拒绝了`systemd`调用beat的命令, 所以通过 `systemctl status metricbeat` 并不能发现错误原因,只能看到(exit code 203)
最后通过 `journalctl -xe` 发现日志中存在权限拒绝, 且搜索`exit code 203`发现许多文章提到了`selinux`时基本确定原因.

`audit2why`等命令可以分析SELinux的audit.log给出原因
```
type=AVC msg=audit(1716541528.048:199393): avc:  denied  { read } for  pid=1 comm="systemd" name="filebeat.service" dev="dm-0" ino=538007929 scontext=system_u:system_r:init_t:s0 tcontext=unconfined_u:object_r:user_home_t:s0 tclass=file pe rmissive=0

        Was caused by:
                Missing type enforcement (TE) allow rule.

                You can use audit2allow to generate a loadable module to allow this access.
```
由于默认安装位置为 /home 目录下, 导致解压出的文件类型都是`user_home_t`, 而systemd的范围是`init_t` 所以导致执行命令被拒绝.
而之前公司部署的环境中由于k8s部署时需要关闭`selinux`, 所以在公司等k8s环境下部署beat没碰到过错误

> PS: 没有找到如何查询`SELinux`的类型配置的信息
> 如果可以知道 init_t 和哪些文件类型匹配, 可以通过手动修改文件 selinux-type 标签来赋予权限

## 解决办法
手动修改部署, 将部署方式改成标准的目录.

> 1. 也可以采用修改SELiunx策略等办法, 但是没搞明白且存在风险. 所以放弃该方案
> 2. 经过测试发现把安装地址配置为 `/usr/share` 可以正常启动

## 相关连接

1. [ReaHat的SELinux文档](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html/selinux_users_and_administrators_guide/part_i-selinux)
> 不同系统版本的相关文档内容不同, 可以多看看不同版本的文档

2. [csdn上翻到的指令示例比较全面的文章](https://blog.csdn.net/sinat_41942180/article/details/134225509)
