---
layout:     post
title:      "Makefile 用法"
subtitle:   ""
date:       2019-01-05 11:00:00
author:     "frank"
header-style: text
tags:
    - 编程
---

# Makefile 使用

Makefile 是一个构建编译的文件，默认名为 makefile 或者 Makefile

Makefile 文件语法：		
```
<target> : <prerequisites> 
[tab]  <commands>
```

target 是目标文件，prerequisites 是需要的前置条件（依赖），commands 是要执行的操作， 如果 target 是“伪目标”，是一个操作的集合而不是指某个文件，那么就要给 target 声明
```
.PHONY: all
```

有了这个声明之后 make 就不会再去检查是否有 all 这个文件了

make 没有指定目标时，就会默认执行第一个目标，所以我们通过把每一个目标定义为 all ：			
```
all:  build 

build:
	go build -o test

.PHONY:  all build
```

每个 commands 命令之前必须要有一个 tab 键。
每条命令在单独的 shell 中执行，不会互相影响，并且不需要指定 shell , 如果想执行多次命令，可以写在同一条，用逗号分开， 如果太长，可以使用反斜杠换行。	
```
	# 同一行执行
	 cd center; rm *.pb.go

	# 反斜杠换行
	cd center; \
	rm *.pb.go
```

正常情况下，make 会打印每条命令，然后再执行，这叫 echoing, 可以在命令的前面加上 @ 关闭 echoing

Makefile 可以使用自定义变量，使用 $() 或者 ${} 来调用变量，这两个操作符也适用于调用 shell 变量，另外调用 shell 变量也可以使用 $$ 符号。		
```
GO = go
GOGET = $(GO) get

# 调用 gopath 环境变量写法
path:
	ehco ${GOPATH}
	echo $(GOPATH)
	echo $$GOPATH		
```

make 命令的自动变量
- $@ 指当前目标，如 make debug 中的 debug
- $< 指第一个前置条件
- $? 指比目标（时间戳）更新的所有前置条件
- $^ 指所胡的前置条件
- $* 指匹配 % 的部分
- $(@D) 指当前目标的目录名
- $(@F) 指当前目标的文件名
- $(<D) 指第一个前置条件的目录名
- $(<F) 指第一个前置条件的文件名

```
dest/%.txt: src/%.txt
    @[ -d dest ] || mkdir dest
    cp $< $@
```

目标文件 dest 目录下面的所有 txt 文件， 前置条件  src 目录下的所有 txt 文件， 然后 @ 关闭 echoing， [ -d dest ] 检查目录是否存在,  如果不存在就创建 dest 目录，复制第一个前置文件 到当前上件文件 

使用 bash 语法

```
ifeq ($(CC),gcc)
  libs=$(libs_for_gcc)
else
  libs=$(normal_libs)
endif


LIST = one two three
all:
    for i in $(LIST); do \
        echo $$i; \
    done

# 等同于

all:
    for i in one two three; do \
        echo $i; \
    done

```

用 shell 函数来执行 shell 命令
```
srcfiles := $(shell echo src/{00..99}.txt)

```

参考文档：

[阮一峰老师的 Make 命令教程](http://www.ruanyifeng.com/blog/2015/02/make.html)
