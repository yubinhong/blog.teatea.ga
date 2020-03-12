---
title: 'ELK 使用系列:一、ELK安装'
date: 2019-04-02 10:48:53
tags:
categories: ELK
keywords: ELk elk安装
description: ELK安装
---
#### 现在许多公司都开始使用ELK进行日志的收集，分析以及展示。今天我就讲讲ELK的安装，后续会继续深入讲解。
### 一、安装环境
``` bash
系统：centos7
配置：2核8G
JDK版本：1.8.0_191
```
### 二、安装步骤
``` bash
#下载最新版elasticsearch,kibana,logstash
bash# wget https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-6.7.0.tar.gz
bash# wget https://artifacts.elastic.co/downloads/kibana/kibana-6.7.0-linux-x86_64.tar.gz
bash# tar xf elasticsearch-6.7.0.tar.gz -C /usr/local/
bash# tar xf kibana-6.7.0-linux-x86_64.tar.gz -C /usr/local/
bash# ln -s /usr/local/elasticsearch-6.7.0 /usr/local/elasticsearch
bash# ln -s /usr/local/kibana-6.7.0 /usr/local/kibana
```
### 三、配置elastcisearch
``` bash
#打开/usr/local/elasticsearch/config/elasticsearch.yml
bash# vim /usr/local/elasticsearch/config/elasticsearch.yml
修改如下内容：
cluster.name: test-elastic   #这个可以自己取名
path.data: /data/elastic/data  #存储数据的目录，可自定义
path.logs: /data/elastic/logs  #存储日志的目录，可自定义
network.host: 127.0.0.1  #监听的ip
http.port: 10086  #监听的http端口
```
``` bash
#打开/usr/local/elasticsearch/config/jvm.options
bash# vim /usr/local/elasticsearch/config/jvm.options
修改如下内容：
-Xms4g
-Xmx4g
PS.根据服务器配置可以相应调整内存参数。
```
创建目录
``` bash
bash#mkdir -p /data/elastic/{data,logs}
```
#### 因为elastci不能使用root用户启动，新建一个普通用户
``` bash
bash# useradd ybh
bash# chown -R ybh.ybh /data/elastic /usr/local/elasticsearch /usr/local/kibana
```
### 四、配置kibana
``` bash
#打开/usr/local/kibana/config/kibana.yml
bash# vim /usr/local/kibana/config/kibana.yml
修改如下内容：
server.port: 10087 #kibana 监听端口
server.host: "0.0.0.0" #kibana 监听地址
elasticsearch.url: "http://localhost:10086" #elasticsearch的地址
elasticsearch.requestTimeout: 300000 #elasticsearch 超时时间
```
### 五、安装supervisor
#### 使用supervisor来管理kibana,做成后台服务，方便管理
``` bash
bash# yum install supervisor
bash# vim /etc/supervisord.d/kibana.ini
添加如下内容：
[program:kibana]
directory=/usr/local/kibana
command=/usr/local/kibana/bin/kibana serve -c /usr/local/kibana/config/kibana.yml
user=ybh
autostart=true
autorestart=true
startretries=1
startsecs=1
redirect_stderr=true
stdout_logfile=/data/logs/kibana/kibana.log
stdout_logfile_maxbytes=5MB
stdout_logfile_backups=5
stopasgroup=true
killasgroup=true
bash# mkdir -p /data/logs/kibana
bash# chown -R ybh.ybh /data/logs
```
### 六、启动elasticsearch,kibana
``` bash
bash# cd /usr/local/elasticsearch/bin
bash# ./elasticsearch -d
bash# supervisorctl start kibana
```
### 七、验证
``` bash
bash# curl http://127.0.0.1:10086
{
  "name" : "iS7URyg",
  "cluster_name" : "test-elastic",
  "cluster_uuid" : "U1vyeKlnSoSoQXitZ5r0SQ",
  "version" : {
    "number" : "6.7.0",
    "build_flavor" : "default",
    "build_type" : "tar",
    "build_hash" : "fe40335",
    "build_date" : "2018-10-30T23:17:19.084789Z",
    "build_snapshot" : false,
    "lucene_version" : "7.4.0",
    "minimum_wire_compatibility_version" : "5.6.0",
    "minimum_index_compatibility_version" : "5.0.0"
  },
  "tagline" : "You Know, for Search"
}
PS.当出现如上字样，表示elasticsearch启动成功了。
```
#### 访问http://ip:10087，看是否成功，如果成功，将看到如下页面
![](/images/kibana.png)
#### logstash需要与filebeat一起进行配置，才能起作用，后续再来。
