# 用树莓派实现蓝牙照片打印 作者：Matt Richardson 翻译：Kalimov
标签（空格分隔）： 树莓派 蓝牙 打印
---
>[原文链接](http://mattrichardson.com/Raspberry-Pi-Wireless-Photo-Printing/index.html)
![](http://doask.qiniudn.com/Raspberry-Pi-Wireless-Photo-Printing1.jpg)

LG便携照片打印机PD233型支持蓝牙打印，以专用应用向其蓝牙发送3X2寸照片便能打印。虽说那个应用看似专用的，但机器只要经蓝牙接收到档案，就自动打印。

如果你有一部装有USB蓝牙的树莓派，意味着你就能直接以它打印照片。对于一些摄影棚或类似拍立得相机来说是个福音。倘若您还不熟悉如何使用树莓派的话，不妨脑补一下入门说明。

本教程用到：

树莓派
Azio BTD-V201 USB蓝牙（其他型号应该也可以，保险起见还是查看这份教程所提到确保有效工作的型号。）
LG PD233便携照片打印机
在这份教程中，我使用的是用NOOBS新装的Raspbian系统，其他版本的树莓派系统也应该可以做到。

pi@raspberrypi ~ $ uname -a
Linux raspberrypi 3.6.11+ #538 PREEMPT Fri Aug 30 20:42:08 BST 2013 armv6l GNU/Linux

设置蓝牙

插入蓝牙天线后，检查一下树莓派识别设备了没有，在命令行打入“dmesg”即可得知。翻过一大堆文字，最重要的日志记录在最后显示如下：

[  175.722175] usb 1-1.2.3: new full-speed USB device number 8 using dwc_otg
[  176.916559] usb 1-1.2.3: New USB device found, idVendor=0a12, idProduct=0001
[  176.916590] usb 1-1.2.3: New USB device strings: Mfr=0, Product=0, SerialNumber=0
[  177.019917] Bluetooth: Core ver 2.16
[  177.023810] NET: Registered protocol family 31
[  177.023840] Bluetooth: HCI device and connection manager initialized
[  177.023855] Bluetooth: HCI socket layer initialized
[  177.023864] Bluetooth: L2CAP socket layer initialized
[  177.023918] Bluetooth: SCO socket layer initialized
[  177.045934] usbcore: registered new interface driver btusb

亦可用“lsusb"命令来查看系统识别了哪些USB设备。

pi@raspberrypi ~ $ lsusb
Bus 001 Device 002: ID 0424:9512 Standard Microsystems Corp.
Bus 001 Device 001: ID 1d6b:0002 Linux Foundation 2.0 root hub
Bus 001 Device 003: ID 0424:ec00 Standard Microsystems Corp.
Bus 001 Device 004: ID 2101:8500 ActionStar
Bus 001 Device 005: ID 2101:8501 ActionStar
Bus 001 Device 006: ID 1c4f:0032 SiGma Micro
Bus 001 Device 008: ID 0a12:0001 Cambridge Silicon Radio, Ltd Bluetooth Dongle (HCI mode)
Bus 001 Device 007: ID 05af:0802 Jing-Mold Enterprise Co., Ltd

好的开始是成功的一半，下一步就看树莓派能否识别蓝牙打印机了。多亏了David的提点，以下您会用到apt-get命令安装bluez，bluez套件和obexftp的集成安装包。（一如既往，先运行sudo apt-get的更新。）

pi@raspberrypi ~ $ sudo apt-get install bluez bluez-utils obexftp

安装包需要依赖不少资源，bluez、bluez套件和obexftp需要其他程序软件资源和命令集支持，于是等多几分钟吧。值得庆幸的是，apt-get命令会自动下载安装我们需要的资源，可遇不可求啊！

搜寻打印机

做完apt-get安装更新，手头上就有了一些新的工具。其中一个，“sdptool”命令行程序是用于浏览蓝牙设备的。运行下列指令代码，利用关闭打印机后再打开来确定到底哪一个列表设备指的是打印机。

sdptool browse

原理上是找出打印机的MAC地址和频道，然后用于发送图片。MAC地址的格式类似“00:1F:47:37:7C:02”。

如果你在附近有一大堆蓝牙设备，那就头大了。要使这过程轻松些，试试限制只搜索使用OBEX推送服务的设备，而打印机正是使用这个来接收文件。正因为没多少设备使用它，不啻是个很好用于寻找您的打印机的指令。

pi@raspberrypi ~ $ sdptool search OPUSH
Inquiring ...
Searching for OPUSH on E0:F8:47:36:AB:C1 ...
Searching for OPUSH on 00:1F:47:37:7C:02 ...
Service Name: OPP
Service RecHandle: 0x10002
Service Class ID List:
  "OBEX Object Push" (0x1105)
Protocol Descriptor List:
  "L2CAP" (0x0100)
  "RFCOMM" (0x0003)
    Channel: 4
  "OBEX" (0x0008)
Profile Descriptor List:
  "OBEX Object Push" (0x1105)
    Version: 0x0100

Failed to connect to SDP server on 00:1E:C2:96:30:D5: Host is down

倘若遇上上面的出错提示，无视亦可。重点是在出错信息前出现的“Searching for OPUSH on 00:1F:47:37:7C:02 ... ”这句话，sdptool对于在蓝牙接收范围内出现的设备挨个询问它们支持的服务导致了这个现象发生。第一个设备不支持OPUSH，于是它就查询下一个设备，也就是我们的打印机。最终结果在MAC地址后列出来，就是上面所看到的。

用命令行打印照片

获得了打印机的MAC地址和频道，您就能够用obexftp发送一份文件。以下是用命令行发送一份叫做“photo. jpg"的例子，假定您已经在文件所在目录。

obexftp --nopath --noconn --uuid none --bluetooth 00:1F:47:37:7C:02 --channel 4 -p photo.jpg

头三个选项，--nopath、--noconn和--uuid none使您用OBEX 方式发送文件。简单来说，您不需要访问主机上特定路径的文件，只需建立连接，发送文件即可。

--bluetooth 00:1F:47:37:7C:02，你的打印机MAC地址。

--channel 4是您的频道。

-p photo. jpg是您要发到打印机上的文件名。

键入指令后，打印机屏幕上该有反馈显示，电源按钮附近的LED也会闪烁。片刻后，您的照片就开始打印了。

当您能用命令行控制打印，意味着设置成功了，以您使用的任何编程语言处理这条指令也不成问题。你也可以找找周围的人如何以其他计算机语言使用OBEX，这能帮您完善您的程序代码。

以Python打印照片

虽然我还没试过，Python 模组蓝灯显示它有能力以OBEX发送文件。

照片注意事项

您发送的照片长宽比应为3：2，不然打印机可能会把照片裁剪或拉伸得很奇怪。
算好裁剪出血位。既然打印机以纸张整版打印，那就必须小心裁剪照片边缘。下面照片的红框显示出血位的大致位置可用于裁剪印好的照片：
![](http://doask.qiniudn.com/Raspberry-Pi-Wireless-Photo-Printing2.jpg)
