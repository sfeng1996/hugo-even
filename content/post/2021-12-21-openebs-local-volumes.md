---
layout:     post
title:      "openebs local-pv"
description: "使用 openebs 作为本地存储实现数据持久化"
except: "使用 openebs 作为本地存储实现数据持久化"
date:     2021-12-21
author:  "sfeng"
categories: ["Tech", "cloudNative"]
tags: ["kubernetes", "storage"]
URL: "/2021/12/21/openebs-local-volumes/"
---

## openebs

**[openebs](https://openebs.io/)** 是一款使用Go语言编写的基于容器的块存储开源软件。OpenEBS 使得在容器中运行关键性任务和需要数据持久化的负载变得更可靠。

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/6d672096-7462-489c-9911-8c9755736d13/Untitled.png)

**openebs** 有两类存储类型

- Local Volumes
- Replicated Volumes

**Local Volumes** 类型实现了自动分配 **local-pv**，应用如果对存储 **io** 性能高的话，可能会考虑本地存储，那么**local-pv** 是一个比较好的选择，但是使用 **local-pv** 的话需要手动创建 **pv、storageclass。openebs** 就帮助我们实现了这一步，使得使用 **local-pv** 和使用分布式存储一样透明，简洁。

## 如何部署openebs

这里我们只使用 **local-pv-hostpath**，即使用本地文件系统的存储，根据[官网](https://openebs.io/docs/user-guides/prerequisites)介绍，不需要提前安装 **iscsi**

### Helm 部署

```bash
$ helm repo add openebs https://openebs.github.io/charts
$ helm repo update
$ helm install openebs --namespace openebs openebs/openebs --create-namespace
```

## kubectl 部署

```bash
kubectl apply -f https://openebs.github.io/charts/openebs-operator.yaml
```

安装成功后，默认会部署两个 **storageclass** ，我们目前需要 **openebs-hostpath。**默认存储目录为 ****`/var/openebs/local`

```bash
$ kubectl get pods -n openebs
NAME                                           READY   STATUS    RESTARTS   AGE
openebs-localpv-provisioner-69c8648db7-cnj45   1/1     Running   0          33m
openebs-ndm-bbgpv                              1/1     Running   0          33m
openebs-ndm-bxsbb                              1/1     Running   0          33m
openebs-ndm-cluster-exporter-9d75d564d-qvqz6   1/1     Running   0          33m
openebs-ndm-kdg7b                              1/1     Running   0          33m
openebs-ndm-node-exporter-9zm62                1/1     Running   0          33m
openebs-ndm-node-exporter-hlj7h                1/1     Running   0          33m
openebs-ndm-node-exporter-j6wj7                1/1     Running   0          33m
openebs-ndm-node-exporter-s5hk4                1/1     Running   0          33m
openebs-ndm-operator-789985cc47-r4hwj          1/1     Running   0          33m
openebs-ndm-qm4sw                              1/1     Running   0          33m
```

```bash
$ kubectl get sc
NAME               PROVISIONER        RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
openebs-device     openebs.io/local   Delete          WaitForFirstConsumer   false                  116s
openebs-hostpath   openebs.io/local   Delete          WaitForFirstConsumer   false                  116s
```

如果需要更改的话，直接修改 `openebs-operator.yaml`，`value` 字段即可

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: openebs-hostpath
  annotations:
    openebs.io/cas-type: local
    cas.openebs.io/config: |
      #hostpath type will create a PV by 
      # creating a sub-directory under the
      # BASEPATH provided below.
      - name: StorageType
        value: "hostpath"
      #Specify the location (directory) where
      # where PV(volume) data will be saved. 
      # A sub-directory with pv-name will be 
      # created. When the volume is deleted, 
      # the PV sub-directory will be deleted.
      #Default value is /var/openebs/local
      - name: BasePath
        value: "/var/openebs/local/"
```

## 验证

接下来我们创建一个 PVC 资源对象，Pods 使用这个 PVC 就可以从 OpenEBS 动态 Local PV Provisioner 中请求 Hostpath Local PV 了。

```yaml
# local-hostpath-pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: local-hostpath-pvc
spec:
  storageClassName: openebs-hostpath
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
```

直接创建这个 PVC 即可：

```bash
$ kubectl apply -f local-hostpath-pvc.yaml
$ kubectl get pvc local-hostpath-pvc
NAME                 STATUS    VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS       AGE
local-hostpath-pvc   Pending                                      openebs-hostpath   12s
```

我们可以看到这个 PVC 的状态是 `Pending`，这是因为对应的 StorageClass 是延迟绑定模式，所以需要等到 Pod 消费这个 PVC 后才会去绑定，接下来我们去创建一个 Pod 来使用这个 PVC。

声明一个如下所示的 Pod 资源清单：

```yaml
# local-hostpath-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: hello-local-hostpath-pod
spec:
  volumes:
  - name: local-storage
    persistentVolumeClaim:
      claimName: local-hostpath-pvc
  containers:
  - name: hello-container
    image: busybox
    command:
       - sh
       - -c
       - 'while true; do echo "`date` [`hostname`] Hello from OpenEBS Local PV." >> /mnt/store/greet.txt; sleep $(($RANDOM % 5 + 300)); done'
    volumeMounts:
    - mountPath: /mnt/store
      name: local-storage

```

直接创建这个 Pod：

```c
$ kubectl apply -f local-hostpath-pod.yaml
$ kubectl get pods hello-local-hostpath-pod
NAME                       READY   STATUS    RESTARTS   AGE
hello-local-hostpath-pod   1/1     Running   0          2m7s
$ kubectl get pvc local-hostpath-pvc
NAME                 STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS       AGE
local-hostpath-pvc   Bound    pvc-3f4a1a65-6cbc-42bf-a1f8-87ad238c0b88   5Gi        RWO            openebs-hostpath   5m41s

```

可以看到 Pod 运行成功后，PVC 也绑定上了一个自动生成的 PV，我们可以查看这个 PV 的详细信息：

```bash
$ kubectl get pv pvc-3f4a1a65-6cbc-42bf-a1f8-87ad238c0b88 -o yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  annotations:
    pv.kubernetes.io/provisioned-by: openebs.io/local
  creationTimestamp: "2021-01-07T02:48:14Z"
  finalizers:
  - kubernetes.io/pv-protection
  labels:
    openebs.io/cas-type: local-hostpath
  ......
  name: pvc-3f4a1a65-6cbc-42bf-a1f8-87ad238c0b88
  resourceVersion: "21193802"
  selfLink: /api/v1/persistentvolumes/pvc-3f4a1a65-6cbc-42bf-a1f8-87ad238c0b88
  uid: f7cccdb3-d23a-4831-86c3-4363eb1a8dee
spec:
  accessModes:
  - ReadWriteOnce
  capacity:
    storage: 5Gi
  claimRef:
    apiVersion: v1
    kind: PersistentVolumeClaim
    name: local-hostpath-pvc
    namespace: default
    resourceVersion: "21193645"
    uid: 3f4a1a65-6cbc-42bf-a1f8-87ad238c0b88
  local:
    fsType: ""
    path: /var/openebs/local/pvc-3f4a1a65-6cbc-42bf-a1f8-87ad238c0b88
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - node2
  persistentVolumeReclaimPolicy: Delete
  storageClassName: openebs-hostpath
  volumeMode: Filesystem
status:
  phase: Bound

```

我们可以看到这个自动生成的 PV 和我们前面自己手动创建的 Local PV 基本上是一致的，和 node2 节点是亲和关系，本地数据目录位于 `/var/openebs/local/pvc-3f4a1a65-6cbc-42bf-a1f8-87ad238c0b88` 下面

接着我们来验证下 volume 数据，前往 node2 节点查看下上面的数据目录中的数据：

```bash
[root@node2 ~]# ls /var/openebs/local/pvc-3f4a1a65-6cbc-42bf-a1f8-87ad238c0b88
greet.txt
[root@node2 ~]# cat /var/openebs/local/pvc-3f4a1a65-6cbc-42bf-a1f8-87ad238c0b88/greet.txt
Thu Jan  7 10:48:49 CST 2021 [hello-local-hostpath-pod] Hello from OpenEBS Local PV.
Thu Jan  7 10:53:50 CST 2021 [hello-local-hostpath-pod] Hello from OpenEBS Local PV.

```

可以看到 Pod 容器中的数据已经持久化到 Local PV 对应的目录中去了。但是需要注意的是 StorageClass 默认的数据回收策略是 Delete，所以如果将 PVC 删掉后数据会自动删除，我们可以 `Velero` 这样的工具来进行备份还原。

## 引用

[openebs 官网](https://openebs.io/)

[kubernetes 训练营](https://www.qikqiak.com/k8strain2/storage/openebs/)