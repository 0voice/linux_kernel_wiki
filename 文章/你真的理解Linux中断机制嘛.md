Linux中断是指在CPU正常运行期间，由于内外部事件或由程序预先安排的事件引起的CPU暂时停止正在运行的程序，转而为该内部或外部事件或预先安排的事件服务的程序中去，服务完毕后再返回去继续运行被暂时中断的程序。

进程的不可中断状态是系统的一种保护机制，可以保证硬件的交互过程不被意外打断。所以，短时间的不可中断状态是很正常的。但是，当进程长时间都处于不可中断状态时，你就需要提起注意力确认下是不是磁盘I/O存在问题，相关的进程和磁盘设备是否工作正常。

今天我们详细了解一下中断的机制，进而对其中的软中断进行一个剖析。

## 概念解释

（1）中断：是一种异步的事件处理机制，可以提高系统的并发处理能力。

（2）如何解决中断处理程序执行过长和中断丢失的问题：<br>
Linux 将中断处理过程分成了两个阶段，也就是上半部和下半部。<br>
上半部用来快速处理中断，它在中断禁止模式下运行，主要处理跟硬件紧密相关的或时间敏感的工作。也就是我们常说的硬中断，特点是快速执行。<br>
下半部用来延迟处理上半部未完成的工作，通常以内核线程的方式运行。也就是我们常说的软中断，特点是延迟执行。

（3）proc 文件系统：是一种内核空间和用户空间进行通信的机制，可以用来查看内核的数据结构，或者用来动态修改内核的配置。<br>
/proc/softirqs 提供了软中断的运行情况；<br>
/proc/interrupts 提供了硬中断的运行情况。<br>

（4）硬中断：硬中断是由硬件产生的，比如，像磁盘，网卡，键盘，时钟等。每个设备或设备集都有它自己的IRQ（中断请求）。基于IRQ，CPU可以将相应的请求分发到对应的硬件驱动上。硬中断可以直接中断CPU，引起内核中相关的代码被触发。

（5）软中断：软中断仅与内核相关，由当前正在运行的进程所产生。 通常，软中断是一些对I/O的请求，这些请求会调用内核中可以调度I/O发生的程序。 软中断并不会直接中断CPU，也只有当前正在运行的代码（或进程）才会产生软中断。这种中断是一种需要内核为正在运行的进程去做一些事情（通常为I/O）的请求。
除了iowait(等待I/O的CPU使用率)升高，软中断(softirq)CPU使用率升高也是最常见的一种性能问题。

## 查看软中断和内核线程

小伙伴们肯定好奇该怎么查看系统里有哪些软中断？接下来将教给大家方法。<br>
前面有提到过proc文件系统，它是一种内核空间和用户空间进行通信的机制， 可以用来查看内核的数据结构，或者用来动态修改内核的配置。

* /proc/softirqs 提供了软中断的运行情况；
* /proc/interrupts 提供了硬中断的运行情况。

（1）如何查看各种类型软中断在不同 CPU上的累积运行次数：

```
$ cat /proc/softirqs
                    CPU0       CPU1       CPU2       CPU3
          HI:     276180     286764    2509097     254357
       TIMER:    1550133    1285854    1440533    1812909
      NET_TX:     102895         16         15         57
      NET_RX:        155        178        115    1619192
       BLOCK:       1713      15048     251826       1082
    IRQ_POLL:          0          0          0          0
     TASKLET:          9         63          6       2830
       SCHED:    1484942    1207449    1310735    1724911
     HRTIMER:          0          0          0          0
         RCU:     690954     685825     787447     878963

```

软中断的类型：对应第1列，包含了10个类别，分别对应不同的工作类型。比如说NET_RX 表示网络接收中断，而 NET_TX 表示网络发送中 断。

同一种软中断类型在不同CPU上的分布情况：对应每一行，正常情况下，同一种中断类型在不同CPU上的累计次数基本在同一个数量级。但是也有例外，比如TASKLET

拓展：什么是TASKLET？<br>
* TASKLET是最常用的软中断实现机制，每个TASKLET只会运行一次就会结束，并且只在调用它的函数所在的CPU上运行，不能并行而只能串行执行。
* 多个不同类型的TASKLET可以并行在多个CPU上。
* 软中断是静态，只能支持有限的几种软中断类型，一旦内核编译好之后就不能改变；而TASKLET灵活很多，可以通过添加内核模块的方式在运行时修改。


（2）如何查看软中断内核线程的运行状况？<br>
软中断是以内核线程的方式运行的，每个CPU都会对应一个软中断内核线程，查看的方式如下：

```
$ ps -ef|grep softirq
root    7    2  0 Nov04 ?    00:00:53 [ksoftirqd/0]
root   16    2  0 Nov04 ?    00:00:51 [ksoftirqd/1]
root   22    2  0 Nov04 ?    00:00:53 [ksoftirqd/2]
root   28    2  0 Nov04 ?    00:00:53 [ksoftirqd/3]
```

这些线程的名字外面都有中括号，这说明 ps 无法获取它们的命令行参数 （cmline）。

一般来说，ps 的输出中，名字括在中括号里的，一般都是内核线程。

（3）如何查看硬中断运行情况

```
$ cat /proc/interrupts 
           CPU0       CPU1       CPU2       CPU3            
  0:         33          0          0          0      IO-APIC-edge      timer
  1:         10          0          0          0      IO-APIC-edge      i8042
  4:        325          0          0          0      IO-APIC-edge      serial
  8:          1          0          0          0      IO-APIC-edge      rtc0
  9:          0          0          0          0      IO-APIC-fasteoi   acpi
 10:          0          0          0          0      IO-APIC-fasteoi   virtio3
 40:          0          0          0          0      PCI-MSI-edge      virtio1-config
 41:   16669006          0          0          0      PCI-MSI-edge      virtio1-requests
 42:          0          0          0          0      PCI-MSI-edge      virtio2-config
 43:   59166530          0          0          0      PCI-MSI-edge      virtio2-requests
 44:          0          0          0          0      PCI-MSI-edge      virtio0-config
 45:    6689988          0          0          0      PCI-MSI-edge      virtio0-input.0
 46:          0          0          0          0      PCI-MSI-edge      virtio0-output.0
 47: 2093616484          0          0          0      PCI-MSI-edge      peth1-TxRx-0
 48:          5 2045859720          0          0      PCI-MSI-edge      peth1-TxRx-1
 49:         81          0          0          0      PCI-MSI-edge      peth1
NMI:          0          0          0          0      Non-maskable interrupts
LOC: 2936184495  965056330 1641503935 1442909354      Local timer interrupts
SPU:          0          0          0          0      Spurious interrupts
PMI:          0          0          0          0      Performance monitoring interrupts
IWI:   53775871   47387196   47737572   44243915      IRQ work interrupts
RTR:          0          0          0          0      APIC ICR read retries
RES: 1198594562  964481221  966552350  902484234      Rescheduling interrupts
CAL: 4294967071       4438  430547422  419910155      Function call interrupts
TLB: 1206563963   65932469 1378887038 1028081848      TLB shootdowns
TRM:          0          0          0          0      Thermal event interrupts
THR:          0          0          0          0      Threshold APIC interrupts
MCE:          0          0          0          0      Machine check exceptions
MCP:      65623      65623      65623      65623      Machine check polls
ERR:          0
```

## 工具与技巧

![image](https://user-images.githubusercontent.com/87457873/127655271-080f7e53-f2b0-4797-bbbc-972f77afb8a0.png)


（1）sar 是一个系统活动报告工具，既可以实时查看系统的当前活动，又可以配置保存和报告历史统计数据。<br>
命令：sar -n DEV 1<br>
含义：-n DEV 1 表示显示网络收发的报告，间隔1秒输出一组数据<br>

```
$ sar -n DEV 1
16:01:21       IFACE   rxpck/s   txpck/s    rxkB/s    txkB/s   rxcmp/s   txcmp/s  rxmcst/s   %ifutil
16:01:22        eth0  12605.00   6304.00    664.86    358.11      0.00      0.00      0.00      0.01
16:01:22     docker0   6302.00  12604.00    270.79    664.66      0.00      0.00      0.00      0.00
16:01:22          lo      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00
16:01:22 veth9f6bbcd   6302.00  12604.00    356.95    664.66      0.00      0.00      0.00      0.05
```

第1列：表示报告的时间

第2列：IFACE 表示网卡

第3，4列：rxpck/s 和 txpck/s 分别表示每秒接收、发送的网络帧数，也就是 PPS

第5，6列：rxkB/s 和 txkB/s 分别表示每秒接收、发送的千字节数，也就是 BPS。

（2）tcpdump 是一个常用的网络抓包工具，常用来分析各种网络问题。

命令：tcpdump -i eth0 -n tcp port 80<br>
含义：-i eth0 只抓取eth0网卡，-n不解析协议名和主机名<br>
           tcp port 80表示只抓取tcp协议并且端口号为80的网络帧

