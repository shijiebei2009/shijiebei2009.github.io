title: "使用坚果云同步IntelliJ IDEA的配置文件"
date: 2016-08-09 21:50:39
tags: [IntelliJ]
categories: Programming Notes

---

对于IDEA这样的神器，每个人都必然会有很多个性化的配置，那么如何在多台终端同步IDEA的配置呢？配合强大的坚果云同步功能来自动同步你的配置文件吧。另外坚果云免费版虽然对流量有限制，但是同步一个小小的配置文件夹还是足够了。

此方法也适用于JetBrains家的其它IDE系列产品，稍有不同之处请自行调整。
- IntelliJ IDEA，一套智慧型的Java整合开发工具，特别专注与强调程序员的开发撰写效率提升
- PHPStorm，PHP集成开发工具
- PyCharm，智能Python集成开发工具
- RubyMine，一个为Ruby和Rails开发者准备的IDE，其带有所有开发者必须的功能，并将之紧密集成于便捷的开发环境中
- WebStorm，智能HTML/CSS/JS开发工具
- AppCode，开发Obj-C的IDE，是一个XCode的替代物

IDEA默认配置文件存放位置
- Windows 保存在 C:/Users/username/.IntelliJIdeaXX (XX表示产品的版本号，当前版本是2016.2)
- Unix/Linux 保存在 ~/.IntelliJIdeaXX (~ 就是/home/目录)
- Mac 保存在 ~/Library/Preferences/IntelliJIdeaXX

**注意：**如果是 IntelliJ IDEA Community，那么文件名就是 IdeaICXX。坚果云的客户端安装就不说了，关闭IDEA，从你的默认配置目录里剪切config这个目录到你的坚果云同步目录。如果你想要同步多个工具的配置目录，那么可以为config搭配一个父目录使用。例如我的：
- E:/jianguoyun/IDEA/config &emsp;&emsp;#IDEA的配置文件路径
- E:/jianguoyun/PyCharm/config &emsp;&emsp;#PyCharm的配置文件路径
- E:/jianguoyun/PHPStorm/config &emsp;&emsp;#PHPStorm的配置文件路径

打开IDEA安装目录`C:/Program Files (x86)/JetBrains/IntelliJ IDEA 2016.2/bin/idea.properties`，找到第8行，把idea.config.path前面的#号去掉，=号后面的路径修改成之前的配置文件夹的目录，例如IDEA修改为`idea.config.path=E:/jianguoyun/IDEA/config`。记得把路径中的“\”换成“/”。

第二天到公司直接修改idea.properties即可，前提是同样安装了坚果云客户端哦。