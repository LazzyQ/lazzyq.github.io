---
title: "使用Kubeadm部署k8s集群"
date: 2021-01-17T14:32:18+08:00
draft: false
original: true
categories: 
  - k8s
tags: 
  - k8s
---


### Step 1: Delete SWAP

在安裝 k8s 前，必須把所有 node 的 swap disable 。

```
$ sudo swapoff -a
```

### Step 2: Install docker

### Step 3: Install Kubectl Kubeadm Kubelet

```
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

<!--more-->

#### Step 3–1 : 在 master 中初始化 kubeadm

```
$ kubeadm init

...

Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

Alternatively, if you are the root user, you can run:

  export KUBECONFIG=/etc/kubernetes/admin.conf

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 192.168.0.199:6443 --token u0u1ul.5idimtl8jabsecz3 \
    --discovery-token-ca-cert-hash sha256:2b9101056e14cc406c950e570e6a05a750ea15e132317cc1d336e1c5a00fd396
```

执行下面的命令，`STATUS`显示是`NotReady`，因为还没安装CNI

```
$ kubectl get nodes
NAME                    STATUS     ROLES                  AGE     VERSION
zengqiang96-pc-server   NotReady   control-plane,master   4m41s   v1.20.2
```
看一下目前現有的 pods 有哪些。

```
$ kubectl -n kube-system get pods

NAME                                            READY   STATUS    RESTARTS   AGE
coredns-74ff55c5b-7pknl                         0/1     Pending   0          5m23s
coredns-74ff55c5b-zjr8w                         0/1     Pending   0          5m23s
etcd-zengqiang96-pc-server                      1/1     Running   0          5m21s
kube-apiserver-zengqiang96-pc-server            1/1     Running   0          5m21s
kube-controller-manager-zengqiang96-pc-server   1/1     Running   0          5m21s
kube-proxy-5pntf                                1/1     Running   0          5m23s
kube-scheduler-zengqiang96-pc-server            1/1     Running   0          5m21s
```

### Step 4: Install CNI

接著安裝 CNI ，我所使用的是 Calico 因為這是官方建議的


```
$ curl https://docs.projectcalico.org/manifests/calico.yaml -O
$ kubectl apply -f calico.yaml
$ kubectl -n kube-system get pod -w
$ kubectl get  nodes
NAME                    STATUS   ROLES                  AGE   VERSION
zengqiang96-pc-server   Ready    control-plane,master   28m   v1.20.2
```

### 刪除 Kubenetes cluster

```
$ kubeadm reset
$ sudo apt-get purge kubeadm kubectl kubelet kubernetes-cni kube*   
$ sudo apt-get autoremove  
$ sudo rm -rf ~/.kube
```


### 遇到的问题

1. 安装完k8s后，重启机器后，k8s启动失败

因为安装的时候使用`swapoff -a`关闭的SWAP，所以重启的时候SWAP又挂载了，需要将SWAP的自动挂载去掉

```
vim  /etc/fstab 

注释掉 SWAP 的自动挂载
```