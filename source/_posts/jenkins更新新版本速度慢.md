---
title: jenkins更新新版本速度慢
date: 2020-06-04 09:53:53
tags:
---

* Linux
cd /root/.jenkins/updates/

* OSX
cd /Users/Shared/Jenkins/Home/updates
cd ~/.jenkins/updates

sed -i 's/http:\/\/updates.jenkins-ci.org\/download/https:\/\/mirrors.tuna.tsinghua.edu.cn\/jenkins/g' default.json && sed -i 's/http:\/\/www.google.com/https:\/\/www.baidu.com/g' default.json
