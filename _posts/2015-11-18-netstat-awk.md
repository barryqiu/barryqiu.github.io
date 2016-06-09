---
title: 利用netstat和awk统计当前系统的tcp连接数
layout: post
categories: 运维
---

# 1 问题简介

TCP连接数可以反映当前服务器的运行状态，统计出当前服务器各种类型的tcp连接可以用于服务运行监控和压力测试，netstat可以查看当前服务器的所有连接状况，利用netstat和awk工具可以进行tcp连接的统计。下面是普通netstat命令的结果：

![mem](http://7xj536.com1.z0.glb.clouddn.com/blognetstat.jpg)

图1 netstat运行结果

要充分理解这个结果还得从TCP的原理讲起

# 2 TCP的介绍

关于TCP部分的知识，仔细来讲，几本大砖头都讲不完，下面通过一张TCP的状态机把最基本的原理讲一讲：

## 2.1 TPC状态机

![mem](http://7xj536.com1.z0.glb.clouddn.com/blogtcp.jpg)

图2 TCP状态机

一个完整的TCP连接，有三个阶段:

* TCP三次握手
* 数据传送
* TCP四次挥手

几个关键值：

* SYN: (同步序列编号,Synchronize Sequence Numbers)该标志仅在三次握手建立TCP连接时有效，表示一个新的TCP连接请求。

* ACK: (确认编号,Acknowledgement Number)是对TCP请求的确认标志，同时提示对端系统已经成功接收所有数据。

* FIN: (结束标志,FINish)用来结束一个TCP回话，但对应端口仍处于开放状态,准备接收后续数据。

## 2.2 TCP状态

讲完上面的基础知识再来了解一下netstat命令返回的几种TCP状态的涵义：

* **LISTEN**：服务端需要打开一个socket进行监听，状态为LISTEN，用于侦听来自远方TCP端口的连接请求——The socket is listening for incoming connections.

* **SYN\_SENT**：客户端（在网络通信中，linux主机也可能是客户端例如应用程序相对于数据库服务器就可能是客户端）通过应用程序调用connect进行active open，于是客户端tcp发送一个SYN以请求建立一个连接，之后状态置为SYN_SENT，在发送连接请求后等待匹配的连接请求——The socket is actively attempting to establish a connection. 

* **SYN\_RECV**：服务端应发出ACK确认客户端的SYN，同时自己向客户端发送一个SYN， 之后状态置为SYN_RECV，在收到和发送一个连接请求后等待对连接请求的确认——A connection request has been received from the network. 

* **ESTABLISHED**: 代表一个打开的连接，双方可以进行或已经在数据交互了，代表一个打开的连接，数据可以传送给用户——The socket has an established connection.

* **FIN\_WAIT1**：主动关闭(active close)端应用程序调用close，于是其TCP发出FIN请求主动关闭连接，之后进入FIN_WAIT1状态，等待远程TCP的连接中断请求，或先前的连接中断请求的确认——The socket is closed, and the connection is shutting down.

* **CLOSE\_WAIT**：被动关闭(passive close)端TCP接到FIN后，就发出ACK以回应FIN请求(它的接收也作为文件结束符传递给上层应用程序)，并进入CLOSE\_WAIT，等待从本地用户发来的连接中断请求—— The remote end has shut down, waiting for the socket to close.

* **FIN\_WAIT2**：主动关闭端接到ACK后，就进入了FIN-WAIT-2，从远程TCP等待连接中断请求——Connection is closed, and the socket is waiting for a shutdown from the remote end. 

* **LAST\_ACK**：被动关闭端一段时间后，接收到文件结束符的应用程序将调用CLOSE关闭连接。这导致它的TCP也发送一个 FIN，等待对方的ACK，就进入了LAST-ACK，等待原来发向远程TCP的连接中断请求的确认——The remote end has shut down, and the socket is closed. Waiting for acknowledgement. 

* **TIME\_WAIT**：在主动关闭端接收到FIN后，TCP就发送ACK包，并进入TIME-WAIT状态，等待足够的时间以确保远程TCP接收到连接中断请求的确认——The socket is waiting after close to handle packets still in the network.

* **CLOSING**: 比较少见，等待远程TCP对连接中断的确认——Both sockets are shut down but we still don’t have all our data sent.

* **CLOSED**: 被动关闭端在接受到ACK包后，就进入了closed的状态，连接结束，没有任何连接状态——The socket is not being used. 

说明：
TIME_WAIT状态的形成只发生在主动关闭连接的一方。
主动关闭方在接收到被动关闭方的FIN请求后，发送成功给对方一个ACK后,将自己的状态由FIN\_WAIT2修改为TIME\_WAIT，而必须再等2倍 的MSL(Maximum Segment Lifetime，MSL是一个数据报在internetwork中能存在的时间)时间之后双方才能把状态都改为CLOSED以关闭连接。目前RHEL里保持TIME\_WAIT状态的时间为60秒。

# 2.3 补充——长连接和短连接

通俗点讲:短连接就是一次TCP请求得到结果后，连接马上结束。而长连接并不马上断开，而一直保持着，直到长连接TIMEOUT。长连接可以避免不断的进行TCP三次握手和四次挥手。
长连接(keepalive)是需要靠双方不断的发送探测包来维持的，keepalive期间服务端和客户端的TCP连接状态是ESTABLISHED。目前http 1.1版本里默认都是keepalive(1.0版本默认是不keepalive的)。
一个应用至于到底是该使用短连接还是长连接，应该视具体情况而定。一般的应用应该使用长连接。

# 3 使用AWK统计TCP状态

可以使用AWK工具来统计系统中各个TCP状态的连接数有多少，shell代码如下:

```bash
#注释
netstat -an | awk '/^tcp/ {++state[$NF]} END {for(i in state) print i,"\t",state[i]}'
```

运行结果如下：
![mem](http://7xj536.com1.z0.glb.clouddn.com/blognetstatawkresult.jpg)

图3 TCP连接统计

这里说明一下，使用$NF是因为TCP状态在每一行的最后一列，直接使用列数就可以了。
