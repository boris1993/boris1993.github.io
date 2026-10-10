---
title: "通过Hexo博客完成NIP-05验证"
source: https://www.boris1993.com/nostr-nip-05-with-hexo.html
date: 2023-02-04
updated: 2026-10-10
tags: [nostr, NIP-05, Damus]
categories: [瞎折腾]
---

# 通过Hexo博客完成NIP-05验证

## 让Hexo在生成时包含`.well-known`目录

编辑`config.yml`，添加如下配置：

```yaml
include:
  - ".well-known/**"
```

## 转换公钥格式

Damus生成的`npub`开头的公钥不能直接用，要先转换成十六进制格式的公钥。可以用这个网站来转换。

把`npub`开头的公钥粘贴进`damus key`一栏，`hex key`会自动计算出来，复制`hex key`的内容备用。

## 创建文件

在`source`目录下新建一个名为`.well-known`的目录，然后在`.well-known`目录中创建`nostr.json`，填写如下信息：

```json
{
    "names": {
        "你的用户名": "刚刚转换得到的hex key"
    }
}
```

在本地运行`hexo clean && hexo g`，生成成功后检查`public`目录中是否存在`.well-known/nostr.json`文件，如存在则说明配置正确。

## 配置Nostr

提交代码并部署成功后，即可在`nostr`客户端配置`NIP-05`认证，本文以`Damus`为例。

点击左上角头像，进入`Profile`，点击`Edit`，在`NIP-05 VERIFICATION`一栏中填写`你的username@你的博客网站`，比如我填写的是`boris1993@www.boris1993.com`，最后点击`Save`。

如果一切正常的话，在你的Profile页面就能看到一个对勾图标，代表验证成功。
