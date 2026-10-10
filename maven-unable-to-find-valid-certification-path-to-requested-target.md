---
title: "解决Maven的unable to find valid certification path to requested target错误"
source: https://www.boris1993.com/maven-unable-to-find-valid-certification-path-to-requested-target.html
date: 2020-03-30
updated: 2026-10-10
tags: [Maven]
categories: [小技巧]
---

# 解决Maven的unable to find valid certification path to requested target错误

今天我在从公司的Maven私库上下载依赖时，Maven报出了`unable to find valid certification path to requested target`的问题，经查询是因为没有将私库使用的自签名证书导入到Java的keystore。这里写一下我的两个解决方案，方便发生了同样问题的人参考。

## 让Maven忽略证书错误

因为在默认情况下，自签名证书是不受信任的，那么我们只要让Maven忽略证书错误，就可以解决这个问题了。

我们可以在`.bashrc`或`.zshrc`（或者其他位置，看你用哪个shell了）里面加入如下语句：

```bash
export MAVEN_OPTS='-Dmaven.wagon.http.ssl.insecure=true -Dmaven.wagon.http.ssl.allowall=true -Djsse.enableSNIExtension=false'
```

或者，也可以在`~/.mavenrc`中配置这个参数：

```properties
MAVEN_OPTS="-Dmaven.wagon.http.ssl.insecure=true -Dmaven.wagon.http.ssl.allowall=true -Djsse.enableSNIExtension=false"
```

## 导入证书到keystore

首先使用这个命令得到网站上面的证书：

```bash
# 注意将mavenrepository.mycompany.com替换成实际的地址
openssl s_client -showcerts -connect mavenrepository.mycompany.com:443
```

在输出中可以看到类似这样的证书信息：

```text
s:/C=US/ST=North Carolina/L=Raleigh/O=Red Hat, Inc./OU=Red Hat Network/CN=Entitlement Master CA/emailAddress=ca-support@redhat.com

i:/C=US/ST=North Carolina/L=Raleigh/O=Red Hat, Inc./OU=Red Hat Network/CN=Entitlement Master CA/emailAddress=ca-support@redhat.com
```

我们要在这里面寻找`issuer(i:)`和`subject(s:)`相同的证书，这个就是我们要导出的根证书。找到之后，保存证书的哈希（即以`—–BEGIN CERTIFICATE—–`开始，以`—–END CERTIFICATE—–`结束的这部分内容）到一个文件中，比如`root.crt`。

接下来，我们需要使用`keytool`把这个证书导入到Java的keystore中，命令如下：

```bash
sudo keytool \
    -importcert \
    # keystore文件位置需要修改成实际的位置
    -keystore /usr/local/java/jdk1.8.0_60/jre/lib/security/cacerts \
    # keystore密码需要修改成实际的密码，默认密码是changeit
    -storepass changeit \
    # 要导入的证书，就是刚才保存的那个，注意使用实际的位置
    -file ~/Downloads/root.crt \
    # 给证书起个名字
    -alias "mavenrepository-mycompany"
```

执行后keytool会询问是否信任这个证书，输入`yes`来信任。

导入完成后，Maven就可以从这个私库下载依赖了。如果你使用IDEA的话，那么可能需要重新启动一下IDEA才会使新的keystore生效。

\[^1\]: How to resolve “unable to find valid certification path to requested target” error? - JFrog  
\[^2\]: How to tell Maven to disregard SSL errors (and trusting all certs)?
