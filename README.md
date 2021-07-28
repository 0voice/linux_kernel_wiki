<div align=center>
  
  # 👍👍👍Linux内核最强资料：200+篇经典内核文章，100+篇内核论文，50+内核项目，500+道内核面试题，80+内核讲解视频
    
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


	
**[内存管理](https://github.com/0voice/linux_kernel_wiki/tree/main/%E6%96%87%E7%AB%A0/%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86)**

[1、硬件原理 和 分页管理](https://github.com/0voice/linux_kernel_wiki/blob/main/%E6%96%87%E7%AB%A0/%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86/%E7%A1%AC%E4%BB%B6%E5%8E%9F%E7%90%86%20%E5%92%8C%20%E5%88%86%E9%A1%B5%E7%AE%A1%E7%90%86.md)<br>
[2、内存的动态申请和释放](https://github.com/0voice/linux_kernel_wiki/blob/main/%E6%96%87%E7%AB%A0/%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86/%E5%86%85%E5%AD%98%E7%9A%84%E5%8A%A8%E6%80%81%E7%94%B3%E8%AF%B7%E5%92%8C%E9%87%8A%E6%94%BE.md)<br>
[3、进程的内存消耗和泄漏](https://github.com/0voice/linux_kernel_wiki/blob/main/%E6%96%87%E7%AB%A0/%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86/%E8%BF%9B%E7%A8%8B%E7%9A%84%E5%86%85%E5%AD%98%E6%B6%88%E8%80%97%E5%92%8C%E6%B3%84%E6%BC%8F.md)<br>
[4、内存与I/O的交换](https://github.com/0voice/linux_kernel_wiki/blob/main/%E6%96%87%E7%AB%A0/%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86/%E5%86%85%E5%AD%98%E4%B8%8EIO%E7%9A%84%E4%BA%A4%E6%8D%A2.md)<br>
[5、其他工程问题以及调优](https://github.com/0voice/linux_kernel_wiki/blob/main/%E6%96%87%E7%AB%A0/%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86/%E5%85%B6%E4%BB%96%E5%B7%A5%E7%A8%8B%E9%97%AE%E9%A2%98%E4%BB%A5%E5%8F%8A%E8%B0%83%E4%BC%98.md)<br>

<br>

**[进程管理](https://github.com/0voice/linux_kernel_wiki/tree/main/%E6%96%87%E7%AB%A0/%E8%BF%9B%E7%A8%8B%E7%AE%A1%E7%90%86)**

[1、Linux进程、线程、调度(一)](https://github.com/0voice/linux_kernel_wiki/blob/main/%E6%96%87%E7%AB%A0/%E8%BF%9B%E7%A8%8B%E7%AE%A1%E7%90%86/Linux%E8%BF%9B%E7%A8%8B%E3%80%81%E7%BA%BF%E7%A8%8B%E3%80%81%E8%B0%83%E5%BA%A6(%E4%B8%80).md)<br>
[2、Linux进程、线程、调度(二)](https://github.com/0voice/linux_kernel_wiki/blob/main/%E6%96%87%E7%AB%A0/%E8%BF%9B%E7%A8%8B%E7%AE%A1%E7%90%86/Linux%E8%BF%9B%E7%A8%8B%E3%80%81%E7%BA%BF%E7%A8%8B%E3%80%81%E8%B0%83%E5%BA%A6(%E4%BA%8C).md)<br>
[3、Linux进程、线程、调度(三)](https://github.com/0voice/linux_kernel_wiki/blob/main/%E6%96%87%E7%AB%A0/%E8%BF%9B%E7%A8%8B%E7%AE%A1%E7%90%86/Linux%E8%BF%9B%E7%A8%8B%E3%80%81%E7%BA%BF%E7%A8%8B%E3%80%81%E8%B0%83%E5%BA%A6(%E4%B8%89).md)<br>
[4、Linux进程、线程、调度(四)](https://github.com/0voice/linux_kernel_wiki/blob/main/%E6%96%87%E7%AB%A0/%E8%BF%9B%E7%A8%8B%E7%AE%A1%E7%90%86/Linux%E8%BF%9B%E7%A8%8B%E3%80%81%E7%BA%BF%E7%A8%8B%E3%80%81%E8%B0%83%E5%BA%A6(%E5%9B%9B).md)<br>

<br>

**[文件系统](https://github.com/0voice/linux_kernel_wiki/tree/main/%E6%96%87%E7%AB%A0/%E6%96%87%E4%BB%B6%E7%B3%BB%E7%BB%9F)**

[1、Linux 操作系统原理-文件系统(一)](https://github.com/0voice/linux_kernel_wiki/blob/main/%E6%96%87%E7%AB%A0/%E6%96%87%E4%BB%B6%E7%B3%BB%E7%BB%9F/Linux%20%E6%93%8D%E4%BD%9C%E7%B3%BB%E7%BB%9F%E5%8E%9F%E7%90%86-%E6%96%87%E4%BB%B6%E7%B3%BB%E7%BB%9F(1).md)<br>
[2、Linux 操作系统原理-文件系统(二)](https://github.com/0voice/linux_kernel_wiki/blob/main/%E6%96%87%E7%AB%A0/%E6%96%87%E4%BB%B6%E7%B3%BB%E7%BB%9F/Linux%20%E6%93%8D%E4%BD%9C%E7%B3%BB%E7%BB%9F%E5%8E%9F%E7%90%86-%E6%96%87%E4%BB%B6%E7%B3%BB%E7%BB%9F(2).md)

<br>

**[网络协议栈](https://github.com/0voice/linux_kernel_wiki/tree/main/%E6%96%87%E7%AB%A0/%E7%BD%91%E7%BB%9C%E5%8D%8F%E8%AE%AE%E6%A0%88)**

[1、Linux内核网络udp数据包发送(一)](https://github.com/0voice/linux_kernel_wiki/blob/main/%E6%96%87%E7%AB%A0/%E7%BD%91%E7%BB%9C%E5%8D%8F%E8%AE%AE%E6%A0%88/Linux%E5%86%85%E6%A0%B8%E7%BD%91%E7%BB%9Cudp%E6%95%B0%E6%8D%AE%E5%8C%85%E5%8F%91%E9%80%81(%E4%B8%80).md)<br>
[2、Linux内核网络udp数据包发送（二）-UDP协议层分析](https://github.com/0voice/linux_kernel_wiki/blob/main/%E6%96%87%E7%AB%A0/%E7%BD%91%E7%BB%9C%E5%8D%8F%E8%AE%AE%E6%A0%88/Linux%E5%86%85%E6%A0%B8%E7%BD%91%E7%BB%9Cudp%E6%95%B0%E6%8D%AE%E5%8C%85%E5%8F%91%E9%80%81%EF%BC%88%E4%BA%8C%EF%BC%89-UDP%E5%8D%8F%E8%AE%AE%E5%B1%82%E5%88%86%E6%9E%90.md)<br>
[3、Linux内核网络UDP数据包发送（三）—IP协议层分析](https://github.com/0voice/linux_kernel_wiki/blob/main/%E6%96%87%E7%AB%A0/%E7%BD%91%E7%BB%9C%E5%8D%8F%E8%AE%AE%E6%A0%88/Linux%E5%86%85%E6%A0%B8%E7%BD%91%E7%BB%9CUDP%E6%95%B0%E6%8D%AE%E5%8C%85%E5%8F%91%E9%80%81%EF%BC%88%E4%B8%89%EF%BC%89%E2%80%94IP%E5%8D%8F%E8%AE%AE%E5%B1%82%E5%88%86%E6%9E%90.md)<br>
[4、Linux操作系统原理—内核网络协议栈](https://github.com/0voice/linux_kernel_wiki/blob/main/%E6%96%87%E7%AB%A0/%E7%BD%91%E7%BB%9C%E5%8D%8F%E8%AE%AE%E6%A0%88/Linux%E6%93%8D%E4%BD%9C%E7%B3%BB%E7%BB%9F%E5%8E%9F%E7%90%86%E2%80%94%E5%86%85%E6%A0%B8%E7%BD%91%E7%BB%9C%E5%8D%8F%E8%AE%AE%E6%A0%88.md)<br>
[5、Linux网络栈解剖](https://github.com/0voice/linux_kernel_wiki/blob/main/%E6%96%87%E7%AB%A0/%E7%BD%91%E7%BB%9C%E5%8D%8F%E8%AE%AE%E6%A0%88/Linux%E7%BD%91%E7%BB%9C%E6%A0%88%E8%A7%A3%E5%89%96.md)<br>

<br>

**[设备驱动](https://github.com/0voice/linux_kernel_wiki/tree/main/%E6%96%87%E7%AB%A0/%E8%AE%BE%E5%A4%87%E9%A9%B1%E5%8A%A8)**

[1、Linux 总线、设备、驱动模型的探究](https://github.com/0voice/linux_kernel_wiki/blob/main/%E6%96%87%E7%AB%A0/%E8%AE%BE%E5%A4%87%E9%A9%B1%E5%8A%A8/Linux%20%E6%80%BB%E7%BA%BF%E3%80%81%E8%AE%BE%E5%A4%87%E3%80%81%E9%A9%B1%E5%8A%A8%E6%A8%A1%E5%9E%8B%E7%9A%84%E6%8E%A2%E7%A9%B6.md)<br>
[2、Linux 设备和驱动的相遇](https://github.com/0voice/linux_kernel_wiki/blob/main/%E6%96%87%E7%AB%A0/%E8%AE%BE%E5%A4%87%E9%A9%B1%E5%8A%A8/Linux%20%E8%AE%BE%E5%A4%87%E5%92%8C%E9%A9%B1%E5%8A%A8%E7%9A%84%E7%9B%B8%E9%81%87.md)<br>

<br>

<h2 id="6">论文</h2>

<div align=center>

No.|Title|Translation（参考）|Company
:-------: | :---------------: | :------------: | :-------:
1|[《A dataset of feature additions and feature removals from the Linux kernel》](https://github.com/0voice/linux_kernel_wiki/blob/main/%E8%AE%BA%E6%96%87/%E3%80%8AA%20dataset%20of%20feature%20additions%20and%20feature%20removals%20from%20the%20Linux%20kernel%E3%80%8B.pdf)|《从Linux内核中添加和删除特性的数据集》| 滑铁卢大学
2|[《A Novel DDoS Floods Detection and Testing Approaches for Network Traffic based on Linux Techniques》](https://github.com/0voice/linux_kernel_wiki/blob/main/%E8%AE%BA%E6%96%87/%E3%80%8AA%20Novel%20DDoS%20Floods%20Detection%20and%20Testing%20Approaches%20for%20Network%20Traffic%20based%20on%20Linux%20Techniques%E3%80%8B.pdf)|《基于Linux技术的网络流量DDoS flood检测与测试方法》| 大连理工大学软件学院
3|[《A Permission Check Analysis Framework for Linux Kernel》](https://github.com/0voice/linux_kernel_wiki/blob/main/%E8%AE%BA%E6%96%87/%E3%80%8AA%20Permission%20Check%20Analysis%20Framework%20for%20Linux%20Kernel%E3%80%8B.pdf)|《Linux内核权限检查分析框架》| 弗吉尼亚理工学院
4|[《An Evaluation of Adaptive Partitioning of Real-Time Workloads on Linux》](https://github.com/0voice/linux_kernel_wiki/blob/main/%E8%AE%BA%E6%96%87/%E3%80%8AAn%20Evaluation%20of%20Adaptive%20Partitioning%20of%20Real-Time%20Workloads%20on%20Linux%E3%80%8B.pdf)|《Linux下实时工作负载自适应分区的评估》| Red Hat
5|[《Analysis and Study of Security Mechanisms inside Linux Kernel 》](https://github.com/0voice/linux_kernel_wiki/blob/main/%E8%AE%BA%E6%96%87/%E3%80%8AAnalysis%20and%20Study%20of%20Security%20Mechanisms%20inside%20Linux%20Kernel%20%E3%80%8B.pdf)|《Linux内核内部安全机制分析与研究》| 北京交通大学计算机与信息技术学院
6|[《Architecture of the Linux kernel》](https://github.com/0voice/linux_kernel_wiki/blob/main/%E8%AE%BA%E6%96%87/%E3%80%8AArchitecture%20of%20the%20Linux%20kernel%E3%80%8B.pdf)|《Linux内核的架构》| 未知
7|[《Automated Patch Backporting in Linux》](https://github.com/0voice/linux_kernel_wiki/blob/main/%E8%AE%BA%E6%96%87/%E3%80%8AAutomated%20Patch%20Backporting%20in%20Linux%E3%80%8B.pdf)|《Linux中的自动补丁支持》| 新加坡国立大学
8|[《Automated Voxel Placement A Linux-based Suite of Tools for Accurate and Reliable Single Voxel Coregistration》](https://github.com/0voice/linux_kernel_wiki/blob/main/%E8%AE%BA%E6%96%87/%E3%80%8AAutomated%20Voxel%20Placement%20A%20Linux-based%20Suite%20of%20Tools%20for%20Accurate%20and%20Reliable%20Single%20Voxel%20Coregistration%E3%80%8B.pdf)|《自动化体素放置基于linux的精确可靠的单体素共配准工具套件》| 韦恩州立大学
9|[《Automatic Rebootless Kernel Updates》](https://github.com/0voice/linux_kernel_wiki/blob/main/%E8%AE%BA%E6%96%87/%E3%80%8AAutomatic%20Rebootless%20Kernel%20Updates%E3%80%8B.pdf)|《自动重启内核更新》| 麻省理工学院
10|[《Communication on Linux using Socket Programming in ‘C’》](https://github.com/0voice/linux_kernel_wiki/blob/main/%E8%AE%BA%E6%96%87/%E3%80%8ACommunication%20on%20Linux%20using%20Socket%20Programming%20in%20%E2%80%98C%E2%80%99%E3%80%8B.pdf)|《基于C语言套接字编程的Linux通信》| 印度北阿坎德邦现代技术学院计算机科学与信息技术系
11|[《Compatibility of Linux Architecture for Diskless Technology System》](https://github.com/0voice/linux_kernel_wiki/blob/main/%E8%AE%BA%E6%96%87/%E3%80%8ACompatibility%20of%20Linux%20Architecture%20for%20Diskless%20Technology%20System%E3%80%8B.pdf)|《Linux体系结构对无磁盘技术系统的兼容性》| 台南科技大学电气工程学院
12|[《Concurrency in the Linux kernel》](https://github.com/0voice/linux_kernel_wiki/blob/main/%E8%AE%BA%E6%96%87/%E3%80%8AConcurrency%20in%20the%20Linux%20kernel%E3%80%8B.pdf)|《Linux内核中的并发性》| 伦敦大学
13|[《Container-based real-time scheduling in the Linux kernel》](https://github.com/0voice/linux_kernel_wiki/blob/main/%E8%AE%BA%E6%96%87/%E3%80%8AContainer-based%20real-time%20scheduling%20in%20the%20Linux%20kernel%E3%80%8B.pdf)|《Linux内核中基于容器的实时调度》| 未知
14|[《Crash Consistency Test Generation for the Linux Kernel》](https://github.com/0voice/linux_kernel_wiki/blob/main/%E8%AE%BA%E6%96%87/%E3%80%8ACrash%20Consistency%20Test%20Generation%20for%20the%20Linux%20Kernel%E3%80%8B.pdf)|《Linux内核的崩溃一致性测试生成》|德克萨斯大学奥斯汀分校
15|[《Designing of a Virtual File System》](https://github.com/0voice/linux_kernel_wiki/blob/main/%E8%AE%BA%E6%96%87/%E3%80%8ADesigning%20of%20a%20Virtual%20File%20System%E3%80%8B.pdf)|《虚拟文件系统的设计》| 未知
16|[《Efficient Formal Verification for the Linux Kernel》](https://github.com/0voice/linux_kernel_wiki/blob/main/%E8%AE%BA%E6%96%87/%E3%80%8AEfficient%20Formal%20Verification%20for%20the%20Linux%20Kernel%E3%80%8B.pdf)|《Linux内核的有效正式验证》| RETIS实验室
17|[《Exploiting Uses of Uninitialized Stack Variables in Linux Kernels to Leak Kernel Pointers》](https://github.com/0voice/linux_kernel_wiki/blob/main/%E8%AE%BA%E6%96%87/%E3%80%8AExploiting%20Uses%20of%20Uninitialized%20Stack%20Variables%20in%20Linux%20Kernels%20to%20Leak%20Kernel%20Pointers%E3%80%8B.pdf)|《利用Linux内核中未初始化的堆栈变量泄漏内核指针》| 亚利桑那州立大学
18|[《Hybrid Fuzzing on the Linux Kernel》](https://github.com/0voice/linux_kernel_wiki/blob/main/%E8%AE%BA%E6%96%87/%E3%80%8AHybrid%20Fuzzing%20on%20the%20Linux%20Kernel%E3%80%8B.pdf)|《Linux内核的混合Fuzzing》| 俄勒冈州立大学
19|[《In-Process Memory Isolation for Modern Linux Systems》](https://github.com/0voice/linux_kernel_wiki/blob/main/%E8%AE%BA%E6%96%87/%E3%80%8AIn-Process%20Memory%20Isolation%20for%20Modern%20Linux%20Systems%E3%80%8B.pdf)|《面向现代Linux系统的进程内内存隔离》| 罗格斯大学
20|[《Introduction to the Linux kernel challenges and case studies》](https://github.com/0voice/linux_kernel_wiki/blob/main/%E8%AE%BA%E6%96%87/%E3%80%8AIntroduction%20to%20the%20Linux%20kernel%20challenges%20and%20case%20studies%E3%80%8B.pdf)|《介绍Linux内核的挑战和案例研究》| 马德里康普顿斯大学
21|[《Kernel Mode Linux Toward an Operating System Protected by a Type Theory》](https://github.com/0voice/linux_kernel_wiki/blob/main/%E8%AE%BA%E6%96%87/%E3%80%8AKernel%20Mode%20Linux%20Toward%20an%20Operating%20System%20Protected%20by%20a%20Type%20Theory%E3%80%8B.pdf)|《面向类型理论保护的操作系统的内核模式Linux》| 东京大学
22|[《Linux Kernel development》](https://github.com/0voice/linux_kernel_wiki/blob/main/%E8%AE%BA%E6%96%87/%E3%80%8ALinux%20Kernel%20development%E3%80%8B.pdf)|《Linux内核开发》|怀卡托大学 
23|[《Linux Kernel Transport Layer Security》](https://github.com/0voice/linux_kernel_wiki/blob/main/%E8%AE%BA%E6%96%87/%E3%80%8ALinux%20Kernel%20Transport%20Layer%20Security%E3%80%8B.pdf)|《Linux内核传输层安全》| Facebook
24|[《Linux kernel vulnerabilities State-of-the-art defenses and open problems》](https://github.com/0voice/linux_kernel_wiki/blob/main/%E8%AE%BA%E6%96%87/%E3%80%8ALinux%20kernel%20vulnerabilities%20State-of-the-art%20defenses%20and%20open%20problems%E3%80%8B.pdf)|《Linux内核漏洞最新的防御和开放问题》| 麻省理工学院
25|[《Linux Kernel Workshop Hacking the Kernel for Fun and Profit》](https://github.com/0voice/linux_kernel_wiki/blob/main/%E8%AE%BA%E6%96%87/%E3%80%8ALinux%20Kernel%20Workshop%20Hacking%20the%20Kernel%20for%20Fun%20and%20Profit%E3%80%8B.pdf)|《Linux内核研讨会黑客内核的乐趣和利润》| IBM
26|[《Linux Network Device Drivers an Overview》](https://github.com/0voice/linux_kernel_wiki/blob/main/%E8%AE%BA%E6%96%87/%E3%80%8ALinux%20Network%20Device%20Drivers%20an%20Overview%E3%80%8B.pdf)|《Linux网络设备驱动程序概述》| 印度理工学院
27|[《Linux Physical Memory Page Allocation》](https://github.com/0voice/linux_kernel_wiki/blob/main/%E8%AE%BA%E6%96%87/%E3%80%8ALinux%20Physical%20Memory%20Page%20Allocation%E3%80%8B.pdf)|《Linux物理内存页分配》| 未知
28|[《Linux Random Number Generator – A New Approach》](https://github.com/0voice/linux_kernel_wiki/blob/main/%E8%AE%BA%E6%96%87/%E3%80%8ALinux%20Random%20Number%20Generator%20%E2%80%93%20A%20New%20Approach%E3%80%8B.pdf)|《Linux随机数生成器-一种新的方法》| 未知
29|[《Machine Learning for Load Balancing in the Linux Kernel》](https://github.com/0voice/linux_kernel_wiki/blob/main/%E8%AE%BA%E6%96%87/%E3%80%8AMachine%20Learning%20for%20Load%20Balancing%20in%20the%20Linux%20Kernel%E3%80%8B.pdf)|《Linux内核负载平衡的机器学习》| 伊利诺伊大学
30|[《Performance of IPv6 Segment Routing in Linux Kernel》](https://github.com/0voice/linux_kernel_wiki/blob/main/%E8%AE%BA%E6%96%87/%E3%80%8APerformance%20of%20IPv6%20Segment%20Routing%20in%20Linux%20Kernel%E3%80%8B.pdf)|《Linux内核下的IPv6段路由性能》| Cisco
31|[《Professional Linux® Kernel Architecture》](https://github.com/0voice/linux_kernel_wiki/blob/main/%E8%AE%BA%E6%96%87/%E3%80%8AProfessional%20Linux%C2%AE%20Kernel%20Architecture%E3%80%8B.pdf)|《专业的Linux®内核架构》| Wolfgang Mauerer
32|[《Research of Performance Linux Kernel File Systems》](https://github.com/0voice/linux_kernel_wiki/blob/main/%E8%AE%BA%E6%96%87/%E3%80%8AResearch%20of%20Performance%20Linux%20Kernel%20File%20Systems%E3%80%8B.pdf)|《高性能Linux内核文件系统的研究》| 乌拉尔国立技术大学
33|[《Resource Management for Hardware Accelerated Linux Kernel Network Functions》](https://github.com/0voice/linux_kernel_wiki/blob/main/%E8%AE%BA%E6%96%87/%E3%80%8AResource%20Management%20for%20Hardware%20Accelerated%20Linux%20Kernel%20Network%20Functions%E3%80%8B.pdf)|《硬件加速Linux内核网络功能的资源管理》| 积云网络
34|[《Rethinking Compiler Optimizations for the Linux Kernel An Explorative Study》](https://github.com/0voice/linux_kernel_wiki/blob/main/%E8%AE%BA%E6%96%87/%E3%80%8ARethinking%20Compiler%20Optimizations%20for%20the%20Linux%20Kernel%20An%20Explorative%20Study%E3%80%8B.pdf)|《重新思考Linux内核的编译器优化——探索性研究》| 北京大学电子工程与计算机学院
35|[《Securing the Linux Boot Process From Start to Finish》](https://github.com/0voice/linux_kernel_wiki/blob/main/%E8%AE%BA%E6%96%87/%E3%80%8ASecuring%20the%20Linux%20Boot%20Process%20From%20Start%20to%20Finish%E3%80%8B.pdf)|《从开始到结束保护Linux引导过程》| 奥地利波尔顿应用科学大学
36|[《Security Applications of Extended BPF Under the Linux Kernel》](https://github.com/0voice/linux_kernel_wiki/blob/main/%E8%AE%BA%E6%96%87/%E3%80%8ASecurity%20Applications%20of%20Extended%20BPF%20Under%20the%20Linux%20Kernel%E3%80%8B.pdf)|《Linux内核下扩展BPF的安全应用》| 卡尔顿大学计算机科学学院
37|[《Simple and precise static analysis of untrusted Linux kernel extensions》](https://github.com/0voice/linux_kernel_wiki/blob/main/%E8%AE%BA%E6%96%87/%E3%80%8ASimple%20and%20precise%20static%20analysis%20of%20untrusted%20Linux%20kernel%20extensions%E3%80%8B.pdf)|《简单和精确的静态分析不可信的Linux内核扩展》| VMware
38|[《Speeding Up Linux Disk Encryption》](https://github.com/0voice/linux_kernel_wiki/blob/main/%E8%AE%BA%E6%96%87/%E3%80%8ASpeeding%20Up%20Linux%20Disk%20Encryption%E3%80%8B.pdf)|《加快Linux磁盘加密》| 未知
39|[《Stateless model checking of the Linux kernel's hierarchical read-copy-update (tree RCU)》](https://github.com/0voice/linux_kernel_wiki/blob/main/%E8%AE%BA%E6%96%87/%E3%80%8AStateless%20model%20checking%20of%20the%20Linux%20kernel's%20hierarchical%20read-copy-update%20(tree%20RCU)%E3%80%8B.pdf)|《Linux内核的分层read-copy-update (tree RCU)的无状态模型检查》| 雅典国立技术大学
40|[《Survey Paper on Adding System Call in Linux Kernel 3.2+ & 3.16》](https://github.com/0voice/linux_kernel_wiki/blob/main/%E8%AE%BA%E6%96%87/%E3%80%8ASurvey%20Paper%20on%20Adding%20System%20Call%20in%20Linux%20Kernel%203.2%2B%20%26%203.16%E3%80%8B.pdf)|《关于在Linux内核3.2+ & 3.16中添加系统调用的调查报告》| 北阿坎德邦技术大学Shivalik工程学院计算机科学与工程
41|[《The benefits and costs of writing a POSIX kernel in a high-level language》](https://github.com/0voice/linux_kernel_wiki/blob/main/%E8%AE%BA%E6%96%87/%E3%80%8AThe%20benefits%20and%20costs%20of%20writing%20a%20POSIX%20kernel%20in%20a%20high-level%20language%E3%80%8B.pdf)|《用高级语言编写POSIX内核的好处和成本》| 麻省理工学院
42|[《The real-time Linux kernel a Survey on PREEMPT_RT》](https://github.com/0voice/linux_kernel_wiki/blob/main/%E8%AE%BA%E6%96%87/%E3%80%8AThe%20real-time%20Linux%20kernel%20a%20Survey%20on%20PREEMPT_RT%E3%80%8B.pdf)|《实时Linux内核PREEMPT_RT综述》| 米兰理工大学
43|[《Trace-Based Analysis of Locking in the Linux Kernel》](https://github.com/0voice/linux_kernel_wiki/blob/main/%E8%AE%BA%E6%96%87/%E3%80%8ATrace-Based%20Analysis%20of%20Locking%20in%20the%20Linux%20Kernel%E3%80%8B.pdf)|《基于跟踪的Linux内核锁定分析》| 奥斯纳布吕克大学
44|[《Tracing Network Packets in the Linux Kernel using eBPF》](https://github.com/0voice/linux_kernel_wiki/blob/main/%E8%AE%BA%E6%96%87/%E3%80%8ATracing%20Network%20Packets%20in%20the%20Linux%20Kernel%20using%20eBPF%E3%80%8B.pdf)|《使用eBPF跟踪Linux内核中的网络包》| 圣彼得堡国立大学
45|[《TRACING THE WAY OF DATA IN A TCP CONNECTION THROUGH THE LINUX KERNEL》](https://github.com/0voice/linux_kernel_wiki/blob/main/%E8%AE%BA%E6%96%87/%E3%80%8ATRACING%20THE%20WAY%20OF%20DATA%20IN%20A%20TCP%20CONNECTION%20THROUGH%20THE%20LINUX%20KERNEL%E3%80%8B.pdf)|《通过Linux内核跟踪TCP连接中的数据方式》| 加纳马尼理工学院
46|[《Understanding Linux Kernel》](https://github.com/0voice/linux_kernel_wiki/blob/main/%E8%AE%BA%E6%96%87/%E3%80%8AUnderstanding%20Linux%20Kernel%E3%80%8B.pdf)|《了解Linux内核》| 未知
47|[《Understanding Linux Malware》](https://github.com/0voice/linux_kernel_wiki/blob/main/%E8%AE%BA%E6%96%87/%E3%80%8AUnderstanding%20Linux%20Malware%E3%80%8B.pdf)|《了解Linux的恶意软件》| Cisco
48|[《Using kAFS on Linux for Network Home Directories》](https://github.com/0voice/linux_kernel_wiki/blob/main/%E8%AE%BA%E6%96%87/%E3%80%8AUsing%20kAFS%20on%20Linux%20for%20Network%20Home%20Directories%E3%80%8B.pdf)|《在Linux上使用kfs用于网络主目录》| 密歇根大学
49|[《Verification of tree-based hierarchical read-copy update in the Linux kernel》](https://github.com/0voice/linux_kernel_wiki/blob/main/%E8%AE%BA%E6%96%87/%E3%80%8AVerification%20of%20tree-based%20hierarchical%20read-copy%20update%20in%20the%20Linux%20kernel%E3%80%8B.pdf)|《Linux内核中基于树的分层读拷贝更新的验证》| 牛津大学
50|[《XDP test suite for Linux kernel》](https://github.com/0voice/linux_kernel_wiki/blob/main/%E8%AE%BA%E6%96%87/%E3%80%8AXDP%20test%20suite%20for%20Linux%20kernel%E3%80%8B.pdf)|《Linux内核的XDP测试套件》| 马萨里克大学
	
</div>
<br>

<h2 id="7">视频</h2>
待更新，敬请关注。。。
<br>

<h2 id="8">面试题</h2>

**面试题一**

1、什么是Linux?<br>
2、Unix和Linux有什么区别？<br>
3、什么是 Linux 内核？<br>
4、Linux的基本组件是什么？<br>
5、Linux 的体系结构?<br>
6、BASH和DOS之间的基本区别是什么？<br>
7、Linux 开机启动过程？<br>
8、Linux系统缺省的运行级别？<br>
9、Linux 使用的进程间通信方式？<br>
10、Linux 有哪些系统日志文件？<br>
11、Linux系统安装多个桌面环境有帮助吗？<br>
12、什么是交换空间？<br>
13、什么是root帐户?<br>
14、什么是LILO？<br>
15、什么是BASH？<br>
16、什么是CLI？<br>
17、什么是GUI？<br>
18、开源的优势是什么？<br>
19、简单 Linux 文件系统？<br>
20、Linux 的目录结构是怎样的？<br>
21、什么是 inode ？<br>
22、什么是硬链接和软链接？<br>
23、RAID 是什么?<br>
24、一台 Linux 系统初始化环境后需要做一些什么安全工作？<br>
25、什么叫 CC 攻击？什么叫 DDOS 攻击？<br>
26、什么是网站数据库注入？<br>
27、Shell 脚本是什么？<br>
28、可以在 Shell 脚本中使用哪些类型的变量？<br>
29、Shell 脚本中 `if` 语法如何嵌套?<br>
30、Shell 脚本中 `case` 语句的语法?<br>
31、Shell 脚本中 `for` 循环语法？<br>
32、Shell 脚本中 `while` 循环语法？<br>
33、如何使脚本可执行?<br>
34、在 Shell 脚本如何定义函数呢？<br>
35、判断一文件是不是字符设备文件，如果是将其拷贝到 `/dev` 目录下？<br>
36、添加一个新组为 class1 ，然后添加属于这个组的 30 个用户，用户名的形式为 stdxx ，其中 xx 从 01 到 30 ？<br>
37、写一个 sed 命令，修改 `/tmp/input.txt` 文件的内容？<br>
38、用户进程间通信主要哪几种方式?<br>
39、通过伙伴系统申请内核内存的函数有哪些?<br>
40、Linux 虚拟文件系统的关键数据结构有哪些?(至少写出四个)<br>
41、对文件或设备的操作函数保存在那个数据结构中?<br>
42、Linux 中的文件包括哪些?<br>
43、创建进程的系统调用有那些?<br>
44、调用 schedule()进行进程切换的方式有几种?<br>
45、Linux 调度程序是根据进程的动态优先级还是静态优先级来调度进程的?<br>
46、进程调度的核心数据结构是哪个?<br>
47、如何加载、卸载一个模块?<br>
48、模块和应用程序分别运行在什么空间?<br>
49、Linux 中的浮点运算由应用程序实现还是内核实现?<br>
50、模块程序能否使用可链接的库函数?<br>
51、TLB 中缓存的是什么内容?<br>
52、Linux 中有哪几种设备?<br>
53、字符设备驱动程序的关键数据结构是哪个?<br>
54、设备驱动程序包括哪些功能函数?<br>
55、如何唯一标识一个设备?<br>
56、Linux 通过什么方式实现系统调用?<br>
57、Linux 软中断和工作队列的作用是什么?<br>
58、Linux开机启动过程？ <br>
59、Linux系统缺省的运行级别 <br>
60、Linux系统是由那些部分组成？ <br>
61、硬链接和软链接有什么区别？ <br>
62、如何规划一台Linux主机，步骤是怎样？ <br>
63、查看系统当前进程连接数？ <br>
64、如何在/usr目录下找出大小超过10MB的文件? <br>
65、添加一条到192.168.3.0/24的路由，网关为192.168.1.254？ <br>
66、如何在/var目录下找出90天之内未被访问过的文件? <br>
67、如何在/home目录下找出120天之前被修改过的文件? <br>
68、在整个目录树下查找文件“core”，如发现则无需提示直接删除它们。 <br>
69、有一普通用户想在每周日凌晨零点零分定期备份/user/backup到/tmp目录下，该用户应如何做? <br>
70、每周一下午三点将/tmp/logs目录下面的后缀为*.log的所有文件rsync同步到备份服务器192.168.1.100中同样的目录下面，crontab配置项该如何写？<br>
71、找到/tmp/目录下面的所有名称以"_s1.jpg"结尾的普通文件，如果其修改日期在一天内，则将其打包到/tmp/back.tar.gz文件中 <br>
72、配置mysql服务器的时候，配置了auto_increment_increment=3，请问这里的3意味着什么？<br>
73、详细说明keepalived的故障切换工作原理<br>

<br>

<h2 id="9">内核项目</h2>

<div align=center>

No.|Project|Introduction|
:-------: | :---------------: | :------------:
1 | [esp8089](https://github.com/al177/esp8089) | ESP8089 WiFi芯片的Linux内核模块驱动程序
2 | [fibdrv](https://github.com/sysprog21/fibdrv) | 计算斐波那契数列的Linux内核模块
3 | [exfat-linux](https://github.com/arter97/exfat-linux) | 这个用于Linux内核的exFAT文件系统模块是三星最新的Linux主线的exFAT驱动程序的后端端口。这个项目可以用于日常Linux用户，只需简单地做make && make install。Ubuntu用户可以简单地添加一个PPA并开始使用它，甚至不需要下载代码。这也可以直接插入到现有的Linux内核源代码中，以内联地构建文件系统驱动程序，这对Android内核开发人员应该很有用。
4 | [ipt-netflow](https://github.com/aabc/ipt-netflow) | 适用于 Linux 的高性能 NetFlow v5、v9、IPFIX 流数据导出模块核心。创建用于高吞吐量网络中的 linux 路由器。
5 | [buildKernelAndModules](https://github.com/JetsonHacksNano/buildKernelAndModules) | 在NVIDIA Jetson Nano Developer Kit上构建Linux内核和模块
6 | [kernel-modules-hook](https://github.com/saber-nyan/kernel-modules-hook) | 使Arch Linux在内核升级后完全功能		
7 | [rfm12b-linux](https://github.com/gkaindl/rfm12b-linux) | HopeRF公司生产的RFM12B和RFM69CW数字射频模块的Linux内核驱动程序。它的目标是提供SPI接口的嵌入式Linux板。
8 | [khttpd](https://github.com/sysprog21/khttpd) | khttpd是一个实验性的HTTP服务器，实现为Linux内核模块。服务器默认端口为8081，但是可以使用命令行参数port=?当您准备加载内核模块时。
9 | [Kernel_Rootkit](https://github.com/varshapaidi/Kernel_Rootkit) | Kernel_Rootkit是一种特殊类型的恶意软件，它通过修改操作系统内核来向用户和系统管理员隐藏自己的存在。rootkit是一个内核模块——一个动态加载到内核中的库。		
10 | [ktls](https://github.com/djwatson/ktls) | Linux内核传输层安全模块 	
11 | [frdev](https://github.com/hnes/frdev) | 一个高效的ip黑/白名单防火墙(作为一个linux内核模块)。		
12 | [HomaModule](https://github.com/PlatformLab/HomaModule) | 一个实现了Homa传输协议的Linux内核模块。 		
13 | [PCRE](https://github.com/smcho-kr/kpcre) | Linux内核模块&PCRE文本搜索引擎		
14 | [acpi_call](https://github.com/mkottman/acpi_call) | 一个内核简单模块，允许您通过将方法名称后跟参数写入/proc/ ACPI /call来调用ACPI方法。
13 | [Linux Modules](https://github.com/DianaNites/linux_modules) | 这是一个管理Linux内核模块的工具。它是modprobe的一个替代方案，支持列出、添加和删除模块，以及显示模块上的信息。	
14 | [LiME](https://github.com/504ensicsLabs/LiME) | 一个可加载内核模块(LKM)，它允许从Linux和基于Linux的设备(如Android)获取易失性内存。这使得LiME独一无二，因为它是Android设备上第一个允许全内存捕获的工具。它还在获取过程中最小化了用户和内核空间进程之间的交互，这使得它能够生成比其他为Linux内存获取而设计的工具更可靠的内存捕获。		
15 | [kplugs](https://github.com/avielw/kplugs) | KPlugs是一个Linux内核模块，它提供了在Linux内核中动态执行脚本的接口。	
16 | [rapiddisk](https://github.com/pkoutoupis/rapiddisk) | 一个高级Linux RAM驱动器和缓存内核模块。动态分配RAM作为块设备。使用它们作为独立的磁盘驱动器，甚至将它们映射为缓存节点到较慢的本地磁盘驱动器。		
17 | [forge_socket](https://github.com/ewust/forge_socket) | Linux内核模块，用于从用户空间检查/修改TCP套接字状态
18 | [CCKiller](https://github.com/jagerzhang/CCKiller) | Linux轻量级CC攻击防御工具脚本
19 | [libNetGo](https://github.com/gotoolkits/libnetgo) | Linux网络分析工具 
20 | [wgcloud](https://github.com/tianshiyeben/wgcloud) | linux运维监控工具，支持系统信息，内存，cpu，温度，磁盘空间及IO，硬盘smart，系统负载，网络流量，进程等监控，API接口，大屏展示，拓扑图，端口监控，docker监控，日志文件监控，数据可视化，webSSH工具，堡垒机(跳板机)
21 | [hookso](https://github.com/esrrhs/hookso) | hookso是一个Linux动态链接库的注入修改查找工具，用来修改其他进程的动态链接库行为。
22 | [LinuxPerformanceTools](https://github.com/melin/LinuxPerformanceTools) | Linux性能监控工具
23 | [jon](https://github.com/JonGates/jon#jon-version-01) | jon 是一款Linux系统攻防工具箱，包含扫描，入侵，痕迹清理，木马，网站测试等各种黑客工具。
24 | [perf-tools](https://github.com/brendangregg/perf-tools) | 用于Linux ftrace和perf_events(也就是“perf”命令)的各种开发中且不受支持的性能分析工具。ftrace和perf都是核心的Linux跟踪工具，包含在内核源代码中。您的系统可能已经有了ftrace，而perf通常只是一个添加包(参见先决条件)。
25 | [FlameGraph](https://github.com/brendangregg/FlameGraph) | 火焰图形可视化分析器
26 | [bcc](https://github.com/brendangregg/bcc) | BCC 是一个用于创建高效内核跟踪和操作程序的工具包，包括几个有用的工具和示例。它利用了扩展 BPF（伯克利数据包过滤器），正式名称为 eBPF，这是首次添加到 Linux 3.15 的新功能。BCC 使用的大部分内容都需要 Linux 4.1 及更高版本。
27 | [fhe-toolkit-linux](https://github.com/IBM/fhe-toolkit-linux) | IBM Linux的完全同态加密(FHE)工具包被打包为Docker容器，这使得开始和试验完全同态加密技术变得更容易。
28 | [linux-smart-enumeration](https://github.com/diego-treitos/linux-smart-enumeration) | 用于渗透测试的Linux枚举工具和具有冗长级别的ctf
29 | [gpu-monitoring-tools](https://github.com/NVIDIA/gpu-monitoring-tools) | Linux上监视NVIDIA gpu的工具
30 | [linux-inject](https://github.com/gaffe23/linux-inject) | 将共享对象注入Linux进程的工具
31 | [ntttcp-for-linux](https://github.com/microsoft/ntttcp-for-linux) | 一个Linux网络吞吐量多线程基准测试工具
32 | [linux-pentest](https://github.com/ankh2054/linux-pentest) | Linux穿透测试工具

</div>

<br>

<h1 id="5">鸣谢</h1>


