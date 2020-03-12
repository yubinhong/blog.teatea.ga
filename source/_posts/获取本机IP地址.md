---
title: 获取本机IP地址
date: 2019-05-30 11:34:03
tags:
categories: linux
keywords: ddns 动态dns 本机IP cloudflare 树莓派
description: 获取本机IP地址
---
我在家里放了一个树莓派，作为服务器使用，经常需要登陆使用，但是家里的网络是动态IP，无奈只能使用ddns（动态dns）的方式。之前使用的是[https://dns.he.net/](https://dns.he.net/)，但是这个经常会被墙。
我现在使用的是python获取本机IP地址，然后通过cloudflare的API更新IP解析，实现ddns的效果。

python脚本：
``` python
#!/usr/bin/python
#date: 2019-05-09
#author: ybh
from lxml import etree
import requests
url = "https://ip.newb.ga"
r = requests.get(url)
ip = r.json()['ip']

print(ip)
cloudflare_api = "https://api.cloudflare.com/client/v4/zones/xxxxxxxxxxxxxxxxxxx/dns_records/xxxxxxxxxxxxxxxxxxxxxxxx"
params = {}
params['type'] = 'A'
params['name'] = 'git'
params['content'] = str(ip)
headers = {"Content-Type": "application/json",
          "X-Auth-Email": "704523059@qq.com",
          "X-Auth-Key": "xxxxxxxxxxxxxxxxxxxxxxxxxxx"}
req = requests.put(cloudflare_api, json=params, headers=headers)
print(req.text)
```

其中使用到的[https://ip.newb.ga](https://ip.newb.ga)是自己写的一个服务，可以看我的[github](https://github.com/yubinhong/ip).
然后设置定时跑脚本，更新IP。
现在就可以愉快的连接树莓派了。
