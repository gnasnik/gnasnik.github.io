---
layout:     post
title:      "什么是 npx"
subtitle:   ""
date:       2021-11-26 2:49:26
author:     "frank"
header-style: text
#header-img: img/home-bg-o.jpg
tags:
    - 前端
---

首先，我们知道 `npm` 是包管理工具，像执行 `npm install` 可以下载相关的依赖包到本地或全局 `node_module` 目录下，那么 `npx` 又是什么呢？ 

`npx` 可以说是 `npm` 提供的一个独立的工具，在安装 `npm` 的时候一起安装到本地。 `npx` 指的是 node package excute, 即用来运行某个包。

用法：

```
» npx @vue/cli create my-vue-name

Need to install the following packages:
  @vue/cli
Ok to proceed? (y) 

```

`npx` 会先检查本地有没有安装对应的包，如果没有直接使用远程服务器的包，不需要下载到本地。
 
