## **1、编译前的准备**

下载linux源文件：https://www.kernel.org/，我下载的是linux-3.7.4版本，解压到/usr/src/kernels目录中，然后进入/usr/src/kernels/linux-3.7.4中，用make menuconfig命令来选择要编译的模块，但使用make menuconfig（重新编译内核常用的命令，还可以用其他的）报下面的错误：

![img](https://pic2.zhimg.com/80/v2-99c76217f4575d98a9001a4fd8c1681d_720w.webp)

说缺少ncurses库，然后安装ncurses开发库就可以了，ubuntu下貌似是libncurses-dev包

yum install ncurses-devel.i686

再次使用make menuconfig，出现下面的界面：

![img](https://pic4.zhimg.com/80/v2-9749efd129f16d1a3c6f980cf3d5c7a3_720w.webp)

然后我直接保存了，都用的默认的选项。

## **2、编译内核**

如果你是第一次重新编译内核，先用"make mrproper"命令处理一下内核代码目录中残留的文件，由于我们不知道源代码文件中是否包含像.o之类的文件。

如果不是第一次的话，使用"make clean"命令来清楚.o等编译内核产生的中间文件，但不会删除配置文件。

使用"make bzImage"命令来编译内核，这个内核是经过压缩的。或者使用“make -j10 bzImage”命令，表示用10个线程来编译。

使用"make modules"来编译模块，这个会花费比较长的时间。

上面的命令会花费非常长的时间，编译的动作依据你选择的项目以及你主机硬件的效能而不同。 最后制作出来的数据是被放置在 /usr/src/kernels/linux-3.7.4/ 这个目录下，还没有被放到系统的相关路径中喔！在上面的编译过程当中，如果有发生任何错误的话， 很可能是由于核心模块选择的不好，可能你需要重新以 make menuconfig 再次的检查一下你的相关配置喔！ 如果还是无法成功的话，那么或许将原本系统的内核代码中的 .config 文件，复制到你的内核代码目录下， 然后据以修改，应该就可以顺利的编译出你的核心了。可以发现你的核心已经编译好而且放置在 /usr/src/kernels/linux-3.7.4/arch/x86/boot/bzImage。

## **3、安装模块**

使用命令"make modules_install"安装模块，执行成功后，最终会在 /lib/modules 底下创建起你这个核心的相关模块，我的模块放在/lib/modules/3.7.4目录下，其中3.7.4就是默认的模块名称。

## **4、安装内核**

有两种方法，一种是手工的，一种是自动的。

如果是用手工的，将编译好的内核 /usr/src/kernels/linux-3.7.4/arch/x86/boot/bzImage复制到/boot/中，命名为vmlinuz-3.7.4。用命令"mkinitrd -v /boot/initrd-3.7.4.img 3.7.4"前面一个参数是生成initrd文件，后面一个参数是对应内核模块的名称，mkinitrd回去查找lib/modules/3.7.4中的模块，将需要的模块插入initrd文件中。为什么我们要制作initrd文件呢？

initrd 文件，他的目的在于提供启动过程中所需要的最重要的核心模块，以让系统启动过程可以顺利完成。 会需要 initrd 的原因，是因为核心模块放至于/lib/modules/$(uname -r)/kernel/ 当中， 这些模块必须要根目录 (/) 被挂载时才能够被读取。但是如果核心本身不具备磁盘的驱动程序时， 当然无法挂载根目录，也就没有办法取得驱动程序，因此造成两难的地步，如果没有initrd文件，启动系统时会报下面的错误。

![img](https://pic1.zhimg.com/80/v2-a9dc897ce503c0295a1e3f10a734bf44_720w.webp)

mkinitrd可以将 /lib/modules/.... 内的模块（启动过程当中一定需要的模块）包成一个文件 (就是initrd文件)， 然后在启动时透过主机的 INT 13 硬件功能将该文件读出来解压缩，并且 initrd 在内存内会模拟成为根目录， 由于此虚拟文件系统 (Initial RAM Disk) 主要包含磁盘与文件系统的模块，因此我们的核心最后就能够认识实际的磁盘， 那就能够进行实际根目录的挂载！所以说：initrd 内所包含的模块大多是与启动过程有关，而主要以文件系统及硬盘模块 (如 usb, SCSI 等) 为主！（参考鸟哥的书）

如果是自动的话，直接在 /usr/src/kernels/linux-3.7.4目录下输入"make install"就ok了。

## **5、在启动项中加载新编译的内核**

由于我用的是fedora16,所以直接输入grub2-mkconfig命令，就会在/boot/grub2/grub.cfg文件中将我们刚编译的模块作为一个启动项了。

其它系统可以通过更改/boot/grub/menu.lst文件来添加启动项。

## **6、重新启动系统**

最后就是重新启动系统，选择刚编译的内核那项启动，不过我的系统总是报下面的错误

![img](https://pic3.zhimg.com/80/v2-f74eb8dd5e44dea0806e1aa7bd52e452_720w.webp)

在网上找了半天，很多都是按我上面写的编译内核的，也没出现这个问题，不知道是不是因为我是在vmware下编译内核的原因。汗，按提示是没有找到根设备，但如果我用原本的内核启动，却不会出错，原来的内核启动也是通过uuid来找根设备的。

---

版权声明：本文为知乎博主「[极致Linux内核](https://www.zhihu.com/people/linuxwang-xian-sheng)」的原创文章，遵循CC 4.0 BY-SA版权协议，转载请附上原文出处链接及本声明。

原文链接：https://zhuanlan.zhihu.com/p/547970302