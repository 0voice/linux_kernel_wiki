* Ext2/3/4 的layout
* 文件系统的一致性： append一个文件的全流程
* 掉电与文件系统的一致性
* fsck
* 文件系统的日志
* ext4 mount选项
* 文件系统的debug和dump
* Copy On Write 文件系统： btrfs

预备知识：数据库里的transaction(事务)有什么特性？

* 原子性（Atomicity）：事务作为一个整体被执行，包含在其中的对数据库的操作要么全部被执行，要么都不执行。
* 一致性（Consistency）：事务应确保数据库的状态从一个一致状态转变为另一个一致状态。
* 持久性（Durability）：一个事务一旦提交，他对数据库的修改应该永久保存在数据库中。

## Ext2/3/4 的layout

![image](https://user-images.githubusercontent.com/87457873/127652263-8b59563d-16e5-4582-9731-514a6533b537.png)

如上图，任何一个文件，在硬盘上有inode、 datablocks，和一些元数据信息(- 描述数据的数据)。其中，inode的信息包括，inode bitmap 和 inode table。通过inode bitmap和block bitmap来描述具体的inode table 和data blocks是否被占用。inode table包括文件的 读写权限和 指针表。

![image](https://user-images.githubusercontent.com/87457873/127652286-380dd2af-a4d1-4883-aa94-8ad08b112a1e.png)

Linux对硬盘上一个文件，是分不同角度描述。创建一个文件，包括修改inode bitmap 和 block bitmap的描述。包括修改datablock和 inode bitmap的信息等等。所以“修改文件”这个操作，并不是原子的。所以存在文件系统的执行一致性的问题。

![image](https://user-images.githubusercontent.com/87457873/127652319-055ab693-25f5-44a0-8d51-65affeade3e2.png)

分Group的好处，在同一个目录下的东西，尽量放在同一个group，用来减少硬盘的来回寻道。

文件系统的一致性： append一个文件的全流程

![image](https://user-images.githubusercontent.com/87457873/127652347-245e71c8-7278-455e-b5dd-a6cfd4243510.png)
