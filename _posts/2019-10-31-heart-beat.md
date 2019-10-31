---
layout:    post
title:     基于 TCP C/S 设计的 BUG 排查及心跳设计思路
category:  TCP
tags: TCP
---

## 概述
最近在修一个之前 C/S 架构的产品的 bug：服务器端在某些情况下，会无法正确的感知到客户端连接已经断开，从而导致服务端存在很多假连接（连接的另一端已经不在了，在服务端却仍未关闭），而导致占用服务器的相关资源。

## 概念
为了方便描述，本文中 TCP 的两端我会简称为 A端（服务端） 和 Z端（客户端）。
<pre>
                     --------------                 --------------
                    | A端（服务端）  | <---socket--->|  Z端（客户端） | 
                     --------------                 -------------- 
</pre>

## 问题原因探究
网络正常情况下，TCP 协议会保证，若 Z端的连接断开，在连接的 A端是会收到相关信号，并由应用程序主动调用 close 方法来关闭连接在 A端的 socket 句柄。这样，A 到 Z 的整条链路就能正常关闭。通常大部分情况都是这样的。然而，网络这个东西，存在太多的不确定性。试想，若 **Z 端所在机器突然断电**、**A 端与 Z 端之间的网络突然中断、Z 端机器连接的网络网线突然被剪断、Z 端应用程序异常退出**，都有可能导致 **Z 端根本来不及或者无法将连接断开的信号发送到 A 端**。在这种情况下，```A 端的连接就无法正常收到此信号并关闭已经断开连接的 socket 句柄，从而导致很多假连接的产生```。请参考 [RFC-793](https://tools.ietf.org/html/rfc793) 中 TCP 的状态机图。
<pre>
                              +---------+ ---------\      active OPEN
                              |  CLOSED |            \    -----------
                              +---------+<---------\   \   create TCB
                                |     ^              \   \  snd SYN
                   passive OPEN |     |   CLOSE        \   \
                   ------------ |     | ----------       \   \
                    create TCB  |     | delete TCB         \   \
                                V     |                      \   \
                              +---------+            CLOSE    |    \
                              |  LISTEN |          ---------- |     |
                              +---------+          delete TCB |     |
                   rcv SYN      |     |     SEND              |     |
                  -----------   |     |    -------            |     V
 +---------+      snd SYN,ACK  /       \   snd SYN          +---------+
 |         |<-----------------           ------------------>|         |
 |   SYN   |                    rcv SYN                     |   SYN   |
 |   RCVD  |<-----------------------------------------------|   SENT  |
 |         |                    snd ACK                     |         |
 |         |------------------           -------------------|         |
 +---------+   rcv ACK of SYN  \       /  rcv SYN,ACK       +---------+
   |           --------------   |     |   -----------
   |                  x         |     |     snd ACK
   |                            V     V
   |  CLOSE                   +---------+
   | -------                  |  ESTAB  |
   | snd FIN                  +---------+
   |                   CLOSE    |     |    rcv FIN
   V                  -------   |     |    -------
 +---------+          snd FIN  /       \   snd ACK          +---------+
 |  FIN    |<-----------------           ------------------>|  CLOSE  |
 | WAIT-1  |------------------                              |   WAIT  |
 +---------+          rcv FIN  \                            +---------+
   | rcv ACK of FIN   -------   |                            CLOSE  |
   | --------------   snd ACK   |                           ------- |
   V        x                   V                           snd FIN V
 +---------+                  +---------+                   +---------+
 |FINWAIT-2|                  | CLOSING |                   | LAST-ACK|
 +---------+                  +---------+                   +---------+
   |                rcv ACK of FIN |                 rcv ACK of FIN |
   |  rcv FIN       -------------- |    Timeout=2MSL -------------- |
   |  -------              x       V    ------------        x       V
    \ snd ACK                 +---------+delete TCB         +---------+
     ------------------------>|TIME WAIT|------------------>| CLOSED  |
                              +---------+                   +---------+

                      TCP Connection State Diagram
                               Figure 6.
</pre>
上图中的```ESTAB```状态即为连接建立状态，如果此时 Z 端连接关闭，是会走左边，即 ```snd FIN```，向 A 端发送```FIN```，同时自身进入 ```FIN WAIT-1``` 的状态，等待 A 端返回```ACK```，如此流转下去，直到双方都最终走向```CLOSED```状态。一般而言，我们 close 一个 socket 连接，我们首先看到的是 ```TIME WAIT``` 状态，之后连接才真正关闭。<br>
所以在以上描述异常情况中，Z 端就是在```ESTAB```状态退出时，未向 A 端发送```FIN```、或发送了 A 端未接受到此信号而造成了 A 端存在假连接的问题。

## 解决方案
上述问题，实际上在 Linux kernel 上有相关的解决方案，那就是众人皆知的```KEEP_ALIVE```机制。而在实际的应用过程中，很少有人直接使用它，基本都是根据业务方需求自己来实现业务心跳，从而保证连接在极端情况下也能正常关闭，从而防止出现“假连接”、“连接泄漏”等异常问题。

### KEEP_ALIVE
这里简单描述一下其工作原理，更多详细的资料可参考 [TCP-Keepalive-HOWTO](https://www.tldp.org/HOWTO/html_single/TCP-Keepalive-HOWTO/)。
TCP内嵌有心跳包，以 A 端为例，当 A 端检测到超过一定时间(/proc/sys/net/ipv4/tcp_keepalive_time 7200 即2小时)没有数据传输，那么会向 Z端发送一个keepalive packet，此时 Z端有三种反应:
* Z端连接正常，返回一个ACK。A 端收到 ACK 后重置计时器，在2小时后在发送探测。如果2小时内连接上有数据传输，那么在该时间的基础上向后推延2小时发送探测包（该机制即保证了只有在连接空闲超过 2 小时以上才会发送心跳包，最大程度降低了网络消耗）；
* Z端异常关闭，或网络断开。Z 端无响应，A 端收不到ACK，在一定时间(/proc/sys/net/ipv4/tcp_keepalive_intvl 75 即75秒)后重发keepalive packet, 并且重发一定次数(/proc/sys/net/ipv4/tcp_keepalive_probes 9 即9次);
* Z 端曾经崩溃,但已经重启，A 端收到的探测响应是一个复位，A端终止连接。

由上述我们可以看到，tcp_keepalive_time、tcp_keepalive_intvl、tcp_keepalive_probes等参数的值都比较大，可能不能满足我们的业务需求，所以在实际应用过程中，我们可以通过来控制这些选项的实际大小，以更好的满足业务需求。两种方案可以达到修改的目的。
* 一种是修改内核关于网络方面的配置参数；
  1. /proc/sys/net/ipv4/tcp_keepalive_time
  2. /proc/sys/net/ipv4/tcp_keepalive_intvl
  3. /proc/sys/net/ipv4/tcp_keepalive_probes
* 另外一种就是通过```setsockopt```来设置SOL_TCP字段的TCP_KEEPIDLE， TCP_KEEPINTVL， TCP_KEEPCNT。
    ```c
    // 单位都是 s
    int                 keepIdle = 1000;
    int                 keepInterval = 10;
    // 次数
    int                 keepCount = 10;
    // 打开 socket 的 KEEPALIVE 机制
    int                 opt = 1;
    setsockopt(listenfd, SOL_SOCKET, SO_KEEPALIVE, &opt, sizeof(opt));
    setsockopt(listenfd, SOL_TCP, TCP_KEEPIDLE, (void *)&keepIdle, sizeof(keepIdle));
    setsockopt(listenfd, SOL_TCP,TCP_KEEPINTVL, (void *)&keepInterval, sizeof(keepInterval));
    setsockopt(listenfd, SOL_TCP, TCP_KEEPCNT, (void *)&keepCount, sizeof(keepCount));
    ```

### 业务心跳设计
有同学或许会好奇，为啥 kernel 已经有相关心跳机制了，为啥还需要在业务侧重新设计业务心跳呢？这个我谈一下自己的看法。首先，kernel 提供的心跳检测机制受 kernel 的配置影响，即使自己在程序中手动指定相关参数，也不能灵活的满足业务需求；其次，有时候业务需要对连上的连接，但是一直没有做业务操作的连接进行强制离线，以达到降低 A 端的资源使用和服务器压力，这是 kernel 的 keepalive 机制无法实现的；另外，如果我们需要在心跳包上附带一些额外信息时，这个时候 kernel 的 keepalive 机制也无法满足。```一句话总结就是，业务自己设计心跳包存在更大的灵活性，你计划可以为所欲为```。

#### 一个具体的实现方案
实际上我们可以看到，kernel 的心跳检测机制是双向的，即 A->Z 和 Z->A 端都有。然而，实际上在很多业务中，这并不一定需要。很多情况下，只需要在 A 端设置心跳检测即可。在 A 端进行心跳检测的方式简单来说有两种
1. 开启定时器，或者 daemon 线程，定时向客户端发送心跳信息，若在指定的超时时间之内接收到客户端的回复，则认为连接正常，否则在连续 n 次检测都没接收到回复，则认为客户端已经下线；
2. 由客户端，按一定的时间间隔，向服务端发送心跳包，服务端管理每个连接的最近访问时间 recent_touch_time（正常数据包或者心跳数据包过来的时间），设定一个超时时间 timeout，若检测发现 current_time - recent_touch_time > timeout，则认为客户端已经离线；
当服务器端通过上述两种方案的任意一种检测方式，发现客户端已经离线，则在服务器端直接将客户端的连接关闭掉。

#### 需要注意的点
1. ```心跳检测不需要忙检测```
即心跳检测只有在没有数据读写时才会检测，所以设定的time_out 对于普通的读写请求都需要去更新;
2. ```心跳检测需要容错```
即网络通信永远要考虑到最坏的情况，一次心跳失败，不能认定为连接不通，多次心跳失败，才能采取相应的措施。

## 总结
心跳检测是网络编程中常常遇到的问题，其目的主要是：检测 TCP 连接的两端是否都还正常在线。至于是否要同时进行双向检测，这可以根据具体业务来，没有完全的标准说"yes" or "no"。至于为什么要进行心跳检测，在文中的开篇也说到了，这里不赘述。另外想说的一点是，网络编程确实坑很多，需要注意的点、需要了解和掌握的知识都需要很丰富。如果没有这些理论知识作为支撑，写出来的应用程序可能是七疮八孔、漏洞百出的。一旦遇到类似网络编程中的协议问题，可以查看有关的标准，RFC 规范等文档，可以有效帮助了解事情的来龙去脉和根本。

## 参考资料
* [TCP-Keepalive-HOWTO](https://www.tldp.org/HOWTO/html_single/TCP-Keepalive-HOWTO/)
* [TCP在FIN_WAIT1状态到底能持续多久以及TCP假连接问题](https://blog.csdn.net/dog250/article/details/81697403)
* [SO_KEEPALIVE选项](https://www.cnblogs.com/tekkaman/p/4849767.html)