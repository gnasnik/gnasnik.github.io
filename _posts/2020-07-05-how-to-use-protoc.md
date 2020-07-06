---
layout:     post
title:      "protoc 编译 protobuf 详解"
subtitle:   ""
date:       2020-07-05 23:49:50
author:     "frank"
# header-style: text
header-img: "img/post-bg-alitrip.jpg"
tags:
    - 编程
---

# protoc 编译 protobuf 详解

## protoc 下载

下载 protoc ,选择合适版本, [官网 Release 链接](https://github.com/protocolbuffers/protobuf/releases) , 解压后把 `protoc` 拷贝到 `/usr/local/bin` 目录下

## protoc 编译

**用法:**   
```
protoc [OPTION] PROTO_FILES
```

下面是几个比较重要的参数,详细参数使用 `protoc --help` 查看   

```
-IPATH, --proto_path=PATH  指定 import 的目录，可以指定多次，默认为当前目录
--descriptor_set_out=FILE 根据 descriptor.proto 生成一个描述文件
--plugin=EXECUTABLE 指定 PATH 目录外插件可执行文件，默认是在 PATH 里寻找
--go_out=OUT_DIR 使用 go 插件，这里的插件可以替换为其他语言的，如 c,java,js
@<filename> 从文件中读取参数
```

**protoc-gen-go**

protoc-gen-go 是 protobuf 编译系列中的一个 go 插件，可以用来生成 golang 版本 protobuf 协议文件，因为是 go 写的，所以安装很简单

```
go get -u github.com/golang/protobuf/protoc-gen-go
```

执行完之后就可以在 $GOPATH/bin 目录下找到这个可执行文件，当 `protoc` 指定 `--go_out=` 参数时，`protoc` 会自动到环境中去寻找 `protoc-gen-go` 插件


`protoc-gen-go`主要参数，多个参数之间使用 , 分开
```
plugins= 指定 protoc-gen-go 编译时使用的插件，如 grpc ,micro 
paths= import 或者 source_relative 选项, import 按 proto 文件中的 op_package 来生成目录层级，source_relative 按 go 的源文件来生成目录层级，默认是 import 
```

官方推荐在 proto 文件中指定 op_package 声明，可以让其他 go 依赖包正确 import 到本 go 包，**所以在 proto 文件中指定了 op_package 声明之后，就必须使用 source_relative 参数**, `op_option` 与 `path=source_relative` 应该成对出现，更多参数查阅[ golang/protobuf 文档](https://github.com/golang/protobuf#parameters)

另外还可以给 `protoc-gen-go` 添加插件，如 grpc, micro，`protoc-gen-go` 同样会从 PATH 中寻找这些插件，所以需要自行下载对应的插件。
```
# grpc 插件
protoc --go_out=plugins=grpc:. hello.proto

# micro 插件
protoc --go_out=plugins=micro:. hello.proto
```


使用 grpc 插件之后生成的协议文件会多一些 grpc 通讯相关的方法，如 `RegisterWorkerServiceServer` 和 `NewWorkerServiceClient`, 可以简单的创建一个 grpc 的服务端和客户端。

当然，想自己实现一个编译插件也可以的, 只要插件名称遵循 `protoc-gen-xxx` 规则,然后在编译时指定参数, 使用 protoc 生成协议文件时指定 `--xxx_out=`  参数，例如：

```
protoc --micro_out=. hello.proto
```

在编译 go 语言的 protobuf 微服务协议时，推荐使用 `protoc-gen-micro` 插件, 使用方法

```
protoc --micro_out=:. --go_out=:. hello.proto
```

官方解释：
>We use protoc-gen-micro to reduce boilerplate code.


对于`protoc-gen-go`来说，使用`protoc-gen-micro`可以减少生成不必要的模板文件。



**参考文档：**      
[protoc-gen-go 介绍与源代码分析](https://blog.csdn.net/u013272009/article/details/100018002)