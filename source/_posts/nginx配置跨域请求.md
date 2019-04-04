---
title: nginx配置跨域请求
date: 2019-03-29 13:53:56
tags:
categories: nginx
---
当出现403跨域错误的时候 No 'Access-Control-Allow-Origin' header is present on the requested resource，需要给Nginx服务器配置响应的header参数：
## 一、解决方案
### 只需要在Nginx的配置文件中配置以下参数：
``` bash
server {
    .....
    .....
    if ($http_origin ~* "^http://192.168.1.39:8080$") {
           set $cors_origin $http_origin;
    }
       add_header 'Access-Control-Allow-Origin' $cors_origin always;
       add_header 'Access-Control-Allow-Credentials' 'true' always;
       add_header 'Access-Control-Allow-Headers' 'Accept, Authorization,DNT,X-CustomHeader,Keep-Alive,User-Agent,X-Requested-With,If-Modified-Since,Cache-Control,Content-Type';
       add_header Access-Control-Allow-Methods 'GET, POST, OPTIONS';

    location / {
       if ($request_method = 'OPTIONS') {
	return 204;
       }
       try_files $uri $uri/ /index.php$is_args$args;
    }
    ....
}
```
## 二、解释
### 1. Access-Control-Allow-Origin
``` bash
服务器默认是不被允许跨域的。给Nginx服务器配置`Access-Control-Allow-Origin *`后，表示服务器可以接受所有的请求源（Origin）,即接受所有跨域的请求。
```

### 2. Access-Control-Allow-Headers 是为了防止出现以下错误：
``` bash
Request header field Content-Type is not allowed by Access-Control-Allow-Headers in preflight response.

这个错误表示当前请求Content-Type的值不被支持。其实是我们发起了"application/json"的类型请求导致的。这里涉及到一个概念：预检请求（preflight request）,请看下面"预检请求"的介绍。
```

### 3. Access-Control-Allow-Methods 是为了防止出现以下错误：
``` bash
Content-Type is not allowed by Access-Control-Allow-Headers in preflight response.
```

### 4. Access-Control-Allow-Credentials
``` bash
Access-Control-Allow-Credentials 头指定了当浏览器的credentials设置为true时是否允许浏览器读取response的内容。当用在对preflight预检测请求的响应中时，它指定了实际的请求是否可以使用credentials。请注意：简单 GET 请求不会被预检；如果对此类请求的响应中不包含该字段，这个响应将被忽略掉，并且浏览器也不会将相应内容返回给网页。
```

### 5.给OPTIONS 添加 204的返回，是为了处理在发送POST请求时Nginx依然拒绝访问的错误
``` bash
发送"预检请求"时，需要用到方法 OPTIONS ,所以服务器需要允许该方法。
```

## 三、 预检请求（preflight request）
### 其实上面的配置涉及到了一个W3C标准：CROS,全称是跨域资源共享 (Cross-origin resource sharing)，它的提出就是为了解决跨域请求的。
``` bash
跨域资源共享(CORS)标准新增了一组 HTTP 首部字段，允许服务器声明哪些源站有权限访问哪些资源。另外，规范要求，对那些可能对服务器数据产生副作用的HTTP 请求方法（特别是 GET 以外的 HTTP 请求，或者搭配某些 MIME 类型的 POST 请求），浏览器必须首先使用 OPTIONS 方法发起一个预检请求（preflight request），从而获知服务端是否允许该跨域请求。服务器确认允许之后，才发起实际的 HTTP 请求。在预检请求的返回中，服务器端也可以通知客户端，是否需要携带身份凭证（包括 Cookies 和 HTTP 认证相关数据）。
```

### 其实Content-Type字段的类型为application/json的请求就是上面所说的搭配某些 MIME 类型的 POST 请求,CORS规定，Content-Type不属于以下MIME类型的，都属于预检请求：
``` bash
application/x-www-form-urlencoded
multipart/form-data
text/plain
```
### 所以 application/json的请求 会在正式通信之前，增加一次"预检"请求，这次"预检"请求会带上头部信息 Access-Control-Request-Headers: Content-Type：
``` bash
OPTIONS /api/test HTTP/1.1
Origin: http://foo.example
Access-Control-Request-Method: POST
Access-Control-Request-Headers: Content-Type
... 省略了一些
```
### 服务器回应时，返回的头部信息如果不包含Access-Control-Allow-Headers: Content-Type则表示不接受非默认的的Content-Type。即出现以下错误：
``` bash
Request header field Content-Type is not allowed by Access-Control-Allow-Headers in preflight response.
```
