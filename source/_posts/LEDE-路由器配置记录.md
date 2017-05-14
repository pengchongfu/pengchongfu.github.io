---
title: LEDE 路由器配置记录
date: 2017-05-13 19:24:25
tags:
- LEDE
- openwrt
- shadowsocks
---

最近总有人问起我该如何配置 LEDE 路由器的网络代理功能。而想到我之前也帮好几个朋友配置过路由器，他们以后可能也需要了解了解关于这方面的知识。干脆从零开始写一个简单的配置记录吧。

想起了 openwrt 的第一次接触，应该是大三的时候了吧。当时啥都不懂，连 linux 的基本操作都不会。但是想着能连接路由器用 ipv6，就给路由器刷上了 openwrt。当然因为对网络一窍不通，最后以失败告终。唯一的收获是成功关上了路由器上的 led :)。

所以开始配置路由器吧，前提条件是对 linux 和网络有基本的了解。

### 安装 LEDE

[LEDE](https://lede-project.org/start) 是一个基于 openwrt 的发行版。貌似是觉得 openwrt 开发太慢了，于是 fork 了一份代码重新开发。

安装的教程网上有很多，此处就不赘述了。大概思路就是下载固件，`scp` 到路由器上（当然要是从原生的系统刷的话还要用到 `ftp`），然后安装。这里我们选择目前最新的稳定版 LEDE 17.01.1。

装上之后修改一下密码，打开 wifi，配置好 wifi。

具体的教程可以参考[这里](/post/time-waits-for-no-one/)

### 配置 ipv6

下一步先配置 ipv6。我所使用过的 ipv6 配置方式主要有两种：一种是 nat，一种是像这篇[文章](/post/time-waits-for-no-one/)里的使用邻居发现协议的方法。

第二种方法看起来很不错，因为可以获取公有 ipv6 地址，但是需要上级路由器支持。。而高校内的设备有很多并不支持邻居发现协议。
这就会导致：电脑发送的包成功送到了 vps；vps 通过 `traceroute` 只能追踪到路由器的上级高校设备。
而通过抓包发现，上级高校设备广播 `who has` 报文，询问电脑在哪；路由器也成功向上级高校设备发送了响应报文，说电脑在我这。但是上级高校设备不予理会，继续发送 `who has` 报文。。。


所以我们使用 nat 的方式比较稳定可靠。我们可以参考这个[教程](http://blog.csdn.net/cod1ng/article/details/45421025)和这里的[讨论](https://github.com/tuna/ipv6.tsinghua.edu.cn/issues/7)来进行配置。

安装需要的软件

```
opkg update
opkg install kmod-ipt-nat6
```

向 `/etc/firewall.user` 添加以下一行已进行 ipv6 的 nat

```
ip6tables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
```

修改 `/etc/config/dhcp`，添加 `ra_default` 和 `ra_management` 两项

```
config dhcp 'lan'
    option interface 'lan'
    option start '100'
    option limit '150'
    option leasetime '12h'
    option dhcpv6 'server'
    option ra 'server'
    option ra_management '1'
    option ra_default '1'
```

当然现在还是不行的。因为路由器的路由表是有问题的。我们必须修改路由表。在路由器上输入命令

```
ip -6 route show default
```

可以看到默认的路由表。其中 `via` 字段后的就是上级设备的 ip，我们只需要将这个路由规则用到所有的情况就可以了

```
ip -6 route add default via xxxx::xxxx dev eth0 metric 512
```

这时应该就可以了，当然这个配置是临时的，我们需要加到启动项中
`/etc/hotplug.d/iface/` 下新建一个文件 `90-ipv6`

```
#!/bin/sh
[ "$ACTION" = ifup ] || exit 0
ip -6 route add default via xxxx::xxxx dev eth0 metric 512
```


### 挂载 u盘

由于我的路由器太古老了。。容量太低，不挂载 u盘的话连 shadowsocks 都装不上。因此需要挂载 u盘。要是路由器容量比较大的话可以省略这一步。
主要参考官方[文档](https://lede-project.org/docs/user-guide/extroot_configuration)就可以了，文档很详细明了。我的路由器容量实在太低，需要自行编译固件，不赘述了，所以还是买一个大容量的路由器比较靠谱。。

### 安装 shadowsocks

到 github 的 [release](https://github.com/shadowsocks/openwrt-shadowsocks/releases) 处下载对应硬件版本的安装包进行安装。

添加配置文件 `/etc/shadowsocks.json`

```
{
    "server": "X.X.X.X",
    "server_port": "8388",
    "password": "password",
    "local_port": "1080",
    "method": "aes-256-cfb"
}
```
*地址填 ipv6 的话可以代理 ipv4 流量，大大缓解校园网计费的问题，虽然会有些卡*

添加启动脚本 `/etc/init.d/shadowsocks`

```
#!/bin/sh /etc/rc.common

START=95

SERVICE_USE_PID=1
SERVICE_WRITE_PID=1
SERVICE_DAEMONIZE=1
SERVICE_PID_FILE=/var/run/shadowsocks.pid
CONFIG=/etc/shadowsocks.json

start() {
    service_start /usr/bin/ss-redir -c $CONFIG -b 0.0.0.0 -f $SERVICE_PID_FILE
}

stop() {
    service_stop /usr/bin/ss-redir
}
```

添加执行权限

```
chmod +x /etc/init.d/shadowsocks
```

运行并设置开机启动

```
/etc/init.d/shadowsocks enable
/etc/init.d/shadowsocks start
```

参考 [这里](https://gist.github.com/wen-long/8644243)，我们可以先执行以下几条命令

```
iptables -t nat -N SHADOWSOCKS                                      # 新建链
iptables -t nat -A SHADOWSOCKS -p tcp -j REDIRECT --to-ports 1080   # 插入重定向规则
iptables -t nat -I PREROUTING -p tcp -j SHADOWSOCKS                 # 使得链生效
```

因此，不想走代理的 ip 只需要在重定向之前 return 就可以了

参照上边链接里的介绍就能自定义规则了。当然，我希望默认是全部代理的模式，因为可以代理 ipv4 的流量，所以直接将代理全部的规则写入 `/etc/firewall.user` 中，保证开机时能自动加载。

### DNS 配置

下一个是关于 DNS 的配置。由于 shadowsocks 自带 `ss-tunnel` 命令，我们直接使用它进行 DNS 代理即可。

修改 `/etc/init.d/shadowsocks`

```
#!/bin/sh /etc/rc.common

START=95

SERVICE_USE_PID=1
SERVICE_WRITE_PID=1
SERVICE_DAEMONIZE=1
SERVICE_PID_FILE=/var/run/shadowsocks.pid
CONFIG=/etc/shadowsocks.json

start() {
    service_start /usr/bin/ss-redir -c $CONFIG -b 0.0.0.0 -f $SERVICE_PID_FILE
    service_start /usr/bin/ss-tunnel -c $CONFIG -b 0.0.0.0 -l 5353 -L 8.8.8.8:53 -u
}

stop() {
    service_stop /usr/bin/ss-redir
    service_stop /usr/bin/ss-tunnel
}
```

添加了两条关于 `ss-tunnel` 的命令。监听路由器 5353 端口，向 8.8.8.8:53 查询 DNS。
这时可以在电脑上执行

```
dig @192.168.1.1 -p 5353 www.google.com
```

验证程序是否正常执行。

然后我们需要将 DNS 查询导向路由器 5353 端口。lede 的 DNS 查询是由 dnsmasq 负责的。

修改 `/etc/dnsmasq.conf`，在最后加入 `conf-dir=/etc/dnsmasq.d`，让该程序读取我们的配置。

接着在 `/etc/dnsmasq.d` 内新建文件 `proxy.conf`，加入一下一行规则

```
server=/#/127.0.0.1#5353
```

重启 `dnsmasq`，`/etc/init.d/dnsmasq restart`

这样所有向路由器 53 端口查询 DNS 的请求都会导向路由器 5353 端口，通过 `ss-tunnel` 进行查询。

这样的话会有个小问题，就是所有 DNS 查询都会导向 vps，会比较慢，所以我们可以下载[这里](https://github.com/softwaredownload/openwrt-fanqiang/blob/master/openwrt/default/etc/dnsmasq.d/accelerated-domains.china.conf)的规则列表。同样放到这个文件夹中，优先匹配到的会直接向 `114.114.114.114` 查询，这样就能加速部分网站的域名解析过程。

### 总结

至此，整个配置流程就完成了。很多地方写得不够详细，只是有个大体思路而已。

配置路由器还是很麻烦的，需要对 linux 和 网络有基本的了解，并且熟悉其中的各种命令，遇到问题能够通过自行调试解决。

总的来说不适合小白（当我是个小白的时候完全不知所以，稍微有一个地方和教程不同就进行不下去了）。。还是找个程序员来配置吧。