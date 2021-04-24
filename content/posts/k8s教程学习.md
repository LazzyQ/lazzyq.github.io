---
title: "K8s教程学习"
date: 2021-04-24T23:36:36+08:00
draft: false
original: true
categories: 
  - k8s
tags: 
  - k8s
---

视频教程： [https://www.youtube.com/watch?v=X48VuDVv0do](https://www.youtube.com/watch?v=X48VuDVv0do)

# Kubernetes Components

## Node&Pod

![](/k8s教程学习/Untitled.png)

- Node:
    - 物理机或虚拟机
- Pod:
    - k8s最小的单元；
    - 在container之上的抽象；
    - 通常每个pod运行1个应用程序；
    - 每个pod拥有自己的IP地址

如果pod重启，ip会发生变化，所以需要另外一个组件service

## Service&Ingress

![](/k8s教程学习/Untitled01.png)

- Service:
    - 永久的IP地址；
    - Pod和Service的生命周期不是关联的；Pod挂掉后，Service地址仍然存在；
- Ingress：
    - 对外提供访问能力；

## ConfigMap&Secret

![](/k8s教程学习/Untitled02.png)

应用之间可以通过Service进行通信，比如我们DB的Service是mongo-db-service，这时候在my-app应用里面会通过mongo-db-service URL访问DB，如果直接在代码里面配置URL，如果mongo-db-service变成了mongo-db，这时候就不得不修改代码，重新build，push。这就有点没人性了。

![](/k8s教程学习/Untitled03.png)

k8s提供了ConfigMap组件作为应用的外部配置，将DB_URL配置在ConfigMap中来解决上述的问题。但是通常访问数据库需要提供user,pwd等认证信息，这些信息可能也会发生改变，但是

Don't pit credentials into ConfigMap

对于这类信息k8s提供了Secret组件

![](/k8s教程学习/Untitled04.png)

Secret和ConfigMap差不多，通常用来存储私密的数据，并且进行base64编码

在Pod内部可以使用环境变量或属性文件来访问ConfigMap或Secret中的数据

## Volumes

接下来了解k8s中的数据存储组件

![](/k8s教程学习/Untitled05.png)

如果DB容器重启，其内部的数据也随之消失，但是DB数据我们需要长期存储的，这时候就需要使用Volumes组件

![](/k8s教程学习/Untitled06.png)

Volumes组件将存储attach到Pod，存储可以是本地机器上的存储设备，也可以是k8s集群外部的存储设备

![](/k8s教程学习/Untitled07.png)

可以认为这是k8s集群的外部存储，k8s自身并不会管理数据的持久化

## Deployment&Stateful Set

到目前为止一切ok，用户能够通过浏览器访问应用。但是当my-app这个pod挂了之后，整个系统就不能访问了，于是我们需要副本来保证可用性。

![](/k8s教程学习/Untitled08.png)

Service组件提供了2种能力

- 永久IP
- 负载均衡

因此我们需要为my-app部署2个pod

![](/k8s教程学习/Untitled09.png)

但是我们不直接创建pod，而是使用Deployment组件

- pod的buleprint，也就是怎么创建一个pod
- Pods的抽象

在实际操作中，我们通常创建的都是deployment，而不是pod

![](/k8s教程学习/Untitled10.png)

DB同样会存在类似的单点问题，但是不能简单的使用Deployment来解决，因为DB是有状态的

![](/k8s教程学习/Untitled11.png)

多个DB pd访问相同的volumes存储，为了避免数据的不一致，通常DB是有状态的，这时候就需要StatefulSet组件

![](/k8s教程学习/Untitled12.png)

但是部署StatefulSet是一件复杂的时候，所以通常DB都是部署在k8s集群之外的

# k8s architecture

## Node Processes

![](/k8s教程学习/Untitled13.png)

worker节点需要3 Processes

- Conatiner runtimer
- kubelet
    - interacts with both(the container & node)
    - starts the node with a container inside
- Kube Proxy
    - forwards the  requests

kube Proxy转发请求，途中node1的my-app请求DB Service，并不会 请求到node2上的DB，而是转发到本机上的DB

## Master Processes

master需要4 processes

![](/k8s教程学习/Untitled14.png)

- Api Server
    - cluster gateway
    - acts as a gatekeeper for authentication
- Scheduler
    - just decides on which Node New Pod should be scheduled
- Controller manager
    - detects cluster  state changes
- etcd
    - cluster brain
    - 存储k8s集群的数据信息

Application data is NOT stored in etcd!

- API Server is load balanced
- Distributed storage across all master nodes

# 主要的 kubectl 命令

创建deployment

```go
$ kubectl create deployment nginx-deploy --image=nginx

deployment.apps/nginx-deploy created
```

获取deployment列表

```go
$ kubectl get deploy

NAME           READY   UP-TO-DATE   AVAILABLE   AGE
nginx-deploy   1/1     1            1           51s
```

获取replicaset列表

```go
$ kubectl get replicaset

NAME                     DESIRED   CURRENT   READY   AGE
nginx-deploy-8588f9dfb   1         1         1       2m39s
```

获取pod列表

```go
$ kubectl get pod

NAME                           READY   STATUS    RESTARTS   AGE
nginx-deploy-8588f9dfb-gpxhv   1/1     Running   0          3m2s
```

![](/k8s教程学习/Untitled15.png)

编辑deployment

```go
$ kubectl edit  deploy nginx-deploy
```

删除deployment

```go
$ kubectl delete deployment nginx-deploy

deployment.apps "nginx-deploy" deleted
```

使用文件

```go
kubectl apply -f nginx-deployment.yaml
```

```go
$ cat nginx-deployment.yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deploy
  labels:
    app: nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.16
        ports:
        - containerPort: 80
```

```go
$ kubectl delete -f nginx-deployment.yaml
```

# 部署实例

整体看一下需要部署那些组件

![](/k8s教程学习/Untitled16.png)

## mongo deployment

创建secret

```go
$ cat mongo-secret.yaml

apiVersion: v1
kind: Secret
metadata:
  name: mongodb-secret
type: Opaque
data:
  mongo-root-username: dXNlcm5hbWU=
  mongo-root-password: cGFzc3dvcmQ=
```

```go
$ kubectl apply -f mongo-secret.yaml

secret/mongodb-secret created
```

```go
$  kubectl get secret

NAME                  TYPE                                  DATA   AGE
default-token-4x7cl   kubernetes.io/service-account-token   3      97d
mongodb-secret        Opaque                                2      29s
```

创建mongo deployment

```go
$ cat mongo-deployment.yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: mongodb-deploy
  labels:
    app: mongodb
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mongodb
  template:
    metadata:
      labels:
        app: mongodb
    spec:
      containers:
      - name: mongodb
        image: mongo
        ports:
        - containerPort: 27017
        env:
        - name: MONGO_INITDB_ROOT_USERNAME
          valueFrom:
            secretKeyRef:
              name: mongodb-secret
              key: mongo-root-username
        - name: MONGO_INITDB_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mongodb-secret
              key: mongo-root-password
```

```go
$ kubectl apply -f mongo-deployment.yaml
```

创建mongo service

```go
$ cat  mongo-service.yaml

apiVersion: v1
kind: Service
metadata:
  name: mongodb-service
spec:
  selector:
    app: mongodb
  ports:
  - protocol: TCP
    port: 27017
    targetPort: 27017
```

```go
$kubectl apply -f mongo-service.yaml

service/mongodb-service created
```

当然也可以将mongo-deployment.yaml和mongo-service.yaml放到1个文件中

```go
$ cat mongo.yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: mongodb-deploy
  labels:
    app: mongodb
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mongodb
  template:
    metadata:
      labels:
        app: mongodb
    spec:
      containers:
      - name: mongodb
        image: mongo
        ports:
        - containerPort: 27017
        env:
        - name: MONGO_INITDB_ROOT_USERNAME
          valueFrom:
            secretKeyRef:
              name: mongodb-secret
              key: mongo-root-username
        - name: MONGO_INITDB_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mongodb-secret
              key: mongo-root-password
---
apiVersion: v1
kind: Service
metadata:
  name: mongodb-service
spec:
  selector:
    app: mongodb
  ports:
  - protocol: TCP
    port: 27017
    targetPort: 27017
```

**部署mongo-express**

创建mongo-configmap

```go

$ cat mongo-configmap.yaml

apiVersion: v1
kind: ConfigMap
metadata:
  name: mongodb-configmap
data:
  database_url: mongodb-service
```

创建mongo-express-deployment

```go
$ cat mongo-express-deployment.yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: mongo-express-deploy
  labels:
    app: mongo-express
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mongo-express
  template:
    metadata:
      labels:
        app: mongo-express
    spec:
      containers:
      - name: mongo-express
        image: mongo-express
        ports:
        - containerPort: 8081
        env:
        - name: ME_CONFIG_MONGODB_ADMINUSERNAME
          valueFrom:
            secretKeyRef:
              name: mongodb-secret
              key: mongo-root-username
        - name: ME_CONFIG_MONGODB_ADMINPASSWORD
          valueFrom:
            secretKeyRef:
              name: mongodb-secret
              key: mongo-root-password
        - name: ME_CONFIG_MONGODB_SERVER
          valueFrom:
            configMapKeyRef:
              name: mongodb-configmap
              key: database_url
```

```go
$ kubectl apply -f mongo-express-deployment.yaml
deployment.apps/mongo-express-deploy created
```

```go
$ kubectl logs -f mongo-express-deploy-78fcf796b8-s5b9l

Waiting for mongodb-service:27017...
Welcome to mongo-express
------------------------

Mongo Express server listening at http://0.0.0.0:8081
Server is open to allow connections from anyone (0.0.0.0)
basicAuth credentials are "admin:pass", it is recommended you change this in your config.js!
Database connected
Admin Database connected
```

创建mongo-express service

为了能从浏览器访问到 mongo-express，需要让mongo-express service变成External Service

![](/k8s教程学习/Untitled17.png)

```go
$ cat mongo-express-service.yaml

apiVersion: v1
kind: Service
metadata:
  name: mongo-express-service
spec:
  selector:
    app: mongo-express
  type: LoadBalancer
  ports:
  - protocol: TCP
    port: 8081
    targetPort: 8081
    nodePort: 30000
```

```go
$ kubectl apply -f  mongo-express-service.yaml

service/mongo-express-service created
```

# Namespace

namespace的使用场景

![](/k8s教程学习/Untitled18.png)

修改默认的namepsace

```go
kubectl config set-context --current --namespace=<namespace>
```

# Ingress

安装Ingress-controller

```go
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v0.45.0/deploy/static/provider/baremetal/deploy.yaml
```

部署ingress

```go
$ cat mongo-ingress.yaml

apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: mongo-express-ingress
  namespace: genunities
spec:
  rules:
  - host: mongoexpress.com
    http:
      paths:
      - pathType: Prefix
        path: "/"
        backend:
          service:
            name: mongo-express-service
            port:
              number: 8081
```

然后就能通过浏览器访问了

![](/k8s教程学习/Untitled19.png)