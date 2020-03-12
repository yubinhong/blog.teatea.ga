---
title: 解决iotop不能使用的问题
date: 2019-07-10 14:35:49
tags:
categories: python
keywords: linux iotop python
description:
---
安装iotop工具，想要查看磁盘读写情况，发现运行报错，报错如下：
``` bash
Traceback (most recent call last):
  File "/sbin/iotop", line 17, in <module>
    main()
  File "/usr/lib/python2.7/site-packages/iotop/ui.py", line 620, in main
    main_loop()
  File "/usr/lib/python2.7/site-packages/iotop/ui.py", line 610, in <lambda>
    main_loop = lambda: run_iotop(options)
  File "/usr/lib/python2.7/site-packages/iotop/ui.py", line 508, in run_iotop
    return curses.wrapper(run_iotop_window, options)
  File "/usr/lib64/python2.7/curses/wrapper.py", line 43, in wrapper
    return func(stdscr, *args, **kwds)
  File "/usr/lib/python2.7/site-packages/iotop/ui.py", line 501, in run_iotop_window
    ui.run()
  File "/usr/lib/python2.7/site-packages/iotop/ui.py", line 155, in run
    self.process_list.duration)
  File "/usr/lib/python2.7/site-packages/iotop/ui.py", line 434, in refresh_display
    lines = self.get_data()
  File "/usr/lib/python2.7/site-packages/iotop/ui.py", line 415, in get_data
    return list(map(format, processes))
  File "/usr/lib/python2.7/site-packages/iotop/ui.py", line 388, in format
    cmdline = p.get_cmdline()
  File "/usr/lib/python2.7/site-packages/iotop/data.py", line 292, in get_cmdline
    proc_status = parse_proc_pid_status(self.pid)
  File "/usr/lib/python2.7/site-packages/iotop/data.py", line 196, in parse_proc_pid_status
    key, value = line.split(':\t', 1)
ValueError: need more than 1 value to unpack
```
google了一下，解决方法如下：
vi /usr/lib/python2.7/site-packages/iotop/data.py
跳到196行，修改如下内容：
原代码
``` bash
def parse_proc_pid_status(pid):
    result_dict = {}
    try:
        for line in open('/proc/%d/status' % pid):
            key, value = line.split(':\t', 1)
            result_dict[key] = value.strip()
    except IOError:
        pass  # No such process
    return result_dict
```
修改为：
``` bash
def parse_proc_pid_status(pid):
    result_dict = {}
    try:
        for line in open('/proc/%d/status' % pid):
            try:
                key, value = line.split(':\t', 1)
            except:
                break
            result_dict[key] = value.strip()
    except IOError:
        pass  # No such process
    return result_dict
```
保存退出，运行iotop正常。
