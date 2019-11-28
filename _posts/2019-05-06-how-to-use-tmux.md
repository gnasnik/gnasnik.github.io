---
layout:     post
title:      "Tmux 使用教程"
subtitle:   ""
date:       2019-05-06 14:00:00
author:     "frank"
# header-style: text
header-img: "img/post-bg-halting.jpg"
tags:
    - Tmux
    - 编程
    - Linux
---

## Tmux 有什么用

开发过程中，我们经常要连接远程服务器来部署服务或者查看日志。以前我们在 Mac 使用 `Termial` , 在 Windows 上使用 `Xshell` 或者其他远程连接工具，但经常会出现这样的情况，SSH 连接长时间没收到数据包自动断开，或者遇到断网、程序意外关闭，在这种情况下，我们要重新使用 SSH 连接远程的服务器，但是之前打开的记录全丢失了，找不到上一次的命令，这非常麻烦，`Tmux` 就是来解决这一个问题的，只要我们在服务器安装 `Tmux`, 然后使用 `Tmux` 的会话窗口，即使关闭连接，服务器上仍然保留 `Tmux` 的状态，重新连上服务器后打开 `Tmux` 即可。

## 开始安装

这里以centos7为例

- 通过yum安装tmux 1.8版本：

```
yum install -y tmux
```
- 编译安装tmux 2.x 以上版本, 现在最高版本是 tmux-3.0 ：

```
wget https://github.com/tmux/tmux/releases/download/3.0/tmux-3.0.tar.gz      
tar zxvf tmux-3.0.tar.gz           
cd tmux-3.0/                   
./configure && make             
make install                    
```

## 快捷键

`Tmux` 使用起来非常方便高效，可以通过终端敲打命令，但更好用的是快捷键，只要熟练掌握几个常用的快捷键，又可以装 X 又提升工作效率。

常用的命令
```
tmux 新建一个会话

tmux ls 查看会话

tmux -V 查看当前版本
```

前缀键 <kbd>Ctrl</kbd>+<kbd> B </kbd> +  `?` , 选按前缀键，再按下面的快捷键

|  窗口快捷键   | 描述  | 面板快捷键   | 描述  |
|  ----  | ----  | ----  | ----  |
| c  | 创建新窗口 |! | 当前面板创建新窗口 |
| &  | 关闭当前窗口 |x  | 关闭当前面板 |
| n  | 切换至下一窗口 | "| 垂直新建面板 |
| p  | 切换至上一窗口 | %| 水平新建面板 |
| 数字键  | 切换指定一窗口 |方向键 | 切换到指定面板 | 
| l  | 前后两个窗口进行切换 | ;| 前后两个面板进行切换 | 
| w  | 通过窗口列表切换窗口 | o| 切换至下一面板 |
| ,  | 重命名当前窗口 | { | 前置当前面板 |
| .  | 移动窗口位置 |  }  | 后置当前面板 |
| f  | 在所有窗口中查找 | z | 全屏/恢复 当前面板

还有其他系统的快捷键, 前缀键同样是 <kbd>Ctrl</kbd>+<kbd> B </kbd> ：

```
d  脱离当前会话
D  选择要脱离的会话
ctrl+z   挂起当前会话
s  通过列表切换会话
[ 复制模式，窗口可以滚动
t 显示当前时间
```

## 修改前缀键

有些同学可能觉得 <kbd>Ctrl</kbd>+<kbd> B </kbd>  这个组合键太远(~~手短~~)，按起来不是很方便，没关系，我们在 `~/.tmux.conf` 里可以自己定义。

- 查看当前的设置

```
tmux show-options -g | grep prefix 

查询结果
#prefix C-b 默认是 Ctrl+b
#prefix2 none
```

- 即时生效可以在命令行中修改

```
tmux set -g prefix C-x
tmux unbind C-b 
tmux bind C-x send-prefix
```

- 永久修改 `vim ~/.tmux.conf` 

```
set -g prefix C-x
unbind C-b
bind C-x send-prefix
```
