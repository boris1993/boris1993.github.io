---
title: "常用lsof命令备忘"
source: https://www.boris1993.com/useful-lsof-commands.html
date: 2017-12-26
updated: 2026-10-10
tags: [Shell, lsof]
categories: [小技巧]
---

# 常用lsof命令备忘

记录lsof命令常见用法备忘

# 网络

```bash
# 显示所有网络连接
lsof -i
```

## 根据协议类型筛选

```bash
# 仅显示IPv4网络连接
lsof -i 4

# 仅显示IPv6网络连接
lsof -i 6

# 仅显示TCP连接
lsof -iTCP

# 仅显示UDP连接
lsof -iUDP
```

## 根据目标地址和端口号筛选

```bash
# 根据目标地址筛选
lsof -i@${HOSTNAME_OR_IP_ADDRESS}

# 根据端口号筛选
lsof -i :${PORT_NUMBER}

# 组合
lsof -i@${HOSTNAME_OR_IP_ADDRESS}:${PORT_NUMBER}
```

## 根据端口状态筛选

```bash
lsof -i -sTCP:${STATE}

# 示例
lsof -i -sTCP:LISTEN
lsof -i -sTCP:ESTABLISHED
```

## 查看某进程端口占用

```bash
lsof -p ${PID}
```

# 用户

```bash
# 显示当前用户打开的文件
lsof -u ${USER}

# 显示除当前用户以外的用户打开的文件
lsof -u ^${USER}
```

# 进程

```bash
# 仅显示PID而不是所有输出信息
lsof -t

# 根据程序名筛选
lsof -c ${COMMAND}
```
