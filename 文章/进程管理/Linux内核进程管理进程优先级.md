> **前言：**进程优先级实际上是系统对进程重要性的一个客观评价。根据这个评价的结果来为进程分配不同的系统资源，这个资源包括内存资源和CPU资源。为了保证“公平公正”的评价每个进程，Google工程师为此设计了一套评价系统。

![img](https://pic3.zhimg.com/80/v2-ed56495e2c5f289842576ffebae899e2_720w.webp)

## 为什么要有进程优先级？

这似乎不用过多的解释，毕竟自从多任务操作系统诞生以来，进程执行占用cpu的能力就是一个必须要可以人为控制的事情。因为有的进程相对重要，而有的进程则没那么重要。进程优先级起作用的方式从发明以来基本没有什么变化，无论是只有一个cpu的时代，还是多核cpu时代，都是通过控制进程占用cpu时间的长短来实现的。就是说在同一个调度周期中，优先级高的进程占用的时间长些，而优先级低的进程占用的短些。从这个角度看，进程优先级其实也跟cgroup的cpu限制一样，都是一种针对cpu占用的QOS机制。我曾经一直很困惑一点，为什么已经有了优先级，还要再设计一个针对cpu的cgroup？得到的答案大概是因为，优先级这个值不能很直观的反馈出资源分配的比例吧？不过这不重要，实际上从内核目前的进程调度器cfs的角度说，同时实现cpushare方式的cgroup和优先级这两个机制完全是相同的概念，并不会因为增加一个机制而提高什么实现成本。既然如此，而cgroup又显得那么酷，那么何乐而不为呢？

## NICE值

nice值应该是熟悉Linux/UNIX的人很了解的概念了，我们都知它是一个反映一个进程“优先级”状态的值，其取值范围是-20至19，一共40个级别。这个值越小，表示进程”优先级”越高，而值越大“优先级”越低。我们可以通过nice命令来对一个将要执行的命令进行nice值设置，方法是：

```text
[root@zorrozou-pc0 zorro]# nice -n 10 bash
```

这样我就又打开了一个bash，并且其nice值设置为10，而默认情况下，进程的优先级应该是从父进程继承来的，这个值一般是0。我们可以通过nice命令直接查看到当前shell的nice值

```text
[root@zorrozou-pc0 zorro]# nice
```

**对比一下正常情况：**

```cpp
[root@zorrozou-pc0 zorro]# exit
```

**退出当前nice值为10的bash，打开一个正常的bash：**

```cpp
[root@zorrozou-pc0 zorro]# bash
[root@zorrozou-pc0 zorro]# nice
```

另外，使用renice命令可以对一个正在运行的进程进行nice值的调整，我们也可以使用比如top、ps等命令查看进程的nice值，具体方法我就不多说了，大家可以参阅相关manpage。

需要大家注意的是，我在这里都在使用nice值这一称谓，而非优先级（priority）这个说法。当然，nice和renice的man手册中，也说的是priority这个概念，但是要强调一下，请大家真的不要混淆了系统中的这两个概念，一个是nice值，一个是priority值，他们有着千丝万缕的关系，但对于当前的Linux系统来说，它们并不是同一个概念。

**我们看这个命令：**

```cpp
[root@zorrozou-pc0 zorro]# ps -l
F S   UID   PID  PPID  C PRI  NI ADDR SZ WCHAN  TTY          TIME CMD
4 S     0  6924  5776  0  80   0 - 17952 poll_s pts/5    00:00:00 sudo
4 S     0  6925  6924  0  80   0 -  4435 wait   pts/5    00:00:00 bash
0 R     0 12971  6925  0  80   0 -  8514 -      pts/5    00:00:00 ps
```

**大家是否真的明白其中PRI列和NI列的具体含义有什么区别？同样的，如果是top命令：**

```cpp
Tasks: 1587 total,   7 running, 1570 sleeping,   0 stopped,  10 zombie
Cpu(s): 13.0%us,  6.9%sy,  0.0%ni, 78.6%id,  0.0%wa,  0.0%hi,  1.5%si,  0.0%st
Mem:  132256952k total, 107483920k used, 24773032k free,  2264772k buffers
Swap:  2101192k total,      508k used,  2100684k free, 88594404k cached

  PID USER      PR  NI  VIRT  RES  SHR S %CPU %MEM    TIME+  COMMAND                                                                                                                                                                                                                                                                          
 3001 root      20   0  232m  21m 4500 S 12.9  0.0   0:15.09 python                                                                                                                                                                                                                                                                                
11541 root      20   0 17456 2400  888 R  7.4  0.0   0:00.06 top    
```

大家是否搞清楚了这其中PR值和NI值的差别？如果没有，那么我们可以首先搞清楚什么是nice值。

nice值虽然不是priority，但是它确实可以影响进程的优先级。在英语中，如果我们形容一个人nice，那一般说明这个人的人缘比较好。什么样的人人缘好？往往是谦让、有礼貌的人。比如，你跟一个nice的人一起去吃午饭，点了两个一样的饭，先上了一份后，nice的那位一般都会说：“你先吃你先吃！”，这就是人缘好，这人nice！但是如果另一份上的很晚，那么这位nice的人就要饿着了。这说明什么？越nice的人抢占资源的能力就越差，而越不nice的人抢占能力就越强。这就是nice值大小的含义，nice值越低，说明进程越不nice，抢占cpu的能力就越强，优先级就越高。在原来使用O1调度的Linux上，我们还会把nice值叫做静态优先级，这也基本符合nice值的特点，就是当nice值设定好了之后，除非我们用renice去改它，否则它是不变的。而priority的值在之前内核的O1调度器上表现是会变化的，所以也叫做动态优先级。

## 优先级和实时进程

简单了解nice值的概念之后，我们再来看看什么是priority值，就是ps命令中看到的PRI值或者top命令中看到的PR值。本文为了区分这些概念，以后统一用nice值表示NI值，或者叫做静态优先级，也就是用nice和renice命令来调整的优先级；而实用priority值表示PRI和PR值，或者叫动态优先级。我们也统一将“优先级”这个词的概念规定为表示priority值的意思。

在内核中，进程优先级的取值范围是通过一个宏定义的，这个宏的名称是MAX_PRIO，它的值为140。而这个值又是由另外两个值相加组成的，一个是代表nice值取值范围的NICE_WIDTH宏，另一个是代表实时进程（realtime）优先级范围的MAX_RT_PRIO宏。说白了就是，Linux实际上实现了140个优先级范围，取值范围是从0-139，这个值越小，优先级越高。nice值的-20到19，映射到实际的优先级范围是100-139。新产生进程的默认优先级被定义为：

```cpp
#define DEFAULT_PRIO            (MAX_RT_PRIO + NICE_WIDTH / 2)
```

实际上对应的就是nice值的0。正常情况下，任何一个进程的优先级都是这个值，即使我们通过nice和renice命令调整了进程的优先级，它的取值范围也不会超出100-139的范围，除非这个进程是一个实时进程，那么它的优先级取值才会变成0-99这个范围中的一个。这里隐含了一个信息，就是说当前的Linux是一种已经支持实时进程的操作系统。

什么是实时操作系统，我们就不再这里详细解释其含义以及在工业领域的应用了，有兴趣的可以参考一下实时操作系统的维基百科。简单来说，实时操作系统需要保证相关的实时进程在较短的时间内响应，不会有较长的延时，并且要求最小的中断延时和进程切换延时。对于这样的需求，一般的进程调度算法，无论是O1还是CFS都是无法满足的，所以内核在设计的时候，将实时进程单独映射了100个优先级，这些优先级都要高与正常进程的优先级（nice值），而实时进程的调度算法也不同，它们采用更简单的调度算法来减少调度开销。

**总的来说，Linux系统中运行的进程可以分成两类：**

1. **实时进程**
2. **非实时进程**

它们的主要区别就是通过优先级来区分的。所有优先级值在0-99范围内的，都是实时进程，所以这个优先级范围也可以叫做实时进程优先级，而100-139范围内的是非实时进程。在系统中可以使用chrt命令来查看、设置一个进程的实时优先级状态。

**我们可以先来看一下chrt命令的使用：**

```cpp
[root@zorrozou-pc0 zorro]# chrt
Show or change the real-time scheduling attributes of a process.

Set policy:
 chrt [options] <priority> <command> [<arg>...]
 chrt [options] -p <priority> <pid>

Get policy:
 chrt [options] -p <pid>

Policy options:
 -b, --batch          set policy to SCHED_OTHER
 -f, --fifo           set policy to SCHED_FIFO
 -i, --idle           set policy to SCHED_IDLE
 -o, --other          set policy to SCHED_OTHER
 -r, --rr             set policy to SCHED_RR (default)

Scheduling flag:
 -R, --reset-on-fork  set SCHED_RESET_ON_FORK for FIFO or RR

Other options:
 -a, --all-tasks      operate on all the tasks (threads) for a given pid
 -m, --max            show min and max valid priorities
 -p, --pid            operate on existing given pid
 -v, --verbose        display status information

 -h, --help     display this help and exit
 -V, --version  output version information and exit

For more details see chrt(1).
```

我们先来关注显示出的Policy options部分，会发现系统给个种进程提供了5种调度策略。但是这里并没有说明的是，这五种调度策略是分别给两种进程用的，对于实时进程可以用的调度策略是：**SCHED_FIFO**、**SCHED_RR**，而对于非实时进程则是：**SCHED_OTHER**、**SCHED_OTHER**、**SCHED_IDLE**。

系统的整体优先级策略是：如果系统中存在需要执行的实时进程，则优先执行实时进程。直到实时进程退出或者主动让出CPU时，才会调度执行非实时进程。实时进程可以指定的优先级范围为1-99，将一个要执行的程序以实时方式执行的方法为：

```cpp
[root@zorrozou-pc0 zorro]# chrt 10 bash
[root@zorrozou-pc0 zorro]# chrt -p $$
pid 14840's current scheduling policy: SCHED_RR
pid 14840's current scheduling priority: 10
```

可以看到，新打开的bash已经是实时进程，默认调度策略为SCHED_RR，优先级为10。如果想修改调度策略，就加个参数：

```cpp
[root@zorrozou-pc0 zorro]# chrt -f 10 bash
[root@zorrozou-pc0 zorro]# chrt -p $$
pid 14843's current scheduling policy: SCHED_FIFO
pid 14843's current scheduling priority: 10
```

刚才说过，SCHED_RR和SCHED_FIFO都是实时调度策略，只能给实时进程设置。对于所有实时进程来说，优先级高的（就是priority数字小的）进程一定会保证先于优先级低的进程执行。SCHED_RR和SCHED_FIFO的调度策略只有当两个实时进程的优先级一样的时候才会发生作用，其区别也是顾名思义：

**SCHED_FIFO:**以先进先出的队列方式进行调度，在优先级一样的情况下，谁先执行的就先调度谁，除非它退出或者主动释放CPU。

**SCHED_RR:**以时间片轮转的方式对相同优先级的多个进程进行处理。时间片长度为100ms。

这就是Linux对于实时进程的优先级和相关调度算法的描述。整体很简单，也很实用。而相对更麻烦的是非实时进程，它们才是Linux上进程的主要分类。对于非实时进程优先级的处理，我们首先还是要来介绍一下它们相关的调度算法：O1和CFS。

## O1调度

O1调度算法是在Linux 2.6开始引入的，到Linux 2.6.23之后内核将调度算法替换成了CFS。虽然O1算法已经不是当前内核所默认使用的调度算法了，但是由于大量线上的服务器可能使用的Linux版本还是老版本，所以我相信很多服务器还是在使用着O1调度器，那么费一点口舌简单交代一下这个调度器也是有意义的。这个调度器的名字之所以叫做O1，主要是因为其算法的时间复杂度是O1。

O1调度器仍然是根据经典的时间片分配的思路来进行整体设计的。简单来说，时间片的思路就是将CPU的执行时间分成一小段一小段的，假如是5ms一段。于是多个进程如果要“同时”执行，实际上就是每个进程轮流占用5ms的cpu时间，而从1s的时间尺度上看，这些进程就是在“同时”执行的。当然，对于多核系统来说，就是把每个核心都这样做就行了。而在这种情况下，如何支持优先级呢？实际上就是将时间片分配成大小不等的若干种，优先级高的进程使用大的时间片，优先级小的进程使用小的时间片。这样在一个周期结速后，优先级大的进程就会占用更多的时间而因此得到特殊待遇。O1算法还有一个比较特殊的地方是，即使是相同的nice值的进程，也会再根据其CPU的占用情况将其分成两种类型：CPU消耗型和IO消耗性。典型的CPU消耗型的进程的特点是，它总是要一直占用CPU进行运算，分给它的时间片总是会被耗尽之后，程序才可能发生调度。比如常见的各种算数运算程序。而IO消耗型的特点是，它经常时间片没有耗尽就自己主动先释放CPU了，比如vi，emacs这样的编辑器就是典型的IO消耗型进程。

为什么要这样区分呢？因为IO消耗型的进程经常是跟人交互的进程，比如shell、编辑器等。当系统中既有这种进程，又有CPU消耗型进程存在，并且其nice值一样时，假设给它们分的时间片长度是一样的，都是500ms，那么人的操作可能会因为CPU消耗型的进程一直占用CPU而变的卡顿。可以想象，当bash在等待人输入的时候，是不占CPU的，此时CPU消耗的程序会一直运算，假设每次都分到500ms的时间片，此时人在bash上敲入一个字符的时候，那么bash很可能要等个几百ms才能给出响应，因为在人敲入字符的时候，别的进程的时间片很可能并没有耗尽，所以系统不会调度bash程度进行处理。为了提高IO消耗型进程的响应速度，系统将区分这两类进程，并动态调整CPU消耗的进程将其优先级降低，而IO消耗型的将其优先级变高，以降低CPU消耗进程的时间片的实际长度。已知nice值的范围是-20-19，其对应priority值的范围是100－139，对于一个默认nice值为0的进程来说，其初始priority值应该是120，随着其不断执行，内核会观察进程的CPU消耗状态，并动态调整priority值，可调整的范围是+-5。就是说，最高其优先级可以呗自动调整到115，最低到125。这也是为什么nice值叫做静态优先级而priority值叫做动态优先级的原因。不过这个动态调整的功能在调度器换成CFS之后就不需要了，因为CFS换了另外一种CPU时间分配方式，这个我们后面再说。

再简单了解了O1算法按时间片分配CPU的思路之后，我们再来结合进程的状态简单看看其算法描述。我们都知道进程有5种状态：

1. **S（Interruptible sleep）：**可中断休眠状态。
2. **D（Uninterruptible sleep）：**不可中断休眠状态。
3. **R（Running or runnable）：**执行或者在可执行队列中。
4. **Z（Zombie process）：**僵尸。
5. **T（Stopped）：**暂停。

在CPU调度时，主要只关心R状态进程，因为其他状态进程并不会被放倒调度队列中进行调度。调度队列中的进程一般主要有两种情况，一种是进程已经被调度到CPU上执行，另一种是进程正在等待被调度。出现这两种状态的原因应该好理解，因为需要执行的进程数可能多于硬件的CPU核心数，比如需要执行的进程有8个而CPU核心只有4个，此时cpu满载的时候，一定会有4个进程处在“等待”状态，因为此时有另外四个进程正在占用CPU执行。

根据以上情况我们可以理解，系统当下需要同时进行调度处理的进程数（R状态进程数）和系统CPU的比值，可以一定程度的反应系统的“繁忙”程度。需要调度的进程越多，核心越少，则意味着系统越繁忙。除了进程执行本身需要占用CPU以外，多个进程的调度切换也会让系统繁忙程度增加的更多。所以，我们往往会发现，R状态进程数量在增长的情况下，系统的性能表现会下降。系统中可以使用uptime命令查看系统平均负载指数（load average）：

```cpp
[zorro@zorrozou-pc0 ~]$ uptime 
 16:40:56 up  2:12,  1 user,  load average: 0.05, 0.11, 0.16
```

其中load average中分别显示的是1分钟，5分钟，15分钟之内的平均负载指数（可以简单认为是相映时间范围内的R状态进程个数）。但是这个命令显示的数字是绝对个数，并没有表示出不同CPU核心数的实际情况。比如，如果我们的1分钟load average为16，而CPU核心数为32的话，那么这个系统的其实并不繁忙。但是如果CPU个数是8的话，那可能就意味着比较忙了。但是实际情况往往可能比这更加复杂，比如进程消耗类型也会对这个数字的解读有影响。总之，这个值的绝对高低并不能直观的反馈出来当前系统的繁忙程度，还需要根据系统的其它指标综合考虑。

**O1调度器在处理流程上大概是这样进行调度的：**

1. 首先，进程产生（fork）的时候会给一个进程分配一个时间片长度。这个新进程的时间片一般是父进程的一半，而父进程也会因此减少它的时间片长度为原来的一半。就是说，如果一个进程产生了子进程，那么它们将会平分当前时间片长度。比如，如果父进程时间片还剩100ms，那么一个fork产生一个子进程之后，子进程的时间片是50ms，父进程剩余的时间片是也是50ms。这样设计的目的是，为了防止进程通过fork的方式让自己所处理的任务一直有时间片。不过这样做也会带来少许的不公平，因为先产生的子进程获得的时间片将会比后产生的长，第一个子进程分到父进程的一半，那么第二个子进程就只能分到1/4。对于一个长期工作的进程组来说，这种影响可以忽略，因为第一轮时间片在耗尽后，系统会在给它们分配长度相当的时间片。
2. 针对所有R状态进程，O1算法使用两个队列组织进程，其中一个叫做活动队列，另一个叫做过期队列。活动队列中放的都是时间片未被耗尽的进程，而过期队列中放时间片被耗尽的进程。
3. 如1所述，新产生的进程都会先获得一个时间片，进入活动队列等待调度到CPU执行。而内核会在每个tick间隔期间对正在CPU上执行的进程进行检查。一般的tick间隔时间就是cpu时钟中断间隔，每秒钟会有1000个，即频率为1000HZ。每个tick间隔周期主要检查两个内容：1、当前正在占用CPU的进程是不是时间片已经耗尽了？2、是不是有更高优先级的进程在活动队列中等待调度？如果任何一种情况成立，就把则当前进程的执行状态终止，放到等待队列中，换当前在等待队列中优先级最高的那个进程执行。

以上就是O1调度的基本调度思路，当然实际情况是，还要加上SMP（对称多处理）的逻辑，以满足多核CPU的需求。目前在我的archlinux上可以用以下命令查看内核HZ的配置：

```cpp
[zorro@zorrozou-pc0 ~]$ zgrep CONFIG_HZ /proc/config.gz 
# CONFIG_HZ_PERIODIC is not set
# CONFIG_HZ_100 is not set
# CONFIG_HZ_250 is not set
CONFIG_HZ_300=y
# CONFIG_HZ_1000 is not set
CONFIG_HZ=300
```

我们发现我当前系统的HZ配置为300，而不是一般情况下的1000。大家也可以思考一下，配置成不同的数字（100、250、300、1000），对系统的性能到底会有什么影响？

## CFS完全公平调度

O1已经是上一代调度器了，由于其对多核、多CPU系统的支持性能并不好，并且内核功能上要加入cgroup等因素，Linux在2.6.23之后开始启用CFS作为对一般优先级(SCHED_OTHER)进程调度方法。在这个重新设计的调度器中，时间片，动态、静态优先级以及IO消耗，CPU消耗的概念都不再重要。CFS采用了一种全新的方式，对上述功能进行了比较完善的支持。

其设计的基本思路是，我们想要实现一个对所有进程完全公平的调度器。又是那个老问题：如何做到完全公平？答案跟上一篇IO调度中CFQ的思路类似：如果当前有n个进程需要调度执行，那么调度器应该在一个比较小的时间范围内，把这n个进程全都调度执行一遍，并且它们平分cpu时间，这样就可以做到所有进程的公平调度。那么这个比较小的时间就是任意一个R状态进程被调度的最大延时时间，即：任意一个R状态进程，都一定会在这个时间范围内被调度相应。这个时间也可以叫做调度周期，其英文名字叫做：sched_latency_ns。进程越多，每个进程在周期内被执行的时间就会被平分的越小。调度器只需要对所有进程维护一个累积占用CPU时间数，就可以衡量出每个进程目前占用的CPU时间总量是不是过大或者过小，这个数字记录在每个进程的vruntime中。所有待执行进程都以vruntime为key放到一个由红黑树组成的队列中，每次被调度执行的进程，都是这个红黑树的最左子树上的那个进程，即vruntime时间最少的进程，这样就保证了所有进程的相对公平。

在基本驱动机制上CFS跟O1一样，每次时钟中断来临的时候，都会进行队列调度检查，判断是否要进程调度。当然还有别的时机需要调度检查，**发生调度的时机可以总结为这样几个：**

1. 当前进程的状态转换时。主要是指当前进程终止退出或者进程休眠的时候。
2. 当前进程主动放弃CPU时。状态变为sleep也可以理解为主动放弃CPU，但是当前内核给了一个方法，可以使用sched_yield()在不发生状态切换的情况下主动让出CPU。
3. 当前进程的vruntime时间大于每个进程的理想占用时间时（delta_exec >
   ideal_runtime）。这里的ideal_runtime实际上就是上文说的sched_latency_ns／进程数n。当然这个值并不是一定这样得出，下文会有更详细解释。
4. 当进程从中断、异常或系统调用返回时，会发生调度检查。比如时钟中断。

## CFS的优先级

当然，CFS中还需要支持优先级。在新的体系中，优先级是以时间消耗（vruntime增长）的快慢来决定的。就是说，对于CFS来说，衡量的时间累积的绝对值都是一样纪录在vruntime中的，但是不同优先级的进程时间增长的比率是不同的，高优先级进程时间增长的慢，低优先级时间增长的快。比如，优先级为19的进程，实际占用cpu为1秒，那么在vruntime中就记录1s。但是如果是-20优先级的进程，那么它很可能实际占CPU用10s，在vruntime中才会纪录1s。CFS真实实现的不同nice值的cpu消耗时间比例在内核中是按照“每差一级cpu占用时间差10%左右”这个原则来设定的。这里的大概意思是说，如果有两个nice值为0的进程同时占用cpu，那么它们应该每人占50%的cpu，如果将其中一个进程的nice值调整为1的话，那么此时应保证优先级高的进程比低的多占用10%的cpu，就是nice值为0的占55%，nice值为1的占45%。那么它们占用cpu时间的比例为55:45。这个值的比例约为1.25。就是说，相邻的两个nice值之间的cpu占用时间比例的差别应该大约为1.25。根据这个原则，内核对40个nice值做了时间计算比例的对应关系，**它在内核中以一个数组存在：**

```cpp
static const int prio_to_weight[40] = {
 /* -20 */     88761,     71755,     56483,     46273,     36291,
 /* -15 */     29154,     23254,     18705,     14949,     11916,
 /* -10 */      9548,      7620,      6100,      4904,      3906,
 /*  -5 */      3121,      2501,      1991,      1586,      1277,
 /*   0 */      1024,       820,       655,       526,       423,
 /*   5 */       335,       272,       215,       172,       137,
 /*  10 */       110,        87,        70,        56,        45,
 /*  15 */        36,        29,        23,        18,        15,
};
```

我们看到，实际上nice值的最高优先级和最低优先级的时间比例差距还是很大的，绝不仅仅是例子中的十倍。由此我们也可以推导出每一个nice值级别计算vruntime的公式为：

```text
delta vruntime ＝ delta Time * 1024 / load
```

这个公式的意思是说，在nice值为0的时候（对应的比例值为1024），计算这个进程vruntime的实际增长时间值（delta vruntime）为：CPU占用时间（delta Time）* 1024 / load。在这个公式中load代表当前sched_entity的值，其实就可以理解为需要调度的进程（R状态进程）个数。load越大，那么每个进程所能分到的时间就越少。CPU调度是内核中会频繁进行处理的一个时间，于是上面的delta vruntime的运算会被频繁计算。除法运算会占用更多的cpu时间，所以内核编程中的一个原则就是，尽可能的不用除法。内核中要用除法的地方，基本都用乘法和位移运算来代替，所以上面这个公式就会变成：

```cpp
delta vruntime ＝ 
delta time * 1024 * (2^32 / (load * 2^32)) =
 (delta time * 1024 * Inverse（load）) >> 32
```

内核中为了方便不同nice值的Inverse(load)的相关计算，对做好了一个跟prio_to_weight数组一一对应的数组，在计算中可以直接拿来使用，**减少计算时的CPU消耗：**

```cpp
static const u32 prio_to_wmult[40] = {
 /* -20 */     48388,     59856,     76040,     92818,    118348,
 /* -15 */    147320,    184698,    229616,    287308,    360437,
 /* -10 */    449829,    563644,    704093,    875809,   1099582,
 /*  -5 */   1376151,   1717300,   2157191,   2708050,   3363326,
 /*   0 */   4194304,   5237765,   6557202,   8165337,  10153587,
 /*   5 */  12820798,  15790321,  19976592,  24970740,  31350126,
 /*  10 */  39045157,  49367440,  61356676,  76695844,  95443717,
 /*  15 */ 119304647, 148102320, 186737708, 238609294, 286331153,
};
```

具体计算细节不在这里细解释了，有兴趣的可以自行阅读代码：kernel/shced/fair.c（Linux 4.4）中的__calc_delta（）函数实现。

根据CFS的特性，我们知道调度器总是选择vruntime最小的进程进行调度。那么如果有两个进程的初始化vruntime时间一样时，一个进程被选择进行调度处理，那么只要一进行处理，它的vruntime时间就会大于另一个进程，CFS难道要马上换另一个进程处理么？出于减少频繁切换进程所带来的成本考虑，显然并不应该这样。CFS设计了一个sched_min_granularity_ns参数，用来设定进程被调度执行之后的最小CPU占用时间。

```cpp
[zorro@zorrozou-pc0 ~]$ cat /proc/sys/kernel/sched_min_granularity_ns 
2250000
```

一个进程被调度执行后至少要被执行这么长时间才会发生调度切换。我们知道无论到少个进程要执行，它们都有一个预期延迟时间，即：sched_latency_ns，系统中可以通过如下命令来查看这个时间：

```cpp
[zorro@zorrozou-pc0 ~]$ cat /proc/sys/kernel/sched_latency_ns 
18000000
```

在这种情况下，如果需要调度的进程个数为n，那么平均每个进程占用的CPU时间为sched_latency_ns／n。显然，每个进程实际占用的CPU时间会因为n的增大而减小。但是实现上不可能让它无限的变小，所以sched_min_granularity_ns的值也限定了每个进程可以获得的执行时间周期的最小值。当进程很多，导致使用了sched_min_granularity_ns作为最小调度周期时，对应的调度延时也就不在遵循sched_latency_ns的限制，而是以实际的需要调度的进程个数n * sched_min_granularity_ns进行计算。当然，我们也可以把这理解为CFS的”时间片”，不过我们还是要强调，CFS是没有跟O1类似的“时间片“的概念的，具体区别大家可以自己琢磨一下。

## 新进程的VRUNTIME值

CFS是通过vruntime最小值来选择需要调度的进程的，那么可以想象，在一个已经有多个进程执行了相对较长的系统中，这个队列中的vruntime时间纪录的数值都会比较长。如果新产生的进程直接将自己的vruntime值设置为0的话，那么它将在执行开始的时间内抢占很多的CPU时间，直到自己的vruntime追赶上其他进程后才可能调度其他进程，这种情况显然是不公平的。所以CFS对每个CPU的执行队列都维护一个min_vruntime值，这个值纪录了这个CPU执行队列中vruntime的最小值，当队列中出现一个新建的进程时，它的初始化vruntime将不会被设置为0，而是根据min_vruntime的值为基础来设置。这样就保证了新建进程的vruntime与老进程的差距在一定范围内，不会因为vruntime设置为0而在进程开始的时候占用过多的CPU。

新建进程获得的实际vruntime值跟一些设置有关，比如：

```text
[zorro@zorrozou-pc0 ~]$ cat /proc/sys/kernel/sched_child_runs_first 
```

这个文件是fork之后是否让子进程优先于父进程执行的开关。0为关闭，1为打开。如果这个开关打开，就意味着子进程创建后，保证子进程在父进程之前被调度。另外，在源代码目录下的kernel/sched/features.h文件中，还规定了一系列调度器属性开关。而其中：

```text
/*
 * Place new tasks ahead so that they do not starve already running
 * tasks
 */
SCHED_FEAT(START_DEBIT, true)
```

这个参数规定了新进程启动之后第一次运行会有延时。这意味着新进程的vruntime设置要比默认值大一些，这样做的目的是防止应用通过不停的fork来尽可能多的获得执行时间。子进程在创建的时候，vruntime的定义的步骤如下，首先vruntime被设置为min_vruntime。然后判断START_DEBIT位是否被值为true，如果是则会在min_vruntime的基础上增大一些，增大的时间实际上就是一个进程的调度延时时间，即上面描述过的calc_delta_fair()函数得到的结果。这个时间设置完毕之后，就检查sched_child_runs_first开关是否打开，如果打开（值被设置为1），就比较新进程的vruntime和父进程的vruntime哪个更小，并将新进程的vruntime设置为更小的那个值，而父进程的vruntime设置为更大的那个值，以此保证子进程一定在父进程之前被调度。

## IO消耗型进程的处理

根据前文，我们知道除了可能会一直占用CPU时间的CPU消耗型进程以外，还有一类叫做IO消耗类型的进程，它们的特点是基本不占用CPU，主要行为是在S状态等待响应。这类进程典型的是vim，bash等跟人交互的进程，以及一些压力不大的，使用了多进程（线程）的或select、poll、epoll的网络代理程序。如果CFS采用默认的策略处理这些程序的话，相比CPU消耗程序来说，这些应用由于绝大多数时间都处在sleep状态，它们的vruntime时间基本是不变的，一旦它们进入了调度队列，将会很快被选择调度执行。对比O1调度算法，这种行为相当于自然的提高了这些IO消耗型进程的优先级，于是就不需要特殊对它们的优先级进行“动态调整”了。

但这样的默认策略也是有问题的，有时CPU消耗型和IO消耗型进程的区分不是那么明显，有些进程可能会等一会，然后调度之后也会长时间占用CPU。这种情况下，如果休眠的时候进程的vruntime保持不变，那么等到休眠被唤醒之后，这个进程的vruntime时间就可能会比别人小很多，从而导致不公平。所以对于这样的进程，CFS也会对其进行时间补偿。补偿方式为，如果进程是从sleep状态被唤醒的，而且GENTLE_FAIR_SLEEPERS属性的值为true，则vruntime被设置为sched_latency_ns的一半和当前进程的vruntime值中比较大的那个。sched_latency_ns的值可以在这个文件中进行设置：

```text
[zorro@zorrozou-pc0 ~]$ cat /proc/sys/kernel/sched_latency_ns 
18000000
```

因为系统中这种调度补偿的存在，IO消耗型的进程总是可以更快的获得响应速度。这是CFS处理与人交互的进程时的策略，即：通过提高响应速度让人的操作感受更好。但是有时候也会因为这样的策略导致整体性能受损。在很多使用了多进程（线程）或select、poll、epoll的网络代理程序，一般是由多个进程组成的进程组进行工作，典型的如apche、nginx和php-fpm这样的处理程序。它们往往都是由一个或者多个进程使用nanosleep()进行周期性的检查是否有新任务，如果有责唤醒一个子进程进行处理，子进程的处理可能会消耗CPU，而父进程则主要是sleep等待唤醒。这个时候，由于系统对sleep进程的补偿策略的存在，新唤醒的进程就可能会打断正在处理的子进程的过程，抢占CPU进行处理。当这种打断很多很频繁的时候，CPU处理的过程就会因为频繁的进程上下文切换而变的很低效，从而使系统整体吞吐量下降。此时我们可以使用开关禁止唤醒抢占的特性。

```text
[root@zorrozou-pc0 zorro]# cat /sys/kernel/debug/sched_features
GENTLE_FAIR_SLEEPERS START_DEBIT NO_NEXT_BUDDY LAST_BUDDY CACHE_HOT_BUDDY WAKEUP_PREEMPTION NO_HRTICK NO_DOUBLE_TICK LB_BIAS NONTASK_CAPACITY TTWU_QUEUE RT_PUSH_IPI NO_FORCE_SD_OVERLAP RT_RUNTIME_SHARE NO_LB_MIN ATTACH_AGE_LOAD
```

上面显示的这个文件的内容就是系统中用来控制kernel/sched/features.h这个文件所列内容的开关文件，其中WAKEUP_PREEMPTION表示：目前的系统状态是打开sleep唤醒进程的抢占属性的。可以使用如下命令关闭这个属性：

```text
[root@zorrozou-pc0 zorro]# echo NO_WAKEUP_PREEMPTION > /sys/kernel/debug/sched_features
[root@zorrozou-pc0 zorro]# cat /sys/kernel/debug/sched_features
GENTLE_FAIR_SLEEPERS START_DEBIT NO_NEXT_BUDDY LAST_BUDDY CACHE_HOT_BUDDY NO_WAKEUP_PREEMPTION NO_HRTICK NO_DOUBLE_TICK LB_BIAS NONTASK_CAPACITY TTWU_QUEUE RT_PUSH_IPI NO_FORCE_SD_OVERLAP RT_RUNTIME_SHARE NO_LB_MIN ATTACH_AGE_LOAD
```

其他相关参数的调整也是类似这样的方式。其他我没讲到的属性的含义，大家可以看kernel/sched/features.h文件中的注释。

系统中还提供了一个sched_wakeup_granularity_ns配置文件，这个文件的值决定了唤醒进程是否可以抢占的一个时间粒度条件。默认CFS的调度策略是，如果唤醒的进程vruntime小于当前正在执行的进程，那么就会发生唤醒进程抢占的情况。而sched_wakeup_granularity_ns这个参数是说，只有在当前进程的vruntime时间减唤醒进程的vruntime时间所得的差大于sched_wakeup_granularity_ns时，才回发生抢占。就是说sched_wakeup_granularity_ns的值越大，越不容易发生抢占。

## CFS和其他调度策略

**SCHED_BATCH**

在上文中我们说过，CFS调度策略主要是针对chrt命令显示的SCHED_OTHER范围的进程，实际上就是一般的非实时进程。我们也已经知道，这样的一般进程还包括另外两种：SCHED_BATCH和SCHED_IDLE。在CFS的实现中，集成了对SCHED_BATCH策略的支持，并且其功能和SCHED_OTHER策略几乎是一致的。唯一的区别在于，如果一个进程被用chrt命令标记成SCHED_OTHER策略的话，CFS将永远认为这个进程是CPU消耗型的进程，不会对其进行IO消耗进程的时间补偿。这样做的唯一目的是，可以在确认进程是CPU消耗型的进程的前提下，对其尽可能的进行批处理方式调度（batch），以减少进程切换带来的损耗，提高吞度量。实际上这个策略的作用并不大，内核中真正的处理区别只是在标记为SCHED_BATCH时进程在sched_yield主动让出cpu的行为发生是不去更新cfs的队列时间，这样就让这些进程在主动让出CPU的时候（执行sched_yield）不会纪录其vruntime的更新，从而可以继续优先被调度到。对于其他行为，并无不同。

**SCHED_IDLE**

如果一个进程被标记成了SCHED_IDLE策略，调度器将认为这个优先级是很低很低的，比nice值为19的优先级还要低。系统将只在CPU空闲的时候才会对这样的进程进行调度执行。若果存在多个这样的进程，它们之间的调度方式跟正常的CFS相同。

**SCHED_DEADLINE**

最新的Linux内核还实现了一个最新的调度方式叫做SCHED_DEADLINE。跟IO调度类似，这个算法也是要实现一个可以在最终期限到达前让进程可以调度执行的方法，保证进程不会饿死。目前大多数系统上的chrt还没给配置接口，暂且不做深入分析。

另外要注意的是，SCHED_BATCH和SCHED_IDLE一样，只能对静态优先级（即nice值）为0的进程设置。操作命令如下：

```text
[zorro@zorrozou-pc0 ~]$ chrt -i 0 bash
[zorro@zorrozou-pc0 ~]$ chrt -p $$
pid 5478's current scheduling policy: SCHED_IDLE
pid 5478's current scheduling priority: 0

[zorro@zorrozou-pc0 ~]$ chrt -b 0 bash
[zorro@zorrozou-pc0 ~]$ chrt -p $$
pid 5502's current scheduling policy: SCHED_BATCH
pid 5502's current scheduling priority: 0
```

## 多CPU的CFS调度

在上面的叙述中，我们可以认为系统中只有一个CPU，那么相关的调度队列只有一个。实际情况是系统是有多核甚至多个CPU的，CFS从一开始就考虑了这种情况，它对每个CPU核心都维护一个调度队列，这样每个CPU都对自己的队列进程调度即可。这也是CFS比O1调度算法更高效的根本原因：每个CPU一个队列，就可以避免对全局队列使用大内核锁，从而提高了并行效率。当然，这样最直接的影响就是CPU之间的负载可能不均，为了维持CPU之间的负载均衡，CFS要定期对所有CPU进行load balance操作，于是就有可能发生进程在不同CPU的调度队列上切换的行为。这种操作的过程也需要对相关的CPU队列进行锁操作，从而降低了多个运行队列带来的并行性。不过总的来说，CFS的并行队列方式还是要比O1的全局队列方式要高效。尤其是在CPU核心越来越多的情况下，全局锁的效率下降显著增加。

CFS对多个CPU进行负载均衡的行为是idle_balance()函数实现的，这个函数会在CPU空闲的时候由schedule()进行调用，让空闲的CPU从其他繁忙的CPU队列中取进程来执行。我们可以通过查看/proc/sched_debug的信息来查看所有CPU的调度队列状态信息以及系统中所有进程的调度信息。内容较多，我就不在这里一一列出了，有兴趣的同学可以自己根据相关参考资料（最好的资料就是内核源码）了解其中显示的相关内容分别是什么意思。

在CFS对不同CPU的调度队列做均衡的时候，可能会将某个进程切换到另一个CPU上执行。此时，CFS会在将这个进程出队的时候将vruntime减去当前队列的min_vruntime，其差值作为结果会在入队另一个队列的时候再加上所入队列的min_vruntime，以此来保持队列切换后CPU队列的相对公平。

## 最后

本文的目的是从Linux系统进程的优先级为出发点，通过了解相关的知识点，希望大家对系统的进程调度有个整体的了解。其中我们也对CFS调度算法进行了比较深入的分析。在我的经验来看，这些知识对我们在观察系统的状态和相关优化的时候都是非常有用的。比如在使用top命令的时候，NI和PR值到底是什么意思？类似的地方还有ps命令中的NI和PRI值、ulimit命令-e和-r参数的区别等等。当然，希望看完本文后，能让大家对这些命令显示的了解更加深入。除此之外，我们还会发现，虽然top命令中的PR值和ps -l命令中的PRI值的含义是一样的，但是在优先级相同的情况下，它们显示的值确不一样。那么你知道为什么它们显示会有区别吗？这个问题的答案留给大家自己去寻找吧。

-----

版权声明：本文为知乎博主「玩转Linux内核」的原创文章，遵循CC 4.0 BY-SA版权协议，转载请附上原文出处链接及本声明。
原文链接：https://zhuanlan.zhihu.com/p/426399519