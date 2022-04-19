---
layout:     post
title:      "go get 私有仓库"
description: "go get 私有仓库"
date:     2022-04-018
author: sfeng
categories: ["Tech", "dev"]
tags: ["GOLANG"]
URL: "/2022/04/18/go-get-private/"
---

## 背景

在go 项目开发时，我们大多需要引入包时，都会去github仓库去拉取开源代码，但是有的时候需要依赖私有仓库的包，那么就需要去私有仓库拉取。

## 设置GOPRIVATE

如果不设置这个变量的话，go 默认使用`GOPROXY`代理去访问这个仓库，当然无法访问，配置这个变量的话，可以忽略代理直接访问

`/etc/profile` 配置一下内容

```go
export GOPRIVATE="git.private.com"
```

使用 `source /etc/profile` 刷新配置，或者重新开一个terminal，如果有下面红色部分，说明生效

```go
$ go env
GO111MODULE="on"
GOARCH="amd64"
GOBIN=""
GOCACHE="/root/.cache/go-build"
GOENV="/root/.config/go/env"
GOEXE=""
GOEXPERIMENT=""
GOFLAGS=""
GOHOSTARCH="amd64"
GOHOSTOS="linux"
GOINSECURE=""
GOMODCACHE="/root/go/pkg/mod"
GONOPROXY="git.private.com"
GONOSUMDB="git.private.com"
GOOS="linux"
GOPATH="/root/go"
GOPRIVATE="git.private.com"
GOPROXY="https://goproxy.cn/"
GOROOT="/usr/local/go"
GOSUMDB="sum.golang.org"
GOTMPDIR=""
GOTOOLDIR="/usr/local/go/pkg/tool/linux_amd64"
GOVCS=""
GOVERSION="go1.17.5"
GCCGO="gccgo"
AR="ar"
CC="gcc"
CXX="g++"
CGO_ENABLED="1"
GOMOD="/dev/null"
CGO_CFLAGS="-g -O2"
CGO_CPPFLAGS=""
CGO_CXXFLAGS="-g -O2"
CGO_FFLAGS="-g -O2"
CGO_LDFLAGS="-g -O2"
PKG_CONFIG="pkg-config"
GOGCCFLAGS="-fPIC -m64 -pthread -fmessage-length=0 -fdebug-prefix-map=/tmp/go-build2259252641=/tmp/go-build -gno-record-gcc-switches"
```

## 设置多级子group

如果我们的私有仓库项目存在多级子group，比如：`git.private.com/sfeng/test/test.git` 那么也是无法拉取的，

新建~/.netrc文件，配置git.private.com登录信息，样式如下：

```go
machine git.private.com
login xxx
password xbyeUsBB6VkLKfsMyYs-
```
