---
title: windows登录自动化 PPTP
date: 2019-08-15 02:35:47
tags:
---
#### 前情提要

公司网络管制严重，之前就很多正常网站不能访问，最近更是直接屏蔽了外网的ssh，导致我自己的服务器不能登录操作。无奈只能一直使用 PPTP 连回家里网络上网。这本来没什么问题。但是测试服务器和很多公司服务都是公司的内网网段，连上 PPTP 这些地址都不能访问了，需要自己手动添加路由表设置。这本来也也没什么问题。但是每次电脑睡眠回来，都要重复连接 PPTP、执行路由表添加命令，真是不胜其烦。秉着程序员就要自动化的原则，就下决心要处理一下这个事情。



#### 思路

自动化的本质其实就是脚本化。经过一番搜索我找到如下相关信息。

1. windows下连接系统设置的VPN的命令是 `rasdial`,基本格式是

```
rasdial <VPN NAME> <USERNAME> <PASSWORD>
```

0. `route ADD`命令添加路由表，对于我公司`10.*.*.*`网段简化格式如下

```
route ADD 10.0.0.0 MASK 255.0.0.0 <GATEWAY>
```

0. windows下定时执行工具叫 Task Scheduler（任务计划）,是个 GUI 程序，并且可以设置触发器不用循环执行。

所以整体思路就是创建一个脚本检查当前网络环境，是否是在公司内部，如果是在公司内部并且没有 PPTP 连接就执行`rasdial`和`route`命令。

#### 脚本

```
ipconfig | findstr /n 10\.4\.90\.[0-9]*
IF NOT ERRORLEVEL 1 ( 
    ipconfig | findstr /n 192\.168\.123\.[0-9]*
    IF ERRORLEVEL 1 ( 
        echo AT COMPANY AND NO PPTP.
        rasdial <VPN NAME> <USERNAME> <PASSWORD>
        route ADD 10.0.0.0 MASK 255.0.0.0 10.4.90.1
    )
)

```

#### 设置任务计划

任务计划设置直接GUI操作没什么好说的，就注意两点

1. 常规选项卡上勾选以最高权限执行
2. 触发器添加在工作站解锁时运行

完美。



![](http://www.jswh.me/wp-content/uploads/2019/08/8.jpg)