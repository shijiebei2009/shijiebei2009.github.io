title: "Github Pages个人博客，从Octopress转向Hexo"
date: 2015-04-06 16:03:13
tags: [Hexo, Github, Octopress]
categories: Git/GitHub

---

环境&版本
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
如果显示的是繁体中文，那么修改_config.xml中的language: zh-CN。
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
修改_config.xml文件，添加你的Github中仓库地址，该仓库名称必须是 yourusername.github.io，添加如下内容到_config.xml中

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
修改_config.yml，增加以下内容
```
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
```
title: postName #文章页面上的显示名称，可以任意修改，不会出现在URL中
date: 2013-12-02 15:30:16 #文章生成时间，一般不改，当然也可以任意修改
categories: example #分类
tags: [tag1,tag2,tag3] #文章标签，可空，多标签请用格式，注意:后面有个空格
description: 附加一段文章摘要，字数最好在140字以内。
---

设置摘要有两种方法

1、使用<!--more-->标签
2、不使用<!--more-->标签，仅显示部分摘要。在 D:/hexo/themes/jacman/_config.xml文件中修改为
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
    <input name="pay" type="image" value="转账" src="http://7xig3q.com1.z0.glb.clouddn.com/alipay-donate-website.png" />
</form>
</div>
```
添加完该文件之后，要在D:/hexo/themes/jacman/_config.yml文件中启用，如下所示，添加zhifubao
```bash
widgets:
- category
- tag
- links
- tagcloud
- zhifubao
- rss
```

##### 二维码捐赠
首先需要到[这里](https://qr.alipay.com/paipai/open.htm)获取你的支付宝账户的二维码图片，支付宝提供了自定义功能，可以添加自定义文字。

我的二维码扫描捐赠添加在about页面，当然你也可以添加到其它页面，在D:\hexo\source\about下有index.md，打开，在适当位置添加
```html
<center>
欢迎您捐赠本站，您的支持是我最大的动力！
![][1]
[1]: http://7xig3q.com1.z0.glb.clouddn.com/alipay-donate.png
</center>
<br/>
```
`<center>`可以让图片居中显示，注意将图片链接地址换成你的即可。

### 其它实用功能
#### 插件推荐
[hexo官方文档](http://hexo.io/docs/)
[hexo插件大全](https://github.com/hexojs/hexo/wiki/Plugins)
[自定义网站logo](http://www.faviconer.com/)
[MarkDown中文网](http://www.markdown.cn/)
[MarkDown语法说明](http://wowubuntu.com/markdown/)
[社交分享推荐使用jiathis](http://www.jiathis.com/)
[评论插件推荐使用duoshuo](http://duoshuo.com/)
[网站流量统计推荐cnzz](http://zhanzhang.cnzz.com/)
[网站流量统计推荐百度统计](http://tongji.baidu.com/web/welcome/login)
#### 作者信息
需要修改与作者有关的一系列信息，修改D:/hexo/themes/jacman/_config.xml中的author/imglogo/favicon/author_img/apple_icon一系列属性即可。

#### 删除warning: LF will be replaced by CRLF警告信息
在hexo deploy时，有时会出现这个提示信息warning: LF will be replaced by CRLF，虽然看起来挺乱糟糟的，但不影响使用，可以忽略不计。若想不提示，可以使用如下方法：
切换到博客的根目录，执行如下命令：
```bash
$ git config --global core.autocrlf false 
$ rm -rf .git #删除掉该目录下的.git文件夹
$ git init #重新初始化
```
再deploy试试吧，清新脱俗了。
#### 写博客或添加页面
```bash
$ hexo new "postName" #新建文章
$ hexo new page "pageName" #新建页面
$ hexo generate #生成静态页面至public目录
$ hexo server #开启预览访问端口（默认端口4000，'ctrl + c'关闭server）
$ hexo deploy #将.deploy目录部署到GitHub
```
常用简写
```bash
$ hexo n == hexo new
$ hexo g == hexo generate
$ hexo s == hexo server
$ hexo d == hexo deploy
```
常用组合
```bash
$ hexo d -g #生成部署
$ hexo s -g #生成预览
```
#### 安装hexo-generator-baidu-sitemap插件，专为百度量身打造
```bash
$ npm install hexo-generator-baidu-sitemap --save
```
然后在 Hexo 根目录下的 _config.yml 里配置一下
```
baidusitemap:
 path: baidusitemap.xml
```
#### 推广博客与提交Sitemap
[百度网址提交入口](http://zhanzhang.baidu.com/sitesubmit/index)
[360网址提交入口](http://info.so.360.cn/site_submit.html)
#### 添加百度站内搜索
[点击进入](http://zhanzhang.baidu.com/guide/index)，点击其它工具->站内检索->现在使用->新建搜索引擎->查看代码，将代码里的id值复制，打开/d/hexo/themes/jacman/_config.xml，配置成如下即可。

```
baidu_search:     ## http://zn.baidu.com/
  enable: true
  id: "1433674487421172828" ## e.g. "783281470518440642"  for your baidu search id
  site: http://zhannei.baidu.com/cse/search ## your can change to your site instead of the default site
```
#### 使用不蒜子添加访客统计
详情参考[搞定你的网站计数](http://ibruce.info/2015/04/04/busuanzi/)，具体做法很简单，就是在你的`themes/your themes/layout/_partial/footer.ejs`底部加入这段脚本
```javascript
<script async src="//dn-lbstatics.qbox.me/busuanzi/2.3/busuanzi.pure.mini.js"></script>
```
然后在`<p class="copyright"></p>`中间添加如下统计信息即可
```html
本站总访问量 <span id="busuanzi_value_site_pv"></span> 次, 访客数 <span id="busuanzi_value_site_uv"></span> 人次, 本文总阅读量 <span id="busuanzi_value_page_pv"></span> 次
```
不蒜子的官方服务网站是[不蒜子](http://service.ibruce.info/)，目前最大的弊端就是不开放注册，所以对于运行了一段时间的网站，不蒜子的数据都是从1开始，没办法设置，只有等后期开放注册之后，登入网站才能对统计计数进行设置。

###Jacman主题相关
####为jacman主题添加最新评论
本方法针对使用hexo搭建Github Pages静态博客，并且使用jacman主题的童鞋们。

首先在`\themes\jacman\layout\_widget`目录下新建`latest_comment.ejs`，放入“多说”最新评论代码，其中“多说”的最新评论代码[点我获取](http://dev.duoshuo.com/docs/4ff28d95552860f21f000010)，注意修改***var duoshuoQuery = {short_name:"您的多说二级域名"};***将其中的“short_name”设置为在多说配置的二级域名即可。在`latest_comment.ejs`的首行注意添加
```html
<p class="asidetitle">最新评论</p>
```
然后进入`\themes\jacman\_config.yml`：在`widgets`下添加`latest_comment`即可，注意Windows下编码一定要采用UTF-8无BOM编码。

####为jacman主题添加热评文章
与上一条类似，首先在`\themes\jacman\layout\_widget`目录下新建`hot_comments.ejs`，然后去你在多说网站的后台管理界面`http://codepub.duoshuo.com/admin/tools/`中点击工具->热评文章获取代码，将代码复制到`hot_comments.ejs`文件中。同样在行首添加
```html
<p class="asidetitle">热评文章</p>
```
然后在`_config.yml`中启用该widgets即可。

####Jacman主题问题解答
Q：如何添加数学公式`mathjax`？
A：主题支持写`LaTex`数学公式。只需要在文章文件开头的`front-matter`中，加上一行`mathjax: true`，即可在文中写`LaTex`公式。

Q：自定义字体 ShowCustomFont
A：是否启用自定义字体，默认开启，主要用于显示网站底部的字体。如果你有一定前端基础可以修改 **font.styl** 替换为你喜欢的字体。

Q：图片默认都是居左的，我怎么设置能让图片居中呢？
A：使用 `<img src="" style="display:block;margin:auto"/>`的HTML标签或者是使用`<center>`包裹图片。

Q：如何建立一篇图片类文章（Gallery Post）？
A：使用**hexo new photo "your titile"**建立图片类文章，或者直接新建一个 Markdown 文件，将其**front-matter**修改为如下，即可看到主题为图片类文章提供的样式。
```bash
---
layout: photo
title: Gallery Post
photos:
- http://i.minus.com/ibobbTlfxZgITW.jpg
- http://i.minus.com/iedpg90Y0exFS.jpg
---
```

Q：我在配置文件中给某一项设置了值，但为什么总是看不到效果啊？
A：`_config.yml`文件中的每个属性值前面必须留一个空格，建议在`Sublime/Notepad++`中开启显示所有空格模式。另每篇文章的`front-matter`也要注意这个问题。

Q：如何建立自我介绍页面（About页面）？
A：首先在主目录找到`_config.yml`，找到url添加`about_dir: about`到这个板块。然后在/source里面建立about文件夹。在about文件夹里建立index.md。编辑index.md就和发布其他的文章一样，格式都一样。

Q：楼主我不喜欢你的配色，怎么换主题的颜色呢？
A：包括颜色在内的很多变量都在`jacman/source/css/_base/variable.styl`文件中，可以修改成你喜欢的。

参考资料
【1】http://wuchong.me/blog/2014/11/20/how-to-use-jacman/#
【2】https://pages.github.com/
【3】http://www.jianshu.com/p/05289a4bc8b2
【4】http://www.pchou.info/web-build/2014/07/04/build-github-blog-page-08.html
【5】http://wuchong.me/blog/2014/11/20/how-to-use-jacman/
