## **1、前言**

- 环境：处理器架构：arm64内核源码：linux-5.11ubuntu版本：20.04.1代码阅读工具：vim+ctags+cscope

> 我们知道，Linux系统中我们经常将一个块设备上的文件系统挂载到某个目录下才能访问这个文件系统下的文件，但是你有没有思考过：为什么块设备挂载之后才能访问文件？挂载文件系统Linux内核到底为我们做了哪些事情？是否可以不将文件系统挂载到具体的目录下也能访问？下面，本文将详细讲解Linxu系统中，文件系统挂载的奥秘。
> 注：**本文主要讲解文件系统挂载核心逻辑，暂不涉及挂载命名空间和绑定挂载等内容（后面的内容可能会涉及），且以ext2磁盘文件系统为例讲解挂载。本专题文章分为上下两篇，上篇主要介绍挂载全貌以及具体文件系统的挂载方法，下篇介绍如何通过挂载实例关联挂载点和超级块。**

## **2、vfs 几个重要对象**

> 在这里我们不介绍整个IO栈，只说明和文件系统相关的vfs和具体文件系统层。我们知道在Linux中通过虚拟文件系统层VFS统一所有具体的文件系统，提取所有具体文件系统的共性，屏蔽具体文件系统的差异。VFS既是向下的接口（所有文件系统都必须实现该接口），同时也是向上的接口（用户进程通过系统调用最终能够访问文件系统功能）。
> 下面我们来看下，vfs中几个比较重要的结构体对象：

### **2.1file_system_type**

- 这个结构来描述一种文件系统类型，一般具体文件系统会定义这个结构，然后注册到系统中；定义了具体文件系统的挂载和卸载方法，文件系统挂载时调用其挂载方法构建超级块、跟dentry等实例。
- 文件系统分为以下几种：

1）磁盘文件系统

> 文件在非易失性存储介质上(如硬盘，flash)，掉电文件不丢失。
> 如ext2,ext4,xfs

2）内存文件系统

文件在内存上，掉电丢失。

如tmpfs

3）伪文件系统

是假的文件系统，是利用虚拟文件系统的接口（可以对用户可见如proc、sysfs,也可以对用户不可见内核可见如sockfs,bdev）。

如proc,sysfs,sockfs,bdev

4）网络文件系统

这种文件系统允许访问另一台计算机上的数据，该计算机通过网络连接到本地计算机。

如nfs文件系统

结构体定义源码路径：include/linux/fs.h +2226

### **2.2super_block**

超级块，用于描述块设备上的一个文件系统总体信息（如文件块大小，最大文件大小，文件系统魔数等），一个块设备上的文件系统可以被挂载多次，但是内存中只能有个super_block来描述（至少对于磁盘文件系统来说）。

结构体定义源码路径：include/linux/fs.h +1414

### **2.3mount**

挂载描述符，用于建立超级块和挂载点等之间的联系，描述文件系统的一次挂载，一个块设备上的文件系统可以被挂载多次，每次挂载内存中有一个mount对象描述。

结构体定义源码路径：fs/mount.h +39

### **2.4inode**

索引节点对象，**描述磁盘上的一个文件元数据**（文件属性、位置等），有些文件系统需要从块设备上读取磁盘上的索引节点,然后在内存中创建vfs的索引节点对象，一般在文件第一次打开时创建。

结构体定义源码路径：include/linux/fs.h +610

### **2.5dentry**

目录项对象，用于**描述文件的层次结构**，从而构建文件系统的目录树，文件系统将目录当作文件，目录的数据由目录项组成，而每个目录项存储一个目录或文件的名称和索引节点号等内容。每当进程访问一个目录项就会在内存中创建目录项对象（如ext2路径名查找中，通过查找父目录数据块的目录项，找到对应文件/目录的名称，获得inode号来找到对应inode）。

结构体定义源码路径：include/linux/dcache.h +90

### **2.6file**

文件对象，描述进程打开的文件，当进程打开文件时会创建文件对象加入到进程的文件打开表，通过文件描述符来索引文件对象，后面读写等操作都通过文件描述符进行（一个文件可以被多个进程打开，会由多个文件对象加入到各个进程的文件打开表，但是inode只有一个）。

结构体定义源码路径：include/linux/fs.h +915

## **3、挂载总体流程**

### **3.1系统调用处理**

> 用户执行挂载是通过系统调用路径进入内核处理，拷贝用户空间传递参数到内核，挂载委托do_mount。
> //fs/namespace.c
> SYSCALL_DEFINE5(mount

参数：

```text
dev_name 要挂载的块设备
dir_name 挂载点目录
type 文件系统类型名
flags 挂载标志
data 挂载选项
-> kernel_type = copy_mount_string(type); //拷贝文件系统类型名到内核空间
-> kernel_dev = copy_mount_string(dev_name) //拷贝块设备路径名到内核空间
-> options = copy_mount_options(data) //拷贝挂载选项到内核空间
-> do_mount(kernel_dev, dir_name, kernel_type, flags, options) //挂载委托do_mount
```

### **3.2挂载点路径查找**

```text
挂载点路径查找，挂载委托path_mount
do_mount
-> user_path_at(AT_FDCWD, dir_name, LOOKUP_FOLLOW, &path) //挂载点路径查找 查找挂载点目录的 vfsmount和dentry 存放在 path
-> path_mount(dev_name, &path, type_page, flags, data_page) //挂载委托path_mount
```

### **3.3参数合法性检查**

> 参数合法性检查， 新挂载委托do_new_mount
> path_mount
> -> 参数合法性检查
> -> 根据挂载标志调用不同函数处理 这里讲解是默认 do_new_mount

### **3.4调用具体文件系统挂载方法**

```text
do_new_mount
-> type = get_fs_type(fstype)  //根据传递的文件系统名  查找已经注册的文件系统类型
-> fc = fs_context_for_mount(type, sb_flags) //为挂载分配文件系统上下文 struct fs_context
 -> alloc_fs_context
   -> 分配fs_context fc = kzalloc(sizeof(struct fs_context), GFP_KERNEL)
   ->  设置 ... 
   ->  fc->fs_type     = get_filesystem(fs_type);  //赋值相应的文件系统类型
   ->  init_fs_context = **fc->fs_type->init_fs_context**;  //新内核使用fs_type->init_fs_context接口  来初始化文件系统上下文
    if (!init_fs_context)   //init_fs_context回掉 主要用于初始化
        init_fs_context = **legacy_init_fs_context**;    //没有 fs_type->init_fs_context接口 
   -> init_fs_context(fc)  //初始化文件系统上下文 (初始化一些回掉函数，供后续使用)
```

来看下文件系统类型没有实现init_fs_context接口的情况：

```text
//fs/fs_context.c
init_fs_context = legacy_init_fs_context
->  fc->ops = &legacy_fs_context_ops   //设置文件系统上下午操作
                    ->.get_tree               = legacy_get_tree  //操作方法的get_tree用于  读取磁盘超级块并在内存创建超级块，创建跟inode， 跟dentry
                        -> root = fc->fs_type->mount(fc->fs_type, fc->sb_flags,
                                         ¦     fc->source, ctx->legacy_data)  //调用文件系统类型的mount方法来读取并创建超级块
                        -> fc->root = root  //赋值创建好的跟dentry


有一些文件系统使用原来的接口(fs_type.mount  = xxx_mount)：如ext2,ext4等
有一些文件系统使用新的接口(fs_type.init_fs_context =  xxx_init_fs_context)：xfs， proc， sys

无论使用哪一种，都会在xxx_init_fs_contex中实现 fc->ops =  &xxx_context_ops 接口，后面会看的都会调用fc->ops.get_tree 来读取创建超级块实例
```

- 继续往下走：

```text
do_new_mount
    -> ...
    ->  fc = fs_context_for_mount(type, sb_flags) //分配 赋值文件系统上下文
    -> parse_monolithic_mount_data(fc, data)  //调用fc->ops->parse_monolithic  解析挂载选项
    -> mount_capable(fc) //检查是否有挂载权限
    -> vfs_get_tree(fc)  //fs/super.c 挂载重点   调用fc->ops->get_tree(fc) 读取创建超级块实例
    ...
```

### **3.5挂载实例添加到全局文件系统树**

```text
do_new_mount
    ...
    ->  do_new_mount_fc(fc, path, mnt_flags)  //创建mount实例 关联挂载点和超级块  添加到命名空间的挂载树中
```

下面主要看下vfs_get_tree和do_new_mount_fc：

## **4、具体文件系统挂载方法**

### **4.1vfs_get_tree**

```text
//以ext2文件系统为例
vfs_get_tree  //fs/namespace.c
-> fc->ops->get_tree(fc)
 -> legacy_get_tree   //上面分析过 fs_type->init_fs_context == NULL使用旧的接口(ext2为NULL)
  ->fc->fs_type->mount
   -> ext2_mount  //fs/ext2/super.c  调用到具体文件系统的挂载方法
```

来看下ext2对挂载的处理：

启动阶段初始化->

```text
//fs/ext2/super.c
module_init(init_ext2_fs)
init_ext2_fs
 ->init_inodecache  //创建ext2_inode_cache 对象缓存
 ->register_filesystem(&ext2_fs_type) //注册ext2的文件系统类型


static struct file_system_type ext2_fs_type = {
        .owner          = THIS_MODULE,
        .name           = "ext2",
        .mount          = ext2_mount,   //挂载时调用  用于读取创建超级块实例
        .kill_sb        = kill_block_super,  //卸载时调用  用于释放超级块
        .fs_flags       = FS_REQUIRES_DEV,  //文件系统标志为  请求块设备，文件系统在块设备上
};
```

挂载时调用->

```text
// fs/ext2/super.c
static struct dentry *ext2_mount(struct file_system_type *fs_type,           
        int flags, const char *dev_name, void *data)
{
        return mount_bdev(fs_type, flags, dev_name, data, ext2_fill_super);
}
```

ext2_mount通过调用mount_bdev来执行实际文件系统的挂载工作，**ext2_fill_super**的一个函数指针作为参数传递给**get_sb_bdev**。该函数用于填充一个超级块对象，如果内存中没有适当的超级块对象，数据就必须从硬盘读取。

mount_bdev是个公用的函数，一般磁盘文件系统会使用它来根据具体文件系统的fill_super方法来读取磁盘上的超级块并在创建内存超级块。

我们来看下mount_bdev的实现（**它执行完成之后会创建vfs的三大数据结构 super_block、根inode和根dentry **）：

### **4.2mount_bdev源码分析**

```text
//fs/super.c
mount_bdev
->bdev = blkdev_get_by_path(dev_name, mode, fs_type)  //通过要挂载的块设备路径名 获得它的块设备描述符block_device（会涉及到路径名查找和通过设备号在bdev文件系统查找block_device，block_device是添加块设备到系统时创建的）
-> s = sget(fs_type, test_bdev_super, set_bdev_super, flags | SB_NOSEC,   
         ¦bdev);  //查找或创建vfs的超级块  （会首先在文件系统类型的fs_supers链表查找是否已经读取过指定的超级块，会对比每个超级块的s_bdev块设备描述符，没有创建一个）
->  if (s->s_root) {   //超级块的根dentry是否被赋值？
  ...
 } else {   //没有赋值说明时新创建的sb
  ...
  -> sb_set_blocksize(s, block_size(bdev)) //根据块设备描述符设置文件系统块大小
  ->  fill_super(s, data, flags & SB_SILENT ? 1 : 0)  //调用传递的具体文件系统的填充超级块方法读取填充超级块等 如ext2_fill_super
  ->  bdev->bd_super = s  //块设备bd_super指向sb
 }
-> return dget(s->s_root)  //返回文件系统的根dentry
```

可以看到mount_bdev主要是：

1.**根据要挂载的块设备文件名查找到对应的块设备描述符（内核后面操作块设备都是使用块设备描述符）；**

**2.首先在文件系统类型的fs_supers链表查找是否已经读取过指定的vfs超级块，会对比每个超级块的s_bdev块设备描述符，没有创建一个vfs超级块;**

**3.新创建的vfs超级块，需要调用具体文件系统的fill_super方法来读取填充超级块。**

那么下面主要集中在具体文件系统的fill_super方法，这里是ext2_fill_super：

分析重点代码如下：

### **4.3ext2_fill_super源码分析**

```text
//fs/ext2/super.c
static int ext2_fill_super(struct super_block *sb, void *data, int silent)               
{                                                                                                          
        struct buffer_head * bh;    //缓冲区头  记录读取的磁盘超级块                                                     
        struct ext2_sb_info * sbi;   //内存的ext2 超级块信息                                                    
        struct ext2_super_block * es;  //磁盘上的  超级块信息                                               
           ...
                                                        
        sbi = kzalloc(sizeof(*sbi), GFP_KERNEL);     //分配 内存的ext2 超级块信息结构                                   
        if (!sbi)                                                                        
                goto failed;                                                             
                                                                                         
         ...                                                  
        sb->s_fs_info = sbi;     //vfs的超级块 的s_fs_info指向内存的ext2 超级块信息结构                                                   
        sbi->s_sb_block = sb_block;                                                      
                                                       
      if (!(bh = sb_bread(sb, logic_sb_block))) {  // 读取磁盘上的超级块到内存的 使用buffer_head关联内存缓冲区和磁盘扇区                                   
              ext2_msg(sb, KERN_ERR, "error: unable to read superblock");                 
              goto failed_sbi;                                                            
      }                                                                                   
                   
      es = (struct ext2_super_block *) (((char *)bh->b_data) + offset);  //转换为struct ext2_super_block 结构                 
      sbi->s_es = es; //  内存的ext2 超级块信息结构的 s_es指向真正的ext2磁盘超级块信息结构                                                                 
      sb->s_magic = le16_to_cpu(es->s_magic); //获得文件系统魔数   ext2为0xEF53                                        
                                                                                          
      if (sb->s_magic != EXT2_SUPER_MAGIC)    //验证 魔数是否正确                                           
              goto cantfind_ext2;

    blocksize = BLOCK_SIZE << le32_to_cpu(sbi->s_es->s_log_block_size); //获得磁盘读取的块大小   

                                                                                          
        /* If the blocksize doesn't match, re-read the thing.. */                         
        if (sb->s_blocksize != blocksize) {  //块大小不匹配 需要重新读取超级块                                              
                brelse(bh);                                                               
                                                                                          
                if (!sb_set_blocksize(sb, blocksize)) {                                   
                        ext2_msg(sb, KERN_ERR,                                            
                                "error: bad blocksize %d", blocksize);                    
                        goto failed_sbi;                                                  
                }                                                                         
                                                                                          
                logic_sb_block = (sb_block*BLOCK_SIZE) / blocksize;                       
                offset = (sb_block*BLOCK_SIZE) % blocksize;                               
                bh = sb_bread(sb, logic_sb_block); //重新 读取超级块                                       
                if(!bh) {                                                                 
                        ext2_msg(sb, KERN_ERR, "error: couldn't read"                     
                                "superblock on 2nd try");                                 
                        goto failed_sbi;                                                  
                }                                                                         
                es = (struct ext2_super_block *) (((char *)bh->b_data) + offset);         
                sbi->s_es = es;                                                           
                if (es->s_magic != cpu_to_le16(EXT2_SUPER_MAGIC)) {                       
                        ext2_msg(sb, KERN_ERR, "error: magic mismatch");                  
                        goto failed_mount;                                                
                }                                                                         
        }                                                                                 
                                                                                          
        sb->s_maxbytes = ext2_max_size(sb->s_blocksize_bits);  //设置最大文件大小                            
        ...                                                           
       
       //读取或设置 inode大小和第一个inode号                                                                        
       if (le32_to_cpu(es->s_rev_level) == EXT2_GOOD_OLD_REV) {                   
               sbi->s_inode_size = EXT2_GOOD_OLD_INODE_SIZE;                      
               sbi->s_first_ino = EXT2_GOOD_OLD_FIRST_INO;                        
       } else {                                                                   
               sbi->s_inode_size = le16_to_cpu(es->s_inode_size);                 
               sbi->s_first_ino = le32_to_cpu(es->s_first_ino);                   
              ...                          
       }                                                                          
                                                                                  
      ...             
                                                                                  
       sbi->s_blocks_per_group = le32_to_cpu(es->s_blocks_per_group);    //赋值每个块组 块个数          
       sbi->s_frags_per_group = le32_to_cpu(es->s_frags_per_group);               
       sbi->s_inodes_per_group = le32_to_cpu(es->s_inodes_per_group);  //赋值每个块组 inode个数             
                                                                                  
       sbi->s_inodes_per_block = sb->s_blocksize / EXT2_INODE_SIZE(sb);    //赋值每个块 inode个数       
       ...                     
       sbi->s_desc_per_block = sb->s_blocksize /                                  
                                       sizeof (struct ext2_group_desc);    //赋值每个块 块组描述符个数     
       sbi->s_sbh = bh;  //赋值读取的超级块缓冲区                                                          
       sbi->s_mount_state = le16_to_cpu(es->s_state);    //赋值挂载状态                           
     ...                                   
                                                                               
    if (sb->s_magic != EXT2_SUPER_MAGIC)                                       
            goto cantfind_ext2;                                                
   
   //一些合法性检查
     ...    
    
  //计算块组描述符 个数
  sbi->s_groups_count = ((le32_to_cpu(es->s_blocks_count) -               
                          le32_to_cpu(es->s_first_data_block) - 1)        
                                  / EXT2_BLOCKS_PER_GROUP(sb)) + 1;       
  db_count = (sbi->s_groups_count + EXT2_DESC_PER_BLOCK(sb) - 1) /        
          ¦  EXT2_DESC_PER_BLOCK(sb);                                     
  sbi->s_group_desc = kmalloc_array(db_count,                             
                                  ¦  sizeof(struct buffer_head *),        
                                  ¦  GFP_KERNEL);  //分配块组描述符 bh数组                       
   
    
    for (i = 0; i < db_count; i++) {      //读取块组描述符                                 
          block = descriptor_loc(sb, logic_sb_block, i);                
          sbi->s_group_desc[i] = sb_bread(sb, block);   //读取的 块组描述符缓冲区保存 到sbi->s_group_desc[i]              
          if (!sbi->s_group_desc[i]) {                                  
                  for (j = 0; j < i; j++)                               
                          brelse (sbi->s_group_desc[j]);                
                  ext2_msg(sb, KERN_ERR,                                
                          "error: unable to read group descriptors");   
                  goto failed_mount_group_desc;                         
          }                                                             
  }                                                                     

                                                                    
  sb->s_op = &ext2_sops;   //赋值超级块操作
  ...
 root = ext2_iget(sb, EXT2_ROOT_INO); //读取根inode  （ext2 根根inode号为2） 

 sb->s_root = d_make_root(root);  //创建根dentry  并建立根inode和根dentry关系
 ext2_write_super(sb);  //同步超级块信息到磁盘 如挂载时间等
```

可以看到ext2_fill_super主要工作为：

**1.读取磁盘上的超级块；**

**2.填充并关联vfs超级块；**

**3.读取块组描述符；**

**4.读取磁盘根inode并建立vfs 根inode;**

**5.创建根dentry关联到根inode**。

**下面给出ext2_fill_super之后ext2相关图解：**

![img](https://pic2.zhimg.com/80/v2-464f0a179a76ef8ee3283bced520126d_720w.webp)

有了这些信息，虽然能够获得块设备上的文件系统全貌，内核也能通过已经建立好的block_device等结构访问块设备，但是用户进程不能真正意义上访问到，用户一般会通过open打开一个文件路径来访问文件，但是现在并没有关联挂载目录的路径，需要将文件系统关联到挂载点，以至于路径名查找的时候查找到挂载点后，在转向文件系统的根目录，而这需要通过do_new_mount_fc来去关联并加入全局的文件系统树中,下一篇我们将做详细讲解。**4、添加到全局文件系统树**

### **4.1do_new_mount_fc**

```text
do_new_mount  //fs/namespace.c
->do_new_mount_fc
 ->     struct vfsmount *mnt;                      
    ->   struct mountpoint *mp;                     
 ->  struct super_block *sb = fc->root->d_sb;  //获得vfs的超级块 （之前已经构建好）
 >  mnt = vfs_create_mount(fc);   //为一个已配置的超级块 分配mount实例
    ->    mp = lock_mount(mountpoint);  //寻找挂载点 如果挂载目录是挂载点（已经有文件系统挂载其上），则将最后一次挂载的文件系统根目录作为挂载点    
 ->  do_add_mount(real_mount(mnt), mp, mountpoint, mnt_flags);  //关联挂载点 加入全局文件系统树
```

### **4.2vfs_create_mount源码分析**

```text
vfs_create_mount
    ->     mnt = alloc_vfsmnt(fc->source ?: "none");  //分配mount实例 
    ->     mnt->mnt.mnt_sb         = fc->root->d_sb;  //mount关联超级块 (使用vfsmount关联)
    ->    mnt->mnt.mnt_root       = dget(fc->root);   //mount关联根dentry (使用vfsmount关联)
    ->    mnt->mnt_mountpoint     = mnt->mnt.mnt_root; // mount关联挂载点 （临时指向根dentry，后面会指向真正的挂载点，以至于对用户可见）
    ->  mnt->mnt_parent         = mnt;  //父挂载指向自己 (临时指向 后面会设置)              
    ->     return &mnt->mnt;  //返回内嵌的vfsmount
```

- 注：老内核使用的是vfsmount来描述文件系统的一次挂载，现在内核都使用mount来描述，而vfsmount被内嵌到mount中，主要来描述文件系统的超级块和跟dentry。

#### 4.2.1vfs_create_mount之后vfs对象数据结构之间关系图如下：

![img](https://pic1.zhimg.com/80/v2-eb423661d85acf15649569f9c4e0d804_720w.webp)

### **4.3lock_mount源码分析**

lock_mount是最不好理解的函数，下面详细讲解：

-> mp = lock_mount(mountpoint);

//不只是加锁， 通过传来的 挂载点的 path（vfsmout, dentry二元组)，来查找最后一次挂载的文件系统的根dentry作为即将挂载文件系统的挂载点

我们看下这个函数

-> 这个函数主要从挂载点的path(即是挂载目录的path结构，如挂载到/mnt下, path为mnt的path) 来找到真正的挂载点 两种情况：

1.如果挂载点的path 是正常的目录，原来不是挂载点，则直接返回这个目录的dentry作为挂载点（mountpoint的m_dentry会指向挂载点的dentry）

2.如果挂载点的path不是正常的目录，原来就是挂载点，说明这个目录已经有其他的文件系统挂载，那么它会查找最后一个挂载到这个目录的文件系统的根dentry,作为真正的挂载点。

我们打开这个黑匣子看一下：首先传递来的path 是一个表示要解析的挂载目录[vfsmount,dentry]二元组，如我们要挂载到 /mnt （path即为<mnt所在文件系统的vfsmount, mnt的dentry>）

```text
//include/linux/path.h  描述一个路径
struct path { 
        struct vfsmount *mnt;
        struct dentry *dentry;
} __randomize_layout;

 //fs/mount.h       描述一个挂载点
struct mountpoint {       
        struct hlist_node m_hash; 
        struct dentry *m_dentry;  
        struct hlist_head m_list; 
        int m_count;              
};                                


static struct mountpoint *lock_mount(struct path *path)           
{                                                                 
        struct vfsmount *mnt;                                     
        struct dentry *dentry = path->dentry; //获得挂载目录的dentry                     
retry:                                                            
        inode_lock(dentry->d_inode);     //写方式申请 inode的读写信号量                        
        if (unlikely(cant_mount(dentry))) { //判断挂载目录能否被挂载
                inode_unlock(dentry->d_inode);                    
                return ERR_PTR(-ENOENT);                          
        }                                                         
        namespace_lock();  //写方式申请 命名空间读写信号量                                      
        mnt = lookup_mnt(path); //查找挂载在path上的第一个子mount    //！！！重点函数，后面分析 ！！！                               
        if (likely(!mnt)) { // mnt为空 说明没有文件系统挂载在这个path上  是我们要找的目标 
    
    //1.如果dentry之前是挂载点 则从mountpoint hash表 查找mountpoint （dentry计算hash）
    // 2. 如果dentry之前不是挂载点 分配mountpoint 加入mountpoint hash表（dentry计算hash）,设置dentry为挂载点
                struct mountpoint *mp = get_mountpoint(dentry); //！！！重点函数,后面会分析 ！！！
      
                if (IS_ERR(mp)) {                                 
                        namespace_unlock();                       
                        inode_unlock(dentry->d_inode);            
                        return mp;                                
                }                                                 
                return mp; //返回找到的挂载点实例 （这个挂载点的dentry之前没有被挂载）                                      
        }
        namespace_unlock(); //释放命名空间读写信号量
        inode_unlock(path->dentry->d_inode); //释放 inode的读写信号量 
        path_put(path);
        path->mnt = mnt; // path->mnt指向找到的vfsmount                                          
        dentry = path->dentry = dget(mnt->mnt_root); //path->dentry指向找到的vfsmount的根dentry！
        goto retry; //继续查找下一个挂载
}
```

#### **4.3.1get_mountpoint源码分析**

```text
static struct mountpoint *get_mountpoint(struct dentry *dentry)                      
{                                                                                    
        struct mountpoint *mp, *new = NULL;                                          
        int ret;                                                                     
                                                                                     
        if (d_mountpoint(dentry)) {  //dentry为挂载点 （当dentry为挂载点时 会设置dentry->d_flags 的DCACHE_MOUNTED标志）                                                
                /* might be worth a WARN_ON() */                                     
                if (d_unlinked(dentry))                                              
                        return ERR_PTR(-ENOENT);                                     
mountpoint:                                                                          
                read_seqlock_excl(&mount_lock);                                      
                mp = lookup_mountpoint(dentry); // 从mountpoint hash表 查找mountpoint （dentry计算hash）                                    
                read_sequnlock_excl(&mount_lock);                                    
                if (mp)                                                              
                        goto done;  //找到直接返回mountpoint实例                                               
        }                                                                            
                                                                                     
        if (!new)    //mountpoint哈希表中没有找到    则分配                                                             
                new = kmalloc(sizeof(struct mountpoint), GFP_KERNEL);             
        if (!new)                                                                    
                return ERR_PTR(-ENOMEM);                                             
                                                                                     
                                                                                     
        /* Exactly one processes may set d_mounted */                                
        ret = d_set_mounted(dentry);   //设置dentry为挂载点
   ->dentry->d_flags |= DCACHE_MOUNTED;  //设置挂载点标志很重要  路径名查找时发现为挂载点则会步进到相关文件系统的跟dentry
                                                                                     
        /* Someone else set d_mounted? */                                            
        if (ret == -EBUSY)                                                           
                goto mountpoint;                                                     
                                                                                     
        /* The dentry is not available as a mountpoint? */                           
        mp = ERR_PTR(ret);                                                           
        if (ret)                                                                     
                goto done;                                                           
                                                              
        /* Add the new mountpoint to the hash table */        
        read_seqlock_excl(&mount_lock);                       
        new->m_dentry = dget(dentry);  //设置mountpoint实例  的m_dentry 指向dentry                       
        new->m_count = 1;                                      
        hlist_add_head(&new->m_hash, mp_hash(dentry));   // mountpoint实例添加到 mountpoint_hashtable     
        INIT_HLIST_HEAD(&new->m_list);  //初始化 挂载链表   mount实例会加入到这个链表                      
        read_sequnlock_excl(&mount_lock);                     
                                                              
        mp = new;   //指向挂载点                                          
        new = NULL;                                           
done:                                                         
        kfree(new);                                           
        return mp;  //返回挂载点                                           
}
```

#### **4.3.2lookup_mnt源码分析**

它在文件系统挂载和路径名查找都会使用到，作用为查找挂载在这个path下的第一个子vfsmount实例。

->文件系统挂载场景中，使用它查找合适的vfsmount实例作为父vfsmount。

->路径名查找场景中，使用它查找一个合适的vfsmount实例作为下一级路径名解析起点的vfsmount。

```text
//fs/namespace.c
lookup_mnt(const struct path *path)
->  struct mount *child_mnt;
 struct vfsmount *m;     
 child_mnt = __lookup_mnt(path->mnt, path->dentry);  //委托__lookup_mnt
 m = child_mnt ? &child_mnt->mnt : NULL; //返回mount实例的vfsmount实例 或NULL            
 return m;

->
struct mount *__lookup_mnt(struct vfsmount *mnt, struct dentry *dentry)        
{               
        struct hlist_head *head = m_hash(mnt, dentry);  // 根据 父vfsmount实例 和 挂载点的dentry查找  mount_hashtable的一个哈希表项
        struct mount *p;
        
        hlist_for_each_entry_rcu(p, head, mnt_hash)  //从哈希表项对应的链表中查找   遍历链表的每个节点
                if (&p->mnt_parent->mnt == mnt && p->mnt_mountpoint == dentry) //节点的mount实例的父mount为mnt 且mount实例的挂载点为 dentry
                        return p; //找到返回mount实例
        return NULL;  //没找到返回NULL
}
```

#### **4.3.3lock_mount情景分析**

1)lock_mount传递的path 之前不是挂载点：

调用链为：

> lock_mount
> ->mnt = lookup_mnt(path) //没有子mount 返回NULL
> ->mp = get_mountpoint(dentry) //分配mountpoint 加入mountpoint hash表（dentry计算hash）,设置dentry为挂载点
> ->return mp //返回找到的挂载点实例

2)lock_mount传递的path 之前是挂载点：我们现在执行 mount -t ext2 /dev/sda4 /mnt

之前 /mnt的挂载情况

> mount /dev/sda1 /mnt （1）
> mount /dev/sda2 /mnt （2）
> mount /dev/sda3 /mnt （3）

调用链为：

> lock_mount
> ->mnt = lookup_mnt(path) //返回（1）的mount实例
> ->path->mnt = mnt //下一次查找的 path->mnt赋值（1）的mount实例
> ->dentry = path->dentry = dget(mnt->mnt_root) // //下一次查找path->dentry 赋值（1）的根dentry

> ->mnt = lookup_mnt(path) //返回（2）的mount实例
> ->path->mnt = mnt //下一次查找的 path->mnt赋值（2）的mount实例
> ->dentry = path->dentry = dget(mnt->mnt_root) // //下一次查找path->dentry 赋值（2）的根dentry
>
> ->mnt = lookup_mnt(path) //返回（3）的mount实例

> ->path->mnt = mnt //下一次查找的 path->mnt赋值（3）的mount实例
> ->dentry = path->dentry = dget(mnt->mnt_root) // //下一次查找path->dentry 赋值（3）的根dentry
> -> mnt = lookup_mnt(path) //没有子mount 返回NULL
> ->mp = get_mountpoint(dentry) //分配mountpoint 加入mountpoint hash表（dentry计算hash）,设置dentry为挂载点（（3）的根dentry作为挂载点）

> ->return mp //返回找到的挂载点实例(也就是最后一次挂载（3） 文件系统的根dentry)

#### **4.3.4do_add_mount源码分析**

准备好了挂载点之后，接下来子mount实例关联挂载点以及添加子mount实例到全局的文件系统挂载树中。

```text
do_add_mount  //添加mount到全局的文件系统挂载树中
->struct mount *parent = real_mount(path->mnt); //获得父挂载点的挂载实例
->graft_tree(newmnt, parent, mp)
 -> mnt_set_mountpoint(dest_mnt, dest_mp, source_mnt)
  ->   child_mnt->mnt_mountpoint = mp->m_dentry;   //关联子mount到挂载点的dentry                 
    child_mnt->mnt_parent = mnt; //子mount->mnt_parent指向父mount
    child_mnt->mnt_mp = mp; //子mount->mnt_mp指向挂载点
    hlist_add_head(&child_mnt->mnt_mp_list, &mp->m_list); //mount添加到挂载点链表

 ->commit_tree //提交挂载树
  ->__attach_mnt(mnt, parent)
   ->  hlist_add_head_rcu(&mnt->mnt_hash,
       ¦  m_hash(&parent->mnt, mnt->mnt_mountpoint));  //添加到mount hash表 ，通过父挂载点的vfsmount和挂载点的dentry作为索引(如上面示例中的<(3)的vfsmount , (3)的根dentry>)
    list_add_tail(&mnt->mnt_child, &parent->mnt_mounts);  //添加到父mount链表
```

上面说了一大堆，主要为了实现：

**将mount实例与挂载点联系起来（会将mount实例加入到mount 哈希表，父文件系统的vfsmount和真正的挂载点的dentry组成的二元组为索引，路径名查找时便于查找），以及mount实例与文件系统的跟dentry联系起来（路径名查找的时候便于沿着跟dentry来访问这个文件系统的所有文件）。**

#### 4.3.5do_add_mount 之后vfs对象数据结构之间关系图（/mnt之前不是挂载点情况）如下：

![img](https://pic1.zhimg.com/80/v2-72f3ca5ef905b57826560d45046eab94_720w.webp)

## **5、mount的应用**

- 上面几章我们分析了文件系统挂载的主要流程，创建并关联了各个vfs的对象，为了打开文件等路径名查找时做准备。

### **5.1路径名查找到挂载点源码分析**

```text
//fs/namei.c 查找一个路径分量
walk_component  
->step_into
 ->handle_mounts
  ->traverse_mounts
   ->flags = smp_load_acquire(&path->dentry->d_flags); //获得dentry标志
   ->__traverse_mounts(path, flags, jumped, count, lookup_flags); //查找挂载点  返回不再是挂载点的path
     -> {
     
       while (flags & DCACHE_MANAGED_DENTRY) {    //找到的dentry是挂载点 则继续查找 ，不是则退出循环
        ...
        if (flags & DCACHE_MOUNTED) {   // something's mounted on it..  是挂载点 和上面lock_mount分析逻辑类似 
         //不断的查找挂载点  直到最后查找到的dentry不是挂载点 退出循环
         struct vfsmount *mounted = lookup_mnt(path);  //查找第一个子mount          
         if (mounted) {          // ... in our namespace         
           dput(path->dentry);                            
           if (need_mntput)
             mntput(path->mnt);
           path->mnt = mounted; //path->mnt赋值子mount
           path->dentry = dget(mounted->mnt_root); //path->dentry 赋值子mount的根dentry （是挂载点就会步进到挂载点的跟dentry）
           // here we know it's positive
           flags = path->dentry->d_flags; //获得dentry标志
           need_mntput = true;
           continue; //继续查找
        }
        ...

       }
       
       //最终返回找到的path（它不再是挂载点），后面继续查找下一个路径
      }
```

### **5.2举例说明**

> 我们做以下的路径名查找：/mnt/test/text.txt
> /mnt/ 目录挂载情况为
> mount /dev/sda1 /mnt （1）
> mount /dev/sda2 /mnt （2）

> mount /dev/sda3 /mnt （3）
> test/text.txt文件在 /dev/sda3 上

则路径名查找时，查找到mnt的dentry发现它是挂载点，就会依次查找（1）的根目录->（2）的根目录 ->（3）的根目录， 最终将（3）的vfsmount和 根目录的dentry 填写到path，进行下一步的查找, 最终查找到/dev/sda3 上的text.txt文件。

注：一个目录被文件系统挂载时，原来目录中包含的其他子目录或文件会被隐藏。

## **6、挂载图解**

为了便于讲解图示中各个实例表示如下：

> Xyn ---> X表示哪个实例对象 如：mount实例使用M表示（第一个大小字母） dentry使用D表示 inode使用I表示 super_block用S表示 vfsmount使用V表示
> y表示是父文件系统中的实例对象还是子文件系统中 如：p（parent）表示父文件系统中实例对象 c（child）表示子文件系统中实例对象
> n 区分同一种对象的不同实例
> 例如：Dc1 表示子文件系统中一个dentry对象

### **6.1mount、super_block、file_system_type三者关系图解**

![img](https://pic1.zhimg.com/80/v2-095bb1781dfa095aae41172484c23e3c_720w.webp)

解释：mount实例、super_block实例、file_system_type实例三种层级逐渐升高，即一个file_system_type实例会包含多个super_block实例，一个super_block实例会包含多个mount实例。一种file_system_type必须先被注册到系统中来宣誓这种文件系统存在，主要提供此类文件系统的挂载和卸载方法等，注册即是加入全局的file_systems链表，等到有块设备上的文件系统要挂载时就会根据挂载时传递的文件系统类型名查找file_system_type实例，如果查找到，就会调用它的挂载方法进行挂载。首先，在file_systems实例的super_block链表中查找有没有super_block实例已经被创建，如果有就不需要从磁盘读取（这就是一个块设备上的文件系统挂载到多个目录上只有一个super_block实例的原因），如果没有从磁盘读取并加入对应的file_systems实例的super_block链表。而每次挂载都会创建一个mount实例来联系挂载点和super_block实例，并以（父vfsmount,挂载点dentry）为索引加入到全局mount哈希表，便于后面访问这个挂载点的文件系统时的路径名查找。

### **6.2父子文件系统挂载关系图解**

![img](https://pic4.zhimg.com/80/v2-c88da59ea00bb451ddceb1cbf060d59f_720w.webp)

解释：图中/dev/sda1中的子文件系统挂载到父文件系统的/mnt目录下。当挂载的时候会创建mount、super_block、跟inode、跟dentry四大数据结构并建立相互关系，将子文件系统的mount加入到(Vp, Dp3)二元组为索引的mount哈希表中，通过设置mnt的目录项（Dp3）的DCACHE_MOUNTED来将其标记为挂载点，并与父文件系统建立亲缘关系挂载就完成了。

当需要访问子文件系统中的某个文件时，就会通过路径名各个分量解析到mnt目录，发现其为挂载点，就会通过(Vp, Dp3)二元组在mount哈希表中找到子文件系统的mount实例(Mc),然后就会从子文件系统的跟dentry（Dc1）开始往下继续查找，最终访问到子文件系统上的文件。

### **6.3单个文件系统多挂载点关系图解**

![img](https://pic1.zhimg.com/80/v2-8a4deddc3f1722a73f4d113b7d64ead8_720w.webp)

解释：图中将/dev/sda1中的文件系统分别挂载到父文件系统的/mnt/a和/mnt/b目录下。当第一次挂载到/mnt/a时，会创建mount、super_block、跟inode、跟dentry四大数据结构(分别对应与Mc1、Sc、Dc1、Ic)并建立相互关系，将子文件系统的Mc1加入到(Vp, Dp3)二元组为索引的mount哈希表中，通过设置/mnt/a的目录项的DCACHE_MOUNTED来将其标记为挂载点，并与父文件系统建立亲缘关系挂载就完成了。然后挂载到/mnt/b时, Sc、Dc1、Ic已经创建好不需要再创建，内存中只会有一份，会创建Mc2来关联super_block和第二次的挂载点，建立这几个数据结构关系，将子文件系统的Mc2加入到(Vp, Dp4)二元组为索引的mount哈希表中，通过设置/mnt/b的目录项的DCACHE_MOUNTED来将其标记为挂载点，并与父文件系统建立亲缘关系挂载就完成了。

当需要访问子文件系统中的某个文件时，就会通过路径名各个分量解析到/mnt/a目录，发现其为挂载点，就会通过(Vp, Dp3)在mount哈希表中找到子文件系统的Mc1,然后就会从子文件系统的Dc1开始往下继续查找，最终访问到子文件系统上的文件。同样，如果解析到/mnt/b目录,发现其为挂载点，就会通过(Vp, Dp4)在mount哈希表中找到子文件系统的Mc2,然后就会从子文件系统的Dc1开始往下继续查找，最终访问到子文件系统上的文件。可以发现，同一个块设备上的文件系统挂载到不同的目录上，相关联的super_block和跟dentry是一样的，这保证了无论从哪个挂载点开始路径名查找都访问到的是同一个文件系统上的文件。

### **6.4多文件系统单挂载点关系图解**

![img](https://pic4.zhimg.com/80/v2-03e69e56d7942f240dad3df406131303_720w.webp)

解释：最后我们来看多文件系统单挂载点的情况，图中先将块设备/dev/sda1中的子文件系统1挂载到/mnt目录，然后再将块设备/dev/sdb1中的子文件系统2挂载到/mnt目录上。

当子文件系统1挂载的时候，会创建mount、super_block、跟inode、跟dentry四大数据结构(分别对应与Mc1、Sc1、Dc1、Ic1)并建立相互关系，将子文件系统的Mc1加入到(Vp, Dp3)二元组为索引的mount哈希表中，通过设置/mnt的目录项的DCACHE_MOUNTED来将其标记为挂载点，并与父文件系统建立亲缘关系挂载就完成了。

当子文件系统2挂载的时候，会创建mount、super_block、跟inode、跟dentry四大数据结构(分别对应与Mc2、Sc2、Dc4、Ic2)并建立相互关系，这个时候会发现/mnt目录是挂载点，则会将子文件系统1的根目录（Dc1）作为文件系统2的挂载点，将子文件系统的Mc2加入到(Vc1, Dc1)二元组为索引的mount哈希表中，通过设置Dc1的DCACHE_MOUNTED来将其标记为挂载点，并与父文件系统建立亲缘关系挂载就完成了。

这个时候，子文件系统1已经被子文件系统2隐藏起来了，当路径名查找到/mnt目录时，发现其为挂载点，则通过(Vp, Dp3)二元组为索引在mount哈希表中找到Mc1，会转向文件系统1的跟目录（Dc1）开始往下继续查找，发现Dc1也是挂载点，则(通过Vc1, Dc1)二元组为索引在mount哈希表中找到Mc2, 会转向文件系统1的跟目录（Dc4）开始往下继续查找，于是就访问到了文件系统2中的文件。除非，文件系统2被卸载，文件系统1的跟dentry（Dc1）不再是挂载点，这个时候文件系统1中的文件才能再次被访问到。

## **7、总结**

Linux中，块设备上的文件系统只有挂载到内存的目录树中的一个目录下，用户进程才能访问，而挂载是创建数据结构关联块设备上的文件系统和挂载点，使得路径名查找的时候能够通过挂载点目录访问到挂载在其下的文件系统。

### **7.1挂载主要步骤**

1.vfs_get_tree 调用具体文件系统的获取填充超级块方法(fs_context_operations.get_tree或者file_system_type.mount)， 在内存构建super_block，然后构建根inode和根dentry(磁盘文件系统可能需要从磁盘读取磁盘超级块构建内存的super_block，从磁盘读取根inode构建内存的inode)。2.do_new_mount_fc 对于每次挂载都会分配mount实例，用于关联挂载点到文件系统。当一个要挂载的目录不是挂载点，会设置这个目录的dentry为挂载点，然后mount实例记录这个挂载点。当一个要挂载的目录是挂载点（之前已经有文件系统被挂载到这个目录），那么新挂载的文件系统将挂载到这个目录最后一次挂载的文件系统的根dentry,之前挂载的文件系统的文件都被隐藏（当子挂载被卸载，原来的文件系统的文件才可见）。

### **7.2文件系统的用户可见性**

**只对内核内部可见：**不需要将文件系统关联到一个挂载点，内核通过文件系统的super_block等结构即可访问到文件系统的文件（如bdev，sockfs）。

**对于用户可见：**需要将文件系统关联到一个挂载点，就需要通过给定的挂载点目录名找到真正的挂载点，然后进行挂载操作， 挂载的实质是：通过mount实例的mnt_mountpoint关联真正的挂载点dentry,然后建立父mount关系，mount实例加入到全局的mount hash table（通过父vfsmount和真正的挂载点dentry作为hash索引），然后用户打开文件的时候通过路径名查找解析各个目录分量，当发现一个目录是挂载点时，就会步进到最后一次挂载到这个目录的文件系统的根dentry中继续查找，知道根dentry就可以继续查找到这个文件系统的任何文件。

### **7.3几条重要规律**

1）文件系统被挂载后都会有以下几大vfs对象被创建：

super_block
mount

根inode

根dentry

注：其中mount为纯软件构造的对象（内嵌vfsmount对象），其他对象视文件系统类型，可能涉及到磁盘操作。

super_block 超级块实例，描述一个文件系统的信息，有的需要磁盘读取在内存中填充来构建（如磁盘文件系统），有的直接内存中填充来构建。

mount 挂载实例，描述一个文件系统的一次挂载，主要关联一个文件系统到挂载点，为路径名查找做重要准备工作。

根inode 每个文件系统都会有根inode，有的需要磁盘读取在内存中填充来构建（如磁盘文件系统，根inode号已知），有的直接内存中填充来构建。

根dentry 每个文件系统都会有根dentry，根据根inode来构建，路径名查找时会步进到文件系统的根dentry来访问这个文件系统的文件。

2）一个目录可以被多个文件系统挂载。第一次挂载是直接挂载这个目录上，新挂载的文件系统实际上是挂载在上一个文件系统的根dentry上。

3）一个目录被多个文件系统挂载时，新挂载导致之前的挂载被隐藏。

4）一个目录被文件系统挂载时，原来目录中包含的其他子目录或文件被隐藏。

5）每次挂载都会有一个mount实例描述本次挂载。

6）一个快设备上的文件系统可以被挂载到多个目录，有多个mount实例，但是只会有一个super_block、根dentry 和根inode。

7）mount实例用于关联挂载点dentry和文件系统，起到路径名查找时“路由”的作用。

8）挂载一个文件系统必须保证所要挂载的文件系统类型已经被注册。

9）挂载时会查询文件系统类型的fs_type->fs_supers链表，检查是否已经有super_block被加入链表，如果没有才会分配并读磁盘超级块填充。

10）对象层次：一个fs_type->fs_supers链表可以挂接属于同一个文件系统的被挂载的超级块，超级块链表可以挂接属于同一个超级块的mount实例 fs_type -> super_block -> mount 从高到低的包含层次。

------

版权声明：本文为知乎博主「Linux内核库」的原创文章，遵循CC 4.0 BY-SA版权协议，转载请附上原文 出处链接及本声明。 

原文链接：https://zhuanlan.zhihu.com/p/515769749
