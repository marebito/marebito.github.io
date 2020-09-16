---
title: 服务器下配置springboot项目开机自启
date: 2020-09-16 10:01:49
tags:
---

> Linux版本 Centos7详细步骤如下：

#### 1.在/etc/init.d/目录下创建shell启动脚本autojar.sh

1. cd /etc/init.d/

2. touch autojar.sh

3. vi autojar.sh

   内容如下：

```shell
#!/bin/sh
# chkconfig: 2345 85 15
# description:auto_run
  
#程序名
RUN_NAME="SpringBoot.jar"
  
#jar包位置
JAVA_OPTS=/software/SpringBoot.jar
#后台运行日志位置
LOG_OPTS=/software/nohup.out
​
#开始方法
start() {
        nohup java -jar $JAVA_OPTS >$LOG_OPTS 2>&1 &
        echo "$RUN_NAME started success."
}
  
#结束方法
stop() {
        echo "stopping $RUN_NAME ..."
        kill -9 `ps -ef|grep $JAVA_OPTS|grep -v grep|grep -v stop|awk '{print $2}'`
}
  
case "$1" in
        start)
            start
            ;;
        stop)
            stop
            ;;
        restart)
            stop
            start
            ;;
        *)
                echo "Userage: $0 {start|stop|restart}"
                exit 1
esac
```

第一行，告诉系统使用的shell,所以的shell脚本都是这样。

第二行，2345代表在设置在那个level中是on的，如果一个都不想on，那就写一个横线"-"，85和15， 后面两个数字代表S和K的默认排序号 ，告诉chkconfig程序，需要在rc2.d~rc5.d目录下，创建名字为S80autojar的文件连接， 第一个字符是S，系统在启动的时候，运行脚本autojar

注意上面的三行中，第二，第三行是必须的，否则在运行chkconfig –add auto_run时，会报错。

85 数字越小 启动优先级别越高

15数字越小 关闭优先级别越高



#### 2.设置执行权限

```
  chmod +x /etc/init.d/autojar.sh
  chmod +x /software/SpringBoot.jar
```

#### 3.添加到chkconfig作为系统服务，并设置开机启动：

系统在启动的时候，就会运行autojar并加上start参数，等同于执行命令autojar start。

```
  chkconfig --add autojar.sh   (添加为系统服务)
  chkconfig autojar.sh on  （开机自启动）
  service autojar.sh start（启动服务）
```

#### 4.重启

```
  reboot
```

#### 5.查看

```
  netstat -ntlp | grep 8082 （查看端口）
  ps aux|grep java（查看服务）
```

#### 说明:

chkconfig提供一种简单的命令行工具来帮助管理员对/etc/rc[0-6].d目录层次下的众多的符号链接进行直接操作。 此命令使用是由chkconfig命令在IRIX操作系统提供授权。不用在/etc/rc[0-6].d目录下直接维护配置信息，而是直接在/etc/rc[0-6]下管理链接文件。在运行级别的目录下的配置信息通知在将会初始启动哪些服务。

**Chkconfig有五个很明确的功能**：为管理增加一个新的功能、删除一个功能、列出当前服务的启动信息、改变一个服务的启动信息和检测特殊服务的启动状态。

当chkconfig没有参数运行时，它将显示其使用方法。如果只给出了一个服务名，它将检测这个服务名是否已经被配置到了当前运行级别中。如果已经配置，返回真，否则返回假。--level选项可以被用做查询多个运行级别而不仅仅是一个。

如果在服务名之后指定了on,、off或reset，chkconfig将改变指定服务的启动信息。On或off标记服务被打开或停止，尤其是在运行级别被改变时。Reset标记重置服务的启动信息。

默认情况下，on或off选项仅对2、3、4、5有影响，而 reset影响所有的运行级。--level选项可以被用于指定哪个运行级别接收影响。

注意：对于每个服务，每一个运行级都有一个开始角本和一个结束角本。当开或关一个运行级时，init不会重新开始一个已经运行的服务，也不会重新停止一个未运行的服务。

选项：

**--level levels** 指定一个运行级别适合的操作。范围为0-7。

**--add name** 增加一个新的服务。

**--del name** 删除一个服务

**--list name** 显示服务的情况

**RUNLEVEL FILES**

每个通过chkconfig管理的服务在其init.d目录下的角本中都需要两行或多行的注释。

第一行告诉chkconfig在默认情况下什么运行级别的服务可以开始，也就是所说的开始或结束的优先级别。如果服务没有默认的级别，建造将在所有运行级别中启动。a – 将用于代替运行级列表。第二个注释行包括对此服务的描述，可以通过反斜线符号扩展为多行。

示例，

auto_run的前三行如下：

\#!/bin/sh

\#chkconfig: 2345 80 90

\#description:auto_run

第一行，告诉系统使用的shell,所以的shell脚本都是这样。

第二行，chkconfig后面有三个参数2345,80和90告诉chkconfig程序，需要在rc2.d~rc5.d目录下，创建名字为 S80auto_run的文件连接，连接到/etc/rc.d/init.d目录下的的auto_run脚本。第一个字符是S，系统在启动的时候，运行脚本auto_run，就会添加一个start参数，告诉脚本，现在是启动模式。同时在rc0.d和rc6.d目录下，创建名字为K90auto_run的文件连接，第一个字符为K，系统在关闭系统的时候，会运行auto_run，添加一个stop，告诉脚本，现在是关闭模式。 注意上面的三行中，第二，第三行是必须的，否则在运行chkconfig --add auto_run时，会报错。

>  常见的错误 “服务不支持 chkconfig”： 请注意检查脚本的前面，是否有完整的两行：

\#chkconfig: 2345 80 90

\#description:auto_run

在脚本前面这两行是不能少的，否则不能chkconfig命令会报错误。 如果运行chkconfig老是报错，如果脚本没有问题，我建议，直接在rc0.d~rc6.d下面创建到脚本的文件连接来解决，原理都是一样的。