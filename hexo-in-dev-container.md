---
title: "在Dev Container里写Hexo博客"
source: https://www.boris1993.com/hexo-in-dev-container.html
date: 2024-12-25
updated: 2026-10-10
tags: [Hexo, Dev Container]
categories: [瞎折腾]
---

# 在Dev Container里写Hexo博客

我们都知道，Docker和容器化技术让运维有了质的飞跃，从此我们不必再担心软件运行所需的繁杂环境，只需要拉取镜像并运行就可以一步到位部署好软件的运行环境。但是在开发过程中，我们仍然需要在本机安装各种依赖，一不小心又会把本机的环境搞乱掉，更不提不同版本的语言之间可能存在的冲突。因此，就出现了Dev Container，顾名思义就是在容器环境中开发，这样我们在开发时也可以享受到统一且隔离的开发环境。

正好我最近也在开始接触Dev Container并使用它给一个开源项目成功提交了代码，正好趁热打铁，把写博客的Hexo环境从本机挪到Dev Container中。

## 配置VSCode

我的计划是用VSCode来在容器中写博客，那么为了让VSCode支持Dev Container，我们需要安装`Dev Containers`这个VSCode扩展。哦对，你还得有个Docker，不论是本机还是在哪个服务器上。我这就用本机的Docker演示了。

装好扩展之后，就可以开始写Dev Container的配置了。因为不同项目对应的开发容器必然是不同的，所以Dev Container的配置会放到项目目录中。通常来说，Dev Container相关的文件都会放到项目根目录下的`.devcontainer`目录中，所以我们可以在项目目录下创建`.devcontainer/devcontainer.json`，并填入如下内容：

```json
{
    // Dev Container的名字
    "name": "blog",
    // 基础Docker镜像
    "image": "node:lts-alpine",
    // 容器创建好之后要执行的命令
    "postCreateCommand": "sh .devcontainer/post_create.sh",
    // 要转发到主机的端口
    "forwardPorts": [4000]
}
```

因为基础镜像`node:lts-alpine`中并不包含我们需要的`git`、`gpg`、`hexo`等环境，所以我们需要通过`postCreateCommand`指定容器在启动之后执行命令来安装这些依赖。`post_create.sh`的内容如下：

```bash
#!/bin/sh

apk add --no-cache gpg git
npm install -g hexo-cli
```

然后用快捷键`ctrl-shift-p`（macOS就是`⌘-shift-p`）唤出命令面板，选择`Dev Containers: Rebuild and Reopen in Container`来构建容器并进入容器开发。如果将来`devcontainer.json`或相关的配置发生变化，我们也可以在命令面板中选择`Dev Containers: Rebuild Container`来重建容器。

## 编写容器的Dockerfile

上面提到的通过`postCreateCommand`来安装依赖的方式，虽然可行，但是不优雅，毕竟每次创建容器都要重新安装一次，就很浪费。所以我们也可以给我们的Dev Container写一个Dockerfile，这样就可以只初始化一次然后一直用下去了。

```dockerfile
FROM node:lts-alpine

LABEL devcontainer.metadata = '[{ \
    "forwardPorts": [4000] \
}]'

RUN apk add --no-cache gpg git \
 && npm install -g hexo-cli
```

与部署用的Dockerfile不同，我们只需要写配置环境相关的命令就可以，不需要把工作空间拷贝进去，因为Dev Container会自动把这个目录mount到容器中。此外，在Dockerfile中还可以在`devcontainer.metadata`中提前指定好一些配置，以减少`devcontainer.json`的行数。

## 配置GPG

上面一步完成之后，Dev Container的配置其实就完成了。但如果你像我一样为Git配置了GPG签名，同时又是在macOS下开发，那么你还需要对GPG做一些额外的配置。首先我们需要安装`pinentry-mac`，然后编辑`~/.gnupg/gpg-agent.conf`文件，添加一行`pinentry-program /usr/local/bin/pinentry-mac`。如果你曾经在JetBrains IDE中点过它的`Configure GPG Agent to support own pinentry`，那么你就要把它添加的那行`pinentry-program`替换成这一条，否则我们在容器中提交更新的时候，GPG会因为找不到`/Users/username/.gnupg/pinentry-ide.sh`而报错。

## 参考文档

-   Developing inside a Container
-   Develop on a remote Docker host
-   使用Dev Container开发 - Claws小花园
