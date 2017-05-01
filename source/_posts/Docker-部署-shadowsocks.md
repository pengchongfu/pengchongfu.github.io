---
title: Docker 部署 shadowsocks
date: 2017-05-02 01:03:46
tags: Docker shadowsocks
---

最近工作时也慢慢地接触到了后端方面的项目，很多项目部署时都用了 Docker Compose。因此也学习了一些 Docker 的基本操作。

然后突发奇想地想用 Docker 跑一跑 shadowsocks，这样的话可以减少配置的麻烦（虽然原来的方法已经很方便了。。）

### shadowsocks

之前一直装的是 python 版本，后来发现 shadowsocks-libev 在 ubuntu 16.10 或者更高版本上能够直接通过 apt 安装了，而这也正是 Docker 的优势所在，能够直接在期望的系统上执行命令。所以果断决定装 shadowsocks-libev。

### Docker

安装 Docker 很简单，只需要到官网按着教程走就行了，然后再学习学习 Docker 的基本原理和命令就可以。

所以剩下的事就简单了，可以写这样的一个 Dockerfile

```
FROM ubuntu:16.10

RUN apt update && apt install -y shadowsocks-libev

CMD /usr/bin/ss-server -s ::0 -s 0.0.0.0 -p 8388 -m aes-256-cfb -k password -a root -u
```

pull 镜像，安装 shadowsocks-libev，启动 shadowsocks-libev，这样就结束了，非常简单。启动的时候同时监听了 ipv4 和 ipv6，并且监听容器内部 8388 端口，设置加密方式和密码，以 root 权限运行，支持 udp。

注意，需要开启 Docker对 ipv6 的支持，参考[文档](https://docs.docker.com/engine/userguide/networking/default_network/ipv6/)）

由于 dockerd 是一个 service，因此可以直接添加配置文件 `/etc/docker/daemon.json`

```
{
    "ipv6": true,
    "fixed-cidr-v6": "2001:db8:1::/64"
}
```

然后再重启 dockerd

```
service docker restart
```

这样就支持 ipv6 了。当然要是不需要 ipv6 的话直接去掉 `-s ::0` 就可以了。

### build

build Docker 镜像

```
docker build -t ss:v1 .
```

### 启动

最后的话得对本机和 Docker 容器进行端口映射，映射本机 443 端口到 Docker 容器 8388 端口

```
docker run --rm -p 443:8388 -d ss:v1
```

done