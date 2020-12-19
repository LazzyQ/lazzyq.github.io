---
title: "Docker快速入门"
date: 2020-12-19T21:29:55+08:00
draft: false
original: true
categories: 
categories: 
  - Docker
tags: 
  - Docker
---


### part 1: 目标与快速入门

欢迎!我们很高兴你对学习Docker那么感兴趣。这篇快速入门教程包括如下内容：

1. 配置Docker环境
2. 创建一个Docker镜像并在容器中运行
3. 扩容你的应用，在多个容器中运行
4. 在集群中运行你的应用
5. 添加数据库组成服务堆栈
6. 生产环境部署你的应用

<!--more-->

#### Docker 中的概念

Docker是开发人员和系统管理员使用容器开发，部署和运行应用程序的平台。 使用Linux容器部署应用程序被称为容器化。 容器的概念并不是新的，但是将容器用来轻松的部署应用程序却是一个新颖的想法

容器化变得如此受欢迎主要因为容器：

* 灵活: 即使复杂度很高的应用同样可以容器化
* 轻量级: 容器之间共享宿主机内核
* 快速升级: 动态部署更新和升级
* 可移植: 可以本地build，部署到云上，并且可以在任何地方运行
* 可扩展: 可以增加并自动分发容器副本
* 服务堆栈: You can stack services vertically and on-the-fly.

#### 镜像和容器

通过运行一个镜像可以启动一个容器。镜像是包含运行应用程序所需环境的可执行包，运行环境包括程序代码，运行时，各种依赖库，环境变量和配置文件

容器是一个镜像的运行实例。可以通过`docker ps`查看运行中的容器

#### 容器和虚拟机

容器在Linux上本机运行，并与其他容器共享主机的内核。 它运行一个独立的进程，不占用任何其他可执行文件的内存，使其轻量级

相比之下，虚拟机（VM）运行一个完整的“客户”操作系统，通过虚拟机管理程序对主机资源进行虚拟访问。 通常，VM提供的环境比大多数应用程序需要的资源更多

![Container stack example](/Docker快速入门/Container.png)

![Virtual machine stack example](/Docker快速入门/VM.png)



#### 准备Docker环境

安装docker，这一步具体[这篇文章](xx)

##### 验证docker版本

1. 运行`docker --version`确认docker版本

```
$ docker --version
Docker version 19.03.1, build 74b1e89
```

2. 运行`docker info`或`docker version`查看更多docker版本细节

```
$ docker info
Client:
 Debug Mode: false

Server:
 Containers: 1
  Running: 0
  Paused: 0
  Stopped: 1
 Images: 10
 Server Version: 19.03.1
 Storage Driver: overlay2
  Backing Filesystem: extfs
  Supports d_type: true
  Native Overlay Diff: true
 Logging Driver: json-file
 Cgroup Driver: cgroupfs
 Plugins:
  Volume: local
  Network: bridge host ipvlan macvlan null overlay
  Log: awslogs fluentd gcplogs gelf journald json-file local logentries splunk syslog
 Swarm: inactive
 Runtimes: runc
 Default Runtime: runc
 Init Binary: docker-init
 containerd version: 894b81a4b802e4eb2a91d1ce216b8817763c29fb
 runc version: 425e105d5a03fabd737a126ad93d62a9eeede87f
 init version: fec3683
 Security Options:
  seccomp
   Profile: default
 Kernel Version: 4.9.184-linuxkit
 Operating System: Docker Desktop
 OSType: linux
 Architecture: x86_64
 CPUs: 2
 Total Memory: 1.952GiB
 Name: docker-desktop
 ID: XOGQ:6LZ4:DWI4:MH24:T7NA:MBOH:OHXR:2QYW:Q5CO:I5FR:TEVJ:Y5B7
 Docker Root Dir: /var/lib/docker
 Debug Mode: true
  File Descriptors: 32
  Goroutines: 49
  System Time: 2019-08-27T08:02:13.8093806Z
  EventsListeners: 2
 HTTP Proxy: gateway.docker.internal:3128
 HTTPS Proxy: gateway.docker.internal:3129
 Registry: https://index.docker.io/v1/
 Labels:
 Experimental: false
 Insecure Registries:
  127.0.0.0/8
 Registry Mirrors:
  http://f1361db2.m.daocloud.io/
 Live Restore Enabled: false
 Product License: Community Engine
```

为了避免权限问题(使用`sudo`运行)，需要将用户添加到`docker`用户组中。具体参考[这篇文章]()

##### 验证doker安装

1. 通过运行简单的`hello-world`镜像测试docker安装是否成功

   ```
   $ docker run hello-world
   
   Unable to find image 'hello-world:latest' locally
   latest: Pulling from library/hello-world
   [DEPRECATION NOTICE] registry v2 schema1 support will be removed in an upcoming release. Please contact admins of the docker.io registry NOW to avoid future disruption.
   1b930d010525: Pull complete
   Digest: sha256:fb158b7ad66f4d58aa66c4455858230cd2eab4cdf29b13e5c3628a6bfc2e9f05
   Status: Downloaded newer image for hello-world:latest
   
   Hello from Docker!
   This message shows that your installation appears to be working correctly.
   
   To generate this message, Docker took the following steps:
    1. The Docker client contacted the Docker daemon.
    2. The Docker daemon pulled the "hello-world" image from the Docker Hub.
       (amd64)
    3. The Docker daemon created a new container from that image which runs the
       executable that produces the output you are currently reading.
    4. The Docker daemon streamed that output to the Docker client, which sent it
       to your terminal.
   
   To try something more ambitious, you can run an Ubuntu container with:
    $ docker run -it ubuntu bash
   
   Share images, automate workflows, and more with a free Docker ID:
    https://hub.docker.com/
   
   For more examples and ideas, visit:
    https://docs.docker.com/get-started/
   ```

2. 列出机器上已经下载的`hello-world`镜像

   ```
   docker image ls或docker images
   
   REPOSITORY          TAG                            IMAGE ID            CREATED             SIZE
   choaya/into         latest                         d013706f5624        About an hour ago   123MB
   minio/minio         RELEASE.2019-08-21T19-40-07Z   7669a864ad18        5 days ago          51.6MB
   nginx               latest                         5a3221f0137b        11 days ago         126MB
   ubuntu              latest                         a2a15febcdf3        12 days ago         64.2MB
   minio/minio         latest                         77a8467ccda1        12 days ago         63.5MB
   python              3.6-alpine                     58496e6dcbca        2 weeks ago         95.1MB
   redis               alpine                         47af6bf42a1e        3 weeks ago         29.3MB
   openjdk             8-jdk-alpine                   a3562aa0b991        3 months ago        105MB
   hello-world         latest                         fce289e99eb9        7 months ago        1.84kB
   ```

3. 列出机器上的容器，`--all`或 `-a`会列出所有的容器，包括已经停止运行的容器

   ```
   $ docker container ls -a 或 $ docker container ls --all
   
   942b61d0b2ce        hello-world         "/hello"                 2 minutes ago       Exited (0) 2 minutes ago                             awesome_grothendieck
   78bc76fae770        d013706f5624        "java -Djava.securit…"   About an hour ago   Exited (143) About an hour ago                       into
   ```

#### 总结和小抄

```
## List Docker CLI commands
docker
docker container --help

## Display Docker version and info
docker --version
docker version
docker info

## Execute Docker image
docker run hello-world

## List Docker images
docker image ls

## List Docker containers (running, all, all in quiet mode)
docker container ls
docker container ls --all
docker container ls -aq
```

#### part 1 结论

容器化使得持续集成/持续部署无缝衔接，比如：

* 应用没有系统依赖
* 可以将更新推送到分布式应用程序的任何部分
* 资源密度可以优化

### Part 2: 容器

#### 前提条件

* 安装版本`1.13`以上的docker

* 完成`Part 1`部分的学习

* 下面命令能正常执行

  ```
  $ docker run hello-world
  ```

#### 简介

现在开始就可以用Docker的方式来开发一个应用程序。我们从一个应用程序最底层，也就是`container`开始。在`container`之上是`service`，这部分将这`part 3`中介绍。最后最顶层是`stack`，这部分将这`part 5`中介绍。

* Stack
* Services
* Container(现在在这)

#### 新的开发环境

过去，如果您你要开始编写Python应用程序，那么你首先就是在你的计算机上安装Python。 但是，你需要保证安装的Python和你机器能兼容而且还要和你生产环境中的环境相匹配。

使用Docker，您可以使用一个可移植的Python环境作为镜像，无需安装。 然后，你可以将基础镜像，应用程序代码，依赖项，运行时一起打包构建。

这些可移植镜像由`Dockerfile`定义。

#### 使用Dockerfile定义容器

Dockerfile定义容器内环境中发生的事情。 对网络接口和磁盘驱动器等资源的访问在此环境中进行虚拟化，该环境与系统的其他部分隔离，因此您需要将端口映射到外部世界，并具体说明要“复制”到哪些文件到这个环境。此Dockerfile中定义的应用程序的构建在其运行时表现完全一致。

**Dockerfile**

在本地机器上创建一个空的目录，并进入到这个目录，在这个目录中创建`Dockerfile`文件，拷贝下面的内容到这个文件，然后保存。`Dockerfile`中的每个语句都有说明。

```
$ mkdir app
$ cd app
$ touch Dockerfile
```



```
# 使用官方的python运行时作为基础镜像
FROM python:2.7-slim

# 设置工作目录为/app
WORKDIR /app

# 拷贝当前目录中的内容到容器中/app目录下
COPY . /app

# 按章requirements.txt文件中定义的依赖
RUN pip install --trusted-host pypi.python.org -r requirements.txt

# 暴露80端口
EXPOSE 80

# 定义环境变量
ENV NAME World

# 容器启动时运行app.py
CMD ["python", "app.py"]
```

`Dockerfile`中使用了一些还没有创建的文件`app.py`和`requirements.txt`。下面我们来创建这些文件。

#### 应用本身

在`Dockerfile`同目录下创建`app.py`和`requirements.txt`文件。

```shell
touch app.py requirements.txt
```

**requirements.txt**

```
Flask
Redis
```

**app.py**

```python
from flask import Flask
from redis import Redis, RedisError
import os
import socket

# Connect to Redis
redis = Redis(host="redis", db=0, socket_connect_timeout=2, socket_timeout=2)

app = Flask(__name__)

@app.route("/")
def hello():
    try:
        visits = redis.incr("counter")
    except RedisError:
        visits = "<i>cannot connect to Redis, counter disabled</i>"

    html = "<h3>Hello {name}!</h3>" \
           "<b>Hostname:</b> {hostname}<br/>" \
           "<b>Visits:</b> {visits}"
    return html.format(name=os.getenv("NAME", "world"), hostname=socket.gethostname(), visits=visits)

if __name__ == "__main__":
    app.run(host='0.0.0.0', port=80)
```

目前我们通过`pip install -r requirements.txt`安装了`Flask`和`Redis`的pthon库，应用程序打印了环境变量`NAME`和`socket.gethostname()`，最后，由于Redis并没有运行，这里应该会打印错误信息。

> 在容器内获取hostname，其实是container ID，类似于 process ID

OK！你并不需要安装Python或者`requirements.txt`中定义的库文件在你的系统，构建或运行这个镜像也不需要在系统上安装它们。 你似乎并没有真正设置Python和Flask的环境，但实际上你已经有了。

#### 构建应用

准备构建这个应用。在此之前，请先确保你在刚才建立的目录下，`ls`列出这个目录下的内容:

```
$ ls
Dockerfile       app.py           requirements.txt
```

现在运行构建命令。这条命令将会创建一个Docker镜像，可以通过`--tag`或`-t`选项来制定镜像名称

```
docker build --tag=friendlyhello .
```

构建的镜像在哪？已经在你的本地机器上了

```
$ docker images
REPOSITORY          TAG                            IMAGE ID            CREATED             SIZE
friendlyhello       latest                         996ecb8e1951        30 seconds ago      148MB
choaya/into         latest                         d013706f5624        2 hours ago         123MB
python              2.7-slim                       6b34ff151910        9 hours ago         137MB
minio/minio         RELEASE.2019-08-21T19-40-07Z   7669a864ad18        5 days ago          51.6MB
nginx               latest                         5a3221f0137b        11 days ago         126MB
ubuntu              latest                         a2a15febcdf3        12 days ago         64.2MB
minio/minio         latest                         77a8467ccda1        12 days ago         63.5MB
python              3.6-alpine                     58496e6dcbca        2 weeks ago         95.1MB
redis               alpine                         47af6bf42a1e        3 weeks ago         29.3MB
openjdk             8-jdk-alpine                   a3562aa0b991        3 months ago        105MB
hello-world         latest                         fce289e99eb9        7 months ago        1.84kB
```

注意tag默认是`latest`。完整的tag语法应该像这个样子`--tag=friendlyhello:v0.0.1`

#### 运行应用程序

运行应用程序，使用`-p`将机器的4000端口映射到容器的80端口

```
docker run -p 4000:80 friendlyhello
```

在运行的日志上，你可以看到你的服务运行在`http://0.0.0.0:80/`，但是这写消息来自于容器内部，容器并不知道你将容器的80端口映射到了4000端口，所以正确的URL应该是`http://localhost:4000`

在浏览器中输入URL，查看页面内容

![应用运行显示内容](/Docker快速入门/image-20190827171826800.png)

同样可以使用`curl`命令得到同样的内容

```
$ curl localhost:4000

<h3>Hello World!</h3><b>Hostname:</b> 14cf737a0cf5<br/><b>Visits:</b> <i>cannot connect to Redis, counter disabled</i>
```

端口重映射`4000:80`演示了 `Dockerfile` 中 `EXPOSE`和`docker run -p`运行的区别。后者将宿主机`4000`端口映射到容器的`80`端口，在容器内部可以使用`http://localhost`来访问。

```
$ docker ps
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS                          NAMES
4f28a2e7c9e6        friendlyhello       "python app.py"     2 minutes ago       Up 2 minutes        80/tcp, 0.0.0.0:80->4000/tcp   pedantic_hermann

$ docker exec -it pedantic_hermann /bin/bash
$ root@4f28a2e7c9e6:/app# curl localhost
<h3>Hello World!</h3><b>Hostname:</b> 14cf737a0cf5<br/><b>Visits:</b> <i>cannot connect to Redis, counter disabled</i>
```

> 注意这里使用docker exec -it pedantic_hermann /bin/bash 进入了容器，由于基础镜像中并没有安装curl，所以需要先通过apt安装curl才能使用。
>
> $ apt update
>
> $ apt-get install curl
>
> 这里只是为了验证内部容器可以通过80端口进行访问，尽量不要在容器内安装多余的内容，这会增加容器的大小

`ctrl+c  `停止运行。

在后台运行应用程序，使用detached模式

```
docker run -d -p 4000:80 friendlyhello
```

执行完后你会得到一个长长的 container ID，然后马上退出返回到命令行。这样容器就会一直在后台运行。可以通过`docker container ls`命令查看到简短的container ID。

```
$ docker container ls

CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS                  NAMES
cf8a2c2a348b        friendlyhello       "python app.py"     4 minutes ago       Up 4 minutes        0.0.0.0:4000->80/tcp   mystifying_brown
```

注意`CONTAINER ID`和`http://localhost:4000`页面上的hostname是匹配的

现在使用`docker container stop`结束容器运行，将`CONTAINER ID`作为参数

```
docker container stop cf8a2c2a348b
```

#### 共享镜像

为了演示刚才关键镜像的便携性，我们需要上传我们刚才的镜像，然后尝试在任何地方运行它。毕竟当你在线上部署容器时，你需要知道怎么push镜像到registries。

registries时一些仓库的集合，仓库时镜像的集合。这个有点像Github仓库。

##### 使用Docker ID登陆

如果没有Docker账户，请在[hub.docker.com](https://hub.docker.com/)上注册一个。注意你的用户名。

在你机器上登陆公共的registry

```
docker login
```

##### 给镜像打标签

通过 `username/repository:tag`可以将本地镜像和registry的一个仓库关联。`tag`是可选的，但是推荐填上。比如`get-started:part2`，这就是将镜像放在`get-started`仓库，`tag`是`part2`

现在运行`docker tag image`传入username，repository和tag

```
docker tag image username/repository:tag
```

比如

```
docker tag friendlyhello choaya/get-started:part2
```

运行`docker images`查看这个新的镜像

```
$ docker images
REPOSITORY           TAG                            IMAGE ID            CREATED             SIZE
choaya/get-started   part2                          996ecb8e1951        About an hour ago   148MB
friendlyhello        latest                         996ecb8e1951        About an hour ago   148MB
choaya/into          latest                         d013706f5624        3 hours ago         123MB
python               2.7-slim                       6b34ff151910        10 hours ago        137MB
minio/minio          RELEASE.2019-08-21T19-40-07Z   7669a864ad18        5 days ago          51.6MB
nginx                latest                         5a3221f0137b        11 days ago         126MB
ubuntu               latest                         a2a15febcdf3        12 days ago         64.2MB
minio/minio          latest                         77a8467ccda1        12 days ago         63.5MB
python               3.6-alpine                     58496e6dcbca        2 weeks ago         95.1MB
redis                alpine                         47af6bf42a1e        3 weeks ago         29.3MB
openjdk              8-jdk-alpine                   a3562aa0b991        3 months ago        105MB
hello-world          latest                         fce289e99eb9        7 months ago        1.84kB
```

##### 发布镜像

上传刚才的镜像到仓库

```
docker push choaya/get-started:part2
```

完成之后，你可以登陆到[Docker Hub](https://hub.docker.com/)，可以看到这个镜像。

##### pull同时运行远程仓库的镜像

从现在开始，你可以用`docker run`直接在你的机器上运行你的应用程序。

```
docker run -p 4000:80 choaya/get-started:part2
```

如果本地没有这个镜像，docker自动会从远程仓库中下载这个镜像到本地机器

由于我们本地已经有这个镜像，为了演示便携性，我先把本地的镜像删除

```
$ docker images
REPOSITORY           TAG                            IMAGE ID            CREATED             SIZE
choaya/get-started   part2                          996ecb8e1951        About an hour ago   148MB
friendlyhello        latest                         996ecb8e1951        About an hour ago   148MB
choaya/into          latest                         d013706f5624        3 hours ago         123MB
python               2.7-slim                       6b34ff151910        11 hours ago        137MB
minio/minio          RELEASE.2019-08-21T19-40-07Z   7669a864ad18        5 days ago          51.6MB
nginx                latest                         5a3221f0137b        11 days ago         126MB
ubuntu               latest                         a2a15febcdf3        12 days ago         64.2MB
minio/minio          latest                         77a8467ccda1        12 days ago         63.5MB
python               3.6-alpine                     58496e6dcbca        2 weeks ago         95.1MB
redis                alpine                         47af6bf42a1e        3 weeks ago         29.3MB
openjdk              8-jdk-alpine                   a3562aa0b991        3 months ago        105MB
hello-world          latest                         fce289e99eb9        7 months ago        1.84kB

$ docker container prune

$ docker image rm -f 996ecb8e1951

$ docker images
REPOSITORY          TAG                            IMAGE ID            CREATED             SIZE
choaya/into         latest                         d013706f5624        3 hours ago         123MB
python              2.7-slim                       6b34ff151910        11 hours ago        137MB
minio/minio         RELEASE.2019-08-21T19-40-07Z   7669a864ad18        5 days ago          51.6MB
nginx               latest                         5a3221f0137b        11 days ago         126MB
ubuntu              latest                         a2a15febcdf3        12 days ago         64.2MB
minio/minio         latest                         77a8467ccda1        12 days ago         63.5MB
python              3.6-alpine                     58496e6dcbca        2 weeks ago         95.1MB
redis               alpine                         47af6bf42a1e        3 weeks ago         29.3MB
openjdk             8-jdk-alpine                   a3562aa0b991        3 months ago        105MB
hello-world         latest                         fce289e99eb9        7 months ago        1.84kB
```

可以看到现在我本地机器上已经没有了刚才创建的镜像，现在直接从远程仓库运行应用程序试试。

```
$ docker run -p 4000:80 choaya/get-started:part2

Unable to find image 'choaya/get-started:part2' locally
part2: Pulling from choaya/get-started
[DEPRECATION NOTICE] registry v2 schema1 support will be removed in an upcoming release. Please contact admins of the docker.io registry NOW to avoid future disruption.
1ab2bdfe9778: Already exists
f9047cc61c29: Already exists
7f58e6c9f6f0: Already exists
7058c90ade99: Already exists
32c0ef8226c3: Pull complete
c67c640dc154: Pull complete
13606c24f595: Pull complete
Digest: sha256:02e4a83a136002050374b2b5d3b15683ff2134ed752bb8435f1f770ecbcd57d1
Status: Downloaded newer image for choaya/get-started:part2
 * Serving Flask app "app" (lazy loading)
 * Environment: production
   WARNING: This is a development server. Do not use it in a production deployment.
   Use a production WSGI server instead.
 * Debug mode: off
 * Running on http://0.0.0.0:80/ (Press CTRL+C to quit)
```

可以看到从远程仓库拉取了镜像，并运行了一个容器，在本地通过`http://localhost:4000`可以正常访问

如论在什么地方执行`docker run`，它都会pull你的镜像，同时Python和`requirements.txt`定义的依赖和你的应用程序代码都会一并打包。你不需要在你机器上安装各种环境。

#### Part 2结论

这部分内容到这就结束了，下一节我们将学习扩容我们应用，我们将以`service`来运行这个`container`

这里是这一节所用到的命令

```
docker build -t friendlyhello .  # Create image using this directory's Dockerfile
docker run -p 4000:80 friendlyhello  # Run "friendlyhello" mapping port 4000 to 80
docker run -d -p 4000:80 friendlyhello         # Same thing, but in detached mode
docker container ls                                # List all running containers
docker container ls -a             # List all containers, even those not running
docker container stop <hash>           # Gracefully stop the specified container
docker container kill <hash>         # Force shutdown of the specified container
docker container rm <hash>        # Remove specified container from this machine
docker container rm $(docker container ls -a -q)         # Remove all containers
docker image ls -a                             # List all images on this machine
docker image rm <image id>            # Remove specified image from this machine
docker image rm $(docker image ls -a -q)   # Remove all images from this machine
docker login             # Log in this CLI session using your Docker credentials
docker tag <image> username/repository:tag  # Tag <image> for upload to registry
docker push username/repository:tag            # Upload tagged image to registry
docker run username/repository:tag                   # Run image from a registry
```

### Part 3: Services

#### 前提条件

* 安装版本`1.13`以上的docker
* 安装`Docker Compose`。Mac OS版和Windows版的Docker已经预装了。在Linux上，你需要[安装](https://github.com/docker/compose/releases)
* 学习完Part 1
* 学习完Part 2
* 确保Part 2中推向远程仓库的镜像能正常运行，这一节我们将会使用这个镜像

#### 简介

在part 3中我们会扩容我们的应用同时启用负载均衡。为了实现这个目标，我们将进入分布式应用的container层级升级到service层

* Stack
* Services(现在在这)
* Container(在part 2中介绍)

#### 关于services

在分布式应用中，应用的不同部分称为"services"。比如，一个视频分享网站可能包括一个存储应用数据到数据库的服务，一个视频转码服务，一个为前端提供数据的服务等等

`services`实际上只是“生产中的容器”。服务只运行一个镜像，但它规定了镜像的运行方式，它应该使用哪些端口，应该运行多少个容器副本，以便`service`具有所需的容量等等。 扩容服务会更改运行该应用的容器实例的数量，从而为应用中的服务分配更多计算资源。

#### 第一个docker-compose.yml文件

`docker-compose.yml`文件是一个YAML文件，它定义了Docker容器应该怎么运行。

**docker-compose.yml**

将下面的内容保存到docker-compose.yml文件。确保你已经将Part 2中的镜像推到了远程仓库，然后将文件中的`username/repo:tag`换成你自己远程仓库镜像

```yaml
version: "3"
services:
  web:
    # replace username/repo:tag with your name and image details
    image: username/repo:tag
    deploy:
      replicas: 5
      resources:
        limits:
          cpus: "0.1"
          memory: 50M
      restart_policy:
        condition: on-failure
    ports:
      - "4000:80"
    networks:
      - webnet
networks:
  webnet:
```

这个`docker-compose.yml`告诉Docker做了下面这些事：

* 从远程仓库pull part 2中我们上传的镜像
* 运行5个镜像的实例作为`web`service，限制每个实例最多只能使用单个CPU核心的10%(1.5表示每个实例最多1.5个核心)，最多50M的内存
* 如果运行失败立即重启
* 映射本地4000端口到`web`的80端口
* Instruct `web`’s containers to share port 80 via a load-balanced network called `webnet`. (Internally, the containers themselves publish to `web`’s port 80 at an ephemeral port.)
* 使用默认设置定义一个名为`webnet`的网络(默认配置是一个负载均衡的overlay网络)

#### 运行新的负载均衡的应用

在使用`docker stack deploy`命令之前，需要先运行

```
docker swarm init
```

> 注意：这条命令的含义在part 4中介绍。如果不运行`dicker swarm init`会报`this node is not a swarm manager`错误

现在就可以运行了，你需要给你的应用一个名字，这里命名为`getstartedlab`

```
$ docker stack deploy -c docker-compose.yml getstartedlab
```

我们在一个宿主机上运行了5个容器实例，每个容器中是我们部署的镜像。下面我们来探索探索

查看我们应用程序`web`service的service ID

```
$ docker service ls
```

查看上面这条命令的输出，以应用名字为前缀。如果你把应用命名同上面一样，那么这里显示的名字就是`getstartedlab_web`。同样输出中列出了service ID，副本数量，镜像名称和暴露的端口

同样，你能运行`docker stack services`，以stack的名字作为参数。以下示例命令允许您查看与`getstartedlab`stack关联的所有服务：

```
$ docker stack services getstartedlab

ID                  NAME                MODE                REPLICAS            IMAGE                      PORTS
k0eqbpwwjbn7        getstartedlab_web   replicated          5/5                 choaya/get-started:part2   *:4000->80/tcp
```

`service`中运行的单个容器称为`task`。每个`task`都有一个唯一ID，最多是`docker-compose.yml`中定义的`replicas`数量。列出你的`service`的`Tasks`

```
$ docker service ps getstartedlab_web

ID                  NAME                  IMAGE                      NODE                DESIRED STATE       CURRENT STATE            ERROR               PORTS
t9f3n9ehci40        getstartedlab_web.1   choaya/get-started:part2   lazzyq-HP           Running             Running 29 minutes ago
ugf8z6qymwcl        getstartedlab_web.2   choaya/get-started:part2   lazzyq-HP           Running             Running 29 minutes ago
m65zkmcnh9uk        getstartedlab_web.3   choaya/get-started:part2   lazzyq-HP           Running             Running 29 minutes ago
a6kl9zalaxc1        getstartedlab_web.4   choaya/get-started:part2   lazzyq-HP           Running             Running 29 minutes ago
cpb6ynlm7680        getstartedlab_web.5   choaya/get-started:part2   lazzyq-HP           Running             Running 29 minutes ago
```

你可以运行在浏览器中输入`http://localhost:4000`，刷新几次观察每次的hostname和容器ID的关系

可以发现，每次刷新，容器ID都会改变一次，这演示了负载均衡的功能。每次请求都会选一个`task`来响应。使用`round-robin`的方式进行响应。容器ID和`docker container ls -q`中的容器ID对应

#### 扩容你的应用

你可以通过改变`docker-compose.yml`文件中`replicas`来实现扩容。保存修改之后，重新运行`docker stack deploy`命令

```
docker stack deploy -c docker-compose.yml getstartedlab
```

Docker就地运行，不需要先停止`stack`和停止任何容器就能实现扩容

现在重新运行`docker container ls -q`查看重新部署的实例。如何扩大副本，则会启动更多的`tasks`，运行更多的容器

#### 停止应用和swarm

* 使用`docker stack rm`命令停止应用

  ```
  docker stack rm getstartedlab
  ```

* 停止swarm

  ```
  docker swarm leave --force
  ```

使用Docker启动应用和扩容就像这么简单。你已经向生产环境运行容器迈了一大步。接下来，你将会学习如何在Docker machines集群上运行真正的swarm应用。

#### 总结和小抄

回顾一下，当运行`docker run`很简单，在生产环境中真正的实现是将容器当作service来运行。在Compose文件中，Services定义了容器的行为，这个文件能用来扩容，限制资源和重新部署应用。service的改变能就地运行，使用`docker stack deploy`命令来部署service

这一节使用到的命令

```
docker stack ls                                            # List stacks or apps
docker stack deploy -c <composefile> <appname>  # Run the specified Compose file
docker service ls                 # List running services associated with an app
docker service ps <service>                  # List tasks associated with an app
docker inspect <task or container>                   # Inspect task or container
docker container ls -q                                      # List container IDs
docker stack rm <appname>                             # Tear down an application
docker swarm leave --force      # Take down a single node swarm from the manager
```

### Part 4: Swarms

#### 前提条件

- 安装版本`1.13`以上的docker
- 安装[Docker Compose](https://docs.docker.com/compose/overview/)
- 安装[Docker Machine](https://docs.docker.com/machine/overview/)，Mac OS版和Windows版的Docker已经预装了。在Linux上需要[安装](https://docs.docker.com/machine/install-machine/#installing-machine-directly)
- 学习完成part 1
- 学习完成part 2
- 确保Part 2中推向远程仓库的镜像能正常运行，这一节我们将会使用这个镜像
- 一份part 3中的`docker-compose.yml`文件的备份

#### 简介

在part 3中，我们使用part 2中构建的镜像，将其以service的方式，扩容到5个实例

在part 4中，我们将部署应用到集群中，并且在多台机器上运行。多容器，多机应用实现基于被称为**swarm**的`Docker化`的集群

#### 理解Swarm集群

Swarm是一组运行Docker的机器集群。在Swarm集群中像原来那样执行Docker命令，现在这些命令被称为`swarm manager`执行。swarm集群中的机器可以是物理机或者虚拟机。这些机器加入swarm后，这些机器被称为**nodes**

Swarm managers可以使用使用多种策略来运行容器，例如`emptiest node`--它使用容器填充利用率最低的机器。或者`global`--它确保每台机器只获得指定容器的一个实例。你可以在Compose文件中指定swarm manager使用那种策略

到现在为止，我们一直在本地机器上使用单主机模式的Docker。但是Docker也能切换到**swarm mode**，这就是使用swarm的原因。启用**swarm mode**使当前计算机成为swarm manager。 启用后Docker就会运行你在管理的swarm上执行的命令，而不仅仅是在当前机器上

#### 启动swarm

swarm集群由多个节点组成，这些节点可以是物理机也可以是虚拟机。基本概念很简单：运行`docker swarm init`启用swarm模式同时使当前机器作为swarm manager，然后在其他机器上运行`docker swarm join`加入swarm作为worker节点。接下来我们使用VM快速创建2台机器的swarm集群

#### 创建集群

你需要一个创建VM的hypervisor，所以在机器上需要[安装Oracle VirtualBox](https://www.virtualbox.org/wiki/Downloads)

现在使用`docker-machine`，`VirtualBox`创建几个VM

```
docker-machine create --driver virtualbox myvm1
docker-machine create --driver virtualbox myvm2
```

#### 列出VM获取它们的IP

现在创建了2个VM，分别为`myvm1`和`myvm2`

使用下面的命令列出VM获取它们的IP

```
$ docker-machine ls

NAME    ACTIVE   DRIVER       STATE     URL                         SWARM   DOCKER     ERRORS
myvm1   -        virtualbox   Running   tcp://192.168.99.100:2376           v19.03.1
myvm2   -        virtualbox   Running   tcp://192.168.99.101:2376           v19.03.1
```

#### 初始化Swarm和添加Nodes

第一台机器充当manager，执行管理命令和对wokers加入到swarm进行授权，第二台机器充当worker

你可以使用`docker-machine ssh`向VM发送命令。使用`docker swarm init`命令指定`myvm1`作为swarm manager。

```
$ docker-machine ssh myvm1 "docker swarm init --advertise-addr <myvm1 ip>:<port>"
Swarm initialized: current node <node ID> is now a manager.

To add a worker to this swarm, run the following command:

  docker swarm join \
  --token <token> \
  <myvm ip>:<port>

To add a manager to this swarm, run 'docker swarm join-token manager' and follow the instructions.
```

> 端口 2377和2376
>
> `docker warm init`和`docker swarm join`尽量使用2377这个端口(swarm 管理端口)，或者不指定端口将默认端口2377
>
>
> `docker-machine ls`命令返回的机器IP中包含2376端口，这个端口是docker的后台端口。不要使用这个端口，不然会有 [报错](https://forums.docker.com/t/docker-swarm-join-with-virtualbox-connection-error-13-bad-certificate/31392/2)

`docker swarm init`命令执行后的结果中可以看到添加node的`docker swarm join`命令格式。使用`docker-machine ssh`命令将`docker swarm join`命令发送给`myvm2`，让`myvm2`作为worker加入到swarm

```
$ docker-machine ssh myvm2 "docker swarm join \
--token <token> \
<ip>:2377"

This node joined a swarm as a worker.
```

恭喜你，你已经创建了你的第一个swarm集群

在manager上运行`docker node ls`查看swarm集群中的节点

```
$ docker-machine ssh myvm1 "docker node ls"

ID                            HOSTNAME            STATUS              AVAILABILITY        MANAGER STATUS
brtu9urxwfd5j0zrmkubhpkbd     myvm2               Ready               Active
rihwohkh3ph38fhillhhb84sk *   myvm1               Ready               Active              Leader
```

> **离开swarm**
>
> 如果你想重新开始，在每个节点上运行`docker swarm leave`

#### 在swarm集群上部署应用

最难的部分已经完了。现在你可以重复part 3的过程部署应用到新的swarm集群上。注意只有swarm manager比如`myvm1`执行docker命令。workers仅仅作为机器资源

#### 配置swarm manager的docker-machine shell命令

目前为止，执行命令都是以`docker-machine ssh`命令将命令发送到机器上执行。可以使用`docker-machine env <machine>`命令配置你当前的shell命令和VM的Docker daemon通信。这个方法可以让你使用本地的`docker-compose.yml`文件部署应用程序到''远程''机器，不用将这个文件拷来拷去

运行`docker-machine env myvm1`，然后命令结果运行的最后一行拷贝出来并运行，配置你当前的shell命令和swarm manager `myvm1`通信

#### Mac或Linux上配置docker-machine shell环境

运行`docker-machine env myvm1`得到配置与`myvm1`通信的命令

```
$ docker-machine env myvm1

export DOCKER_TLS_VERIFY="1"
export DOCKER_HOST="tcp://192.168.99.100:2376"
export DOCKER_CERT_PATH="/Users/sam/.docker/machine/machines/myvm1"
export DOCKER_MACHINE_NAME="myvm1"
# Run this command to configure your shell:
# eval $(docker-machine env myvm1)
```

运行这个命令

```
eval $(docker-machine env myvm1)
```

运行`docker-machine ls`验证`myvm1`现在是激活的机器，`myvm1`被*标识

```
$ docker-machine ls

NAME    ACTIVE   DRIVER       STATE     URL                         SWARM   DOCKER     ERRORS
myvm1   *        virtualbox   Running   tcp://192.168.99.100:2376           v19.03.1
myvm2   -        virtualbox   Running   tcp://192.168.99.101:2376           v19.03.1
```

#### 在swarm manager上部署应用

现在你可以使用在part 3中使用的`docker stack deploy`命令和`docker-compose.yml`文件的备份来部署你的应用。这个命令会花一些时间完成部署，并且需要一段时间才会可用。使用在swarm manager运行`docker service ps <service_name>`验证所有服务重新部署

通过配置docker-machine shell环境与`myvm1`连接，你仍然可以访问本地机器的文件。确保你在之前目录下，目录下有part 3中的`docker-compose.yml`文件

像之前那样，在`myvm1`上运行下面的命令

```
docker stack deploy -c docker-compose.yml getstartedlab
```

就这样，应用已经部署到了swarm集群

> 注意：如果你的镜像不是在Docker Hub上，而是在私有仓库，你需要使用`docker login <your-registry>`登陆，然后添加`--with-registry-auth`到上面的命令，比如：
>
> ```
> docker login registry.example.com
> 
> docker stack deploy --with-registry-auth -c docker-compose.yml getstartedlab
> ```
>
> 上面的命令可以使swarm节点可以从你的私有仓库中pull镜像

现在你可以使用part 3中的[命令](https://docs.docker.com/get-started/part3/#run-your-new-load-balanced-app)。只不过这次服务分别部署到`myvm1`和`myvm2`上

> * 使用`docker-machine env`和`docker-machain ssh`连接到VM。如果想将将shell连接到`myvm2`，在当前shell或者新的shell中再次运行`docker-machine env`，然后再执行`docker-machine env`命令后返回的命令。这条命令只是配置了当前shell。如果你重新开了一个shell窗口，需要重新运行命令进行配置。使用`docker-machine ls`列出机器，看看这些机器处于什么状态，ip地址，确认当前连接到的是哪台机器
>
> * 另外，你可以用`docker-machine ssh <machine> "<command>"`命令来达到同样的目的。这条命令运行的log日志直接打印到VM，不会在本地机器上访问文件
> * 在MacOS和Linux上，你可以使用`docker-machine scp <file> <machine>:~`拷贝文件到机器上

#### 访问集群

你可以使用`myvm1`或`myvm2`的IP地址来访问你的应用

创建网络是共享的，同时实现了负载均衡。运行`docker-machine ls`获取VM的IP地址，可以通过浏览器或`curl`访问4000端口

访问随机得到的结果有5个不同的容器ID，体现了负载均衡

2个机器的IP都访问，是因为在swarm节点加入到路由网格(routing mesh)中。这样确保了在swarm中部署的service无论使用什么端口都得以保留，无论在哪个node上运行的容器都这样。下面这个图展示了一个名为`my-web`服务部署在拥有3个node的swarm中并对外暴露8080端口的结构

![routing mesh diagram](/Docker快速入门/ingress-routing-mesh.png)

#### 迭代和扩容应用

从现在开始，你可以运行你在part 2和part 3中学到的命令

可以更改`docker-compose.yml`进行扩容

修改代码，重新构建镜像，将镜像推送到仓库进行迭代

无论是扩容还是迭代，仅仅再次运行`docker stack deploy`就可以应用这些改变

你可以使用`docker swarm join`命令将任何机器(物理机或虚拟机)加入到swarm中，集群就相当于扩容了。运行`docker stack deploy`，应用就能使用这些资源重新部署

#### 清理和重启

##### stack和swarm

你可以使用`docker stack rm`命令停止stack，例如

```
docker stack rm getstartedlab
```

> 保留swarm还是移除它？
>
> 可以使用`docker-machine ssh myvm2 "docker swarm leave"`让worker从swarm中移除，使用`docker-machine ssh myvm1 "docker swarm leave --force"`让manager从swarm中移除，但是在part 5中还会使用到这个swarm，现在还是留着

##### 取消docker-machine shell命令

你可以使用下面的命令来取消上面配置的docker-machine shell环境

在`Mac或Linux`上使用

```
eval $(docker-machine env -u)
```

在`Windows`上使用

```
& "C:\Program Files\Docker\Docker\Resources\bin\docker-machine.exe" env -u | Invoke-Expression
```

#### 重启Docker machine

如果你把宿主机关机了，Docker machine将停止运行。可以使用`docker-machine ls`查看机器的运行状态

```
$ docker-machine ls
NAME    ACTIVE   DRIVER       STATE     URL   SWARM   DOCKER    ERRORS
myvm1   -        virtualbox   Stopped                 Unknown
myvm2   -        virtualbox   Stopped                 Unknown
```

重启可以使用

```
docker-machine start <machine-name>
```

比如

```
$ docker-machine start myvm1
Starting "myvm1"...
(myvm1) Check network to re-create if needed...
(myvm1) Waiting for an IP...
Machine "myvm1" was started.
Waiting for SSH to be available...
Detecting the provisioner...
Started machines may have new IP addresses. You may need to re-run the `docker-machine env` command.

$ docker-machine start myvm2
Starting "myvm2"...
(myvm2) Check network to re-create if needed...
(myvm2) Waiting for an IP...
Machine "myvm2" was started.
Waiting for SSH to be available...
Detecting the provisioner...
Started machines may have new IP addresses. You may need to re-run the `docker-machine env` command.
```

#### 总结和小抄

在part 4中，你了解了swarm，在swarm中节点怎么成为manager或worker，怎么常见swarm并部署应用到swarm。你可以看到，在part 3中的命令并没有改变，这些命令只是在swarm的manager节点上运行。你也看到了docker中网络的功能，能够在容器之间进行负载均衡，即使容器运行在不同机器上。最后你知道了怎么在集群上迭代和扩容你的应用。

这一节使用到的命令

```
docker-machine create --driver virtualbox myvm1 # Create a VM (Mac, Win7, Linux)
docker-machine create -d hyperv --hyperv-virtual-switch "myswitch" myvm1 # Win10
docker-machine env myvm1                # View basic information about your node
docker-machine ssh myvm1 "docker node ls"         # List the nodes in your swarm
docker-machine ssh myvm1 "docker node inspect <node ID>"        # Inspect a node
docker-machine ssh myvm1 "docker swarm join-token -q worker"   # View join token
docker-machine ssh myvm1   # Open an SSH session with the VM; type "exit" to end
docker node ls                # View nodes in swarm (while logged on to manager)
docker-machine ssh myvm2 "docker swarm leave"  # Make the worker leave the swarm
docker-machine ssh myvm1 "docker swarm leave -f" # Make master leave, kill swarm
docker-machine ls # list VMs, asterisk shows which VM this shell is talking to
docker-machine start myvm1            # Start a VM that is currently not running
docker-machine env myvm1      # show environment variables and command for myvm1
eval $(docker-machine env myvm1)         # Mac command to connect shell to myvm1
& "C:\Program Files\Docker\Docker\Resources\bin\docker-machine.exe" env myvm1 | Invoke-Expression   # Windows command to connect shell to myvm1
docker stack deploy -c <file> <app>  # Deploy an app; command shell must be set to talk to manager (myvm1), uses local Compose file
docker-machine scp docker-compose.yml myvm1:~ # Copy file to node's home dir (only required if you use ssh to connect to manager and deploy the app)
docker-machine ssh myvm1 "docker stack deploy -c <file> <app>"   # Deploy an app using ssh (you must have first copied the Compose file to myvm1)
eval $(docker-machine env -u)     # Disconnect shell from VMs, use native docker
docker-machine stop $(docker-machine ls -q)               # Stop all running VMs
docker-machine rm $(docker-machine ls -q) # Delete all VMs and their disk images
```

### Part 5: Stack

#### 前提条件

- 安装版本`1.13`以上的docker
- 安装[Docker Compose](https://docs.docker.com/compose/overview/)
- 安装[Docker Machine](https://docs.docker.com/machine/overview/)，Mac OS版和Windows版的Docker已经预装了。在Linux上需要[安装](https://docs.docker.com/machine/install-machine/#installing-machine-directly)
- 学习完成part 1
- 学习完成part 2
- 确保Part 2中推向远程仓库的镜像能正常运行，这一节我们将会使用这个镜像
- 一份part 3中的`docker-compose.yml`文件的备份
- 确保part 4中国创建的机器正常运行。运行`docker-machine ls`验证。如果机器停止了，运行`docker-machine start myvm1`启用manager，`docker-machine start myvm2`启动worker
- 确保part 4中swarm正常运行。运行`docker-machine ssh myvm1 "docker node ls"`命令验证。如果swarm正常运行，节点应该是`ready`状态，如果不是，重新初始化swarm

####简介

在part 4中，你已经学会了启用swarm集群并且部署应用在上面

在part 5这部分，你将接触到分布式应用的最高层:stack。stack就是一组相关联、共享依赖，能同时扩容的services，一个stack就能实现整个应用(尽管复杂的应用需要使用多个stack)

好消息是，当你在part 3中创建compose文件并且使用`docker stack deploy`时，就已经使用过stack了。但是这只是单个service的的stack运行在单个host上，但生产环境并不会这么使用。那么现在你将学习多个相关联的services部署在多台机器上

#### 添加一个新service并重新部署

在`docker-compose.yml`中添加服务很简单。首先我们先添加一个免费的可视化服务，这个可视化服务可以帮组我们看到swarm怎么调度容器

1. 使用下面的内容替换原来的`docker-compose.yml`文件，注意将`username/repo:tag`替换成你自己的镜像

   ```
   version: "3"
   services:
     web:
       # replace username/repo:tag with your name and image details
       image: username/repo:tag
       deploy:
         replicas: 5
         restart_policy:
           condition: on-failure
         resources:
           limits:
             cpus: "0.1"
             memory: 50M
       ports:
         - "80:80"
       networks:
         - webnet
     visualizer:
       image: dockersamples/visualizer:stable
       ports:
         - "8080:8080"
       volumes:
         - "/var/run/docker.sock:/var/run/docker.sock"
       deploy:
         placement:
           constraints: [node.role == manager]
       networks:
         - webnet
   networks:
     webnet:
   ```

   唯一的改变就是添加了一个名为`visualizer`的service。注意这里添加了2个新的概念，`volumns`允许`visualizer`能够访问宿主机docker的socket文件；`placement`确保`visualizer`只运行在swarm manager，而不是在worker。

2. 确保你的shell配置成连接到`myvm1`了

   * 运行`docker-machine ls`列出机器确保shell已经配置连接到`myvm1`，如果配置成功，可以在`myvm1`看到*标识

   * 如果可能的话，重新运行`docker-machine env myvm1`，然后运行下列命令配置

     在MacOS或Linux上

     ```
     eval $(docker-machine env myvm1)
     ```

     在Windows上

     ```
     & "C:\Program Files\Docker\Docker\Resources\bin\docker-machine.exe" env myvm1 | Invoke-Expression
     ```

3. 在manager上重新运行`docker stack deploy`命令

   ```
   $ docker stack deploy -c docker-compose.yml getstartedlab
   Updating service getstartedlab_web (id: angi1bf5e4to03qu9f93trnxm)
   Creating service getstartedlab_visualizer (id: l9mnwkeq2jiononb5ihz9u7a4)
   ```

4. 看一眼visualizer

   你可以看到，在compose文件中`visualizer`运行在8080端口。通过`docker-machine ls`命令得到的IP地址来访问

   ![Visualizer screenshot](/Docker快速入门/get-started-visualizer1.png)

   可以看到单个`visualizer`运行在manager上，5个`web`实例分别在swarm各个机器上运行。你能通过`docker stack ps <stack>`来确认：

   ```
   docker stack ps getstartedlab
   ```

#### 持久化数据

接下来我们添加一个redis数据出来存储应用数据

1. 保存新的`docker-machine.yml`文件。只是在前面的基础上添加了Redis service

   ```yaml
   version: "3"
   services:
     web:
       # replace username/repo:tag with your name and image details
       image: username/repo:tag
       deploy:
         replicas: 5
         restart_policy:
           condition: on-failure
         resources:
           limits:
             cpus: "0.1"
             memory: 50M
       ports:
         - "80:80"
       networks:
         - webnet
     visualizer:
       image: dockersamples/visualizer:stable
       ports:
         - "8080:8080"
       volumes:
         - "/var/run/docker.sock:/var/run/docker.sock"
       deploy:
         placement:
           constraints: [node.role == manager]
       networks:
         - webnet
     redis:
       image: redis
       ports:
         - "6379:6379"
       volumes:
         - "/home/docker/data:/data"
       deploy:
         placement:
           constraints: [node.role == manager]
       command: redis-server --appendonly yes
       networks:
         - webnet
   networks:
     webnet:
   ```

   Redis配置使用6379端口，在compose文件中，将6379端口从宿主机暴露，所以你可以通过访问Redis。

   最重要的是，在部署之前，有几件事关于redis持久化的事需要说明

   * redis运行在manager
   * redis存储数据的目录映射到了宿主机的/data目录

   这样，redis数据就存储在了宿主机上，而不是存储在容器内的`/data`目录，容器中的目录在容器重新部署时数据会被清除掉

   要做到数据持久化，必须做到：

   * 限制redis service必须运行在相同的宿主机上
   * 容器内的数据目录映射到了宿主机的文件系统上

   现在可以将添加了redis service的stack重新部署

2. 在manager上创建`./data`目录

   ```
   docker-machine ssh myvm1 "mkdir ./data"
   ```

3. 确保shell配置成连接到`myvm1`

4. 再次运行`docker stack deploy`重新部署

   ```
   $ docker stack deploy -c docker-compose.yml getstartedlab
   ```

5. 运行`docker service ls`验证3个服务是否如期运行

   ```
   $ docker service ls
   ID                  NAME                       MODE                REPLICAS            IMAGE                             PORTS
   x7uij6xb4foj        getstartedlab_redis        replicated          1/1                 redis:latest                      *:6379->6379/tcp
   n5rvhm52ykq7        getstartedlab_visualizer   replicated          1/1                 dockersamples/visualizer:stable   *:8080->8080/tcp
   mifd433bti1d        getstartedlab_web          replicated          5/5                 gordon/getstarted:latest    *:80->80/tcp
   ```

6. 在web页面上访问应用，你看到存储在redis中的数据

   ![Hello World in browser with Redis](/Docker快速入门/app-in-browser-redis.png)

   同样，检查`visualizer`是否在8080端口正常运行，同时`redis`、`web`和`visualizer`3个服务正常运行

   ![Visualizer with redis screenshot](/Docker快速入门/visualizer-with-redis.png)