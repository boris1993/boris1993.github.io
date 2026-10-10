---
title: "搭建ELK日志平台 - 安装Logstash"
source: https://www.boris1993.com/install-elk-part-logstash.html
date: 2018-02-26
updated: 2026-10-10
tags: [ELK, Logstash]
categories: [瞎折腾]
---

# 搭建ELK日志平台 - 安装Logstash

上回书说道，我们已经安装好了Elastic Search。那么这次，我们继续安装Logstash。

# 安装Logstash

为了安装方便，本次依旧选择使用RPM包安装。

```bash
sudo rpm -ivh logstash-6.2.2.rpm
```

安装结束后，运行Logstash以检查安装是否成功。使用如下命令启动Logstash，并配置输入源为基本输入(stdin)，以及输出到基本输出(stdout)：

```bash
# 因为使用RPM方式安装，导致/usr/share/logstash/data仅root才可写入，所以需要使用sudo环境
sudo /usr/share/logstash/bin/logstash -e 'input{ stdin{} } output{ stdout{} }'
```

在日志滚动停止后，随意输入一些字符串，比如”hello world”，并回车，检查输出：

```plaintext
hello world
2018-02-26T07:18:07.904Z localhost.localdomain hello world
```

可见Logstash成功从stdin读取到了输入，并打印到了stdout，证实安装成功。

# 配置Logstash

## Logstash系统配置

编辑`/etc/logstash/logstash.yml`，修改Logstash系统级配置。

```yaml
# 配置节点名，若未配置则默认取本机主机名作为节点名
node.name: elk-logstash-node-0
```

其他配置项略，如有需要请参考Logstash Reference。

## 日志输入输出配置

这里我们配置让Logstash接收Cent OS的系统日志。

```plaintext
input {
  file {
    path => "/var/log/messages*"
    type => "syslog"
  }
}

# Filter not needed. Commented out.
#filter {
#
#}

output {
  elasticsearch {
    hosts => "localhost:9200"
  }
}
```

另外，本例中还需要配置Logstash以root权限运行以读取系统日志(messages文件默认权限为600)，实际使用时需要按照实际需求配置。

编辑`/etc/systemd/system/logstash.service`，修改user和group为root

```ini
[Unit]
Description=logstash

[Service]
Type=simple
User=root
Group=root
# Load env vars from /etc/default/ and /etc/sysconfig/ if they exist.
# Prefixing the path with '-' makes it try to load, but if the file doesn't
# exist, it continues onward.
EnvironmentFile=-/etc/default/logstash
EnvironmentFile=-/etc/sysconfig/logstash
ExecStart=/usr/share/logstash/bin/logstash "--path.settings" "/etc/logstash"
Restart=always
WorkingDirectory=/
Nice=19
LimitNOFILE=16384

[Install]
WantedBy=multi-user.target
```

然后使systemd重新加载配置文件并重新启动Logstash

```bash
sudo systemctl daemon-reload
sudo systemctl restart logstash
```

## 使Logstash开机自启动

由于RPM包安装时已经放好了自启动的配置文件，我们只需要在systemd中激活它就可以了。

```bash
sudo systemctl enable logstash
sudo systemctl start logstash
```

# 参考文档

Logstash Reference
