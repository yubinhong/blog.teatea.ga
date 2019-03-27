---
title: pip设置国内源
date: 2019-03-27 11:17:02
tags:
categories: python
---
### 因为某个不可描述的墙，导致pip下载包异常的慢，所以需要把pip的源改成国内的。
下面列举几个国内的源:
``` bash
http://pypi.douban.com/       豆瓣
http://pypi.hustunique.com/ 华中理工大学
http://pypi.sdutlinux.org/ 山东理工大学
http://pypi.mirrors.ustc.edu.cn/ 中国科学技术大学
http://mirrors.aliyun.com/pypi/simple/  阿里云
https://pypi.tuna.tsinghua.edu.cn/simple/ 清华大学
```
### 具体配置如下：
``` bash
#在当前用户主目录创建.pip文件夹
>cd ~
>mkdir .pip
```
``` bash
#在.pip目录创建并编辑pip.conf
#pip安装需要使用的https加密，所以在此需要添加trusted-host 
[global]
trusted-host = mirrors.ustc.edu.cn
index-url = https://mirrors.ustc.edu.cn/pypi/web/simple
```
