<div align=center>
  
  # 👍👍👍内核最强资料：200+篇经典内核文章，100+篇内核论文，50+内核项目，500+道内核面试题，80+内核讲解视频
    
</div>
<div align=center >
<img height="240" width="280" src="https://img12.360buyimg.com/ddimg/jfs/t1/194768/6/15049/33737/60fe73c5E29d5ae0e/c5592d184e06b78e.png"></img>

#### [源码下载：9vni](https://pan.baidu.com/s/15fOf1EvhV8yv5QqmFWjLBw)
  
</div>

## 前言
在我们学习Linux内核之前，我们首先需要掌握一下几点：

* [了解Linux内核由哪些组成？](#1)（[文章直达🖱](#5)）
* [须知Linux内核源码（下载的链接👆👆👆）组织结构？](#2)（[文章直达🖱](#5)）
* [重点需要学习地知识点有哪些？](#3)（[文章直达🖱](#5)）（[论文直达🖱](#6)）（[视频直达🖱](#7)）（[面试题直达🖱](#8)）
* [最后依据我为大家提供的的学习资料，开启我们的Linux学习之旅。](#4)（[内核项目直达🖱](#9)）

<h3 id="1">Linux内核组成 </h3> 


Linux内核主要由 **进程管理**、**内存管理**、**设备驱动**、**文件系统**、**网络协议栈** 外加一个 **系统调用**。<br>
 
<br>

<div align=center >
  
<img  src="https://img11.360buyimg.com/ddimg/jfs/t1/196681/24/14941/19863/60fe7effEcdbea4bc/d066d46e595f08b9.png"></img>
  
</div>

<h3 id="2">
	
[源码组织结构](https://github.com/0voice/linux_kernel_wiki/blob/main/%E6%BA%90%E7%A0%81%E7%BB%84%E7%BB%87%E7%BB%93%E6%9E%84.png)

</h3>

<br>

<div align=center >
  
<img height="66%" width="66%" src="https://img13.360buyimg.com/ddimg/jfs/t1/186199/12/15986/607663/60fe8465E7299d74a/a4311202fe883c12.png"></img>
  
</div>

<h3 id="3">
	
[Linux内核知识体系](https://github.com/0voice/linux_kernel_wiki/blob/main/Linux%E5%86%85%E6%A0%B8%E7%9F%A5%E8%AF%86%E4%BD%93%E7%B3%BB.png)

</h3>

<br>

<div align=center >
  
<img  src="https://img11.360buyimg.com/ddimg/jfs/t1/180814/34/16127/509705/60fead2eE6a3f4a40/7de96407673e31ea.jpg"></img>
  
</div>

**内存管理**（[文章直达🖱](#5)）（[论文直达🖱](#6)）（[视频直达🖱](#7)）（[面试题直达🖱](#8)）（[内核项目直达🖱](#9)）
* 内存原理
	* SMP/NUMA模型组织
	* 页表/页表缓存
	* CPU缓存
	* 内存映射
* 虚拟内存
	* 伙伴分配器
	* 块分配器
	* 巨型页
	* 页回收
	* 页错误异常处理与反碎片技术
	* 连续内存分配器技术原理
	* 不连续页分配器原理与实现
* 内存系统调用
	* kmalloc/vmalloc
	* 内存池原理与实现
	* 内存优化与实现

<br>

**文件系统**（[文章直达🖱](#5)）（[论文直达🖱](#6)）（[视频直达🖱](#7)）（[面试题直达🖱](#8)）（[内核项目直达🖱](#9)）
* 虚拟文件系统VFS
	* 通用文件模型
	* 数据结构
	* 文件系统调用
	* 挂载文件系统
	* 无存储文件系统
* 磁盘文件系统
	* Ext2/Ext3/Ext4文件系统
	* 日志JBD2
* 用户空间系统
	* FUSE原理机制/接口与实现

<br>

**进程管理**（[文章直达🖱](#5)）（[论文直达🖱](#6)）（[视频直达🖱](#7)）（[面试题直达🖱](#8)）（[内核项目直达🖱](#9)）
* 进程基础
	* 进程原理及状态
	* 生命周期及系统调用
	* task_struct数据结构
* 进程调度
	* 调度策略
	* 进程优先级
	* 调度类分析
	* SMP调度

<br>

**网络协议栈**（[文章直达🖱](#5)）（[论文直达🖱](#6)）（[视频直达🖱](#7)）（[面试题直达🖱](#8)）（[内核项目直达🖱](#9)）
* 网络基础架构
	* SKB/net_device
	* 网络层分析
	* Linux邻近子系统
	* netlink套接字
	* iptables套接字
	* netfilter框架
	* 内核NIC接口分析
	* mac80211无线子系统
* 网络协议栈
	* internet控制消息协议（ICMP）
	* 用户数据报协议（UDP）
	* 传输控制协议（TCP）
	* 流控制传输协议（SCTP）
	* 数据报拥塞控制协议（DCCP）
	* IPv4路由选择子系统* 
	* 组播/策略/多路径路由选择
	* 接收/发送（IPv4/IPv6）数据报
	* infiniBand栈的架构
* 系统API调用
	* POSIX网络API调用
	* epoll内核原理与实现
	* 网络系统参数配置

<br>

**设备驱动**（[文章直达🖱](#5)）（[论文直达🖱](#6)）（[视频直达🖱](#7)）（[面试题直达🖱](#8)）（[内核项目直达🖱](#9)）
* 设备子系统
	* I/O机制原理
	* 设备模型
	* 字符设备子系统
	* 网络接口卡驱动
* Linux设备模型
	* LDM
	* 设备模型和sysfs
* 字符设备驱动
	* 主设备与次设备
	* 设备文件操作
	* 分配与注册字符设备
	* 写文件操作实现
* 网卡设备驱动
	* 数据结构
	* 设备方法
	* 驱动程序
* 块设备驱动
	* 资源管理
	* I/O调度
	* BIO结构原理
	* PCI总线原理
* 蓝牙子系统
	* HCI层/连接
	* L2CAP
	* BNEP
	* 蓝牙数据包接收架构


---
<h1 id="4">学习资料</h1>

<h2 id="5">文章</h2>

<h3 id="">
	
[内存管理](https://github.com/0voice/linux_kernel_wiki/tree/main/%E6%96%87%E7%AB%A0/%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86)
	
</h3>

#### [1、系列文章：](https://github.com/0voice/linux_kernel_wiki/blob/main/%E6%96%87%E7%AB%A0/%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86/%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86%EF%BC%88%E4%B8%80%EF%BC%89.md)[（一）](https://github.com/0voice/linux_kernel_wiki/blob/main/%E6%96%87%E7%AB%A0/%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86/%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86%EF%BC%88%E4%B8%80%EF%BC%89.md)[（二）](https://github.com/0voice/linux_kernel_wiki/blob/main/%E6%96%87%E7%AB%A0/%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86/%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86%EF%BC%88%E4%BA%8C%EF%BC%89.md)



<br>

<h3 id="">进程管理</h3>
待更新，敬请关注。。。
<br>

<h3 id="">文件系统</h3>
待更新，敬请关注。。。
<br>

<h3 id="">网络协议栈</h3>
待更新，敬请关注。。。
<br>

<h3 id="">设备驱动</h3>
待更新，敬请关注。。。
<br>


<h2 id="6">论文</h2>
待更新，敬请关注。。。
<br>

<h2 id="7">视频</h2>
待更新，敬请关注。。。
<br>

<h2 id="8">面试题</h2>
待更新，敬请关注。。。
<br>

<h2 id="9">内核项目</h2>
待更新，敬请关注。。。
<br>

<h1 id="5">鸣谢</h1>
