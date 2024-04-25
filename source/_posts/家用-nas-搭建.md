---
title: 家用 nas 搭建
date: 2024-04-15 07:54:34
tags:
  - nas
  - 树莓派
---

nas 是 network attached storage 的缩写，也就是网络附属存储。顾名思义，主要是用来提供存储功能的设备。
对于家庭来说，可以用来存储拍摄的照片，下载的电影图书等资源；对于企业来说，可以用来存储公司的一些公共资料方便共享。
本质上来说，nas 和普通的电脑没有什么区别，比如 nas 大厂群晖和威联通的操作系统底层都是 linux。
对家庭场景来说，比如家庭影院的需求，智能家居管理的需求，我们需要一台不关机，功耗低的计算设备。这时可以有多种选择，每种选择都有其适用场景

- nas | 硬件上 nas 一般会附带好几个盘位，方便用户组建磁盘阵列，用来保护数据；软件上 nas 厂商会提供完整的套件，常用的文件备份家庭影院功能都能很好地满足
- nuc 小主机 | 硬件上 nuc 小主机没什么特别之处，一般比较小巧，并且网口较多适合做软路由；软件上基本都需要自己寻找开源的或者收费的自行搭建
- 工控机 ｜ 硬件上工控机因为主要为工厂设计，各种散热比较好，但对于家庭场景来说没太大用处，当然网口较多也是优点，同时还有很多工业上用到的接口；软件上也是需要自行搭建
- 树莓派 ｜ 树莓派主要是为了学生学习设计，为了方便做原型设计有很多家庭场景用不到的接口，比如显示屏摄像头 gpio，同时连硬盘接口都没有，只能通过 USB 接口连接硬盘；软件上也是需要自行搭建

因为本人比较喜欢自由，所以并不考虑 nas。而且软件自行搭建的话在 linux 上比 windows 更为方便。因此本文介绍在 linux 上如何搭建满足家庭场景下的常见需求。（基于 debian/ubuntu，但其他 linux 发行版也大同小异）
由于家用场景并不需要树莓派的各种接口，并且工控机和 nuc 小主机区别不大，因此本文选择 nuc 小主机作为家用计算设备

### 预备工作

nuc 小主机到手之后，需要进行安装系统等预备工作，由于很多操作较为简单，网上也有很多教程，因此以下简单列一下

- 刻录 ubuntu 镜像，安装到小主机上
- 安装 openssh-server 开启 ssh 服务
- 修改软件源
- 配置 shell（比如安装 [ohmyzsh](https://github.com/ohmyzsh/ohmyzsh)）和安装常用软件
- 可以考虑在路由器上固定小主机的 ip，或者参考前几篇博客配置 `dhcpcd` 固定 ip。这样之后通过 ipv4 访问小主机时都可以更方便

### 家庭场景下常用软件需求

- 远程连接需要的域名解析工具（ipv6）
- 远程连接需要的内网穿透工具
- 家庭影院
- 满足家庭影院的下载工具
- 软路由
- 家庭文件共享

#### 远程连接需要的域名解析工具（ipv6）

得益于运营商的大力推广，入户光猫基本都支持 ipv6。当然用户自行接入的路由器可能并不支持 ipv6，导致用户的设备拿不到 ipv6 地址，这种情况可以通过配置路由器桥接或者将路由器刷成其他系统例如 openwrt，然后安装软件解决。并且大多数时候我们都需要关闭光猫上的 ipv6 防火墙

为了能够远程连接小主机（比如说在外观看家庭影院的电影，打开下载工具后台新建下载任务），我们需要知道小主机的 ipv6 地址。但是 ipv6 地址过长难以记忆，并且这个地址并不固定，因此我们需要配置 [ddns](https://www.cloudflare.com/zh-cn/learning/dns/glossary/dynamic-dns/)

可以按照 [ddns-go](https://github.com/jeessy2/ddns-go) 项目的 README 进行安装，然后添加自己域名服务商的配置

#### 远程连接需要的内网穿透工具

有些场景下我们可能没有 ipv6 的环境，但运营商一般不会给家庭宽带分配 ipv4 公网地址。因此我们需要找一些内网穿透工具来实现
可以自行安装配置 zerotier 来实现这一功能

#### 家庭影院

家庭影院的选择比较多，比如收费的 plex，开源的 jellyfin。以下是安装 jellyfin 的方案

1. 按照官网安装说明执行
   `curl -s https://repo.jellyfin.org/install-debuntu.sh | sudo bash`
1. 之后按照指引配置即可

#### 满足家庭影院的下载工具

由于国内的各种云盘都需要登录且使用客户端下载，整个操作流程非常麻烦，因此使用 bt 是更好的选择。
而在小主机上直接下载好处更多，小主机不关机，因此可以一直下载。即使我们不在家，也只需要远程新建下载任务，之后只要等下好就行了。

qbittorrent 是一个很好的 bt 下载工具，我们需要安装它的无头版本 qbittorrent-nox

1. `sudo apt install qbittorrent-nox`
1. 由于它没有自带 systemd 配置，我们需要自己手动添加配置 `/etc/systemd/system/qbittorrent-nox.service`

   ```
    [Unit]
    Description=qBittorrent-nox service for user %I
    Documentation=man:qbittorrent-nox(1)
    Wants=network-online.target
    After=local-fs.target network-online.target nss-lookup.target

    [Service]
    Type=exec
    ExecStart=/usr/bin/qbittorrent-nox

    [Install]
    WantedBy=multi-user.target
   ```

1. 更新 systemd
   `sudo systemctl daemon-reload`
1. 开机自启
   `sudo systemctl enable qbittorrent-nox.service`
1. 启动
   `sudo systemctl restart qbittorrent-nox.service`
1. 打开本机 8080 端口的 web ui 进行配置，默认用户名为 admin，默认密码为 adminadmin。如果想要一个支持 rss 的 web ui 可以使用 [qb-web](https://github.com/CzBiX/qb-web)

#### 软路由

软路由的方案在本博客前几篇的内容中已经介绍过了，即使树莓派只有一个网口，应付家庭使用也足够了。当然如果小主机有多个网口那更好了。
强烈推荐 sing-box 作为软路由的方案

1. 安装 sing-box
   `bash <(curl -fsSL https://sing-box.app/deb-install.sh)`
1. 修改 sing-box systemd 配置
   `sudo vim /lib/systemd/system/sing-box.service`
   我喜欢将 sing-box 配置文件夹与工作目录都设置为 `.config/sing-box`，比如 `ExecStart=/usr/bin/sing-box -D /home/currentUser/.config/sing-box -C /home/currentUser/.config/sing-box run`，将 currentUser 替换为自己的用户名
1. 新建 sing-box 文件夹
   ```
   cd .config
   mkdir sing-box
   sudo chmod 777 sing-box
   ```
1. 在这个文件夹下添加 sing-box 的配置
1. 配置 systemd，开机自启以及重启 sing-box
   ```
   sudo systemctl enable sing-box
   sudo systemctl restart sing-box
   ```
1. 做完以上操作之后小主机应该就可以正常访问外部网站了，但是小主机作为网关还不能转发流量，因此需要开启流量转发
   `sudo vim /etc/sysctl.conf`，将 `net.ipv4.ip_forward=1` 的注释去掉，并重启小主机

#### 家庭文件共享

有些时候我们想要临时存一些文件，或者将大文件在不同设备间转移。可以在小主机中实现简单的文件共享功能

1. `sudo apt install samba`
1. 编辑 `/etc/samba/smb.conf` 添加配置即可，以下是一个例子
   ```
   [share]
   comment = share
   browseable = yes
   path = /share
   public = yes
   available = yes
   writable = yes
   ```
1. 重启
   `sudo systemctl restart smbd.service`

### PS

本文并没有介绍其他的一些常用场景，比如个人文件备份，照片同步，个人云盘（这三个场景对数据安全要求较高，最好使用商用 nas 组磁盘阵列）和 home assistant 等等，因为我并没有相关的需求
