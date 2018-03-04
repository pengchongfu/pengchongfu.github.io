---
title: LEDE 配置 ipv6
date: 2018-03-04 22:52:30
tags:
- LEDE
- shadowsocks
- ipv6
---

生命不息，折腾不止。

时间永是流驶，曾经美好的原生 ipv6 环境也失去了。所以只能通过 shadowsocks 的 socks5 代理连接到 ipv6，以下载北邮人的资源。

但是在各个设备上配置总是一个麻烦事，因此考虑在路由器上进行配置。

### 准备工作

LEDE 默认情况下是开启了 ipv6 nat 的，查看一下设备是否成功分配到了 ipv6 地址，如果没有的话可以查查开如何开启 ipv6 nat。这一步很简单就不在这介绍了。

### 配置 shadowsocks

采用的方案其实和【用 shadowsocks ipv6 代理 ipv4 流量】的方案一样，只是现在变成了【用 shadowsocks ipv4 代理 ipv6 流量】。

#### 配置 ss-redir

`ss-redir --help` 可知 `-b` 选项可以设置监听的地址，因此我们需要设置 `-b 0.0.0.0 -b ::` 让 `ss-redir` 监听 ipv6 地址。配置好之后可以 `netstat -nl` 查看是否成功监听了对应端口。

#### 配置 ip6tables

这一步也很简单，直接执行  
```
ip6tables -t nat -A PREROUTING -p tcp -j REDIRECT --to-ports 1234
```
就可以了，也可以加到 `/etc/firewall.user` 里启动时执行。

### 总结

按照上面的操作 ipv6 的 tcp 流量是可以成功代理的，至少可以下载北邮人的资源了😊。udp 流量目前还没有需求，就先不配置了。