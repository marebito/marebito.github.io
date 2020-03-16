---
layout: 
title: 用arduino uno r3给ESP8266-01WiFi模块烧写固件
date: 2020-03-16 13:55:35
tags:
  - IoT
  - Arduino
---

如何在没有USB转串口烧写器的情况下，直接用arduino uno R3给ESP8266-01WiFi模块烧写固件。

<!-- more -->

一. 连接esp8266千先连接电脑打开arduino IDE给UNO板写入初始化程序，程序如下

```c
void setup()
{
    pinMode(0, INPUT_PULLUP);
    pinMode(1, INPUT_PULLUP);
}

void loop()
{
}
```

![](1584338558.jpg)

二. 把esp8826-01连接到UNO板，连接方法如下：
    
![](1584338752.jpg)

三. 打开乐鑫官网下载Flash下载工具，添加准备好的bin格式的固件，设置参数，开始下载。具体步骤见下图：

![](1584338851.jpg)

![](1584338910.jpg)

四. 烧写完成，接下来就是串口调试了，断开GPIO0引脚地连线，打开串口调试工具，选择正确的波特率和端口后点击打开串口，然后插拔CH_PD引脚连线，当出现乱码固件的版本号就显示出来了，OK！

![](1584339052.jpg)
    
> 注意事项
* 在点击开始以后短暂断开CH_PD引脚的接线，重新上电.
* GPIO0引脚必须拉低接GND.
* 调试时必须断开GPIO0引脚.
* VCC和CH_PD引脚供电必须为3.3V.