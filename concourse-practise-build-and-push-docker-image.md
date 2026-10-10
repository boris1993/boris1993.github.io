---
title: "Concourse实战 - 监控GitHub release并自动构建镜像"
source: https://www.boris1993.com/concourse-practise-build-and-push-docker-image.html
date: 2023-01-29
updated: 2026-10-10
tags: [concourse, concourse-ci, vault]
categories: [学知识]
---

# Concourse实战 - 监控GitHub release并自动构建镜像

距离上一篇Concourse相关的文章发布，已过去两年有余，期间因为没什么使用场景，不知道该怎么继续写下去，于是就断了。这次，我终于有机会将Concourse用到我自己的home lab，并成功完成了一条pipeline。

## 背景及需求

偶然在网上看到了一个可以多端直播推流的工具，叫Ant Media Server，但是它的安装程序并不支持我正在用的`Ubuntu 22.04 LTS`，同时它也没有提供制作好的Docker镜像，只能自己手动构建。可手动构建也太不优雅，根本不能忍，所以萌生了一个需求：监控`Ant Media Server`的GitHub releases，如果有新的版本发布，那么就自动构建新的Docker镜像，并推送到我的Docker Hub中。

## 开始动手

首先，我要实现在Concourse里面监控GitHub release。github-release这个resource type就是干这件事的，所以我们可以在pipeline中定义这样一个resource：

```yaml
resources:
- name: ant-media-server
  type: github-release
  source:
    owner: ant-media
    repository: Ant-Media-Server
    # 默认监视的是"release-"开头的tag
    # 但Ant Media Server的tag都是以"ams-"开头的
    # 所以需要指定一下
    tag_filter: "ams-v?([^v].*)"
```

资源光在`resources`里面定义好还不够，我们需要在pipeline里面用`get`这个task来让Concourse做出从这个资源获取数据的操作。所以，开始写pipeline咯。

```yaml
jobs:
- name: build-image # pipeline的名字
  public: true # 公开就意味着用户不需登录也能在dashboard中看到
  plan:
    - get: ant-media-server # 这里要写上面定义的resource的名字
      trigger: true # 这个资源将作为一个触发器
```

这样就实现了让Concourse监控这个GitHub release，并在发布新release的时候触发pipeline运行。而这个task在运行的时候，会将release中的artifact下载到`ant-media-server`这个目录中，所以我们也不用担心下载文件的问题。同时它还会把release的版本号写在`version`这个文件中，后面我们可以利用这个文件来生成Docker镜像的tag。

有了Ant Media Server的成品文件，按照官方文档的说法，接下来只要做两件事：下载Dockerfile，执行`docker build`命令就行。但是放在pipeline里面，就没这么简单了。

先做第一件事，下载Dockerfile。感谢jgriff/http-resource这个仓库，它可以实现在Concourse里面通过HTTP下载一个文件。那么接下来pipeline里面可以这么写：

```yaml
# 因为这不是Concourse官方提供的resource type
# 所以需要在这里定义一个名为http-resource的resource type
# 并声明由jgriff/http-resource这个Docker镜像来实现
resource_types:
  - name: http-resource
    type: docker-image
    source:
      repository: jgriff/http-resource

resources:
# 前略
- name: ant-media-server-dockerfile
  type: http-resource # 上面定义好这个resource type之后，就可以在这里用了
  source:
    # 指定要下载的文件
    url: https://raw.githubusercontent.com/ant-media/Scripts/master/docker/Dockerfile_Process

jobs:
- name: build-image # pipeline的名字
  public: true # 公开就意味着用户不需登录也能在dashboard中看到
  plan:
  # 让这两个task并行执行，节省时间
  - in_parallel:
    - get: ant-media-server
      trigger: true
    # 下载ant-media-server-dockerfile这个resource指定的文件
    - get: ant-media-server-dockerfile
```

现在Dockerfile可以下载到了，但是它是被保存在`ant-media-server-dockerfile/body`这个文件里面的，我们需要把它移动到`ant-media-server`这个目录里，才能保证后面成功运行`docker build`。所以接下来要用`mv`命令把文件移过去。

```yaml
jobs:
- name: build-image
  public: true
  plan:
  # 前略
  - task: move-dockerfile
    config:
      # task运行在Linux环境
      platform: linux
      # task将通过ubuntu这个Docker镜像运行
      image_resource:
        type: docker-image
        source: 
          repository: ubuntu
      # 将这两个资源传给镜像
      inputs: 
      - name: ant-media-server
      - name: ant-media-server-dockerfile
      # 因为修改了ant-media-server这个资源的内容
      # 所以要将其输出，这样后续的task才能取到修改后的内容
      outputs:
      - name: ant-media-server
      run: 
        path: mv
        args: ["ant-media-server-dockerfile/body", "ant-media-server/Dockerfile"]
```

有了Dockerfile，接下来就可以开始着手构建了。不用想，对于构建Docker镜像这样常见的task，Concourse预先制作好了[`concourse/oci-build-task`](https://github.com/concourse/oci-build-task/)这个镜像来给我们用。

但是首先我们需要创建一个包含着build args的文件，因为文档的`docker build`命令中提到了`--build-arg AntMediaServer=<Replace_With_Ant_Media_Server_Zip_File>`这个参数，而Ant Media Server的zip文件名又会随着release而变化，同时`oci-build-task`的参数`BUILD_ARGS_*`并不支持shell命令，也就是说我不能通过`BUILD_ARGS_AntMediaServer=ant-media-server-community-$(cat version).zip`这样的方法来生成，那么只能用`oci-build-task`的`BUILD_ARGS_FILE`参数，传进去一个生成好的build args file。

所以我们需要在pipeline中增加这两步来完成镜像的构建操作。

```yaml
jobs:
- name: build-image
  public: true
  plan:
  # 前略
  # 生成build args file
  - task: generate-build-args
    config:
      platform: linux
      image_resource:
        type: docker-image
        source: 
          repository: ubuntu
      inputs: 
      - name: ant-media-server
      outputs:
      - name: ant-media-server
      run: 
        # 这里我曾经试过 
        # path: echo
        # args: ["AntMediaServer=ant-media-server-community-$(cat ant-media-server/version).zip", ">", "ant-media-server/build_args.txt"]
        # 但是没成功，因为Concourse会把args做字符串拼接处理，最后当成一整个字符串传给命令
        # 所以其实变成了 echo "AntMediaServer=ant-media-server-community-$(cat ant-media-server/version).zip > ant-media-server/build_args.txt"
        # 很明显这只能把这串字符串打在屏幕上，并不能生成文件
        # 所以只能通过调用sh来执行命令，把命令当成参数传给sh
        path: sh
        args: 
          - -exc
          - 'echo "AntMediaServer=ant-media-server-community-$(cat ant-media-server/version).zip" > ant-media-server/build_args.txt'
  # 开始构建镜像
  - task: build-image
    privileged: true
    config:
      platform: linux
      image_resource:
        type: registry-image
        source: 
          repository: concourse/oci-build-task
      # 构建所需的文件都在ant-media-server这个资源中
      # 所以将它传给这个task
      inputs:
      - name: ant-media-server
      # 将task输出的资源命名为image，并将其输出
      outputs:
      - name: image
      params:
        CONTEXT: ant-media-server
        BUILD_ARGS_FILE: ant-media-server/build_args.txt
      # 缓存构建结果，加速将来的新的构建
      caches:
      - path: cache
      run:
        path: build
```

`oci-build-task`成功后，会把镜像保存到`image/image.tar`文件中。要将它上传到Docker Hub，我们还需要定义一个`registry-image`类型的resource，来指定要将镜像上传到哪里。

因为上传Docker Hub需要登陆，而把token写在pipeline里面是非常蠢的行为，所以我把登陆信息放到了Vault中。向Vault放登陆信息很简单，在`/concourse`这个path中新建两个secret就可以了：

-   `/shared/dockerhub_username`，key是`value`，value填写Docker Hub的用户名
-   `/shared/dockerhub_token`，key是`value`，value填写Docker Hub的access token

之所以我把登陆信息放到`/shared`这个path下，是因为我在Vault中配置了这个path作为一个公共的path，在构建的时候要根据实际情况来修改，比如改成team的名字，或者放在`/{team}/{pipeline}/`下面。具体请参考Concourse与Vault集成相关的文档，这里不再赘述。

放好登陆信息后，就可以添加这样一个resource：

```yaml
resources:
# 前略
- name: ant-media-server-docker
  type: registry-image
  icon: docker
  source:
    repository: "((dockerhub_username))/ant-media-server"
    username: "((dockerhub_username))"
    password: "((dockerhub_token))"
```

然后在pipeline最后增加两个task，一个是读取`ant-media-server/version`的值，将其写在名为`tag`的变量中，后面我们会用这个变量来指定镜像的tag；另一个就是对`ant-media-server-docker`这个资源执行`put`的操作，将`image/image.tar`这个镜像上传到Docker Hub。

```yaml
jobs:
- name: build-image
  public: true
  plan:
  # 前略
  # load_var用来从文件读入数据，并将其放在一个变量中
  - load_var: tag
    # 指定要读的文件
    file: ant-media-server/version
    # 为避免自动识别给我识别错，干脆直接指定文件内容的格式
    # trim就是纯文本，读取之后会去掉头尾的空白
    format: trim
  - put: ant-media-server-docker
    params:
      image: image/image.tar
      # 上传时，同时更新latest和相关semver的镜像
      # 比如上传2.5.3时，会同时更新2.5，2，latest这三个tag
      bump_aliases: true
      # 这里注意要指明从local var source中寻找变量，也就是开头的.:
      # 否则会找不到这个变量
      version: "((.:tag))"
```

## 完整的pipeline

至此这个pipeline就完成了，下面我附上已经部署过的版本，供参考。

```yaml
---
resource_types:
  - name: http-resource
    type: docker-image
    source:
      repository: jgriff/http-resource

resources:
- name: ant-media-server
  type: github-release
  source:
    owner: ant-media
    repository: Ant-Media-Server
    tag_filter: "ams-v?([^v].*)"
- name: ant-media-server-dockerfile
  type: http-resource
  source:
    url: https://raw.githubusercontent.com/ant-media/Scripts/master/docker/Dockerfile_Process
- name: ant-media-server-docker
  type: registry-image
  icon: docker
  source:
    repository: "((dockerhub_username))/ant-media-server"
    username: "((dockerhub_username))"
    password: "((dockerhub_token))"

jobs:
- name: build-image
  public: true
  build_log_retention:
    # 只保留最近5次的构建记录，以节省空间
    builds: 5
  plan:
  - in_parallel:
    - get: ant-media-server
      trigger: true
    - get: ant-media-server-dockerfile
  - load_var: tag
    file: ant-media-server/version
    format: trim
    reveal: true
  - task: move-dockerfile
    config:
      platform: linux
      image_resource:
        type: docker-image
        source: 
          repository: ubuntu
      inputs: 
      - name: ant-media-server
      - name: ant-media-server-dockerfile
      outputs:
      - name: ant-media-server
      run: 
        path: mv
        args: ["ant-media-server-dockerfile/body", "ant-media-server/Dockerfile"]
  - task: generate-build-args
    config:
      platform: linux
      image_resource:
        type: docker-image
        source: 
          repository: ubuntu
      inputs: 
      - name: ant-media-server
      outputs:
      - name: ant-media-server
      run: 
        path: sh
        args: 
          - -exc
          - 'echo "AntMediaServer=ant-media-server-community-$(cat ant-media-server/version).zip" > ant-media-server/build_args.txt'
  - task: build-image
    privileged: true
    config:
      platform: linux
      image_resource:
        type: registry-image
        source: 
          repository: concourse/oci-build-task
      inputs:
      - name: ant-media-server
      outputs:
      - name: image
      params:
        CONTEXT: ant-media-server
        BUILD_ARGS_FILE: ant-media-server/build_args.txt
      caches:
      - path: cache
      run:
        path: build
  - put: ant-media-server-docker
    params:
      image: image/image.tar
      bump_aliases: true
      version: "((.:tag))"
```
