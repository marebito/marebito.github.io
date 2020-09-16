---
title: CentOS7下利用init.d启动脚本实现tomcat开机自启动
date: 2020-09-16 10:06:01
tags:
---

# 1. 环境准备

## 1.1 系统

操作系统：CentOS 7（64位）

## 1.2 工具/软件

已安装JDK，并配置好环境变量 
已安装tamcat，可手动启动

# 2. 方法/步骤

## 2.1 JDK环境配置

CentOS7默认安装了OpenJDK，用于支持Tomcat启动是没有问题的。如果项目需要使用Sun的JDK特性的话，就需要重新配置Sun的JDK环境。这里可以参照本人之前的博文：《17101501_CentOS7下卸载openJDK安装Sun公司的JDK》。

## 2.2编写tomcat服务脚本文件

在/etc/init.d/目录下创建tomcat8服务脚本文件。 
执行脚本：

```
[root@localhost /]# vim /etc/init.d/tomcat8
[root@localhost /]# cat /etc/init.d/tomcat8
```

将下面内容进行粘贴：

```shell
#!/bin/bash
#
# tomcat startup script for the Tomcat server
#
#
# chkconfig: 345 80 20
# description: start the tomcat deamon
#
# Source function library
. /etc/rc.d/init.d/functions

prog=tomcat8
JAVA_HOME=/usr/java/jdk1.8.0_151/  # 根据自己的路径改写JAVA_HOME
export JAVA_HOME
CATALANA_HOME=/usr/local/tomcat/   # 根据自己的路径改写CATALANA_HOME
export CATALINA_HOME

case "$1" in
start)
    echo "Starting Tomcat..."
    $CATALANA_HOME/bin/startup.sh
    ;;

stop)
    echo "Stopping Tomcat..."
    $CATALANA_HOME/bin/shutdown.sh
    ;;

restart)
    echo "Stopping Tomcat..."
    $CATALANA_HOME/bin/shutdown.sh
    sleep 2
    echo
    echo "Starting Tomcat..."
    $CATALANA_HOME/bin/startup.sh
    ;;

*)
    echo "Usage: $prog {start|stop|restart}"
    ;;
esac
exit 0 
```

保存退出

## 2.3 赋权限，测试启动脚本

执行脚本：

```
[root@localhost /]# cd /etc/init.d/
[root@localhost init.d]# chmod 755 tomcat8    #赋予权限
[root@localhost init.d]# service tomcat8 start  #启动服务
Starting tomcat8 (via systemctl):                          [  确定  ]
[root@localhost init.d]# service tomcat8 stop   #停止服务
Stopping tomcat8 (via systemctl):                          [  确定  ]
[root@localhost init.d]# service tomcat8 restart  #重启服务
Restarting tomcat8 (via systemctl):                        [  确定  ]
```

 

## 2.4 将服务脚本加入到系统启动队列

执行脚本：

```
[root@localhost zm]# chkconfig tomcat8 on  #服务脚本加入到系统启动队列
[root@localhost zm]# chkconfig --list  tomcat8  #检查 oracle服务是否已经生效
注意：该输出结果只显示 SysV 服务，并不包含原生 systemd 服务。SysV 配置数据可能被原生 systemd 配置覆盖。
      如果您想列出 systemd 服务,请执行 'systemctl list-unit-files'。
      欲查看对特定 target 启用的服务请执行
      'systemctl list-dependencies [target]'。
tomcat8         0:关    1:关    2:开    3:开    4:开    5:开    6:关

```

## 2.5 重启系统，测试配置结果

一般情况下，启动是没有问题的。

这里多说点儿，因为CentOS7的自启动服务开始由systemctl逐渐替代了早期版本中的chkconfig和service形式。 
这里我尝试了一下用指令：systemctl start tomcat8启动服务，系统提示systemctl daemon-reload命令加载服务，执行后，发现可以通过systemctl命令进行简单的控制，如查询状态，启动服务，终止服务，重启服务等操作。但是关于开机启动的设置是不可以的，还需要通过老命令chkconfig实现。

 