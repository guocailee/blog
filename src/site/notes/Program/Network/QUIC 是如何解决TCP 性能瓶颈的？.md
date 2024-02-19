---
{"dg-publish":true,"permalink":"/Program/Network/QUIC 是如何解决TCP 性能瓶颈的？/","dgPassFrontmatter":true}
---


![](https://mmbiz.qpic.cn/mmbiz_png/cYSwmJQric6n3eFFClQt73p7bNUcqyp5dRLouxvzUpDricgUwMibIIanDnUa5wj5BnuqIj08e5rUBE2Rkic1D9iaVug/640?wx_fmt=png)
  
    

**1.1 TCP 为何会有队头阻塞问题**
----------------------

**HTTP/2** 相比HTTP/1.1 设计出的一些优秀的改进方案，大幅提高了HTTP 的网络利用效率。HTTP/2 在应用协议层通过多路复用同一个TCP连接解决了队头阻塞问题，但这是以下层协议比如TCP 协议不出现任何数据包阻塞为前提的。TCP 在实际运行中，特别是遇到网络环境不好时，数据包超时确认或丢失是常有的事，假如某个数据包丢失需要重传时会发生什么呢？  

![](https://mmbiz.qpic.cn/mmbiz_jpg/cYSwmJQric6n3eFFClQt73p7bNUcqyp5d8IQkESyh7T27EC8WsbU8h6JWVSxX7epdaXnNLf3CG6be7JgIq2vhlA/640?wx_fmt=jpeg)

TCP 采用正面确认和超时重传机制来保证数据包的可靠交付。比如主机A 向 主机B 发送数据包，主机B 收到该数据包后会向主机A 返回确认应答报文，表示自己确实收到了该数据包，主机A 收到确认应答报文后才确定上一个数据包已经发送成功，开始发送下一个数据包。

如果超过一定时间（根据每次测量的往返时间RTT估算出的动态阈值）未收到确认应答，则主机A 判断上一个数据包丢失了，重新发送上一个数据包，这就相当于阻塞了下一个数据包的发送。  

![](https://mmbiz.qpic.cn/mmbiz_png/cYSwmJQric6n3eFFClQt73p7bNUcqyp5d7DvUtWOict4R2p2E9P2Y9aqYhUaxCdgqKENEDK9ytQOjElvWWWHa9iaQ/640?wx_fmt=png)
逐个发送数据包，等待确认应答到来后再发送下一个数据包，效率太低了，TCP 采用**滑动窗口机制**来提高数据传输效率。窗口大小就是指无需等待确认应答而可以继续发送数据的最大值，这个机制实现了使用大量的缓冲区，通过对多个数据包同时进行确认应答的功能。

当可发送数据的窗口消耗殆尽时，就需要等待收到连续的确认应答后，当前窗口才会向前滑动，为发送下一批数据包腾出窗口。假设某个数据包超时未收到确认应答，当前窗口就会阻塞在原地，重新发送该数据包，在收到该重发数据包的确认应答前，就不会有新增的可发送数据包了。也就是说，因为某个数据包丢失，当前窗口阻塞在原地，同样阻塞了后续所有数据包的发送。

![](https://mmbiz.qpic.cn/mmbiz_png/cYSwmJQric6n3eFFClQt73p7bNUcqyp5dicWZFgrWt8v6MqhaApiby2m6XWwFKwb1j2ol2NVMgGsQkqia7x0aVBsyw/640?wx_fmt=png)
  

**TCP 因为超时确认或丢包引起的滑动窗口阻塞问题**，是不是有点像HTTP/1.1 管道化机制中出现的队头阻塞问题？HTTP/2 在应用协议层通过**多路复用**解决了队头阻塞问题，但TCP 在传输层依然存在队头阻塞问题，这是TCP 协议的一个主要性能瓶颈。该怎么解决TCP 的队头阻塞问题呢？

**1.2 QUIC 如何解决队头阻塞问题？**
------------------------

**TCP 队头阻塞**的主要原因是数据包超时确认或丢失阻塞了当前窗口向右滑动，我们最容易想到的解决队头阻塞的方案是不让超时确认或丢失的数据包将当前窗口阻塞在原地。QUIC (Quick UDP Internet Connections)也正是采用上述方案来解决TCP 队头阻塞问题的。

TCP 为了保证可靠性，使用了基于字节序号的 Sequence Number 及 Ack 来确认消息的有序到达。QUIC 同样是一个可靠的协议，它使用 Packet Number 代替了 TCP 的 Sequence Number，并且每个 Packet Number 都严格递增，也就是说就算 Packet N 丢失了，重传的 Packet N 的 Packet Number 已经不是 N，而是一个比 N 大的值，比如Packet N+M。  

![](https://mmbiz.qpic.cn/mmbiz_png/cYSwmJQric6n3eFFClQt73p7bNUcqyp5d3AQwv9kyFOQZCELApaBIy0nsQr5mT2h3rnDgZIHcHIia8wibYRXdCdqw/640?wx_fmt=png)

**QUIC 使用的Packet Number 单调递增的设计，可以让数据包不再像TCP 那样必须有序确认，QUIC 支持乱序确认，当数据包Packet N 丢失后，只要有新的已接收数据包确认，当前窗口就会继续向右滑动**。

待发送端获知数据包Packet N 丢失后，会将需要重传的数据包放到待发送队列，重新编号比如数据包Packet N+M 后重新发送给接收端，对重传数据包的处理跟发送新的数据包类似，这样就不会因为丢包重传将当前窗口阻塞在原地，从而解决了队头阻塞问题。那么，既然重传数据包的Packet N+M 与丢失数据包的Packet N 编号并不一致，我们怎么确定这两个数据包的内容一样呢？

  

还记得前篇博文：HTTP/2 是如何解决HTTP/1.1 性能瓶颈的？使用Stream ID 来标识当前数据流属于哪个资源请求，这同时也是数据包多路复用传输到接收端后能正常组装的依据。重传的数据包Packet N+M 和丢失的数据包Packet N 单靠Stream ID 的比对一致仍然不能判断两个数据包内容一致，还需要再新增一个字段Stream Offset，标识当前数据包在当前Stream ID 中的字节偏移量。

**有了Stream Offset 字段信息，属于同一个Stream ID 的数据包也可以乱序传输了**（HTTP/2 中仅靠Stream ID 标识，要求同属于一个Stream ID 的数据帧必须有序传输），通过两个数据包的Stream ID 与 Stream Offset 都一致，就说明这两个数据包的内容一致。  

![](https://mmbiz.qpic.cn/mmbiz_png/cYSwmJQric6n3eFFClQt73p7bNUcqyp5dafmrMtDOCJ8yjjxe8NEk5DnAhpicTEezn7tbqibA5DT0PhPq3553212A/640?wx_fmt=png)
上图中数据包Packet N 丢失了，后面重传该数据包的编号为Packet N+2，丢失的数据包和重传的数据包Stream ID 与 Offset 都一致，说明这两个数据包的内容一致。这些数据包传输到接收端后，接收端能根据Stream ID 与 Offset 字段信息正确组装成完整的资源。

QUIC 通过单向递增的Packet Number，配合Stream ID 与 Offset 字段信息，可以支持非连续确认应答Ack而不影响数据包的正确组装，摆脱了TCP 必须按顺序确认应答Ack 的限制（也即不能出现非连续的空位），解决了TCP 因某个数据包重传而阻塞后续所有待发送数据包的问题（也即队头阻塞问题）。  

QUIC 可以支持非连续的数据包确认应答Ack，自然也就要求每个数据包的确认应答Ack 都能返回给发送端（TCP 中间丢失几个Ack 对数据包的确认应答影响不大），发送端收到该数据包的确认应答后才会释放该数据包所占用的缓存资源，已发送但未收到确认应答的数据包会保存在缓存链表中等待可能的重传。

QUIC 对确认应答Ack 丢失的容忍度比较低，自然对Ack 的传输能力进行了增强，Quic Ack Frame 可以同时提供 256 个 Ack Block，在丢包率比较高的网络下，更多的 Ack Block 可以提高Ack 送达的成功率，减少重传量。  

**1.3 QUIC 没有队头阻塞的多路复用**
------------------------

QUIC 解决了TCP 的队头阻塞问题，同时继承了HTTP/2 的多路复用优点，因为Stream Offset 字段的引入，QUIC 中同一Stream ID 的数据帧也支持乱序传输，不再像HTTP/2 要求的同一Stream ID 的数据帧必须有序传输那么严格。  

![](https://mmbiz.qpic.cn/mmbiz_png/cYSwmJQric6n3eFFClQt73p7bNUcqyp5dpZuvGJIEEy63HibWRya9ysKcXTiaaX3fjLksksoBhpTlkskOTuTg3Hew/640?wx_fmt=png)
  
从上面QUIC 的数据包结构中可以看出，同一个Connection ID 可以同时传输多个Stream ID，由于QUIC 支持非连续的Packet Number 确认，某个Packet N 超时确认或丢失，不会影响其它未包含在该数据包中的Stream Frame 的正常传输。

同一个Packet Number 可承载多个Stream Frame，若该数据包丢失，则其承载的Stream Frame 都需要重新传输。因为同一Stream ID 的数据帧乱序传输后也能正确组装，这些需要重传的Stream Frame 并不会影响其它待发送Stream Frame 的正常传输。  

值得一提的是，TLS 协议加解密前需要对数据进行完整性校验，HTTP/2 中如果TCP 出现丢包，TLS 也会因接收到的数据不完整而无法对其进行处理，也即HTTP/2 中的TLS 协议层也存在队头阻塞问题，该问题如何解决呢？既然TLS 协议是因为接收数据不完整引起的阻塞，我们只需要让TLS 加密认证过程基于一个独立的Packet，不对多个Packet 同时进行加密认证，就能解决TLS 协议层出现的队头阻塞问题，某一个Packet 丢失只会影响封装该Packet 的Record，不会让其它Record 陷入阻塞等待的情况。  

**2.1 TCP连接的本质是什么？**
--------------------

你可能熟悉TCP 建立连接的三次握手和四次挥手过程，但你知道**TCP 建立的连接本质上是什么吗？**这里的连接跟我们熟悉的物理介质连接（比如电路连接）不同，主要是用来说明如何在物理介质上传输数据的。  

为了更直观了解网络连接概念，我们拿面向连接的TCP 与无连接的UDP 做对比，网络传输层的两个主流协议，他们的主要区别是什么呢？UDP 每个分组的处理都独立于所有其他分组，TCP 每个分组的传输都有确认应答过程和可能的丢包重传过程，需要为每个分组数据进行状态信息记录和管理（比如未发送、已发送、未确认、已确认等状态）。  

TCP 建立连接的三次握手过程都做了哪些工作呢？首先确认双方是否能正常收发数据，通信双方交换待发送数据的初始序列编号并作为有序确认应答的基点，通信双方根据预设的状态转换图完成各自的状态迁移过程，通信双方为分组数据的可靠传输和状态信息的记录管理分配控制块缓存资源等。下面给出TCP 连接建立、数据传输、连接释放三个阶段的报文交互过程和状态迁移图示（详见博文：TCP协议与Transmission Control Protocol）：  

![](https://mmbiz.qpic.cn/mmbiz_png/cYSwmJQric6n3eFFClQt73p7bNUcqyp5dyvgzaw9kou3PhoKDVjyL49UZiaTP0a8ibdZcfDs1o5rYTpfXxoY2ibPXA/640?wx_fmt=png)
从上图可以看出，TCP 连接主要是双方记录并同步维护的状态组成的。一般来说，建立连接是为了维护前后分组数据的承继关系，维护前后承继关系最常用的方法就是对其进行状态记录和管理。

TCP 的状态管理可以分为连接状态管理和分组数据状态管理两种，连接状态管理用于双方同步数据发送与接收状态，分组数据状态管理用于保证数据的可靠传输。涉及到状态管理一般都有状态转换图，TCP 连接管理的状态转换图上面已经给出了，HTTP/2 的Stream 实际上也记录并维护了每个Stream Frame 的状态信息，Stream 的状态转换图如下：

![](https://mmbiz.qpic.cn/mmbiz_png/cYSwmJQric6n3eFFClQt73p7bNUcqyp5deyxL0JsmsSTILEScHf933UWtjy4T5FjSVxb4fwSjfzPCWzbpM3LiaOw/640?wx_fmt=png)

**2.2 QUIC 如何减少TCP 建立连接的开销？**
-----------------------------

**TCP 建立连接需要三次握手过程**，第三次握手报文发出后不需要等待应答回复就可以发送数据报文了，所以TCP 建立连接的开销为 1-RTT。既然TCP 连接主要是由双方记录并同步维护的状态组成的，我们能否借鉴TLS 快速恢复简短握手相比完整握手的优化方案呢？  

TLS 简短握手过程是将之前完整握手过程协商的信息记录下来，以Session Ticket 的形式传输给客户端，如果想恢复之前的会话连接，可以将Session Ticket 发送给服务器，就能通过简短的握手过程重建或者恢复之前的连接，通过复用之前的握手信息可以节省 1-RTT 的连接建立开销。  

TCP 也提供了快速建立连接的方案 TFO (TCP Fast Open)，原理跟TLS 类似，也是将首次建立连接的状态信息记录下来，以Cookie 的形式传输给客户端，如果想复用之前的连接，可以将Cookie 发送给服务器，如果服务器通过验证就能快速恢复之前的连接，TFO 技术可以通过复用之前的连接将连接建立开销缩短为 0-RTT。

因为TCP 协议内置于操作系统中，操作系统的升级普及过程较慢，因此TFO 技术至今仍未普及（TFO 在2014年发布于RFC 7413）。  

![](https://mmbiz.qpic.cn/mmbiz_png/cYSwmJQric6n3eFFClQt73p7bNUcqyp5dNNdOMSWkGKa5TX74DZB2fibofHGkaADme7ibghN95AZJlWvzFia1Ll4iaw/640?wx_fmt=png)

从上图可知，TCP 首次建立连接的开销为 1-RTT，快速复用/打开连接的开销为 0-RTT，这与TLS 1.3 协议首次完整握手与快速恢复简短握手的开销一致。  

客户端发送的第一个SYN 握手包是可以携带数据的，但为了防止TCP 泛洪攻击，TCP 的实现者不允许将SYN 携带的数据包上传给应用层。HTTP 协议中TCP 与TLS 常常配合使用，这里TCP 的第一个SYN 握手包可以携带TLS 1.3 的握手包，这就可以将TCP + TLS 总的握手开销进一步降低。

**首次建立连接时**，TCP 和TLS 1.3 都只需要 1-RTT 就可以完成握手过程，由于TCP 第一个SYN 握手包可以携带TLS 的握手包，因此TCP + TLS 1.3 总的首次建立连接开销为 1-RTT。当要快速恢复之前的连接时，TFO 和TLS 1.3 都只需要 0-RTT 就可以完成握手过程，因此TCP + TLS 1.3 总的连接恢复开销为 0-RTT。  

**QUIC** 可以理解为”**TCP + TLS 1.3**“（**QUIC 是基于UDP的，可能使用的是DTLS 1.3**），QUIC 自然也实现了首次建立连接的开销为 1-RTT，快速恢复先前连接的开销为 0-RTT 的效率。QUIC 作为HTTP/2 的改进版，建立连接的开销也有明显降低，下面给出HTTP/2 和QUIC 首次连接和会话恢复过程中，HTTP 请求首个资源的RTT 开销对比：  

|   
 | HTTP/2 + TLS 1.2 首次连接 | HTTP/2 + TLS 1.2 会话恢复 | HTTP/2 + TLS 1.3 首次连接 | HTTP/2 + TLS 1.3 会话恢复 | HTTP/2 + TLS 1.3 会话恢复 + TFO | QUIC 首次连接 | QUIC 会话恢复 |
| --- | --- | --- | --- | --- | --- | --- | --- |
| DNS 解析 | 1-RTT | 0-RTT | 1-RTT | 0-RTT | 0-RTT | 1-RTT | 0-RTT |
| TCP 握手 | 1-RTT | 1-RTT | 1-RTT | 1-RTT | 0-RTT  
(TCP Fast Open) | - | - |
| TLS 握手 | 2-RTT | 1-RTT | 1-RTT | 0-RTT | 0-RTT | - | - |
| QUIC 握手 | - | - | - | - | - | 1-RTT | 0-RTT |
| HTTP 请求 | 1-RTT | 1-RTT | 1-RTT | 1-RTT | 1-RTT | 1-RTT | 1-RTT |
| 总计 | 5-RTT | 3-RTT | 4-RTT | 2-RTT | 1-RTT | 3-RTT | 1-RTT |

从上表可以看出，QUIC 首次建立连接的开销比"HTTP/2 + TLS 1.3"减少了 1-RTT，会话/连接恢复的开销降低到了 0-RTT（除去HTTP 自身请求资源的开销），显著降低了网页请求延迟。  

值得一提的是，TCP 因为报文首部是透明传输的，在安全防护方便比较脆弱，容易受到网络攻击。QUIC 因为有TLS 对数据包首部进行加密和验证，增加了安全防护强度，更不容易受到网络攻击。  

**2.3 QUIC 如何实现连接的无感迁移？**
-------------------------

每个网络连接都应该有一个唯一的标识，用来辨识并区分特定的连接。**TCP 连接使用<Source IP, Source Port, Target IP, Target Port> 这四个信息共同标识**，在早期PC 时代，这四个元素信息可以唯一标识通信双方的主机及端口，报文中也不需要一个专门的字段来标识连接，减少了传输开销。  

到了移动互联网时代，客户端（比如手机）的位置可能一直在变，接入不同的基站可能就会被分配不同的Source IP 和Source Port。即便在家里，客户端可能也需要在LTE 和WIFI 之间切换，这两个网络分配给客户端的Source IP 和Source Port 可能也是不同的。

TCP 用来标识连接的四个信息中的任何一个改变，都相当于TCP 连接标识改变了，也就变成了不同的连接，TCP 需要先断开旧的连接再建立新的连接，很显然连接切换或迁移过程不够顺畅高效。  

未来移动设备越来越多，在通话或者玩游戏等对实时性要求较高的场景中，因为网络迁移或切换导致TCP 断开连接会大大降低网络服务体验，怎么解决TCP 因为网络迁移或切换导致断线重连的问题呢？

早期移动电话使用Mobile IP 技术来解决网络迁移或切换过程引起的断连问题，Mobile IP 主要是通过新建IP 隧道的方式（也即建立一个新连接来转发数据包）保持原来的连接不断开，但这种方式增加了数据包的传输路径，也就增大了数据包的往返时间，降低了数据包的传输效率。Mobile IP 的工作原理如下（移动主机迁移到外部代理后，为了保持原连接不断开，新建了一条到归属代理的IP 隧道，让归属代理以原主机IP 转发数据包）：  

![](https://mmbiz.qpic.cn/mmbiz_png/cYSwmJQric6n3eFFClQt73p7bNUcqyp5dHWt5ktGPBgWwja9Fpotb0fPCdj3oxT0mzP0GYxwZDUx6j2HQmu3kEQ/640?wx_fmt=png)

TCP 为保持前向兼容性，没法重新设计连接标识，但为了解决移动主机连接切换问题还是推出了一套解决方案MPTCP (Multipath TCP，在2013年发布于RFC 6824) 。

针对移动主机同时支持LTE 和WIFI 等多条连接链路的情况，设计的多路径TCP 技术(MPTCP) 允许在一条TCP 链路中建立多个子通道，每个子通道都可以按照三次握手的方式建立连接，每个子通道的连接允许IP 不一致，这些子通道都会绑定到MPTCP Session（比如通过LTE 和WIFI 各建立一个子通道），发送端的数据可以选择其中一条通道进行传输。MPTCP 可以让移动主机在多个连接链路间顺畅切换，切换过程不断开连接。

对于移动主机跨基站连接迁移的问题，也可以在原基站与目标迁移基站之间各建立一个连接链路/子通道，当移动主机从一个基站迁移到另一个基站时，只是从一个链路子通道切换到另一个链路子通道，同样能让连接链路顺畅迁移而不断开连接。MPTCP 跟TFO 技术类似，需要操作系统及网络协议栈支持，更新和部署阻力较大，目前并不适用。

![](https://mmbiz.qpic.cn/mmbiz_png/cYSwmJQric6n3eFFClQt73p7bNUcqyp5dwWLSTDlVj8NoDSxLfzJGXNa9dTWFiaRvKjcxEtAmyiccSBR0bdCficTMw/640?wx_fmt=png)

**QUIC** 摆脱了TCP 的诸多限制，可以重新设计连接标识，还记得前面给出的QUIC 数据包结构吗？QUIC 数据包结构中有一个**Connection ID 字段**专门标识连接，**Connection ID** 是一个64位的通用唯一标识UUID (Universally Unique IDentifier)。

借助Connection ID，QUIC 的连接不再绑定IP 与 Port 信息，即便因为网络迁移或切换导致Source IP 和Source Port 发生变化，只要Connection ID 不变就仍是同一个连接，协议层只需要将控制块中记录的Source IP 和Source Port 信息更新即可，不需要像TCP 那样先断开连接，这就可以保证连接的顺畅迁移或切换，用户基本不会感知到连接切换过程。

![](https://mmbiz.qpic.cn/mmbiz_png/cYSwmJQric6n3eFFClQt73p7bNUcqyp5dOSJSZ7W4icv9yj5dELxUtCMWHEO5vVk1iaqV2iaqn4iaIp1TIpg3Dr0RGQ/640?wx_fmt=png)

**3.1 TCP 拥塞控制机制的瓶颈在哪？**
------------------------

计算机网络都处于一个共享环境中，可能会因为其它主机之间的通信使得**网络拥堵**，如果在通信刚开始时就突然发送大量数据，可能会导致整个网络的拥堵阻塞。TCP为了防止该问题的出现，设计了拥塞控制机制来限制数据包的发送带宽，实际就是控制发送窗口的大小。

TCP 发送数据的速率受到两个因素限制：一个是目前接收窗口的大小，通过接收端的实际接收能力来控制发送速率的机制称为流量控制机制；另一个是目前拥塞窗口的大小，通过慢启动和拥塞避免算法来控制发送速率的机制称为拥塞控制机制，TCP 发送窗口大小被限制为不超过接收窗口和拥塞窗口的较小值。  

TCP 通信开始时，会通过慢启动算法得出的拥塞窗口大小对发送数据速率进行控制，慢启动阶段拥塞窗口大小从1 开始按指数增大（每收到一次确认应答拥塞窗口值加1，收到一个窗口大小数量的确认应答则拥塞窗口大小翻倍），虽然拥塞窗口增长率较快，但由于初始值较小，增长到慢启动阈值仍然需要花费不少时间。

为了防止拥塞窗口后期增长过快，当拥塞窗口大小超过慢启动阈值（一般为发生超时重传或重复确认应答时，拥塞窗口一半的大小）后，就变更为线性增长（每收到一个窗口大小数量的确认应答则拥塞窗口大小增加一个数据段），直到发生超时重传或重复确认应答，拥塞窗口向下调整，拥塞窗口大小变化过程如下图示：  

![](https://mmbiz.qpic.cn/mmbiz_jpg/cYSwmJQric6l1HvYsCdpkibg0DvC6aofRxXSHV2M8Uds2JYmuxnbXuvaQUIaDTQEnXSu8OWP9A0cJib5sNn5Ul67A/640?wx_fmt=jpeg)

从上图可以看出，TCP 发**生超时重传**时，拥塞窗口直接下调为 1，并从慢启动阶段开始逐渐增大拥塞窗口，当超过慢启动阈值后进入拥塞避免阶段，这个过程对网络传输效率影响较大。

TCP 发生重复确认应答而触发快速重传时，判断网络拥堵情况更轻些，因此拥塞窗口下调为慢启动阈值 + 3个数据段的大小，相当于直接跨过慢启动阶段进入拥塞避免阶段，这个过程对网络传输效率影响相对较小，这种机制称为快速恢复机制。

现在网络带宽相比TCP协议刚诞生时有了明显的改善，TCP 的拥塞控制算法也成为影响网络传输效率的一个瓶颈，如果触发超时重传的次数比较多，对网络传输效率的影响相当大。  

**3.2 QUIC 如何降低重传概率？**
----------------------

TCP 的拥塞控制机制是被超时重传或者快速重传触发的，想要提高网络传输效率，容易想到两个方案：一个是改进拥塞控制算法；另一个是降低重传次数。这里先介绍如何降低重传次数/概率？

降低TCP 的重传概率有两个方向：

*   **降低超时重传概率**：可以通过改善网络环境，提高重发超时阈值的计算准确度，也就是提高往返时间RTT 的测量准确度，来降低超时重传概率；
    
*   **降低丢包重传概率**：可以增加传输一定的冗余数据比如纠错码，当丢失部分数据时可以通过纠错码恢复丢失的数据，降低丢包重传的概率。
    

由于TCP 重传 segment 的 Sequence Number 和原始的 segment 的 Sequence Number 保持不变，当发送端触发重传数据包Sequence N后，接收到了该数据包，发送端无法判断接收到的数据包是来自原始请求的响应，还是来自重传请求的响应，这就带来了TCP 重传的歧义性，该问题肯定会影响采样RTT 测量值的准确性，进而影响重发超时阈值计算的准确度，可能会增大数据包超时重传的概率。

![](https://mmbiz.qpic.cn/mmbiz_png/cYSwmJQric6n3eFFClQt73p7bNUcqyp5dCHyEz2HCly9jATIfk5OFsRmzuQz936ZbV5fUqNDhsVR98VickGm6RWQ/640?wx_fmt=png)
QUIC 采用**单向递增的Packet Number** 来标识数据包，原始请求的数据包与重传请求的数据包编号并不一样，自然也就不会引起重传的歧义性，采样RTT 的测量更准确。

除此之外，QUIC 计算RTT 时除去了接收端的应答延迟时间，更准确的反映了网络往返时间，进一步提高了RTT 测量的准确性，降低了数据包超时重传的概率。  

![](https://mmbiz.qpic.cn/mmbiz_png/cYSwmJQric6n3eFFClQt73p7bNUcqyp5dqGiavXmQewNKVCGuyasOPUwamGzIgBhsIDJpmBTVib0bkBtanpxIgsicA/640?wx_fmt=png)
TCP 传输的数据只包括校验码，并没有增加纠错码等冗余数据，如果出现部分数据丢失或损坏，只能重新发送该数据包。没有冗余的数据包虽然降低了传输开销，但增加了丢包重传概率，因为重传触发拥塞控制机制，势必会降低网络传输效率。

适当增加点冗余数据，当丢失或损坏的数据量较少时，就可以靠冗余数据恢复丢失或损坏的部分，降低丢包重传概率。只要冗余数据比例设置得当，提高的网络传输效率就可以超过增加的网络传输开销，带来网络利用率的正向提升。

**QUIC** 引入了前向冗余纠错码（**FEC: Fowrard Error Correcting**），如果接收端出现少量（不超过FEC的纠错能力）的丢包或错包，可以借助冗余纠错码恢复丢失或损坏的数据包，这就不需要再重传该数据包了，降低了丢包重传概率，自然就减少了拥塞控制机制的触发次数，可以维持较高的网络利用效率。

纠错码的原理比较复杂，如果想对纠错码有更多的了解，可以参考文章：二维码的秘密，文中简单介绍了二维码中的纠错码是如何实现信息纠错和补全的。  

**3.3 QUIC 如何改进拥塞控制机制？**
------------------------

TCP 的拥塞控制实际上包含了四个算法：**慢启动、拥塞避免、快速重传、快速恢复**。现在网络环境改善速度较快，TCP 的慢启动与拥塞避免过程需要的时间较长，虽然TCP 也在不断更新改进拥塞控制算法，但由于TCP 内置于操作系统，拥塞控制算法的更新速度太过缓慢，跟不上网络环境改善速度，TCP 落后的拥塞控制算法自然会降低网络利用效率。  

![](https://mmbiz.qpic.cn/mmbiz_png/cYSwmJQric6n3eFFClQt73p7bNUcqyp5dl4GzibUqMWhJZrSwhCt90KdS5DU3AO3ziaJb1EGDpbtVWGicicYV5RVwGQ/640?wx_fmt=png)
**QUIC** 协议当前默认使用了 TCP 的 Cubic 拥塞控制算法，同时也支持 CubicBytes、Reno、RenoBytes、BBR、PCC 等拥塞控制算法，相当于将TCP 的拥塞控制算法照搬过来了，**QUIC 是如何改进TCP 的拥塞控制算法的呢？**

**QUIC** 直接照搬TCP 的拥塞控制算法只是借鉴了TCP 经过验证的成熟方案，由于QUIC 是处于应用层的，可以随浏览器更新，QUIC 的拥塞控制算法就可以有较快的迭代速度，在TCP 的拥塞控制算法基础上快速迭代，可以跟上网络环境改善的速度，尽快提高拥塞恢复的效率。  

**QUIC** 还将拥塞控制算法设计为可插拔模块，可以根据需要为不同的连接配置不同的拥塞控制算法，这样可以为每个连接根据其网络环境配置最适合的拥塞控制算法（可以根据大数据和人工智能计算结果自动精准配置），尽可能让每个连接的网络带宽得到最高效的利用。  

> 作者：流云IoT  
> 
> https://blog.csdn.net/m0_37621078/article/details/106506532

\- EOF -

推荐阅读  点击标题可跳转

1、[看起来满是 bug 的排序代码，居然是对的](http://mp.weixin.qq.com/s?__biz=MzI1MTIzMzI2MA==&mid=2650577370&idx=1&sn=27362c4fcd73ddf527a67ae2c31bdf43&chksm=f1fe2f59c689a64f194ca6d2e53585f475f0642428ce269b2a37864a4ec06591583e1a12642d&scene=21#wechat_redirect)

2、[北大最神博士论文：为什么学校打印店老板大多是湖南人？](http://mp.weixin.qq.com/s?__biz=MzI1MTIzMzI2MA==&mid=2650577292&idx=1&sn=fce4bde69cb5f47629cc24f921a7a0ff&chksm=f1fe2f0fc689a619ff41109be7faea624bdb565688cbdadfb9e13746590503ff31b296601e81&scene=21#wechat_redirect)

3、[路径规划之 A* 算法](http://mp.weixin.qq.com/s?__biz=MzI1MTIzMzI2MA==&mid=2650577292&idx=2&sn=6720cfe8ba5e92c50cfa9b2e2533d1c4&chksm=f1fe2f0fc689a619572a1b20013e684af5ad2723d720c5dc074abd1b41d47014867ca1c86c00&scene=21#wechat_redirect)

觉得本文有帮助？请分享给更多人

****推荐关注「算法爱好者」**，修炼编程内功**

点赞和在看就是最大的支持❤️