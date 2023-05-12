---
layout:     post
title:      "远程调试 golang 程序"
description: "远程调试 golang 程序"
date:     2022-05-06
author: sfeng
tags: ["Golang"]
URL: "/2022/05/remote-debug/"
---

## 背景

远程调试在工作中非常重要，当线上环境出现问题需要 debug 来定位时，就需要使用远程 debug。因为一般很难在本地环境搭出与线上一致的环境，所以需要远程 debug 就能实现远程连接到线上程序实现 debug。

## 步骤

### 安装 dlv

在远程环境安装 dlv，这个远程环境是你程序运行的环境，并不是你的本地环境，具体安装方法

需要提前安装 go sdk，下面命令执行成功，dlv 二进制程序会在 $GOPATH/bin 目录下

```go
go get github.com/go-delve/delve/cmd/dlv
```

### 运行程序

在远程话环境使用 dlv 运行 go 程序，注意 go 程序需要禁止内联优化，不然 dlv 会报错，可以使用下面命令重新编译

```go
go build -o test -gcflags '-l' ./main.go
```

使用 dlv 运行程序，程序需要添加执行参数，可以使用 —

dlv 也可以直接运行来命令行窗口调试，但是太非人类了，结合goland会更友好。如果本地连接不了远程环境那就只能使用 dlv 命令行调试了。

```go
./dlv --listen=:2345 --headless=true --api-version=2 --accept-multiclient --check-go-version=false  exec ./test -- args=test
```

### goland 配置

在本地goland 配置远程 dlv

![](/img/remote-debug.png)

然后就可以点击 goland debug 愉快的调试了。