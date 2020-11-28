---
title: "Docker搭建dns服务器"
date: 2020-11-29T00:58:51+08:00
draft: false
original: true
categories: 
  - Docker
tags: 
  - Docker
  - DNS
---


### 背景

最近在用flutter开发app，准备和后端接口进行联调，发现通过域名请求不到数据，由于我使用的域名*.wemeng.com，这个不是公网的，所以根据这个域名是解析不到正确的ip的

后端nginx识别了*.wemeng.com，将请求转到不同的后端服务，所以必须使用域名进行访问，要不然改动和维护ip就太麻烦了

之前开发后端接口的时候，是直接配置本地的hosts文件，将域名与ip进行关联了，但是在移动app上开发的时候，发现不能修改hosts文件，这很烦，如果需要修改hosts文件需要root，尝试了半天，发现还是很麻烦，而且也没有成功，所以按这个思路走

然后我打算放弃使用域名进行访问，直接用ip进行访问吧，麻烦事麻烦了一点，但是后端使用了minio，生成的访问链接地址是带地址的，如果直接改链接中的域名ip地址的话，由于链接是有签名的，改了就访问不了了，所以逼着我继续使用域名来访问

### 如何解决

我想通过域名来进行访问，但是又不能修改本地的hosts文件，怎么弄呢？

域名是又dns服务器解析并返回响应的ip的，如果DNS服务器能识别我们的域名就好了，之前看到过可以用docker起一个dns服务器，于是思路就来了

1. 我们自建一个dns服务器，这个dns服务器中添加我们想要的域名配置
2. 将手机的dns服务器配置成我们自己dns的服务器

按照这个思路，好像行得通，于是开撸

首先用docker起一个dns服务器

```yaml
version: '3.7'

services:
  dnsmasq:
    container_name: sx-dnsmasq
    image: andyshinn/dnsmasq
    cap_add:
      - NET_ADMIN
    ports:
      - "53:53/tcp"
      - "53:53/udp"
    volumes:
      - ./etc/resolv.dnsmasq:/etc/resolv.dnsmasq
      - ./etc/dnsmasq.conf:/etc/dnsmasq.conf
      - ./etc/dnsmasqhosts:/etc/dnsmasqhosts
    networks:
      - sx-net
networks:
  sx-net:
    external: true
```

详情请参考 https://github.com/zengqiang96/docker-compose/tree/local/dns-server

主要就是三个配置文件`/etc/resolv.dnsmasq`、`/etc/dnsmasq.conf`、`/etc/dnsmasqhosts`

然后就是将mac和手机dns配置成我们dns服务器的地址

![image-20200820150436441](/docker搭建dns服务器/image-20200820150436441.png)

手机配置

![](/docker搭建dns服务器/手机配置dns.jpg)

然后可以用域名访问一下，我在mac上用dig或ping试一下

```
dig minio.wemeng.com

; <<>> DiG 9.10.6 <<>> minio.wemeng.com
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 42415
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;minio.wemeng.com.		IN	A

;; ANSWER SECTION:
minio.wemeng.com.	0	IN	A	192.168.0.199

;; Query time: 2 msec
;; SERVER: 192.168.0.199#53(192.168.0.199)
;; WHEN: Thu Aug 20 15:11:36 CST 2020
;; MSG SIZE  rcvd: 61
```

```
ping minio.wemeng.com
PING minio.wemeng.com (192.168.0.199): 56 data bytes
64 bytes from 192.168.0.199: icmp_seq=0 ttl=64 time=1.886 ms
64 bytes from 192.168.0.199: icmp_seq=1 ttl=64 time=24.368 ms
```

### 问题记录

1. 53端口被占用

dns服务默认是用的53端口被占用了。查看本机端口占用情况：

```
$ lsof -i:53
COMMAND   PID            USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
systemd-r 804 systemd-resolve   12u  IPv4  22393      0t0  UDP localhost:domain
systemd-r 804 systemd-resolve   13u  IPv4  22394      0t0  TCP localhost:domain (LISTEN)
```

发现在宿主机器上有一个**dnsmasq**服务。google一番了解到，原来是**ubuntu**默认安装了**dnsmasq-base**服务。官网提到：

> Note that the package “dnsmasq” interferes with Network Manager which can use “dnsmasq-base” to provide DHCP services when sharing an internet connection. Therefore, if you use network manager (fine in simple set-ups only), then install dnsmasq-base, but not dnsmasq. If you have a more complicated set-up, uninstall network manager, use dnsmasq, or similar software (bind9, dhcpd, etc), and configure things by hand.

发现是systemd-r命令占用了53端口，Google一下发现是systemd-resolved.service启动的，所以停掉该服务

```
systemctl stop systemd-resolved.service
systemctl disable systemd-resolved.service
```

这样虽然可以可以启动**dnsmasq**了，但是你会发现你机器上不能访问外网。

如何让**dnsmasq**和systemd-resolved.service一起，我还没发现渐变的方法，还是先禁用systemd-resolved.service

既然我们已经搭建了**dnsmasq**作为dns服务器 ，能不能让宿主机的dns走**dnsmasq**呢？

当然是没问题的啦，修改`/etc/resolv.conf`文件就行了

```
# This file is managed by man:systemd-resolved(8). Do not edit.
#
# This is a dynamic resolv.conf file for connecting local clients to the
# internal DNS stub resolver of systemd-resolved. This file lists all
# configured search domains.
#
# Run "systemd-resolve --status" to see details about the uplink DNS servers
# currently in use.
#
# Third party programs must not access this file directly, but only through the
# symlink at /etc/resolv.conf. To manage man:resolv.conf(5) in a different way,
# replace this symlink by a static file or a different symlink.
#
# See man:systemd-resolved.service(8) for details about the supported modes of
# operation for /etc/resolv.conf.

#nameserver 127.0.0.53
# 改为127.0.0.1
nameserver 127.0.0.1 
options edns0
```