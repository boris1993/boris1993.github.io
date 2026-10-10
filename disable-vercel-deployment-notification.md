---
title: "关闭Vercel的部署结果通知"
source: https://www.boris1993.com/disable-vercel-deployment-notification.html
date: 2023-02-02
updated: 2026-10-10
tags: [Vercel]
categories: [小技巧]
---

# 关闭Vercel的部署结果通知

每次Vercel部署之后，它都会在部署的commit下面发个类似这样的留言：

> Successfully deployed to the following URLs:
> 
> ## blog – ./
> 
> * * *
> 
> blog-boris1993.vercel.app
> 
> boris1993.com
> 
> [www.boris1993.com](http://www.boris1993.com/)

而且GitHub还会给我发邮件通知这个留言的内容，但是这个消息说实话没啥用，白白麻烦人而已，后来发现，在项目根目录创建一个名为`vercel.json`的文件，里面写上这样的配置就行：

```json
{
    "github": {
        "silent": true
    }
}
```

这个配置的作用就是让Vercel不再往这个repo的commit下面评论部署状态。提交之后，Vercel就会在这次部署开始遵循`vercel.json`的设定，不会再发送评论，自然也就不会有那封“骚扰邮件”了。
