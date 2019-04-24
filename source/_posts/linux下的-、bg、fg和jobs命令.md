---
title: linux下的&、bg、fg和jobs命令
date: 2019-04-12 14:45:10
tags:
categories: linux
---
#### 我们在linux下有时会需要把服务放到后台去运行，下面就将讲解下。
### 一、使用nohup和&
#### 假设现在我们有一个服务xxxx.jar，需要放到后台运行，我们可以这么操作：

``` bash
bash# nohup java -jar xxxx.jar &
#运行上面这条命令就可以将xxxx.jar这个服务放到后台去运行了。
```

### 二、使用bg
#### 如果我们忘记使用nohup和&,怎么办呢？莫慌，还可以这么操作：

``` bash
#使用Ctrl + z,将服务暂停。
bash# bg
#然后执行bg命令就将服务放到后台运行了。
```

### 三、查看现在在后台运行的进程

``` bash
bash# jobs -l
#运行jobs -l将列出在后台的进程
bash# fg %num
#运行fg %num将进程移到前台来，注意不是pid。
```
