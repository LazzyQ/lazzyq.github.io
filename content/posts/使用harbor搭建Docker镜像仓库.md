---
title: "使用harbor搭建Docker镜像仓库"
date: 2021-01-24T15:01:54+08:00
draft: false
original: true
categories: 
  - Docker
tags: 
  - Docker
---

### 下载harbar的安装器

在[official releases](https://github.com/goharbor/harbor/releases)页面可以下载到harbor的安装器，分为在线版和离线版

* 在线版：在线安装器会从Docker Hub上下载Harbor镜像，安装器的体积比较小
* 离线版：离线安装器包含构建好的镜像，所以比在线版的体积大

这里我使用在线版本，下载后解压

```
$ tar  xvf harbor-online-installer-v2.1.3.tgz
```

<!--more-->

### 配置Https访问Harbor


#### 生成自签名根证书

1. 生成CA证书私钥

```
openssl genrsa -out ca.key 4096
```

2. 生成CA证书

```
openssl req -x509 -new -nodes -sha512 -days 3650 -key ca.key -out ca.crt


Country Name (2 letter code) [AU]:CN
State or Province Name (full name) [Some-State]:Sichuan
Locality Name (eg, city) []:Chengdu
Organization Name (eg, company) [Internet Widgits Pty Ltd]:wemeng
Organizational Unit Name (eg, section) []:Personal
Common Name (e.g. server FQDN or YOUR name) []:harbor.wemeng.com
Email Address []:zengqiang96@gmail.com
```

#### 生成服务器证书

1. 生成私钥

```
openssl genrsa -out harbor.wemeng.com.key 4096
```

2. 生成证书签名请求（CSR）

```
openssl req -sha512 -new -key harbor.wemeng.com.key -out harbor.wemeng.com.csr
```

3. 生成x509 v3扩展文件

```
cat > v3.ext <<-EOF
authorityKeyIdentifier=keyid,issuer
basicConstraints=CA:FALSE
keyUsage = digitalSignature, nonRepudiation, keyEncipherment, dataEncipherment
extendedKeyUsage = serverAuth
subjectAltName = @alt_names

[alt_names]
DNS.1=harbor.wemeng.com
EOF
```

4. 使用v3.ext文件生成证书

```
openssl x509 -req -sha512 -days 365 -extfile v3.ext -CA ca.crt -CAkey ca.key -CAcreateserial -in harbor.wemeng.com.csr -out harbor.wemeng.com.crt

```

#### 给Habor和Docker配置证书

1. 拷贝服务端证书和私钥到Harbot机器的证书目录

```
cp harbor.wemeng.com.crt /opt/cert/
cp harbor.wemeng.com.key /opt/cert/
```

2. 转换`harbor.wemeng.com.crt`为`harbor.wemeng.com.cert`给Docker使用

```
openssl x509 -inform PEM -in harbor.wemeng.com.crt -out harbor.wemeng.com.cert
```

3. 拷贝服务器证书，私钥和CA文件到Docker证书目录

```
cp harbor.wemeng.com.cert /etc/docker/certs.d/harbor.wemeng.com/
cp harbor.wemeng.com.key /etc/docker/certs.d/harbor.wemeng.com/
cp ca.crt /etc/docker/certs.d/harbor.wemeng.com
```

### 配置Harbor YML文件

```
#set hostname
hostname = harbor.wemeng.com
......
#The path of cert and key files for nginx, they are applied only the protocol is set to https 
ssl_cert = /opt/cert/yourdomain.com.crt
ssl_cert_key = /opt/cert/yourdomain.com.key
```

### 启动Harbor

```
$ ./install
```

### 参考资料

* [配置Harbor以支持https](https://ivanzz1001.github.io/records/post/docker/2018/04/09/docker-harbor-https)