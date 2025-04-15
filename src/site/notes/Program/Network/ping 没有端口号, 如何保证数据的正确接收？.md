---
{"dg-publish":true,"permalink":"/Program/Network/ping 没有端口号, 如何保证数据的正确接收？/","noteIcon":""}
---



简单说，网卡只是总线上插着的一块普通设备；这个设备可以接受来自网线的、高低起伏的电压，并把这些电压解读为二进制流。

![](https://picx.zhimg.com/80/v2-c76d5d5f0886d2ce3b055dbc62c297d8_720w.webp?source=1940ef5c)

如果“网卡”是简单的COM口通讯，那么二进制流可能使用0x7F表示数据报文开始，0xF7表示数据报文结束。

如果数据中本身就存在7F/F7，则需要做“转义”，就好像在C字符串中不能直接敲`NUL`，而要用`\\`转义阿拉伯数字`0`，从而表示出一个全零字节一样——当然，`\\`字符本身也要转义，比如`\\\`就代表`\\`自身。

  

我们常用的网卡叫“以太网卡”，它的报文有稍微复杂一些的格式，叫做以太网帧：

![](https://picx.zhimg.com/80/v2-2265fb03818268848430d91a72d2db63_720w.webp?source=1940ef5c)

> 前导码：`Ethernet II`是由8个`8‘b10101010`构成，`IEEE802.3`由7个`8‘b10101010+1`个字节SFD..  
> 目的地址：目的设备的MAC物理地址。  
> 源 地址 ：发送设备的MAC物理地址。  
> 类型(`Ethernet II`)：以太网首部 后面所跟数据包的类型，例如Type为0x8000时为IP协议包，Type为8060时，后面为ARP协议包。  
> 长度(IEEE802.3)：当长度小于1500时，说明该帧为`IEEE802.3`帧格式，大于`1500`时，说明该帧为`Ethernet II`帧格式。  
> 数据：数据长度最小为46字节，不足`46`字节时，填充至`46`字节。因为最小帧长度是64字节，所以，`46+6+6+2+4=64`。（不算前导码）  
> FCS: 就是CRC校验值

[以太网数据格式与封装解封](https://www.cnblogs.com/qishui/p/5437301.html)
![](https://pic4.zhimg.com/v2-a31f7782726e3ab502a60a82d29f9eab_180x120.jpg)
  

和`COM口`通讯一样，以太网的[[Program/OS/驱动程序\|驱动程序]]（可以直接固化在网卡固件里，从而不需要驱动，而是直接用网卡自带的芯片解码）通过“前导码”从位流中识别出以太网帧、查看其中的目的地址`MAC`，确认它是不是给自己的——如果不是，就丢弃它（当然，如果网卡被设置到混杂模式Promiscuous Mode，则数据并不会自动丢弃；此时你可以直接在以太网层监控局域网本地交换机下的所有通信）。

然后，IP报文等就在以太网帧的“载荷”，也就是那46~1500字节的“数据”中。

  

这个“载荷”会交给IP协议栈（一组驱动程序，也可以是网卡上的芯片）处理。

第一步，仍然是把数据按IP报文解析：

![](https://pica.zhimg.com/80/v2-1922c8f9653c340aba0093cc3917739f_720w.webp?source=1940ef5c)

注意这里做了一定的简化：实质上，ARP协议虽然是IP协议的一部分，但它是一个单独的、不带IP头的封包。换句话说，并不是所有以太网数据都是IP报文。这要通过首字节的版本号等信息区分。

  

IP数据报文又分为三种子格式，也就是ICMP、TCP、UDP。它们都有一个IP头。

换句话说，ICMP、TCP、UDP都是IP报文的数据。

这是IP报文：

![](https://pica.zhimg.com/80/v2-7b5aef2496d88ae8593fa69cfc663493_720w.webp?source=1940ef5c)

里面可能是如下三种数据之一：

TCP报文：

![](https://pic1.zhimg.com/80/v2-6fa9b9a90a35071a8f3f07a54f466d6f_720w.webp?source=1940ef5c)

UDP报文：

![](https://pic1.zhimg.com/80/v2-dea42041e342153f66be703f99381f8a_720w.webp?source=1940ef5c)

ICMP报文：

![](https://picx.zhimg.com/80/v2-b6d1afc50540bf6d7b28493e24eced45_720w.webp?source=1940ef5c)

  

注意它们都是IP报文的载荷；也就是说，在IP栈内部，要根据IP头格式识别数据格式，然后派发给ICMP、TCP、UDP模块处理。

  

其中，ICMP相关信息直接在驱动层处理就可以了，也就是停留在操作系统空间就可以，不需要用户空间的应用参与处理，它会按照协议规定自动动作。

当然，用户层也可以访问它，使用SOCK_RAW建立socket链接即可。ping命令就是这样编写的，可参考：

[通信编程：基于 ICMP 编写 ping 程序​www.cnblogs.com/linfangnan/p/15729663.html![](https://pic3.zhimg.com/v2-dcf0f09bf77f23257c92a644f08cd50a_180x120.jpg)
](https://link.zhihu.com/?target=https%3A//www.cnblogs.com/linfangnan/p/15729663.html)

  

注意它和TCP/UDP不同。

后两者是通过专门的处理模块、和用户空间应用直接对应的。对应方式就是源端口/目的端口，也就是通过一个四元组 (源地址，目标地址，源端口，目标端口) 唯一的标记一条“虚拟链路”。

而ICMP并没有约定端口号之类东西；对它的读写都直接在IP层进行（SOCK_RAW），是和TCP/UDP不同的另外一套机制。

  

换句话说，它所依赖的IP报文总是可以被正确的投递（到ARP绑定的MAC地址所在的那个网段）、并被预设机制自动响应。

但，如果你想写一个ping程序、支持“同时启动N个ping进程ping远程主机、却又不相互干扰”的话，你需要自己设计一个机制，避免接收不属于自己的echo报文。

  

另外，如果你很熟悉TCP/UDP通讯，就会知道每次从socket读取一个报文，这个报文就会从协议链上被移除；但SOCK_RAW的行为是不一样的：

> It is important to understand that some sockets of type**SOCK_RAW**may receive many unexpected datagrams. For example, a PING program may create a socket of type**SOCK_RAW**to send ICMP echo requests and receive responses. While the application is expecting ICMP echo responses, all other ICMP messages (such as ICMP HOST_UNREACHABLE) may also be delivered to this application. Moreover, if several**SOCK_RAW**sockets are open on a computer at the same time, the same datagrams may be delivered to all the open sockets. An application must have a mechanism to recognize the datagrams of interest and to ignore all others. For a PING program, such a mechanism might include inspecting the received IP header for unique identifiers in the ICMP header (the application's process ID, for example).

[TCP/IP raw sockets - Win32 apps​learn.microsoft.com/en-us/windows/win32/winsock/tcp-ip-raw-sockets-2![](https://pic1.zhimg.com/v2-7c294118a0fbc8fe10908b4211dab938_180x120.jpg)
](https://link.zhihu.com/?target=https%3A//learn.microsoft.com/en-us/windows/win32/winsock/tcp-ip-raw-sockets-2)

根据微软的说明，所有的icmp消息都会返回给每一个试图读取它的应用。也就是你的读取并不会导致ICMP数据从协议链上被删除，每一个报文，不管是不是给你的，都会被你接收到——如果你写了一个ping，然后用户启动了这个ping的100个实例，那么100倍的icmp报文会无遗漏的被每一个ping进程获取。

因此，你必须自己识别你发出去的ping包；ICMP要求echo服务必须复制请求者发来的任何数据载荷，因此你可以在自己的ping包中放入自己的进程ID (pid)，从而把每个进程发出去的ping包区分开来。

[发布于 2023-08-17 11:03](//www.zhihu.com/question/608100461/answer/3169439648)・IP 属地广东
