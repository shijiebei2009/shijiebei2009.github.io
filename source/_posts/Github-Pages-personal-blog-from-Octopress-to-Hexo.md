title: "Github Pages个人博客，从Octopress转向Hexo"
date: 2015-04-06 16:03:13
tags: [Hexo, Github, Octopress]
categories: Git/GitHub

---

>环境&版本
OS：win7 X64
Hexo：V3.0.0
Node.js：V0.12.2
Git：Version 1.9.5.msysgit.1

关于为什么要开博客？请参见[《为什么你要写博客？》](http://zhuanlan.zhihu.com/cnfeat/19743861)[《我的博客时代》](http://zhuanlan.zhihu.com/cnfeat/19743255)

下面就让我们一起开启使用Hexo的全新旅程吧！
### 安装Node.js
[下载Node.js](https://nodejs.org/download/)
参考地址：[安装Node.js](http://www.w3cschool.cc/nodejs/nodejs-install-setup.html)
### 安装Git
下载地址：http://git-scm.com/download/
### 注册GitHub
访问：http://www.github.com/
注册过程参见：[一步步在GitHub上创建博客主页 全系列](http://pchou.info/web-build/2013/01/03/build-github-blog-page-01.html)
### 配置和使用Github
参见：[如何搭建一个独立博客——简明Github Pages与Hexo教程](http://www.jianshu.com/p/05289a4bc8b2)
### 安装Hexo及主题设置
#### 安装hexo
``` bash
$ cd /d/
$ mkdir hexo
$ cd hexo
$ npm install -g hexo
$ hexo init
$ hexo g # 或者hexo generate
$ hexo s # 或者hexo server，可以在http://localhost:4000/查看
```
注意在使用npm安装hexo的时候，如果在Git Bash中出现
```bash
sh.exe": npm: command not found
```
那么需要右击Git Bash以管理员身份运行，再次在Git Bash中输入**npm install -g hexo**即可。
#### 复制主题
``` bash
$ hexo clean
$ git clone https://github.com/wuchong/jacman.git themes/jacman
```
#### 启用主题
修改Hexo目录下的_config.yml配置文件中的theme属性，将其设置为jacman。
#### 更新主题
``` bash
$ cd themes/jacman
$ git pull
$ hexo g # 生成
$ hexo s # 启动本地服务，进行文章预览调试 
```
在浏览器中输入http://localhost:4000/
如果显示的是繁体中文，那么修改`_config.xml`中的`language: zh-CN`。
#### 绑定独立域名
[购买域名](http://www.net.cn/)
在你的域名注册提供商那里配置DNS解析，获取GitHub的IP地址[点击](https://help.github.com/articles/tips-for-configuring-an-a-record-with-your-dns-provider/)，进入source目录下，添加CNAME文件
``` bash
$ cd source/
$ touch CNAME
$ vim CNAME # 输入你的域名，例如codepub.cn
$ git add CNAME
$ git commit -m "add CNAME"
```
修改`_config.xml`文件，添加你的Github中仓库地址，该仓库名称必须是`yourusername.github.io`，添加如下内容到`_config.xml`中

``` bash
deploy:
  type: git
  repository: git@github.com:shijiebei2009/shijiebei2009.github.io.git # 注意换成自己的username
  branch: master
```
不会建仓库的童鞋参见[hexo系列教程：（二）搭建hexo博客](http://zipperary.com/2013/05/28/hexo-guide-2/)
#### 推送到远程仓库
``` bash
$ git init
$ git add .
$ git commit -m "first commit"
$ git remote add origin git@github.com:shijiebei2009/shijiebei2009.github.io.git
$ git push -u origin master
```
如果出现
```bash
$ git push -u origin master
Username for 'https://github.com': shijiebei2009
Password for 'https://shijiebei2009@github.com':
remote: Repository not found.
fatal: repository 'git://github.com/shijiebei2009/shijiebei2009.github.io.git/
' not found
```
那是因为你在github网站上还没有建立该仓库导致。
部署
```bash
$ hexo generate
$ hexo deploy #或者组合命令 hexo d -g
```
如果出现：
```bash
ERROR Deployer not found: git
```
需要运行：
```bash
$ npm install hexo-deployer-git --save
```
### 进阶篇-高级定制
#### 添加插件
添加sitemap和feed插件
```bash
$ npm install hexo-generator-feed
$ npm install hexo-generator-sitemap
```
修改`_config.yml`，增加以下内容
```yml
# Extensions
Plugins:
- hexo-generator-feed
- hexo-generator-sitemap

#Feed Atom
feed:
  type: atom
  path: atom.xml
  limit: 20

#sitemap
sitemap:
  path: sitemap.xml
```
配完之后，就可以访问`http://codepub.cn/atom.xml`和`http://codepub.cn/sitemap.xml`，发现这两个文件已经成功生成了。

#### 添加404公益页面
GitHub Pages有提供制作404页面的指引：[Custom 404 Pages](https://help.github.com/articles/custom-404-pages)。

直接在根目录下创建自己的404.html或者404.md就可以。但是自定义404页面仅对绑定顶级域名的项目才起作用，GitHub默认分配的二级域名是不起作用的，使用hexo server在本机调试也是不起作用的。

推荐使用[腾讯公益404](http://www.qq.com/404/)。
#### 添加about页面
```bash
$ hexo new page "about"
```
之后在\source\about\index.md目录下会生成一个index.md文件，打开输入个人信息即可，如果想要添加版权信息，可以在文件末尾添加：

```html
<div style="font-size:12px;border-bottom: #bbbbbb 1px solid; BORDER-LEFT: #bbbbbb 1px solid; BACKGROUND: #f6f6f6; HEIGHT: 120px; BORDER-TOP: #bbbbbb 1px solid; BORDER-RIGHT: #bbbbbb 1px solid" class=shijiebei2009right>
<div style="MARGIN-TOP: 10px; FLOAT: left; MARGIN-LEFT: 5px; MARGIN-RIGHT: 10px">
<IMG alt="" src="https://avatars3.githubusercontent.com/u/4994697?v=3&u=70a4ca810fb1908f2deeed95f3b962eec64e1787&s=140" width=90 height=100>
</div>
<div style="LINE-HEIGHT: 200%; MARGIN-TOP: 10px; COLOR: #000000">作者： 
<a href="http://shijiebei2009.github.io/">王旭</a> <br/>出处： 
<a href="http://shijiebei2009.github.io/">http://shijiebei2009.github.io/</a>
<br/>本文基于<a target="_blank" title="Creative Commons Attribution-ShareAlike 4.0 International (CC BY-SA 4.0)" href="http://creativecommons.org/licenses/by-sa/4.0/"> 知识共享署名-相同方式共享 4.0 </a>
国际许可协议发布，欢迎转载，演绎或用于商业目的，但是必须保留本文的署名 
<a href="http://shijiebei2009.github.io/">王旭</a>及链接。
</div>
</div>
```

#### 发表文章的markdown语法
```yml
title: postName #文章页面上的显示名称，可以任意修改，不会出现在URL中
date: 2013-12-02 15:30:16 #文章生成时间，一般不改，当然也可以任意修改
categories: example #分类
tags: [tag1,tag2,tag3] #文章标签，可空，多标签请用格式，注意:后面有个空格
description: 附加一段文章摘要，字数最好在140字以内。
---

设置摘要有两种方法

1、使用<!--more-->标签
2、不使用<!--more-->标签，仅显示部分摘要。在`D:/hexo/themes/jacman/_config.xml`文件中修改为
index:
  expand: false ## default is unexpanding,so you can only see the short description of each post.
  excerpt_link: Read More

以下正文
```

#### 使用图床
使用[七牛云存储](http://www.qiniu.com/)
或者采用开源的图床，使用新浪SAE平台，但是鉴于不知道能否持久，自己掂量办，[在线图床](http://qiniupicbed.sinaapp.com/)
#### markdown工具
Windows在线markdown工具：https://www.zybuluo.com/mdeditor
Windows本地markdown工具：[markdownpad](http://markdownpad.com/)
#### 添加Fork me on Github
[获取代码](https://github.com/blog/273-github-ribbons)，选择你喜欢的代码添加到hexo/themes/jacman/layout/layout.ejs的末尾即可，注意要将代码里的you改成你的Github账号名。

#### 添加支付宝捐赠按钮及二维码支付
##### 支付宝捐赠按钮
在D:\hexo\themes\jacman\layout\_widget目录下新建一个zhifubao.ejs文件，内容如下
```html
<p class="asidetitle">打赏他</p>
<div>
<form action="https://shenghuo.alipay.com/send/payment/fill.htm" method="POST" target="_blank" accept-charset="GBK">
    <br/>
    <input name="optEmail" type="hidden" value="your 支付宝账号" />
    <input name="payAmount" type="hidden" value="默认捐赠金额(元)" />
    <input id="title" name="title" type="hidden" value="博主，打赏你的！" />
    <input name="memo" type="hidden" value="你Y加油，继续写博客！" />
    <input name="pay" type="image" value="转账" src="http://7xig3q.com1.z0.glb.clouddn.com/alipay-donate-website