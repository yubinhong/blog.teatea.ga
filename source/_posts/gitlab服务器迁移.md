---
title: gitlab服务器迁移
date: 2019-03-28 17:17:38
tags:
categories: gitlab
---
### 公司之前的gitlab搭在阿里云上，考虑到安全问题，需要将gitlab迁移到本地服务器。下面记录下步骤：
### 1.迁移准备工作和思路:从a服务器迁移到b服务器,由于Gitlab自身的兼容性问题，高版本的Gitlab无法恢复低版本备份的数据,需要注意在b服务器部署和a服务器一样版本的gitlab,部署好环境后开始备份和数据迁移。
``` bash
#查看gitlab版本号
gitlab-rake gitlab:env:info
```
### 2.备份原gitlab服务器的数据
``` bash
gitlab-rake gitlab:backup:create RAILS_ENV=production
#PS: 备份后的文件一般是位于/var/opt/gitlab/backups下, 自动生成文件名文件名如1553575861_2019_03_26_11.4.3-ee_gitlab_backup.tar
```
### 3.将步骤2生成的tar文件拷贝到本地服务器上相应的backups目录下
``` bash
scp username@src_ip:/var/opt/gitlab/backups/1553575861_2019_03_26_11.4.3-ee_gitlab_backup.tar /var/opt/gitlab/backups
#PS: username为原服务器的用户名，src_ip原服务器IP地址
```
### 4.在本地服务器上恢复数据
``` bash
#PS:gitlab恢复数据时需要将1553575861_2019_03_26_11.4.3-ee_gitlab_backup.tar改名为1553575861_gitlab_backup.tar
#将1553575861_gitlab_backup.tar更改属主属组为git.git
mv 1553575861_2019_03_26_11.4.3-ee_gitlab_backup.tar 1553575861_gitlab_backup.tar
chown git.git 1553575861_gitlab_backup.tar
gitlab-rake gitlab:backup:restore RAILS_ENV=production BACKUP=1553575861
```
