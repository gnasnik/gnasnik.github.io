---
layout:     post
title:      "Shell 笔记"
subtitle:   ""
date:       2019-01-02 22:32:00
author:     "frank"
header-style: text
tags:
    - 编程
---

# Shell 笔记

在第一行指定执行的 shell 程序，如 `#! bin/bash` 或 `#！/bin/sh` 让系统知道用什么 shell 去执行脚本，如果没有指定，那么可以在命令行中用 shell 来执行

```
    sh run.sh
```
  
> #！ /bin/bash 于 /bin/sh 通用，多用于实际


**语法**
- 声明变量 name=lin 之间不能有空格
- echo 标准打印
- 使用变量 ${name}
- 取得变量长度${#name}
- 声明数组 names=(a b c d) 没有逗号
- 取得所有元素 ${names[@]} 或者*
- 取得数组长度 ${#names[@]} 或者*
- 数值测试
```
if test $[nums1] -eq $[nums2] 
then 
        xxxxx
else
        xxxxx
fi
```
- 字符串测试   
```
if  test $str1 = $str2
then 
    xxxxx
else
    xxxxx
fi
```

 **系统变量**

- \$0 文件名
- \$1 第一个参数
- \$2 第二个参数
- \$* 所有参数
- \$? 上一个命令是否执行成功 0=成功
- \$# 统计参数的个数
- \$UID 系统用户 ，0=root用户


**echo**
- e 可以加颜色 \033[32m\033[1m


**if**  
- if (())；then 两个括号判断值是否相等 
- If []；then 中括号判断逻辑
- \-f 文件是否存在
- \-d 目录是否存在
- \-z 空字符串
- \>> 追加到文件

例子  （两边要有空格 变量常量要用引号引越来）
```
if [ "$1" = "100" ];then
        echo 我是100
else
    echo 我是0
fi
```
**反引号可执行**
- 当前日期date +%Y%m%d


PS3 是select之后的提示语

```	
select i in 安装mysql 安装php 安装golang
do
        case $i in  ## 这里有 in
                安装mysql)  ## 有半括号
                echo 我是mysql
                break
                ;;    ## 双分号
                安装php)
                echo 我是php
                break
                ;;
                安装golang)
                install_golang ## 调用函数不用括号
                break
                ;;
        esac
done
```

- seq 1 5 循环打印 1-5 
- expr 1 + 2 计算1 + 2
- find . -name '*.sh' 找出当前 .sh 的目录
- tar xvf  -C 指定目录
- ls -l | tail -2 显示最后两行
- -maxdepth 当前第一级目录
- ssh-keygen 生成密钥


read -p 提示，键盘输入
```
while read line
do
	echo $line
done < abc.txt

while [[$? -eq 0 ]];then
	((I++))
done
```

**sed**
- sed 's/old/new/g' 替换
- sed -i s/old/new/g 插入替换
- sed -i /xxx/a test 插入一行test
- sed -i s/^/&id/ 开头添加字符
- sed -n 1，5p 打印一到五行
- sed -n 1；$p打印第一和最后一行

**awk**
- awk {print \$1} 打印那一列 \$NF是最后一列
- awk -F: 按字符分割

组合使用
```
df -h | grep '/$' |awk ‘{print $5}’|sed 's/%/ /g'
```
在第一列前面加字符
```
`awk ‘{print 'add' $1}’
```
**find**
```
find .  -name '*.go'
        -maxdepth 1 目录层级
        -type f/r 文件类型 目录或者根目录
        -mtime +30 /-1 最后修改时间 30天之前/当天修改
        -exec rm -rf ｛｝\； 删除查找到的文件
        -size +50 文件大于50m的
```
**uniq -c 去重**
**sort 排序 sort -nr 倒序**
