---
title: 浅谈Socket学习中的那些事
date: 2016-01-16 21:17:20
categories: iOS
tags: [SOCKET, HTTP, TCP, UDP]
---

Socket是为网络服务提供的一种机制，希望经过此次的学习，能够揭开Socket神秘面纱。在此仅想记录自己的学习历程和一些学习心得。
<!--more-->

# OSI、TCP\IP参考模型
<p>

![](http://7xq5ax.com1.z0.glb.clouddn.com/OSI%E5%8F%82%E8%80%83%E6%A8%A1%E5%9E%8B.png)
<p>

## 简单解释

1. 物理层：主要定义物理设备标准，如网线的接口类型、各种传输介质的传输速率等。2. 

2. 数据链路层：主要将从物理层接收的数据进行MAC地址的封装与解封装。

3. 网络层：选择合适的网间路由和交换结点，确保数据及时传送，将从下层接收到的数据进行IP地址的封装与解封装。

4. 传输层：定义了一些传输数据的协议和端口，如TCP、UDP协议，主要将从下层接收的数据进行分段和传输，到达目的地址后再进行重组，以往把这一层数据叫做段。

5. 会话层：通过传输层建立数据传输通路。

6. 表示层：主要是进行对接收的数据进行解释、压缩与解压缩等，即把计算机能够识别的东西转化成人能够识别的东西（如图片、声音等）。

7. 应用层：主要是一些终端的应用，比如说FTP（各种文件下载）、浏览器、QQ等，可以将其理解为在电脑屏幕上可以看到的东西，也就是终端应用。<p>

## 网络通讯要素

**IP地址**：网络中设备的标示

**端口号**：用来标示进程的逻辑地址，不同进程的标示

**传输协议**：用什么样的方式进行交互，常见协议TCP/UDP

# TCP/UDP

**TCP（传输控制协议）**

1. 建立连接，形成数据传输的通道

2. 在连接中可进行大数据传输（数据的大小不受限制）

3. 通过三次握手建立连接，可靠协议，安全送达

4. 先建立连接，效率较低<p>

**UDP（用户数据报协议）**

1. 不需要建立连接，将数据封装在数据包中

2. 每个数据包得大小限制在64k之内

3. 无需连接，是不可靠协议

4. 不需要连接，速度较快


# Socket

## 简单解释

1. 网络提供服务的一种机制
 
2. 通信的两端都是socket

3. 网络通信其实就是socket间的通信

4. 数据在两个socket间通过IO传输

![](http://7xq5ax.com1.z0.glb.clouddn.com/%E8%BE%93%E5%85%A5%E8%BE%93%E5%87%BA%E6%B5%81.png)

## iOS中常用的两种Socket类型

**流式Socket（SOCK_STREAM）**：流式是一种面向连接的Socket，针对于面向连接的TCP服务应用

**数据报式Socket（SOCK_DGRAM）**：数据报式Socket是一种无连接的Socket，对应于无连接的UDP服务应用

## Http与Socket的区别

1. Http是基于Socket的实现；Http应用层协议，主要解决如何包装数据

2. Http传输的数据格式是规定好的，Socket实现数据传输是最原始，Socket实现的数据传输格式可自定义

3. Http建立的连接称为短连接，Socket建立的连接为长连接

4. Socket是对TCP/IP协议的封装，Socket本身并不是协议，而是一个调用接口(API),通过Socket我们才能使用TCP/IP协议

# 最后
<p>
在学习的过程中会模仿微信做类似的demo，涉及到的一些相关地址如下：

1. iOS XMPP框架：https://github.com/robbiehanson/XMPPFramework

2. Server：http://www.igniterealtime.org/downloads/index.jsp

3. 数据库：http://dev.mysql.com/downloads/mysql/








