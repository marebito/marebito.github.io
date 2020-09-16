---
title: ssh重启报错或者无法重启
date: 2020-09-16 09:48:08
tags:
---

> # ssh重启报错或者无法重启-Failed to restart ssh.service: Unit not found

1.在修改了sshd_config文件之后需要重启sshd，准备执行一下命令进行重启：



```kotlin
/etc/init.d/ssh restart
```

2.发现这个路劲底下根本没有ssh，尝试以下命令：



```bash
# sudo service ssh restart
Redirecting to /bin/systemctl restart ssh.service
Failed to restart ssh.service: Unit not found.
```

出现以下错误



```css
Failed to restart ssh.service: Unit not found.
```

3.原因：以上命令都是centos6里面的命令，在centos7需要用：



```undefined
systemctl restart sshd
```