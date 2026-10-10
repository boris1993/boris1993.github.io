---
title: "学习Concourse CI - 使用变量"
source: https://www.boris1993.com/concourse-using-vars.html
date: 2020-03-20
updated: 2026-10-10
tags: [concourse, concourse-ci, vault]
categories: [学知识]
---

# 学习Concourse CI - 使用变量

在实际编写配置文件时，我们不可能保证所有相关的值都是一成不变的。有的值可能会根据运行时的状态来指定，也有可能它是类似token一样的机密信息，必须从一个可信渠道获取。为了应对这种情况，我们就需要在配置文件中引入变量。

## 声明变量

在Concourse的语法规则中，如果要声明一个变量，就用双括号把变量名字包裹起来，就像这样：`((a-variable))`。借用Concourse Tutorials中的一个例子，我们看一下具体在配置文件中，是如何声明一个变量的。

```yaml
---
jobs:
  - name: show-animal-names
    plan:
      - task: show-animal-names
        config:
          platform: linux
          image_resource:
            type: docker-image
            source: {repository: busybox}
          run:
            path: env
            args: []
          params:
            CAT_NAME: ((cat-name))
            DOG_NAME: ((dog-name))
```

这里需要穿插一个内容，就是task的`params`属性。它的作用是把里面指定的键值对作为环境变量传到容器中。

我们可以看到，它声明了两个变量：`cat-name`和`dog-name`，分别作为环境变量`CAT_NAME`和`DOG_NAME`的值。这样，就完成了变量的声明。

## 为变量传值

变量声明好了，那就要传值了。Concourse支持通过命令行参数、通过文件，以及通过密钥管理器(credentials manager)来传入变量的值。

这里有一点需要注意，通过命令行参数或者文件传入的值，是不会根据外部条件变化的，你可以将其理解成fly会先把配置文件做文本替换然后再注册到Concourse。如果要修改变量的值，那么就需要重新运行`fly set-pipeline`命令来修改。

### 通过命令行参数传值

我们可以在`fly`命令中使用`-v`参数来为变量传值，它的格式是`-v key=value`，要对多个变量赋值就要使用多个`-v`参数。

如果要为上文提到的配置文件传值，那么我们可以这样执行命令：

```bash
# 为变量赋值并注册pipeline
fly -t tutorial sp -p parameters -c pipeline.yml -v cat-name=garfield -v dog-name=odie
```

然后我们可以执行`fly -t tutorial trigger-job -j parameters/show-animal-names -w`这条命令来触发任务执行，来检查传值是否成功。

### 通过文件传值

要使用文件传值，我们首先需要创建一个包含变量名和值的YAML文件。为了方便起见我直接用heredoc创建，当然你可以选择任何你喜欢的方式。

```bash
# 从标准输入创建文件
# 当接收到EOF这个字符串时停止接收内容
$ cat > credentials.yml <<EOF
heredoc> cat-name: garfield
heredoc> dog-name: odie
heredoc> EOF
```

这样我们就得到了一个这样的文件：

```yaml
cat-name: garfield
dog-name: odie
```

然后在`fly`命令中，我们就可以用`-l`参数来通过文件为变量赋值：

```bash
fly -t tutorial sp -p parameters -c pipeline.yml -l credentials.yml
```

### 通过密钥管理器传值

如上文所述，通过参数和文件赋值，不仅存在诸多不便，同时还无法保证机密信息的安全性。所以，Concourse也支持与密钥管理器集成。Concourse支持多种密钥管理器，具体的支持列表可以在官网的Credential Management处查询。这里我们使用Vault进行演示。

由于Vault的安装和配置与本文无关，所以这里略过不讲。

#### Concourse的配置

Concourse默认是没有配置密钥管理器的，如果要将Vault与Concourse集成起来，那么需要为web节点配置如下环境变量：

```bash
# Vault的地址，协议可以是HTTP也可以是HTTPS
CONCOURSE_VAULT_URL=https://vault.example.com:8200
# 你也可以为Concourse指定CA证书所在的位置
CONCOURSE_VAULT_CA_CERT=path/to/ca.crt
```

配置完毕后重启web节点，就完成了Concourse的配置。

#### 在Vault中的准备工作

根据The Vault credential manager的描述，Concourse会在Vault中按照以下的规则查找变量的值，并取出其中`value`这个key所对应的内容：

-   `/concourse/TEAM_NAME/PIPELINE_NAME/VAR_NAME`
-   `/concourse/TEAM_NAME/VAR_NAME`

以上文提到的那个pipeline为例子，我使用系统自带的`main`这个team，并将这个pipeline命名为了`parameters`，那么在寻找`cat-name`这个变量时，Concourse就会先后在`/concourse/main/parameters/cat-name`和`/concourse/main/cat-name`中查找。

所以为了满足这个条件，我们需要在Vault中完成以下几件事：

1.  新建一个secret engine，将其命名为`concourse`
2.  新建一个secret，path填写`main/parameters/cat-name`，secret data新增一条key为`value`，值为`garfield`的记录
3.  再新建一个secret，path填写`main/parameters/dog-name`，secret data新增一条key为`value`，值为`odie`的记录

这样，就在Vault中设定好了变量的值。

#### 测试

此时，我们回到Concourse，触发这个task执行，如果配置正确的话，就可以看到这样的输出了：

![Fetch parameter value from Vault](https://blog-static.boris1993.com/concourse-using-vars/concourse-fetch-parameter-value-from-vault.png)

\[^1\]: Concourse CI  
\[^2\]: Concourse - GitHub  
\[^3\]: Concourse Tutorial
