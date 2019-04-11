---
title: ELK 使用系列 三、kibana创建visualize
date: 2019-04-10 17:48:45
tags:
categories: ELK
---
#### 前两节已经搭建完ELK,那怎么从这些nginx日志中提取出对我们有用的信息呢。本节就来讲解一下：(以统计http状态码为例)
### 一、登录kibana页面，点击Visualize，点击+号创建
![](https://res.cloudinary.com/dc6pgic7p/image/upload/v1554890075/http_status_create.png)
### 二、点击饼图pie
![](https://res.cloudinary.com/dc6pgic7p/image/upload/v1554889609/https_status_pie.png)
### 三、按照如图所示配置
![](https://res.cloudinary.com/dc6pgic7p/image/upload/v1554889623/http_status.png)
### 四、点击三角标按钮，就可以出图了。
### 五、以此为例，依次创建uri请求平均时间，ip访问次数统计等等图表。
### 六、将这些图表添加到dashboard，效果如下图所示。
![](https://res.cloudinary.com/dc6pgic7p/image/upload/v1554890743/kibana_dashboard.png)
