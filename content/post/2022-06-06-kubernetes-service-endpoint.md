---
layout:     post
title:      "手动配置service、endpoint 访问集群外服务"
description: "手动配置service、endpoint 访问集群外服务"
date:     2022-06-06
author: sfeng
tags: ["kubernetes-ops"]
URL: "/2022/06/06/k8s-service-endpoint/"
---

## 背景

在 k8s 集群里面，我们可以通过 ingress —> service —> endpoint 这个数据流来访问我们的服务，但有些情况，有的服务跑在 k8s 集群外，可能是 docker run 启动的，也有可能是直接跑在虚拟机上面，那么这些情况我们如何使用同一个ingress 域名来访问这些集群外服务呢。

## service、endpoint

在 k8s 集群里面，service 会自动通过 endpoint 这个资源来绑定我们的 pod。如果访问集群外服务，那么就需要手动绑定。

这里以一个例子说明，比如我的集群里面通过 docker-compose 跑了一个 harbor 服务，这个harbor 服务也需要被访问，那么我们需要通过 ingress 来代理到这个 harbor。

在创建 Service 的时候，也可以不指定 Selectors，用来将 service 转发到 kubernetes 集群外部的服务（而不是 Pod）。目前支持两种方法

### 自定义 endpoint

（1）自定义 endpoint，即创建**同名**的 service 和 endpoint，在 endpoint 中设置外部服务的 IP 和端口

```yaml
apiVersion: v1
kind: Endpoints
metadata:
  name: harbor
subsets:
- addresses:
  - ip: 10.100.202.24  # harbor 所在节点 ip
  ports:
  - port: 80
    name: http
  - port: 443
    name: https
---
apiVersion: v1
kind: Service
metadata:
  name: harbor
spec:
  ports:
  - port: 80
    name: http
    protocol: TCP
    targetPort: 80
  - port: 443
    name: https
    protocol: TCP
    targetPort: 443
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    nginx.ingress.kubernetes.io/backend-protocol: HTTPS
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
    nginx.ingress.kubernetes.io/ssl-passthrough: "true"
  name: harbor-ingress
spec:
  ingressClassName: nginx
  rules:
  - host: harbor.test.com
    http:
      paths:
      - backend:
          service:
            name: harbor
            port:
              number: 443
        path: /
        pathType: Prefix
  tls:
  - hosts:
    - harbor.test.com
status:
  loadBalancer:
    ingress:
    - ip: 10.100.202.2
```

> 注意：endpoint 在 svc 之后部署
>