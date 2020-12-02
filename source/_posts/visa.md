---
title: 数据采集与测量设备控制
categories:
  - Accelerator Control
tags:
  - Control
  - Accelerator
  - Instrument
  - Chinese
abbrlink: 7f1fc6ab
date: 2020-11-28 22:11:10
---


```python
# @Time    : 2020-11-28
# @Language: Markdown
# @Software: VS Code
# @Author  : Di Wang(KEK Linac)
# @Email   : sdcswd@gmail.com
```

上个月遇到了一个远程读取示波器数据的需求，于是查了一些资料，过程中遇到了很多缩写，看着名字差不多，结果有些是设备标准，有些是公司名词，有些是通信协议，着实给阅读带来不少困难，所以这篇文章就主要以示波器为例，介绍下一些概念：`LXI`，`SCPI`，`VISA`，`VXI-11`，`HiSLIP`等。

<!-- more -->

## 数据采集与测量设备控制

### 简介

先把一些英文缩写展开：

- LXI: LAN eXtensions for Instrumentation
- VXI：VME eXtensions for Instrumentation
- PXI：PCI eXtensions for Instrumentation
- SCPI：Standard Commands for Programmable Instruments
- IVI: IVI (Interchangeable Virtual Instruments) Foundation
- VISA: Virtual instrument software architecture
- HiSLIP: High-Speed LAN Instrument Protocol
- ONC/RPC: Open Network Computing / Remote Procedure Call

下面开始大白话：
这一大堆东西的存在目的：**解决不同生产商，不同的测量和数据采集设备（频谱分析仪，示波器，万用表等等），不同的数据传输协议如何互通的问题。**

为了达成这个目的，设备自身必须有一种标准，比如物理接口类型，提供控制自身的命令接口，因此数据有一套标准，同时，数据传输也要有个标准，客户端如何接收数据和发出控制命令，为了方便客户端编程，还需要一个API方便不同语言调用。

接下来往上套，俩组织，VXIplug&play Alliance 和 IVI基金会，前者如今已经并入IVI基金会，所以我也就不区分太多了，反正就是一群好事者定义了这些标准。

设备使用LXI标准，L代表了LAN，也就是区别于串口提供了基于以太网的数据传输，LXI规定了在使用TCP/IP协议中的一些细节（支持ICMP，HTTP啦等等）。在LXI基础上，VXI-11协议基于ONC/RPC（见上）允许用户控制设备和读取数据，另一种更新的替代协议是HiSLIP。

SCPI是一种设备控制命令的标准，除了通过LXI发送SCPI命令，也可以通过GPIB发送。

VISA是客户端调用的API，通过VISA建立到设备的连接（可以基于VXI-11或HiSLIP协议）发送SCPI指令，指令被设备接收到后，不同设备内部的IVI驱动会识别命令，并转换为自身设备对应的操作。目前主要有两种VISA实现，一种是NI-VISA，另一种是R&S-VISA，可以在GUI上搜索网络内的设备，建立连接，发送命令，保存日志等。我在MacBook上都安装了下，没什么大区别。（吐槽下NI-VISA连Linux都不支持差评，而R&S-VISA支持了Ubuntu和CentOS，并且还支持了树莓派！！！然而我用的是Manjaro，叹气。），下图中上面的是R&S VISA，下面的是NI-VISA。

---------------

{% asset_img visa.png VISA %}

---------------

这GUI主要还是让人测试，讲实用还是要自己编程，支持VISA的语言现在挺多，C#，python，matlab，LabVIEW等等。

### 实例

Python的VISA包可以装PyVisa或者R&S提供的RsInstrument。

直接上python代码，如何使用一目了然，代码来自罗德施瓦茨官网教程。

```python
from RsInstrument.RsInstrument import RsInstrument

resource_string_1 = 'TCPIP::rohdeosc3.linac.kek.jp::INSTR'  # Standard LAN connection (also called VXI-11)
resource_string_2 = 'TCPIP0::192.168.2.100::hislip0'  # High-Speed LAN Instrument Protocol
resource_string_3 = 'GPIB::20::INSTR'  # GPIB Connection
resource_string_4 = 'USB::0x0AAD::0x0119::022019943::INSTR'  # USB-TMC (Test and Measurement Class)
instr = RsInstrument(resource_string_2, True, False)
```

通过VISA，可以建立与设备的连接，代码中用了四种方式，前两种基于以太网，分别用了VXI-11和HiSLIP，后两种是GPIB和USB。

```python
filename = "19840328"
# Query the instrument file to the PC
instr.write_str('HCOP:DEV:LANG PNG')  # Set the screenshot format
instr.write_str(rf"MMEM:NAME 'c:\temp\screen_{filename}.png'")  # Set the screenshot path
instr.write_str("HCOP:IMM")  # Make the screenshot now
instr.query_opc()  # Wait for the screenshot to be saved
instr.read_file_from_instrument_to_pc(rf'c:\temp\screen_{filename}.png',
                                      rf'/home/sdcswd/workspace/python/rs/data/screen_{filename}.png')
print("ok")
instr.close()

```

注释写的挺详细了，基本一看函数名也就知道干嘛，`write_str`发送SCPI命令，`query_opc`执行。代码很简单的设置了截图的格式，截图文件名称和在示波器硬盘上的保存路径，然后发送截图命令，最后把图片文件从示波器传输到本机。

其中`opc`也是SCPI命令，涉及到异步处理，具体可以参考这个页面[Introducing SCPI Commands](https://www.rohde-schwarz.com/pk/driver-pages/remote-control/remote-programming-environments_231250.html#media-gallery-6)，这个页面上也给了详细的SCPI标准手册。

但是真的太长了，建议别看，直接去读自己要使用的设备手册，比如我使用的是RT0-1044 示波器，这个手册非常详细的介绍了如何发送SCPI命令进行示波器触发，测量。[RTO_HTML_UserManual](https://www.rohde-schwarz.com/webhelp/rto_html_usermanual_en_1/RTO_HTML_UserManual_en.htm)。