---
title: "在Spring Boot应用中启用HTTPS"
source: https://www.boris1993.com/spring-boot-enable-https.html
date: 2019-12-17
updated: 2026-10-10
tags: [Java, Spring Boot]
categories: [学知识]
---

# 在Spring Boot应用中启用HTTPS

在`application.yml`中添加如下配置，即可在Spring Boot项目中开启HTTPS。

```yaml
server:
  # 如果有需要的话，配置应用监听HTTPS的默认端口
  port: 443
  ssl:
    # 我把证书放在了 resources/cert/certificate.pfx下
    # 使用时需要按照实际情况修改下面的路径
    key-store: classpath:cert/certificate.pfx
    key-store-password: pA55w0Rd
    # 证书类型需要按照实际类型填写
    # 本例使用PKCS12格式，所以写PKCS12。如果是JKS格式，那么需要填写JKS
    # 因为PKCS12比JKS有更好的通用型，所以个人建议使用PKCS12格式
    key-store-type: PKCS12
    enabled: true
```

使用如下命令即可将JKS证书转换为PKCS12证书：

```bash
keytool -importkeystore -srckeystore keystore.jks -destkeystore keystore.pfx -deststoretype pkcs12
```

\[^1\]: How to enable HTTPS in a Spring Boot Java application  
\[^2\]: Spring Boot Docs - Configure SSL
