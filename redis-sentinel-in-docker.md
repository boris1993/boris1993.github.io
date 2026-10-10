---
title: "使用Docker配置Redis哨兵"
source: https://www.boris1993.com/redis-sentinel-in-docker.html
date: 2019-06-05
updated: 2026-10-10
tags: [Redis]
categories: [学知识]
---

# 使用Docker配置Redis哨兵

接上文[使用Docker配置Redis主从复制](https://www.boris1993.com/database/Redis/redis-replication-in-docker.html)完成之后，这篇文章主要介绍如何使用Docker在本机搭建Redis的哨兵，内容包括涉及的目录结构、`docker-compose.yml`的编写。

## 目录结构

本文将采用如下的目录结构，其中`data`目录将用于存放各个容器的数据，`server`目录存放`docker-compose.yml`以及针对`master`和`slave`节点的配置文件，`sentinel`目录存放哨兵的配置文件和`docker-compose.yml`。

```text
.
├── data
│   ├── redis-master
│   │   └── dump.rdb
│   ├── redis-slave-1
│   │   └── dump.rdb
│   └── redis-slave-2
│       └── dump.rdb
├── sentinel
│   ├── docker-compose.yml
│   └── redis-sentinel.conf
└── server
    ├── docker-compose.yml
    ├── redis-master.conf
    └── redis-slave.conf
```

## 配置哨兵

### 节点配置示意图

下面的示意图中，将采用Redis官方文档所使用的表示方法，`M`代表一个Master节点，`R`代表一个Replica节点，`S`代表一个Sentinel节点。

从整个集群的稳定性角度考虑，首先Master节点和各个Replica节点不应同时处于同一台服务器上，以避免单台虚拟机或物理机不可用造成整个集群失效。

```text
       +----+
       | M1 |
       | S1 |
       +----+
          |
+----+    |    +----+
| R2 |----+----| R3 |
| S2 |         | S3 |
+----+         +----+

配置最少投票节点为2，即有2个哨兵节点投票选举出新的Master即可完成切换。
```

在本示例中，我将使用如下的节点配置，各个节点运行在不同的Docker容器中，来模拟运行在不同服务器中的效果：

```text
+----+   +----+   +----+
| M1 |   | R1 |   | R2 |
+----+   +----+   +----+
  |        |        |
  +--------+--------+
  |        |        |
+----+   +----+   +----+
| S1 |   | S2 |   | S3 |
+----+   +----+   +----+
```

### 编辑配置文件

编辑`redis-sentinel-1.conf`，修改下列配置：

```plaintext
bind 127.0.0.1

# 哨兵的端口号
# 因为各个哨兵节点会运行在单独的Docker容器中
# 所以无需担心端口重复使用
# 如果需要在单机
port 26379

# 配置哨兵的监控参数
# 格式：sentinel monitor <master-name> <ip> <redis-port> <quorum>
# master-name是为这个被监控的master起的名字
# ip是被监控的master的IP或主机名。因为Docker容器之间可以使用容器名访问，所以这里写master节点的容器名
# redis-port是被监控节点所监听的端口号
# quorom设定了当几个哨兵判定这个节点失效后，才认为这个节点真的失效了
sentinel monitor local-master 127.0.0.1 6379 2

# 连接主节点的密码
# 格式：sentinel auth-pass <master-name> <password>
sentinel auth-pass local-master redis

# master在连续多长时间无法响应PING指令后，就会主观判定节点下线，默认是30秒
# 格式：sentinel down-after-milliseconds <master-name> <milliseconds>
sentinel down-after-milliseconds local-master 30000
```

编辑`redis-sentinel-2.conf`和`redis-sentinel-3.conf`，分别修改监听端口号为`26380`和`26381`，其余部分不变。

## 配置及启动容器

### 编写`docker-compose.yml`

这里继续使用`docker-compose`管理容器。

```yaml
---

version: '3'

services:
  redis-sentinel-1:
    image: redis
    container_name: redis-sentinel-1
    restart: always
    # 为了规避Docker中端口映射可能带来的问题
    # 这里选择使用host网络
    network_mode: host
    volumes:
      - ./redis-sentinel-1.conf:/usr/local/etc/redis/redis-sentinel.conf
    # 指定时区，保证容器内时间正确
    environment:
      TZ: "Asia/Shanghai"
    sysctls:
      net.core.somaxconn: '511'
    command: ["redis-sentinel", "/usr/local/etc/redis/redis-sentinel.conf"]
  redis-sentinel-2:
    image: redis
    container_name: redis-sentinel-2
    restart: always
    network_mode: host
    volumes:
      - ./redis-sentinel-2.conf:/usr/local/etc/redis/redis-sentinel.conf
    environment:
      TZ: "Asia/Shanghai"
    sysctls:
      net.core.somaxconn: '511'
    command: ["redis-sentinel", "/usr/local/etc/redis/redis-sentinel.conf"]
  redis-sentinel-3:
    image: redis
    container_name: redis-sentinel-3
    restart: always
    network_mode: host
    volumes:
      - ./redis-sentinel-3.conf:/usr/local/etc/redis/redis-sentinel.conf
    environment:
      TZ: "Asia/Shanghai"
    sysctls:
      net.core.somaxconn: '511'
    command: ["redis-sentinel", "/usr/local/etc/redis/redis-sentinel.conf"]
```

### 启动容器

这里同样使用`docker-compose up -d`启动容器，启动日志中可以看到哨兵开始监控Master节点，以及哨兵完成互相发现。

```text
redis-sentinel-2    | 1:X 11 Nov 2019 14:33:06.871 # +monitor master local-master 127.0.0.1 6379 quorum 2
redis-sentinel-2    | 1:X 11 Nov 2019 14:33:08.996 * +sentinel sentinel 3dc4e0bff631b994a492d51e99a7ebc48e35a054 127.0.0.1 26381 @ local-master 127.0.0.1 6379
redis-sentinel-3    | 1:X 11 Nov 2019 14:33:06.990 # +monitor master local-master 127.0.0.1 6379 quorum 2
redis-sentinel-3    | 1:X 11 Nov 2019 14:33:07.001 * +slave slave 127.0.0.1:6381 127.0.0.1 6381 @ local-master 127.0.0.1 6379
redis-sentinel-3    | 1:X 11 Nov 2019 14:33:07.010 * +slave slave 127.0.0.1:6380 127.0.0.1 6380 @ local-master 127.0.0.1 6379
redis-sentinel-3    | 1:X 11 Nov 2019 14:33:08.876 * +sentinel sentinel 6f646433feb264b582ffa73b5d6bed6626b97966 127.0.0.1 26380 @ local-master 127.0.0.1 6379
redis-sentinel-3    | 1:X 11 Nov 2019 14:33:08.968 * +sentinel sentinel c3b07d8c4ac3686511e436e71043a615e9b1d420 127.0.0.1 26379 @ local-master 127.0.0.1 6379
redis-sentinel-1    | 1:X 11 Nov 2019 14:33:06.948 # +monitor master local-master 127.0.0.1 6379 quorum 2
redis-sentinel-1    | 1:X 11 Nov 2019 14:33:08.997 * +sentinel sentinel 3dc4e0bff631b994a492d51e99a7ebc48e35a054 127.0.0.1 26381 @ local-master 127.0.0.1 6379
```

然后使用`redis-cli`连接到哨兵节点，连接成功后，可以使用`info sentinel`检查哨兵的信息。

```plaintext
127.0.0.1:26379> info sentinel
# Sentinel
sentinel_masters:1
sentinel_tilt:0
sentinel_running_scripts:0
sentinel_scripts_queue_length:0
sentinel_simulate_failure_flags:0
master0:name=local-master,status=ok,address=127.0.0.1:6379,slaves=2,sentinels=3
```

其中，`sentinel_masters:1`说明这个哨兵在监控一个master，最后一行中写明了`master0`这个节点别名为`local-master`，状态为`OK`，地址是`10.1.0.2:6379`，有2个从节点，并有3个哨兵在监控。

### 测试一下

哨兵光是启动了还是不够的，还需要测试一下当被监控节点下线之后，哨兵是否能作出反应。

我先停掉一个从节点，redis-server-slave-2，等了30秒后，三个哨兵主观认为redis-server-slave-2下线。

```text
redis-sentinel-2    | 1:X 11 Nov 2019 14:37:42.232 # +sdown slave 127.0.0.1:6381 127.0.0.1 6381 @ local-master 127.0.0.1 6379
redis-sentinel-3    | 1:X 11 Nov 2019 14:37:42.290 # +sdown slave 127.0.0.1:6381 127.0.0.1 6381 @ local-master 127.0.0.1 6379
redis-sentinel-1    | 1:X 11 Nov 2019 14:37:42.291 # +sdown slave 127.0.0.1:6381 127.0.0.1 6381 @ local-master 127.0.0.1 6379
```

重新启动redis-server-slave-2后，三个哨兵节点都宣布不再主观认为该节点下线。

```text
redis-sentinel-1    | 1:X 11 Nov 2019 14:40:19.160 * +reboot slave 127.0.0.1:6381 127.0.0.1 6381 @ local-master 127.0.0.1 6379
redis-sentinel-1    | 1:X 11 Nov 2019 14:40:19.243 # -sdown slave 127.0.0.1:6381 127.0.0.1 6381 @ local-master 127.0.0.1 6379
redis-sentinel-2    | 1:X 11 Nov 2019 14:40:19.403 * +reboot slave 127.0.0.1:6381 127.0.0.1 6381 @ local-master 127.0.0.1 6379
redis-sentinel-3    | 1:X 11 Nov 2019 14:40:19.161 * +reboot slave 127.0.0.1:6381 127.0.0.1 6381 @ local-master 127.0.0.1 6379
redis-sentinel-3    | 1:X 11 Nov 2019 14:40:19.242 # -sdown slave 127.0.0.1:6381 127.0.0.1 6381 @ local-master 127.0.0.1 6379
redis-sentinel-2    | 1:X 11 Nov 2019 14:40:19.502 # -sdown slave 127.0.0.1:6381 127.0.0.1 6381 @ local-master 127.0.0.1 6379
```

这次我停掉主节点，并经过30秒后，哨兵输出了一大堆日志，不要紧，我们一边看一边解读：

```text
redis-sentinel-1    | 1:X 11 Nov 2019 14:44:11.639 # +sdown master local-master 127.0.0.1 6379
redis-sentinel-2    | 1:X 11 Nov 2019 14:44:11.695 # +sdown master local-master 127.0.0.1 6379
redis-sentinel-2    | 1:X 11 Nov 2019 14:44:11.752 # +new-epoch 1
redis-sentinel-2    | 1:X 11 Nov 2019 14:44:11.755 # +vote-for-leader 3dc4e0bff631b994a492d51e99a7ebc48e35a054 1
redis-sentinel-2    | 1:X 11 Nov 2019 14:44:11.758 # +odown master local-master 127.0.0.1 6379 #quorum 3/2
redis-sentinel-2    | 1:X 11 Nov 2019 14:44:11.759 # Next failover delay: I will not start a failover before Mon Nov 11 14:50:11 2019
redis-sentinel-3    | 1:X 11 Nov 2019 14:44:11.661 # +sdown master local-master 127.0.0.1 6379
redis-sentinel-3    | 1:X 11 Nov 2019 14:44:11.746 # +odown master local-master 127.0.0.1 6379 #quorum 2/2
redis-sentinel-3    | 1:X 11 Nov 2019 14:44:11.746 # +new-epoch 1
redis-sentinel-3    | 1:X 11 Nov 2019 14:44:11.747 # +try-failover master local-master 127.0.0.1 6379
redis-sentinel-3    | 1:X 11 Nov 2019 14:44:11.749 # +vote-for-leader 3dc4e0bff631b994a492d51e99a7ebc48e35a054 1
redis-sentinel-3    | 1:X 11 Nov 2019 14:44:11.755 # c3b07d8c4ac3686511e436e71043a615e9b1d420 voted for 3dc4e0bff631b994a492d51e99a7ebc48e35a054 1
redis-sentinel-3    | 1:X 11 Nov 2019 14:44:11.756 # 6f646433feb264b582ffa73b5d6bed6626b97966 voted for 3dc4e0bff631b994a492d51e99a7ebc48e35a054 1
redis-sentinel-1    | 1:X 11 Nov 2019 14:44:11.753 # +new-epoch 1
redis-sentinel-1    | 1:X 11 Nov 2019 14:44:11.754 # +vote-for-leader 3dc4e0bff631b994a492d51e99a7ebc48e35a054 1
redis-sentinel-3    | 1:X 11 Nov 2019 14:44:11.826 # +elected-leader master local-master 127.0.0.1 6379
redis-sentinel-3    | 1:X 11 Nov 2019 14:44:11.832 # +failover-state-select-slave master local-master 127.0.0.1 6379
redis-sentinel-3    | 1:X 11 Nov 2019 14:44:11.894 # +selected-slave slave 127.0.0.1:6380 127.0.0.1 6380 @ local-master 127.0.0.1 6379
redis-sentinel-3    | 1:X 11 Nov 2019 14:44:11.895 * +failover-state-send-slaveof-noone slave 127.0.0.1:6380 127.0.0.1 6380 @ local-master 127.0.0.1 6379
redis-sentinel-3    | 1:X 11 Nov 2019 14:44:11.971 * +failover-state-wait-promotion slave 127.0.0.1:6380 127.0.0.1 6380 @ local-master 127.0.0.1 6379
redis-sentinel-1    | 1:X 11 Nov 2019 14:44:12.436 # +config-update-from sentinel 3dc4e0bff631b994a492d51e99a7ebc48e35a054 127.0.0.1 26381 @ local-master 127.0.0.1 6379
redis-sentinel-1    | 1:X 11 Nov 2019 14:44:12.436 # +switch-master local-master 127.0.0.1 6379 127.0.0.1 6380
redis-sentinel-1    | 1:X 11 Nov 2019 14:44:12.437 * +slave slave 127.0.0.1:6381 127.0.0.1 6381 @ local-master 127.0.0.1 6380
redis-sentinel-1    | 1:X 11 Nov 2019 14:44:12.439 * +slave slave 127.0.0.1:6379 127.0.0.1 6379 @ local-master 127.0.0.1 6380
redis-sentinel-2    | 1:X 11 Nov 2019 14:44:12.434 # +config-update-from sentinel 3dc4e0bff631b994a492d51e99a7ebc48e35a054 127.0.0.1 26381 @ local-master 127.0.0.1 6379
redis-sentinel-2    | 1:X 11 Nov 2019 14:44:12.435 # +switch-master local-master 127.0.0.1 6379 127.0.0.1 6380
redis-sentinel-2    | 1:X 11 Nov 2019 14:44:12.435 * +slave slave 127.0.0.1:6381 127.0.0.1 6381 @ local-master 127.0.0.1 6380
redis-sentinel-2    | 1:X 11 Nov 2019 14:44:12.437 * +slave slave 127.0.0.1:6379 127.0.0.1 6379 @ local-master 127.0.0.1 6380
redis-sentinel-3    | 1:X 11 Nov 2019 14:44:12.372 # +promoted-slave slave 127.0.0.1:6380 127.0.0.1 6380 @ local-master 127.0.0.1 6379
redis-sentinel-3    | 1:X 11 Nov 2019 14:44:12.373 # +failover-state-reconf-slaves master local-master 127.0.0.1 6379
redis-sentinel-3    | 1:X 11 Nov 2019 14:44:12.433 * +slave-reconf-sent slave 127.0.0.1:6381 127.0.0.1 6381 @ local-master 127.0.0.1 6379
redis-sentinel-3    | 1:X 11 Nov 2019 14:44:12.753 * +slave-reconf-inprog slave 127.0.0.1:6381 127.0.0.1 6381 @ local-master 127.0.0.1 6379
redis-sentinel-3    | 1:X 11 Nov 2019 14:44:12.920 # -odown master local-master 127.0.0.1 6379
redis-sentinel-3    | 1:X 11 Nov 2019 14:44:13.825 * +slave-reconf-done slave 127.0.0.1:6381 127.0.0.1 6381 @ local-master 127.0.0.1 6379
redis-sentinel-3    | 1:X 11 Nov 2019 14:44:13.883 # +failover-end master local-master 127.0.0.1 6379
redis-sentinel-3    | 1:X 11 Nov 2019 14:44:13.883 # +switch-master local-master 127.0.0.1 6379 127.0.0.1 6380
redis-sentinel-3    | 1:X 11 Nov 2019 14:44:13.884 * +slave slave 127.0.0.1:6381 127.0.0.1 6381 @ local-master 127.0.0.1 6380
redis-sentinel-3    | 1:X 11 Nov 2019 14:44:13.885 * +slave slave 127.0.0.1:6379 127.0.0.1 6379 @ local-master 127.0.0.1 6380
redis-sentinel-2    | 1:X 11 Nov 2019 14:44:42.446 # +sdown slave 127.0.0.1:6379 127.0.0.1 6379 @ local-master 127.0.0.1 6380
redis-sentinel-1    | 1:X 11 Nov 2019 14:44:42.465 # +sdown slave 127.0.0.1:6379 127.0.0.1 6379 @ local-master 127.0.0.1 6380
redis-sentinel-3    | 1:X 11 Nov 2019 14:44:43.887 # +sdown slave 127.0.0.1:6379 127.0.0.1 6379 @ local-master 127.0.0.1 6380
```

首先，三台哨兵都宣布Master节点主观下线。

因为在配置文件中，我们指定了当最少2个哨兵认为Master节点失效后就会开始选举(就是quorom)，所以哨兵2提出选举新的Master节点。

接下来，哨兵将开始投票，从Slave节点中选举出新的Master节点。在达成一致后，被选举的Slave节点将成为新的Master节点，其配置文件将会被改写，来让这个变动永久生效。

然后，哨兵会通知这个集群的其他节点来加入新的Master，包括挂掉的那个之前的Master。

这样就完成了一次failover切换。

此时，如果重启之前的Master节点，哨兵会发现节点上线，并不再主观认为该节点下线。但是，现在这个节点已经变成了一个Slave节点。

```text
redis-sentinel-1    | 1:X 11 Nov 2019 14:56:32.936 # -sdown slave 127.0.0.1:6379 127.0.0.1 6379 @ local-master 127.0.0.1 6380
redis-sentinel-2    | 1:X 11 Nov 2019 14:56:33.202 # -sdown slave 127.0.0.1:6379 127.0.0.1 6379 @ local-master 127.0.0.1 6380
redis-sentinel-3    | 1:X 11 Nov 2019 14:56:33.707 # -sdown slave 127.0.0.1:6379 127.0.0.1 6379 @ local-master 127.0.0.1 6380
```

## 参考文档

-   Sentinel, Docker, NAT, and possible issues - Redis Sentinel Documentation

## 系列博文

-   [使用Docker配置Redis主从复制](https://www.boris1993.com/database/Redis/redis-replication-in-docker.html)
-   [使用Docker配置Redis哨兵](https://www.boris1993.com/database/Redis/redis-sentinel-in-docker.html)
