---
title: nginx反向代理websocket
date: 2019-07-05 10:20:37
tags:
categories: nginx
keywords: nginx websocket
description: 
---
### 1.原理
一般我们开发的WebSocket服务程序使用ws协议，明文的。但是怎样让它安全的通过互联网传输呢？这时候可以通过nginx在客户端和服务端直接做一个转发了， 客户端通过wss访问，然后nginx和服务端通过ws协议通信。
### 2.配置
当前服务器放在内网，所以直接用http协议演示。
新建nginx配置文件/etc/nginx/conf.d/websocket.conf，内容如下:
``` bash
upstream websocket {
    server 127.0.0.1:9999; # appserver_ip:ws_port
}

server {
     listen 80;
     location = /ws/ {
         proxy_pass http://websocket;
         
         proxy_set_header Host $host;
         proxy_set_header X-Real-IP $remote_addr;
         proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
         
         proxy_http_version 1.1;
         proxy_set_header Upgrade $http_upgrade;
         proxy_set_header Connection "upgrade";
     }
}
```
解释一下关键配置：
最重要的就是在反向代理的配置中增加了如下两行，其它的部分和普通的HTTP反向代理没有任何差别。
``` bash
proxy_set_header Upgrade $http_upgrade;
proxy_set_header Connection "upgrade";
```
这里面的关键部分在于HTTP的请求中多了如下头部：
这两个字段表示请求服务器升级协议为WebSocket。服务器处理完请求后，响应如下报文：
``` bash
# 状态码为101
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: upgrade
```
告诉客户端已成功切换协议，升级为Websocket协议。握手成功之后，服务器端和客户端便角色对等，就像普通的Socket一样，能够双向通信。 不再进行HTTP的交互，而是开始WebSocket的数据帧协议实现数据交换。
