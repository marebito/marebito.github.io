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
`cat /etc/fstab`
 
 #
 # /etc/fstab
 # Created by anaconda on Sun Oct 15 15:19:00 2017
 #
 # Accessible filesystems, by reference, are maintained under '/dev/disk'
 # See man pages fstab(5), findfs(8), mount(8) and/or blkid(8) for more info
#
UUID=00e25cc9-6880-442b-b430-62bb8061394d /                       ext4    defaults        1 1
/www/swap    swap    swap    defaults    0 0

`lsblk -f`

NAME   FSTYPE LABEL UUID                                 MOUNTPOINT
vda                                                      
└─vda1 ext4         eb448abb-3012-4d8d-bcde-94434d586a31 /

fstab实际挂载点与打印的挂载点不一致，则修改/etc/fstabw文件中的挂载点为打印挂载点.

重启即可！