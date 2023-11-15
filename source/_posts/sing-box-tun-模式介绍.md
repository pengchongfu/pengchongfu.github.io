---
title: sing-box tun 模式介绍
date: 2023-11-12 21:24:59
tags:
- 代理
- sing-box
- 树莓派
---

上一篇文章中介绍了使用 clash 将树莓派配置为代理网关的方案，但因为众所周知的原因，clash 已经删库了。并且开源版的 clash 并不支持 tun 模式。
而不管是 redirect、tproxy 都需要修改树莓派的防火墙配置，步骤较为繁琐。因此本文介绍一下 sing-box 这个项目的安装使用。
sing-box 有以下优点：
- 支持 tun
- 支持 iOS（以及其他 apple 设备）

### 安装
参考 [sing-box](https://sing-box.sagernet.org/examples/linux-server-installation/) 安装教程安装到树莓派上（安装教程会自动生成 systemd 配置）。

### 配置
```
{
    "dns": {
        "servers": [
            {
                "tag": "aliyun",
                "address": "223.5.5.5",
                "detour": "direct"
            },
            {
                "tag": "fakeip",
                "address": "fakeip"
            }
        ],
        "rules": [
            {
                "domain_suffix": [
                    ".cn"
                ],
                "geosite": [
                    "cn",
                    "apple-cn"
                ],
                "server": "aliyun"
            },
            {
                "outbound": "any",
                "server": "aliyun"
            },
            {
                "query_type": [
                    "A",
                    "AAAA"
                ],
                "server": "fakeip",
                "rewrite_ttl": 10
            }
        ],
        "fakeip": {
            "enabled": true,
            "inet4_range": "198.18.0.0/15",
            "inet6_range": "fc00::/18"
        },
        "independent_cache": true
    },
    "route": {
        "final": "proxy",
        "auto_detect_interface": true,
        "geoip": {
            "download_url": "https://github.com/soffchen/sing-geoip/releases/latest/download/geoip.db",
            "download_detour": "proxy"
        },
        "geosite": {
            "download_url": "https://github.com/soffchen/sing-geosite/releases/latest/download/geosite.db",
            "download_detour": "proxy"
        },
        "rules": [
            {
                "protocol": "dns",
                "outbound": "dns-out"
            },
            {
                "inbound": "dns-in",
                "outbound": "dns-out"
            },
            {
                "domain_suffix": [
                    ".cn"
                ],
                "geosite": [
                    "cn",
                    "apple-cn"
                ],
                "geoip": [
                    "cn"
                ],
                "ip_cidr": [
                    "2000::/3"
                ],
                "outbound": "direct"
            }
        ]
    },
    "inbounds": [
        {
            "type": "tun",
            "tag": "tun",
            "inet4_address": "172.19.0.1/30",
            "inet6_address": "fdfe:dcba:9876::1/126",
            "auto_route": true,
            "sniff": true
        },
        {
            "type": "direct",
            "tag": "dns-in",
            "listen": "::",
            "listen_port": 53
        }
    ],
    "outbounds": [
        {
            "type": "direct",
            "tag": "direct"
        },
        {
            "type": "dns",
            "tag": "dns-out"
        },
        {
            "type": "shadowsocks",
            "tag": "proxy",
            "server": "server",
            "server_port": server_port,
            "method": "method",
            "password": "password"
        }
    ]
}
```

这是一份比较简单的分流配置，将 `"tag": "proxy"` 处修改为你自己的代理配置，然后重启 sing-box。这时候应该能正常运行，可以访问相应的网站
```
➜  ~ curl google.com
<HTML><HEAD><meta http-equiv="content-type" content="text/html;charset=utf-8">
<TITLE>301 Moved</TITLE></HEAD><BODY>
<H1>301 Moved</H1>
The document has moved
<A HREF="http://www.google.com/">here</A>.
</BODY></HTML>
```

### tun 模式原理

tun 可以简单理解成为一个网络接口。在 linux 中，防火墙负责对数据包做拦截、转换之类的处理。而数据包流向主要由策略路由和路由表负责。
我们查看一下策略路由：

```
➜  ~ sudo ip rule
0:	from all lookup local
9000:	from all to 172.19.0.0/30 lookup 2022
9001:	from all ipproto icmp goto 9010
9002:	not from all dport 53 lookup main suppress_prefixlength 0
9002:	not from all iif lo lookup 2022
9002:	from 0.0.0.0 iif lo lookup 2022
9002:	from 172.19.0.0/30 iif lo lookup 2022
9010:	from all nop
32766:	from all lookup main
32767:	from all lookup default
```

1. 第一行主要是负责处理发送到本机的数据包
1. 第二行将目的地址为 tun 所在子网的数据包，查询 2022 路由表
1. 第三行处理 icmp
1. 第四行将目的端口不为 53 的数据包，查询 main 路由表，并且 suppress_prefixlength 0 表示不匹配默认路由
1. 之后的几行都是查询 2022 路由表

因此，当我们执行 `curl google.com` 时
1. 首先是本机到 dns 服务器的 dns 请求，会匹配到第五行查询 2022 路由表
1. 接着发起请求时，也会查询 2022 路由表

那么让我们看一下 2022 路由表的内容

```
➜  ~ sudo ip route show table 2022
default dev tun0
```

可以看到非常简单，就是直接将所有数据包交由 tun 处理

### sing-box 配置介绍

首先，最好先简单阅读一遍官方文档。这样会有个大概印象。

- `inbounds` 是数据包的输入。在上面的配置中只有一个 tun 模式的 inbound
    - `auto_route` 会自动配置策略路由，也就是上面提到的
    - `sniff` 会嗅探出数据包对应的协议，以便在 `route` 中使用
- `outbounds` 是数据包的目的地。三个 outbound 中，direct 和 proxy 用于分流。国内流量走 direct，国外走 proxy。dns 为一个特殊的出口，可以看作是 `dns` 配置中对应的 dns 服务器
- `dns` 可以看作一个 dns 服务器（虽然可能实际并不监听端口），`dns.rules` 对 dns 请求做分流，国内的 dns 请求走 tag 为 aliyun 的 dns 服务器（当然因为 outbound 中代理服务器的地址也可能是一个域名，因此 outbound 的 dns 请求也走这个），其他的 dns 请求走 tag 为 fakeip 的 dns 服务器，直接返回 fake ip 以减少没必要的请求
- `route` 会对流量进行分流。`route.ruls` 中，嗅探结果为 dns 的数据包，路由到 tag 为 dns 的 outbound；请求国内域名、IP的，路由到 tag 为 direct 的 outbound; 其他的由 `route.final` 配置兜底，都发往 tag 为 proxy 的 outbound


### PS
需要注意的是，子网内的设备配置时，dns 服务器不能配置为路由器的地址。因为路由器在子网之内，dns 请求将直接发往路由器而不会发到配置的树莓派网关，没法做 dns 劫持