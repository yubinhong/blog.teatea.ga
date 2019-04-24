---
title: ubuntu版本升级
date: 2019-04-17 16:39:56
tags:
categories: linux
---
公司里有一台服务器的系统是ubuntu 16.04，现在想要升级到18.04，以下是升级步骤：

#### 1、安装update-manager-core
``` bash
sudo apt install update-manager-core
```

#### 2、修改/etc/update-manager/release-upgrades
``` bash
sudo vim /etc/update-manager/release-upgrades
修改Prompt为lts
```
#### 3、升级
``` bash
sudo sudo do-release-upgrade
```
