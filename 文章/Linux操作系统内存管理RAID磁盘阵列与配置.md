## 1、RAID磁盘阵列

简称：独立冗余磁盘阵列

把多块独立的物理硬盘按不同的方式组合起来形成一个硬盘组（逻辑硬盘）。从而提供比单个硬盘更高的存储性能和提供数据备份技术。

### 1.1RAID级别

组成磁盘阵列的不同方式称为RAID级别（RAID Levels）

**常用的RAID级别：**

RAID0、RAID1、RAID5、RAID6、RAID1+0等

①、RAID 0（条带化存储）

1. RAID 0连续以位或字节为单位分割数据，并行读/写于多个磁盘上，因此具有很高的数据传输率，但它没有数据冗余。
2. RAID 0只是单纯地提高性能，并没有为数据的可靠性提供保证，而且其中的一个磁盘失效将影响到所有数据
3. RAID 0不能应用于数据安全性要求高的场合

②、RAID 1（镜像存储）

- 通过磁盘数据镜像实现数据冗余，在成对的独立磁盘上产生互为备份的数据
- 当原始数据繁忙时，可直接从镜像拷贝中读取数据，因此RAID 1可以提高读取性能
- RAID 1是磁盘阵列中单位成本最高的。但提供了很高的数据安全性和可用性。当一个磁盘失效时，系统可以自动切换到镜像磁盘上读写，而不需要重组失效的数据。

③、RAID 5

- N(N≥3)块盘组成阵列，一份数据产生N-1个条带，同时还有一份校验数据，共N份数据在N块盘上循环均衡存储
- N块盘同时读写，读性能很高，但由于有校验机制的问题，写性能相对不高
- （N-1）/N 磁盘利用率
- 可靠性高，允许坏一块盘，不影响所有数据

④、RAID 6

- N（N≥4）块盘组成阵列，（N-2）/N 磁盘利用率
- 与RAID 5相比，RAID 6增加了第二块独立的奇偶校验信息块
- 两个独立的奇偶系统使用不同的算法，即使两块磁盘同时失效也不会影响数据的使用
- 相对于RAID 5有更大的“写损失”，因此写性能较差

⑤、RAID 1+0(先做镜像，再做条带)

- N (偶数，N>=4)。块盘两两镜像后，再组合成一个RAID 0
- N/2磁盘利用率
- N/2块盘同时写入，N块盘同时读取
- 性能高，可靠性高

⑥、RAID 0+1(先做条带，再做镜像)

读写性能与RAID 10相同

安全性低于RAID 10

![img](https://pic1.zhimg.com/80/v2-3af4a5d53438b5675fccb5c4e59ad8b4_720w.webp)

## 2、创建软 RAID 磁盘阵列实验

**1、检查是否已安装mdadm 软件包**

![img](https://pic2.zhimg.com/80/v2-07e14384ea2796426306b78a16e2cb85_720w.webp)

**2、先关闭虚拟机，然后编辑虚拟机设置，添加4块硬盘，每块分配40G，点击确认后开启虚拟机**

![img](https://pic2.zhimg.com/80/v2-da6b4b1a67b20524e30997498144f001_720w.webp)

**3、我们使用xshell来进行连接，使用fdisk -l来查看分区情况**

![img](https://pic3.zhimg.com/80/v2-5e60c0f2344221b30cebb7e272880aae_720w.webp)


**4、对分区进行管理，创建分区并修改分区类型，这里示范一个/dev/sdb，其余的操作一样，就不示范了**

![img](https://pic3.zhimg.com/80/v2-cccf6811a3aa526774d0f2c6865778aa_720w.webp)

**5、使用fdisk -l看一下分区情况，是否全部转换完成**

![img](https://pic3.zhimg.com/80/v2-c5314db4d9bfbe88465dde7d5428dbde_720w.webp)

**6、验证一下磁盘是否已做raid，然后开始创建raid，这里我们创建一个raid名为md0，级别使用RAID5，然后-l3设置使用三个磁盘，-x1使用一块备份磁盘，再进行查看创建速度。**

![img](https://pic3.zhimg.com/80/v2-887d7e3d66e1f9c40ee65bd0014d32e2_720w.webp)

**7、这里已经创建好了，我们开始验证一下**

![img](https://pic2.zhimg.com/80/v2-1e889ee7e3cc46bb421dda188944b291_720w.webp)

**8、我们模拟让它坏掉一个磁盘，来测试一下备份磁盘是否会自动顶上**

![img](https://pic1.zhimg.com/80/v2-11ac37fcb3db5e8346ecca6e4ae76314_720w.webp)

**9、想使用起来得先进行格式化，再进行挂载，我接着尝试了一下在格式化和挂载之后备用磁盘是否还能自动顶上，实验结果：可以。**

![img](https://pic4.zhimg.com/80/v2-471b24d8561b269c443c53bd91b99143_720w.webp)

## 3、创建软 RAID 磁盘阵列步骤命令

**1、检查是否已安装mdadm 软件包**

```cpp
rpm -q mdadm
yum install -y mdadm
```

2、使用fdisk工具将新磁盘设备/dev/sdb、/dev/sdc、/dev/sdd、/dev/sde划分出主分区sdb1、sdc1、sdd1、sde1，并且把分区类型的 ID 标记号改为“fd”

```text
fdisk /dev/sdb
fdisk /dev/sdc
```

**3、创建 RAID 设备**

```cpp
#创建RAID5
mdadm -C -v /dev/md0 [-a yes] -l5 -n3 /dev/sd[bcd]1 -x1          /dev/sde1
```

- -C：表示新建；
- -v：显示创建过程中的详细信息。
- /dev/md0：创建 RAID5 的名称。
- -a yes：–auto，表示如果有什么设备文件没有存在的话就自动创建，可省略。
- -l：指定 RAID 的级别，l5 表示创建 RAID5。
- -n：指定使用几块硬盘创建 RAID，n3 表示使用 3 块硬盘创建 RAID。
- /dev/sd[bcd]1：指定使用这四块磁盘分区去创建 RAID。
- -x：指定使用几块硬盘做RAID的热备用盘，x1表示保留1块空闲的硬盘作备用
- /dev/sde1：指定用作于备用的磁盘

```cpp
cat /proc/mdstat		#还能查看创建RAID的进度
或者
mdadm -D /dev/md0       #查看RAID磁盘详细信息
mdadm -E /dev/sd[b-e]1  #检查磁盘是否已做RAID
```

**4、创建并挂载文件系统**

```cpp
mkfs -t xfs /dev/md0
mkdir /myraid
mount /dev/md0 /myraid/
df -Th
cp /etc/fstab /etc/fstab.bak
vim /etc/fstab
/dev/md0      /myraid        xfs   	 defaults   0  0
```

**5、实现故障恢复**

```cpp
mdadm /dev/md0 -f /dev/sdb1 		#模拟/dev/sdb1 故障
mdadm -D /dev/md0					#查看发现sde1已顶替sdb1
```

**mdadm命令其它常用选项：**

- -r：移除设备
- -a：添加设备
- -S：停止RAID
- -A：启动RAID

```cpp
mdadm -S /dev/md0
mdadm /dev/md0 -r /dev/sdb1
```

----

版权声明：本文为知乎博主「玩转Linux内核」的原创文章，遵循CC 4.0 BY-SA版权协议，转载请附上原文出处链接及本声明。
原文链接：https://zhuanlan.zhihu.com/p/448266123