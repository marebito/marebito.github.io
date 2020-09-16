---
title: Mac下启动ssh服务
date: 2020-09-16 10:22:03
tags:
---

mac本身安装了ssh服务，默认情况下不会开机自启

1、启动sshd服务：

```ruby
$ sudo launchctl load -w /System/Library/LaunchDaemons/ssh.plist
```

2、停止sshd服务：

```ruby
$ sudo launchctl unload -w /System/Library/LaunchDaemons/ssh.plist
```

3、查看是否启动：

```cpp
$ sudo launchctl list | grep ssh
```

如果看到下面的输出表示成功启动了：

```css
 - 0  com.openssh.sshd
```