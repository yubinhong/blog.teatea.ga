---
title: nginx try_files配置详解
date: 2019-03-25 16:14:07
tags:
categories: nginx
---
## Nginx try_files配置
``` bash
location / {
 try_files $uri $uri/ / ;
}
```
#### 解释：在项目目录下匹配$uri这个路径，如果有存在文件，则返回该文件，如果不存在该文件，则匹配目录，还没有的话，直接返回首页。
