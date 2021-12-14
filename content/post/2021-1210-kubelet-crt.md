---
layout:     post
title:      "kubelet-client证书自签"
#subtitle:   "本文翻译自istio官方文档"
description: "kubelet-client证书自签"
excerpt: "kubelet-client证书自签"
date:     2021-12-10
#author:     ""
image: "https://unsplash.com/blog/content/images/2021/04/Bruna-image-1-blog-2.jpg"
categories: [ "Tech"]
tags:
- Kubernetes
URL: "/2021/12/10/kubelet-client-crt/"
---

## 背景

今天发现线上集群一个node节点kubelet-client证书莫名其妙消失了，导致该节点 NotReady，因为该证书已经没了，导致controller-manager无法自动轮换该证书，所以需要自己签发证书。

## 环境信息

```bash
$ kubenertes: v1.15.3
$ centos: 7.5
$ kernel: 4.14.49
$ cfssl: 1.2.0
```

## 签发证书

kubelet访问api-server是双向https协议，所以kubelet 会验证api-server，api-server也会验证kubelet。目前出问题的是api-server验证kubelet时，kubelet无法提供证书，所以我们需要使用api-server验证kubelet的CA来签发kubelet-client.crt。

这里使用cfssl工具签发证书。

1、安装cfssl

```bash
https://github.com/cloudflare/cfssl/releases
mv cfssl /usr/bin
```

2、获取ca.crt，ca.key

可以直接从api-server所在服务器拷贝

3、生成ca-config.json

```bash
# 将一下信息写入ca-config.json
{
  "signing": {
    "default": {
      "expiry": "87600h"
    },
    "profiles": {
      "kubernetes": {
        "usages": [
            "signing",
            "key encipherment",
            "server auth",
            "client auth"
        ],
        "expiry": "876000h"
      }
    }
  }
}
```

4、生成kubelet-client证书请求文件

```bash
# 将以下信息写入kubelet-client-csr.json
{
        "CN": "system:node:node-1",   # 修改对应节点hostname
        "key": {
                "algo": "rsa",
                "size": 2048
        },
        "names": [{
                "O": "system:nodes"
        }]
}
```

5、生成证书

```bash
$ cfssl gencert -ca=ca.crt -ca-key=ca.key --config=ca-config.json -profile=kubernetes kubelet-client-csr.json | cfssljson -bare kubelet-client
```

6、修改kubelet.conf

查看kubelet 访问api-server的凭证文件即kubelet.conf，默认在/etc/kubernetes/kubelet.conf下。

将生成crt，key放入对应目录下，这里为/var/lib/kubelet/pki/kubelet-client.crt、/var/lib/kubelet/pki/kubelet-client.key

```bash
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSUN5RENDQWJDZ0F3SUJBZ0lCQURBTkJna3Foa2lHOXcwQkFRc0ZBREFWTVJNd0VRWURWUVFERXdwcmRXSmwKY201bGRHVnpNQjRYRFRJeE1URXhOakF5TkRFMU4xb1hEVE14TVRFeE5EQXlOREUxTjFvd0ZURVRNQkVHQTFVRQpBeE1LYTNWaVpYSnVaWFJsY3pDQ0FTSXdEUVlKS29aSWh2Y05BUUVCQlFBRGdnRVBBRENDQVFvQ2dnRUJBTUN5Ck1xVFpYNjlOc3p2WTRpMk5FTUdtMUd6RXQrSVN4K2Z1UXAwbDJjVUk2bFk4UnJqTkdsK1ZKTVNjVlBITUtDSzYKSkFKcUVCb3FKdXZaN3FsalZ5V0JpZmc4Y1RaMmRtUnNKU3gyYWMyd2VDb0lVQlJBYVRrOG5yRWQyNVFkVnViYgpHbDRYNnlFbm9tT0RpODQzL25od1NCb05vZE1OdmdKaW1IR291NCszZWhZSVNWVVVoWTNzT3BlVTVQSlcxK0FwCmZkeTk3S3RrMUJ6NTYvU1lBS3h0bHkyelhYRnkzaGJ4WTM2NWpjRHNGcGdRTDJ4cWdENWcwS1NCMjlHNXZWWnUKVWtwTWFXZmp4b0w5UmlFd2J5VmpUZFJrM01YNWxTZkFrM3NlYmNENEhSczJmMnJHVjFlaHpnbVBxQmVnUlVMYQo1dnpTYzU2c3FNc08xbUJZWVkwQ0F3RUFBYU1qTUNFd0RnWURWUjBQQVFIL0JBUURBZ0trTUE4R0ExVWRFd0VCCi93UUZNQU1CQWY4d0RRWUpLb1pJaHZjTkFRRUxCUUFEZ2dFQkFHbmlmbTNNRFIzV2VrODE3Q3ZvZVBWOGF4ckoKVlVRenMxSFJrNVR1WHdhSmNOQUhnQlZyNXZOWUk3Q3F3QWdCOWcwdGtxTjlpQmNtU2JLMVVnVlVsT3krUnE4LwpUcGp4OEVPN0pCQVh0UTd1cGJRWU5BdEMrTk8xOURIUVM4K0IvaktJbU5GRmNWL0tsZ0RTQzB0KzU1ZVRxQjdtCmFIZlpmUzl0dVhUWit1OUthVlpkcVpOUkdzOGh4eVBqazBpcS9mU1c1aDVNb1djaFM0TkVhbmhidkF3QlR6QTcKc1BMZWRaZUxLVzlXemZaRFJWN0M3M3NlNVZsbStnR2RhSTNoT0dSaXcxMVg4cFRrMVo5YnVocnc5bEYwNzFQRgowaXlJNUFrTGRCVzkxQXpXT3IyZVhwYzA1cElLVTlldGRqWXp2RTIrT1FvK1VUeVZKR2cvYVFzSnRiTT0KLS0tLS1FTkQgQ0VSVElGSUNBVEUtLS0tLQo=
    server: https://172.30.13.88:6443
  name: default-cluster
contexts:
- context:
    cluster: default-cluster
    namespace: default
    user: default-auth
  name: default-context
current-context: default-context
kind: Config
preferences: {}
users:
- name: default-auth
  user:
    client-certificate: /var/lib/kubelet/pki/kubelet-client.crt
    client-key: /var/lib/kubelet/pki/kubelet-client.key
```

7、重启kubelet

```bash
systemctl restart kubelet
```