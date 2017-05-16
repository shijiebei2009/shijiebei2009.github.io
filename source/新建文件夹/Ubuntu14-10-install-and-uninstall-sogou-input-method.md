title: Ubuntu 14.10安装和卸载搜狗拼音输入法
date: 2015-11-27 22:58:34
tags: [Ubuntu]
categories: Operating System

---

###安装搜狗输入法
- 参考了其他一些资料都说添加`fcitx`的`PPA`，但是我不确定有木有用

```bash
eric@eric-VirtualBox:/etc/apt$ sudo add-apt-repository ppa:fcitx-team/nightly
 Experimental releases of Fcitx, use with caution.
 More info: https://launchpad.net/~fcitx-team/+archive/ubuntu/nightly
Press [ENTER] to continue or ctrl-c to cancel adding it

gpg: keyring `/tmp/tmp_2dkioxm/secring.gpg' created
gpg: keyring `/tmp/tmp_2dkioxm/pubring.gpg' created
gpg: requesting key 7E5FA1EE from hkp server keyserver.ubuntu.com
gpg: /tmp/tmp_2dkioxm/trustdb.gpg: trustdb created
gpg: key 7E5FA1EE: public key "Launchpad PPA for Fcitx Team PPA" imported
gpg: Total number processed: 1
gpg:               imported: 1  (RSA: 1)
OK
```

- 然后刷新软件源，安装一些依赖软件

```bash
sudo apt-get update
```
刷新完成之后，安装`im-config`
```bash
sudo apt-get install im-config
```
继续安装搜狗输入法依赖的`fcitx`一系列文件，输入命令：`sudo apt-get -f install`，遇到有`Y/N`的地方直接输`Y`
```bash
eric@eric-VirtualBox:~/Desktop$ sudo apt-get -f install
Reading package lists... Done
Building dependency tree
Reading state information... Done
Correcting dependencies... Done
The following extra packages will be installed:
  fcitx fcitx-bin fcitx-config-common fcitx-config-gtk fcitx-data
  fcitx-frontend-all fcitx-frontend-gtk2 fcitx-frontend-gtk3
  fcitx-frontend-qt4 fcitx-libs fcitx-libs-gclient fcitx-libs-qt
  fcitx-module-dbus fcitx-module-kimpanel fcitx-module-lua fcitx-module-x11
  fcitx-modules fcitx-ui-classic
Suggested packages:
  fcitx-tools fcitx-m17n kdebase-bin plasma-widgets-kimpanel
Recommended packages:
  fcitx-frontend-qt5
The following NEW packages will be installed:
  fcitx fcitx-bin fcitx-config-common fcitx-config-gtk fcitx-data
  fcitx-frontend-all fcitx-frontend-gtk2 fcitx-frontend-gtk3
  fcitx-frontend-qt4 fcitx-libs fcitx-libs-gclient fcitx-libs-qt
  fcitx-module-dbus fcitx-module-kimpanel fcitx-module-lua fcitx-module-x11
  fcitx-modules fcitx-ui-classic
0 upgraded, 18 newly installed, 0 to remove and 7 not upgraded.
1 not fully installed or removed.
Need to get 35.6 kB/2,315 kB of archives.
After this operation, 8,956 kB of additional disk space will be used.
Do you want to continue? [Y/n] y
WARNING: The following packages cannot be authenticated!
  fcitx-config-common fcitx-config-gtk
Install these packages without verification? [y/N] y
Get:1 http://mirrors.aliyun.com/ubuntu/ precise/universe fcitx-config-common all 0.4.0-2 [3,548 B]
Get:2 http://mirrors.aliyun.com/ubuntu/ precise/universe fcitx-config-gtk amd64 0.4.0-2 [32.1 kB]
Fetched 35.6 kB in 0s (247 kB/s)
Selecting previously unselected package fcitx-libs:amd64.
(Reading database ... 168557 files and directories currently installed.)
Preparing to unpack .../fcitx-libs_1%3a4.2.8.5-2~utopic1_amd64.deb ...
Unpacking fcitx-libs:amd64 (1:4.2.8.5-2~utopic1) ...
Selecting previously unselected package fcitx-bin.
Preparing to unpack .../fcitx-bin_1%3a4.2.8.5-2~utopic1_amd64.deb ...
Unpacking fcitx-bin (1:4.2.8.5-2~utopic1) ...
Selecting previously unselected package fcitx-data.
Preparing to unpack .../fcitx-data_1%3a4.2.8.5-2~utopic1_all.deb ...
Unpacking fcitx-data (1:4.2.8.5-2~utopic1) ...
Selecting previously unselected package fcitx-modules.
Preparing to unpack .../fcitx-modules_1%3a4.2.8.5-2~utopic1_amd64.deb ...
Unpacking fcitx-modules (1:4.2.8.5-2~utopic1) ...
Selecting previously unselected package fcitx.
Preparing to unpack .../fcitx_1%3a4.2.8.5-2~utopic1_all.deb ...
Unpacking fcitx (1:4.2.8.5-2~utopic1) ...
Selecting previously unselected package fcitx-libs-gclient:amd64.
Preparing to unpack .../fcitx-libs-gclient_1%3a4.2.8.5-2~utopic1_amd64.deb ...
Unpacking fcitx-libs-gclient:amd64 (1:4.2.8.5-2~utopic1) ...
Selecting previously unselected package fcitx-module-dbus.
Preparing to unpack .../fcitx-module-dbus_1%3a4.2.8.5-2~utopic1_amd64.deb ...
Unpacking fcitx-module-dbus (1:4.2.8.5-2~utopic1) ...
Selecting previously unselected package fcitx-frontend-gtk2:amd64.
Preparing to unpack .../fcitx-frontend-gtk2_1%3a4.2.8.5-2~utopic1_amd64.deb ...
Unpacking fcitx-frontend-gtk2:amd64 (1:4.2.8.5-2~utopic1) ...
Selecting previously unselected package fcitx-frontend-gtk3:amd64.
Preparing to unpack .../fcitx-frontend-gtk3_1%3a4.2.8.5-2~utopic1_amd64.deb ...
Unpacking fcitx-frontend-gtk3:amd64 (1:4.2.8.5-2~utopic1) ...
Selecting previously unselected package fcitx-libs-qt:amd64.
Preparing to unpack .../fcitx-libs-qt_1%3a4.2.8.5-2~utopic1_amd64.deb ...
Unpacking fcitx-libs-qt:amd64 (1:4.2.8.5-2~utopic1) ...
Selecting previously unselected package fcitx-frontend-qt4:amd64.
Preparing to unpack .../fcitx-frontend-qt4_1%3a4.2.8.5-2~utopic1_amd64.deb ...
Unpacking fcitx-frontend-qt4:amd64 (1:4.2.8.5-2~utopic1) ...
Selecting previously unselected package fcitx-module-kimpanel.
Preparing to unpack .../fcitx-module-kimpanel_1%3a4.2.8.5-2~utopic1_amd64.deb ...
Unpacking fcitx-module-kimpanel (1:4.2.8.5-2~utopic1) ...
Selecting previously unselected package fcitx-config-common.
Preparing to unpack .../fcitx-config-common_0.4.0-2_all.deb ...
Unpacking fcitx-config-common (0.4.0-2) ...
Selecting previously unselected package fcitx-config-gtk.
Preparing to unpack .../fcitx-config-gtk_0.4.0-2_amd64.deb ...
Unpacking fcitx-config-gtk (0.4.0-2) ...
Selecting previously unselected package fcitx-frontend-all.
Preparing to unpack .../fcitx-frontend-all_1%3a4.2.8.5-2~utopic1_all.deb ...
Unpacking fcitx-frontend-all (1:4.2.8.5-2~utopic1) ...
Selecting previously unselected package fcitx-module-lua.
Preparing to unpack .../fcitx-module-lua_1%3a4.2.8.5-2~utopic1_amd64.deb ...
Unpacking fcitx-module-lua (1:4.2.8.5-2~utopic1) ...
Selecting previously unselected package fcitx-module-x11.
Preparing to unpack .../fcitx-module-x11_1%3a4.2.8.5-2~utopic1_amd64.deb ...
Unpacking fcitx-module-x11 (1:4.2.8.5-2~utopic1) ...
Selecting previously unselected package fcitx-ui-classic.
Preparing to unpack .../fcitx-ui-classic_1%3a4.2.8.5-2~utopic1_amd64.deb ...
Unpacking fcitx-ui-classic (1:4.2.8.5-2~utopic1) ...
Processing triggers for man-db (2.7.0.2-2) ...
Processing triggers for shared-mime-info (1.2-0ubuntu3) ...
Processing triggers for gnome-menus (3.10.1-0ubuntu2) ...
Processing triggers for desktop-file-utils (0.22-1ubuntu2) ...
Processing triggers for bamfdaemon (0.5.1+14.10.20140925-0ubuntu1) ...
Rebuilding /usr/share/applications/bamf-2.index...
Processing triggers for mime-support (3.55ubuntu1) ...
Processing triggers for hicolor-icon-theme (0.13-1) ...
Processing triggers for libgtk2.0-0:amd64 (2.24.25-0ubuntu1) ...
Processing triggers for libgtk-3-0:amd64 (3.12.2-0ubuntu15) ...
```

- 到搜狗官网下载搜狗拼音输入法，选择你系统对应的软件包，我系统是64位的，所以我选择了amd64的

下载地址[点我](http://pinyin.sogou.com/linux/?r=pinyin)，另外搜狗也提供了安装说明帮助页面，[点我](http://pinyin.sogou.com/linux/help.php)查看帮助。


- 下载完成之后，可以直接将文件放在桌面上进行安装

先看看下载的`deb`文件
```bash
eric@eric-VirtualBox:~/Desktop$ ll
total 18304
drwxr-xr-x  2 eric eric     4096 11月 27 22:06 ./
drwxr-xr-x 18 eric eric     4096 11月 27 21:34 ../
-rw-rw-r--  1 eric eric      627 11月 27 22:06 cache.txt
-rwxr-x---  1 root root 18730988 11月 27 00:01 sogoupinyin_2.0.0.0068_amd64.deb*
eric@eric-VirtualBox:~/Desktop$
```
输入`sudo dpkg -i sogoupinyin_2.0.0.0068_amd64.deb`命令开始安装输入法，可使用`tab`键自动补全命令，安装完成之后使用如下命令启用`sudo im-config -s fcitx -z default`，完成之后别忘记重启系统。重启之后点击右上角的那个软件盘即可看到搜狗输入法安装成功，点击即可输入汉字了。
![](http://7xig3q.com1.z0.glb.clouddn.com/ubuntu-sogou-input-method-success.jpg)

###卸载搜狗输入法

- 首先使用命令查看下安装的搜狗拼音输入法`sudo dpkg -l so*`，然后先卸载搜狗拼音`sudo apt-get purge sogoupinyin`

![](http://7xig3q.com1.z0.glb.clouddn.com/ubuntu-uninstall-sogou-input-method1.jpg)

- 卸载fcitx，`sudo apt-get purge fcitx`

![](http://7xig3q.com1.z0.glb.clouddn.com/ubuntu-uninstall-sogou-input-method2.jpg)

- 彻底卸载fcitx及相关配置，`sudo apt-get autoremove`

![](http://7xig3q.com1.z0.glb.clouddn.com/ubuntu-uninstall-sogou-input-method3.jpg)

- 最后别忘注销或者重启系统，如果注销按钮不能使用，可以使用命令`sudo pkill Xorg`，当再次登录系统之后，可以看到搜狗输入法已经完全被卸载干净了。