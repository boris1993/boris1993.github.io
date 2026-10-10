---
title: "为Maven配置阿里云镜像和代理服务器"
source: https://www.boris1993.com/maven-configure-mirrors.html
date: 2017-12-11
updated: 2026-10-10
tags: [Maven]
categories: [小技巧]
---

# 为Maven配置阿里云镜像和代理服务器

Maven中央仓库在国内的速度简直是感人，好在阿里云提供了Maven中央仓库的镜像，配置方法在此记录备用。

打开Maven的用户配置文件(默认位置在 `~/.m2/settings.xml`)，在`mirrrors`段加入如下内容：

```xml
<mirror>
    <!-- 镜像ID，自行定义 -->
    <id>nexus-aliyun</id>
    <!-- 该镜像对应的仓库名，central即中央仓库 -->
    <!-- 个人建议不要将其设为星号 [注] -->
    <mirrorOf>central</mirrorOf>
    <!-- 镜像名，自行定义 -->
    <name>Nexus aliyun</name>
    <!-- 镜像的地址 -->
    <url>http://maven.aliyun.com/nexus/content/groups/public</url>
</mirror>
```

**\[注\]** 有些教程在 `mirrorOf` 字段中填写的是星号，但根据Using Mirrors for Repositories中 `Using A Single Repository` 一段的解释，这将会强制使用该镜像处理所有的仓库请求，而阿里云镜像并不能达到这样的效果，所以个人建议仅使用该镜像代理中央仓库的请求。
