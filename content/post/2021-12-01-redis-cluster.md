---
layout:     post
title:      "使用statefulset部署redis-cluster"
#subtitle:   "本文翻译自istio官方文档"
description: "本文演示使用statefulset部署redis-cluster"
excerpt: "本文演示使用statefulset部署redis-cluster"
date:     2021-12-01
#author:     ""
image: "https://unsplash.com/blog/content/images/2021/09/Back-to-School-blog.jpg"
categories: [ "Tech"]
tags:
- Kubertes  
URL: "/2021/12/01/redis-cluster/"
---

# 使用statefulset部署redis-cluster

内核参数优化

```python
# sysctl.conf中配置fs.file-max、net.core.somaxconn两个属性
$ cat << EOF >> /etc/sysctl.conf
fs.file-max=655350
net.core.somaxconn=20480
EOF
sysctl -p

# limits.conf中配置文件句柄数及进程数的硬限制和软限制
$ cat << 'EOF' >> /etc/security/limits.conf
*	hard	nofile	655350
*	soft	nofile	655350
*	hard	nproc	655350
*	soft	nproc	655350
EOF

# 关闭内存transparent_hugepage特性
$ cat << 'EOF' >> /etc/rc.local
echo never > /sys/kernel/mm/transparent_hugepage/enabled
EOF
$ echo never > /sys/kernel/mm/transparent_hugepage/enabled

# kubelet中允许修改pod的net.core.somaxconn内核参数
$ cat /etc/systemd/system/kubelet.service
...
ExecStart=/usr/local/bin/kubelet \
   ...
   --allowed-unsafe-sysctls=net.core.somaxconn \
   ...
   
# 修改pod的net.core.somaxconn内核参数
$ kubectl -n demo get statefulsets  redis-redis-cluster -o yaml
...
podSpec:
	securityContext:
    sysctls:
    - name: net.core.somaxconn
    value: "20480"
...
```

使用configmap发布redis启动文件，redis-config.yaml

说明：2个文件，node-update.sh用来redis启动时执行脚本，具体后面再介绍为何要增加一个启动脚本；redis.conf为redis启动配置文件。

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
 name: redis-config
 namespace: dydz
data:
 update-node.sh: |
  #!/bin/sh
  REDIS_NODES="/data/nodes.conf"
  sed -i -e "/myself/ s/[0-9]\{1,3\}\.[0-9]\{1,3\}\.[0-9]\{1,3\}\.[0-9]\{1,3\}/${MY_POD_IP}/" ${REDIS_NODES}
  exec "$@"
 redis.conf: |
  port 7001
  protected-mode no
  cluster-enabled yes
  cluster-config-file /data/nodes.conf
  cluster-node-timeout 15000
  #cluster-announce-ip ${MY_POD_IP}
  dbfilename dump.rdb
  dir "/data"
  save 900 1
  save 300 10
  save 60 10000
  cluster-announce-port 7001
  cluster-announce-bus-port 17001
  logfile "/data/redis.log"
  requirepass "LzkpOFLY4I9B"
```

使用headless service创建redis-cluster访问方式

```yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    app: redis-cluster
  name: redis-cluster
  namespace: dydz
spec:
  ports:
    - name: redis-port
      port: 7001
      protocol: TCP
      targetPort: 7001
  selector:
    app: redis-cluster
  type: ClusterIP
  clusterIP: None
```

使用statefulset部署redis-cluster，集群里要提前创建storageclass，这里提供的是ceph 块存储

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  labels:
    app: redis-cluster
  name: redis-cluster
  namespace: dydz
spec:
  replicas: 6
  selector:
    matchLabels:
      app: redis-cluster
  serviceName: redis-cluster
  template:
    metadata:
      labels:
        app: redis-cluster
    spec:
      securityContext:
        sysctls:
        - name: net.core.somaxconn
          value: "20480"
      containers:
        - command:
            ["/bin/bash", "/etc/redis/update-node.sh", "redis-server", "/etc/redis/redis.conf"]
          #args:
          #  - /usr/local/etc/redis/redis.conf
          #  - --cluster-announce-ip
          #  - "$(MY_POD_IP)"
          env:
            - name: MY_POD_IP
              valueFrom:
                fieldRef:
                  fieldPath: status.podIP
            - name: TZ
              value: Asia/Shanghai
          image: 'redis:5.0.8'
          imagePullPolicy: IfNotPresent
          name: redis
          ports:
            - containerPort: 7001
              #hostPort: 7001
              name: redis-port
              protocol: TCP
          volumeMounts:
            - mountPath: /data
              name: redis-cluster-data
              subPath: data
              readOnly: false
            - mountPath: /etc/redis
              name: redis-config
              readOnly: false
      dnsPolicy: ClusterFirst
      volumes:
        - name: redis-config
          configMap:
           name: redis-config
  volumeClaimTemplates:  #PVC模板
  - metadata:
      name: redis-cluster-data
    spec:
      accessModes: [ "ReadWriteOnce"]
      storageClassName: rook-ceph-block  # 创建的共享存储storageclass 名称，会自动创建pv与pvc关联
      resources:
        requests:
          storage: 10Gi
```

初始化集群

```yaml
redis-cli -p 7001 -a LzkpOFLY4I9B --cluster create --cluster-replicas 1 10.233.112.89:7001 10.233.106.74:7001 10.233.109.71:7001 10.233.113.60:7001 10.233.112.90:7001
```

验证集群

```yaml
kubectl exec -it redis-cluster-0 -- redis-cli cluster info
```

## **踩坑总结**

问题1，集群初始化时，一直等待 Waiting for the cluster to join解决：最开始部署时，从使用docker-comopose部署的方法套用过来，由于redis.conf配置文件中参数cluster-announce-ip配置了宿主机的ip，当初始化时集群节点之间需要通讯，但是再k8s中宿主机ip已经不适用，导致无法通讯一致卡住在那个位置。禁用#cluster-announce-ip

问题2，集群创建时，[ERR] Node 10.244.7.224:7001 is not empty. Either the node already knows other nodes (check with CLUSTER NODES) or contains some key in database 0.解决：由于前面初始化集群时卡住，导致部分节点的nodes.conf文件中更新了新节点数据，需要删除数据。可以删除文件，也可以通过如下命令redis-cli -p 7001 -c 登录每个redis节点重置:

```php
flushdb
cluster reset
```

问题3，pvc如何和pv一对一绑定？说明：分静态和动态绑定首先静态，之前一直没想明白，写好了pv的yaml文件，service里面定义好pvc之后，如何让pv和pvc进行绑定？只要我们定义的pv存储空间大小和pvc申请大小一致就会匹配绑定bound,当然也可以通过lebel标签来关联pvc到特定的pv。另外，redis集群多个节点，持久化存储目录肯定不能一样，如果yaml中直接指定PVC的名字，只能指定一个名字，这样是办不到关联到多个不同的PV，于是需要volumeClaimTemplates（PVC模板）。

```php
  volumeClaimTemplates:
  - metadata:
      name: redis-cluster-data
namespace: default
    spec:
      accessModes: [ "ReadWriteMany" ]
      resources:
        requests:
          storage: 1Gi
```