---
title: "搭建ELK日志平台 - 安装Kibana"
source: https://www.boris1993.com/install-elk-part-kibana.html
date: 2018-02-27
updated: 2026-10-10
tags: [ELK, Kibana]
categories: [瞎折腾]
---

# 搭建ELK日志平台 - 安装Kibana

上一次我们安装好了Elastic Search和Logstash，本次我们继续安装Kibana。

# 安装Kibana

Kibana也提供了RPM安装包，所以还是一样的套路：

```bash
sudo rpm -ivh kibana-6.2.2-x86_64.rpm
```

# 配置Kibana及防火墙

编辑`/etc/kibana.yml`

这里比较关键的一点，是要指定Elastic Search的位置。如果Elastic Search是安装在本机，并监听默认的9200端口的话，则不需要修改该配置。

```yaml
# The URL of the Elasticsearch instance to use for all your queries.
#elasticsearch.url: "http://localhost:9200"
```

另外Kibana默认仅能从本机访问，若要开放给局域网，还需要修改Kibana监听的地址和端口号，并配置防火墙允许该端口通信：

```yaml
# Kibana is served by a back end server. This setting specifies the port to use.
#server.port: 5601

# Specifies the address to which the Kibana server will bind. IP addresses and host names are both valid values.
# The default is 'localhost', which usually means remote machines will not be able to connect.
# To allow connections from remote users, set this parameter to a non-loopback address.
# 如果要绑定到特定的某一块网卡，那么就将这里的地址设为那块网卡的IP地址
server.host: "0.0.0.0"
```

Cent OS 7使用firewalld管理防火墙，所以使用如下命令开放Kibana的端口：

```bash
sudo firewall-cmd --zone=public --add-port=5601/tcp --permanent
sudo firewall-cmd --reload
```

# 启动Kibana

我们这里同样使用systemd来管理Kibana的起停和自启动。

```bash
sudo systemctl enable kibana
sudo systemctl start kibana
```

然后即可使用浏览器访问Kibana

# 配置index pattern

Kibana启动后，会要求配置索引，根据提示步骤配置即可。配置过程结束后，可到Discover页检查是否读到数据。

# 参考文档

Kibana User Guide
