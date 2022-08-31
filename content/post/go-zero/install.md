---
layout: post
title: 1、go-zero 安装
date: 2022-08-31
tags: 
  - golang
  - go-zero
categories:
  - golang
description: go-zero 是一个集成了各种工程实践的 web 和 rpc 框架。通过弹性设计保障了大并发服务端的稳定性，经受了充分的实战检验。
---
<!--more-->
## goctl 安装
```shell
# Go 1.15 及之前版本
GO111MODULE=on GOPROXY=https://goproxy.cn/,direct go get -u github.com/zeromicro/go-zero/tools/goctl

# Go 1.16 及以后版本
GOPROXY=https://goproxy.cn/,direct go install github.com/zeromicro/go-zero/tools/goctl@latest
```

## protoc & protoc-gen-go 安装

> protoc 是一款用 C++ 编写的工具，其可以将 proto 文件翻译为指定语言的代码。在 go-zer o的微服务中，我们采用 grpc 进行服务间的通信，而 grpc 的编写就需要用到 protoc 和翻译成 go 语言 rpc stub 代码的插件 protoc-gen-go。


### 这里使用 goctl 一键安装
```
goctl env check -i -f --verbose
```