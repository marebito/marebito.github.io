---
title: NodeMCU使用OLED
date: 2020-05-09 09:08:58
tags:
  - NodeMCU
  - OLED
---

硬件准备：
1. NodeMCU
2. I2c-12864-oled液晶屏模块 0.96寸 （12864就是128*64点阵的屏幕，所以叫12864！）
![](1588986804.jpg)

3. 母对母杜邦线4根
4. microUSB口-用来连接ESP8266

软件准备：
Win：https://github.com/nodemcu/nodemcu-flasher/tree/master/Win64/Release
Mac：https://github.com/marcelstoer/nodemcu-pyflasher/releases

<!-- more -->

## 给NodeMCU刷入固件

1. 在nodemcu官网https://nodemcu-build.com/中下载固件

| 基本信息 | 参数 |
|---|---|
| 驱动电压  | 3.3~5V |
| 分辨率  | 128x64 |
| 驱动接口 | I2C |
| I2C地址 | 0x3c(默认)\0x3d可选 |

![](1588987148.png)
构建模块中勾选I2C和U8G2。
字体需要勾选中文，默认不支持中文字体
```
u8g2_font_wqy15_t_chinese1
u8g2_font_wqy15_t_chinese2
u8g2_font_wqy15_t_chinese3
```
![](1588987612.png)
![](1588988007.png)

在邮箱中查看
![](1588988246.png)

一般下载integer的固件

## 用ESPFlashDownloadTool写入固件
## 按照接线图进行接线
![](1588988622.png)

| NodeMCU | OLED |
|---|---|
| D5  | SDA |
| D6  | SCL |
| GND | GND |
| 3V | VCC |


## 测试程序

```c++
//"hello world" test
#include <U8g2lib.h>

U8G2_SSD1306_128X64_NONAME_F_HW_I2C u8g2(U8G2_R0);

void setup(void) {
  u8g2.begin();
}

void loop(void) {
  u8g2.clearBuffer();                   // clear the internal memory
  u8g2.setFont(u8g2_font_ncenB14_tr);   // choose a suitable font
  u8g2.drawStr(0,20,"Hello World!");    // write something to the internal memory
  u8g2.sendBuffer();                    // transfer internal memory to the display
  delay(1000);  
}
```

