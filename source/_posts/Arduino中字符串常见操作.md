---
title: Arduino中字符串常见操作
date: 2020-06-11 11:12:37
tags:
---

```
isAlphaNumeric()  // 判断是否为字母数字

isAlpha()        // 判断是否为字母

isAscii()        // 判断是否为 ASCII 码

isWhitespace()    // 判断是否为空格符

isControl()          // 判断是否为控制字符

isDigit()              // 判断是否为数字

isGraph()            // 判断是否为可打印的字符，不是空格

isLowerCase()       // 判断是否为小写

isPrintable()        // 判断是否为可打印的字符

isPunct()            // 判断是否为标点符号

isSpace()            // 判断是否为空格

isUpperCase()     // 判断是否为大写

isHexadecimalDigit()  // 判断是否为十六进制数字(i.e. 0 - 9, a - F, or A - F)
```

1. 字符串转char *

```
String str = "Hello";
int str_len = str() + 1; 
char str_char[str_len];
str.toCharArray(str_char, str_len);
```

2. 字符串转整型

```
String str = "65535";
int intValue = str.toInt();
```

3. 字符串切割

```
String str = "Hello, World";
int str_len = str() + 1; 
char str_char[str_len];
strcpy(str_char, str());
const char split_token[2] = ".";
int charArray[str_len];
int i = 0;
char *charValue = strtok(str_char, split_token);                 
while( charValue != NULL ) {
    charArray[i++] = charValue;
    charValue = strtok(NULL, split_token);
}
```

> 参考连接: https://www.jianshu.com/p/5d4c5f318b73