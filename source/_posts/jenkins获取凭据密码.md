---
title: jenkins获取凭据密码
date: 2019-03-27 15:59:25
tags:
categories: jenkins
---
### 本人忘记jenkins上的gitlab CI/CD的账号密码，经过google一番搜索，可以使用以下方式找回密码，
在此记录，以防忘记。
``` bash
1.登录jenkins,进入凭据页面
2.点击需要找回密码的账号，进入编辑页面
3.打开浏览器的开发者模式，将密码框的type=password去掉，这时密码框会显示加密过的密码，将这串密码记录下来。
4.打开http://{jenkins_url}/script,进入script console界面，在输入框输入println( hudson.util.Secret.decrypt("$password") )，其中$password 就是记录下来的密码，点击Run。密码已经出来了。
```
