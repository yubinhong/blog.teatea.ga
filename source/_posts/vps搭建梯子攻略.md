---
title: vps 搭建梯子攻略
date: 2019-06-12 14:16:19
tags:
categories: vps
keywords: vps bbr shadowsocks v2ray 梯子
description: vps安装bbr提高速度
---
因为某些原因，我们通过vps搭建梯子访问外网，但是一般速度都不快，需要安装一些工具进行加速。
### 1.安装bbr内核
``` bash
cd /usr/src && wget -N --no-check-certificate "https://raw.githubusercontent.com/chiakge/Linux-NetSpeed/master/tcp.sh" && chmod +x tcp.sh && ./tcp.sh
```
注意在弹出的安装界面首先选择1，安装BBR内核,安装过程可能时间较长,耐心等待。

### 2.安装BBR魔改版加速
``` bash
cd /usr/src && ./tcp.sh
```
在弹出安装界面,输入5,然后回车，使用BBR魔改版加速，等待安装完成提示bbr启动成功即可。

### 3.安装shadowsocks
``` bash
cd /usr/src/ && wget -N --no-check-certificate "https://raw.githubusercontent.com/yubinhong/tools/master/shadowsocks-all.sh" && chmod +x shadowsocks-all.sh && ./shadowsocks-all.sh
```
按照提示安装就可以了。

