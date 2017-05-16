title: Windows客户端配置GitHub的SSH公钥
date: 2015-06-02 15:44:10
tags: [Git]
categories: Git/GitHub
---

###检查SSH keys的设置
```bash
$ cd ~/.ssh/
```
如果显示"No such file or directory"，跳到第三步，否则继续。
###备份和移除原来的SSH key设置
如果已经存在key文件，需要备份该数据并删除之
```bash
$ ls
id_rsa  id_rsa.pub  known_hosts
$ mkdir key_backup
$ cp id_rsa* key_backup/
$ rm id_rsa*
```
###生成新的SSH key
输入下面的代码，可以生成新的key文件，只需要使用默认的设置即可，当需要输入文件名的时候，回车即可

```bash
$ ssh-keygen -t rsa -C "你的邮箱@qq.com"
Generating public/private rsa key pair.
Enter file in which to save the key (/c/Users/WX/.ssh/id_rsa):
Enter passphrase (empty for no passphrase):
Enter same passphrase again:
Your identification has been saved in /c/Users/WX/.ssh/id_rsa.
Your public key has been saved in /c/Users/WX/.ssh/id_rsa.pub.
The key fingerprint is:
1d:cc:7e:b3:7e:92:f7:ab:c6:75:56:73:62:30:bc:8c 你的邮箱@qq.com
The key's randomart image is:
+--[ RSA 2048]----+
|           .     |
|         o  +    |
|          +o +   |
|         oE.o o.o|
|        S o o. .+|
|           . o .o|
|            o....|
|           .ooo  |
|            o=.oo|
+-----------------+
```
###添加SSH key到GitHub
用文本编辑工具打开id_rsa.pub文件，如果看不到这个文件，你需要设置显示隐藏文件。准确的复制这个文件的内容，才能保证设置的成功。
在GitHub的主页上点击设置按钮，选择SSH Keys项，把复制的内容粘贴进去，然后点击Add Key按钮即可，Title任意选择。

###测试一下
输入下面命令，测试是否设置成功
```bash
$ ssh -T git@github.com
Hi shijiebei2009! You've successfully authenticated, but GitHub does not provide shell access.
```

###设置你的账号信息
现在你已经可以通过SSH链接到GitHub了，还有一些个人信息需要完善的。

Git会根据用户的名字和邮箱来记录提交。GitHub也是用这些信息来做权限的处理，输入下面的代码进行个人信息的设置，把名称和邮箱替换成你自己的，名字必须是你的真名，而不是GitHub的昵称。
```bash
$ git config --global user.name "你的名字"
$ git config --global user.email "your_email@youremail.com"
```
###解决本地多个SSH key问题
如果在eclipse或者其它云平台，可能也需要使用SSH key来认证，如果每次都覆盖原来的is_rsa文件，那么之前的认证就失效了，这个问题可以通过在~/.ssh目录下增加config文件来解决。
####依然需要配置git用户名和邮箱
```bash
git config user.name "用户名"
git config user.email "邮箱"
```
####生成ssh key时同时指定保存的文件名
```bash
ssh-keygen -t rsa -f ~/.ssh/id_rsa.codepub -C "email"
```
上面的id_rsa.codepub就是我们指定的文件名，这时~/.ssh目录下会多出id_rsa.codepub和id_rsa.codepub.pub两个文件，id_rsa.codepub.pub里保存的就是我们要使用的key。

####新增并配置config文件
如果config文件不存在，先添加；存在则直接修改
```bash
touch ~/.ssh/config
```
在config文件里添加如下内容(User表示你的用户名)
```bash
Host *.codepub.cn
IdentityFile ~/.ssh/id_rsa.codepub
User admin
```
####上传key到云平台后台并测试
```bash
sh -T git@git.codepub.cn
```
成功的话，会输出welcome欢迎信息！
日后如需添加，则按照上述配置生成key，并修改config文件即可。

参考资料
【1】http://beiyuu.com/github-pages/
【2】http://riny.net/2014/git-ssh-key/