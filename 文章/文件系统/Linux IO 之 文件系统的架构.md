## 本文概述：
应用程序 ->read ->文件系统的代码 如何实现？<br>
当目录里面 A/B/C ，是如何找到C的全过程？<br>
文件系统如何描述文件在磁盘的哪些位置？<br>
硬链接和 符号链接的详细区别？<br>
userspace的文件系统的实现？<br>

## 一切都是文件： VFS

![image](https://user-images.githubusercontent.com/87457873/127651033-de62eda0-638e-4a5b-b384-6e82ad301a46.png)

文件系统的设计，类似 抽象基类，面向对象的思想。

虚函数都必须由底层派生出的实例实现，使用成员函数 file_operations。在linux里面的文件操作，底层都要实现file_operations，抽象出owner，write，open，release。所以，无论是字符块，还是文件系统的文件，最终操作就必须是file_operations。

例如，实现一个字符设备驱动，就是去实现file_operations。VFS_read时就会调用字符设备的file_operations。


## 字符设备文件、块设备文件

块设备的两种访问方法，一是访问裸分区，二是访问文件系统。

当直接访问裸分区，是通过fs/block_dev.c 中的 file_operations def_blk_fops，也有read,write,open，一切继承到file_operations。如果是访问文件系统，就会通过实现 {ext4}_file_operations 来对接VFS对文件的操作。

块设备驱动就不需要知道file_operations，无论是裸设备，还是文件系统的file。他们实现的file_operations是把linux中的各种设备，hook进 VFS的方法。

### 文件最终如何转化成对磁盘的访问？

file_operation 跟pagecache 以及硬盘的关系？

![image](https://user-images.githubusercontent.com/87457873/127651253-4b14bf4b-1f25-4c5a-9315-66ba3be77deb.png)

整个文件系统里，除了放文件本身的数据，还包括文件的管理数据，包括

* super block，保存在全局的 superblock结构中。
* inode，是文件的唯一特定标识，文件系统使用bitmap来标识，inode是否使用。
* block bitmap，来表示这些block是否占用，它在改变文件大小，创建删除等操作时，都会改变。
* inode table/diagram ： bitmap 只是表示inode和block是否被占用。

![image](https://user-images.githubusercontent.com/87457873/127651295-1cb0542f-0340-41a7-9a0e-b17cf5ea2865.png)

## 超级块、目录、inode

* file_system_type 数据结构： 指的是 文件系统的类型，mount/umount 的时候会用。
* superblock数据结构：包含super_operations，其中包含如何分配/销毁一个inode。
* inode 数据结构：包含 inode_operations 和 file_operations。

> file_operations里面记录这种类型的文件，包含哪些操作。
> inode_operations里面包含如何生成新的inode，根据文件名字找到inode，如何mkdir,unlink.

* dentry 数据结构: 对应路径，目录在文件系统里面是一个特殊的文件，文件的内容是一个inode和文件的表格。
* file 数据结构:

![image](https://user-images.githubusercontent.com/87457873/127651380-4e127a96-d14d-420e-9cd5-f9721affee67.png)

* inode表：包含文件的一些基本信息，大小，创建日期，属性。还有一些成员指向硬盘所在的位置。
申请slab区域，比如 ext4_inode_cache , ext3_inode_cache. 这些cache会创建单独的slab，这些slab和内存里的page一一对应。

ext2/ext4文件系统中存在间接映射表。

![image](https://user-images.githubusercontent.com/87457873/127651428-2fafcb0f-dc5b-4e63-860b-b5d49ade8dff.png)

![image](https://user-images.githubusercontent.com/87457873/127651439-0b3d58bb-7bab-4a33-8b4d-cb0c8e420cac.png)

![image](https://user-images.githubusercontent.com/87457873/127651448-17ab6ed1-5775-427d-a874-5683f627446d.png)

硬盘里的inode diagram里的数据结构，在内存中会通过slab分配器，组织成 xxx_inode_cache，出现在meminfo的可回收的内存。 inode表也会记录每一个inode 在硬盘中摆放的位置。

## 目录的组织

![image](https://user-images.githubusercontent.com/87457873/127651495-af5b103d-40c2-479b-8339-2565cf0d3800.png)

目录在硬盘里是一个特殊的文件，和之前的file结构体不同。目录在硬盘中对应一个inode，记录文件的名字和inode号。查找一个文件时，文件系统的 根inode和目录，根据根目录和根inode，找到根目录所在硬盘的位置。再去做字符串匹配，能够找到 /A/B/ 。inode表也会记录每一个inode 在硬盘中摆放的位置。

## icache和dcache，slab shrink

![image](https://user-images.githubusercontent.com/87457873/127651908-c221f27e-8b01-4093-8bb6-9a94400b6d36.png)

文件系统在实现时，在vfs这一层的 inode cache 和 dentry cache，不管硬盘的系统，跨所有文件系统的通用信息。

针对这些cache，这些可以回收的slab，linux提供了专门的slab shrink- 收缩函数。<br>
最后所有可回收的内存，都必须通过LRU算法去回收。<br>
有些自己申请的 reclaim的内存，由于没有写 shrink函数，所以就无法进行内存的回收。<br>

## 文件读写如何通过file_operation 和pagecache的关系

![image](https://user-images.githubusercontent.com/87457873/127651982-66ba1471-b32d-4c79-b733-b91564995a96.png)

![image](https://user-images.githubusercontent.com/87457873/127651993-487fe5a8-1bad-438b-ac01-317f18aacb31.png)

![image](https://user-images.githubusercontent.com/87457873/127652007-b57e63eb-5ed9-4125-9626-baf79faa6659.png)


## 发现并读取/usr/bin/xxx的全流程

![image](https://user-images.githubusercontent.com/87457873/127651566-cd18bafe-5ebe-48d6-917b-6eccda09cce1.png)

如上图，当你在硬盘查找 /usr/bin/emacs文件时，从根的inode和dentry，根据/的inode表，找到/ 目录文件所在的硬盘中的位置，读硬盘/目录文件的内容，发现 usr 对应inode 2, bin 对应inode 3, share 对应inode4。再去查inode表，inode 2所在硬盘的位置，即/usr 目录文件所在硬盘的位置。读出内容包括 var 对应 inode 10, bin 对应inode 11, opt对应inode 12，。

这个过程会查找很多inode和 dentry，这些都会通过 icache 和dcache缓存。

## 符号链接 与 硬链接

![image](https://user-images.githubusercontent.com/87457873/127651769-c1b5d554-37b8-4f1a-91ee-5532ee65d2b5.png)

文件名是特殊目录文件的内容，比如 A目录下有b\c\d，其实就是 A这个目录文件，里面对应目录b,c,d和对应inode的表。

硬链接：在硬盘中是同一个inode存在，在目录文件中多了一个目录和inode对应。

符号链接：是linux中是真实存在的实体文件，文件内容指向 其他文件。符号链接和文件是不同的inode。

1、硬链接不能跨本地文件系统<br>
2、硬链接不能针对目录<br>
3、针对目录的软链接，用rm -fr 删除不了目录里的内容<br>
4、针对目录的软链接，"cd .."进的是软链接所在目录的上级目录<br>
5、可以对文件执行unlink或rm，但是不能对目录执行unlink<br>

## 用户空间的文件系统： FUSE

用户空间文件系统 是操作系统中的概念，指完全在用户态实现的文件系统。

目前Linux通过内核模块对此进行支持。一些文件系统如ZFS，glusterfs使用FUSE实现。

![image](https://user-images.githubusercontent.com/87457873/127652044-b91f96f8-ea78-4d2b-aa37-4bcde036735d.png)

FUSE的工作原理如上图所示。假设基于FUSE的用户态文件系统hello挂载在/tmp/fuse目录下。当应用层程序要访问/tmp/fuse下的文件时，通过glibc中的函数进行系统调用，处理这些系统调用的VFS中的函数会调用FUSE在内核中的文件系统；内核中的FUSE文件系统将用户的请求，发送给用户态文件系统hello；用户态文件系统收到请求后，进行处理，将结果返回给内核中的FUSE文件系统；最后，内核中的FUSE文件系统将数据返回给用户态程序。

