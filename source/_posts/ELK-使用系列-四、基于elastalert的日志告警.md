---
title: ELK 使用系列 四、基于elastalert的日志告警
date: 2019-05-08 16:25:16
tags:
categories: ELK
---
前面已经介绍了ELK搭建和使用，日志已经收集起来，当日志出现异常，我们需要第一时间知道，而我们又不能每时每刻都在查看日志，
这时候我们就可以用上elastalert了。
### 一、elastalert 的安装
Elastalert是Yelp公司用python2写的一个报警框架(目前支持python2.6和2.7，不支持3.x),github地址为 
[https://github.com/Yelp/elastalert](https://github.com/Yelp/elastalert)
相关依赖可以查看[文档](https://elastalert.readthedocs.io/en/latest/running_elastalert.html#requirements).
文档中使用的ubuntu 14.x，我使用的是centos，这个不影响使用。

安装
使用pip安装
``` bash
pip install elastalert
```
或者基于源码安装
``` bash
git clone https://github.com/Yelp/elastalert.git
python setup.py install
pip install -r requirements.txt
```

### 二、设置elasticsearch索引
参见[setting-up-elasticsearch](http://elastalert.readthedocs.io/en/latest/running_elastalert.html#setting-up-elasticsearch)
elastalert-create-index 这个命令会在elasticsearch创建索引，这不是必须的步骤，但是强烈建议创建。因为对于，审计，测试很有用，
并且重启elastalert不影响计数和发送alert,默认情况下，创建的索引叫 elastalert_status.
``` bash
elastalert-create-index
New index name (Default elastalert_status)
Name of existing index to copy (Default None)
New index elastalert_status created
Done!
```

### 三、配置文件
``` bash
mkdir /data/alert_rule -p
vi /data/alert_rule/config.yaml
```
``` bash
# This is the folder that contains the rule yaml files
# Any .yaml file will be loaded as a rule
rules_folder: /data/alert_rule/

# How often ElastAlert will query Elasticsearch
# The unit can be anything from weeks to seconds
run_every:
  minutes: 1

# ElastAlert will buffer results from the most recent
# period of time, in case some log sources are not in real time
buffer_time:
  minutes: 15

# The Elasticsearch hostname for metadata writeback
# Note that every rule can have its own Elasticsearch host
es_host: 192.168.2.16

# The Elasticsearch port
es_port: 10086

# The AWS region to use. Set this when using AWS-managed elasticsearch
#aws_region: us-east-1

# The AWS profile to use. Use this if you are using an aws-cli profile.
# See http://docs.aws.amazon.com/cli/latest/userguide/cli-chap-getting-started.html
# for details
#profile: test

# Optional URL prefix for Elasticsearch
#es_url_prefix: elasticsearch

# Connect with TLS to Elasticsearch
#use_ssl: True

# Verify TLS certificates
#verify_certs: True

# GET request with body is the default option for Elasticsearch.
# If it fails for some reason, you can pass 'GET', 'POST' or 'source'.
# See http://elasticsearch-py.readthedocs.io/en/master/connection.html?highlight=send_get_body_as#transport
# for details
#es_send_get_body_as: GET

# Option basic-auth username and password for Elasticsearch
#es_username: someusername
#es_password: somepassword

# Use SSL authentication with client certificates client_cert must be
# a pem file containing both cert and key for client
#verify_certs: True
#ca_certs: /path/to/cacert.pem
#client_cert: /path/to/client_cert.pem
#client_key: /path/to/client_key.key

# The index on es_host which is used for metadata storage
# This can be a unmapped index, but it is recommended that you run
# elastalert-create-index to set a mapping
writeback_index: elastalert_status

disable_rules_on_error: false
# If an alert fails for some reason, ElastAlert will retry
# sending the alert until this time period has elapsed
alert_time_limit:
  days: 2
realert: 0
old_query_limit:
  days: 1
```

``` bash
vim /data/alert_rule/example_rule.yaml
```
``` bash
# From example_rules/example_frequency.yaml
es_host: 192.168.2.16
es_port: 10086
name: xxxxx Alert Log
#type: frequency
type: any
index: logstash-*
num_events: 1
timeframe:
   minutes: 1

filter:
  - query:
    - query_string:
        query: "message:'Traceback' OR message:*Exception* OR source:*Defend*"

alert:
- "alert_plugin.wechat_qiye_alert.WeChatAlerter"
agent_id: "1000006"
#party_id: "7"
user_id: "@all"
```
这边告警插件使用自己编写的，基于企业微信进行告警。详情可以查看我的github项目[wxapi](https://github.com/yubinhong/wxapi).

``` bash
mkdir /data/alert_rule/alert_plugin
cd  /data/alert_rule/alert_plugin
touch __init__.py
vim wechat_qiye_alert.py
```

``` bash
#! /usr/bin/env python
# -*- coding: utf-8 -*-

import json
import datetime
from elastalert.alerts import Alerter, BasicMatchString
from requests.exceptions import RequestException
from elastalert.util import elastalert_logger,EAException #[感谢minminmsn分享](https://github.com/anjia0532/elastalert-wechat-plugin/issues/2#issuecomment-311014492)
import requests
import hashlib
import time
import sys

class WeChatAlerter(Alerter):
    def __init__(self, *args):
        super(WeChatAlerter, self).__init__(*args)
        self.agent_id = self.rule.get('agent_id', '1000006')
        self.user_id = self.rule.get('user_id', '@all')

    def create_default_title(self, matches):
        subject = 'ElastAlert: %s' % (self.rule['name'])
        return subject

    def alert(self, matches):
        matches2 = []
        for match in matches:
            match2 = {}
            if len(match['message']) > 1024:
                match['message'] = match['message'][:1024] + '...'
            match2['host_name'] = match['fields']['feature']
            match2['aws_host'] = match['host']['name']
            match2['source'] = match['source']
            match2['message'] = match['message']
            match2['time'] = match['@timestamp']
            matches2.append(match2)
            if "wallet" in match2['host_name'] or "wallet" in match2['source']:
               if "qa" in match2['host_name']:
                  agent_id = '0'
               else:
                  agent_id = '100xxxx'
            elif "engine" in match2['host_name']:
               agent_id = '100xxxx'
            elif "api" in match2['host_name'] or "admin" in match2['host_name'] or "php" in match2['host_name']:
               agent_id = '100xxxx'
            else:
               agent_id = '100xxxx'
        body = self.create_alert_body(matches2)
        if agent_id != '0':
           self.senddata(body, agent_id)
           elastalert_logger.info("send message to %s" % (agent_id))


    def senddata(self, content, agent_id):
        url="http://xx.xx.xx/api/v2/alert"
        data = {}
        data['user']=self.user_id
        data['agent_id']=agent_id
        data['content']=content
        current_time=int(time.time())
        data['time']='%s' % current_time
        secure_word='KM1yeVcFbrL44ZGv' + '%s' % current_time
        s=hashlib.md5()
        s.update(secure_word.encode(encoding='utf-8'))
        data['signature']=s.hexdigest()
        req=requests.post(url, data=data)
        elastalert_logger.info("send message and response: %s" % (req.content))

    def get_info(self):
        return {'type': 'WeChatAlerter'}
```

### 四、启动elastalert
我使用supervisor进行管理进程
``` bash
yum install supervisor
vim /etc/supervisor.d/alertalert.ini
```

``` bash
[program:elastalert]
command =  python -m elastalert.elastalert --verbose --rule example_rule.yaml
stdout_logfile = /data/logs/elastalert/startup.log
stderr_logfile = /data/logs/elastalert/error.log
user = onecafe
directory = /data/alert_rule
;environment = PYTHONPATH="$PYTHONPATH:/home/wbops/engine/bullet-comments-reflector"
environment=LC_ALL='en_US.UTF-8',LANG='en_US.UTF-8',PYTHONPATH="$PYTHONPATH:/data/alert_rule/"
autostart = true
startsecs = 5
autorestart = true
startretries = 3
stdout_logfile_maxbytes=5MB
stdout_logfile_backups=5
```
最后进行启动
``` bash
systemctl start supervisord
supervisorctl status
```

``` bash
elastalert                       RUNNING   pid 7010, uptime 3 days, 7:04:12
```
至此结束。
