---
layout:     post
title:      "kubelet-client, kubeconfig 证书自签"
description: "kubelet-client, kubeconfig证书自签"
date:     2021-12-10
author: sfeng
categories: ["Tech", "cloudNative", "ops"]
tags: ["kubernetes"]
URL: "/2021/12/10/kubelet-client-crt/"
---

# 背景

今天发现线上集群一个node节点kubelet-client证书莫名其妙消失了，导致该节点 NotReady，因为该证书已经没了，导致controller-manager无法自动轮换该证书，所以需要自己签发证书。
以及 kubeconfig 也失效，导致 kubectl 使用不了

# 环境信息

```bash
$ kubenertes: v1.15.3
$ centos: 7.5
$ kernel: 4.14.49
$ cfssl: 1.2.0

```

# 签发 kubelet 证书

kubelet访问api-server是双向https协议，所以kubelet 会验证api-server，api-server也会验证kubelet。目前出问题的是api-server验证kubelet时，kubelet无法提供证书，所以我们需要使用api-server验证kubelet的CA来签发kubelet-client.crt。

这里使用cfssl工具签发证书。

1、安装cfssl，cfssljson

```bash
https://github.com/cloudflare/cfssl/releases
```

2、获取ca.crt，ca.key，可以直接从api-server所在服务器拷贝

3、生成ca-config.json，将以下信息写入ca-config.json

```json
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

4、生成kubelet-client证书请求文件，将以下信息写入 `kubelet-client-csr.json`

```json

{
        "CN": "system:node:node-1",
        "key": {
                "algo": "rsa",
                "size":2048
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

```yaml
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
$ systemctl restart kubelet
```

# 签发 kubeconfig 证书

1、生成 kubeconfig 证书请求文件，将以下信息写入 `admin-csr.json`

```json

{
	"CN": "kubernetes-admin",
	# 集群 master ip
	"hosts": [
		"172.16.8.60", "172.16.8.61", "172.16.8.62"
	],
	"key": {
		"algo": "rsa",
		"size": 2048
	},
	"names": [{
		"C": "CN",
		"O": "system:masters"
	}]
}

```
2、生成证书

```bash
$ cfssl gencert -ca=ca.crt -ca-key=ca.key --config=ca-config.json -profile=kubernetes admin-csr.json | cfssljson -bare admin

```

3、修改 kubeconfig
将 kubeconfig 文件里证书路径替换为新生成的证书，或者将新生成的证书内容进行 base64 编码，替换 kubeconfig 文件内容。

```bash
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSUM2VENDQWRHZ0F3SUJBZ0lCQURBTkJna3Foa2lHOXcwQkFRc0ZBREFWTVJNd0VRWURWUVFERXdwcmRXSmwKY201bGRHVnpNQ0FYRFRJeU1Ea3dOakE1TWpFMU5sb1lEekl4TWpJd09ERXpNRGt5TVRVMldqQVZNUk13RVFZRApWUVFERXdwcmRXSmxjbTVsZEdWek1JSUJJakFOQmdrcWhraUc5dzBCQVFFRkFBT0NBUThBTUlJQkNnS0NBUUVBCitTWHZXcENhLzVMUHdKWmVhMkZ0RitQZU9na2VZM0hFcGZJTVNsdkNQZjQyL1U4bFJTTXhGeEc5b0Jaa3J3bHoKeEgxRHRlNXhVN0szZGZtMm1zdGd4bVhCQ3FialhnRStHNlk1Q3JLYmdkOE1ucGJjeXhvQ2l4OHB4RVZNaHpFbApwZm1KL29vc2ZtbHcrNk1vZ29hNkVDT2pnRWloZDZidDdNblJMcCs4Z1M4djR6a09SWWNxaEFjSjYwc0t3QmF3Cm51TlFmRERqeVMvbHp2WDhoa0JXeWVFUTdYZzVHQWdNbFFQSVgyZFRoQU1weE1VUWxCZS9LTURVQ0hQeEd0UkcKL3l3Y3BnTG9ZUTE4K2Q1czFVM3VORXdlalVxQzBQRkZabjFyZmZYR3NmZU5CN1EwSy81alpmWUUwaUtncWlDdQpES1pZd0R3ODRseVhOM0tMU3Npcll3SURBUUFCbzBJd1FEQU9CZ05WSFE4QkFmOEVCQU1DQXFRd0R3WURWUjBUCkFRSC9CQVV3QXdFQi96QWRCZ05WSFE0RUZnUVVvdGtEZVBUQkJGemFUWXdXQmpuMXl2SC8yeWN3RFFZSktvWkkKaHZjTkFRRUxCUUFEZ2dFQkFQZjZCcEd6TGY2L0RhdzdkS2FvbTlTYzFQWWZ1dDdRWlQ0Q3hOdXJuSVFTd04rZQpGSys5MzJYVVhoVnExWUx0N1RpcE9xc0JMOXBlS0p6YUtmQmpoN1N1WVV2OTM1TkxZaXEySTkyWXJwVEtyWlpxCjhvQ3BnQWloZEFmaVpOekkyK0NoZVRKOVhOek5yZTRSdW04cVM4ZVRFWEN5UW5Fd2RQU1RyUFJuR2M5OEdXQlcKYW5Gc0NiYnZpYnoyY3d4LytXLzlnSzFGaW5zOUhhNE5yS0hQbldsalppOExycTlSN1RlSDNBRzVFT3dZek0rUAo3UGJnYTdhK24wWUpQVGtqb3VqUWRkYmdaQ01VSjNxc0Z3dE1LemhybTZQZGtGQ0dpSWg5elhlMGthdk4ybnlEClVVaUdVM2FmK2UyTXQyTE42Mk5SM3dpYXBJNS9QTUdrajFjVC9zMD0KLS0tLS1FTkQgQ0VSVElGSUNBVEUtLS0tLQo=
    server: https://apiserver.cluster.local:6443
  name: kubernetes
contexts:
- context:
    cluster: kubernetes
    user: kubernetes-admin
  name: kubernetes-admin@kubernetes
current-context: kubernetes-admin@kubernetes
kind: Config
preferences: {}
users:
- name: kubernetes-admin
  user:
    # 修改这里即可
    client-certificate: /path/to/kubeconfig.pem
    client-key: /path/to/kubeconfig-key.pem

```
