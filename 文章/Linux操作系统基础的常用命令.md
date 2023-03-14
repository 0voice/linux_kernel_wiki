## 1，Linux简介

Linux是一种自由和开放源码的操作系统，存在着许多不同的Linux版本，但它们都使用了Linux内核。Linux可安装在各种计算机硬件设备中，比如手机、平板电脑、路由器、台式计算机。

### 1.1Linux介绍

Linux出现于1991年，是由芬兰赫尔辛基大学学生Linus Torvalds和后来加入的众多爱好者共同开发完成

### 1.2Linux特点

多用户，多任务，丰富的网络功能，可靠的系统安全，良好的可移植性，具有标准兼容性，良好的用户界面，出色的速度性能

### 1.3开源

CentOS

1. 主流：目前的Linux操作系统主要应用于生产环境，主流企业级Linux系统仍旧是RedHat或者CentOS
2. 免费：RedHat 和CentOS差别不大，基于Red Hat Linux 提供的可自由使用源代码的企业CentOS是一个级Linux发行版本
3. 更新方便：CentOS独有的yum命令支持在线升级，可以即时更新系统，不像RedHat 那样需要花钱购买支持服务！

### 1.4Linux目录结构

![img](https://pic2.zhimg.com/80/v2-0ba0cffb7f05d4eccec30210c8e64851_720w.webp)

- bin (binaries)存放二进制可执行文件
- sbin (super user binaries)存放二进制可执行文件，只有root才能访问
- etc (etcetera)存放系统配置文件
- usr (unix shared resources)用于存放共享的系统资源
- home 存放用户文件的根目录
- root 超级用户目录
- dev (devices)用于存放设备文件
- lib (library)存放跟文件系统中的程序运行所需要的共享库及内核模块
- mnt (mount)系统管理员安装临时文件系统的安装点
- boot 存放用于系统引导时使用的各种文件
- tmp (temporary)用于存放各种临时文件
- var (variable)用于存放运行时需要改变数据的文件

## 2，Linux常用命令

### 2.1命令格式：命令 -选项 参数

```cpp
如：ls  -la  /usr

ls：显示文件和目录列表(list)
```

**常用参数：**

```text
-l		(long)
-a	(all)         注意隐藏文件、特殊目录.和..   
-t		(time)
```

### 2.2Linux命令的分类

内部命令：属于Shell解析器的一部分

```text
cd 切换目录（change directory）
pwd 显示当前工作目录（print working directory）
help 帮助
```

**外部命令：独立于Shell解析器之外的文件程序**

```text
ls 显示文件和目录列表（list）
mkdir 创建目录（make directoriy）
cp 复制文件或目录（copy）
```

**查看帮助文档**

```text
内部命令：help + 命令（help cd）
外部命令：man + 命令（man ls）
```

### 2.3操作文件或目录常用命令

```cpp
pwd 显示当前工作目录（print working directory）
touch 创建空文件				                    
mkdir 创建目录（make directoriy）
-p 父目录不存在情况下先生成父目录 （parents）            
cp 复制文件或目录（copy）
-r 递归处理，将指定目录下的文件与子目录一并拷贝（recursive）     
mv 移动文件或目录、文件或目录改名（move）

rm 删除文件（remove）
-r 同时删除该目录下的所有文件（recursive）
-f 强制删除文件或目录（force）
rmdir 删除空目录（remove directoriy）
cat显示文本文件内容 （catenate）
more、less 分页显示文本文件内容
head、tail查看文本中开头或结尾部分的内容
head -n  5  a.log 查看a.log文件的前5行
tail  -F b.log 循环读取（follow）
```

**常用命令**

```text
wc 统计文本的行数、字数、字符数（word count）
-m 统计文本字符数
-w 统计文本字数
-l 统计文本行数
find 在文件系统中查找指定的文件
find /etc/ -name "aaa"
grep 在指定的文本文件中查找指定的字符串
ln 建立链接文件（link）
-s 对源文件建立符号连接，而非硬连接（symbolic）

top 显示当前系统中耗费资源最多的进程 
ps 显示瞬间的进程状态
-e /-A 显示所有进程，环境变量
-f 全格式
-a 显示所有用户的所有进程（包括其它用户）
-u 按用户名和启动时间的顺序来显示进程
-x 显示无控制终端的进程
kill 杀死一个进程
kill -9 pid
df 显示文件系统磁盘空间的使用情况

du 显示指定的文件（目录）已使用的磁盘空间的总
-h文件大小以K，M，G为单位显示（human-readable）
-s只显示各档案大小的总合（summarize）
free 显示当前内存和交换空间的使用情况 
netstat 显示网络状态信息
-a 显示所有连接和监听端口
-t (tcp)仅显示tcp相关选项
-u (udp)仅显示udp相关选项
-n 拒绝显示别名，能显示数字的全部转化成数字。
-p 显示建立相关链接的程序名
ifconfig 网卡网络配置详解 
ping 测试网络的连通性 
```

**备份压缩命令**

```text
gzip 压缩（解压）文件或目录，压缩文件后缀为gz 
bzip2 压缩（解压）文件或目录，压缩文件后缀为bz2 
tar 文件、目录打（解）包
```

**gzip命令**

```text
命令格式：gzip [选项] 压缩（解压缩）的文件名
-d将压缩文件解压（decompress）
-l显示压缩文件的大小，未压缩文件的大小，压缩比（list）
-v显示文件名和压缩比（verbose）
-num用指定的数字num调整压缩的速度，-1或--fast表示最快压缩方法（低压缩比），-9或--best表示最慢压缩方法（高压缩比）。系统缺省值为6
```

**bzip2命令**

```text
命令格式：bzip2 [-cdz] 文档名
-c将压缩的过程产生的数据输出到屏幕上
-d解压缩的参数（decompress）
-z压缩的参数（compress）
-num 用指定的数字num调整压缩的速度，-1或--fast表示最快压缩方法（低压缩比），-9或--best表示最慢压缩方法（高压缩比）。系统缺省值为6
```

**tar命令**

```text
-c 建立一个压缩文件的参数指令（create）
-x 解开一个压缩文件的参数指令（extract）
-z 是否需要用 gzip 压缩
-j 是否需要用 bzip2 压缩
-v 压缩的过程中显示文件（verbose）
-f 使用档名，在 f 之后要立即接档名（file）
```

**关机/重启命令**

```text
shutdown系统关机 
-r 关机后立即重启
-h 关机后不重新启动
halt 关机后关闭电源 shutdown -h
reboot 重新启动 shutdown -r
```

**学习Linux的好习惯**

- 善于查看man page（manual）等帮助文档
- 利用好Tab键
- 掌握好一些快捷键

```text
 ctrl + c（停止当前进程）
 ctrl + r（查看命令历史）
 ctrl + l（清屏，与clear命令作用相同）
```

---

版权声明：本文为知乎博主「玩转Linux内核」的原创文章，遵循CC 4.0 BY-SA版权协议，转载请附上原文出处链接及本声明。
原文链接：https://zhuanlan.zhihu.com/p/434528439