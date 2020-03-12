---
title: Modbus
date: 2020-03-12 15:07:32
tags: 
 - Ardunio
---

Modbus

  Modbus是一个通信协议，它可以用来发送和接收数据通过一条串行总线，像RS232和RS485总线。在本章中，你将学习到如何使用Modbus通讯协议通过RS485总线来连接工业设备到您的基于Arduino的PLC（可编程序逻辑控制器）。Modbus使用master-slave（主-从）架构，配置一个节点作为master（例如，Arduino PLC）并配置其他设备作为slaves（温度传感器，湿度传感器，光传感器等等）。使用RS485的好处是它仅仅使用两条共享线来连接所有设备（slaves）到主节点上。它也支持远距离以及电噪声环境中设备的使用。
  
  为了连接您的Arduino PLC到Modbus通信协议开启设备，首先你应当添加一些硬件模块到Arduino中为了开启它作为一个Modbus主节点。
  
  **RS485/Modbus模块**是一个理想组件，它可以用来在您的Arduino主板上开启Modbus通信协议。另外，你需要一个板子来与您的Arduino连接和交互。市场上有很多Arduino的RS485/Modbus模块。
  
  为了搭建开启RS485和Modbus的Arduino PLC，需要下列操作：
    * Multiprotocol Radio Shield（多协议无线电板)
    * RS485/Modbus模块

### 多协议无线电板

来自Cooking Hacks的多协议无线电板是Arduino UNO搭建开启Modbus PLCs的理想兼容板。该板被设计为同时连接两个通信模块。
![](1583998593.png)

**图 7-1.** 来自Cooking Hacks的多协议无线电板。图片由Libelium提供（[http://www.cooking-hacks.com](http://www.cooking-hacks.com))

> **注意**    多协议板有两个插口（Figure 7-2)，你可以用来连接任何开启UART（通用异步收发传输器）的硬件模块。两个插口被命名为SOCKET0和SOCKET1。插口是由2mm母排针组成的，它一共有20个连接。（UART代表通用一步收发传输器并且它是最流行的串行协议。）

![](1583999579.png)

**图 7-2.** 多协议无线电板的顶视图。图片由Libelium提供（[http://www.cooking-hacks.com](http://www.cooking-hacks.com))
  
  所有插口开启SPI（串行外设接口），所以你可以连接**RS485，RS232，**CAN(Controller Area Network控制局域网路) 总线**模块**到它们上面。对于**SOCKET0，**SPI使用**3.3V**电平，SOCKET1，SPI使用**5V**电平。
  
  有**两个**绕线式接头被焊接到板子上，所以您可以连接它们到任何Arduino UNO或者兼容的板子上。板子物理连接到Arduino像下面这样。
  
  * **Header 1**：Arduino数字引脚，0到7有八个连接。
  * **Header 2**：Arduino模拟引脚，A0到A5有六个连接。
  
  板子也包含**数字开关**来开启和禁用两个插口。您可以使用Arduino IDE中软件定义的库函数控制它们。
  
### Arduino和树莓派的RS485/Modbusm模块
  
  **对于Arduino和树莓派的RS485/Modbus模块**允许你仅仅用两根线就可以连接多于一个的工业设备到Arduino上。你可以通过带有唯一标识的设备使用两根共享线最多连接**32个设备**到Arduino。
  
![](1584000687.png)

**图 7-3.** Arduino和树莓派的RS485/Modbus模块。图片由Libelium提供（[http://www.cooking-hacks.com](http://www.cooking-hacks.com))
  
  表7-1展示了一些Arduino和树莓派的RS485/Modbus模块的技术说明，由Cooking Hacks发布。
  
  
| Standard  |  EIA RS485 |
|---|---|
| **Physical Media**  | Twisted Pair  |
| **Network Topology**  | Point-to-point, multi-dropped, multi-point  |
| **Maximum Devices**  |  Maximum Devices |
| **Voltage Levels**  |  -7V to +12V |
| **Mark(1)**  |  Positive voltages (B-A > +200mV) |
| **Space(0)**  |  Negative voltages (B-A < -200mV) |
| **Available Signals**  |  Tx+/Rx+, Tx-/Rx-(Half Duplex)Tx+,Tx-,Rx+,Rx-(Full Duplex) |

**表 7-1.** Arduino和树莓派的RS485/Modbus模块技术说明

### 为Arduino安装RS485库

为了与**RS485**一起工作，首先你应该安装**Arduino的RS485库**。使用以下步骤来安装它到Arduino IDE。

    1. 接下下载的zip文件，RS485_for_Arduino(参见第一章的“Modbus RS485库”部分获得更多关于下载链接的信息）。使用任何压缩软件。你将得到一个名为RS485_for_Arduino的文件夹。
    2. 文件夹结构和下面的层级非常相似：
    
    RS485_for_Arduino
                        -> RS485
                                    -> ModBusMaster485
                                    -> ModbusSlave485
                                    -> RS485
    
    拷贝ModBusMaster485，ModbusSlave485和RS485文件夹到你的Arduino安装的*libraries*文件夹。
    3. 最终，重启Arduino IDE并确认是否你可以看见样例sketches通过选择File ➤ Examples ➤ RS485

如果你可以看见它们，你已经为Arduino成功安装了RS485库。

### 用Modbus搭建PLC
  
  现在，您将学习如何将温度传感器与Arduino接口，以通过RS485总线使用Modbus通信协议。然后你将学习如何读取传感器值从温度传感器并再Arduino串行显示器上显示它们。
  
#### 搭建硬件设置

搭建硬件设置，需要做下面的事情。
* Arduino UNO
* 多协议无线电板
* Arduino的RS485/Modbus模块
* TQS3-I MODBUS RS485 室内温度计

以下步骤将贯穿整个构建过程。

    1. 使用绕线式接头连接**多协议无线电板**到Arduino UNO上（图 7-4）。
    2. 连接Arduino的RS485/Modbus模块，树莓派连接到**SOCKET1**（图 7-4）。

    ![](1584002689.png)

    **图 7-4.** Arduino设置RS485和Modbus。图片由Libelium提供（[http://www.cooking-hacks.com](http://www.cooking-hacks.com)

    3. **温度传感器**我们将使用来自**PAPOUCH**([www.papouch.com](www.papouch.com))**TQS3-I Modbus RS485室内温度计（图 7-5）。**它支持**Modbus**和**Spinel**通信协议通过**RS485总线**。你可以从[http://www.papouch.com/en/shop/product/tqs3-i-rs485interior-thermometer/tqs3.pdf/_downloadFile.php](http://www.papouch.com/en/shop/product/tqs3-i-rs485interior-thermometer/tqs3.pdf/_downloadFile.php)下载这个产品的文档。

 ![](1584003116.png)
 
 **图 7-5.** TQS3-I Modbus RS485室内温度计。图片由Papouch提供（[http://www.papouch.com](http://www.papouch.com)
   
   **Wago接线端**位于装置内部，用于连接电源和RS485。图7-6显示了标有电源的接线端和RS485连接。
   * 连接7-20V的直流供电，使用+极和-极。
   * 连接RS485总线，使用TX+和TX-端。

   ![](1584004096.png)
   
**图 7-6.** Wago接线端。图片由Papouch提供（[http://www.papouch.com](http://www.papouch.com)
   
   默认的情况下，这个温度传感器配置使用**Spinel**协议（[http://www.papouch.com/en/website/mainmenu/spinel/](http://www.papouch.com/en/website/mainmenu/spinel/)）来通信。对于**Modbus RTU协议**简单的跳线设置可以用来配置它，如图7-7所示，通过缩短安装跳线。
   
   ![](1584004381.png)
   
**图 7-7.** 缩短跳线的设置来启用Modbus RTU。图片由Papouch提供（[http://www.papouch.com](http://www.papouch.com)

  
  现在用两条线连接**温度传感器**到**Arduino和树莓派**的RS485/Modbus模块，如图7-8所示。
  
      1. 连接温度传感器的TX+端到RS485模块标有**A**的一端（同相信号）。
      2. 连接温度传感器的TX-端到RS485模块标有**B**的一端（反相信号）。

      ![](1584004721.png)
  
    **图 7-7.** 信号线端。Image courtesy of Libelium ([https://www.cooking-hacks.com](https://www.cooking-hacks.com))

     3. 使用壁式电源适配器将7-20V之间的DC电源连接到标有+和-的端。

#### Arduino Sketch

RS485库中提供了现成的Arduino草图。使用以下步骤来修改草图按照你的Modbus设备。

    1. 开启你的Arduino IDE并选择File ➤ Examples ➤ RS485 ➤ _RS485_04_modbus_read_input_registers，打开名为_RS485_04_modbus_read_input_registers.ino，如清单7-1所示。文件将打开一个新的窗口。

**清单 7-1** 使用Modbus读取温度传感器示例（_RS485_04_modbus_read_input_register.ino）

```
#include <RS485.h> 
#include <ModbusMaster485.h> 
#include <SPI.h>

// Instantiate ModbusMaster object as slave ID 1 ModbusMaster485 node(254);

// Define one address for reading 
#define address 101
// Define the number of bytes to read 
#define bytesQty 2

void setup() {
    // Power on the USB for viewing data in the serial monitor 
    Serial.begin(115200); 
    delay(100); 
    // Initialize Modbus communication baud rate 
    node.begin(19200);

    // Print hello message 
    Serial.println("Modbus communication over RS-485");
    delay(100);
}

void loop() {
    // This variable will store the result of the communication 
    // result = 0 : no errors 
    // result = 1 : error occurred
    int result = node.readHoldingRegisters(address, bytesQty);
    if (result != 0) { 
        // If no response from the slave, print an error message 
        Serial.println("Communication error"); 
        delay(1000); 
    } else {
        // If all OK 
        Serial.print("Read value : ");

        // Print the read data from the slave 
        Serial.print(node.getResponseBuffer(0)); 
        delay(1000);
    }
    Serial.print("\n"); 
    delay(2000);
    // Clear the response buffer 
    node.clearResponseBuffer();
}
```

    2. 现在按照你的温度传感器的存储寄存器地址修改Arduino草图中address变量的值。看到产品数据清单来找到存储寄存器的正确地址，如表7-2。

    **表 7-2. 存储寄存器的说明

| Address  |  Acess |  Function | Description  |
|---|---|---|---|
| 102 | Read  |  0x03 | RAW value, which is the value as it was received from the sensors  |

  在该示例中，温度值被存储在地址102并且可以被函数readHoldingRegisters()读取。

```
// Define one address for reading 
#define address 102
```

    3. 修改寄存器的大小（以字节为单位）。在该示例中，存储寄存器在地址102可以存储四个字节的数据。

```
// Define the number of bytes to read 
#define bytesQty 4
```

    4. 以下语句将存储ngreadHoldingRegisters()返回的结果。结果为0表示无错误，结果为1表示已发生错误。

```
int result = node.readHoldingRegisters(address, bytesQty);
``` 

    5. 要从响应缓冲区中检索数据，请在loop（）函数内使用以下语句。
  
```
getResponseBuffer(0)
```

    6. 您可以使用以下语句在Arduino串行监视器上打印来自温度传感器的数据：

```   
Serial.print(node.getResponseBuffer(0));
```

    7. 读取数据后，不要忘记清除响应缓冲区。您可以使用clearResponseBuffer（）;功能清除响应缓冲区。

    8. 现在，验证并将Arduino草图上传到Arduino开发板。成功上传草图后，通过选择工具➤串行监视器打开Arduino串行监视器。串行监视器将打印存储在响应缓冲区中的值，如图7-9所示。
    
![](1584006631.png)

**图 7-9.** Arduino串行显示器输出（读取响应缓冲）

通过使用温度传感器数据表中提到的conversion公式，可以将输出值进一步转换为摄氏或华氏温度。但是某些启用了Modbus的设备可以直接以摄氏度或华氏度输出所需的值，而无需进行任何进一步的转换。

### 总结

在本章中，您学习了如何通过RS485总线使用Modbus通信协议将工业设备连接到基于Arduino的PLC并编写基于RS485库的Arduino草图。您还学习了如何从启用了Modbus通信协议的设备中读取值。在下一章中，您将学习如何使用NearBus Cloud Connector将基于Arduino的PLC映射到云中。
