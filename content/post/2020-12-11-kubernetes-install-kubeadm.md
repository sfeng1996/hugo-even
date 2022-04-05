---
layout:     post
title:      "kubeadm 部署kubernetes"
description: "kubeadm 部署kubernetes"
date:     2021-12-11
author: sfeng
categories: ["Tech", "cloudNative", "ops"]
tags: ["kubernetes", "kubeadm"]
URL: "/2021/12/11/kubeadm-install/"
---

## 简介

当需要基于 `kubernetes` 环境进行编译调试时，最好能在本地开发环境搭建一套 `kubernetes`

下面基于 `ubuntu18` 版本部署 `kubernetes v1.18.2`

## 系统准备

本人用的是 `Ubuntu18`，以下以此为例

部署之前，最好切换至 `root` 用户，方便操作

## 系统初始化

```bash
$ sudo ufw disable   // 关闭防火墙
$ sudo systemctl stop ufw
$ sudo systemctl disable ufw  // 永久关闭防火墙
$ sudo swapoff -a    // 关闭swap
$ sed -ri 's/.*swap.*/#&/' /etc/fstab  // 永久关闭swap
ubuntu系统默认没有安装selinux

// 更换国内镜像源
$ cp /etc/apt/sources.list /etc/apt/sources.list.bak
$ sed -i 's@http://cn.mirrors.ustc.edu.cn/ubuntu/@https://mirrors.tuna.tsinghua.edu.cn/ubuntu/@g' /etc/apt/sources.list
$ apt update
```

## 安装docker

```graphql
$ curl -fsSL get.docker.com -o get-docker.sh
$ sudo sh get-docker.sh --mirror Aliyun
$ docker run hello-world  // 验证docker是否安装完成
```

## 添加k8s安装秘钥

```bash
$ sudo apt update &&  sudo apt install -y apt-transport-https curl
$ curl -s https://mirrors.aliyun.com/kubernetes/apt/doc/apt-key.gpg |  sudo apt-key add -
```

## 配置k8s源

```bash
$ sudo touch /etc/apt/sources.list.d/kubernetes.list
这里选择阿里云的源
$ sudo  echo  "deb https://mirrors.aliyun.com/kubernetes/apt/ kubernetes-xenial main"  >> /etc/apt/sources.list.d/kubernetes.list
```

## 安装kubelet，kubeadm，kubectl

```bash
$ sudo  apt-get update
// 这里带上版本号，防止后续部署报错版本不一致问题
$ sudo  apt install -y kubelet=1.18.2-00  
$ sudo  apt install -y kubeadm=1.18.2-00
$ sudo  apt install -y kubectl=1.18.2-00
// 保持版本取消自动更新，这里也可以省略
$ sudo apt-mark hold kubelet kubeadm kubectl
```

## kubeadm 初始化

```bash
$ sudo kubeadm init --image-repository registry.aliyuncs.com/google_containers --kubernetes-version v1.18.2 --pod-network-cidr=10.240.0.0/16
```

等待出现以下信息，则说明初始化成功

```bash
// ...
Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 机器IP:6443 --token q1guce.z76o2a2bb65vhd0u \
    --discovery-token-ca-cert-hash sha256:2a57a27853c66d608bc544742b57602a21d47c3d09fe58eef15258946d4341c0

```

## 配置非 root 的操作，便于普通用户操作集群

```bash
$ mkdir -p $HOME/.kube
$ sudo  cp -i /etc/kubernetes/admin.conf $HOME/.kube/config

$ sudo  chown  $(id -u):$(id -g)  $HOME/.kube/config
```

## coredns 问题解决

这时候node 状态还是NotReady，因为网络插件还没有安装，这里安装calico

```bash
$ kubectl apply -f https://docs.projectcalico.org/v3.10/manifests/calico.yaml
```

安装成功后结果如下

```bash
$ kubectl get node
NAME                    STATUS   ROLES    AGE   VERSION
sfeng-virtual-machine   Ready    master   28m   v1.18.2
$ kubectl get pods --all-namespaces -o wide
NAMESPACE     NAME                                            READY   STATUS    RESTARTS   AGE   IP                NODE                    NOMINATED NODE   READINESS GATES
kube-system   calico-kube-controllers-57546b46d6-hfgx2        1/1     Running   0          23m   192.168.109.129   sfeng-virtual-machine   <none>           <none>
kube-system   calico-node-kpx4p                               1/1     Running   0          23m   192.168.57.23     sfeng-virtual-machine   <none>           <none>
kube-system   coredns-7ff77c879f-gjgjb                        1/1     Running   0          28m   192.168.109.130   sfeng-virtual-machine   <none>           <none>
kube-system   coredns-7ff77c879f-qq6pz                        1/1     Running   0          28m   192.168.109.131   sfeng-virtual-machine   <none>           <none>
kube-system   etcd-sfeng-virtual-machine                      1/1     Running   0          28m   192.168.57.23     sfeng-virtual-machine   <none>           <none>
kube-system   kube-apiserver-sfeng-virtual-machine            1/1     Running   0          28m   192.168.57.23     sfeng-virtual-machine   <none>           <none>
kube-system   kube-controller-manager-sfeng-virtual-machine   1/1     Running   0          28m   192.168.57.23     sfeng-virtual-machine   <none>           <none>
kube-system   kube-proxy-jzfts                                1/1     Running   0          28m   192.168.57.23     sfeng-virtual-machine   <none>           <none>
kube-system   kube-scheduler-sfeng-virtual-machine            1/1     Running   0          28m   192.168.57.23     sfeng-virtual-machine   <none>           <none>
```

## 去掉master污点

```bash
// 这样master就能作为计算节点了哈，不然部署单机也没有意义
$ kubectl taint nodes --all node-role.kubernetes.io/master-
```