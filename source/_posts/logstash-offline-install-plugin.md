---
title: Logstash离线安装插件
date: 2017-09-29 22:03:22
tags: [Logstash]
categories: Programming Notes

---

如果线上服务器可以连外网的话，当然是用官方提供的命令来安装插件最简单了，但是可惜的是，好多公司线上服务器是没有外网访问权限的，这就需要在使用某些插件的时候，进行离线安装。而离线安装有两种方式，一种是在可以联网的机器上安装插件，之后使用`prepare-offline-pack`命令打包，然后将打包文件上传到不能联网的服务器，再使用`prepare-offline-pack`解包，安装。但是这种方式太麻烦，要求你必须要有一个可以联网的机器，最好还是和不能联网的服务器相同的配置环境，这里推荐一种更好的方案，来解决离线安装插件的问题。

先演示一下，正常的联网环境是如何操作的，如下所示
```shell
[elastic@escluster logstash-5.5.0]$ pwd
/home/elastic/elasticsearch/logstash-5.5.0
[elastic@escluster logstash-5.5.0]$ ./bin/logstash-plugin install logstash-filter-fingerprint
Validating logstash-filter-fingerprint
Installing logstash-filter-fingerprint
Installation successful
```

那么无法联网，首先需要在可以联网的机器上下载对应插件的压缩包，打开[Logstash Plugins](https://github.com/logstash-plugins)地址，直接搜索需要安装的插件名称，然后下载对应的`zip`或者`tar.gz`压缩包即可。也许你会说，我用的Logstash是5.5.0的，但是并没有对应版本的插件啊？不用担心，试一下如下命令，显示已经安装的所有插件
```shell
[elastic@escluster logstash-5.5.0]$ ls -la vendor/bundle/jruby/1.9/gems
total 652
drwxrwxr-x. 203 elastic elastic 8192 Jun 30 23:56 .
drwxrwxr-x.   9 elastic elastic  104 Jun 30 23:56 ..
drwxrwxr-x.   7 elastic elastic 4096 Jun 30 23:56 addressable-2.3.8
drwxrwxr-x.   3 elastic elastic 4096 Jun 30 23:56 arr-pm-0.0.10
drwxrwxr-x.   6 elastic elastic 4096 Jun 30 23:56 atomic-1.1.99-java
drwxrwxr-x.   5 elastic elastic   55 Jun 30 23:56 avl_tree-1.2.1
drwxrwxr-x.   4 elastic elastic 4096 Jun 30 23:56 awesome_print-1.8.0
drwxrwxr-x.   3 elastic elastic   16 Jun 30 23:56 aws-sdk-2.3.22
drwxrwxr-x.   5 elastic elastic  104 Jun 30 23:56 aws-sdk-core-2.3.22
drwxrwxr-x.   3 elastic elastic   16 Jun 30 23:56 aws-sdk-resources-2.3.22
drwxrwxr-x.   5 elastic elastic 4096 Jun 30 23:56 aws-sdk-v1-1.67.0
drwxrwxr-x.   7 elastic elastic 4096 Jun 30 23:56 backports-3.8.0
......
......
```
可以看到，各种版本的插件都有，所以说，插件的版本和Logstash的版本并不要求一致，以安装[`logstash-filter-fingerprint`](https://github.com/logstash-plugins/logstash-filter-fingerprint)插件为例，目前最新版是`v3.1.1`，下载上传到无法联网的服务器上，解压
```shell
[elastic@escluster ~]$ gunzip logstash-filter-fingerprint-3.1.1.tar.gz
[elastic@escluster ~]$ tar -xvf logstash-filter-fingerprint-3.1.1.tar
```
进入Logstash的目录，编辑`Gemfile`文件，在文件开头添加
```shell
gem "logstash-filter-fingerprint", :path => "/home/elastic/logstash-filter-fingerprint-3.1.1"
```
保存退出，执行命令`./bin/logstash-plugin install --no-verify`安装，提示如下信息
```shell
[elastic@escluster logstash-5.5.0]$ ./bin/logstash-plugin install --no-verify
Installing...
LogStash::GemfileError: duplicate gem logstash-filter-fingerprint
             add_gem at /home/elastic/elasticsearch/logstash-5.5.0/lib/pluginmanager/gemfile.rb:109
                 gem at /home/elastic/elasticsearch/logstash-5.5.0/lib/pluginmanager/gemfile.rb:207
              (eval) at (eval):52
       instance_eval at org/jruby/RubyBasicObject.java:1598
               parse at /home/elastic/elasticsearch/logstash-5.5.0/lib/pluginmanager/gemfile.rb:195
                load at /home/elastic/elasticsearch/logstash-5.5.0/lib/pluginmanager/gemfile.rb:19
             gemfile at /home/elastic/elasticsearch/logstash-5.5.0/lib/pluginmanager/command.rb:4
  install_gems_list! at /home/elastic/elasticsearch/logstash-5.5.0/lib/pluginmanager/install.rb:146
             execute at /home/elastic/elasticsearch/logstash-5.5.0/lib/pluginmanager/install.rb:61
                 run at /home/elastic/elasticsearch/logstash-5.5.0/vendor/bundle/jruby/1.9/gems/clamp-0.6.5/lib/clamp/command.rb:67
             execute at /home/elastic/elasticsearch/logstash-5.5.0/vendor/bundle/jruby/1.9/gems/clamp-0.6.5/lib/clamp/subcommand/execution.rb:11
                 run at /home/elastic/elasticsearch/logstash-5.5.0/vendor/bundle/jruby/1.9/gems/clamp-0.6.5/lib/clamp/command.rb:67
                 run at /home/elastic/elasticsearch/logstash-5.5.0/vendor/bundle/jruby/1.9/gems/clamp-0.6.5/lib/clamp/command.rb:132
              (root) at /home/elastic/elasticsearch/logstash-5.5.0/lib/pluginmanager/main.rb:48
```
很清晰，这是说在Gemfile中存在重复的`logstash-filter-fingerprint`，因为在原始的Gemfile中，存在通过联网进行安装的`logstash-filter-fingerprint`，打开Gemfile，找到并注释掉即可
```shell
# gem "logstash-filter-fingerprint"
```
再次执行安装命令，即可安装成功
```shell
[elastic@escluster logstash-5.5.0]$ ./bin/logstash-plugin install --no-verify
Installing...
Installation successful
```