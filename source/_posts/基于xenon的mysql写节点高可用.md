---
title: 基于xenon的mysql写节点高可用
date: 2024-08-25 11:49:26
tags: 
categories: mysql
keywords: 高可用 mysql
description:
---
# 基于xenon的mysql 写节点高可用

# **mysql 部署**
## 我这边使用rocky9系统，系统源自带mysql8.0，我这边直接使用，如果使用的是其他系统，请自行安装mysql8.0或者mysql5.7。

## 安装mysql8.0
``` bash
yum install mysql-server -y
```

## 编辑配置文件
``` bash
cat > /etc/my.cnf < EOF
[mysqld]
# 基础设置
user            = mysql
bind-address    = 0.0.0.0
port            = 3306
server-id       = 3
default_authentication_plugin=mysql_native_password

# 数据库文件和日志
datadir         = /var/lib/mysql
log_error       = /var/log/mysql/error.log
pid-file        = /var/run/mysqld/mysqld.pid
socket          = /var/run/mysqld/mysqld.sock

# 性能优化
innodb_buffer_pool_size = 1G
innodb_log_file_size     = 256M
innodb_flush_log_at_trx_commit = 1
innodb_flush_method = O_DIRECT

# 连接设置
max_connections = 200
wait_timeout = 600
interactive_timeout = 600

# 日志设置
general_log       = OFF
general_log_file  = /var/log/mysql/mysql.log
slow_query_log     = ON
slow_query_log_file = /var/log/mysql/mysql-slow.log
long_query_time    = 2

# 其他设置
character_set_server = utf8mb4
collation_server     = utf8mb4_general_ci
default_time_zone    = '+00:00'
#super_read_only = 0

#半同步复制
gtid_mode = ON
enforce_gtid_consistency = ON
EOF
```
## 注意事项：server-id 需要不同。

## 启动mysql
``` bash
systemctl start mysqld
```

## 192.168.1.100执行sql
``` bash
mysql -h127.0.0.1
create user rep@'%' identified with mysql_native_password by 'rep';
grant replication slave on *.* to rep@'%';
create user syin@'%' identified by 'abcd1234';
grant all privileges on *.* to syin@'%';

INSTALL PLUGIN rpl_semi_sync_master SONAME 'semisync_master.so';
INSTALL PLUGIN rpl_semi_sync_slave SONAME 'semisync_slave.so';
set global rpl_semi_sync_master_enabled=1;
set global rpl_semi_sync_master_timeout=2147483648;
set global rpl_semi_sync_slave_enabled=1;
```

## 192.168.1.101 和 192.168.1.102执行sql
``` bash
mysql -h127.0.0.1
INSTALL PLUGIN rpl_semi_sync_master SONAME 'semisync_master.so';
INSTALL PLUGIN rpl_semi_sync_slave SONAME 'semisync_slave.so';
set global rpl_semi_sync_master_enabled=1;
set global rpl_semi_sync_master_timeout=2147483648;
set global rpl_semi_sync_slave_enabled=1;
change master to master_host='192.168.1.100',master_user='rep',master_password='rep',master_port=3306,master_auto_position=1;
start slave;
show slave status\G;
```

# **Xenon是什么**

Xenon [ˈziːnɒn] ([https://github.com/radondb/xenon](https://github.com/radondb/xenon)) 是一款由 RadonDB 开发团队研发并开源的新一代 MySQL 集群高可用工具。基于 Raft 协议进行无中心化选主，实现主从秒级切换；基于 Semi-Sync 机制，保障数据不丢失，实现数据强一致性。

![20210826115756383.png](/images/mysql/20210826115756383.png)

# **Xenon 的优势**

相比 MHA，Xenon 的优势如下：

- 多版本内核支持

支持 MySQL 5.6、5.7、8.0 内核版本。

- 多平台支持

   支持物理机、虚拟机/云平台、容器/ Kubernetes 平台部署。

- 稳定性更好

   MySQL 新版本特性兼容。

- 性能更佳

   与 GTID、MTS（并行复制） 结合，并行日志复制、并行日志回放。

- 架构更简单

   不需要管理节点，机器成本更低。

- 数据更安全

   增强半同步复制不会降级为异步，保证数据零丢失，不会存在 MHA 在 GTID 模式下丢数据的风险。

- 故障修复全自动

   Xenon 对于故障节点会自动先自我修复。

- 节点恢复快

   配合 Xtrabackup 等可以实现快速恢复。

- 操作更简单，维护成本更低

# **Xenon 部署**

- ## 注意事项，官方说使用go1.8以上进行编译，我使用了最新1.23编译失败，我降级到1.14.15编译成功。

- ## 安装Xenon软件
``` bash
git clone https://github.com/radondb/xenon.git
cd xenon
make build
mkdir /usr/local/xenon -p
cp bin/xenon bin/xenoncli /usr/local/xenon
```

- ## 配置变量文件
``` bash
echo "/usr/local/xenon/xenon.conf.json" >/usr/local/xenon/config.path
```

- ## 修改配置参数
``` bash
cat > /usr/local/xenon/xenon.conf.json <<  EOF

{

"server":{

"endpoint":"192.168.1.101:8801"

},

"raft":{

"meta-datadir":"/usr/local/xenon/raft.meta",

"heartbeat-timeout":1000,

"election-timeout":3000,

"admit-defeat-hearbeat-count": 5,

"purge-binlog-disabled": true,

"leader-start-command":"sudo /sbin/ip a a 192.168.1.105/24 dev ens192 && arping -c 3 -A 192.168.1.105 -I ens192",

"leader-stop-command":"sudo /sbin/ip a d 192.168.1.105/24 dev ens192"

},

"mysql":{

"admin":"root",

"passwd":"",

"host":"127.0.0.1",

"port":3306,

"basedir":"/usr",

"defaults-file":"/etc/my.cnf",

"ping-timeout":1000

},

"replication":{

"user":"rep",

"passwd":"rep"

},

"rpc":{

"request-timeout":500

},

"log":{

"level":"ERROR"

}

}

EOF
```
- ## 配置后台服务

# 配置后台服务
``` bash
cat > /usr/lib/systemd/system/xenon.service <<  EOF

[Unit]

Description=xenon service

After=network.target

[Service]

Type=simple

User=root

Group=root

ExecStart=/usr/local/xenon/xenon -c /usr/local/xenon/xenon.conf.json

Restart=on-failure

[Install]

WantedBy=multi-user.target

EOF
```

# 启动 xenon 后台服务
``` bash
systemctl daemon-reload

systemctl enable xenon

systemctl start  xenon

systemctl status xenon
```

- ## 添加节点

注: 每个节点都要执行
``` bash
/usr/local/xenon/xenoncli cluster add 192.168.1.100:8801,192.168.1.101:8801,192.168.1.102:8801
```

- ## 查看状态
``` bash
/usr/local/xenon/xenoncli cluster status

+---------------------+-------------------------------+------------+---------+--------------------------+---------------------+----------------+---------------------+

|         ID          |             Raft              |   Mysqld   | Monitor |          Backup          |        Mysql        | IO/SQL_RUNNING |      MyLeader       |`

+---------------------+-------------------------------+------------+--------+--------------------------+---------------------+----------------+---------------------+

| 192.168.1.101:8801 | [ViewID:3 EpochID:0]@FOLLOWER | NOTRUNNING | ON      | state:[NONE]?            | [ALIVE] [READONLY]  | [true/true]    | 192.168.1.100:8801 |

|                     |                               |            |         | LastError:               |                     |                |                     |

+---------------------+-------------------------------+------------+---------+--------------------------+---------------------+----------------+---------------------+

| 192.168.1.100:8801 | [ViewID:3 EpochID:0]@LEADER   | NOTRUNNING | ON      | state:[NONE]?            | [ALIVE] [READWRITE] | [true/true]    | 192.168.1.100:8801 |

|                     |                               |            |         | LastError:               |                     |                |                     |

+---------------------+-------------------------------+------------+---------+--------------------------+---------------------+----------------+---------------------+

| 192.168.1.102:8801 | [ViewID:3 EpochID:0]@FOLLOWER | NOTRUNNING | ON      | state:[NONE]?            | [ALIVE] [READONLY]  | [true/true]    | 192.168.1.100:8801 |

|                     |                               |            |         | LastError:               |                     |                |                     |

+---------------------+-------------------------------+------------+---------+--------------------------+---------------------+----------------+---------------------+
```

