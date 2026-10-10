---
title: "在NexT主题中接入Google AdSense"
source: https://www.boris1993.com/hexo-next-enable-google-adsense.html
date: 2020-04-05
updated: 2026-10-10
tags: [hexo, NexT, adsense]
categories: [瞎折腾]
---

# 在NexT主题中接入Google AdSense

这个博客从建站到现在，也差不多有两三年时间了，一直以来也都是一种“用爱发电”的状况，看到其他博主都开始通过接入广告联盟来挣零花钱，说不心动那是不可能的。所以就研究了一下怎么在这个博客里面接入Google AdSense。

在使用本文介绍的方法前，请检查您的NexT主题版本是否为`v6.0.2`或后续版本，因为本文涉及的功能是在`v6.0.2`引入的。

虽然网上已经有很多类似的博文了，我为什么还要再写一篇呢？主要是因为，很多文章在添加认证代码的时候，都是直接修改`themes/next/layout/_partials/head.swig`或者`themes/next/layout/_custom/google_adsense.ejs`等位置，说白了就是在NexT主题所在的目录动手了，而我并不喜欢这种侵入性强的操作。经过一点简单的搜索，我发现是可以通过数据文件来实现非侵入性的修改的，所以就想讲一下我的做法。~当然借此机会还能水一篇博文何乐而不为 /滑稽~

那么言归正传，一起来看一下我是怎么配置的吧。

## 注册Google AdSense

要接入AdSense，那当然要有一个AdSense的账户。进入Google AdSense主页，点击“开始使用”，跟随页面提示填好各项信息，这部分操作很简单，就不多废话了。

但是这里可能有一个坑，就是我只能接入一个裸域名，不知道是我的问题还是AdSense就是这么要求的。我的博客地址是`https://www.boris1993.com`，但是在页面上输入网站地址之后，AdSense会把`www`这个二级域名去掉，变成裸域名`https://boris1993.com`，所以我只好把裸域名也关联到了我的博客站上。你们在配置的时候，记得注意一下这个地方。如果你知道具体的原因，或者解决方案，那么也请不吝赐教，在留言区告诉我。

## 添加AdSense代码

在注册完成后，AdSense会给你一串代码，并要求你把这串代码放在`<head></head>`标记中。注意这里就是我跟他们不一样的地方了。

同时再次提醒确认你的NexT主题的版本是不是`v6.0.2`或后续版本。如果不是的话，你要么考虑升级，要么可以点❌关闭了。

首先，我们需要在主题配置文件中指定自定义layout文件的位置。当然，这步也是在数据文件中操作的。什么？你还没有用数据文件这个功能？那你赶紧学习并且用起来吧，省的每次更新主题还要担心丢配置。

在`source/_data/next.yml`中增加如下配置：

```yaml
# 配置的原文可以在themes/next/_config.yml的24行找到(针对v7.5.0)
custom_file_path:
  # 因为我们只需要自定义head部分，所以就只配置head这一个属性，其他都保持默认
  head: source/_data/head.swig
```

指定好`head.swig`的位置之后，我们继续在`source/_data`中创建`head.swig`文件，并把AdSense给我们的那串代码放在`head.swig`里面，就像这样：

```html
<script data-ad-client="██████████" async src="https://pagead2.googlesyndication.com/pagead/js/adsbygoogle.js"></script>
```

然后可以重新启动一下本地的Hexo环境，检查`<head></head>`标记中有没有出现刚才添加的代码。检查没有问题之后，就可以部署到生产环境使上面的修改生效了。为了保险起见，在生产环境上也要检查一下`<head></head>`标记中有没有出现刚才添加的代码。确认没有问题之后，就可以回到AdSense页面中点击验证按钮了。

注意，如果AdSense告诉你检查不通过，你一定不要频繁点击再次检查，一定要先确定检查不通过的原因，并在排除问题之后再重新检查，因为每天检查次数是有限制的，如果超过限制，就只能等24小时之后再检查了~（别问我是怎么知道的~。

然后AdSense会开始审核阶段，这期间我们静静等待审核结果的邮件就好了。

~因为在写这篇博文的时候，我也在等AdSense审核，所以这篇文章暂且先告一段落，等有了结果我再更新后面的内容。~

等了这么久，终于收到Google的邮件说我通过了审核，可以开始投放广告了。那么，我们也可以开始继续完成后面的工作了。

## 配置广告模块

进入AdSense管理后台之后，进入侧边栏的`广告 -> 概览`。在这里我们可以配置广告会在什么位置展示了。

因为我并不希望让我的网站满地都是广告，而影响正常内容的展示，而是，只在首页左侧边栏最下面，和在每篇博文结束的位置，各放置一个广告。所以我就需要到`按广告单元`这个页面手动配置两个广告模块。在这个截图里我们可以看到已经有了两个广告模块，那是因为我已经配置好了。

![Configuring ad modules](https://blog-static.boris1993.com/hexo-next-enable-google-adsense/adsense-conf-ad-module.png)

### 添加广告模块

点击`展示广告`按钮来新建一个类型为`展示`的广告。在顶部文本框填入广告模块的名字，比如`side ad`，然后在右边栏选择广告的尺寸，是自适应的，还是固定长宽的。如果是自适应尺寸，那么还需要选择广告的大致形状。配置完毕后，点击`保存`按钮，一个广告模块就配置好了。

![Sidebar ad module](https://blog-static.boris1993.com/hexo-next-enable-google-adsense/adsense-conf-ad-module-sidebar.png)

按照同样的操作，继续添加一个名为`bottom ad`的展示广告。这样，我们就添加好了两个广告模块。

### 配置博客页面

广告模块是创建好了，但是我们还需要配置广告展示的位置。这里我依旧使用数据文件功能，来实现非侵入性的修改。

首先，在`source/_data/`目录新建两个文件，分别命名为`sidebar.swig`和`post-body-end.swig`，分别用来存放侧边栏广告和文章底部广告的代码。

还记得刚才添加完广告模块之后弹出的那一片代码吗？把它们原样默写到这两个文件里就好啦。开玩笑的，谁记得住那一大堆东西啊。回到`按广告单元`这个页面，在刚才添加的广告单元那里，有一个形似`<>`的按钮，点击它，就可以看到这个广告单元的配置代码了。

把`side ad`的代码复制到`sidebar.swig`中，来把这个侧边栏广告真正的放到侧边栏上。比如我的广告代码是这样的：

```html
<!-- side ad -->
<ins class="adsbygoogle"
     style="display:block"
     data-ad-client="████████████████████"
     data-ad-slot="██████████"
     data-ad-format="auto"
     data-full-width-responsive="true"></ins>
<script>
     // 因为直接执行AdSense的代码会出现 No slot size for availableWidth=0 的错误
     // 并且这个广告位会无法正常展示
     // 所以加一个 event listener，在页面加载完毕之后再执行
     // 参考：https://stackoverflow.com/a/37822404/3833858
     window.addEventListener('load', function() {
          (adsbygoogle = window.adsbygoogle || []).push({});
     });
</script>
```

然后把`bottom ad`的代码复制到`post-body-end.swig`，来把这个广告放到文章底部。

```html
<!-- bottom ad -->
<ins class="adsbygoogle"
     style="display:block"
     data-ad-client="████████████████████"
     data-ad-slot="██████████"
     data-ad-format="auto"
     data-full-width-responsive="true"></ins>
<script>
     (adsbygoogle = window.adsbygoogle || []).push({});
</script>
```

接下来继续修改`source/_data/next.yml`中`custom_file_path`的配置：

```yaml
custom_file_path:
  head: source/_data/head.swig
  # 加入下面两行
  sidebar: source/_data/sidebar.swig # 侧边栏广告
  postBodyEnd: source/_data/post-body-end.swig # 博文末尾广告
```

这样，广告就放置好了。接下来，把博客发布上去，就可以开始打广告了。
