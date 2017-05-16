title: "CentOS 7.1 切换命令行模式与桌面模式"
tags: [CentOS]
categories: Operating System
date: 2015-06-22 19:37:06
toc: false

---

首先你需要知道自己的Linux版本信息，下面介绍一些常用的查看Linux系统版本的命令
1. 查看内核版本命令，以下三个命令任选
```bash
[hadoop@localhost ~]$ cat /proc/version
Linux version 3.10.0-229.el7.x86_64 (builder@kbuilder.dev.centos.org) (gcc version 4.8.2 20140120 (Red Hat 4.8.2-16) (GCC) ) #1 SMP Fri Mar 6 11:36:42 UTC 2015

[hadoop@localhost ~]$ uname -a
Linux localhost.localdomain 3.10.0-229.el7.x86_64 #1 SMP Fri Mar 6 11:36:42 UTC 2015 x86_64 x86_64 x86_64 GNU/Linux

[hadoop@localhost ~]$ uname -r
3.10.0-229.el7.x86_64
```
2. 查看linux版本
```bash
[hadoop@localhost ~]$ cat /etc/redhat-release 
CentOS Linux release 7.1.1503 (Core) 
```

那么知道了版本之后如何修改默认启动时进入命令行还是桌面环境呢？在`CentOS7.x`之前的版本都是通过修改`/etc/inittab`文件来设置启动顺序，具体可参考[这里](http://www.habadog.com/2012/03/03/centos-model-switch/)。但是此种方法并不适应于`CentOS7.x`版本，在该版本中，我们查看`/etc/inittab`文件可得
```bash
[hadoop@localhost ~]$ vim /etc/inittab
# inittab is no longer used when using systemd.
#
# ADDING CONFIGURATION HERE WILL HAVE NO EFFECT ON YOUR SYSTEM.
#
# Ctrl-Alt-Delete is handled by /usr/lib/systemd/system/ctrl-alt-del.target
#
# systemd uses 'targets' instead of runlevels. By default, there are two main targets:
#
# multi-user.target: analogous to runlevel 3
# graphical.target: analogous to runlevel 5
#
# To view current default target, run:
# systemctl get-default
#
# To set a default target, run:
# systemctl set-default TARGET.target
```
该文件中已经详细说明了，不再使用inittab文件而是使用systemd代替，并且还指出，现在只有`multi-user`相当于运行级别是3和`graphical`相当于运行级别是5，现在可以使用如下命令设置默认启动级别了，注意要以root用户，或者是使用sudo权限
```bash
[root@localhost hadoop]# systemctl set-default multi-user.target
rm '/etc/systemd/system/default.target'
ln -s '/usr/lib/systemd/system/multi-user.target' '/etc/systemd/system/default.target'
[root@localhost hadoop]# 
```
设置成功之后reboot一下，即可顺利进入命令行界面了，如果想要再次进入图形界面，在命令行中运行`startx`即可。

