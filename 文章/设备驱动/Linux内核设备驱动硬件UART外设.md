今天给大家分享的是**IAR下调试信息输出机制之硬件UART外设**。

在嵌入式世界里，输出打印信息是一种非常常用的辅助调试手段，借助打印信息，我们可以比较容易地定位和分析程序问题。在嵌入式应用设计里实现打印信息输出的方式有很多，本系列将以 IAR 环境为例逐一介绍 ARM Cortex-M 内核 MCU 下打印信息输出方法。

本篇是第一篇，我们先介绍最常见的输出打印信息方式，即利用 MCU 芯片内的硬件 UART 外设。本篇其实并不是要具体介绍 UART 外设模块使用方法，而是重点分析 IAR 下是如何联系 C 标准头文件 stdio.h 定义的 printf() 函数与 UART 外设底层驱动函数的。

> Note：本文使用的 IAR EWARM 软件版本是 v9.10.2。

## **1、打印输出整体框图**

首先，介绍一下打印输出方法整体软硬件框图，硬件上主要是 PC 主机、MCU 目标板、一根连接线（连接线有两种方案：一种是 RS232 串口线、另一种是 TTL 串口转 USB 模块板）。

软件上 PC 这边需要安装一个串口调试助手软件，然后目标板 MCU 应用程序需要包含打印输出相关代码，当 MCU 程序运行起来后，驱动片内 UART 外设实现打印字符数据 (hello world.) 物理传输，在 PC 上串口调试助手软件里可以看到打印信息。

![img](https://pic3.zhimg.com/80/v2-79bec5c17ea37214f212e701451ab552_720w.webp)

上图里的 MCU 应用程序是在 IAR 环境下编译链接的，因此我们的重点就是 stdio.h 头文件里的 printf() 在 IAR 下到底是如何与 UART 外设驱动函数“勾搭”起来的。

## **2、C 标准头文件 stdio.h**

熟悉嵌入式工程的朋友应该都知道 stdio.h 头文件并不在用户工程文件夹里，无需我们手动添加该文件进工程目录，该文件是 C 标准定义的头文件，由工具链自动提供。

stdio.h 是 C 语言为输入输出提供的标准库头文件，其前身是迈克·莱斯克 20 世纪 70 年代编写的“可移植输入输出程序库”。C 语言中的所有输入和输出都由抽象的字节流来完成，对文件的访问也通过关联的输入或输出流进行。

> stdio.h 原型：[https://cplusplus.com/reference/cstdio/](https://link.zhihu.com/?target=https%3A//cplusplus.com/reference/cstdio/)



大部分人学 C 语言一般都是在 Visual Studio / C++ 环境下，在这个环境里 stdio.h 定义的那些函数底层实现都由 Visual Studio 软件直接搞定，我们通常无需关心其实现细节。

在嵌入式 IAR 环境下，这些标准 C 定义的头文件大部分也都是可以被支持的，我们可以在如下 IAR 软件目录找到它们，当我们在工程代码里加入 #include <stdio.h> 等语句时，实际上就是添加 IAR 软件目录里的文件进工程编译。

```text
\IAR Systems\Embedded Workbench 9.10.2\arm\inc\c\stdio.h
```

但是 IAR 目录下 stdio.h 文件里定义的诸如 printf() 函数具体实现我们是需要关注的，毕竟是要编译链接生成具体机器码下载进 MCU 运行的，但是 printf() 函数原型在哪呢？我们先留个悬念。

## **3、UART 外设驱动函数**

说到 UART 外设驱动函数，这个大家应该再熟悉不过了。我们以恩智浦 i.MXRT1060 型号（ARM Cortex-M7 内核）为例来具体介绍，在其官方 SDK 包里有相应的 LPUART 驱动文件：

```text
\SDK_2.11.0_EVK-MIMXRT1060\devices\MIMXRT1062\drivers\fsl_lpuart.h
\SDK_2.11.0_EVK-MIMXRT1060\devices\MIMXRT1062\drivers\fsl_lpuart.c
```

这个 LPUART 驱动库里的 LPUART_WriteBlocking() 和 LPUART_ReadBlocking() 函数可以完成用户数据包的发送和接收，其实单纯利用 LPUART_WriteBlocking() 函数也可以实现打印信息输出，只是没有 printf() 函数那样包含格式化输出的强大功能。

```text
status_t LPUART_Init(LPUART_Type *base, const lpuart_config_t *config, uint32_t srcClock_Hz)
status_t LPUART_WriteBlocking(LPUART_Type *base, const uint8_t *data, size_t length)
status_t LPUART_ReadBlocking(LPUART_Type *base, uint8_t *data, size_t length)
```

## **4、IAR 对 C 标准 I/O 库的支持**

IAR 显然是对 C 标准 I/O 库有支持的，不然我们不可能在工程里能使用 printf() 函数，只是这个支持我们如何去轻松发现呢？痞子衡今天教大家一个方法，就是看工程编译链接后生成的 .map 文件，这个 map 文件里会列出工程里所有函数的来源。

### **4.1引出底层接口 __write()**

我们以 \SDK_2.11.0_EVK-MIMXRT1060\boards\evkmimxrt1060\demo_apps\hello_world\iar 工程为例来介绍，需要简单改造一下工程里 hello_world.c 文件里的 main() 函数，将原来代码全部删掉（原来的打印输出涉及恩智浦 SDK 封装，本文没必要关心其实现），只要如下一句打印即可：

```text
#include <stdio.h>
int main(void)
{
    printf("hello world.\r\n");
    while (1);
}
```

然后注意工程选项里跟 Library 实现相关的如下三处设置。其中 **Library** 选项配置的是 runtime lib 的功能，有 Normal 和 Full 两个选项（可按需选择）；**Printf formatter** 选项决定格式化输出功能细节，分 Full、Large、Small、Tiny 四个选项（可按需选择）。

**Library low-level interface implementation** 选项决定低层 I/O 实现，这里我们选 None，即由用户来实现。

![img](https://pic1.zhimg.com/80/v2-2417b372d998a4473700b9575edc923c_720w.webp)

配置好 Library 后编译工程会发现有如下报错，根据这个报错我们可以猜到 dl7M_tln.a 是 IAR 编译好的 C/C++ 库，库里面实现了 printf() 函数及其所依赖的 putchar() 函数，而 puchar() 函数对外提供了底层 I/O 接口函数，这个 I/O 函数名字叫 __write()，它就是需要用户结合芯片 UART 外设去实现的发送函数。

```text
Error[Li005]: no definition for "__write" [referenced from putchar.o(dl7M_tln.a)]
```

在 IAR 目录下我们可以找到 dl7M_tln.a 文件路径，经过测试，工程 **Library** 设置里 Normal 和 Full 选项其实就是选 dl7M_tln.a 还是 dl7M_tlf.a 进用户工程去链接。

![img](https://pic1.zhimg.com/80/v2-9c0c3c06803d513796a2f701d6209d58_720w.webp)

### **4.2DLIB底层 I/O 接口设计**

找到了 __write() 函数，但是它的原型到底是什么？我们该如何实现它？这时候需要去查万能的 \IAR Systems\Embedded Workbench 9.10.2\arm\doc\EWARM_DevelopmentGuide.ENU 手册，在里面搜索 __write 字样可以找到如下设计，原来我们在代码里调用的 C 标准 I/O 接口均是由 IAR 底层预编译好的 DLIB 去具体实现的，这个 DLIB 库也留下了给用户实现的最底层与硬件相关的接口函数。

![img](https://pic1.zhimg.com/80/v2-ad69a3c14d92c80e6add86cb19d76eb0_720w.webp)

IAR 为 DLIB 里那些最底层的 I/O 接口函数都创建了模板源文件，在这些模板文件里我们可以找到它们的原型，所以我们在 write.c 文件里找到了 __write() 原型及其示例实现。

```text
size_t __write(int handle, const unsigned char * buffer, size_t size)
```

![img](https://pic1.zhimg.com/80/v2-b53770f7edd58593a4f397555096b928_720w.webp)

### **4.3DLIB库 I/O 相关源码实现**

有了 __write() 原型及示例代码，我们很容易便能用 LPUART_WriteBlocking() 函数去实现它，将这个代码添加进 hello_world 工程编译，这时候就不会报错了（当然要想真正在板子上测试打印功能，main 函数里还得加入 LPUART 初始化代码）。

```text
#include "fsl_lpuart.h"
size_t __write(int handle, const unsigned char *buf, size_t size)
{
    // 假设使用 LPUART1 去做输出
    (void)LPUART_WriteBlocking(LPUART1, buf, size);

    return 0;
}
```

工程编译完成后，查看生成的 hello_world.map 文件，找到 dl7M_tln.a 部分的信息，可以看到其由很多个 .o 文件组成（功能比较丰富），这些 .o 文件都是可以在 IAR 安装目录下找到其源码的。

```text
*******************************************************************************
*** MODULE SUMMARY
***

    Module                ro code  ro data  rw data
    ------                -------  -------  -------
dl7M_tln.a: [10]
    abort.o                     6
    exit.o                      4
    low_level_init.o            4
    printf.o                   40
    putchar.o                  32
    xfail_s.o                  64                 4
    xprintfsmall_nomb.o     1'281
    xprout.o                   22
    -----------------------------------------------
    Total:                  1'453                 4
```

DLIB 库中关于 I/O 相关的源码放在了如下目录里，感兴趣的可以去查看其具体实现，这里特别提一下 formatter 文件夹下 xprintf 有很多种不同的源文件实现，其实就对应了工程选项 **Printf formatter** 里的不同配置。

```text
\IAR Systems\Embedded Workbench 9.10.2\arm\src\lib\dlib\file
\IAR Systems\Embedded Workbench 9.10.2\arm\src\lib\dlib\formatters
```

![img](https://pic4.zhimg.com/80/v2-042ef8125c5e923c3f5f185a28967bd7_720w.webp)

至此，IAR下调试信息输出机制之硬件UART外设痞子衡便介绍完毕了。

------

版权声明：本文为知乎博主「Linux内核库」的原创文章，遵循CC 4.0 BY-SA版权协议，转载请附上原文 出处链接及本声明。  

原文链接：https://zhuanlan.zhihu.com/p/586298947

