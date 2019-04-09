---
title: gitlab升级
date: 2019-04-08 14:35:00
tags:
categories: gitlab
---
#### gitlab属于比较重要的服务，需要时常关注安全问题，官方有安全更新时，都应该尽快进行升级。升级步骤：(这里主要基于yum和apt-get升级)
### 1.关闭unicorn，sidekiq和nginx服务
``` bash
gitlab-ctl stop unicorn
gitlab-ctl stop sidekiq
gitlab-ctl stop nginx
```
### 2.备份gitlab
``` bash
gitlab-rake gitlab:backup:create
```
### 3.升级gitlab
``` bash
#centos7
yum update gitlab-ce
#ubuntu
apt-get install gitlab-ce
```
### 4.启动unicorn，sidekiq和nginx服务
``` bash
gitlab-ctl start unicorn
gitlab-ctl start sidekiq
gitlab-ctl start nginx
```
