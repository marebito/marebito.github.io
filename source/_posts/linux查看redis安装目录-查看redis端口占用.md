---
title: linux查看redis安装目录_查看redis端口占用
date: 2020-09-14 11:31:18
tags:
---

如果命令 which 和whereis 都找不到安装目录，可使用以下办法

  ps -ef|grep redis

得到了进程号 xxxx

 然后 ls -l /proc/xxxx/cwd



![image-20200914113316172](/Users/boyka/Library/Application Support/typora-user-images/image-20200914113316172.png)



查看端口占用

netstat -tunpl|grep 6379

如果：-bash: netstat: command not found

yum install net-tools