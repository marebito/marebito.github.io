---
title: Linux查找并终止指定进程
date: 2020-04-29 10:28:23
tags:
---

> ps -ef | grep  php | grep -v 'grep' | awk '{print $2}'| xargs kill -9