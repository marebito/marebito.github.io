---
title: docker部署mysql：无法登陆
date: 2020-09-14 11:22:18
tags:
---



docker 部署了mysql ：无法登陆问题

1. sudo docker run -p 3306:3306 --name mysql -e MYSQL_ROOT_PASSWORD=123456 -d mysql:5.6
2. .sudo docker exec -it mysql bash
3. mysql -uroot -p123456
4. ![登陆成功图](https://www.pianshen.com/images/875/1157d024a60a748fbc86a1f61c317b03.png)

