---
layout:     post
title:      "二进制部署kubernetes"
description: "二进制部署kubernetes"
date:     2021-12-10
author: sfeng
categories: ["Tech", "cloudNative", "ops"]
tags: ["kubernetes"]
URL: "/2021/12/10/kubernetes-install-binary/"
---

## 简介

部署 `k8s` 有多种方式，下面我们采取二进制的部署方式来部署 `k8s` 集群，二进制部署麻烦点，但是可以在我们通过部署各个组件的时候，也能让我们更好的深入了解组件之间的关联，更加了解原理。

## 软件版本

- `k8s 1.19.8`
- `docker 19-ce`
- `etcd 3.4.3`
- `calico 3.10.1-2`

## 主机规划

我们这里使用四台centos7.5 虚拟机，具体信息如下：

服务器内核建议升级，不升级也不妨碍安装

| 系统类型 | IP | 节点角色 | 机器配置 | 主机名 | 安装软件 |
| --- | --- | --- | --- | --- | --- |
| CentOS7.5/4.14.9 | 172.25.120.17 | master | ≥2，≥4G | master-1 | docker、kubelet、kube-apiserver、kube-controller-manager、kube-scheduler、etcd、kube-proxy、haproxy、keepalived |
| CentOS7.5/4.14.9 | 172.25.120.18 | master | ≥2，≥4G | master-2 | docker、kubelet、kube-apiserver、kube-controller-manager、kube-scheduler、etcd、kube-proxy、haproxy、keepalived |
| CentOS7.5/4.14.9 | 172.25.120.19 | master | ≥2，≥4G | master-3 | docker、kubelet、kube-apiserver、kube-controller-manager、kube-scheduler、etcd、kube-proxy、haproxy、keepalived |
| CentOS7.5/4.14.9 | 172.25.120.20 | worker | ≥2，≥4G | node-1 | kubelet、kube-proxy、docker |
| // | 172.25.120.1 | VIP |  |  |  |

## 配置策略

### kube-apiserver

- 使用节点本地 nginx 4 层透明代理实现高可用；
- 关闭非安全端口 8080 和匿名访问；
- 在安全端口 6443 接收 https 请求；
- 严格的认证和授权策略 (x509、token、RBAC)；
- 开启 bootstrap token 认证，支持 kubelet TLS bootstrapping；
- 使用 https 访问 kubelet、etcd，加密通信；

### kube-controller-manager

- 3 节点高可用；
- 关闭非安全端口 10252，在安全端口 10257 接收 https 请求；
- 使用 kubeconfig 访问 apiserver 的安全端口；
- 自动 approve kubelet 证书签名请求 (CSR)，证书过期后自动轮转；
- 各 controller 使用自己的 ServiceAccount 访问 apiserver；

### kube-scheduler

- 3 节点高可用；
- 关闭非安全端口 10251，在安全端口 10259 接收 https 请求；
- 使用 kubeconfig 访问 apiserver 的安全端口；

### kubelet

- 使用 kubeadm 动态创建 bootstrap token，而不是在 apiserver 中静态配置；
- 使用 TLS bootstrap 机制自动生成 client 和 server 证书，过期后自动轮转；
- 在 KubeletConfiguration 类型的 JSON 文件配置主要参数；
- 关闭只读端口 10255，在安全端口 10250 接收 https 请求，对请求进行认证和授权，拒绝匿名访问和非授权访问；
- 使用 kubeconfig 访问 apiserver 的安全端口；

### kube-proxy

- 使用 kubeconfig 访问 apiserver 的安全端口；
- 在 KubeProxyConfiguration 类型的 JSON 文件配置主要参数；
- 使用 ipvs 代理模式；

## 主机环境初始化

在所有节点上操作

```bash
#更改主机名hostnamectl set-hostname masterhostnamectl set-hostname node-1

$ hostnamectl set-hostname master-1

#关闭防火墙
$ systemctl stop firewalld ; systemctl disable firewalld
#关闭selinux
$ setenforce 0 ;sed -i 's/enforcing/disabled/' /etc/selinux/config
#关闭swap分区
$ swapoff -a ; sed -ri 's/.*swap.*/#&/' /etc/fstab
#添加hosts
$ cat >> /etc/hosts << EOF
172.25.120.17 master-1
172.25.120.18 master-2
172.25.120.19 master-3
172.25.120.20 node-1
EOF
#添加防火墙转发
$ cat > /etc/sysctl.d/k8s.conf << EOF
net.ipv4.ip_forward = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
$ modprobe br_netfilter
$ sysctl --system ##生效
#时间同步
$ yum install -y ntpdate  ##安装时间同步工具
$ ntpdate time.windows.com  #同步windwos时间服务器#磁盘分区，建议由数据盘的首先给/var/lib/docker做个lvm分区
```

在 `master-1` 节点操作，用于免密

```bash
#生成秘钥对
$ ssh-keygen -t rsa

#将公钥拷贝至每台主机
$ ssh-copy-id root@master-1
$ ssh-copy-id root@master-2
$ ssh-copy-id root@master-3
$ ssh-copy-id root@node-1
```

## 部署etcd集群

`Etcd` 是一个分布式键值存储系统，`Kubernetes`使用`Etcd`进行数据存储

### 签发 etcd 证书

1、安装 `cfssl`

`cfssl`是一个开源的证书管理工具，使用`json`文件生成证书，相比`openssl`更方便使用。

在`master`上操作:

```bash
##获取证书管理工具
$ wget https://pkg.cfssl.org/R1.2/cfssl_linux-amd64
$ wget https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64
$ wget https://pkg.cfssl.org/R1.2/cfssl-certinfo_linux-amd64
##添加看执行权限并放进可执行目录
$ chmod +x cfssl_linux-amd64 cfssljson_linux-amd64 cfssl-certinfo_linux-amd64
$ mv cfssl_linux-amd64 /usr/local/bin/cfssl
$ mv cfssljson_linux-amd64 /usr/local/bin/cfssljson
$ mv cfssl-certinfo_linux-amd64 /usr/bin/cfssl-certinfo
```

2、生成Etcd证书，先签发CA

创建证书目录

```bash
$ mkdir -p ~/TLS/{etcd,k8s} 
$ cd ~/TLS/etcd  ##进入证书目录
```

自签CA

```bash
cat > ca-config.json << EOF
{
  "signing": {
    "default": {
      "expiry": "87600h"
    },
    "profiles": {
      "www": {
         "expiry": "87600h",
         "usages": [
            "signing",
            "key encipherment",
            "server auth",
            "client auth"
        ]
      }
    }
  }
}
EOF

cat > ca-csr.json << EOF
{
    "CN": "etcd CA",
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "L": "Beijing",
            "ST": "Beijing"
        }
    ]
}
EOF
```

生成CA

```bash
$ cfssl gencert -initca ca-csr.json | cfssljson -bare ca -
$ ls *pem 
ca-key.pem  ca.pem
```

3、使用自签CA签发Etcd HTTPS证书

这里为了方便，etcd peer，client，server使用同一套证书

创建证书申请文件：

```bash
cat > etcd-csr.json << EOF
{
    "CN": "etcd",
    "hosts": [
    "172.25.120.17",
    "172.25.120.18",
    "172.25.120.19"
    ],
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "L": "BeiJing",
            "ST": "BeiJing"
        }
    ]
}
EOF
```

生成 `etcd`全套证书:

```bash
$ cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=www etcd-csr.json | cfssljson -bare etcd
$ ls server*pem  ##可以看到生成了一个key，一个证书
etcd-key.pem  etcd.pem
```

### 下载etcd二进制文件

文件地址:`https://github.com/etcd-io/etcd/releases/download/v3.4.9/etcd-v3.4.9-linux-amd64.tar.gz`

以下操作在`master-1`上操作

### 启动etcd

创建工作目录并解压文件

```bash
$ mkdir /home/k8s/etcd/{bin,cfg,ssl} -p
$ tar zxvf etcd-v3.4.9-linux-amd64.tar.gz
$ mv etcd-v3.4.9-linux-amd64/{etcd,etcdctl} /home/k8s/etcd/bin/
```

创建etcd配置文件

```bash
$ cat > /home/k8s/etcd/cfg/etcd.conf << EOF
#[Member]
ETCD_NAME="etcd-1"
ETCD_DATA_DIR="/var/lib/etcd/default.etcd"
ETCD_LISTEN_PEER_URLS="https://172.25.120.17:2380"
ETCD_LISTEN_CLIENT_URLS="https://172.25.120.17:2379"
#[Clustering]
ETCD_INITIAL_ADVERTISE_PEER_URLS="https://172.25.120.17:2380"
ETCD_ADVERTISE_CLIENT_URLS="https://172.25.120.17:2379"
ETCD_INITIAL_CLUSTER="etcd-1=https://172.25.129.17:2380,etcd-2=https://172.25.120.18:2380,etcd-3=https://172.25.120.19:2380"
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster"
ETCD_INITIAL_CLUSTER_STATE="new"
EOF
```

参数详解:

- ETCD_DATA_DIR：数据目录
- ETCD_LISTEN_PEER_URLS：集群通信监听地址
- ETCD_LISTEN_CLIENT_URLS：客户端访问监听地址
- ETCD_INITIAL_ADVERTISE_PEER_URLS：集群通告地址
- ETCD_ADVERTISE_CLIENT_URLS：客户端通告地址
- ETCD_INITIAL_CLUSTER：集群节点地址
- ETCD_INITIAL_CLUSTER_TOKEN：集群Token
- ETCD_INITIAL_CLUSTER_STATE：加入集群的当前状态，new是新集群，existing表示加入已有集群

配置systemd管理etcd

```bash
$ cat > /usr/lib/systemd/system/etcd.service << EOF
[Unit]
Description=Etcd Server
After=network.target
After=network-online.target
Wants=network-online.target
[Service]
Type=notify
EnvironmentFile=/home/k8s/etcd/cfg/etcd.conf
ExecStart=/home/k8s/etcd/bin/etcd \
--cert-file=/home/k8s/etcd/ssl/server.pem \
--key-file=/home/k8s/etcd/ssl/server-key.pem \
--peer-cert-file=/home/k8s/etcd/ssl/server.pem \
--peer-key-file=/home/k8s/etcd/ssl/server-key.pem \
--trusted-ca-file=/home/k8s/etcd/ssl/ca.pem \
--peer-trusted-ca-file=/home/k8s/etcd/ssl/ca.pem \
--logger=zap
Restart=on-failure
LimitNOFILE=65536
[Install]
WantedBy=multi-user.target
EOF
```

把刚才生成的证书拷贝到配置文件中的路径：

```bash
cp ~/TLS/etcd/ca*pem ~/TLS/etcd/server*pem /home/k8s/etcd/ssl/
```

将 master-1 生成的所有文件拷贝到master-2，master-3

```bash
$ scp -r /home/k8s/etcd/ master-2:/home/k8s/
$ scp /usr/lib/systemd/system/etcd.service master-2:/usr/lib/systemd/system/
$ scp -r /home/k8s/etcd/ master-3:/home/k8s/
$ scp /usr/lib/systemd/system/etcd.service master-3:/usr/lib/systemd/system/
```

在master-2，master-3分别修改 `etcd.conf` 配置文件中的节点名称和当前服务器IP

```bash
$ sed -i '4,8s/172.25.120.17/172.25.120.18/' /home/k8s/etcd/cfg/etcd.conf  ; sed -i '2s/etcd-1/etcd-2/'  /home/k8s/etcd/cfg/etcd.conf 
$ sed -i '4,8s/172.25.120.17/172.25.120.19/' /home/k8s/etcd/cfg/etcd.conf  ; sed -i '2s/etcd-1/etcd-3/'  /home/k8s/etcd/cfg/etcd.conf 
```

启动3个master节点的etcd并加入开机自启，在三各节点操作

```bash
$ systemctl daemon-reload
$ systemctl start etcd
$ systemctl enable etcd
```

查看etcd集群状态

```bash
$ ETCDCTL_API=3 /home/k8s/etcd/bin/etcdctl --cacert=/home/k8s/etcd/ssl/ca.pem --cert=/home/k8s/etcd/ssl/server.pem --key=/home/k8s/etcd/ssl/server-key.pem --endpoints="https://172.25.120.17:2379,https://172.25.120.18:2379,https://172.25.120.19:2379" endpoint health
https://172.25.120.18:2379 is healthy: successfully committed proposal: took = 14.194738ms
https://172.25.120.19:2379 is healthy: successfully committed proposal: took = 14.97292ms
https://172.25.120.17:2379 is healthy: successfully committed proposal: took = 14.847968ms
```

出现successfully，表示etcd部署成功，如果有异常情况可以使用`systemctl stautus etcd -l`进一步查看报错信息

## 安装Docker

可以使用yum安装，这次我们采用二进制的方式

以下所有操作在所有节点

获取docker安装包

```bash
$ wget https://download.docker.com/linux/static/stable/x86_64/docker-19.03.9.tgz
```

解压docker二进制包

```bash
$ tar zxvf docker-19.03.9.tgz
$ mv docker/* /usr/bin
```

配置systemd管理docker

```bash
$ cat > /usr/lib/systemd/system/docker.service << EOF
[Unit]
Description=Docker Application Container Engine
Documentation=https://docs.docker.com
After=network-online.target firewalld.service
Wants=network-online.target
[Service]
Type=notify
ExecStart=/usr/bin/dockerd
ExecReload=/bin/kill -s HUP $MAINPID
LimitNOFILE=infinity
LimitNPROC=infinity
LimitCORE=infinity
TimeoutStartSec=0
Delegate=yes
KillMode=process
Restart=on-failure
StartLimitBurst=3
StartLimitInterval=60s
[Install]
WantedBy=multi-user.target
EOF
```

配置docker加速器

```bash
$ mkdir /etc/docker
$ cat > /etc/docker/daemon.json << EOF
 {
"registry-mirrors": ["https://jo6348gu.mirror.aliyuncs.com"]
 }
EOF
```

启动docker并加入开机自启

```bash
$ systemctl daemon-reload 
$ systemctl start docker
$ systemctl enable docker
```

## 部署master

以下操作在master上

### 部署 kube-scheduler

生成kube-apiserver证书(这里再自建一个CA，没有复用前面的etcd ca)

自签证书颁发机构（CA）

```bash
$ cd TLS/k8s
$ cat > ca-config.json << EOF
{
  "signing": {
    "default": {
      "expiry": "87600h"
    },
    "profiles": {
      "kubernetes": {
         "expiry": "87600h",
         "usages": [
            "signing",
            "key encipherment",
            "server auth",
            "client auth"
        ]
      }
    }
  }
}
EOF
$ cat > ca-csr.json << EOF
{
    "CN": "kubernetes",
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "L": "Beijing",
            "ST": "Beijing",
            "O": "k8s",
            "OU": "System"
        }
    ]
}
EOF
```

生成CA

```bash
$ cfssl gencert -initca ca-csr.json | cfssljson -bare ca -
# ls *pem
ca-key.pem  ca.pem  
```

使用自签CA签发kube-apiserver HTTPS证书

创建证书申请文件：

```bash
# hosts: 表示访问api-server 的各个方式，所以需要对每个访问方式都要签证，比如10.0.0.1 是api-server的svc地址
$ cat > server-csr.json << EOF
{
    "CN": "kubernetes",
    "hosts": [
      "10.0.0.1", 
      "127.0.0.1",
      "172.25.120.17",
      "172.25.120.18",
      "172.25.120.19",
      "kubernetes",
      "kubernetes.default",
      "kubernetes.default.svc",
      "kubernetes.default.svc.cluster",
      "kubernetes.default.svc.cluster.local"
    ],
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "L": "BeiJing",
            "ST": "BeiJing",
            "O": "k8s",
            "OU": "System"
        }
    ]
}
EOF
```

生成证书:

```bash
$ cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes server-csr.json | cfssljson -bare server
$ ls server*pem
server-key.pem  server.pem 
```

获取二进制包

```bash
$ wget  https://dl.k8s.io/v1.19.8/kubernetes-server-linux-amd64.tar.gz
```

解压，每台master 节点上操作

```bash
$ mkdir -p /home/k8s/kubernetes/{bin,cfg,ssl,logs} 
$ tar zxvf kubernetes-server-linux-amd64.tar.gz
$ cd kubernetes/server/bin
$ cp kube-apiserver kube-scheduler kube-controller-manager /home/k8s/kubernetes/bin
$ cp kubectl /usr/bin/
```

### 启动kube-apiserver

创建配置文件

```bash
$ cat > /home/k8s/kubernetes/cfg/kube-apiserver.conf << EOF
KUBE_APISERVER_OPTS="--logtostderr=false \\
--v=4 \\
--log-dir=/home/k8s/kubernetes/logs \\
--etcd-servers=https://172.25.120.17:2379,https://172.25.120.18:2379,https://172.25.120.19:2379 \\
--bind-address=172.25.120.17 \\
--secure-port=6443 \\
--advertise-address=172.25.120.17 \\
--allow-privileged=true \\
--service-cluster-ip-range=10.0.0.0/24 \\
--enable-admission-plugins=NamespaceLifecycle,LimitRanger,ServiceAccount,ResourceQuota,NodeRestriction \\
--authorization-mode=RBAC,Node \\
--enable-bootstrap-token-auth=true \\
--token-auth-file=/home/k8s/kubernetes/cfg/token.csv \\
--service-node-port-range=30000-32767 \\
--kubelet-client-certificate=/home/k8s/kubernetes/ssl/server.pem \\
--kubelet-client-key=/home/k8s/kubernetes/ssl/server-key.pem \\
--tls-cert-file=/home/k8s/kubernetes/ssl/server.pem  \\
--tls-private-key-file=/home/k8s/kubernetes/ssl/server-key.pem \\
--client-ca-file=/home/k8s/kubernetes/ssl/ca.pem \\
--service-account-key-file=/home/k8s/kubernetes/ssl/ca-key.pem \\
--etcd-cafile=/home/k8s/etcd/ssl/ca.pem \\
--etcd-certfile=/home/k8s/etcd/ssl/server.pem \\
--etcd-keyfile=/home/k8s/etcd/ssl/server-key.pem \\
--audit-log-maxage=30 \\
--audit-log-maxbackup=3 \\
--audit-log-maxsize=100 \\
--audit-log-path=/home/k8s/kubernetes/logs/k8s-audit.log"
EOF
```

- 参数详解:
- –logtostderr：启用日志
- —v：日志等级
- –log-dir：日志目录
- –etcd-servers：etcd集群地址
- –bind-address：监听地址
- –secure-port：https安全端口
- –advertise-address：集群通告地址
- –allow-privileged：启用授权
- –service-cluster-ip-range：Service虚拟IP地址段
- –enable-admission-plugins：准入控制模块
- –authorization-mode：认证授权，启用RBAC授权和节点自管理
- –enable-bootstrap-token-auth：启用TLS bootstrap机制
- –token-auth-file：bootstrap token文件
- –service-node-port-range：Service nodeport类型默认分配端口范围
- –kubelet-client-xxx：apiserver访问kubelet客户端证书
- –tls-xxx-file：apiserver https证书
- –etcd-xxxfile：连接Etcd集群证书
- –audit-log-xxx：审计日志

把刚才生成的证书拷贝到配置文件中的路径：

```bash
$ cp ~/TLS/k8s/ca*pem ~/TLS/k8s/server*pem /home/k8s/kubernetes/ssl/
```

启用 TLS Bootstrapping 机制

TLS Bootstraping：Master apiserver启用TLS认证后，Node节点kubelet和kube-proxy要与kube-apiserver进行通信，必须使用CA签发的有效证书才可以，当Node节点很多时，这种客户端证书颁发需要大量工作，同样也会增加集群扩展复杂度。为了简化流程，Kubernetes引入了TLS bootstraping机制来自动颁发客户端证书，kubelet会以一个低权限用户自动向apiserver申请证书，kubelet的证书由apiserver动态签署。所以强烈建议在Node上使用这种方式，目前主要用于kubelet，kube-proxy还是由我们统一颁发一个证书。**该功能当前仅支持为kubelet生成证书**

TLS bootstraping 工作流程：

![http://upload-images.jianshu.io/upload_images/21497969-fa0aa22ba60f64ea.png?imageMogr2/auto-orient/strip|imageView2/2/w/676/format/webp](http://upload-images.jianshu.io/upload_images/21497969-fa0aa22ba60f64ea.png?imageMogr2/auto-orient/strip|imageView2/2/w/676/format/webp)

创建上述配置文件中token文件：

```bash
# 这里使用用户名，不要使用UID
$ cat > /home/k8s/kubernetes/cfg/token.csv << EOF
b1dc586d69159ff4e3ef7efa9db60e48,kubelet-bootstrap,"system:node-bootstrapper"
EOF
```

格式：token，用户名，用户组token也可自行生成替换：

```bash
$ head -c 16 /dev/urandom | od -An -t x | tr -d ' '
```

systemd管理apiserver

```bash
$ cat > /usr/lib/systemd/system/kube-apiserver.service << EOF
[Unit]
Description=Kubernetes API Server
Documentation=https://github.com/kubernetes/kubernetes
[Service]
EnvironmentFile=/home/k8s/kubernetes/cfg/kube-apiserver.conf
ExecStart=/home/k8s/kubernetes/bin/kube-apiserver \$KUBE_APISERVER_OPTS
Restart=on-failure
[Install]
WantedBy=multi-user.target
EOF
```

同步apiserver 相关文件到 master-2、master-3 节点

```bash
# 同步 token.csv
scp /home/k8s/kubernetes/cfg/token.csv master-2:/home/k8s/kubernetes/cfg/
scp /home/k8s/kubernetes/cfg/token.csv master-3:/home/k8s/kubernetes/cfg/
# 同步 apiserver 证书
scp /home/k8s/kubernetes/ssl/server*.pem master-2:/home/k8s/kubernetes/ssl/
scp /home/k8s/kubernetes/ssl/server*.pem master-3:/home/k8s/kubernetes/ssl/
# 同步 ca 证书
scp /home/k8s/kubernetes/ssl/ca*.pem master-2:/home/k8s/kubernetes/ssl/
scp /home/k8s/kubernetes/ssl/ca*.pem master-2:/home/k8s/kubernetes/ssl/
# 同步 apiserver 启动配置参数文件
scp /home/k8s/kubernetes/cfg/kube-apiserver.conf/kube-apiserver.conf master-2:/home/k8s/kubernetes/cfg/
scp /home/k8s/kubernetes/cfg/kube-apiserver.conf/kube-apiserver.conf master-3:/home/k8s/kubernetes/cfg/
# 同步 apiserver systemd 管理文件
scp /usr/lib/systemd/system/kube-apiserver.service master-2:/usr/lib/systemd/system/
scp /usr/lib/systemd/system/kube-apiserver.service master-3:/usr/lib/systemd/system/
```

在master-2，master-3分别修改 `kube-apiserver.conf` 配置文件中的节点名称和当前服务器IP

```bash
$ sed -i '5,7s/172.25.120.17/172.25.120.18/' /home/k8s/kubernetes/cfg/kube-apiserver.conf  
$ sed -i '5,7s/172.25.120.17/172.25.120.19/' /home/k8s/kubernetes/cfg/kube-apiserver.conf
```

启动并设置开机启动

```bash
$ systemctl daemon-reload
$ systemctl start kube-apiserver
$ systemctl enable kube-apiserver
```

授权kubelet-bootstrap用户允许请求证书

```bash
$ kubectl create clusterrolebinding kubelet-bootstrap \
--clusterrole=system:node-bootstrapper \
--user=kubelet-bootstrap
```

### 部署haproxy和keepalived 高可用

安装haproxy、keepalived

```bash
$ yum install keepalived haproxy -y
```

配置haproxy

```bash
cat >/etc/haproxy/haproxy.cfg<<"EOF"
global
 maxconn 2000
 ulimit-n 16384
 log 127.0.0.1 local0 err
 stats timeout 30s

defaults
 log global
 mode http
 option httplog
 timeout connect 5000
 timeout client 50000
 timeout server 50000
 timeout http-request 15s
 timeout http-keep-alive 15s

frontend monitor-in
 bind *:33305
 mode http
 option httplog
 monitor-uri /monitor

frontend k8s-master
 bind 0.0.0.0:16443
 bind 127.0.0.1:16443
 mode tcp
 option tcplog
 tcp-request inspect-delay 5s
 default_backend k8s-master

backend k8s-master
 mode tcp
 option tcplog
 option tcp-check
 balance roundrobin
 default-server inter 10s downinter 5s rise 2 fall 2 slowstart 60s maxconn 250 maxqueue 256 weight 100
 server  master-1  172.25.120.17:6443 check
 server  master-2  172.25.120.18:6443 check
 server  master-3  172.25.120.19:6443 check
EOF
```

配置keepalived

每个masrer配置不一样，注意区分

```bash
#master-1 配置：
cat >/etc/keepalived/keepalived.conf<<"EOF"
! Configuration File for keepalived
global_defs {
   router_id LVS_DEVEL
script_user root
   enable_script_security
}
vrrp_script chk_apiserver {
   script "/etc/keepalived/check_apiserver.sh"
   interval 5
   weight -5
   fall 2
rise 1
}
vrrp_instance VI_1 {
   state MASTER
   interface ens160
   mcast_src_ip 172.25.120.17
   virtual_router_id 51
   priority 100
   advert_int 2
   authentication {
       auth_type PASS
       auth_pass K8SHA_KA_AUTH
   }
   virtual_ipaddress {
       172.25.120.1
   }
   track_script {
      chk_apiserver
   }
}
EOF

#Master-2 配置：
cat >/etc/keepalived/keepalived.conf<<"EOF"
! Configuration File for keepalived
global_defs {
   router_id LVS_DEVEL
script_user root
   enable_script_security
}
vrrp_script chk_apiserver {
   script "/etc/keepalived/check_apiserver.sh"
  interval 5
   weight -5
   fall 2
rise 1
}
vrrp_instance VI_1 {
   state BACKUP
   interface ens160
   mcast_src_ip 172.25.120.18
   virtual_router_id 51
   priority 99
   advert_int 2
   authentication {
       auth_type PASS
       auth_pass K8SHA_KA_AUTH
   }
   virtual_ipaddress {
       172.25.120.1
   }
   track_script {
      chk_apiserver
   }
}
EOF

#Master-3 配置：
cat >/etc/keepalived/keepalived.conf<<"EOF"
! Configuration File for keepalived
global_defs {
   router_id LVS_DEVEL
script_user root
   enable_script_security
}
vrrp_script chk_apiserver {
   script "/etc/keepalived/check_apiserver.sh"
 interval 5
   weight -5
   fall 2
rise 1
}
vrrp_instance VI_1 {
   state BACKUP
   interface ens160
   mcast_src_ip 172.25.120.19
   virtual_router_id 51
   priority 98
   advert_int 2
   authentication {
       auth_type PASS
       auth_pass K8SHA_KA_AUTH
   }
   virtual_ipaddress {
       172.25.120.1
   }
    track_script {
      chk_apiserver
   }
EOF
```

配置健康检查脚本

```bash
cat > /etc/keepalived/check_apiserver.sh <<"EOF"
#!/bin/bash
err=0
for k in $(seq 1 3)
do
   check_code=$(pgrep haproxy)
   if [[ $check_code == "" ]]; then
       err=$(expr $err + 1)
       sleep 1
       continue
   else
       err=0
       break
   fi
done

if [[ $err != "0" ]]; then
   echo "systemctl stop keepalived"
   /usr/bin/systemctl stop keepalived
   exit 1
else
   exit 0
fi
EOF

chmod u+x /etc/keepalived/check_apiserver.sh
```

启动服务

```bash
$ systemctl daemon-reload
$ systemctl enable --now haproxy
$ systemctl enable --now keepalived
```

### 部署kubectl

创建 csr 文件

```bash
$ cat > cat > /home/k8s/kubernetes/ssl/admin-csr.json << "EOF"
{
  "CN": "admin",
  "hosts": [],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "Hefei",
      "L": "shiyan",
      "O": "system:masters",             
      "OU": "system"
    }
  ]
}
EOF
```

> 说明：
后续 kube-apiserver 使用 RBAC 对客户端(如 kubelet、kube-proxy、Pod)请求进行授权；
kube-apiserver 预定义了一些 RBAC 使用的 RoleBindings，如 cluster-admin 将 Group system:masters 与 Role cluster-admin 绑定，该 Role 授予了调用kube-apiserver 的所有 API的权限；
O指定该证书的 Group 为 system:masters，kubelet 使用该证书访问 kube-apiserver 时 ，由于证书被 CA 签名，所以认证通过，同时由于证书用户组为经过预授权的 system:masters，所以被授予访问所有 API 的权限；
注：
这个admin 证书，是将来生成管理员用的kube config 配置文件用的，现在我们一般建议使用RBAC 来对kubernetes 进行角色权限控制， kubernetes 将证书中的CN 字段 作为User， O 字段作为 Group；
"O": "system:masters", 必须是system:masters，否则后面kubectl create clusterrolebinding报错。
>

生成证书

```bash
$ cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes admin-csr.json | cfssljson -bare admin
$ cp admin*.pem /etc/kubernetes/ssl/
```



生成kubeconfig

kube.config 为 kubectl 的配置文件，包含访问 apiserver 的所有信息，如 apiserver 地址、CA 证书和自身使用的证书

```bash
$ kubectl config set-cluster kubernetes --certificate-authority=ca.pem --embed-certs=true --server=https://172.25.120.1:16443 --kubeconfig=kube.config

$ kubectl config set-credentials admin --client-certificate=admin.pem --client-key=admin-key.pem --embed-certs=true --kubeconfig=kube.config

$ kubectl config set-context kubernetes --cluster=kubernetes --user=admin --kubeconfig=kube.config

$ kubectl config use-context kubernetes --kubeconfig=kube.config

$ mkdir ~/.kube
$ cp kube.config ~/.kube/config
$ kubectl create clusterrolebinding kube-apiserver:kubelet-apis --clusterrole=system:kubelet-api-admin --user kubernetes --kubeconfig=~/.kube/config
```

同步 kubectl 到其他master 节点

```bash
$ scp /root/.kube/config master-2:/root/.kube/
$ scp /root/.kube/config master-3:/root/.kube/
```

### 部署kube-controller-manager

创建csr 请求文件

```bash
$ cat > /home/k8s/kubernetes/ssl/kube-controller-manager-csr.json << "EOF"
{
    "CN": "system:kube-controller-manager",
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "hosts": [
      "127.0.0.1",
      "172.25.120.17",
      "172.25.120.18",
      "172.25.120.19",
    ],
    "names": [
      {
        "C": "CN",
        "ST": "Hefei",
        "L": "shiyan",
        "O": "system:kube-controller-manager",
        "OU": "system"
      }
    ]
}
EOF
```

> hosts 列表包含所有 kube-controller-manager 节点 IP；
CN 为 system:kube-controller-manager、O 为 system:kube-controller-manager，kubernetes 内置的 ClusterRoleBindings system:kube-controller-manager 赋予 kube-controller-manager 工作所需的权限
>

生成证书

```bash
$ cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes kube-controller-manager-csr.json | cfssljson -bare kube-controller-manager
```

生成controller-manager的kubeconfig

```bash
$ cd /home/k8s/kubernetes/cfg
$ kubectl config set-cluster kubernetes --certificate-authority=ca.pem --embed-certs=true --server=https://172.25.120.1:16443 --kubeconfig=kube-controller-manager.kubeconfig

$ kubectl config set-credentials system:kube-controller-manager --client-certificate=kube-controller-manager.pem --client-key=kube-controller-manager-key.pem --embed-certs=true --kubeconfig=kube-controller-manager.kubeconfig

$ kubectl config set-context system:kube-controller-manager --cluster=kubernetes --user=system:kube-controller-manager --kubeconfig=kube-controller-manager.kubeconfig

kubectl config use-context system:kube-controller-manager --kubeconfig=kube-controller-manager.kubeconfig
```

创建配置文件

```bash
$ cat > /home/k8s/kubernetes/cfg/kube-controller-manager.conf << EOF
KUBE_CONTROLLER_MANAGER_OPTS="--allocate-node-cidrs=true \\
    - --authentication-kubeconfig=/home/k8s/kubernetes/cfg/controller-manager.conf \\
    - --authorization-kubeconfig=/home/k8s/kubernetes/cfg/controller-manager.conf \\
    - --bind-address=172.25.120.17 \\
    - --client-ca-file=/home/k8s/kubernetes/ssl/ca.crt \\
    - --cluster-cidr=100.64.0.0/10 \\
    - --cluster-name=kubernetes \\
    - --cluster-signing-cert-file=/home/k8s/kubernetes/ssl/ca.crt \\
    - --cluster-signing-key-file=/home/k8s/kubernetes/ssl/ca.key \\
    - --controllers=*,bootstrapsigner,tokencleaner \\
    - --experimental-cluster-signing-duration=876000h \\
    - --feature-gates=TTLAfterFinished=true,EphemeralContainers=true \\
    - --kubeconfig=/home/k8s/kubernetes/cfg/controller-manager.conf \\
    - --leader-elect=true \\
    - --node-cidr-mask-size=24 \\
    - --port=0 \\
    - --root-ca-file=/home/k8s/kubernetes/ssl/ca.crt \\
    - --service-cluster-ip-range=10.96.0.0/22 \\
    - --use-service-account-credentials=true \\
EOF
```

systemd管理controller-manager

```bash
$ cat > /usr/lib/systemd/system/kube-controller-manager.service << EOF
[Unit]
Description=Kubernetes Controller Manager
Documentation=https://github.com/kubernetes/kubernetes
[Service]
EnvironmentFile=/home/k8s/kubernetes/cfg/kube-controller-manager.conf
ExecStart=/home/k8s/kubernetes/bin/kube-controller-manager \$KUBE_CONTROLLER_MANAGER_OPTS
Restart=on-failure
[Install]
WantedBy=multi-user.target
EOF
```

同步 kube-controller-manager 相关文件到master-2、master-3

```bash
scp kube-controller-manager.kubeconfig kube-controller-manager.conf master-2:/home/k8s/kubernetes/cfg/
scp kube-controller-manager.kubeconfig kube-controller-manager.conf master-3:/home/k8s/kubernetes/cfg/
scp /usr/lib/systemd/system/kube-controller-manager.service master-2:/usr/lib/systemd/system/
scp /usr/lib/systemd/system/kube-controller-manager.service master-3:/usr/lib/systemd/system/
```

在master-2，master-3分别修改 `kube-controller-manager.conf` 配置文件中的节点名称和当前服务器IP

```bash
$ sed -i '4s/172.25.120.17/172.25.120.18/' /home/k8s/kubernetes/cfg/kube-controller-manager.conf 
$ sed -i '4s/172.25.120.17/172.25.120.19/' /home/k8s/kubernetes/cfg/kube-controller-manager.conf
```

启动并设置开机启动

```bash
$ systemctl daemon-reload
$ systemctl start kube-controller-manager
$ systemctl enable kube-controller-manager
```

### 部署kube-scheduler

创建csr请求文件

```bash
$ cat > /home/k8s/kubernetes/ssl/kube-scheduler-csr.json << "EOF"
{
    "CN": "system:kube-scheduler",
    "hosts": [
      "127.0.0.1",
      "172.25.120.17",
      "172.25.120.18",
      "172.25.120.19",
    ],
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
      {
        "C": "CN",
        "ST": "Hefei",
        "L": "shiyan",
        "O": "system:kube-scheduler",
        "OU": "system"
      }
    ]
}
EOF
```

生成证书

```bash
$ cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes kube-scheduler-csr.json | cfssljson -bare kube-scheduler
```

创建kube-scheduler的kubeconfig

```bash
$ kubectl config set-cluster kubernetes --certificate-authority=ca.pem --embed-certs=true --server=https://172.25.120.1:16443 --kubeconfig=kube-scheduler.kubeconfig

$ kubectl config set-credentials system:kube-scheduler --client-certificate=kube-scheduler.pem --client-key=kube-scheduler-key.pem --embed-certs=true --kubeconfig=kube-scheduler.kubeconfig

$ kubectl config set-context system:kube-scheduler --cluster=kubernetes --user=system:kube-scheduler --kubeconfig=kube-scheduler.kubeconfig

$ kubectl config use-context system:kube-scheduler --kubeconfig=kube-scheduler.kubeconfig
```

创建配置文件

```bash
$ cat > /home/k8s/kubernetes/cfg/kube-scheduler.conf << EOF
KUBE_SCHEDULER_OPTS="--logtostderr=false \
--kubeconfig=/home/k8s/kubernetes/cfg/kube-scheduler.kubeconfig \
--v=2 \
--log-dir=/home/k8s/kubernetes/logs \
--leader-elect=true \
--bind-address=172.25.120.17"
EOF
```

systemd管理scheduler

```bash
$ cat > /usr/lib/systemd/system/kube-scheduler.service << EOF
[Unit]
Description=Kubernetes Scheduler
Documentation=https://github.com/kubernetes/kubernetes
[Service]
EnvironmentFile=/home/k8s/kubernetes/cfg/kube-scheduler.conf
ExecStart=/home/k8s/kubernetes/bin/kube-scheduler \$KUBE_SCHEDULER_OPTS
Restart=on-failure
[Install]
WantedBy=multi-user.target
EOF
```

同步 kube-scheduler 相关文件到master-2、master-3

```bash
scp kube-scheduler.kubeconfig kube-scheduler.conf master-2:/home/k8s/kubernetes/cfg/
scp kube-scheduler.kubeconfig kube-scheduler.conf master-3:/home/k8s/kubernetes/cfg/
scp /usr/lib/systemd/system/kube-scheduler.service master-2:/usr/lib/systemd/system/
scp /usr/lib/systemd/system/kube-scheduler.service master-3:/usr/lib/systemd/system/
```

启动并设置开机启动

```bash
$ systemctl daemon-reload
$ systemctl start kube-scheduler
$ systemctl enable kube-scheduler
```

查看集群状态

所有组件都已经启动成功，通过`kubectl get cs`命令查看当前集群组件状态：

```bash
$ kubectl get cs
NAME                 STATUS    MESSAGE             ERROR
controller-manager   Healthy   ok                  
scheduler            Healthy   ok                  
etcd-0               Healthy   {"health":"true"}   
etcd-1               Healthy   {"health":"true"}   
etcd-2               Healthy   {"health":"true"}
```

## 部署Worker Node

下面还是在master节点上操作，即同时作为Worker Node

### 部署kubelet

拷贝二进制文件

```bash
$ cd ~/kubernetes/server/bin
$ cp kubelet kube-proxy /home/k8s/kubernetes/bin
```

创建配置文件

```bash
$ cat > /home/k8s/kubernetes/cfg/kubelet.conf << EOF
KUBELET_OPTS="--logtostderr=false \\
--v=4 \\
--log-dir=/home/k8s/kubernetes/logs \\
--hostname-override=k8s-master \\
--network-plugin=cni \\
--kubeconfig=/home/k8s/kubernetes/cfg/kubelet.kubeconfig \\
--bootstrap-kubeconfig=/home/k8s/kubernetes/cfg/bootstrap.kubeconfig \\
--config=/home/k8s/kubernetes/cfg/kubelet-config.yml \\
--cert-dir=/home/k8s/kubernetes/ssl \\
--pod-infra-container-image=lizhenliang/pause-amd64:3.0"
EOF
```

参数详解:

- –hostname-override：显示名称，集群中唯一
- –network-plugin：启用CNI
- –kubeconfig：空路径，会自动生成，后面用于连接apiserver
- –bootstrap-kubeconfig：首次启动向apiserver申请证书
- –config：配置参数文件
- –cert-dir：kubelet证书生成目录
- –pod-infra-container-image：管理Pod网络容器的镜像

创建配置参数yaml文件

```bash
$ cat > /home/k8s/kubernetes/cfg/kubelet-config.yml << EOF
kind: KubeletConfiguration
apiVersion: kubelet.config.k8s.io/v1beta1
address: 0.0.0.0
port: 10250
readOnlyPort: 10255
cgroupDriver: cgroupfs
clusterDNS:
- 10.0.0.2
clusterDomain: cluster.local 
failSwapOn: false
authentication:
  anonymous:
    enabled: false
  webhook:
    cacheTTL: 2m0s
    enabled: true
  x509:
    clientCAFile: /home/k8s/kubernetes/ssl/ca.pem 
authorization:
  mode: Webhook
  webhook:
    cacheAuthorizedTTL: 5m0s
    cacheUnauthorizedTTL: 30s
evictionHard:
  imagefs.available: 15%
  memory.available: 100Mi
  nodefs.available: 10%
  nodefs.inodesFree: 5%
maxOpenFiles: 1000000
maxPods: 110
EOF
```

生成bootstrap.kubeconfig文件

```bash
##设置环境变量
$ KUBE_APISERVER="https://172.25.120.1:6443" # apiserver IP:PORT
$ TOKEN="b1dc586d69159ff4e3ef7efa9db60e48" # 与token.csv里保持一致

# 生成 kubelet bootstrap kubeconfig 配置文件
$ kubectl config set-cluster kubernetes \
  --certificate-authority=/home/k8s/kubernetes/ssl/ca.pem \
  --embed-certs=true \
  --server=${KUBE_APISERVER} \
  --kubeconfig=bootstrap.kubeconfig
$ kubectl config set-credentials "kubelet-bootstrap" \
  --token=${TOKEN} \
  --kubeconfig=bootstrap.kubeconfig
$ kubectl config set-context default \
  --cluster=kubernetes \
  --user="kubelet-bootstrap" \
  --kubeconfig=bootstrap.kubeconfig
$ kubectl config use-context default --kubeconfig=bootstrap.kubeconfig
```

拷贝到配置文件路径：

```bash
$ cp bootstrap.kubeconfig /home/k8s/kubernetes/cfg
```

systemd管理kubelet

```bash
$ cat > /usr/lib/systemd/system/kubelet.service << EOF
[Unit]
Description=Kubernetes Kubelet
After=docker.service
[Service]
EnvironmentFile=/home/k8s/kubernetes/cfg/kubelet.conf
ExecStart=/home/k8s/kubernetes/bin/kubelet \$KUBELET_OPTS
Restart=on-failure
LimitNOFILE=65536
[Install]
WantedBy=multi-user.target
EOF
```

启动并设置开机启动

```bash
$ systemctl daemon-reload
$ systemctl start kubelet
$ systemctl enable kubelet
```

批准kubelet证书申请并加入集群

```bash
# 查看kubelet证书请求
$ kubectl get csr
NAME                                                   AGE   SIGNERNAME                                    REQUESTOR           CONDITION
node-csr-d-UyqVObT-tnWdXd881Ppc3oNVr6xkCBXV7VRlWyhf8   30s   kubernetes.io/kube-apiserver-client-kubelet   kubelet-bootstrap   Pending
# 批准申请
$ kubectl certificate approve node-csr-d-UyqVObT-tnWdXd881Ppc3oNVr6xkCBXV7VRlWyhf8
# 查看节点
$ kubectl get node
NAME         STATUS     ROLES    AGE   VERSION
k8s-master   NotReady   <none>   15s   v1.18.3  ##由于没有部署网络插件,所以节点是NotReady
```

发现自动生成一个kubelet客户端证书，该证书就是controller-manager自动为kubelet生成的

```bash
$ ll
total 32
-rw------- 1 root root 1675 Dec  1 11:37 api-server-key.pem
-rw-r--r-- 1 root root 1627 Dec  1 11:37 api-server.pem
-rw------- 1 root root 1679 Dec  1 11:37 ca-key.pem
-rw-r--r-- 1 root root 1359 Dec  1 11:37 ca.pem
-rw------- 1 root root 1224 Dec  1 17:02 kubelet-client-2020-12-01-17-02-02.pem
lrwxrwxrwx 1 root root   63 Dec  1 17:02 kubelet-client-current.pem -> /home/k8s/kubernetes/ssl/kubelet-client-2020-12-01-17-02-02.pem
-rw-r--r-- 1 root root 2177 Dec  1 13:56 kubelet.crt
~~~~-rw------- 1 root root 1679 Dec  1 13:56 kubelet.key
$ pwd
/home/k8s/kubernetes/ssl
```

### 部署kube-proxy

创建配置文件

```bash
$ cat > /home/k8s/kubernetes/cfg/kube-proxy.conf << EOF
KUBE_PROXY_OPTS="--logtostderr=false \\
--v=4 \\
--log-dir=/home/k8s/kubernetes/logs \\
--config=/home/k8s/kubernetes/cfg/kube-proxy-config.yml"
EOF
```

配置参数文件

```bash
$ cat > /home/k8s/kubernetes/cfg/kube-proxy-config.yml << EOF
kind: KubeProxyConfiguration
apiVersion: kubeproxy.config.k8s.io/v1alpha1
bindAddress: 0.0.0.0
metricsBindAddress: 0.0.0.0:10249
clientConnection:
  kubeconfig: /home/k8s/kubernetes/cfg/kube-proxy.kubeconfig
hostnameOverride: k8s-master
clusterCIDR: 10.0.0.0/24
EOF
```

生成kube-proxy证书：

```bash
# 切换工作目录
$ cd ~/TLS/k8s
# 创建证书请求文件
$ cat > kube-proxy-csr.json << EOF
{
  "CN": "system:kube-proxy",
  "hosts": [],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "L": "BeiJing",
      "ST": "BeiJing",
      "O": "k8s",
      "OU": "System"
    }
  ]
}
EOF
# 生成证书
$ cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes kube-proxy-csr.json | cfssljson -bare kube-proxy

$ ls kube-proxy*pem
kube-proxy-key.pem  kube-proxy.pem
```

生成kubeconfig文件

```bash
#创建环境变量
$ KUBE_APISERVER="https://172.25.120.1:6443"

$ kubectl config set-cluster kubernetes \
  --certificate-authority=/home/k8s/kubernetes/ssl/ca.pem \
  --embed-certs=true \
  --server=${KUBE_APISERVER} \
  --kubeconfig=kube-proxy.kubeconfig
$ kubectl config set-credentials kube-proxy \
  --client-certificate=./kube-proxy.pem \
  --client-key=./kube-proxy-key.pem \
  --embed-certs=true \
  --kubeconfig=kube-proxy.kubeconfig
$ kubectl config set-context default \
  --cluster=kubernetes \
  --user=kube-proxy \
  --kubeconfig=kube-proxy.kubeconfig
$ kubectl config use-context default --kubeconfig=kube-proxy.kubeconfig
```

拷贝到配置文件指定路径：

```bash
cp kube-proxy.kubeconfig /home/k8s/kubernetes/cfg/
```

systemd管理kube-proxy

```bash
$ cat > /usr/lib/systemd/system/kube-proxy.service << EOF
[Unit]
Description=Kubernetes Proxy
After=network.target
[Service]
EnvironmentFile=/home/k8s/kubernetes/cfg/kube-proxy.conf
ExecStart=/home/k8s/kubernetes/bin/kube-proxy \$KUBE_PROXY_OPTS
Restart=on-failure
LimitNOFILE=65536
[Install]
WantedBy=multi-user.target
EOF
```

启动并设置开机启动

```bash
$ systemctl daemon-reload
$ systemctl start kube-proxy
$ systemctl enable kube-proxy
```

### 部署CNI网络

先下载CNI二进制文件：

```bash
$ wget https://github.com/containernetworking/plugins/releases/download/v0.8.6/cni-plugins-linux-amd64-v0.8.6.tgz
```

解压二进制包并移动到默认工作目录

```bash
$ mkdir -p /opt/cni/bin
$ tar zxvf cni-plugins-linux-amd64-v0.8.6.tgz -C /opt/cni/bin
```

获取flannel网络yaml文件,并修改镜像地址

```bash
$ echo "151.101.76.133 raw.githubusercontent.com" >>/etc/hosts
$ wget https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
##默认镜像地址无法访问，修改为docker hub镜像仓库。
$ sed -i -r "s#quay.io/coreos/flannel:.*-amd64#lizhenliang/flannel:v0.12.0-amd64#g" kube-flannel.yml  
```

开始部署CNI网络：

```bash
$ kubectl apply -f kube-flannel.yml
##查看pod是否运行成功
$ kubectl get pods -n kube-system
NAME                          READY   STATUS    RESTARTS   AGE
$ kube-flannel-ds-amd64-p9tdp   1/1     Running   0  
##运行成功后,再查看节点是否运行正常
$ kubectl get nodes
NAME         STATUS   ROLES    AGE   VERSION
k8s-master   Ready    <none>   19m   v1.18.3
```

### 增加worker 节点

在master节点将Worker Node涉及文件拷贝到节点172.16.210..54/55

```bash
$ scp -r /home/k8s/kubernetes root@172.25.120.18:/home/k8s/
$ scp -r /usr/lib/systemd/system/{kubelet,kube-proxy}.service root@172.25.120.18:/usr/lib/systemd/system
$ scp -r /home/k8s/cni/ root@172.25.120.18:/home/k8s/
$ scp /home/k8s/kubernetes/ssl/ca.pem root@172.25.120.18:/home/k8s/kubernetes/ssl
```

在node节点删除kubelet证书和kubeconfig文件

```bash
$ rm -f /home/k8s/kubernetes/cfg/kubelet.kubeconfig 
$ rm -f /home/k8s/kubernetes/ssl/kubelet*
```

修改主机名

```bash
 #加入node2的主机只需要把这条命令的k8s-node1改成k8s-node2即可
$ sed -i 's/k8s-master/node-1/g' /home/k8s/kubernetes/cfg/kubelet.conf /home/k8s/kubernetes/cfg/kube-proxy-config.yml  
```

授权apiserver访问kubelet

```bash
$ cat > apiserver-to-kubelet-rbac.yaml << EOF
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
  name: system:kube-apiserver-to-kubelet
rules:
  - apiGroups:
      - ""
    resources:
      - nodes/proxy
      - nodes/stats
      - nodes/log
      - nodes/spec
      - nodes/metrics
      - pods/log
    verbs:
      - "*"
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: system:kube-apiserver
  namespace: ""
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:kube-apiserver-to-kubelet
subjects:
  - apiGroup: rbac.authorization.k8s.io
    kind: User
    name: kubernetes
EOF

$ kubectl apply -f apiserver-to-kubelet-rbac.yaml
```

启动并设置开机启动

```bash
$ systemctl daemon-reload
$ systemctl start kubelet
$ systemctl enable kubelet
$ systemctl start kube-proxy
$ systemctl enable kube-proxy
```

在Master上批准新Node kubelet证书申请

```bash
$ kubectl get csr
NAME                                                   AGE   SIGNERNAME                                    REQUESTOR           CONDITION
node-csr--t2cjSYX0z7ba4Tyh4GCnngZaGBUwmAHyY1xuxU40j0   28s   kubernetes.io/kube-apiserver-client-kubelet   kubelet-bootstrap   Pending

$ kubectl certificate approve node-csr--t2cjSYX0z7ba4Tyh4GCnngZaGBUwmAHyY1xuxU40j0
```

查看Node状态

```bash
$ kubectl get nodes
NAME         STATUS   ROLES    AGE     VERSION
k8s-master   Ready    <none>   46m     v1.18.3
k8s-node1    Ready    <none>   8m57s   v1.18.3
k8s-node2    Ready    <none>   3m59s   v1.18.3
```

Node2（172.25.120.19 ）节点同上。记得修改主机名

## 部署CoreDNS

CoreDNS用于集群内部Service名称解析

```bash
`# Warning: This is a file generated from the base underscore template file: coredns.yaml.base

apiVersion: v1
kind: ServiceAccount
metadata:
  name: coredns
  namespace: kube-system
  labels:
      kubernetes.io/cluster-service: "true"
      addonmanager.kubernetes.io/mode: Reconcile
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
    addonmanager.kubernetes.io/mode: Reconcile
  name: system:coredns
rules:
- apiGroups:
  - ""
  resources:
  - endpoints
  - services
  - pods
  - namespaces
  verbs:
  - list
  - watch
- apiGroups:
  - ""
  resources:
  - nodes
  verbs:
  - get
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
    addonmanager.kubernetes.io/mode: EnsureExists
  name: system:coredns
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:coredns
subjects:
- kind: ServiceAccount
  name: coredns
  namespace: kube-system
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: coredns
  namespace: kube-system
  labels:
      addonmanager.kubernetes.io/mode: EnsureExists
data:
  Corefile: |
    .:53 {
        log
        errors
        health {
            lameduck 5s
        }
        ready
        kubernetes cluster.local in-addr.arpa ip6.arpa {
            pods insecure
            fallthrough in-addr.arpa ip6.arpa
            ttl 30
        }
        prometheus :9153
        forward . /etc/resolv.conf
        cache 30
        loop
        reload
        loadbalance
    }
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: coredns
  namespace: kube-system
  labels:
    k8s-app: kube-dns
    kubernetes.io/cluster-service: "true"
    addonmanager.kubernetes.io/mode: Reconcile
    kubernetes.io/name: "CoreDNS"
spec:
  # replicas: not specified here:
  # 1. In order to make Addon Manager do not reconcile this replicas parameter.
  # 2. Default is 1.
  # 3. Will be tuned in real time if DNS horizontal auto-scaling is turned on.
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
  selector:
    matchLabels:
      k8s-app: kube-dns
  template:
    metadata:
      labels:
        k8s-app: kube-dns
      annotations:
        seccomp.security.alpha.kubernetes.io/pod: 'runtime/default'
    spec:
      priorityClassName: system-cluster-critical
      serviceAccountName: coredns
      tolerations:
        - key: "CriticalAddonsOnly"
          operator: "Exists"
      nodeSelector:
        kubernetes.io/os: linux
      containers:
      - name: coredns
        image: lizhenliang/coredns:1.6.7
        imagePullPolicy: IfNotPresent
        resources:
          limits:
            memory: 512Mi 
          requests:
            cpu: 100m
            memory: 70Mi
        args: [ "-conf", "/etc/coredns/Corefile" ]
        volumeMounts:
        - name: config-volume
          mountPath: /etc/coredns
          readOnly: true
        ports:
        - containerPort: 53
          name: dns
          protocol: UDP
        - containerPort: 53
          name: dns-tcp
          protocol: TCP
        - containerPort: 9153
          name: metrics
          protocol: TCP
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
            scheme: HTTP
          initialDelaySeconds: 60
          timeoutSeconds: 5
          successThreshold: 1
          failureThreshold: 5
        readinessProbe:
          httpGet:
            path: /ready
            port: 8181
            scheme: HTTP
        securityContext:
          allowPrivilegeEscalation: false
          capabilities:
            add:
            - NET_BIND_SERVICE
            drop:
            - all
          readOnlyRootFilesystem: true
      dnsPolicy: Default
      volumes:
        - name: config-volume
          configMap:
            name: coredns
            items:
            - key: Corefile
              path: Corefile
---
apiVersion: v1
kind: Service
metadata:
  name: kube-dns
  namespace: kube-system
  annotations:
    prometheus.io/port: "9153"
    prometheus.io/scrape: "true"
  labels:
    k8s-app: kube-dns
    kubernetes.io/cluster-service: "true"
    addonmanager.kubernetes.io/mode: Reconcile
    kubernetes.io/name: "CoreDNS"
spec:
  selector:
    k8s-app: kube-dns
  clusterIP: 10.0.0.2 
  ports:
  - name: dns
    port: 53
    protocol: UDP
  - name: dns-tcp
    port: 53
    protocol: TCP
  - name: metrics
    port: 9153
    protocol: TCP`

$ kubectl apply -f coredns.yaml 

$ kubectl get pods -n kube-system ##查看coredns的pod是否运行正常
NAME                          READY   STATUS    RESTARTS   AGE
coredns-5ffbfd976d-rkcmt      1/1     Running   0          23s
kube-flannel-ds-amd64-2kmcm   1/1     Running   0          14m
kube-flannel-ds-amd64-p9tdp   1/1     Running   0          39m
kube-flannel-ds-amd64-zg7xz   1/1     Running   0          19m`
```

测试

```bash
$ kubectl run -it --rm dns-test --image=busybox:1.28.4 sh
If you don't see a command prompt, try pressing enter.
/ # nslookup kubernetes
Server:    10.0.0.2
Address 1: 10.0.0.2 kube-dns.kube-system.svc.cluster.local

Name:      kubernetes
Address 1: 10.0.0.1 kubernetes.default.svc.cluster.local
```