## 1、前言

上文中我们分析了虚拟文件系统的结构以及常见的文件操作从用户态到虚拟文件系统再到底层实际文件系统的过程。而实际上我们并没有说明实际的文件系统如ext4是如何和磁盘进行交互的，这就是本文和下篇文章的重点：I/O之块设备和字符设备。输入输出设备我们大致可以分为两类：块设备（Block Device）和字符设备（Character Device）。

- 块设备将信息存储在固定大小的块中，每个块都有自己的地址。如硬盘就是常见的块设备。
- 字符设备发送或接收的是字节流，而不用考虑任何块结构，没有办法寻址。如鼠标就是常见的字符设备。

本文首先介绍虚拟文件系统下层直至硬件输入输出设备的结构关系，然后重点分析字符设备相关的整体逻辑情况。

## 2、 I/O架构

由于各种输入输出设备具有不同的硬件结构、驱动程序，因此我们采取了设备控制器这一中间层对上提供统一接口。设备控制器通过缓存来处理CPU和硬件I/O之间的交互关系，通过中断进行通知，因此我们需要有中断处理器对各种中断进行统一。由于每种设备的控制器的寄存器、缓冲区等使用模式，指令都不同，所以对于操作系统还需要一层对接各个设备控制器的设备驱动程序。

这里需要注意的是，设备控制器不属于操作系统的一部分，但是设备驱动程序属于操作系统的一部分。操作系统的内核代码可以像调用本地代码一样调用驱动程序的代码，而驱动程序的代码需要发出特殊的面向设备控制器的指令，才能操作设备控制器。设备驱动程序中是一些面向特殊设备控制器的代码，不同的设备不同。但是对于操作系统其它部分的代码而言，设备驱动程序有统一的接口。

![img](https://pic3.zhimg.com/80/v2-52f763375b79876fbd4e38121d185d9a_720w.webp)

设备驱动本身作为一个内核模块，通常以ko文件的形式存在，它有着其独特的代码结构：

- 头文件部分,设备驱动程序至少需要以下头文件

```text
#include <linux/module.h>
#include <linux/init.h>
```

- 调用MODULE_LICENSE声明lisence
- 初始化函数module_init和退出函数module_exit的定义，用于加载和卸载ko文件
- 文件系统的接口file_operation结构体定义
- 定义需要的函数

在下文的分析中，我们就将按照此顺序来剖析字符设备的源码，以弄懂字符设备的一般运行逻辑。关于设备驱动代码编写的详细知识可以参考《Linux设备驱动》一书，本文重点不在于如何编写代码，而是在于操作系统中的字符设备和块设备如何工作。

## 3、字符设备基本构成

**一个字符设备由3个部分组成：**

- 封装对于外部设备的操作的设备驱动程序，即 ko 文件模块，里面有模块初始化函数、中断处理函数、设备操作函数等。加载设备驱动程序模块的时候，模块初始化函数会被调用。在内核维护所有字符设备驱动的数据结构 cdev_map 里面注册设备号后，我们就可以很容易根据设备号找到相应的设备驱动程序。
- 特殊的设备驱动文件系统 devtmpfs，在/dev 目录下生成一个文件表示这个设备。打开一个字符设备文件和打开一个普通的文件有类似的数据结构，有文件描述符、有 struct file、指向字符设备文件的 dentry 和 inode。其对应的 inode 是一个特殊的 inode，里面有设备号。通过它我们可以在 cdev_map 中找到设备驱动程序，里面还有针对字符设备文件的默认操作 def_chr_fops。
- 字符设备文件的相关操作 file_operations 。一开始会指向 def_chr_fops，在调用 def_chr_fops 里面的 chrdev_open 函数的时候修改为指向设备操作函数，从而读写一个字符设备文件就会直接变成读写外部设备了。 这里主要涉及到了两个结构体：字符设备信息存储的struct cdev以及管理字符设备的cdev_map。

```cpp
struct cdev {
    struct kobject kobj;                  //内嵌的内核对象.
    struct module *owner;                 //该字符设备所在的内核模块的对象指针.
    const struct file_operations *ops;    //该结构描述了字符设备所能实现的方法，是极为关键的一个结构体.
    struct list_head list;                //用来将已经向内核注册的所有字符设备形成链表.
    dev_t dev;                            //字符设备的设备号，由主设备号和次设备号构成.
    unsigned int count;                   //隶属于同一主设备号的次设备号的个数.
} __randomize_layout;
```

cdev结构体还有另一个相关联的结构体char_device_struct。这里首先会定义主设备号和次设备号：主设备号用来标识与设备文件相连的驱动程序，用来反映设备类型。次设备号被驱动程序用来辨别操作的是哪个设备，用来区分同类型的设备。这里minorct指的是分配的区域，用于主设备号和次设备号的分配工作。

```cpp
static struct char_device_struct {
    struct char_device_struct *next;
    unsigned int major;
    unsigned int baseminor;
    int minorct;
    char name[64];
    struct cdev *cdev;		/* will die */
} *chrdevs[CHRDEV_MAJOR_HASH_SIZE];
```

cdev_map用于维护所有字符设备驱动，实际是结构体kobj_map，主要包括了一个互斥锁lock，一个probes[255]数组，数组元素为struct probe的指针，该结构体包括链表项、设备号、设备号范围等。所以我们将字符设备驱动最后保存为一个probe，并用cdev_map/kobj_map进行统一管理。

```cpp
static struct kobj_map *cdev_map;

struct kobj_map {
    struct probe {
        struct probe *next;
        dev_t dev;
        unsigned long range;
        struct module *owner;
        kobj_probe_t *get;
        int (*lock)(dev_t, void *);
        void *data;
    } *probes[255];
    struct mutex *lock;
};
```

## 4、打开字符设备

字符设备有很多种，这里以打印机设备为输出设备的例子，源码位于drivers/char/lp.c。以鼠标为输入设备的例子，源码位于drivers/input/mouse/logibm.c。下面将根据上述的字符设备的三个组成部分分别剖析如何创建并打开字符设备。

### 4.1 加载

字符设备的使用从加载开始，通常我们会使用insmod命令或者modprobe命令加载ko文件，ko文件的加载则从module_init调用该设备自定义的初始函数开始。对于打印机来说，其初始化函数定义为lp_init_module()，实际调用lp_init()。lp_init()会初始化打印机结构体，并调用register_chardev()注册该字符设备。

```cpp
module_init(lp_init_module);

static int __init lp_init_module(void)
{
......
    return lp_init();
}

static int __init lp_init(void)
{
......
    if (register_chrdev(LP_MAJOR, "lp", &lp_fops)) {
        printk(KERN_ERR "lp: unable to get major %d\n", LP_MAJOR);
        return -EIO;
    }
......
}
```

register_chrdev()实际调用__register_chrdev()，该函数会进行字符设备的注册操作。其主要逻辑如下

- 调用__register_chrdev_region()注册字符设备的主设备号和名称
- 调用cdev_alloc()分配结构体struct cdev
- 将 cdev 的 ops 成员变量指向这个模块声明的 file_operations
- 调用cdev_add()将这个字符设备添加到结构体 struct kobj_map *cdev_map ，该结构体用于统一管理所有字符设备。

```cpp
static inline int register_chrdev(unsigned int major, const char *name,
                  const struct file_operations *fops)
{
    return __register_chrdev(major, 0, 256, name, fops);
}

int __register_chrdev(unsigned int major, unsigned int baseminor,
              unsigned int count, const char *name,
              const struct file_operations *fops)
{
    struct char_device_struct *cd;
    struct cdev *cdev;
    int err = -ENOMEM;
    cd = __register_chrdev_region(major, baseminor, count, name);
    if (IS_ERR(cd))
        return PTR_ERR(cd);
    cdev = cdev_alloc();
    if (!cdev)
        goto out2;
    cdev->owner = fops->owner;
    cdev->ops = fops;
    kobject_set_name(&cdev->kobj, "%s", name);
    err = cdev_add(cdev, MKDEV(cd->major, baseminor), count);
......
}
// 拼接ma和mi
#define MINORBITS	20
#define MKDEV(ma,mi)	(((ma) << MINORBITS) | (mi))
```

对于鼠标来说，加载也是类似的：注册为logibm_init()函数。但是这里没有调用register_chrdev()而是使用input_register_device()，原因在于输入设备会统一由input_init()初始化，之后加入的输入设备通过input_register_device()注册到input的管理结构体中进行统一管理。

```text
module_init(logibm_init);

static int __init logibm_init(void)
{
......
    err = input_register_device(logibm_dev);
......
}
```

### 4.2 创建文件设备

加载完ko文件后，Linux内核会通过mknod在/dev目录下创建一个设备文件，只有有了这个设备文件，我们才能通过文件系统的接口对这个设备文件进行操作。mknod本身是一个系统调用，主要逻辑为调用user_path_create()为该设备文件创建dentry，然后对于S_IFCHAR或者S_IFBLK会调用vfs_mknod()去调用对应文件系统的操作。

```cpp
SYSCALL_DEFINE3(mknod, const char __user *, filename, umode_t, mode, unsigned, dev)
{
    return sys_mknodat(AT_FDCWD, filename, mode, dev);
}

SYSCALL_DEFINE4(mknodat, int, dfd, const char __user *, filename, umode_t, mode,
    unsigned, dev)
{
    struct dentry *dentry;
    struct path path;
......
    dentry = user_path_create(dfd, filename, &path, lookup_flags);
......
    switch (mode & S_IFMT) {
......
        case S_IFCHR: case S_IFBLK:
            error = vfs_mknod(path.dentry->d_inode,dentry,mode,
            new_decode_dev(dev));
        break;
......
  }
}

int vfs_mknod(struct inode *dir, struct dentry *dentry, umode_t mode, dev_t dev)
{
......
    error = dir->i_op->mknod(dir, dentry, mode, dev);
......
}
```

对于/dev目录下的设备驱动来说，所属的文件系统为devtmpfs文件系统，即设备驱动临时文件系统。devtmpfs对应的文件系统定义如下

```cpp
static struct file_system_type dev_fs_type = { 
    .name = "devtmpfs", 
    .mount = dev_mount, 
    .kill_sb = kill_litter_super,
};

static struct dentry *dev_mount(struct file_system_type *fs_type, int flags, 
         const char *dev_name, void *data)
{
#ifdef CONFIG_TMPFS 
    return mount_single(fs_type, flags, data, shmem_fill_super);
#else return 
    mount_single(fs_type, flags, data, ramfs_fill_super);
#endif
}
```

从这里可以看出，devtmpfs 在挂载的时候有两种模式：一种是 ramfs，一种是 shmem ，都是基于内存的文件系统。这两个 mknod 虽然实现不同，但是都会调用到同一个函数 init_special_inode()。显然这个文件是个特殊文件，inode 也是特殊的。这里这个 inode 可以关联字符设备、块设备、FIFO 文件、Socket 等。我们这里只看字符设备。这里的 inode 的 file_operations 指向一个 def_chr_fops，这里面只有一个 open，就等着你打开它。另外，inode 的 i_rdev 指向这个设备的 dev_t。通过这个 dev_t，可以找到我们刚刚加载的字符设备 cdev。

```cpp
static const struct inode_operations ramfs_dir_inode_operations = {
......
  .mknod    = ramfs_mknod,
};

static const struct inode_operations shmem_dir_inode_operations = {
#ifdef CONFIG_TMPFS
......
  .mknod    = shmem_mknod,
};

void init_special_inode(struct inode *inode, umode_t mode, dev_t rdev)
{
    inode->i_mode = mode;
    if (S_ISCHR(mode)) {
        inode->i_fop = &def_chr_fops;
        inode->i_rdev = rdev;
    } else if (S_ISBLK(mode)) {
        inode->i_fop = &def_blk_fops;
        inode->i_rdev = rdev;
    } else if (S_ISFIFO(mode))
        inode->i_fop = &pipefifo_fops;
    else if (S_ISSOCK(mode))
        ;  /* leave it no_open_fops */
    else
        printk(KERN_DEBUG "init_special_inode: bogus i_mode (%o) for"
                  " inode %s:%lu\n", mode, inode->i_sb->s_id,
                  inode->i_ino);    
}

const struct file_operations def_chr_fops = {
    .open = chrdev_open,
};
```

由此我们完成了/dev下文件的创建，并利用rdev和生成的字符设备进行了关联。

### 4.3 打开字符设备

如打开普通文件一样，打开字符设备也会首先分配对应的文件描述符fd，然后生成struct file结构体与其绑定，并将file关联到对应的dentry从而可以接触inode。在进程里面调用 open() 函数，最终会调用到这个特殊的 inode 的 open() 函数，也就是 chrdev_open()。

**chrdev_open()主要逻辑为;**

- 调用kobj_lookup()，通过设备号i_cdev关联对应的设备驱动程序
- 调用fops_get()将设备驱动程序自己定义的文件操作p->ops赋值给fops
- 调用设备驱动程序的 file_operations 的 open() 函数真正打开设备。对于打印机，调用的是 lp_open()。对于鼠标调用的是 input_proc_devices_open()，最终会调用到 logibm_open()。

```cpp
/*
 * Called every time a character special file is opened
 */
static int chrdev_open(struct inode *inode, struct file *filp)
{
    const struct file_operations *fops;
    struct cdev *p;
    struct cdev *new = NULL;
......
    p = inode->i_cdev;
......
    kobj = kobj_lookup(cdev_map, inode->i_rdev, &idx);
......      
    fops = fops_get(p->ops);
......
    replace_fops(filp, fops);
    if (filp->f_op->open) {
        ret = filp->f_op->open(inode, filp);
......
}
```

**上述过程借用极客时间中的图来作为总结：**

![img](https://pic3.zhimg.com/80/v2-462b2f8f647974148c76f413e19390a2_720w.webp)

## 5、 写入字符设备

写入字符设备和写入普通文件一样，调用write()函数执行。该函数在内核里查询系统调用表最终调用sys_write()，并根据fd描述符获取对应的file结构体，接着调用vfs_write()去调用对应的文件系统自定义的写入函数file->f_op->write()。对于打印机来说，最终调用的是自定义的lp_write()函数。

这里写入的重点在于调用 copy_from_user() 将数据从用户态拷贝到内核态的缓存中，然后调用 parport_write() 写入外部设备。这里还有一个 schedule() 函数，也即写入的过程中，给其他线程抢占 CPU 的机会。如果写入字节数多，不能一次写完，就会在循环里一直调用 copy_from_user() 和 parport_write()，直到写完为止。

```cpp
static ssize_t lp_write(struct file *file, const char __user *buf,
            size_t count, loff_t *ppos)
{
    unsigned int minor = iminor(file_inode(file));
    struct parport *port = lp_table[minor].dev->port;
    char *kbuf = lp_table[minor].lp_buffer;
    ssize_t retv = 0;
    ssize_t written;
    size_t copy_size = count;
......
    /* Need to copy the data from user-space. */
    if (copy_size > LP_BUFFER_SIZE)
        copy_size = LP_BUFFER_SIZE;
......
    if (copy_from_user(kbuf, buf, copy_size)) {
        retv = -EFAULT;
        goto out_unlock;
    }
......
    do {
        /* Write the data. */
        written = parport_write(port, kbuf, copy_size);
        if (written > 0) {
            copy_size -= written;
            count -= written;
            buf  += written;
            retv += written;
        }
......
        if (need_resched())
            schedule(); 
        if (count) {
            copy_size = count;
            if (copy_size > LP_BUFFER_SIZE)
                copy_size = LP_BUFFER_SIZE;
            if (copy_from_user(kbuf, buf, copy_size)) {
                if (retv == 0)
                    retv = -EFAULT;
                break;
            }
        }
    } while (count > 0);
......
}
```

## 6、字符设备的控制

在Linux中，我们常用ioctl()来对I/O设备进行一些读写之外的特殊操作。其参数主要由文件描述符fd，命令cmd以及命令参数arg构成。其中cmd由几个部分拼接成整型，主要结构为

- 最低8位为 NR，表示命令号；
- 次低8位为 TYPE，表示类型；
- 14位表示参数的大小；
- 最高2位是 DIR，表示写入、读出，还是读写。

![img](https://pic2.zhimg.com/80/v2-9fcd5c0b10447ecb5b31d1584ac3fb5d_720w.webp)

ioctl()也是一个系统调用，其中fd 是这个设备的文件描述符，cmd 是传给这个设备的命令，arg 是命令的参数。主要调用do_vfs_ioctl()完成实际功能。

```cpp
SYSCALL_DEFINE3(ioctl, unsigned int, fd, unsigned int, cmd, unsigned long, arg)
{
    int error;
    struct fd f = fdget(fd);
......
    error = do_vfs_ioctl(f.file, fd, cmd, arg);
    fdput(f);
    return error;
}
```

do_vfs_ioctl()对于已经定义好的 cmd进行相应的处理。如果不是默认定义好的 cmd，则执行默认操作：对于普通文件，调用 file_ioctl，对于其他文件调用 vfs_ioctl。

```cpp
/*
 * When you add any new common ioctls to the switches above and below
 * please update compat_sys_ioctl() too.
 *
 * do_vfs_ioctl() is not for drivers and not intended to be EXPORT_SYMBOL()'d.
 * It's just a simple helper for sys_ioctl and compat_sys_ioctl.
 */
int do_vfs_ioctl(struct file *filp, unsigned int fd, unsigned int cmd,
         unsigned long arg)
{
    int error = 0;
    int __user *argp = (int __user *)arg;
    struct inode *inode = file_inode(filp);
    switch (cmd) {
    case FIOCLEX:
        set_close_on_exec(fd, 1);
        break;
    case FIONCLEX:
        set_close_on_exec(fd, 0);
        break;
    case FIONBIO:
        error = ioctl_fionbio(filp, argp);
        break;
    case FIOASYNC:
        error = ioctl_fioasync(fd, filp, argp);
        break;
......
    default:
        if (S_ISREG(inode->i_mode))
            error = file_ioctl(filp, cmd, arg);
        else
            error = vfs_ioctl(filp, cmd, arg);
        break;
    }
    return error;
}
```

对于字符设备驱动程序，最终会调用vfs_ioctl()。这里面调用的是 struct file 里 file_operations 的 unlocked_ioctl() 函数。我们前面初始化设备驱动的时候，已经将 file_operations 指向设备驱动的 file_operations 了。这里调用的是设备驱动的 unlocked_ioctl。对于打印机程序来讲，调用的是 lp_ioctl()。

```cpp
long vfs_ioctl(struct file *filp, unsigned int cmd, unsigned long arg)
{
    int error = -ENOTTY;
    if (!filp->f_op->unlocked_ioctl)
        goto out;
    error = filp->f_op->unlocked_ioctl(filp, cmd, arg);
    if (error == -ENOIOCTLCMD)
        error = -ENOTTY;
 out:
    return error;
} EXPORT_SYMBOL(vfs_ioctl);
```

打印机的lp_do_ioctl()主要逻辑也是针对cmd采用switch()语句分情况进行处理。主要包括使用LP_XXX()宏定义赋值标记位和调用copy_to_user()将用户想得到的信息返回给用户态。

```cpp
static int lp_do_ioctl(unsigned int minor, unsigned int cmd,
    unsigned long arg, void __user *argp)
{
    int status;
    int retval = 0;
......
    switch ( cmd ) {
        case LPTIME:
            if (arg > UINT_MAX / HZ)
                return -EINVAL;
            LP_TIME(minor) = arg * HZ/100;
            break;
        case LPCHAR:
            LP_CHAR(minor) = arg;
            break;
        case LPABORT:
            if (arg)
                LP_F(minor) |= LP_ABORT;
            else
                LP_F(minor) &= ~LP_ABORT;
            break;
    ......
        case LPGETIRQ:
            if (copy_to_user(argp, &LP_IRQ(minor),
                    sizeof(int)))
                return -EFAULT;
            break;
......
}
```

## 7、总结

本文简单介绍了设备驱动程序的结构，并在此基础上介绍了字符设备从创建到打开、写入以及控制的整个流程。

----

版权声明：本文为知乎博主「玩转Linux内核」的原创文章，遵循CC 4.0 BY-SA版权协议，转载请附上原文出处链接及本声明。
原文链接：https://zhuanlan.zhihu.com/p/445188798