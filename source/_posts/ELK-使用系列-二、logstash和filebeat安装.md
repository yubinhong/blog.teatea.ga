---
title: ELK 使用系列 二、logstash和filebeat安装
date: 2019-04-04 11:20:53
tags:
categories: ELK
---
#### 上一节讲到elasticsearch和kibana的安装，这一节就来讲logstash和fielbeat安装。
### 一、filebeat的安装和配置
#### filebeat的作用就是日志采集，将采集到的日志发送到中间件，我这边以redis为例，也可以用其他的(kafka)等等。
``` bash 
bash# wget https://artifacts.elastic.co/downloads/beats/filebeat/filebeat-6.7.0-linux-x86_64.tar.gz
bash# tar xf filebeat-6.7.0-linux-x86_64.tar.gz -C /usr/local/
bash# ln -s /usr/local/filebeat-6.7.0-linux-x86_64 /usr/local/filebeat
```
#### 编辑/usr/local/filebeat/filebeat.yml进行配置
``` bash
#vim /usr/local/filebeat/filebeat.yml
filebeat.inputs:

- type: log
  paths:
    - /data/logs/nginx/*/*.log   #nginx日志
    - /data/logs/xxxx/*/*.log   #自主应用的日志
  #处理异常日志，合并成一行，方便查看
  multiline.pattern: '^method|^params|^data|^[[:space:]]{2,4}|^Traceback|.*Error:|^[[:space:]]+at|^Caused by:|.*Exception'
  multiline.negate: false
  multiline.match: after
  fields:
   feature: test-host-192.168.1.2   #自定义字段,用于区分服务器

#redis地址
output.redis:
  hosts: ["192.168.1.3"]
  port: 6379
  password: ""
```
#### 将filebeat做成系统服务，filebeat.service文件内容如下：
``` bash
bash# vim /etc/systemd/system/filebeat.yml
# It is not recommended to modify this file in-place, because it will
# be overwritten during package upgrades. If you want to add further
# options or overwrite existing ones then use
# See "man systemd.service" for details.

# Note that almost all daemon options could be specified in

[Unit]
Description=Filebeat Service
After=network.target

[Service]
ExecStart=/usr/local/filebeat/filebeat -c /usr/local/filebeat/filebeat.yml
User=root
Type=simple
Restart=on-failure

# Hardening measures
####################

# Provide a private /tmp and /var/tmp.
PrivateTmp=true

# Mount /usr, /boot/ and /etc read-only for the process.
#ProtectSystem=full

# Disallow the process and all of its children to gain
# new privileges through execve().
#NoNewPrivileges=true

# Use a new /dev namespace only populated with API pseudo devices
# such as /dev/null, /dev/zero and /dev/random.
#PrivateDevices=true

# Deny the creation of writable and executable memory mappings.
#MemoryDenyWriteExecute=true

[Install]
WantedBy=multi-user.target
```
### 二、安装logstash
#### logstash服务负责将redis的数据读取出来，经过处理写到elasticsearch。
``` bash
bash# wget https://artifacts.elastic.co/downloads/logstash/logstash-6.7.0.tar.gz
bash# tar xf logstash-6.7.0.tar.gz -C /usr/local/
bash# ln -s /usr/local/logstash-6.7 /usr/local/logstash
```
编辑/usr/local/logstash/config/client-redis.conf进行配置
``` bash
bash# vim /usr/local/logstash/config/client-redis.conf
input {
    redis {
        host => "192.168.1.3"
        port => "6379"
        key => "filebeat"
        data_type => "list"
        password => ""
        threads => 10
    }
}
filter{
 ####将nginx日志和自主应用日志分开处理
 if "/data/logs/nginx" in [source] {
   grok {
    patterns_dir => "/usr/local/logstash/patterns"
    match => { "message" => "%{NGINXACCESS}" }
   }
   geoip {
      source => "remote_addr"
   }

   date {
      match => [ "log_timestamp" , "dd/MMM/YYYY:HH:mm:ss Z" ]
      target => "@timestamp"
   }

 }else{
  grok {
   match => {"message" => "(?<datetime>\d{4}-\d{2}-\d{2}\s\d{2}:\d{2}:\d{2}\.\d{3})"}
  }
  date {
    match => ["datetime", "yyyy-MM-dd HH:mm:ss.SSS"]
    target => "@timestamp"
  }
  if [source] =~ "api" or [source] =~ "admin" {
   ruby {
    code => "event.set('timestamp', event.get('@timestamp').time.localtime - 8*60*60)"
   }
   ruby {
    code => "event.set('@timestamp', event.get('timestamp'))"
   }
  }
 }
}
###nginx和自主应用分别使用不同的模版和index
output {
   if "/data/logs/nginx" in [source] {
     elasticsearch {
        hosts => ["127.0.0.1:10086"]
        index => "logstash-nginx-%{+YYYY.MM.dd}"
        template => "/usr/local/logstash/config/nginx-template.json"
        template_name => "wb-nginx"
        template_overwrite => true
     }
   }else{
    elasticsearch {
        hosts => ["127.0.0.1:10086"]
        index => "logstash-%{+YYYY.MM.dd}"
        template => "/usr/local/logstash/config/app-template.json"
        template_name => "wb_index"
        template_overwrite => true
    }
   }
    stdout {codec => rubydebug}
}
```
#### 编辑/usr/local/logstash/config/nginx-template.json
``` bash
bash# vim /usr/local/logstash/config/nginx-template.json
{
  "aliases": {},
  "index_patterns": [
    "logstash-nginx-*"
  ],
  "mappings": {
    "doc": {
      "properties": {
        "body_bytes_sent": {
          "type": "long"
        },
        "request_length": {
          "type": "long"
        },
        "remote_addr": {
          "type": "ip"
        },
        "x_forword_for": {
          "type": "keyword"
        },
        "geoip": {
          "dynamic": true,
          "properties": {
            "location": {
              "type": "geo_point"
            }
          },
          "type": "object"
        },
        "request_time": {
          "type": "float"
        },
        "upstream_response_time": {
          "type": "float"
        },
        "upstream_status": {
          "type": "integer"
        },
        "http_status": {
          "type": "integer"
        }
      }
    }
  },
  "order": 1,
  "version": 60001,
  "settings": {
    "index": {
      "number_of_shards": "1",
      "number_of_replicas": "0",
      "refresh_interval": "5s"
    }
  }
}
```
#### 编辑/usr/local/logstash/config/app-template.json
``` bash
bash# vim /usr/local/logstash/config/app-template.json
{
    "order": 1,
    "version": 60002,
    "index_patterns": [
      "logstash-*"
    ],
    "settings": {
      "index": {
        "refresh_interval": "5s",
        "number_of_shards": 1,
        "number_of_replicas": 0
      }
    },
    "mappings": {
      "_default_": {
        "dynamic_templates": [
          {
            "message_field": {
              "path_match": "message",
              "match_mapping_type": "string",
              "mapping": {
                "type": "text",
                "norms": false
              }
            }
          },
          {
            "string_fields": {
              "match": "*",
              "match_mapping_type": "string",
              "mapping": {
                "type": "text",
                "norms": false,
                "fields": {
                  "keyword": {
                    "type": "keyword",
                    "ignore_above": 256
                  }
                }
              }
            }
          }
        ],
        "properties": {
          "@timestamp": {
            "type": "date"
          },
          "@version": {
            "type": "keyword"
          },
          "geoip": {
            "dynamic": true,
            "properties": {
              "ip": {
                "type": "ip"
              },
              "location": {
                "type": "geo_point"
              },
              "latitude": {
                "type": "half_float"
              },
              "longitude": {
                "type": "half_float"
              }
            }
          }
        }
      }
    },
    "aliases": {}
}
```
#### 编辑nginx日志的匹配模版文件
``` bash
bash# mkdir /usr/local/logstash/patterns
bash# vi /usr/local/logstash/patterns/nginx
URIPARM1 [A-Za-z0-9$.+!*'|(){},~@#%&/=:;^\\_<>`?\-\[\]]*

URIPATH1 (?:/[\\A-Za-z0-9$.+!*'(){},~:;=@#% \[\]_<>^\-&?]*)+

HOSTNAME1 \b(?:[0-9A-Za-z_\-][0-9A-Za-z-_\-]{0,62})(?:\.(?:[0-9A-Za-z_\-][0-9A-Za-z-:\-_]{0,62}))*(\.?|\b)

STATUS ([0-9.]{0,3}[, ]{0,2})+

HOSTPORT1 (%{IP}:%{POSINT}[, ]{0,2})+

FORWORD (?:%{IP}[,]?[ ]?)+|%{WORD}

URIPARM [A-Za-z0-9$.+!*'|(){},~@#%&/=:;_?\-\[\]]*

URIPATH (?:/[A-Za-z0-9$.+!*'(){},~:;=@#%&_\- ]*)+

URI1 (%{URIPROTO}://)?(?:%{USER}(?::[^@]*)?@)?(?:%{URIHOST})?(?:%{URIPATHPARAM})?

NGINXACCESS %{IPORHOST:remote_addr} - (%{USERNAME:user}|-) \[%{HTTPDATE:log_timestamp}\] \"%{WORD:request_method} %{URIPATH1:uri} HTTP/%{NUMBER:http_version}\" %{BASE10NUM:http_status} (?:%{BASE10NUM:body_bytes_sent}|-) \"(?:%{GREEDYDATA:http_referrer}|-)\" \"(%{GREEDYDATA:user_agent}|-)\" \"(%{FORWORD:x_forword_for}|-)\" \"(%{FORWORD:x_real_ip}|-)\" \"(%{IPORHOST:domain}|%{URIHOST:domain}|-)\" \"(%{IPORHOST:http_x_host}|%{URIHOST:http_x_host}|-)\" \"(%{BASE16FLOAT:request_time}|-)\" \"(?:%{BASE10NUM:request_length}|-)\" \"(?:%{HOSTPORT1:upstream_addr}|-)\" \"(%{BASE16FLOAT:upstream_response_time}|-)\" \"(%{BASE10NUM:upstream_status}|-)\"
```
#### nginx的日志格式为
``` bash
log_format main	'$remote_addr - $remote_user [$time_local] "$request" '
		'$status $body_bytes_sent "$http_referer" '
		'"$http_user_agent" "$http_x_forwarded_for" "$http_x_real_ip" "$host" "$http_x_host" "$request_time" "$request_length" '
		'"$upstream_addr" "$upstream_response_time" "$upstream_status"';
```
#### 使用supervisor进行管理logstash进程
``` bash
bash# vim /etc/supervisor.d/logstash.ini
[program:logstash]
directory=/usr/local/logstash
command=/usr/local/logstash/bin/logstash -f /usr/local/logstash/config/client-redis.conf
environment=JAVA_HOME=/usr/local/jdk/
user=onecafe
autostart=true
autorestart=true
startretries=1
startsecs=1
redirect_stderr=true
stdout_logfile=/data/logs/logstash/logstash.log
stdout_logfile_maxbytes=5MB
stdout_logfile_backups=5
stderr_logfile=/data/logs/logstash/logstash-err.log
stderr_logfile_maxbytes=5MB
stderr_logfile_backups=5
stopasgroup=true
killasgroup=true
```
### 三、启动logstash和filebeat
``` bash
bash# systemctl start filebeat
bash# supervisorctl start logstash
```
#### 到这边，整个elk就算搭建好，下一节继续讲解kibana的使用。
