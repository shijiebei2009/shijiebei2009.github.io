title: 多说已死，切换Hexo博客评论插件到Disqus
date: 2017-03-23 22:26:35
tags: [Hexo]
categories: Git/GitHub

---

本站点之前的评论插件一直用的都是多说，作为一款免费的第三方社会化评论插件，总体来说，多说做的还算可以，唯独其号称智能的防垃圾评论系统，就像空气人一样，完全无用，导致多说垃圾评论泛滥，令人作呕。恰逢最近多说宣称要进行业务转型，自然评论系统也要关闭，国内的目前比较好的评论系统只有畅言不错，但是畅言需要备案，而我不愿意备案，无奈只能选Disqus了，所以将本站点的多说评论转成Disqus了。

因为Disqus在国内被墙，所以使用Disqus需要自带翻墙功能或者说需要自带科学上网功能，否则无法加载评论框，自然也就无法评论了，这是我天朝一特色，除了这非常蛋疼的一点，Disqus做得非常好。切换评论系统，首要任务是将评论数据转移到新的系统中，这样基本上就大功告成了。转移评论数据参考[多说评论迁移至Disqus](http://urouge.github.io/migrate-to-disqus/)，亲测有效，已经成功转移，需要注意一点，在从多说导出数据的时候，选择工具，导出数据，一定要勾选如下两个选项
- 包含文章数据
- 包含评论数据

这样在使用脚本解析的时候，才不会报错。

启用Disqus，需要编辑站点和主题的`_config.yml`文件，添加`disqus_shortname`字段（先搜索，如果有就不用），设置如下
> disqus_shortname: your-disqus-shortname

如需取消某个页面的评论，在`md`文件的`front-matter`中增加
> comments: false

关于Next.Mist主题启用Disqus评论详情可以参考[这里](https://github.com/iissnan/hexo-theme-next/wiki/%E8%AE%BE%E7%BD%AE%E5%A4%9A%E8%AF%B4-DISQUS)。

同样之前使用的多说分享，以及多说热评等都将被停用，分享推荐使用百度分享，但是Next主题并不支持百度分享，证据在[这里](https://github.com/iissnan/hexo-theme-next/issues/425)，所以只能使用JiaThis分享代替，但是JiaThis的侧栏式分享有BUG，会铺满整个屏幕，不推荐使用，经过亲自实验，最后选择了图标式，分享图标会显示在评论框的上面，除了图标比较丑外，够用。

点[JiaThis™图标式代码](http://www.jiathis.com/getcode/icon/?style=24x24&btn=qzone,tsina,tqq,weixin,renren,jicon&codestyle=standard&showshares=true&renren-data=width%3D100&tsina-data=width%3D120&showujian=true)获取JiaThis分享图标代码，添加到`D:/hexo/themes/next/layout/_partials/share/jiathis.swig`中即可，然后在`D:/hexo/themes/next/_config.yml`中启用

```json
# Share
jiathis: true
add_this_id: 填你自己的JiaThis id
```
