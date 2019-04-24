---
title: zabbix小版本升级
date: 2019-04-15 16:18:26
tags:
categories: zabbix
---
#### 我们公司之前使用的zabbix版本为3.4.10,因为安全问题,现在要把版本升级到3.4最高版本3.4.15。升级步骤如下：
### 1.备份数据库和告警脚本
``` bash
mkdir -p /zabbix_dir/{sql,scripts} #创建备份目录
cd /zabbix_dir/sql
mysqldump -uzabbix -pzabbix --opt --skip-lock-tables --flush-logs --database zabbix --ignore-table=zabbix.history --ignore-table=zabbix.history_log --ignore-table=zabbix.history_str --ignore-table=zabbix.history_text --ignore-table=zabbix.history_uint --ignore-table=zabbix.trends --ignore-table=zabbix.trends_uint > zabbix.sql  #备份数据库
cd ../scripts
cp -a /usr/lib/zabbix/alertscripts .   #备份告警脚本
```

### 2.升级安装zabbix-server

#### ubuntu 16.04
``` bash
wget https://mirrors.tuna.tsinghua.edu.cn/zabbix/zabbix/3.4/ubuntu/pool/main/z/zabbix/zabbix-server-mysql_3.4.15-1+xenial_amd64.deb
wget https://mirrors.tuna.tsinghua.edu.cn/zabbix/zabbix/3.4/ubuntu/pool/main/z/zabbix/zabbix-frontend-php_3.4.15-1+xenial_all.deb
dpkg -i zabbix-server-mysql_3.4.15-1+xenial_amd64.deb
dpkg -i zabbix-frontend-php_3.4.15-1+xenial_all.deb
systemctl restart zabbix-serveer
```

#### centos 7.x
``` bash
yum install https://mirrors.tuna.tsinghua.edu.cn/zabbix/zabbix/3.4/rhel/7/x86_64/zabbix-server-mysql-3.4.15-1.el7.x86_64.rpm
systemctl restart zabbix-server
```

### 3.升级安装zabbix-proxy(如果有)

#### ubuntu 16.04 
``` bash
wget https://mirrors.tuna.tsinghua.edu.cn/zabbix/zabbix/3.4/ubuntu/pool/main/z/zabbix/zabbix-proxy-mysql_3.4.15-1+xenial_amd64.deb
dpkg -i zabbix-proxy-mysql_3.4.15-1+xenial_amd64.deb
systemctl restart zabbix-proxy
```

#### centos 7.x
``` bash
yum install https://mirrors.tuna.tsinghua.edu.cn/zabbix/zabbix/3.4/rhel/7/x86_64/zabbix-proxy-mysql-3.4.15-1.el7.x86_64.rpm
systemctl restart zabbix-proxy
```

### 4.升级安装zabbix-agent(可选)

#### ubuntu 16.04
``` bash
wget https://mirrors.tuna.tsinghua.edu.cn/zabbix/zabbix/3.4/ubuntu/pool/main/z/zabbix/zabbix-agent_3.4.15-1+xenial_amd64.deb
dpkg -i zabbix-agent_3.4.15-1+xenial_amd64.deb
systemctl restart zabbix-agent
```

#### centos 7.x
``` bash
yum install https://mirrors.tuna.tsinghua.edu.cn/zabbix/zabbix/3.4/rhel/7/x86_64/zabbix-agent-3.4.15-1.el7.x86_64.rpm
systemctl restart zabbix-agent
```
