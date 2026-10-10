---
title: "使用Docker配置Redis主从复制"
source: https://www.boris1993.com/redis-replication-in-docker.html
date: 2019-05-23
updated: 2026-10-10
tags: [Redis]
categories: [学知识]
---

# 使用Docker配置Redis主从复制

这篇文章主要介绍如何使用Docker在本机搭建一个带有主从复制功能的Redis环境，内容包括涉及的目录结构、`docker-compose.yml`的编写，以及结果的验证。

## 目录结构

本文将采用如下的目录结构，其中`data`目录将用于存放各个容器的数据，`server`目录存放`docker-compose.yml`以及针对`master`和`slave`节点的配置文件。

```text
.
├── data
└── server
    ├── docker-compose.yml
    ├── redis-master.conf
    ├── redis-slave1.conf
    └── redis-slave2.conf
```

## 配置Redis节点

### 配置`master`节点

编辑`redis-master.conf`，修改下列配置：

```plaintext
bind 127.0.0.1

# 启用保护模式
# 即在没有使用bind指令绑定具体地址时
# 或在没有设定密码时
# Redis将拒绝来自外部的连接
protected-mode yes

# 监听端口
port 6379

# 启动时不打印logo
# 这个不重要，想看logo就打开它
always-show-logo no

# 设定密码认证
requirepass redis

# 禁用KEYS命令
# 一方面 KEYS * 命令可以列出所有的键，会影响数据安全
# 另一方面 KEYS 命令会阻塞数据库，在数据库中存储了大量数据时，该命令会消耗很长时间
# 期间对Redis的访问也会被阻塞，而当锁释放的一瞬间，大量请求涌入Redis，会造成Redis直接崩溃
rename-command KEYS ""

# 此外还应禁止 FLUSHALL 和 FLUSHDB 命令
# 这两个命令会清空数据，并且不会失败
```

### 配置`slave`节点

创建`redis-slave1.conf`，修改下列配置：

```plaintext
bind 127.0.0.1

# 启用保护模式
# 即在没有使用bind指令绑定具体地址时
# 或在没有设定密码时
# Redis将拒绝来自外部的连接
protected-mode yes

# 监听端口
port 6380

# 启动时不打印logo
# 这个不重要，想看logo就打开它
always-show-logo no

# 设定密码认证
requirepass redis

# 禁用KEYS命令
# 一方面 KEYS * 命令可以列出所有的键，会影响数据安全
# 另一方面 KEYS 命令会阻塞数据库，在数据库中存储了大量数据时，该命令会消耗很长时间
# 期间对Redis的访问也会被阻塞，而当锁释放的一瞬间，大量请求涌入Redis，会造成Redis直接崩溃
rename-command KEYS ""

# 此外还应禁止 FLUSHALL 和 FLUSHDB 命令
# 这两个命令会清空数据，并且不会失败

# 配置master节点信息
# 格式：
#slaveof <masterip> <masterport>
# 此处masterip所指定的redis-server-master是运行master节点的容器名
# Docker容器间可以使用容器名代替实际的IP地址来通信
slaveof 127.0.0.1 6379

# 设定连接主节点所使用的密码
masterauth "redis"
```

创建`redis-slave2.conf`，修改监听端口号为`6381`，其余配置与`redis-slave1.conf`相同。

## 配置及启动容器

### 编写`docker-compose.yml`

本例中使用`docker-compose`编排相关容器。要说为什么不用`Kubernetes`，那是因为对于一个示例来说这玩意太重了。~说的一套一套的还不是因为不会用~

```yaml
---

version: '3'

services:
  # 主节点的容器
  redis-server-master:
    image: redis
    container_name: redis-server-master
    restart: always
    # 为了规避Docker中端口映射可能带来的问题
    # 这里选择使用host网络
    network_mode: host
    # 指定时区，保证容器内时间正确
    environment:
      TZ: "Asia/Shanghai"
    volumes:
      # 映射配置文件和数据目录
      - ./redis-master.conf:/usr/local/etc/redis/redis.conf
      - ../data/redis-master:/data
    sysctls:
      # 必要的内核参数
      net.core.somaxconn: '511'
    command: ["redis-server", "/usr/local/etc/redis/redis.conf"]
  # 从节点1的容器
  redis-server-slave-1:
    image: redis
    container_name: redis-server-slave-1
    restart: always
    network_mode: host
    depends_on:
      - redis-server-master
    environment:
      TZ: "Asia/Shanghai"
    volumes:
      - ./redis-slave1.conf:/usr/local/etc/redis/redis.conf
      - ../data/redis-slave-1:/data
    sysctls:
      net.core.somaxconn: '511'
    command: ["redis-server", "/usr/local/etc/redis/redis.conf"]
  # 从节点2的容器
  redis-server-slave-2:
    image: redis
    container_name: redis-server-slave-2
    restart: always
    network_mode: host
    depends_on:
      - redis-server-master
    environment:
      TZ: "Asia/Shanghai"
    volumes:
      - ./redis-slave2.conf:/usr/local/etc/redis/redis.conf
      - ../data/redis-slave-2:/data
    sysctls:
      net.core.somaxconn: '511'
    command: ["redis-server", "/usr/local/etc/redis/redis.conf"]
```

### 启动容器

在`docker-compose.yml`所在位置执行`docker-compose up`即可启动上述三个容器，`docker-compose`会将容器日志打印到终端，在日志中可以看到三个`Redis`服务器在启动过程中的动作，以及从节点加入主节点的信息。

启动成功后，可以在本机使用`redis-cli`连接至主节点。连接成功后，可以使用`info replication`命令检查主从复制的信息。

```plaintext
127.0.0.1:6379> auth redis
OK
127.0.0.1:6379> info replication
# Replication
role:master
connected_slaves:2
slave0:ip=127.0.0.1,port=6381,state=online,offset=14,lag=1
slave1:ip=127.0.0.1,port=6380,state=online,offset=14,lag=1
master_replid:156ad6d206def2c9d423ef95f5311365d454f57d
master_replid2:0000000000000000000000000000000000000000
master_repl_offset:14
second_repl_offset:-1
repl_backlog_active:1
repl_backlog_size:1048576
repl_backlog_first_byte_offset:1
repl_backlog_histlen:14
```

其中第5行的`role:master`指示该节点为主节点，第6行的`connected_slaves:2`说明当前有2个从节点，第7、8行则是两个从节点的信息，包括它们的地址、端口号，和状态。

如果此时查看该项目的目录结构，则可以发现在`data`目录中增加了三个Redis服务器的数据目录。

```text
.
├── data
│   ├── redis-master
│   │   └── dump.rdb
│   ├── redis-slave-1
│   │   └── dump.rdb
│   └── redis-slave-2
│       └── dump.rdb
└── server
    ├── docker-compose.yml
    ├── redis-master.conf
    └── redis-slave.conf
```

### 测试一下

光是启动成功还不够，还需要测试一下从节点是否能同步主节点的数据。

首先连接到主节点，新增一个set：

```plaintext
127.0.0.1:6379> auth redis
OK
127.0.0.1:6379> set foo bar
OK
127.0.0.1:6379> get foo
"bar"
```

好的，在主节点里面成功添加了一条数据。那么接下来连接到`slave-1`，看一下数据有没有同步过去：

```plaintext
127.0.0.1:6380> auth redis
OK
127.0.0.1:6380> get foo
"bar"
127.0.0.1:6380> set foo baz
(error) READONLY You can't write against a read only replica.
```

看来`slave-1`成功的从主节点同步了数据，并且这个节点也按照设定，是一个只读的节点。那么`slave-2`呢？

```plaintext
127.0.0.1:6381> auth redis
OK
127.0.0.1:6381> get foo
"bar"
127.0.0.1:6381> set foo baz
(error) READONLY You can't write against a read only replica.
```

OK，`slave-2`也成功的同步了数据，并且正在作为一个只读节点运行着。

## 参考文档

-   Configuring replication in Docker and NAT - Redis Replication Document

## 系列博文

-   [使用Docker配置Redis主从复制](https://www.boris1993.com/database/Redis/redis-replication-in-docker.html)
-   [使用Docker配置Redis哨兵](https://www.boris1993.com/database/Redis/redis-sentinel-in-docker.html)
