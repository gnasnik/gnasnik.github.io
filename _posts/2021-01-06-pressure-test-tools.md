---
layout:     post
title:      "压力测试工具"
subtitle:   ""
date:       2021-01-06 15:02:26
author:     "frank"
# header-style: text
header-img: img/home-bg-o.jpg
tags:
    - 编程
---

# 压力测试工具

## wrk 

安装

```
git clone https://github.com/wg/wrk.git 
cd wrk
make 
cp wrk /usr/local/bin
```

用法：

```
wrk -c 1000 -t 10 -d 30 http://127.0.0.1:2345/miner/fetch\?taskType\="seal/v0/addpiece"\&count\=1
Running 30s test @ http://127.0.0.1:2345/miner/fetch?taskType=seal/v0/addpiece&count=1
  10 threads and 1000 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency   138.48ms    8.35ms 309.99ms   84.45%
    Req/Sec   723.77    205.93     1.01k    63.43%
  216245 requests in 30.07s, 24.95MB read
Requests/sec:   7192.23
Transfer/sec:    849.86KB
```

从结果可以看出来，QPS为 7192

参数说明
```
Usage: wrk <options> <url>                            
  Options:                                            
    -c, --connections <N>  Connections to keep open   
    -d, --duration    <T>  Duration of test           
    -t, --threads     <N>  Number of threads to use   
                                                      
    -s, --script      <S>  Load Lua script file       
    -H, --header      <H>  Add header to request      
        --latency          Print latency statistics   
        --timeout     <T>  Socket/request timeout     
    -v, --version          Print version details 
```

| 参数 | 说明 |
| ---- | ---- |
| -c   | 保持的连接数 |
| -d   | 压力测试时间， 单位秒|
| -t   | 启动的线程数量|
| -s   | 执行 lua 脚本 |
| -H   | 设置 http header 信息|