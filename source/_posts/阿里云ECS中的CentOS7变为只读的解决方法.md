---
title: 阿里云ECS中的CentOS7变为只读的解决方法
date: 2020-04-29 09:34:24
tags:
  - 阿里云ECS
  - Linux
---

升级阿里云ECS配置及带宽后，CentOS 7系统变为只读状态，导致nginx、MySQL等服务无法启动。

<!-- more -->

解决方案如下：

* 在终端中输入mount，查看ro挂载的分区,如果发现有ro，就重新mount

1. mount方式
```
umount /dev/sda1
mount /dev/sda1 /boot
如果发现有提示“device is busy”，找到是什么进程使得他busy
fuser -m /boot 将会显示使用这个模块的pid
fuser -mk /boot 将会直接kill那个pid
然后重新mount即可。
```
2. remount方式
```
mount -o rw,remount /boot
或者mount -o remount,rw /boot
```
1. 重启
2. 使用用 fsck – y /dev/hdc6 (/dev/hdc6指你需要修复的分区) 来修复文件系统
3. 查看文件系统状态/etc/fstab发现里面是空的，于是找到了两种解决方案：
4. 重新挂载系统
 #mount -o remount,rw /                            //这样每次重新开启虚拟机都要运行一次该命令！
1. 将fstab写入正确内容
 #lsblk -f        //查看UUID

进入fstab将UUID等内容写进fstab，格式如下：
        UUID=12345678-abcd-efgh-ijkl-mnopqrstuvwx /         xfs     defaults 00

        UUID=kakjfglf-hfga-jfkg-kgjf-ggjfghkagghk         /boot xfs     defaults 0 0

        UUID=ksgsgfgd-bgga-sgfg-ghfj-jfgksbnjuths       swap  swap default0 0

        nodev/mnt/huge hugetlbfs defaults 0 0        （上面的UUID对应关系可以从上面的命令显示中看出来。另外因为我装机时分了三个区/，/boot，swap所以这里只有UUID）

        重启即可！