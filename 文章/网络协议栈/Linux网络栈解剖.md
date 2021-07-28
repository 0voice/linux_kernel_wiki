Linux®操作系统的最大功能之一是其网络栈。它最初是BSD协议栈的衍生物，并且组织良好，具有一组干净的接口。其接口范围从协议无关接口（如通用socket层接口或设备层）到各个网络协议的特定接口。本文从网络栈各层的角度探讨了Linux网络栈的结构，并探讨了其一些主要结构体。

## 协议介绍
虽然网络的正式介绍通常是指七层开放系统互连（OSI）模型，但是Linux中基本网络栈的介绍使用互联网模型的四层模型（参见图1）。

![image](https://user-images.githubusercontent.com/87457873/127294918-7891a1f3-3010-4018-8aae-0512afe7538a.png)

网络栈的底部是链路层。链路层是指提供访问物理层的设备驱动程序（物理层可以是各种各样的介质，诸如串行链路或以太网设备）。链路层上面是网络层，负责将数据包引导到目的地。再上一层是传输层，它负责点对点通信（例如，在主机内，像ssh 127.0.0.1）。网络层管理主机之间的通信，传输层管理这些主机之上的端点(Endpoint)之间的通信。最后是应用层，即可以理解所传输的数据的语义层。








## 核心网络架构


现在讨论Linux网络栈的架构以及它如何实现Internet模型。图2提供了Linux网络栈的高级视图。顶部是定义网络栈用户的用户空间层或应用层。底部是提供与网络（串行或高速网络（如以太网））连接的物理设备。在中间的内核空间，是本文重点要讨论的网络子系统。通过网络栈的内部流量socket缓冲区（sk_buffs），它们在源和目的之间移动数据包数据。你很快会看到sk_buff结构。

![Uploading image.png…]()


首先简要介绍Linux网络子系统的核心元素，当然后面还会有更详细的介绍。在顶部（见图2）是系统调用接口。这只是为用户空间应用程序提供访问内核网络子系统的一种方式。接下来是一个协议无关的层，提供了一种常用的方法来处理底层的传输层协议。接下来是实际的协议，在Linux中包括TCP，UDP和IP的内置协议。接下来是另一个设备无关的层，它允许使用通用接口与单个设备驱动程序交互，之后是各个设备驱动程序本身。




### 系统调用接口



系统调用接口可以从两个角度进行描述。当用户进行网络调用时，通过系统调用接口多路复用到内核中。这最终作为 `sys_socketcall(./net/socket.c)`中的调用，然后进一步解复用到其预期目标的调用。

```c
SYSCALL_DEFINE3(socket, int, family, int, type, int, protocol)
SYSCALL_DEFINE3(bind, int, fd, struct sockaddr __user *, umyaddr, int, addrlen)
SYSCALL_DEFINE2(listen, int, fd, int, backlog)
SYSCALL_DEFINE3(accept, int, fd, struct sockaddr __user *, upeer_sockaddr,
		int __user *, upeer_addrlen)
        SYSCALL_DEFINE3(connect, int, fd, struct sockaddr __user *, uservaddr,
        		int, addrlen)
...
```

系统调用接口的另一个角度是使用正常的文件操作进行网络I/O。例如，典型的读写操作可以在网络socket（由文件描述符表示，就像普通文件）一样执行。因此，虽然存在一些特定于网络的操作（调用socket创建socket，调用connect将socket连接到目的地等等），但还是有一些适用于网络对象的标准文件操作，就像常规文件一样。


### 协议无关接口
### 网络协议
### 设备无关接口
### 设备驱动程序
