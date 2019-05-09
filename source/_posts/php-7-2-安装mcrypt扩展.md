---
title: php 7.2 安装mcrypt扩展
date: 2019-05-09 09:26:27
tags:
categories: linux
---
微信的接口需要使用到mcrypt加解密，但是php 7.1开始已经移除mcrypt扩展，需要自行安装。
### 1.环境
``` bash
系统：centos 7
PHP: 7.2.10
```

### 2.安装依赖包
``` bash
yum install libmcrypt libmcrypt-devel mcrypt mhash
```

### 3.安装mcrypt扩展
``` bash
wget  http://pecl.php.net/get/mcrypt-1.0.2.tgz
tar xf mcrypt-1.0.2.tgz
cd mcrypt-1.0.2
/usr/local/web/php/bin/phpize
./configure --with-php-config=/usr/local/web/php/bin/php-config  && make && make install
vim /usr/local/web/php/etc/php.ini
```
``` bash
......
......
......
#在最后面加上扩展
extension=mcrypt.so
```

### 4.重载php-fpm配置
``` bash
kill -USR2 `cat /var/run/php.pid` #php-fpm可以通过信号重载配置
```
