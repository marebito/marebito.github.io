---
title: docker安装MySQL5.7
date: 2020-09-11 16:05:53
tags: - Docker
	  - MySQL
---

1. 安装mysql5.7 docker镜像

拉取官方mysql5.7镜像 

`docker pull mysql:5.7`

![img](https://img-blog.csdnimg.cn/2019061713323556.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MDQ2MTI4MQ==,size_16,color_FFFFFF,t_70)

查看镜像库

`docker images`

![img](https://img-blog.csdnimg.cn/20190617133338751.png)



2. 创建MySQL容器

在本地创建mysql的映射目录

`mkdir -p /root/mysql/data /root/mysql/logs /root/mysql/conf`



在/root/mysql/conf中创建 *.cnf 文件(叫什么都行)

`touch my.cnf`



创建容器,将数据,日志,配置文件映射到本机

```
docker run -p 3306:3306 --name mysql -v /root/mysql/conf:/etc/mysql/conf.d -v /root/mysql/logs:/logs -v /root/mysql/data:/var/lib/mysql -e MYSQL_ROOT_PASSWORD=root -d mysql:5.7
```



**-d:** 后台运行容器

**-p** 将容器的端口映射到本机的端口

**-v** 将主机目录挂载到容器的目录

**-e** 设置参数



![img](https://img-blog.csdnimg.cn/20190617135000551.png)



启动mysql容器

`docker start mysql`



![img](https://img-blog.csdnimg.cn/20190617135050730.png)



查看/root/mysql/data目录是否有数据文件

![img](https://img-blog.csdnimg.cn/20190617135810787.png)