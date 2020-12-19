---
title: "Docker使用"
date: 2020-12-19T21:24:14+08:00
draft: false
original: true
categories: 
  - Docker
tags: 
  - Docker
---

###  安装docker

#### 官网安装文档

##### mac 安装

通过[链接](https://hub.docker.com/editions/community/docker-ce-desktop-mac)下载安装包，注意需要登陆才能下载。

下载完成双击`Docker.dmg`，将docker拖进应用列表中

![Install Docker app](/Docker使用/docker-app-drag.png)

在Launchpad中点击docker的图标启动

![Docker app in Hockeyapp](/Docker使用/docker-app-in-apps.png)

启动后，菜单栏会出现docker的图标，等docker的状态变成`running`后就可以使用了。

![Whale in menu bar](/Docker使用/whale-in-menu-bar.png)

<!-- more -->

##### Ubuntu 安装

参考[官网文档](https://docs.docker.com/install/linux/docker-ce/ubuntu/)

###### 从仓库安装

**配置仓库**

```
$ sudo apt-get update

$ sudo apt-get install \
    apt-transport-https \
    ca-certificates \
    curl \
    gnupg-agent \
    software-properties-common
    
$ curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -

$ sudo apt-key fingerprint 0EBFCD88

$ sudo add-apt-repository \
   "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
   $(lsb_release -cs) \
   stable"
```

**安装docker**

```
$ sudo apt-get update

$ sudo apt-get install docker-ce docker-ce-cli containerd.io
```

#### DaoCloud 加速安装

Docker 的 [安装资源文件 ](https://get.docker.com/)存放在Amazon S3，会间歇性连接失败。所以安装Docker的时候，会比较慢。 
你可以通过执行下面的命令，高速安装Docker。

```
curl -sSL https://get.daocloud.io/docker | sh
```

适用于Ubuntu，Debian,Centos等大部分Linux，会3小时同步一次Docker官方资源

### 安装Docker Compose

参考[官网文档](https://docs.docker.com/compose/install/)

#### Mac OS 安装Docker Compose

Mac OS 的安装包已经带有Docker Compose，不需要单独安装

#### Ubuntu 安装Docker Compose

```
sudo curl -L "https://github.com/docker/compose/releases/download/1.24.1/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose

sudo chmod +x /usr/local/bin/docker-compose
```

如果docker-compose命令运行失败，可以创建一个连接到`/usr/bin`

```
sudo ln -s /usr/local/bin/docker-compose /usr/bin/docker-compose
```

#### DaoCloud 加速安装

参考[DaoCloud文档](http://get.daocloud.io/#install-docker)

[Docker Compose ](https://docs.docker.com/compose/install/)存放在Git Hub，不太稳定。 
你可以也通过执行下面的命令，高速安装Docker Compose。

```
curl -L https://get.daocloud.io/docker/compose/releases/download/1.24.1/docker-compose-`uname -s`-`uname -m` > /usr/local/bin/docker-compose
chmod +x /usr/local/bin/docker-compose
```

你可以通过修改URL中的版本，可以自定义您的需要的版本。

### 安装Docker Machine

1. 需先安装Docker

2. 下载Docker M achinede的二进制文件，然后解压到PATH

   MacOS上:

   ```
   $ base=https://github.com/docker/machine/releases/download/v0.16.0 &&
     curl -L $base/docker-machine-$(uname -s)-$(uname -m) >/usr/local/bin/docker-machine &&
   $ chmod +x /usr/local/bin/docker-machine
   ```

   Linux上:

   ```
   $ base=https://github.com/docker/machine/releases/download/v0.16.0 &&
     curl -L $base/docker-machine-$(uname -s)-$(uname -m) >/tmp/docker-machine &&
     sudo mv /tmp/docker-machine /usr/local/bin/docker-machine
   ```

3. 检查Docker Machine的安装版本

   ```
   $ docker-machine version
   docker-machine version 0.16.0, build 9371605
   ```

#### 安装VirtualBox

下载[相应版本的VirtualBox](https://www.virtualbox.org/wiki/Downloads)，直接安装即可

**Ubuntu 18.04安装**

下载对应版本的VirtualBox安装后，运行`docker-machine create --driver virtualbox myvm1`发现报错

```
$ docker-machine create --driver virtualbox myvm1
Running pre-create checks...
Error with pre-create check: "We support Virtualbox starting with version 5. Your VirtualBox install is \"WARNING: The vboxdrv kernel module is not loaded. Either there is no module\\n         available for the current kernel (4.18.0-24-generic) or it failed to\\n         load. Please recompile the kernel module and install it by\\n\\n           sudo /sbin/vboxconfig\\n\\n         You will not be able to start VMs until this problem is fixed.\\n6.0.10r132072\". Please upgrade at https://www.virtualbox.org"
```

网上搜到的解决方法是`/etc/rc.d/vboxdrv setup`，可是我都没这个vboxdrv。还好搜包的时候看到这么一句。
`virtualbox-dkms - x86 virtualization solution - kernel module sources for dkms`
那就装一下额，意外的好了 - -
所以这个问题解决方法是

```
$ sudo apt-get install virtualbox-dkms
```

virtualbox-dkms安装完成后再次运行`docker-machine create --driver virtualbox myvm1`

```
$ docker-machine create --driver virtualbox myvm1
Running pre-create checks...
Creating machine...
(myvm1) Copying /home/zengqiang96/.docker/machine/cache/boot2docker.iso to /home/zengqiang96/.docker/machine/machines/myvm1/boot2docker.iso...
(myvm1) Creating VirtualBox VM...
(myvm1) Creating SSH key...
(myvm1) Starting the VM...
(myvm1) Check network to re-create if needed...
Error creating machine: Error in driver during machine creation: This computer doesn't have VT-X/AMD-v enabled. Enabling it in the BIOS is mandatory
```

还是报错，错误类型原因是`Error creating machine: Error in driver during machine creation: This computer doesn't have VT-X/AMD-v enabled. Enabling it in the BIOS is mandatory`

在网上寻找解决方案，发现很多都是Windows上使用虚拟机没有开启虚拟导致的，但是我是在物理机上安装的Ubuntu，并没有使用虚拟机，根据提示需要在BIOS中手动启用虚拟化，所以重启机器后进入BIOS，启用了处理器虚拟化功能后再次运行就OK了

### 仓库镜像加速

**Linux加速**

在 /etc/docker/daemon.json 文件中添加以下参数（没有该文件则新建）：

```json
{
  "registry-mirrors": ["https://9cpn8tt6.mirror.aliyuncs.com"]
}
```

其他的加速站

[https://docker.mirrors.ustc.edu.cn/](https://mirrors.ustc.edu.cn/help/dockerhub.html)

[https://9cpn8tt6.mirror.aliyuncs.com](https://help.aliyun.com/document_detail/60750.html)

重启docker服务

```
sudo systemctl restart docker.service
```

**Mac OS 加速**

右键点击桌面顶栏的 docker 图标，选择 Preferences ，在 Daemon 标签（Docker 17.03 之前版本为 Advanced 标签）下的 Registry mirrors 列表中加入下面的镜像地址:

```
http://f1361db2.m.daocloud.io
```

点击 Apply & Restart 按钮使设置生效。

### Docker命令

#### 批量删除Docker镜像

在使用一段时间Docker后，会有很多我们不再需要的镜像，我们通过`docker image rm`可以逐个删除，但是效率很低，有没有批量删除的方法呢？

```
docker rmi `docker images | grep xxxx | awk '{print $3}'`