---
title: "自建Tailscale的DERP节点"
source: https://www.boris1993.com/self-hosting-tailscale-derp.html
date: 2023-02-08
updated: 2026-10-10
tags: [Tailscale, DERP, 内网穿透]
categories: [瞎折腾]
---

# 自建Tailscale的DERP节点

迫于Tailscale官方的中继节点在国内几乎没法用，于是在轻量云香港区自建了一个DERP服务。

## 安装derper

因为derper是依赖`go install`实现分发，所以要先安装Go语言SDK。

```bash
$ wget https://go.dev/dl/go1.20.linux-amd64.tar.gz
$ sudo rm -rf /usr/local/go && sudo tar -C /usr/local -xzf go1.20.linux-amd64.tar.gz
```

安装成功后，需要配置环境变量，把Go的可执行文件，和`~/go/bin`目录加入`PATH`。在`.bashrc`（如果你用`zsh`，那就是`.zshrc`）中添加如下命令：

```bash
export PATH=$PATH:/usr/local/go/bin:~/go/bin
```

重新载入`.bashrc`后，执行`go install tailscale.com/cmd/derper@main`，安装`derper`。

此外，因为derper不能自动更新，所以需要设定一个定时任务来定期重新运行`go install`命令，如：

```plaintext
0 0 * * * go install tailscale.com/cmd/derper@main
```

## 测试derper

在运行derper之前，你需要给这个服务器绑定一个域名，derper会用这个域名来申请Let’s Encrypt的SSL证书。申请和购买域名我就不在这废话了，网上有很多相关文章。

此外，因为derper需要监听443端口，而我又不想以root身份运行它，所以我选择给它分配`CAP_NET_BIND_SERVICE`这个capability，来让它有监听低位端口的权限。

```bash
sudo setcap CAP_NET_BIND_SERVICE=+eip ~/go/bin/derper
```

然后就可以手动运行`derper`来测试了。

```bash
# derper.conf它会自动创建，你给它指定一个位置就行
$ go/bin/derper -c derper/derper.conf -hostname derp.mydomain.com
2023/02/08 13:35:11 derper: serving on :443 with TLS
2023/02/08 13:35:11 running STUN server on [::]:3478
```

这时候derper就会开始监听443端口。

如果443端口被占用，那么就需要手动指定监听的端口（比如4443），SSL证书也要自己通过`acme.sh`之类的工具生成，而且证书的名字必须符合`域名.crt`（即`derp.mydomain.com.crt`）这种格式。

```bash
derper -c=derper/derper.conf -hostname derp.mydomain.com -a :4443 -certmode manual -certdir /home/admin/certs
```

此时用浏览器访问`https://derp.mydomain.com`，会得到一个这样的网页：

```html
<html><body>
<h1>DERP</h1>
<p>
  This is a
  <a href="https://tailscale.com/">Tailscale</a>
  <a href="https://pkg.go.dev/tailscale.com/derp">DERP</a>
  server.
</p>
```

## 配置服务和防火墙

每次手动执行derper命令肯定不现实，所以这时候就要`systemd`出马了。进入`/etc/systemd/system`，新建一个名为`derper.service`的文件，输入如下内容：

```ini
[Unit]
# 服务名字
Description=Tailscale DERP Server
# 在网络服务启动后启动这个服务
After=network.target

[Service]
# 改成你的用户名
User=admin
# 总是自动重新启动
Restart=always
# 重启前等待5秒
RestartSec=5
# 启动derper的命令，跟上面测试用的命令一样
ExecStart=/home/admin/go/bin/derper -c=/home/admin/derper/derper.conf -hostname derp.mydomain.com
# 停止derper的命令
ExecStop=/bin/kill $MAINPID
# 赋予CAP_NET_BIND_SERVICE这个capability
AmbientCapabilities=CAP_NET_BIND_SERVICE

[Install]
WantedBy=multi-user.target
```

然后分别执行如下命令：

```bash
# 载入service文件的变更
$ sudo systemctl daemon-reload

# 启动derper
$ sudo systemctl start derper

# 检查状态
$ sudo systemctl status derper
● derper.service - Tailscale DERP Server
     Loaded: loaded (/etc/systemd/system/derper.service; enabled; vendor preset: enabled)
     Active: active (running) since Wed 2023-02-08 14:27:13 CST; 2s ago
   Main PID: 4944 (derper)
      Tasks: 6 (limit: 1006)
     Memory: 2.6M
        CPU: 16ms
     CGroup: /system.slice/derper.service
             └─4944 /home/admin/go/bin/derper -c=/home/admin/derper/derper.conf -hostname derp.mydomain.com

Feb 08 14:27:13 iZj6caykwpo11gr659a9avZ systemd[1]: Started Tailscale DERP Server.
Feb 08 14:27:13 iZj6caykwpo11gr659a9avZ derper[4944]: 2023/02/08 14:27:13 derper: serving on :443 with TLS
Feb 08 14:27:13 iZj6caykwpo11gr659a9avZ derper[4944]: 2023/02/08 14:27:13 running STUN server on [::]:3478

# 开机自启动
$ sudo systemctl enable derper
Created symlink /etc/systemd/system/multiuser.target.wants/derper.service → /etc/systemd/system/derper.service.
Unit /etc/systemd/system/derper.service is added as a dependency to a non-existent unit multiuser.target.
```

因为derper依赖HTTP、HTTPS和STUN协议，所以需要配置防火墙或安全组，开放`80/tcp`，`443/tcp`，和`3478/udp`端口。

## 配置Tailscale

现在自建的DERP节点就成功启动了，接下来我们就需要让Tailscale知道这个节点的信息。进入Tailscale的Admin console，进入`Access controls`，在JSON中增加如下配置：

```json
{
    // 前略
    "derpMap": {
        // 如果想要所有节点只使用自建中继的话，就启用这条配置
        // "OmitDefaultRegions": true,
        "Regions": {
            "900": {
                "RegionID":   900,
                "RegionCode": "Aliyun-HKG",
                "Nodes": [
                    {
                        "Name": "Aliyun-HKG-1",
                        "RegionID": 900,
                        "HostName": "derp.mydomain.com",
                    },
                ],
            },
            // 如果有多个区域、多个节点，或者使用了自定义端口，那么可以参考这部分
            "901": {
                "RegionID": 901,
                "RegionCode": "Oracle-OSAKA",
                "Nodes": [
                    {
                        "Name": "Oracle-OSAKA-1",
                        "RegionID": 901,
                        "HostName": "osaka1.derp.mydomain.com",
                        "DERPPort": 4443,
                    },
                    {
                        "Name": "Oracle-OSAKA-2",
                        "RegionID": 901,
                        "HostName": "osaka2.derp.mydomain.com",
                        "DERPPort": 4443,
                    },
                ]
            }
        },
    },
}
```

最后不要忘了点击`Save`保存。保存成功后，可以在`Machines`中随便点进一台机器，看`Relays`里面有没有出现刚刚添加的DERP服务器。

![](https://blog-static.boris1993.com/self-hosting-tailscale-derp/relays.png)

## 官方文档

-   Custom DERP Servers
