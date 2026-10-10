---
title: "搭建ELK日志平台 - 安装Elastic Search"
source: https://www.boris1993.com/install-elk-part-es.html
date: 2018-02-04
updated: 2026-10-10
tags: [ELK, Elastic Search]
categories: [瞎折腾]
---

# 搭建ELK日志平台 - 安装Elastic Search

最近搭建了一次ELK日志平台，在此记录一下安装步骤。由于本次模拟的是服务器不能连接互联网的情况，所以全部安装步骤皆使用RPM或tar包的方式安装。本文主要记录安装Elastic Search的过程。

# 安装JRE

首先这套平台是基于Java的，所以Java运行环境当然是不能少。但因为这上面不涉及Java的开发，所以不需要装JDK，装JRE就够了，还能省下一些磁盘空间。我这里选择`JRE8u161`。

我这次选择使用RPM包安装。

```bash
sudo rpm -ivh jre-8u161-linux-x64.rpm
```

安装完毕后，验证安装是否成功：

```bash
# 检验当前用户下是否安装成功
java -version

# 检验sudo环境下是否安装成功
sudo java -version
```

若都输出如下内容则说明安装成功：

```plaintext
java version "1.8.0_161"
Java(TM) SE Runtime Environment (build 1.8.0_161-b12)
Java HotSpot(TM) 64-Bit Server VM (build 25.161-b12, mixed mode)
```

至此Java环境配置完成

# 安装Elastic Search

## 安装过程

### 使用RPM包安装

直接使用rpm命令安装该RPM包

```bash
sudo rpm --install elasticsearch-6.2.2.rpm
```

CentOS 7使用systemd管理开机自启动项，而且安装过程已经配置好针对systemd的启动脚本，使用如下命令激活

```bash
sudo systemctl daemon-reload
sudo systemctl enable elasticsearch.service
```

### 使用tar.gz包安装

首先新建一个名为elk的用户，用于运行ELK平台

```bash
useradd -m elk
```

下载好Elastic Search的安装包，将其复制到/opt并解压，然后试运行

```bash
sudo cp elasticsearch-6.1.3.tar.gz /opt
cd /opt
sudo tar xvzf elasticsearch-6.1.3.tar.gz

# 需要将Elastic Search目录的所有权设为将要运行该软件的用户
# Elastic Search不允许以root用户运行，安全方面亦不建议以root权限运行程序
sudo chown -R elk:elk elasticsearch-6.1.3

cd elasticsearch-6.1.3/bin
./elelasticsearch
```

启动成功后，在另一终端使用curl尝试连接Elastic Search

```bash
curl http://127.0.0.1:9200
```

若有如下返回，则说明Elastic Search启动成功

```json
{
  "name" : "LWmSd17",
  "cluster_name" : "elasticsearch",
  "cluster_uuid" : "lkbXufQpQuiLaE5kzVKAeA",
  "version" : {
    "number" : "6.1.3",
    "build_hash" : "af51318",
    "build_date" : "2018-01-26T18:22:55.523Z",
    "build_snapshot" : false,
    "lucene_version" : "7.1.0",
    "minimum_wire_compatibility_version" : "5.6.0",
    "minimum_index_compatibility_version" : "5.0.0"
  },
  "tagline" : "You Know, for Search"
}
```

## 安装后的配置

### 系统配置文件修改

Elastic Search需要调整文件描述符大于65535、最大线程数大于4096、以及`vm.max_map_count`大于262144。所以修改操作系统配置文件以满足此要求。

在`/etc/security/limits.conf`插入如下内容

```plaintext
* hard nofile 65536
* soft nofile 65536
* hard nproc  4096
* soft nproc  4096
```

在`/etc/sysctl.conf`中插入如下内容

```plaintext
vm.max_map_count=262144
```

然后执行`sysctl -p`，并重新登录，使配置生效。若配置成功，则可见Elastic Search启动过程中相关的警告信息将不再出现。

### Elastic Search配置文件修改

**以下文件位置根据安装方法不同而不同**  
**若使用RPM包方式安装，则文件位于/etc/elasticsearch**  
**若使用tar包方式安装，则文件位于解压出来的目录的conf文件夹中**

-   修改`cluster.name`  
    我们应当将集群名设置成一个能清晰地表明该集群的作用的名字，如`logging-prod`。
    
-   修改`node.name`  
    为每个Elastic Search节点起一个清晰易懂的名字绝不会是一件坏事。  
    节点名字可以是一个自定义的名字，如`prod-data-2`，也可以使用`${HOSTNAME}`来把本机的主机名作为该节点的节点名。
    
-   其他详细配置  
    要想了解更多配置，可以参考Elasticsearch Reference 的**Set up Elasticsearch**部分。
    

### 配置自动启动

如果使用RPM包方式安装，则此步可忽略。

若使用tar包方式安装，则进入Elastic Search的bin目录后运行

```plaintext
./elasticsearch -d -p ../logs/elasticsearch.pid
```

使Elastic Search以daemon模式启动并监控启动过程。

# 参考文献

-   Elastic Search Installation Guide
