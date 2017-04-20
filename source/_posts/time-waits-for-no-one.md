---
title: time waits for no one
date: 2016-04-29 09:30:43
tags:
---
这几天又开始折腾起openwrt来了。。工作还没找到却在各种瞎折腾让我有一种time waits for no one的感觉。

折腾的缘由是之前各种抓包之时，想到还不如直接用路由器来抓包，这样就能在将wifi共享给他人之时练习抓包技巧。

说起openwrt，大三时为了晚上断电之后也能愉快的玩手机所以买了一个tp-link mr10u

然后为了能够用上ipv6刷了openwrt进行了各种折腾，期间还弄坏了固件导致不能启动，只能买了一条ttl线拆机重新把系统刷上

结果最终还是以失败告终，唯一的收获是可以把路由器上闪人的led灯给关掉

然后这次折腾终于把ipv6给折腾好了（虽然由于路由器带宽太窄最快下载速度只有3M/s），并且成功挂载上了u盘，装上了tcpdump，为了防止以后忘了还是写下来吧

### 目标

路由器刷上openwrt之后，配置ipv6，挂载u盘扩充容量，安装软件进行抓包

### 准备

安装并学习linux的使用，阅读孙冰的openwrt[教程](http://www.leiphone.com/author/hoowa)，学习网络原理（要是没看完这一些还是别瞎折腾了，，买个极路由保平安）

openwrt路由器一个，我的路由器系统为BARRIER BREAKER(14.07)，网络环境为清华大学紫荆宿舍

### ipv6

安装好openwrt之后，由于路由器的wifi默认关闭，因此网线连接路由器，登陆路由器`ssh root@192.168.1.1`，（windows下用putty）

（如果是第一次登陆的话`telnet 192.168.1.1`，然后`passwd`设置密码）

修改无线配置

`vi /etc/config/wireless`

这个配置文件中有wifi设备和接口的设置（在openwrt的[官网](https://wiki.openwrt.org/zh-cn/doc/uci/wireless)可以看到关于各个配置文件的介绍）

只进行了两处修改：
1. 将config wifi-device中的option disabled的值改为0（这样会将wifi开关打开）
2. 然后设置一个接口文件，设置账号密码加密方式之类的，这是我的配置文件，仅供参考

```
config wifi-device  radio0
        option type     mac80211
        option channel  11
        option hwmode   11g
        option path     'platform/ar933x_wmac'
        option htmode   HT20
        # REMOVE THIS LINE TO ENABLE WIFI:
#       option disabled 1

config wifi-iface
        option device   radio0
        option network  lan
        option mode     ap
        option ssid     youracount
        option encryption psk2
        option key yourpassword
```

下一步是设置网络配置

`vi /etc/config/network`

配置说明直接看[教程](https://wiki.openwrt.org/zh-cn/doc/uci/network)

进行了三处修改：
1. 修改interface lan，注释掉option ifname eth0（这是因为我的路由器上只有一个网口，可以将这个接到wan上，也可以接到lan上，现在我们把它接到wan上）
2. 添加interface wan
3. 添加interface wan6（ipv6配置，ifname选项@wan表示与wan相同）

这是我的配置文件
```
config interface 'loopback'                                                                                                           
        option ifname 'lo'                                                                                                            
        option proto 'static'                                                                                                         
        option ipaddr '127.0.0.1'                                                                                                     
        option netmask '255.0.0.0'                                                                                                    
                                                                                                                                      
config globals 'globals'                                                                                                              
        option ula_prefix 'fdae:9a33:93b6::/48'                                                                                       
                                                                                                                                      
config interface 'lan'                                                                                                                
#       option ifname 'eth0'                                                                                                          
        option force_link '1'                                                                                                         
        option type 'bridge'                                                                                                          
        option proto 'static'                                                                                                         
        option ipaddr '192.168.1.1'                                                                                                   
        option netmask '255.255.255.0'                                                                                                
        option ip6assign '60'                                                                                                         
                                                                                                                                      
config interface wan                                                                                                                  
        option ifname eth0                                                                                                            
        option proto dhcp                                                                                                             
                                                                                                                                      
config interface wan6                                                                                                                 
        option ifname @wan                                                                                                            
        option proto dhcpv6 
```

下一步配置dhcp

`vi /etc/config/dhcp`

这一步的各种设置我也不是特别清楚，主要是添加各种relay？因此直接贴配置文件好了(还是学好网络原理比较重要)
```
config dnsmasq                                                                                                                                                                                  
        option domainneeded '1'                                                                                                                                                                 
        option boguspriv '1'                                                                                                                                                                    
        option filterwin2k '0'                                                                                                                                                                  
        option localise_queries '1'                                                                                                                                                             
        option rebind_protection '1'                                                                                                                                                            
        option rebind_localhost '1'                                                                                                                                                             
        option local '/lan/'                                                                                                                                                                    
        option domain 'lan'                                                                                                                                                                     
        option expandhosts '1'                                                                                                                                                                  
        option nonegcache '0'                                                                                                                                                                   
        option authoritative '1'                                                                                                                                                                
        option readethers '1'                                                                                                                                                                   
        option leasefile '/tmp/dhcp.leases'                                                                                                                                                     
        option resolvfile '/tmp/resolv.conf.auto'                                                                                                                                               
                                                                                                                                                                                                
config dhcp 'lan'                                                                                                                                                                               
        option interface 'lan'                                                                                                                                                                  
        option start '100'                                                                                                                                                                      
        option limit '150'                                                                                                                                                                      
        option leasetime '12h'                                                                                                                                                                  
        option ra 'relay'                                                                                                                                                                       
        option ndp relay                                                                                                                                                                        
                                                                                                                                                                                                
config dhcp 'wan'                                                                                                                                                                               
        option interface 'wan'                                                                                                                                                                  
        option ignore '1'                                                                                                                                                                       
        option ra 'relay'                                                                                                                                                                       
        option ndp relay                                                                                                                                                                        
        option master 1                                                                                                                                                                                                                                                                            
        
config odhcpd 'odhcpd'                                                                                                                                                                          
        option maindhcp '0'                                                                                                                                                                     
        option leasefile '/tmp/hosts/odhcpd'                                                                                                                                                    
        option leasetrigger '/usr/sbin/odhcpd-update' 
```

重启路由器，连上路由器，应该就能连上网以及ipv6了

### 挂载u盘

安装好openwrt之后可用空间只有可怜的400k了，因此需要扩容，这个[教程](http://www.jakehu.me/2015/TP-Link-703N-Openwrt-Shadowsocks/)说的很清楚

唯一不同的是当分区大于2G时mount会出错，所以只能将分区限制在2G

### 安装软件

安装软件的过程很简单

```
opkg update
opkg install tcpdump
```

安装完成之后就能和之前一样抓包了