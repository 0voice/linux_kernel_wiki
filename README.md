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

**文件系统**

<br>

**网络协议栈**

<br>

**设备驱动**
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
待更新，敬请关注。。。
<br>

<h2 id="9">内核项目</h2>
待更新，敬请关注。。。
<br>

<h1 id="5">鸣谢</h1>
