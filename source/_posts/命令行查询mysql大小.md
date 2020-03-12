---
title: 命令行查询mysql大小
date: 2020-01-16 10:52:38
tags: 
categories: mysql
keywords: mysql
description:
---
### 我们的数据库空间不足，需要找出哪些库和哪些表占用比较大的空间。
## 具体步骤：
### 1.查看该数据库实例下所有库大小，得到的结果是以MB为单位（除1024为KB，再除1024为MB），下同
``` bash
> use information_schema
> select table_schema,sum(data_length)/1024/1024 as data_length,sum(index_length)/1024/1024 as index_length,sum(data_length+index_length)/1024/1024 as sum from tables;
+--------------------+--------------+--------------+--------------+
| table_schema       | data_length  | index_length | sum          |
+--------------------+--------------+--------------+--------------+
| information_schema | 554.23122311 | 163.24804688 | 717.47926998 |
+--------------------+--------------+--------------+--------------+
1 row in set (0.32 sec)
```
### 2.查看该数据库实例下各个库大小
``` bash
> use information_schema
> select table_schema,sum(data_length)/1024/1024 as data_length,sum(index_length)/1024/1024 as index_length,sum(data_length+index_length)/1024/1024 as sum from tables group by table_schema;
+--------------------+--------------+--------------+--------------+
| table_schema       | data_length  | index_length | sum          |
+--------------------+--------------+--------------+--------------+
| coolnull1          |   0.49992847 |   0.54492188 |   1.04485035 |
| coolnull           |  13.79830647 |   0.74121094 |  14.53951740 |
| coolnull2          |   0.00114059 |   0.05078125 |   0.05192184 |
| coolnull3          | 101.22271824 |   1.88183594 | 103.10455418 |
| coolnull4          |  14.25625229 |   2.78710938 |  17.04336166 |
| information_schema |   0.00000000 |   0.00781250 |   0.00781250 |
| mysql              |   0.51842022 |   0.08691406 |   0.60533428 |
| coolnull5          |   0.79851532 |   0.05175781 |   0.85027313 |
| coolnull6          |  16.85469151 |   1.04882813 |  17.90351963 |
| zabbix             | 404.59375000 | 153.34375000 | 557.93750000 |
| zabbix_proxy       |   1.68750000 |   2.70312500 |   4.39062500 |
+--------------------+--------------+--------------+--------------+
11 rows in set (0.44 sec)
```
### 3.查看coolnull库各表大小
``` bash
> use information_schema
> select table_name,data_length/1024/1024 as data_length,index_length/1024/1024 as index_length,(data_length+index_length)/1024/1024 as sum from tables where table_schema='coolnull';
+-----------------------+-------------+--------------+------------+
| table_name            | data_length | index_length | sum        |
+-----------------------+-------------+--------------+------------+
| wp_commentmeta        |  5.99062729 |   0.20312500 | 6.19375229 |
| wp_comments           |  2.07605743 |   0.07031250 | 2.14636993 |
| wp_links              |  0.00066376 |   0.00292969 | 0.00359344 |
| wp_options            |  0.60661697 |   0.01953125 | 0.62614822 |
| wp_postmeta           |  0.18944931 |   0.08496094 | 0.27441025 |
| wp_posts              |  4.82567596 |   0.16503906 | 4.99071503 |
| wp_term_relationships |  0.03670979 |   0.08691406 | 0.12362385 |
| wp_term_taxonomy      |  0.03935623 |   0.03417969 | 0.07353592 |
| wp_terms              |  0.03020859 |   0.06054688 | 0.09075546 |
| wp_usermeta           |  0.00271606 |   0.00976563 | 0.01248169 |
| wp_users              |  0.00022507 |   0.00390625 | 0.00413132 |
+-----------------------+-------------+--------------+------------+
11 rows in set (0.00 sec)
```
引用 [http://coolnull.com/3934.html](http://coolnull.com/3934.html) 结果。
