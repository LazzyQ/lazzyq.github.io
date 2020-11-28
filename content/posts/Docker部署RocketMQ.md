---
title: "Docker部署RocketMQ"
date: 2020-11-29T00:55:00+08:00
draft: false
original: true
categories: 
  - Docker
tags: 
  - Docker
  - RocketMQ
---


RocketMQ不像MySQL，Redis等在仓库中有官方镜像。如果我们要使用docker部署RockerMQ需要自己构建好RockerMQ的镜像

幸运的是，在[rocketmq-docker](https://github.com/apache/rocketmq-docker)项目中已经提供了生成rocketMQ的构建脚本了，大家可以参考该项目的说明进行部署。这里我们借鉴一下这个项目，按照自己的思路来部署一下RocketMQ

首先我们需要构建自己的RocketMQ镜像

### 构建RocketMQ镜像

直接上构建RocketMQ镜像的Dockerfile文件

<!--more-->

```yml
# 基于openjdk8镜像构建
FROM openjdk:8-alpine

# 安装需要的工具
RUN apk add --no-cache bash gettext nmap-ncat openssl busybox-extras 

ARG user=rocketmq
ARG group=rocketmq
ARG uid=3000
ARG gid=3000

# RocketMQ is run with user `rocketmq`, uid = 3000
# If you bind mount a volume from the host or a data container,
# ensure you use the same uid
RUN addgroup --gid ${gid} ${group} \
    && adduser --uid ${uid} -G ${group} ${user} -s /bin/bash -D
    
# 可以通过命令行指定构建RocketMQ的版本
ARG version 

# Rocketmq version
ENV ROCKETMQ_VERSION ${version}

# 设置RocketMQ的路径
ENV ROCKETMQ_HOME  /home/rocketmq/rocketmq-${ROCKETMQ_VERSION} 

WORKDIR  ${ROCKETMQ_HOME}

# Install
RUN set -eux; \
    apk add --virtual .build-deps curl gnupg unzip; \
    curl https://dist.apache.org/repos/dist/release/rocketmq/${ROCKETMQ_VERSION}/rocketmq-all-${ROCKETMQ_VERSION}-bin-release.zip -o rocketmq.zip; \
    curl https://dist.apache.org/repos/dist/release/rocketmq/${ROCKETMQ_VERSION}/rocketmq-all-${ROCKETMQ_VERSION}-bin-release.zip.asc -o rocketmq.zip.asc; \
    #https://www.apache.org/dist/rocketmq/KEYS
	curl https://www.apache.org/dist/rocketmq/KEYS -o KEYS; \
	\
	gpg --import KEYS; \
    gpg --batch --verify rocketmq.zip.asc rocketmq.zip; \
    unzip rocketmq.zip; \
	mv rocketmq-all*/* . ; \
	rmdir rocketmq-all* ; \
	rm rocketmq.zip rocketmq.zip.asc KEYS; \
	apk del .build-deps ; \
    rm -rf /var/cache/apk/* ; \
    rm -rf /tmp/*


# 拷贝自定义的运行脚本，在scripts目录下可以修改配置，比如JVM启动时的堆大小等，RocketMQ发行版本中的JVM的堆大小设置成4G，我们可以通过这个自定义脚本覆盖这些配置
COPY scripts/ ${ROCKETMQ_HOME}/bin/

RUN chown -R ${uid}:${gid} ${ROCKETMQ_HOME}

# Expose namesrv port
EXPOSE 9876

# 覆盖namesrv的脚本
RUN mv ${ROCKETMQ_HOME}/bin/runserver-customize.sh ${ROCKETMQ_HOME}/bin/runserver.sh \
 && chmod a+x ${ROCKETMQ_HOME}/bin/runserver.sh \
 && chmod a+x ${ROCKETMQ_HOME}/bin/mqnamesrv
 
 # Expose broker ports
EXPOSE 10909 10911 10912

# 覆盖broker的脚本
RUN mv ${ROCKETMQ_HOME}/bin/runbroker-customize.sh ${ROCKETMQ_HOME}/bin/runbroker.sh \
 && chmod a+x ${ROCKETMQ_HOME}/bin/runbroker.sh \
 && chmod a+x ${ROCKETMQ_HOME}/bin/mqbroker

# Export Java options
# 在RocketMQ源码中有些默认值是根据用户名的路径下，所以这么设置很方便，但是就我部署来看，这条语句还想并没有生效
# 可以参考 https://stackoverflow.com/questions/33379393/docker-env-vs-run-export
# RUN export JAVA_OPT=" -Duser.home=/opt" 

# Add ${JAVA_HOME}/lib/ext as java.ext.dirs
RUN sed -i 's/${JAVA_HOME}\/jre\/lib\/ext/${JAVA_HOME}\/jre\/lib\/ext:${JAVA_HOME}\/lib\/ext/' ${ROCKETMQ_HOME}/bin/tools.sh

USER ${user}

WORKDIR ${ROCKETMQ_HOME}/bin
```

在构建yml文件中使用了自定义的脚本文件，这些文件在scripts目录下，所以还需要创建scripts目录，并且使用2个自定义的脚本配置文件，分别是server(namesrv是通过这个启动的)和broker的

先看runserver-customize.sh

```shell
#!/bin/bash

error_exit ()
{
    echo "ERROR: $1 !!"
    exit 1
}

find_java_home()
{
    case "`uname`" in
        Darwin)
            JAVA_HOME=$(/usr/libexec/java_home)
        ;;
        *)
            JAVA_HOME=$(dirname $(dirname $(readlink -f $(which javac))))
        ;;
    esac
}

find_java_home

[ ! -e "$JAVA_HOME/bin/java" ] && JAVA_HOME=$HOME/jdk/java
[ ! -e "$JAVA_HOME/bin/java" ] && JAVA_HOME=/usr/java
[ ! -e "$JAVA_HOME/bin/java" ] && error_exit "Please set the JAVA_HOME variable in your environment, We need java(x64)!"

export JAVA_HOME
export JAVA="$JAVA_HOME/bin/java"
export BASE_DIR=$(dirname $0)/..
export CLASSPATH=.:${BASE_DIR}/conf:${CLASSPATH}

#===========================================================================================
# JVM Configuration
#===========================================================================================
calculate_heap_sizes()
{
    case "`uname`" in
        Linux)
            system_memory_in_mb=`free -m| sed -n '2p' | awk '{print $2}'`
            system_cpu_cores=`egrep -c 'processor([[:space:]]+):.*' /proc/cpuinfo`
        ;;
        FreeBSD)
            system_memory_in_bytes=`sysctl hw.physmem | awk '{print $2}'`
            system_memory_in_mb=`expr $system_memory_in_bytes / 1024 / 1024`
            system_cpu_cores=`sysctl hw.ncpu | awk '{print $2}'`
        ;;
        SunOS)
            system_memory_in_mb=`prtconf | awk '/Memory size:/ {print $3}'`
            system_cpu_cores=`psrinfo | wc -l`
        ;;
        Darwin)
            system_memory_in_bytes=`sysctl hw.memsize | awk '{print $2}'`
            system_memory_in_mb=`expr $system_memory_in_bytes / 1024 / 1024`
            system_cpu_cores=`sysctl hw.ncpu | awk '{print $2}'`
        ;;
        *)
            # assume reasonable defaults for e.g. a modern desktop or
            # cheap server
            system_memory_in_mb="2048"
            system_cpu_cores="2"
        ;;
    esac

    # some systems like the raspberry pi don't report cores, use at least 1
    if [ "$system_cpu_cores" -lt "1" ]
    then
        system_cpu_cores="1"
    fi

    # set max heap size based on the following
    # max(min(1/2 ram, 1024MB), min(1/4 ram, 8GB))
    # calculate 1/2 ram and cap to 1024MB
    # calculate 1/4 ram and cap to 8192MB
    # pick the max
    half_system_memory_in_mb=`expr $system_memory_in_mb / 2`
    quarter_system_memory_in_mb=`expr $half_system_memory_in_mb / 2`
    if [ "$half_system_memory_in_mb" -gt "1024" ]
    then
        half_system_memory_in_mb="1024"
    fi
    if [ "$quarter_system_memory_in_mb" -gt "8192" ]
    then
        quarter_system_memory_in_mb="8192"
    fi
    if [ "$half_system_memory_in_mb" -gt "$quarter_system_memory_in_mb" ]
    then
        max_heap_size_in_mb="$half_system_memory_in_mb"
    else
        max_heap_size_in_mb="$quarter_system_memory_in_mb"
    fi
    MAX_HEAP_SIZE="${max_heap_size_in_mb}M"

    # Young gen: min(max_sensible_per_modern_cpu_core * num_cores, 1/4 * heap size)
    max_sensible_yg_per_core_in_mb="100"
    max_sensible_yg_in_mb=`expr $max_sensible_yg_per_core_in_mb "*" $system_cpu_cores`

    desired_yg_in_mb=`expr $max_heap_size_in_mb / 4`

    if [ "$desired_yg_in_mb" -gt "$max_sensible_yg_in_mb" ]
    then
        HEAP_NEWSIZE="${max_sensible_yg_in_mb}M"
    else
        HEAP_NEWSIZE="${desired_yg_in_mb}M"
    fi
}

calculate_heap_sizes

# Dynamically calculate parameters, for reference.
Xms=$MAX_HEAP_SIZE
Xmx=$MAX_HEAP_SIZE
Xmn=$HEAP_NEWSIZE
# Set for `JAVA_OPT`.
JAVA_OPT="${JAVA_OPT} -server -Xms${Xms} -Xmx${Xmx} -Xmn${Xmn}"
JAVA_OPT="${JAVA_OPT} -XX:+UseConcMarkSweepGC -XX:+UseCMSCompactAtFullCollection -XX:CMSInitiatingOccupancyFraction=70 -XX:+CMSParallelRemarkEnabled -XX:SoftRefLRUPolicyMSPerMB=0 -XX:+CMSClassUnloadingEnabled -XX:SurvivorRatio=8  -XX:-UseParNewGC"
JAVA_OPT="${JAVA_OPT} -verbose:gc -Xloggc:/dev/shm/rmq_srv_gc.log -XX:+PrintGCDetails"
JAVA_OPT="${JAVA_OPT} -XX:-OmitStackTraceInFastThrow"
JAVA_OPT="${JAVA_OPT}  -XX:-UseLargePages"
JAVA_OPT="${JAVA_OPT} -Djava.ext.dirs=${JAVA_HOME}/jre/lib/ext:${BASE_DIR}/lib"
#JAVA_OPT="${JAVA_OPT} -Xdebug -Xrunjdwp:transport=dt_socket,address=9555,server=y,suspend=n"
JAVA_OPT="${JAVA_OPT} ${JAVA_OPT_EXT}"
JAVA_OPT="${JAVA_OPT} -cp ${CLASSPATH}"

$JAVA ${JAVA_OPT} $@
```

可以直接修改上面的`JAVA_OPT`成我们想要的配置

看完了runserver-customize.sh，再看看runbroker-customize.sh

```shell
#!/bin/bash

error_exit ()
{
    echo "ERROR: $1 !!"
    exit 1
}

find_java_home()
{
    case "`uname`" in
        Darwin)
            JAVA_HOME=$(/usr/libexec/java_home)
        ;;
        *)
            JAVA_HOME=$(dirname $(dirname $(readlink -f $(which javac))))
        ;;
    esac
}

find_java_home

[ ! -e "$JAVA_HOME/bin/java" ] && JAVA_HOME=$HOME/jdk/java
[ ! -e "$JAVA_HOME/bin/java" ] && JAVA_HOME=/usr/java
[ ! -e "$JAVA_HOME/bin/java" ] && error_exit "Please set the JAVA_HOME variable in your environment, We need java(x64)!"

export JAVA_HOME
export JAVA="$JAVA_HOME/bin/java"
export BASE_DIR=$(dirname $0)/..
export CLASSPATH=.:${BASE_DIR}/conf:${CLASSPATH}

#===========================================================================================
# JVM Configuration
#===========================================================================================
calculate_heap_sizes()
{
    case "`uname`" in
        Linux)
            system_memory_in_mb=`free -m| sed -n '2p' | awk '{print $2}'`
            system_cpu_cores=`egrep -c 'processor([[:space:]]+):.*' /proc/cpuinfo`
        ;;
        FreeBSD)
            system_memory_in_bytes=`sysctl hw.physmem | awk '{print $2}'`
            system_memory_in_mb=`expr $system_memory_in_bytes / 1024 / 1024`
            system_cpu_cores=`sysctl hw.ncpu | awk '{print $2}'`
        ;;
        SunOS)
            system_memory_in_mb=`prtconf | awk '/Memory size:/ {print $3}'`
            system_cpu_cores=`psrinfo | wc -l`
        ;;
        Darwin)
            system_memory_in_bytes=`sysctl hw.memsize | awk '{print $2}'`
            system_memory_in_mb=`expr $system_memory_in_bytes / 1024 / 1024`
            system_cpu_cores=`sysctl hw.ncpu | awk '{print $2}'`
        ;;
        *)
            # assume reasonable defaults for e.g. a modern desktop or
            # cheap server
            system_memory_in_mb="2048"
            system_cpu_cores="2"
        ;;
    esac

    # some systems like the raspberry pi don't report cores, use at least 1
    if [ "$system_cpu_cores" -lt "1" ]
    then
        system_cpu_cores="1"
    fi

    # set max heap size based on the following
    # max(min(1/2 ram, 1024MB), min(1/4 ram, 8GB))
    # calculate 1/2 ram and cap to 1024MB
    # calculate 1/4 ram and cap to 8192MB
    # pick the max
    half_system_memory_in_mb=`expr $system_memory_in_mb / 2`
    quarter_system_memory_in_mb=`expr $half_system_memory_in_mb / 2`
    if [ "$half_system_memory_in_mb" -gt "1024" ]
    then
        half_system_memory_in_mb="1024"
    fi
    if [ "$quarter_system_memory_in_mb" -gt "8192" ]
    then
        quarter_system_memory_in_mb="8192"
    fi
    if [ "$half_system_memory_in_mb" -gt "$quarter_system_memory_in_mb" ]
    then
        max_heap_size_in_mb="$half_system_memory_in_mb"
    else
        max_heap_size_in_mb="$quarter_system_memory_in_mb"
    fi
    MAX_HEAP_SIZE="${max_heap_size_in_mb}M"

    # Young gen: min(max_sensible_per_modern_cpu_core * num_cores, 1/4 * heap size)
    max_sensible_yg_per_core_in_mb="100"
    max_sensible_yg_in_mb=`expr $max_sensible_yg_per_core_in_mb "*" $system_cpu_cores`

    desired_yg_in_mb=`expr $max_heap_size_in_mb / 4`

    if [ "$desired_yg_in_mb" -gt "$max_sensible_yg_in_mb" ]
    then
        HEAP_NEWSIZE="${max_sensible_yg_in_mb}M"
    else
        HEAP_NEWSIZE="${desired_yg_in_mb}M"
    fi
}

calculate_heap_sizes

# Dynamically calculate parameters, for reference.
Xms=$MAX_HEAP_SIZE
Xmx=$MAX_HEAP_SIZE
Xmn=$HEAP_NEWSIZE
MaxDirectMemorySize=$MAX_HEAP_SIZE
# Set for `JAVA_OPT`.
JAVA_OPT="${JAVA_OPT} -server -Xms${Xms} -Xmx${Xmx} -Xmn${Xmn}"
JAVA_OPT="${JAVA_OPT} -XX:+UseG1GC -XX:G1HeapRegionSize=16m -XX:G1ReservePercent=25 -XX:InitiatingHeapOccupancyPercent=30 -XX:SoftRefLRUPolicyMSPerMB=0 -XX:SurvivorRatio=8"
JAVA_OPT="${JAVA_OPT} -verbose:gc -Xloggc:/dev/shm/mq_gc_%p.log -XX:+PrintGCDetails -XX:+PrintGCDateStamps -XX:+PrintGCApplicationStoppedTime -XX:+PrintAdaptiveSizePolicy"
JAVA_OPT="${JAVA_OPT} -XX:+UseGCLogFileRotation -XX:NumberOfGCLogFiles=5 -XX:GCLogFileSize=30m"
JAVA_OPT="${JAVA_OPT} -XX:-OmitStackTraceInFastThrow"
JAVA_OPT="${JAVA_OPT} -XX:+AlwaysPreTouch"
JAVA_OPT="${JAVA_OPT} -XX:MaxDirectMemorySize=${MaxDirectMemorySize}"
JAVA_OPT="${JAVA_OPT} -XX:-UseLargePages -XX:-UseBiasedLocking"
JAVA_OPT="${JAVA_OPT} -Djava.ext.dirs=${JAVA_HOME}/jre/lib/ext:${BASE_DIR}/lib"
#JAVA_OPT="${JAVA_OPT} -Xdebug -Xrunjdwp:transport=dt_socket,address=9555,server=y,suspend=n"
JAVA_OPT="${JAVA_OPT} ${JAVA_OPT_EXT}"
JAVA_OPT="${JAVA_OPT} -cp ${CLASSPATH}"

numactl --interleave=all pwd > /dev/null 2>&1
if [ $? -eq 0 ]
then
	if [ -z "$RMQ_NUMA_NODE" ] ; then
		numactl --interleave=all $JAVA ${JAVA_OPT} $@
	else
		numactl --cpunodebind=$RMQ_NUMA_NODE --membind=$RMQ_NUMA_NODE $JAVA ${JAVA_OPT} $@
	fi
else
	$JAVA ${JAVA_OPT} $@
fi
```

到现在我们的目录如下

```
.
├── Dockerfile
└── scripts
    ├── runbroker-customize.sh
    └── runserver-customize.sh
```

现在尝试在这个目录下运行下面的命令来构建RocketMQ镜像

```shell
docker build --no-cache -t rocketmq:${ROCKETMQ_VERSION}-alpine \
--build-arg version=${ROCKETMQ_VERSION} \
--build-arg uid=$(id -u) \
--build-arg gid=$(id -g) .
```

其中ROCKETMQ_VERSION是我们想要构建的RocketMQ的版本，可以参考[这里](https://dist.apache.org/repos/dist/release/rocketmq/)查看RocketMQ的发行版本

比如我现在制定版本是`4.5.2`

```shell
docker build --no-cache -t rocketmq:4.5.2-alpine \
--build-arg version=4.5.2 \
--build-arg uid=$(id -u) \
--build-arg gid=$(id -g) .
```

> 注意：在写Dockerfile时，注释最好单独成行，不要在语句后面写注释，会出现一些异常
>
> 比如 Error response from daemon: Dockerfile parse error line 18: ARG requires exactly one argument

对于RocketMQ Console可以直接使用已经制作好的镜像[rocketmq-console-ng](https://hub.docker.com/r/styletang/rocketmq-console-ng)

如果想单独运行rocketmq-console-ng，可以参考[文档](https://github.com/apache/rocketmq-externals/tree/master/rocketmq-console)，比如直接运行下面命令

```shell
docker pull styletang/rocketmq-console-ng

docker run -e "JAVA_OPTS=-Drocketmq.namesrv.addr=127.0.0.1:9876 -Dcom.rocketmq.sendMessageWithVIPChannel=false" -p 8080:8080 -t styletang/rocketmq-console-ng
```

### 编写docker-compose.yml文件

正如文章开头说的那样，生产环境肯定不能通过命令行单独去部署namesrv，broker和console的，所以我们需要编写docker-compose.yml文件，同样可以与其他服务一起使用docker stack进行部署

```yml
version: '3.7'
services:
  namesrv:
    image: choaya/rocketmq:4.5.2-alpine
    container_name: rmqnamesrv
    ports:
      - 9876:9876
    networks:
      - net
    volumes:
      # 在Dockerfile中我们制定了user.home是/opt，RocketMQ源码中的文件默认都是基于user.home的
      - ~/wemeng/rocketMQ/data/namesrv/logs:/home/rocketmq/logs
    command: sh mqnamesrv
  broker1:
    image: choaya/rocketmq:4.5.2-alpine
    container_name: rmqbroker-a
    networks:
      - net
    ports:
      - 10909:10909
      - 10911:10911
      - 10912:10912
    depends_on:
      - namesrv
    environment:
      - NAMESRV_ADDR=namesrv:9876
    volumes:
      - ~/wemeng/rocketMQ/data/broker/broker1/logs:/home/rocketmq/logs
      - ~/wemeng/rocketMQ/data/broker/broker1/store:/home/rocketmq/store
      - ~/wemeng/rocketMQ/data/broker/broker1/conf/broker.conf:/home/rocketmq/rocketmq-4.5.2/conf/broker.conf
    command: sh mqbroker -c ../conf/broker.conf
  broker2:
    image: choaya/rocketmq:4.5.2-alpine
    container_name: rmqbroker-b
    networks:
      - net
    ports:
      - 10929:10909
      - 10931:10911
      - 10932:10912
    depends_on:
      - namesrv
    environment:
      - NAMESRV_ADDR=namesrv:9876
    volumes:
      - ~/wemeng/rocketMQ/data/broker/broker2/logs:/home/rocketmq/logs
      - ~/wemeng/rocketMQ/data/broker/broker2/store:/home/rocketmq/store
      - ~/wemeng/rocketMQ/data/broker/broker2/conf/broker.conf:/home/rocketmq/rocketmq-4.5.2/conf/broker.conf
    command: sh mqbroker -c ../conf/broker.conf
  console:
    image: styletang/rocketmq-console-ng:latest
    container_name: rmqconsole
    networks:
      - net
    ports:
      - 8080:8080
    depends_on:
      - namesrv
    environment:
     JAVA_OPTS: -Drocketmq.config.namesrvAddr=namesrv:9876 -Dcom.rocketmq.sendMessageWithVIPChannel=false
networks:
  net:
    ipam:
      config:
        - subnet: 172.27.0.0/16
```

broker.conf的配置这里就不贴了，个人根据自己的需求进行配置就行

然后使用部署

```shell
docker-compose -f docker-compose.yml up
```

### 验证部署成功

访问RocketMQ的Console页面，可以看到Namesrv和Broker的信息就算是成功了

![Console中查看Namesrv](/Docker部署RocketMQ/image-20191001151603057.png)

![Console中查看Broker](/Docker部署RocketMQ/image-20191001151634690.png)

> 注意：可以发现在console中broker的地址注册的ip是172.27.0.5，172.27.0.3，这个ip是docker的ip

如果想发送和消费消息，初始化Client时指定的是namesrv的地址，broker会将自己的ip信息注册到namesrv，由于是在docker环境中，broker注册到namesrv的ip是docker的ip，这个就是上面console中看到docker ip的原因

那么Producer和Consumer是根据namesrv获取到broker的ip的，所以它们会拿着这个docker ip去建立连接并发送和消费消息，可想而知肯定是连接不上的，ip都不正确，如果你尝试发送消息的话会看到如下的错误

```
Exception in thread "main" org.apache.rocketmq.remoting.exception.RemotingTooMuchRequestException: sendDefaultImpl call timeout
	at org.apache.rocketmq.client.impl.producer.DefaultMQProducerImpl.sendDefaultImpl(DefaultMQProducerImpl.java:640)
	at org.apache.rocketmq.client.impl.producer.DefaultMQProducerImpl.send(DefaultMQProducerImpl.java:1310)
	at org.apache.rocketmq.client.impl.producer.DefaultMQProducerImpl.send(DefaultMQProducerImpl.java:1256)
	at org.apache.rocketmq.client.producer.DefaultMQProducer.send(DefaultMQProducer.java:339)
	at com.lazzyq.unit.rocketmq.SyncProducer.main(SyncProducer.java:16)
22:04:37.807 [NettyClientSelector_1] INFO  RocketmqRemoting closeChannel: close the connection to remote address[] result: true
22:04:38.323 [NettyClientSelector_1] INFO  RocketmqRemoting closeChannel: close the connection to remote address[] result: true
22:04:41.329 [NettyClientSelector_1] INFO  RocketmqRemoting closeChannel: close the connection to remote address[] result: true
```

那么该如何解决这个问题呢，知道原因了，解决问题还不简单，我们已经知道是由于docker环境下broker上报自己的ip不正确导致的，那么我们看一下这段逻辑，看看能不能修改一下上报的ip就可以了呗，这里就不看源码了，以后再分析吧。直接给解决方案

在docker-compose.yml中，我们映射了broker的配置文件，可以添加brokerIP1配置，值就是你主机正确的ip地址，源码中BrokerConfig中有brokerIP1属性，上报namesrv的就是这个属性，所以通过配置文件直接覆盖了这个属性

```
 - ~/wemeng/rocketMQ/data/broker/broker2/conf/broker.conf:/home/rocketmq/rocketmq-4.5.2/conf/broker.conf
```

```
# brokerIP1 上报到namesrv的ip地址，docker环境下上报的是docker的ip
# brokerIP1=[你主机的ip地址]
```