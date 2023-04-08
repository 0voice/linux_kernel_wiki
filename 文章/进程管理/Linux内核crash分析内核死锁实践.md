**上文**讲了[内核死锁的debug方法](https://zhuanlan.zhihu.com/p/530245778)通过lockdep的方式可以debug出死锁的信息，但是如果出问题的系统没有lockdep的配置，或者没有相关的日志该怎么办？这里分享通过crash工具来动态检测死锁时的问题

## 1、**用crash初步分析**

一般卡死时可能是因为核心线程处在UNINTERRUPTIBLE状态，所以先在crash环境下用ps命令查看系统中UNINTERRUPTIBLE状态的线程，参数-u可过滤掉内核线程：



![img](https://pic1.zhimg.com/80/v2-af83b50e3cb6e82e395c446479b405c8_720w.webp)



bt命令可查看某个线程的调用栈，我们看一下上面UN状态的最关键的watchdog线程：



![img](https://pic3.zhimg.com/80/v2-0676823081756009a0ab52c5eadce836_720w.webp)



从调用栈中可以看到proc_pid_cmdline_read()函数中被阻塞的，对应的代码为：

```text
static ssize_t proc_pid_cmdline_read(struct file *file, char __user *buf,
         size_t _count, loff_t *pos)
{
  ......
 tsk = get_proc_task(file_inode(file));
 if (!tsk)
  return -ESRCH;
 mm = get_task_mm(tsk);
 put_task_struct(tsk);
  ......
 down_read(&mm->mmap_sem);
  ......
}
```

这里是要获取被某个线程mm的mmap_sem锁，而这个锁又被另外一个线程持有。

## 2、**推导读写锁**

要想知道哪个线程持有了这把锁，我们得先用汇编推导出这个锁的具体值。可用dis命令看一下proc_pid_cmdline_read()的汇编代码：



![img](https://pic1.zhimg.com/80/v2-618822b148611a685866300712d9e578_720w.webp)



0xffffff99a680aaa0处就是调用down_read()的地方，它的第一个参数x0就是sem锁，如：

```text
void __sched down_read(struct rw_semaphore *sem)
```

x0和x28寄存器存放的就是sem的值，那x21自然就是mm_struct的地址了，因为mm_struct的mmap_sem成员的offset就是104（0x68），用whatis命令可以查看结构体的声明，如：

![img](https://pic4.zhimg.com/80/v2-28b0837d9adbeb4131739c9c8b586f07_720w.webp)

因此我们只需要知道**x21**或者**x28**就知道mm和mmap_sem锁的值。

函数调用时被调用函数会在自己的栈帧中保存即将被修改到的寄存器，所以我们可以在down_read()及它之后的函数调用中找到这两个寄存器：

也就是说下面几个函数中，只要找到用到x21或x28，必然会在它的栈帧中保存这些寄存器。

![img](https://pic3.zhimg.com/80/v2-41d9ab4c760b815cc8928610f87ce80a_720w.webp)

先从最底部的down_read()开始找：



![img](https://pic2.zhimg.com/80/v2-3f0b47735415221a89761ca90982e5f5_720w.webp)



显然它没有用到x21或x28，继续看rwsem_down_read_failed()的汇编代码：



![img](https://pic1.zhimg.com/80/v2-e22079801eb115f51f5f661a56e466f8_720w.webp)



在这个函数中找到x21，它保存在rwsem_down_read_failed栈帧的偏移32字节的位置。rwsem_down_read_failed()的sp是0xffffffd6d9e4bcb0

![img](https://pic1.zhimg.com/80/v2-c16f4d079297c7590b798848d0f7d450_720w.webp)

sp + 32 =0xffffffd6d9e4bcd0，用rd命令查看地址0xffffffd6d9e4bcd0中存放的x21的值为：

![img](https://pic2.zhimg.com/80/v2-f907dbc910c60f04bbe19d62d70cfad9_720w.webp)

用struct命令查看这个mm_struct：

![img](https://pic3.zhimg.com/80/v2-e9df81b47d226780e9de9ed57a3c8402_720w.webp)

这里的owner是mm_struct所属线程的task_struct：

![img](https://pic2.zhimg.com/80/v2-6e4c4c75de9002d5500d7a9ec99efe99_720w.webp)

sem锁的地址为0xffffffd76e349a00+0x68 = **0xffffffd76e349a68**，所以：

![img](https://pic3.zhimg.com/80/v2-9ffa5ba58bda3876216ca0bc7413e0c2_720w.webp)

分析到这里我们知道watchdog线程是在读取1651线程的proc节点时被阻塞了，原因是这个进程的mm，它的mmap_sem锁被其他线程给拿住了，那到底是谁持了这把锁呢？



## **3、持读写锁的线程**

带着问题我们继续分析，通过search命令加-t参数从系统中所有的线程的栈空间里查找当前锁：

![img](https://pic4.zhimg.com/80/v2-32685f147f05f632f329effd8a510923_720w.webp)

一般锁的值都会保存在寄存器中，而寄存器又会在子函数调用过程中保存在栈中。所以只要在栈空间中找到当前锁的值（**0xffffffd76e349a68**），那这个线程很可能就是持锁或者等锁线程

这里搜出的20个线程中19个就是前面提到的等锁线程，剩下的1个很可能就是持锁线程了：

![img](https://pic2.zhimg.com/80/v2-9816f4f7ac4253a82371c004c2a4b621_720w.webp)

查看这个线程的调用栈：



![img](https://pic2.zhimg.com/80/v2-4ac94a68e687757b6ade9e3b1fbf2271_720w.webp)



由于**2124**线程中存放锁的地址是0xffffffd6d396b8b0，这个是在handle_mm_fault()的栈帧范围内，因此可以推断持锁的函数应该是在handle_mm_fault()之前。

我们先看一下do_page_fault函数：

![img](https://pic3.zhimg.com/80/v2-0f6766d9b808edd1dc03ba347b7601d2_720w.webp)

代码中确实是存在持mmap_sem的地方，并且是读者，因此可以确定是**2124**持有的读写锁阻塞了watchdog在内的19个线程。

接下来我们需要看一下**2124**线程为什么会持锁后迟迟不释放就可以了。

## 4、死锁

可以看出2124线程是等待fuse的处理结果，而我们知道fuse的请求是sdcard来处理的。

在log中我们看到的确有sdcard相关的UNINTERRUPTIBLE状态的线程**2767**：



![img](https://pic3.zhimg.com/80/v2-8e77ae4d347f45d482c43afb400bf1a6_720w.webp)



得2767线程等待的mutex锁是0xffffffd6948f4090，

它的owner的task和pid为：



![img](https://pic1.zhimg.com/80/v2-616923a2b415a2306f9765afe108baa0_720w.webp)



先通过bt命令查找2124的栈范围为0xffffffd6d396b4b0～0xffffffd6d396be70：

![img](https://pic1.zhimg.com/80/v2-75e7a8881d999f518bbc62c9e5fd6084_720w.webp)

从栈里面可以找到mutex：

![img](https://pic4.zhimg.com/80/v2-a1de4a1d80ec85e490fbaf680e02e2ff_720w.webp)

mutex值在ffffffd6d396bc40这个地址上找到了，它是在__generic_file_write_iter的栈帧里。



![img](https://pic2.zhimg.com/80/v2-625bd18a344e15f11f707e70b53a9309_720w.webp)



那可以肯定是在__generic_file_write_iter之前就持锁了，并且很可能是ext4_file_write_iter中，查看其源码：

![img](https://pic2.zhimg.com/80/v2-a1a16f2a1eff090dce02df44864b271d_720w.webp)

这下清楚了，原来2124在等待2767处理fuse请求，而2767又被2124线程持有的mutex锁给锁住了，也就是说两个线程互锁了。

------

版权声明：本文为知乎博主「Linux内核库」的原创文章，遵循CC 4.0 BY-SA版权协议，转载请附上原文 出处链接及本声明。

原文链接：https://zhuanlan.zhihu.com/p/530352450
