---
title: 使用grep、sed正则提取字符串
date: 2020-09-03 14:13:51
tags: - 正则
	  - 提取字符串
---

## grep

```
echo "number : 123456" | grep -oP '(?<=number : )(\d+)'
```

## sed

```
echo "number : 123456" | sed  's/number : \([0-9]*\)/\1/g'
```

