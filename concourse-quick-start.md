---
title: "学习Concourse CI - 快速入门"
source: https://www.boris1993.com/concourse-quick-start.html
date: 2020-03-10
updated: 2026-10-10
tags: [concourse, concourse-ci]
categories: [学知识]
---

# 学习Concourse CI - 快速入门

最近公司需要用到一个名叫`Concourse CI`的`CI/CD`工具，那么我当然就要学习一下啦。顺便还能水一篇，啊不，写一篇博客，当作学习过程中的笔记。

## 准备数据库

Concourse使用`PostgreSQL`数据库来存储数据，所以首先要初始化好一个数据库。

如果要使用自建的数据库，那么可以参考这篇官方文档。

我这里用的是[`Railway`](https://railway.app/?referralCode=WQBcCO)的数据库实例，准备步骤如下：

```sql
-- 给Concourse创建个单独的schema
CREATE SCHEMA concourse;
-- 创建用户
CREATE ROLE concourse WITH ENCRYPTED PASSWORD 'concourse';
-- 使新用户可以登陆
ALTER ROLE concourse WITH LOGIN;
-- 给用户concourse赋予schema级的权限
GRANT USAGE,CREATE ON SCHEMA concourse TO concourse;
-- 给用户concourse赋予所有表的全部权限
GRANT ALL ON ALL TABLES IN SCHEMA concourse TO concourse;
-- 给用户concourse赋予所有序列的全部权限
GRANT ALL ON ALL SEQUENCES IN SCHEMA concourse TO concourse;
```

## 安装Concourse CI

这里我将用两台服务器完成Concourse的部署，一个用来部署`web`节点，一个用来部署`worker`节点。

### Web节点

Concourse的web节点中会运行一个名为`TSA`的服务用来注册`worker`节点，所以首先我们要在`web`节点创建`TSA`服务所需的SSH密钥对。

```bash
cd ~/docker/concourse
# 用于让web节点生成和验证用户的session token
ssh-keygen -t rsa -b 4096 -m PEM -f ./session_signing_key
# 生成TSA端的密钥对
ssh-keygen -t rsa -b 4096 -m PEM -f ./tsa_host_key
# 稍后要将worker节点的公钥放在这里面
# 其实就是SSH的authorized_keys
touch authorized_worker_keys
```

然后编写`docker-compose.yml`：

```yaml
version: '3'

services:
  concourse:
    image: concourse/concourse:latest
    restart: always
    container_name: concourse
    # 因为Concourse要开好几个端口，我懒得一个个配，直接host网络拉倒
    network_mode: host
    privileged: true
    # 让Concourse启动web节点
    command: web
    # 把刚刚创建的密钥挂载进容器
    volumes:
      - /home/boris1993/docker/concourse:/keys
    environment:
      TZ: Asia/Shanghai
      # HTTP代理配置，按需
      # 为啥要配懂得都懂，就是网络加速，如果你的网络能顺畅拉资源那不配也没问题
      HTTP_PROXY: http://127.0.0.1:8899
      HTTPS_PROXY: http://127.0.0.1:8899
      ALL_PROXY: socks5://127.0.0.1:8899
      # Web节点监听8085端口
      CONCOURSE_BIND_PORT: 8085
      # 外部访问地址，因为我只在内网用，所以就配个内网IP就行
      CONCOURSE_EXTERNAL_URL: http://192.168.1.123:8085
      # 密钥配置
      CONCOURSE_SESSION_SIGNING_KEY: /keys/session_signing_key
      CONCOURSE_TSA_HOST_KEY: /keys/tsa_host_key
      CONCOURSE_TSA_AUTHORIZED_KEYS: /keys/authorized_worker_keys
      # 数据库配置
      CONCOURSE_POSTGRES_HOST: containers-us-east-123.railway.app
      CONCOURSE_POSTGRES_USER: concourse
      CONCOURSE_POSTGRES_PORT: 5511
      CONCOURSE_POSTGRES_PASSWORD: concourse
      CONCOURSE_POSTGRES_DATABASE: railway
      # 配置一个本地用户用于首次登陆
      CONCOURSE_ADD_LOCAL_USER: concourse:concourse
      # 将这个本地用户加入main team，即将其作为管理员
      CONCOURSE_MAIN_TEAM_LOCAL_USER: concourse
```

接下来执行`docker compose up -d`启动容器，过几分钟就可以在`http://192.168.1.123:8085`打开Concourse的页面了。首次启动可能耗时比较久，因为要花时间初始化数据库里面的各种表。

### Worker节点

上面启动的`web`节点只是用来给我们看的，它并不能执行任何的构建任务，所以还需要启动至少一个`worker`节点来运行构建任务。

首先还是生成密钥：

```bash
cd ~/docker/concourse-worker
# 只需要生成worker的SSH密钥
ssh-keygen -t rsa -b 4096 -m PEM -f ./worker_key
```

生成了worker节点的SSH密钥对之后，我们需要把`worker_key.pub`中的内容添加到web节点的`authorized_worker_keys`文件中，以通知web节点可以接受这个worker的加入请求。`authorized_worker_keys`文件改好后需要重启web节点的Docker容器以使修改生效。

接下来编写`docker-compose.yml`：

```yaml
version: '3'

services:
  concourse-worker:
    image: concourse/concourse:latest
    restart: always
    container_name: concourse_worker
    network_mode: host
    privileged: true
    # 让Concourse以worker模式运行
    command: worker
    volumes:
      # 密钥所在的位置
      - /home/ubuntu/docker/concourse:/keys
      # worker节点的数据目录
      - /home/ubuntu/docker/concourse/data:/opt/concourse/
    environment:
      # 节点名字
      CONCOURSE_NAME: 'worker-1'
      # 在Docker中运行的话，必须手动指定运行环境是containerd
      CONCOURSE_RUNTIME: containerd
      CONCOURSE_CONTAINERD_DNS_SERVER: 8.8.8.8
      # web节点TSA服务的位置
      CONCOURSE_TSA_HOST: 192.168.1.123:2222
      # worker节点的密钥
      CONCOURSE_TSA_PUBLIC_KEY: /keys/tsa_host_key.pub
      CONCOURSE_TSA_WORKER_PRIVATE_KEY: /keys/worker_key
      CONCOURSE_WORK_DIR: /opt/concourse/worker
```

然后执行`docker compose up -d`启动即可。

## 安装Fly CLI

虽然Concourse带有一个Web界面，但是我们在Web界面里面干不了什么，因为它的所有管理操作都需要通过它的`Fly CLI`来完成。

要安装`Fly CLI`，你可以从刚才打开的Dashboard里面下载，也可以到Concourse的GitHub Releases中下载。

macOS用户可能会想，我能不能用`Homebrew`来安装这个东西？一开始我也是这么想的，但是后面我发现，fly的版本是要跟着web节点的版本走的，所以死了这条心，老老实实从Dashboard里面下载吧。

## 检查worker的状态

为了确保worker节点是成功连接到web节点，我们需要用`fly`命令来检查worker节点的状态。

```bash
# 添加一个名为default的target，登陆至http://192.168.1.123:8085
# 需要点击下面显示的URL，在浏览器中完成登陆过程
$ fly login -t default -c http://192.168.1.123:8085
logging in to team 'main'

navigate to the following URL in your browser:

  http://192.168.1.123:8085/login?fly_port=49290

or enter token manually (input hidden):
target saved

# 列出这个target的workers
# 看到刚刚启动的worker节点，即说明这个worker成功连上了
$ fly -t default workers
name          containers  platform  tags  team  state    version  age
worker-1      0           linux     none  none  running  2.4      14h14m
```

## Hello World

世间万物都可以从一个hello world学起，Concourse也不例外。我们可以跟着Concourse Tutorial\[^3\]中`Hello World`一节的描述，把这个task执行起来。

```bash
$ git clone https://github.com/starkandwayne/concourse-tutorial.git
Cloning into 'concourse-tutorial'...
remote: Enumerating objects: 5, done.
remote: Counting objects: 100% (5/5), done.
remote: Compressing objects: 100% (5/5), done.
remote: Total 3794 (delta 0), reused 4 (delta 0), pack-reused 3789
Receiving objects: 100% (3794/3794), 11.18 MiB | 25.00 KiB/s, done.
Resolving deltas: 100% (2270/2270), done.
$ cd concourse-tutorial/tutorials/basic/task-hello-world
$ fly -t default execute -c task_hello_world.yml
uploading task-hello-world done
executing build 1 at http://localhost:8080/builds/1
initializing
waiting for docker to come up...
Pulling busybox@sha256:afe605d272837ce1732f390966166c2afff5391208ddd57de10942748694049d...
sha256:afe605d272837ce1732f390966166c2afff5391208ddd57de10942748694049d: Pulling from library/busybox
0669b0daf1fb: Pulling fs layer
0669b0daf1fb: Verifying Checksum
0669b0daf1fb: Download complete
0669b0daf1fb: Pull complete
Digest: sha256:afe605d272837ce1732f390966166c2afff5391208ddd57de10942748694049d
Status: Downloaded newer image for busybox@sha256:afe605d272837ce1732f390966166c2afff5391208ddd57de10942748694049d

Successfully pulled busybox@sha256:afe605d272837ce1732f390966166c2afff5391208ddd57de10942748694049d.

running echo hello world
hello world
succeeded
```

可以看到，Concourse收到这个task之后，下载了一个Busybox的Docker镜像，然后执行了`echo hello world`这条命令。那么，Concourse是怎么知道要如何执行一个task呢？这就得从上面运行的`task_hello_world.yml`说起了。

## 一个task的配置文件

Task是Concourse的流水线(pipeline)中最小的配置单元，我们可以把它理解成一个函数，在我们配置好它的行为之后，它将永远按照这个固定的逻辑进行操作。

上面的`task_hello_world.yml`就是配置了一个task所要进行的操作，它的内容不多，我们一块一块拆开来看。

```yaml
---
platform: linux

image_resource:
  type: docker-image
  source: {repository: busybox}

run:
  path: echo
  args: [hello world]
```

`platform`属性指定了这个task要运行在哪种环境下。需要注意，这里指的是worker运行的环境，比如这里指定的`linux`，就意味着Concourse将会挑选一个运行在Linux中的worker。

`image_resource`属性指定了这个task将会运行在一个镜像容器中。其中的`type`属性说明这个镜像是一个Docker镜像，`source`中`{repository: busybox}`说明了要使用Docker仓库中的`busybox`作为基础镜像。

`run`属性就是这个task实际要执行的任务，其中的`path`指定了要运行的命令，这里可以是指向命令的绝对路径、相对路径，如果命令在`$PATH`中，那么也可以直接写命令的名称；`args`就是要传递给这个命令的参数。

如果要执行的命令非常复杂，我们也可以把命令写在一个shell脚本中，然后在`run.path`中指向这个脚本，比如这样：

```yaml
run:
  path: ./hello-world.sh
```

这样一来，就很清楚了。这个task会在一台Linux宿主机中执行，它将在一个busybox镜像中运行`echo hello world`这条命令。

## 把多个task串起来

虽然我们在上面已经有了一个能用的task，但是上面说了，task只是一个pipeline的最小组成部分。而且在正式环境中，一个CI/CD任务可能会用到多个task来完成完整的构建任务。那么，怎么把多个task串起来呢？手动去做这件事显然不现实，所以就有了pipeline。

这里我们还是用Concourse Tutorial\[^3\]中的示例来演示。

首先我们先看一下这个配置文件的内容：

```yaml
---
jobs:
  - name: job-hello-world
    public: true
    plan:
      - task: hello-world
        config:
          platform: linux
          image_resource:
            type: docker-image
            source: {repository: busybox}
          run:
            path: echo
            args: [hello world]
```

一个pipeline可以有多个job，这些job决定了这个pipeline将会以怎样的形式来执行。而一个job中最重要的配置，是plan，即需要执行的步骤。一个plan中的作业步，可以用来获取或更新某个资源，也可以用来执行某一个task。

上面这个pipeline只有一个名为`job-hello-world`的job，这个job里面只有一个作业步，名为`hello-world`，是一个task，操作是在一个busybox镜像中执行`echo hello world`命令。

在使用这个pipeline之前，我们需要把它注册到Concourse中。

```bash
# -t 指明要操作的target
# -c 指明pipeline的配置文件
# -p 指明pipeline的名字
$ fly -t default set-pipeline -c pipeline.yml -p hello-world
jobs:
  job job-hello-world has been added:
+ name: job-hello-world
+ plan:
+ - config:
+     container_limits: {}
+     image_resource:
+       source:
+         repository: busybox
+       type: docker-image
+     platform: linux
+     run:
+       args:
+       - hello world
+       path: echo
+   task: hello-world
+ public: true

apply configuration? [yN]: y
pipeline created!
you can view your pipeline here: http://localhost:8080/teams/main/pipelines/hello-world

the pipeline is currently paused. to unpause, either:
  - run the unpause-pipeline command:
    fly -t default unpause-pipeline -p hello-world
  - click play next to the pipeline in the web ui
```

现在一个新的pipeline就被注册到Concourse中了。在它的Web UI中也能看到这个pipeline。

![Pipeline](https://blog-static.boris1993.com/concourse-quick-start/concourse-with-pipeline.png)

但是，这个pipeline现在还是暂停状态的，需要把它恢复之后才能使用。那么怎么恢复呢？其实上面`set-pipeline`操作的输出已经告诉我们了。

> the pipeline is currently paused. to unpause, either:  
> \- run the unpause-pipeline command:  
> `fly -t default unpause-pipeline -p hello-world`  
> \- click play next to the pipeline in the web ui
> 
> 这个pipeline目前是被暂停的，如果要恢复，可以使用下面两种方法之一：  
> \- 运行unpause-pipeline命令：  
> `fly -t default unpause-pipeline -p hello-world`  
> \- 在Web UI中点击pipeline的播放按钮

在成功恢复pipeline之后，我们可以看到原来蓝色的paused字样变成了灰色的pending字样，说明现在这个pipeline正在等待任务。

接下来我们就可以手动执行一下这个pipeline，来检查它是否正常。具体操作说起来太啰嗦，我直接借用Concourse Tutorial里面的一个动图来替我说明。

![Manually start a pipeline](https://blog-static.boris1993.com/concourse-quick-start/concourse-manually-run-pipeline.gif)

## 自动触发job

虽然我们在Web UI上点一下加号就能触发job开始执行，但是CI/CD讲究的就是一个自动化，每次更新都手动去点一下，显然谁都受不了这么折腾。所以，Concourse也提供了几种自动触发job执行的方法。

一种方法是向Concourse API发送一个`POST`请求。这种就是webhook，没什么特殊的，在版本控制系统里面配置好webhook的参数就好了。

另一种方法是让Concourse监视某一个资源，在资源发生改变之后自动触发job执行。下面我详细说说这个功能。

这里我们假设一个场景：我们有一个Git仓库，里面有一个名为`test.txt`的文件。我们想在每次这个仓库收到新commit之后，打印出`test.txt`的内容。

按照这个思路，我在Concourse中注册了如下的pipeline：

```yaml
---
# 先定义一个Git资源
resources:
  - name: resource-git-test
    type: git
    source:
      # 这里换成你自己的一个Git仓库
      uri: https://gitee.com/boris1993/git-test.git
      branch: master
  # 如果想要这个任务定期执行，那么可以在这里定义一个计时器
  - name: timer
    type: time
    source:
      # 这里定义这个计时器
      interval: 2m

jobs:
  # 定义一个job
  - name: job-show-file-content
    public: true
    plan:
      # 第一步：获取resource-git-test中定义的资源
      - get: resource-git-test
        # 在资源发生更新的时候触发
        trigger: true
      # 如果要让任务定期重复执行，那么这里也要将定时器作为一个资源
      # 并打开trigger开关
      - get: timer
        trigger: true
      # 第二步：在控制台打印文件内容
      - task: show-file-content
        config:
          platform: linux
          inputs:
            # resource-git-test中定义的资源将作为这个步骤的输入资源
            # 即让resource-git-test中的文件对该步骤可见
            - name: resource-git-test
          image_resource:
            type: docker-image
            # 因为我们都懂的原因，Docker中心仓库有可能会连不上
            # 而在执行构建的时候，Concourse会到仓库检查镜像的版本
            # 所以这里用registry_mirror配置了一个Docker仓库的镜像站
            source: {repository: busybox, registry_mirror: https://dockerhub.azk8s.cn}
          run:
            path: cat
            # 在引入input资源后，工作目录下就可以看到这个资源相关的文件夹
            args: ["./resource-git-test/test.txt"]
```

创建`git-test`仓库、编辑`test.txt`等等操作不是重点，也没啥难度，这里不啰嗦了。在完成编辑文件，和push到远程仓库后，我们等待Concourse检查远程仓库更新，并执行构建步骤。

在pipeline视图中点击`resource-git-test`这个资源，就可以看到这个资源的检查历史，展开某条记录后，还可以看到这条历史相关的构建。

![Git resource trigger](https://blog-static.boris1993.com/concourse-quick-start/concourse-git-resource-trigger.png)

在Concourse检查到git仓库的更新后，就会执行下面指定的构建步骤。结果大概会是这个样子的：

![Git resource trigger execution result](https://blog-static.boris1993.com/concourse-quick-start/concourse-git-resource-trigger-result.png)

## 结束语

至此，我们完整的配置了一个简单的pipeline。后面我会根据文档，或者根据工作中遇到的情况，继续补充权限管理、复杂的case等相关的博文。

\[^1\]: Concourse CI  
\[^2\]: Concourse - GitHub  
\[^3\]: Concourse Tutorial
