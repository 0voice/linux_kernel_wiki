内核源码：linux-2.6.38.8.tar.bz2

目标平台：ARM体系结构

sysfs是基于内存的文件系统，用于向用户空间导出内核对象并且能对其进行读写。

1、sysfs文件系统不支持特殊文件，只支持目录、普通文件（文本或二进制文件）和符号链接文件等三种类型，在内核中都使用struct sysfs_dirent结构体来表示，相当于其他文件系统在硬盘或flash里的数据。源代码如下所示：

```text
/* fs/sysfs/sysfs.h */
struct sysfs_dirent {
	atomic_t		s_count;	//struct sysfs_dirent结构体实例自身的引用计数
	atomic_t		s_active;	//struct sysfs_elem_*所涉及的外部对象的引用计数
#ifdef CONFIG_DEBUG_LOCK_ALLOC
	struct lockdep_map	dep_map; //死锁检测模块，针对s_attr.attr中的key或skey
#endif
	struct sysfs_dirent	*s_parent;	//指向父节点
	struct sysfs_dirent	*s_sibling; //同级节点链表，插入到父节点的s_dir.children链表中
	const char		*s_name;	//文件名
 
	const void		*s_ns;		//命名空间
	union {
		struct sysfs_elem_dir		s_dir;		//目录
		struct sysfs_elem_symlink	s_symlink;	//符号链接文件
		struct sysfs_elem_attr		s_attr;		//文本文件
		struct sysfs_elem_bin_attr	s_bin_attr;	//二进制文件
	};
 
	unsigned int		s_flags;	//标志，表示struct sysfs_dirent类型、命名空间类型等信息
	unsigned short		s_mode;		//文件访问权限，包含文件类型信息
	ino_t			s_ino;			//对应于i节点号
	struct sysfs_inode_attrs *s_iattr;	//文件属性
};
 
struct sysfs_inode_attrs {
	struct iattr	ia_iattr;	//文件属性
	void		*ia_secdata;	//安全检测模块所用数据
	u32		ia_secdata_len;		//ia_secdata所指数据的长度
};
 
/* include/linux/fs.h */
struct iattr {
	unsigned int	ia_valid;	//文件属性标志
	umode_t		ia_mode;		//文件类型及其访问权限
	uid_t		ia_uid;			//用户ID
	gid_t		ia_gid;			//组ID
	loff_t		ia_size;		//文件大小
	struct timespec	ia_atime;	//访问时间
	struct timespec	ia_mtime;	//数据修改时间
	struct timespec	ia_ctime;	//元数据修改时间
 
	struct file	*ia_file;	//辅助信息，用于想实现ftruncate等方法的文件系统
};
```



[struct](https://link.zhihu.com/?target=https%3A//so.csdn.net/so/search%3Fq%3Dstruct%26spm%3D1001.2101.3001.7020) sysfs_dirent分为四种类型，如下所示：

```text
/* fs/sysfs/sysfs.h */
struct sysfs_elem_dir {
	struct kobject		*kobj;	//指向内核对象
	//子节点链表，只有目录才有可能包含文件或子目录
	//子节点通过s_sibling成员以s_ino成员升序的方式插入到该链表
	struct sysfs_dirent	*children;
};
 
struct sysfs_elem_symlink {
	struct sysfs_dirent	*target_sd;	//指向所链接的节点
};
 
struct sysfs_elem_attr {
	struct attribute	*attr;	//对象属性
	struct sysfs_open_dirent *open;	//文件open信息，其中buffers成员是struct sysfs_buffer的链表
};
 
struct sysfs_elem_bin_attr {
	struct bin_attribute	*bin_attr;	//二进制的对象属性
	struct hlist_head	buffers; //struct bin_buffer的链表
};
```

struct sysfs_open_dirent等结构体的详细信息后文说明。

2、sysfs文件系统的初始化是由sysfs_init函数来完成的（该函数由vfs_caches_init函数所调用的mnt_init函数调用），执行文件系统的注册和挂载，与rootfs根文件系统的相关操作类似。源代码如下所示：

```text
/* fs/sysfs/mount.c */
static struct vfsmount *sysfs_mnt;
struct kmem_cache *sysfs_dir_cachep;
 
int __init sysfs_init(void)
{
	int err = -ENOMEM;
 
	sysfs_dir_cachep = kmem_cache_create("sysfs_dir_cache",
					      sizeof(struct sysfs_dirent),
					      0, 0, NULL);  //创建用于struct sysfs_dirent的高速缓存
	if (!sysfs_dir_cachep)
		goto out;
	
	//初始化后备存储介质相关结构体struct backing_dev_info（sysfs文件系统基于内存，无须数据同步）
	err = sysfs_inode_init(); 
	if (err)
		goto out_err;
 
	err = register_filesystem(&sysfs_fs_type); //注册文件系统
	if (!err) { //成功返回零
		sysfs_mnt = kern_mount(&sysfs_fs_type); //挂载文件系统，不过没有将sysfs_mnt所指的结构体实例插入到挂载树中
		if (IS_ERR(sysfs_mnt)) {
			printk(KERN_ERR "sysfs: could not mount!\n");
			err = PTR_ERR(sysfs_mnt);
			sysfs_mnt = NULL;
			unregister_filesystem(&sysfs_fs_type);
			goto out_err;
		}
	} else
		goto out_err;
out:
	return err;
out_err:
	kmem_cache_destroy(sysfs_dir_cachep);
	sysfs_dir_cachep = NULL;
	goto out;
}
```

在用户空间一般都将sysfs文件系统挂载在/sys目录，而这里也有一次通过kern_mount函数的挂载，这样的话 sysfs文件系统就会挂载两次？其实是没有的，后者的挂载并没有将当前的struct vfsmount结构体实例插入到挂载树中，而是保存在全局指针sysfs_mnt中，并且会与用户空间挂载sysfs文件系统时所创建的struct vfsmount结构体实例共享相同的超级块。因此，无论用户空间的挂载操作是否执行，sysfs文件系统都会存在于内核之中（编译内核有配置CONFIG_SYSFS选项）。sysfs文件系统的struct file_system_type结构体实例如下所示：

```text
/* fs/sysfs/mount.c */
static struct file_system_type sysfs_fs_type = {
	.name		= "sysfs",
	.mount		= sysfs_mount,
	.kill_sb	= sysfs_kill_sb,
};
 
static struct dentry *sysfs_mount(struct file_system_type *fs_type,
	int flags, const char *dev_name, void *data)
{
	struct sysfs_super_info *info;
	enum kobj_ns_type type;
	struct super_block *sb;
	int error;
 
	info = kzalloc(sizeof(*info), GFP_KERNEL); //分配私有数据所用内存
	if (!info)
		return ERR_PTR(-ENOMEM);
	
	//构建超级块私有数据，用于sysfs_test_super和sysfs_set_super函数
	for (type = KOBJ_NS_TYPE_NONE; type < KOBJ_NS_TYPES; type++) 
		info->ns[type] = kobj_ns_current(type);
 
	sb = sget(fs_type, sysfs_test_super, sysfs_set_super, info); //查找或创建超级块
	if (IS_ERR(sb) || sb->s_fs_info != info)  
		kfree(info);
	if (IS_ERR(sb))
		return ERR_CAST(sb);
	if (!sb->s_root) { //如果根目录项为空指针，则说明超级块sb是新创建的
		sb->s_flags = flags;
		error = sysfs_fill_super(sb, data, flags & MS_SILENT ? 1 : 0); //填充超级块
		if (error) {
			deactivate_locked_super(sb);
			return ERR_PTR(error);
		}
		sb->s_flags |= MS_ACTIVE;
	}
 
	return dget(sb->s_root);
}
```



其中，sysfs_test_super函数用于判断struct sysfs_super_info结构体数据是否相同以便确认是否可以共享超级块，源代码如下所示：

```text
/* include/linux/kobject_ns.h */
enum kobj_ns_type {
	KOBJ_NS_TYPE_NONE = 0,
	KOBJ_NS_TYPE_NET,
	KOBJ_NS_TYPES
};
 
/* fs/sysfs/sysfs.h */
struct sysfs_super_info {
	const void *ns[KOBJ_NS_TYPES];
};
 
/* fs/sysfs/mount.c */
static int sysfs_test_super(struct super_block *sb, void *data)
{
	struct sysfs_super_info *sb_info = sysfs_info(sb); //sb->s_fs_info
	struct sysfs_super_info *info = data;
	enum kobj_ns_type type;
	int found = 1;
 
	for (type = KOBJ_NS_TYPE_NONE; type < KOBJ_NS_TYPES; type++) {
		if (sb_info->ns[type] != info->ns[type]) //只要有任何一项的值不相同则函数返回0
			found = 0;
	}
	return found;
}
```

sysfs_fill_super函数主要用于创建sysfs文件系统根目录所对应的目录项及其i节点，并且将sysfs_root作为根目录项的私有数据。源代码如下所示：

```text
/* fs/sysfs/mount.c */
static int sysfs_fill_super(struct super_block *sb, void *data, int silent)
{
	struct inode *inode;
	struct dentry *root;
 
	sb->s_blocksize = PAGE_CACHE_SIZE; //与内存页大小相同
	sb->s_blocksize_bits = PAGE_CACHE_SHIFT;
	sb->s_magic = SYSFS_MAGIC;
	sb->s_op = &sysfs_ops;
	sb->s_time_gran = 1;

	//创建并初始化根目录的i节点
	mutex_lock(&sysfs_mutex);
	inode = sysfs_get_inode(sb, &sysfs_root);
	mutex_unlock(&sysfs_mutex);
	if (!inode) {
		pr_debug("sysfs: could not get root inode\n");
		return -ENOMEM;
	}

	//创建并初始化sysfs文件系统的根目录项并关联根目录的i节点
	root = d_alloc_root(inode);
	if (!root) {
		pr_debug("%s: could not get root dentry!\n",__func__);
		iput(inode);
		return -ENOMEM;
	}
	root->d_fsdata = &sysfs_root;  //根目录项的d_fsdata成员指向sysfs文件系统的根数据项sysfs_root
	sb->s_root = root;
	return 0;
}
 
struct sysfs_dirent sysfs_root = {
	.s_name		= "",
	.s_count	= ATOMIC_INIT(1),
	.s_flags	= SYSFS_DIR | (KOBJ_NS_TYPE_NONE << SYSFS_NS_TYPE_SHIFT),
	.s_mode		= S_IFDIR | S_IRWXU | S_IRUGO | S_IXUGO, //目录文件，访问权限0755
	.s_ino		= 1,   //起始i节点号为1
};
 
static const struct super_operations sysfs_ops = { //超级块操作函数
    .statfs        = simple_statfs,
    .drop_inode    = generic_delete_inode,
    .evict_inode    = sysfs_evict_inode,
};
```

3、文件系统最核心的内容是要看其i节点是如何构建的，sysfs文件系统使用sysfs_init_inode函数来创建。源代码如下所示：

```text
/* fs/sysfs/inode.c */
static void sysfs_init_inode(struct sysfs_dirent *sd, struct inode *inode)
{
	struct bin_attribute *bin_attr;
 
	inode->i_private = sysfs_get(sd); //指向引用计数递增之后的sd
	inode->i_mapping->a_ops = &sysfs_aops; //地址空间操作函数
	inode->i_mapping->backing_dev_info = &sysfs_backing_dev_info; //后备存储介质的相关信息
	inode->i_op = &sysfs_inode_operations;
 
	set_default_inode_attr(inode, sd->s_mode);
	sysfs_refresh_inode(sd, inode);

	switch (sysfs_type(sd)) { //struct sysfs_dirent类型
	case SYSFS_DIR: //目录
		inode->i_op = &sysfs_dir_inode_operations;
		inode->i_fop = &sysfs_dir_operations;
		break;
	case SYSFS_KOBJ_ATTR: //文本文件
		inode->i_size = PAGE_SIZE; //文件大小固定为一页内存
		inode->i_fop = &sysfs_file_operations;
		break;
	case SYSFS_KOBJ_BIN_ATTR: //二进制文件
		bin_attr = sd->s_bin_attr.bin_attr;
		inode->i_size = bin_attr->size;
		inode->i_fop = &bin_fops;
		break;
	case SYSFS_KOBJ_LINK: //符号链接文件
		inode->i_op = &sysfs_symlink_inode_operations;
		break;
	default:
		BUG();
	}
 
	unlock_new_inode(inode);
}
 
static inline void set_default_inode_attr(struct inode * inode, mode_t mode)
{
	inode->i_mode = mode;
	inode->i_atime = inode->i_mtime = inode->i_ctime = CURRENT_TIME;
}
 
static void sysfs_refresh_inode(struct sysfs_dirent *sd, struct inode *inode)
{
	struct sysfs_inode_attrs *iattrs = sd->s_iattr;
 
	inode->i_mode = sd->s_mode;
	if (iattrs) { //sd->s_iattr为真
		//从iattrs->ia_iattr拷贝ia_uid、ia_gid、ia_atime、ia_mtime和ia_ctime等成员的值给i节点相应的成员
		set_inode_attr(inode, &iattrs->ia_iattr); 
		security_inode_notifysecctx(inode,
					    iattrs->ia_secdata,
					    iattrs->ia_secdata_len); //安全检测模块
	}
 
	if (sysfs_type(sd) == SYSFS_DIR)
		inode->i_nlink = sysfs_count_nlink(sd); //计算目录的硬链接数目（等于子目录数+2）
}
```

四种structsysfs_dirent类型对应三种文件类型，其中SYSFS_KOBJ_ATTR和SYSFS_KOBJ_BIN_ATTR都为普通文件。文件系统针对不同的文件类型，i节点操作函数（struct inode_operations）和文件内容操作函数（struct file_operations）都会有不同的实现，并且其中的函数也是根据文件类型来决定是否实现（大部分成员为空指针）。

3.1、目录的i节点操作函数和文件内容操作函数分别为sysfs_dir_inode_operations和sysfs_dir_operations，其中i节点操作函数的create、mkdir和rmdir等成员都为空指针，表示sysfs文件系统的目录或文件无法从用户空间通过系统调用来创建和删除。对于sysfs文件系统中的目录来说，i节点操作函数最重要的是查找函数sysfs_lookup，文件内容操作函数最重要的是遍历目录函数sysfs_readdir。源代码如下所示：

```text
/* fs/sysfs/dir.c */
const struct inode_operations sysfs_dir_inode_operations = {
	.lookup		= sysfs_lookup,
	.permission	= sysfs_permission,
	.setattr	= sysfs_setattr,
	.getattr	= sysfs_getattr,
	.setxattr	= sysfs_setxattr,
};
 
const struct file_operations sysfs_dir_operations = {
	.read		= generic_read_dir,
	.readdir	= sysfs_readdir,
	.release	= sysfs_dir_release,
	.llseek		= generic_file_llseek,
};
 
static struct dentry * sysfs_lookup(struct inode *dir, struct dentry *dentry,
				struct nameidata *nd) //dir为父目录的i节点，dentry为被查找对象的目录项(这时为d_alloc函数操作之后的状态)
{
	struct dentry *ret = NULL;
	struct dentry *parent = dentry->d_parent; //父目录项
	struct sysfs_dirent *parent_sd = parent->d_fsdata;
	struct sysfs_dirent *sd;
	struct inode *inode;
	enum kobj_ns_type type;
	const void *ns;
 
	mutex_lock(&sysfs_mutex);

	//获取命名空间
	type = sysfs_ns_type(parent_sd);
	ns = sysfs_info(dir->i_sb)->ns[type];

	//从其parent_sd->s_dir.children链表中查找相同命名空间并且名字相同的子项
	sd = sysfs_find_dirent(parent_sd, ns, dentry->d_name.name);
	if (!sd) { //该子项不存在
		ret = ERR_PTR(-ENOENT);
		goto out_unlock;
	}
 
	//从inode_hashtable哈希表中查找该子项对应的i节点，若不存在则构建一个新的i节点实例
	inode = sysfs_get_inode(dir->i_sb, sd);
	if (!inode) { //构建失败
		ret = ERR_PTR(-ENOMEM);
		goto out_unlock;
	}

	//对于目录来说，只能有一个目录项，并且只有在其为空目录或者为当前文件系统的根目录时才有可能被设置DCACHE_UNHASHED状态
	//当i节点为IS_ROOT和DCACHE_DISCONNECTED时，d_find_alias函数返回真
	ret = d_find_alias(inode);
	if (!ret) {
		d_set_d_op(dentry, &sysfs_dentry_ops); //sysfs_dentry_ops为sysfs文件系统目录项的操作函数
		dentry->d_fsdata = sysfs_get(sd);
		d_add(dentry, inode);	//调用d_instantiate函数将目录项dentry插入到inode->i_dentry链表并且dentry->d_inode指向i节点inode
								//调用d_rehash函数将目录项dentry插入到dentry_hashtable哈希表中
	} else {
		d_move(ret, dentry); //重新使用所返回的目录项ret并与目录项dentry交换d_name等数据
		iput(inode); //销毁i节点
	}
 
 out_unlock:
	mutex_unlock(&sysfs_mutex);
	return ret;
}
```

对于目录来说，只能有一个目录项别名。

```text
/* fs/sysfs/dir.c */
static int sysfs_readdir(struct file * filp, void * dirent, filldir_t filldir)
{
	struct dentry *dentry = filp->f_path.dentry; //当前目录所对应的目录项
	struct sysfs_dirent * parent_sd = dentry->d_fsdata;
	struct sysfs_dirent *pos = filp->private_data;
	enum kobj_ns_type type;
	const void *ns;
	ino_t ino;

	//获取命名空间
	type = sysfs_ns_type(parent_sd);
	ns = sysfs_info(dentry->d_sb)->ns[type];

	//当前目录
	if (filp->f_pos == 0) {
		ino = parent_sd->s_ino;
		if (filldir(dirent, ".", 1, filp->f_pos, ino, DT_DIR) == 0)
			filp->f_pos++;
	}
	//父目录
	if (filp->f_pos == 1) {
		if (parent_sd->s_parent)
			ino = parent_sd->s_parent->s_ino;
		else
			ino = parent_sd->s_ino; //文件系统根目录将自身当作父目录
		if (filldir(dirent, "..", 2, filp->f_pos, ino, DT_DIR) == 0)
			filp->f_pos++;
	}
	mutex_lock(&sysfs_mutex);
	for (pos = sysfs_dir_pos(ns, parent_sd, filp->f_pos, pos); //这时filp->f_pos等于2，pos为NULL
	     pos;
	     pos = sysfs_dir_next_pos(ns, parent_sd, filp->f_pos, pos)) { //遍历当前目录
		const char * name;
		unsigned int type;
		int len, ret;
 
		name = pos->s_name;
		len = strlen(name);
		ino = pos->s_ino;
		type = dt_type(pos); //文件类型
		filp->f_pos = ino; //将i节点号作为上一个struct linux_dirent实例中d_off成员的值
		filp->private_data = sysfs_get(pos);
 
		mutex_unlock(&sysfs_mutex);
		ret = filldir(dirent, name, len, filp->f_pos, ino, type);
		mutex_lock(&sysfs_mutex);
		if (ret < 0)
			break;
	}
	mutex_unlock(&sysfs_mutex);
	if ((filp->f_pos > 1) && !pos) { //遍历完全
		filp->f_pos = INT_MAX;
		filp->private_data = NULL;
	}
	return 0;
}
 
static struct sysfs_dirent *sysfs_dir_pos(const void *ns,
	struct sysfs_dirent *parent_sd,	ino_t ino, struct sysfs_dirent *pos)
{
	if (pos) {
		int valid = !(pos->s_flags & SYSFS_FLAG_REMOVED) && //非SYSFS_FLAG_REMOVED项
			pos->s_parent == parent_sd &&  //属于目录parent_sd
			ino == pos->s_ino;
		sysfs_put(pos);
		if (!valid)
			pos = NULL; //重新遍历该目录
	}
	if (!pos && (ino > 1) && (ino < INT_MAX)) { //这时pos为空指针，且i节点号的大小在有效范围内
		pos = parent_sd->s_dir.children;
		while (pos && (ino > pos->s_ino)) //过滤掉i节点号比它小的
			pos = pos->s_sibling;
	}
	while (pos && pos->s_ns && pos->s_ns != ns) //过滤掉不是相同命名空间的
		pos = pos->s_sibling;
	return pos;
}
 
static struct sysfs_dirent *sysfs_dir_next_pos(const void *ns,
	struct sysfs_dirent *parent_sd,	ino_t ino, struct sysfs_dirent *pos)
{
	pos = sysfs_dir_pos(ns, parent_sd, ino, pos);
	if (pos)
		pos = pos->s_sibling; //获取下一个项
	while (pos && pos->s_ns && pos->s_ns != ns)
		pos = pos->s_sibling;
	return pos;
}
```

当在用户空间通过ls等命令查看sysfs文件系统的目录时，会通过系统调用getdents调用vfs_readdir，然后再调用sysfs_readdir函数。其中参数filp表示该目录打开之后的文件指针，dirent实际为struct getdents_callback类型的数据，filldir为回调函数指针，指向同名的filldir函数。源代码如下所示：

```text
/* include/linux/kernel.h */
#define __ALIGN_KERNEL(x, a)		__ALIGN_KERNEL_MASK(x, (typeof(x))(a) - 1)
#define __ALIGN_KERNEL_MASK(x, mask)	(((x) + (mask)) & ~(mask))
#define ALIGN(x, a)		__ALIGN_KERNEL((x), (a))
 
/* include/linux/compiler-gcc4.h */
#define __compiler_offsetof(a,b) __builtin_offsetof(a,b)  //GCC编译器内置函数，计算成员偏移量
 
/* include/linux/stddef.h */
#undef offsetof
#ifdef __compiler_offsetof
#define offsetof(TYPE,MEMBER) __compiler_offsetof(TYPE,MEMBER)
#else
#define offsetof(TYPE, MEMBER) ((size_t) &((TYPE *)0)->MEMBER)
#endif
 
/* fs/readdir.c */
struct linux_dirent {
	unsigned long	d_ino; //i节点号
	unsigned long	d_off; //偏移量，无实际意义，在sysfs文件系统的实现中，它的值为上一个所遍历的文件或目录的i节点号（最后一个为INT_MAX）
	unsigned short	d_reclen; //整个struct linux_dirent实例的长度（经过对齐修正）
	char		d_name[1]; //文件名，大小由实际文件名的长度决定（空字符结尾）
};
 
struct getdents_callback {
	struct linux_dirent __user * current_dir;	//初始化为系统调用传入的内存地址
	struct linux_dirent __user * previous;		//指向上一个struct linux_dirent实例
	int count;	//初始化为系统调用传入的内存的总大小（字节数）
	int error;	//保存错误码
};
 
static int filldir(void * __buf, const char * name, int namlen, loff_t offset,
		   u64 ino, unsigned int d_type)
{
	struct linux_dirent __user * dirent;
	struct getdents_callback * buf = (struct getdents_callback *) __buf;
	unsigned long d_ino;
	//功能等效于(len / sizeof(long) + (len % sizeof(long) ? 1 : 0)) * sizeof(long)，其中len为ALIGN的第一个参数的值
	int reclen = ALIGN(offsetof(struct linux_dirent, d_name) + namlen + 2,	//其中加2个字节的意义是：一个字节用来存储字符串的结束符，
		sizeof(long)); 														//另一个字节用来存储文件类型。
 
	buf->error = -EINVAL;
	if (reclen > buf->count) //剩余的内存不够
		return -EINVAL;
	d_ino = ino;
	if (sizeof(d_ino) < sizeof(ino) && d_ino != ino) { //d_ino的数据类型较小且导致数据溢出
		buf->error = -EOVERFLOW;
		return -EOVERFLOW;
	}
	dirent = buf->previous;
	if (dirent) {
		if (__put_user(offset, &dirent->d_off)) //以当前的offset值填充上一个struct linux_dirent实例的d_off成员
			goto efault;
	}
	dirent = buf->current_dir;
	if (__put_user(d_ino, &dirent->d_ino)) //i节点号
		goto efault;
	if (__put_user(reclen, &dirent->d_reclen)) //当前struct linux_dirent实例的总大小（字节数）
		goto efault;
	if (copy_to_user(dirent->d_name, name, namlen)) //拷贝文件名
		goto efault;
	if (__put_user(0, dirent->d_name + namlen)) //在文件名后加字符串结束符'\0'
		goto efault;
	if (__put_user(d_type, (char __user *) dirent + reclen - 1)) //最后一个字节用来保存文件类型信息
		goto efault;
	buf->previous = dirent;
	dirent = (void __user *)dirent + reclen;
	buf->current_dir = dirent; //指向下一个尚未使用的struct linux_dirent实例
	buf->count -= reclen; //计算剩余内存数量
	return 0;
efault:
	buf->error = -EFAULT;
	return -EFAULT;
}
```

3.2、sysfs文件系统针对文本文件和二进制文件实现了不同的文件内容操作函数，而i节点操作函数则相同。源代码如下所示：

```text
/* fs/sysfs/inode.c */
static const struct inode_operations sysfs_inode_operations ={
	.permission	= sysfs_permission,
	.setattr	= sysfs_setattr,
	.getattr	= sysfs_getattr,
	.setxattr	= sysfs_setxattr,
};
 
/* fs/sysfs/file.c */
const struct file_operations sysfs_file_operations = { //文本文件
	.read		= sysfs_read_file,
	.write		= sysfs_write_file,
	.llseek		= generic_file_llseek,
	.open		= sysfs_open_file,
	.release	= sysfs_release,
	.poll		= sysfs_poll,
};
 
/* fs/sysfs/bin.c */
const struct file_operations bin_fops = { //二进制文件
	.read		= read,
	.write		= write,
	.mmap		= mmap,
	.llseek		= generic_file_llseek,
	.open		= open,
	.release	= release,
};
```

针对普通文件，最重要的是观察它的文件内容操作函数的实现，如open、read和write等函数。下面则以文本文件为例，看看sysfs文件系统是如何读写文件的。源代码如下所示：

```text
/* include/linux/limits.h */
#define PATH_MAX        4096
 
/* fs/sysfs/file.c */
static char last_sysfs_file[PATH_MAX];
 
static int sysfs_open_file(struct inode *inode, struct file *file)
{
	struct sysfs_dirent *attr_sd = file->f_path.dentry->d_fsdata; //sysfs数据项，表示当前被打开的文件
	struct kobject *kobj = attr_sd->s_parent->s_dir.kobj; //所属目录所对应的内核对象
	struct sysfs_buffer *buffer;
	const struct sysfs_ops *ops;
	int error = -EACCES;
	char *p;

	//获得该文件的全路径并依次保存在last_sysfs_file数组的尾部
	p = d_path(&file->f_path, last_sysfs_file, sizeof(last_sysfs_file));
	if (!IS_ERR(p))
		memmove(last_sysfs_file, p, strlen(p) + 1); //将路径移动到数组的开头
	
	//获取活动引用计数
	if (!sysfs_get_active(attr_sd))
		return -ENODEV;
	
	if (kobj->ktype && kobj->ktype->sysfs_ops) //内核对象针对所属属性的读写函数必须存在
		ops = kobj->ktype->sysfs_ops;
	else {
		WARN(1, KERN_ERR "missing sysfs attribute operations for "
		       "kobject: %s\n", kobject_name(kobj));
		goto err_out;
	}

	if (file->f_mode & FMODE_WRITE) { //写文件
		if (!(inode->i_mode & S_IWUGO) || !ops->store) //需S_IWUGO访问权限且store函数必须定义
			goto err_out;
	}

	if (file->f_mode & FMODE_READ) { //读文件
		if (!(inode->i_mode & S_IRUGO) || !ops->show) //需S_IRUGO访问权限且show函数必须定义
			goto err_out;
	}

	//每次打开的文本文件都对应一个struct sysfs_buffer实例
	error = -ENOMEM;
	buffer = kzalloc(sizeof(struct sysfs_buffer), GFP_KERNEL);
	if (!buffer)
		goto err_out;
 
	mutex_init(&buffer->mutex);
	buffer->needs_read_fill = 1;
	buffer->ops = ops;
	file->private_data = buffer; //保存在文件指针的私有数据中
 
	//分配struct sysfs_open_dirent结构体实例
	error = sysfs_get_open_dirent(attr_sd, buffer);
	if (error)
		goto err_free;
	
	//打开成功，释放活动引用计数
	sysfs_put_active(attr_sd);
	return 0;
 
 err_free:
	kfree(buffer);
 err_out:
	sysfs_put_active(attr_sd);
	return error;
}
 
static DEFINE_SPINLOCK(sysfs_open_dirent_lock);
 
static int sysfs_get_open_dirent(struct sysfs_dirent *sd,
				 struct sysfs_buffer *buffer)
{
	struct sysfs_open_dirent *od, *new_od = NULL;
 
 retry:
	spin_lock_irq(&sysfs_open_dirent_lock);
 
	if (!sd->s_attr.open && new_od) { //每个文本文件的struct sysfs_dirent都有一个open成员
		sd->s_attr.open = new_od; //指向新分配并初始化的struct sysfs_open_dirent实例
		new_od = NULL;
	}
 
	od = sd->s_attr.open;
	if (od) {
		atomic_inc(&od->refcnt); //递增引用计数
		//将buffer作为链表元素插入到od->buffers链表（该链表中元素的数量就是该文本文件正被打开的次数）
		list_add_tail(&buffer->list, &od->buffers); 
	}
 
	spin_unlock_irq(&sysfs_open_dirent_lock);
 
	if (od) {
		kfree(new_od);
		return 0;
	}
 
	//为struct sysfs_open_dirent结构体实例分配内存
	new_od = kmalloc(sizeof(*new_od), GFP_KERNEL);
	if (!new_od)
		return -ENOMEM;
	
	//初始化成员
	atomic_set(&new_od->refcnt, 0);
	atomic_set(&new_od->event, 1);
	init_waitqueue_head(&new_od->poll);
	INIT_LIST_HEAD(&new_od->buffers);
	goto retry;
}
```

在sysfs文件系统中，文本文件使用struct sysfs_elem_attr来表示，而该结构体有一个struct sysfs_open_dirent类型的open成员，主要用于管理每次打开并读/写该文件时所使用的内存（struct sysfs_buffer）。源代码如下所示：

```text
/* fs/sysfs/file.c */
struct sysfs_open_dirent {
	atomic_t		refcnt; //打开次数
	atomic_t		event;
	wait_queue_head_t	poll; //等待队列
	struct list_head	buffers; //sysfs_buffer.list的链表
};
 
struct sysfs_buffer {
	size_t			count;	//数据大小（字节数）
	loff_t			pos;	//偏移量
	char			* page;	//指向一页内存，用于存储数据
	const struct sysfs_ops	* ops;	//操作函数
	struct mutex		mutex;		//互斥锁
	int			needs_read_fill;	//是否已填充数据
	int			event;
	struct list_head	list;	//插入所属的struct sysfs_open_dirent的buffers链表
};
```

sysfs_read_file为sysfs文件系统文本文件的读函数，源代码如下所示：

```text
/* fs/sysfs/file.c */
static ssize_t
sysfs_read_file(struct file *file, char __user *buf, size_t count, loff_t *ppos)
{
	struct sysfs_buffer * buffer = file->private_data;  //调用sysfs_open_file时所生成的
	ssize_t retval = 0;
 
	mutex_lock(&buffer->mutex);
	if (buffer->needs_read_fill || *ppos == 0) { //尚未获取数据或者文件偏移量为零
		retval = fill_read_buffer(file->f_path.dentry,buffer); //一次读取
		if (retval) //获取数据失败（返回零表示成功）
			goto out;
	}
	pr_debug("%s: count = %zd, ppos = %lld, buf = %s\n",
		 __func__, count, *ppos, buffer->page);
		 
	//将count个字节的数据拷贝到用户空间内存buf，buffer->count表示可拷贝数据的最大字节数（可能须要多次拷贝才能读取完整个内存）
	retval = simple_read_from_buffer(buf, count, ppos, buffer->page,
					 buffer->count); 
out:
	mutex_unlock(&buffer->mutex);
	return retval;
}
 
static int fill_read_buffer(struct dentry * dentry, struct sysfs_buffer * buffer)
{
	struct sysfs_dirent *attr_sd = dentry->d_fsdata; //sysfs数据项
	struct kobject *kobj = attr_sd->s_parent->s_dir.kobj; //所属内核对象
	const struct sysfs_ops * ops = buffer->ops;
	int ret = 0;
	ssize_t count;
 
	if (!buffer->page) //分配存储数据的内存
		buffer->page = (char *) get_zeroed_page(GFP_KERNEL);
	if (!buffer->page) //内存分配失败
		return -ENOMEM;
	
	if (!sysfs_get_active(attr_sd)) //获取活动引用计数
		return -ENODEV;
 
	buffer->event = atomic_read(&attr_sd->s_attr.open->event);
	count = ops->show(kobj, attr_sd->s_attr.attr, buffer->page);  //从该内核对象相应的属性中获取数据并保存到刚才所分配的内存中
 
	sysfs_put_active(attr_sd); //释放活动引用计数

	if (count >= (ssize_t)PAGE_SIZE) { //至多能读取PAGE_SIZE - 1个字节
		print_symbol("fill_read_buffer: %s returned bad count\n",
			(unsigned long)ops->show);
		/* Try to struggle along */
		count = PAGE_SIZE - 1;
	}
	if (count >= 0) {
		buffer->needs_read_fill = 0; //表示填充过数据
		buffer->count = count;
	} else {
		ret = count; //获取失败
	}
	return ret;
}
 
/* fs/libfs.c */
ssize_t simple_read_from_buffer(void __user *to, size_t count, loff_t *ppos,
				const void *from, size_t available)
{
	loff_t pos = *ppos;
	size_t ret;
 
	if (pos < 0) //文件偏移量不能为负数
		return -EINVAL;
	if (pos >= available || !count) //已无数据可读或者是想读取零个字节
		return 0;
	if (count > available - pos)  //只剩available - pos个字节
		count = available - pos;
	ret = copy_to_user(to, from + pos, count); //拷贝数据
	if (ret == count) //返回值为没有拷贝成功的字节数
		return -EFAULT;
	count -= ret; //获得count个字节的数据
	*ppos = pos + count; //更新文件偏移量
	return count;
}
```

sysfs_write_file为sysfs文件系统的写函数，源代码如下所示：

```text
/* fs/sysfs/file.c */
static ssize_t
sysfs_write_file(struct file *file, const char __user *buf, size_t count, loff_t *ppos) //ppos为文件偏移量
{
	struct sysfs_buffer * buffer = file->private_data; //调用sysfs_open_file时所生成的
	ssize_t len;
 
	mutex_lock(&buffer->mutex);
	len = fill_write_buffer(buffer, buf, count); //一次写入，从用户空间内存buf中拷贝count个字节到内核空间
	if (len > 0) //成功拷贝len个字节
		len = flush_write_buffer(file->f_path.dentry, buffer, len); //根据获得的数据更新该内核对象相应的属性
	if (len > 0)
		*ppos += len; //更改文件偏移量，对buffer->page中的数据无意义，后面写入的数据会覆盖前面写入的数据
	mutex_unlock(&buffer->mutex);
	return len;
}
 
static int 
fill_write_buffer(struct sysfs_buffer * buffer, const char __user * buf, size_t count)
{
	int error;
 
	if (!buffer->page) //分配存储数据的内存
		buffer->page = (char *)get_zeroed_page(GFP_KERNEL);
	if (!buffer->page) //内存分配失败
		return -ENOMEM;
 
	if (count >= PAGE_SIZE) //写入的数据最多只能为PAGE_SIZE - 1个字节
		count = PAGE_SIZE - 1;
	error = copy_from_user(buffer->page,buf,count); //拷贝数据，成功函数返回零
	buffer->needs_read_fill = 1;
	buffer->page[count] = 0; //字符串结束符'\0'
	return error ? -EFAULT : count;
}
 
static int
flush_write_buffer(struct dentry * dentry, struct sysfs_buffer * buffer, size_t count)
{
	struct sysfs_dirent *attr_sd = dentry->d_fsdata; //sysfs数据项，这里表示一个文本文件
	struct kobject *kobj = attr_sd->s_parent->s_dir.kobj; //所属内核对象
	const struct sysfs_ops * ops = buffer->ops;
	int rc;
 
	if (!sysfs_get_active(attr_sd)) //获取活动引用计数
		return -ENODEV;
 
	rc = ops->store(kobj, attr_sd->s_attr.attr, buffer->page, count); //调用store函数
 
	sysfs_put_active(attr_sd); //释放活动引用计数
 
	return rc;
}
```

关闭文件时，打开、读/写文件时所分配的内存都会释放，并不会一直存在于内核中。

3.3、对于符号链接文件来说，它没有文件内容操作函数，只有i节点操作函数，其中最重要的函数为符号链接文件的解析函数sysfs_follow_link，源代码如下所示：

```text
/* fs/sysfs/symlink.c */
const struct inode_operations sysfs_symlink_inode_operations = {
	.setxattr	= sysfs_setxattr,
	.readlink	= generic_readlink,
	.follow_link	= sysfs_follow_link,
	.put_link	= sysfs_put_link,
	.setattr	= sysfs_setattr,
	.getattr	= sysfs_getattr,
	.permission	= sysfs_permission,
};
 
static void *sysfs_follow_link(struct dentry *dentry, struct nameidata *nd)
{
	int error = -ENOMEM;
	unsigned long page = get_zeroed_page(GFP_KERNEL); //分配内存
	if (page) {
		error = sysfs_getlink(dentry, (char *) page); 
		if (error < 0)  //成功时sysfs_getlink返回零
			free_page((unsigned long)page);
	}
	nd_set_link(nd, error ? ERR_PTR(error) : (char *)page); //保存获得的相对路径或错误码
	return NULL;
}
 
static int sysfs_getlink(struct dentry *dentry, char * path)
{
	struct sysfs_dirent *sd = dentry->d_fsdata; //sysfs数据项
	struct sysfs_dirent *parent_sd = sd->s_parent; //父sysfs数据项
	struct sysfs_dirent *target_sd = sd->s_symlink.target_sd; //所链接到的sysfs数据项
	int error;
 
	mutex_lock(&sysfs_mutex);
	error = sysfs_get_target_path(parent_sd, target_sd, path); //获取从链接文件到链接对象的相对路径
	mutex_unlock(&sysfs_mutex);
 
	return error;
}
 
//sysfs链接文件是两个内核对象之间的链接，也就是目录之间的链接
static int sysfs_get_target_path(struct sysfs_dirent *parent_sd,
				 struct sysfs_dirent *target_sd, char *path)
{
	struct sysfs_dirent *base, *sd;
	char *s = path;
	int len = 0;

	base = parent_sd;
	while (base->s_parent) { //sysfs的根数据项为sysfs_root，该数据项的s_parent成员为空指针
		sd = target_sd->s_parent;
		while (sd->s_parent && base != sd)  //如果base一直不等于sd，则循环直到根数据项才会停止
			sd = sd->s_parent;
 
		if (base == sd) //两者相等，这时链接文件和被链接对象的直接或间接的父目录相同
			break;
 
		strcpy(s, "../");  //拷贝字符串“../”，意味着两者不是同在这一目录下
		s += 3;
		base = base->s_parent; //接下来将比较链接文件的上一级目录的数据项
	}

	//这时，base已指向链接文件和被链接对象首个共有的直接或间接的父目录的数据项

	//计算整个路径的长度
	sd = target_sd;
	while (sd->s_parent && sd != base) {
		len += strlen(sd->s_name) + 1; //其中的加1表示目录项分隔符“/”所占的字节
		sd = sd->s_parent;
	}

	if (len < 2) //名称为空字符串
		return -EINVAL;
	len--; //减去一个多余的目录项分隔符所占的字节
	if ((s - path) + len > PATH_MAX) //总长度超过path数组的大小
		return -ENAMETOOLONG;
 
	//从被链接对象开始以倒序的方式拷贝目录项名称，直到base数据项（但不包括它的）
	sd = target_sd;
	while (sd->s_parent && sd != base) {
		int slen = strlen(sd->s_name);
 
		len -= slen;
		strncpy(s + len, sd->s_name, slen); //拷贝名称
		if (len) //等于0时表示最后一个名称的前面不需要再加分隔符
			s[--len] = '/'; //在该名称前加目录项分隔符“/”
 
		sd = sd->s_parent; //接着处理上一级目录
	}
 
	return 0;
}
```

读取符号链接文件内容的函数generic_readlink主要就是靠解析函数来实现的，源代码如下所示：

```text
/* fs/namei.c */
int generic_readlink(struct dentry *dentry, char __user *buffer, int buflen)
{
	struct nameidata nd;
	void *cookie;
	int res;
 
	nd.depth = 0;
	cookie = dentry->d_inode->i_op->follow_link(dentry, &nd);
	if (IS_ERR(cookie)) //返回错误码
		return PTR_ERR(cookie);
 
	res = vfs_readlink(dentry, buffer, buflen, nd_get_link(&nd)); //通过nd_get_link获取follow_link保存的路径或错误码
	if (dentry->d_inode->i_op->put_link)
		dentry->d_inode->i_op->put_link(dentry, &nd, cookie); //这里put_link指向sysfs_put_link函数
	return res;
}
 
int vfs_readlink(struct dentry *dentry, char __user *buffer, int buflen, const char *link)
{
	int len;
 
	len = PTR_ERR(link);
	if (IS_ERR(link)) //这里的link也有可能是错误码
		goto out;
 
	len = strlen(link);
	if (len > (unsigned) buflen) //路径长度比传入的内存大
		len = buflen;
	if (copy_to_user(buffer, link, len)) //拷贝到用户空间内存（但不包括字符串结束符）
		len = -EFAULT;
out:
	return len;
}
 
/* fs/sysfs/symlink.c */
static void sysfs_put_link(struct dentry *dentry, struct nameidata *nd, void *cookie)
{
	char *page = nd_get_link(nd);
	if (!IS_ERR(page)) //非错误码
		free_page((unsigned long)page); //释放内存
}
```

其中，sysfs_put_link函数执行与sysfs_follow_link函数相反的操作，这里只是释放由sysfs_follow_link函数分配的内存。

4、对于sysfs文件系统来说，在用户空间只能读写文件的内容，而无法创建或删除文件或目录，只能在内核中通过sysfs_create_dir、sysfs_create_file等等函数来实现。

4.1、内核对象（struct kobject）对应于sysfs文件系统中的目录，可使用sysfs_create_dir函数来创建，源代码如下所示：

```text
/* fs/sysfs/dir.c */
int sysfs_create_dir(struct kobject * kobj)
{
	enum kobj_ns_type type;
	struct sysfs_dirent *parent_sd, *sd;
	const void *ns = NULL;
	int error = 0;
 
	BUG_ON(!kobj);
 
	if (kobj->parent) //父内核对象为空时，则在sysfs文件系统的根目录下创建目录
		parent_sd = kobj->parent->sd;
	else
		parent_sd = &sysfs_root;
	
	//获取命名空间及其类型
	if (sysfs_ns_type(parent_sd))
		ns = kobj->ktype->namespace(kobj);
	type = sysfs_read_ns_type(kobj);
 
	error = create_dir(kobj, parent_sd, type, ns, kobject_name(kobj), &sd);
	if (!error) //成功则返回零
		kobj->sd = sd;  //保存相应的sysfs数据项
	return error;
}
 
/* include/linux/kobject.h */
static inline const char *kobject_name(const struct kobject *kobj)
{
	return kobj->name;
}
 
/* fs/sysfs/dir.c */
static int create_dir(struct kobject *kobj, struct sysfs_dirent *parent_sd,
	enum kobj_ns_type type, const void *ns, const char *name,
	struct sysfs_dirent **p_sd)
{
	umode_t mode = S_IFDIR| S_IRWXU | S_IRUGO | S_IXUGO; //文件类型为目录，访问权限为0755
	struct sysfs_addrm_cxt acxt;
	struct sysfs_dirent *sd;
	int rc;
 
	//分配sysfs数据项并初始化
	sd = sysfs_new_dirent(name, mode, SYSFS_DIR); //数据项类型为目录
	if (!sd)
		return -ENOMEM;
 
	sd->s_flags |= (type << SYSFS_NS_TYPE_SHIFT); //命名空间类型，占用s_flags倒数第二个8位
	sd->s_ns = ns; //命名空间
	sd->s_dir.kobj = kobj; //关联内核对象

	sysfs_addrm_start(&acxt, parent_sd); //加锁并携带父数据项
	rc = sysfs_add_one(&acxt, sd); //关联父数据项
	sysfs_addrm_finish(&acxt); //解锁
 
	if (rc == 0)  //成功返回
		*p_sd = sd;
	else
		sysfs_put(sd); //释放数据项
 
	return rc;
}
 
struct sysfs_dirent *sysfs_new_dirent(const char *name, umode_t mode, int type)
{
	char *dup_name = NULL;
	struct sysfs_dirent *sd;
 
	if (type & SYSFS_COPY_NAME) { //目录或符号链接文件需要拷贝文件名，它们对应的都是内核对象
		name = dup_name = kstrdup(name, GFP_KERNEL); //分配内存并拷贝文件名
		if (!name) //当不为空指针时则表示name指向新分配的内存
			return NULL;
	}
 
	sd = kmem_cache_zalloc(sysfs_dir_cachep, GFP_KERNEL); //从sysfs_dir_cachep缓存中分配sysfs数据项
	if (!sd)
		goto err_out1;
 
	if (sysfs_alloc_ino(&sd->s_ino)) //分配i节点号
		goto err_out2;
	
	//引用计数
	atomic_set(&sd->s_count, 1);
	atomic_set(&sd->s_active, 0);
 
	sd->s_name = name; //目录项名称
	sd->s_mode = mode; //文件类型及访问权限
	sd->s_flags = type; //sysfs数据项类型，占用s_flags低8位
 
	return sd;
 
 err_out2:
	kmem_cache_free(sysfs_dir_cachep, sd);
 err_out1:
	kfree(dup_name);
	return NULL;
}
 
int sysfs_add_one(struct sysfs_addrm_cxt *acxt, struct sysfs_dirent *sd)
{
	int ret;
 
	ret = __sysfs_add_one(acxt, sd);
	if (ret == -EEXIST) { //同名数据项已经存在则输出告警信息
		char *path = kzalloc(PATH_MAX, GFP_KERNEL);
		WARN(1, KERN_WARNING
		     "sysfs: cannot create duplicate filename '%s'\n",
		     (path == NULL) ? sd->s_name :
		     strcat(strcat(sysfs_pathname(acxt->parent_sd, path), "/"),
		            sd->s_name)); //数据项的全路径
		kfree(path);
	}
 
	return ret;
}
 
int __sysfs_add_one(struct sysfs_addrm_cxt *acxt, struct sysfs_dirent *sd)
{
	struct sysfs_inode_attrs *ps_iattr;
 
	if (sysfs_find_dirent(acxt->parent_sd, sd->s_ns, sd->s_name)) //查找该父目录下是否存在同名的数据项
		return -EEXIST; //已存在则返回错误码
 
	sd->s_parent = sysfs_get(acxt->parent_sd); //递增父数据项的引用计数，然后指向该父数据项
 
	sysfs_link_sibling(sd); //加入到父数据项的子数据项链表
 
	//更新父数据项的时间戳
	ps_iattr = acxt->parent_sd->s_iattr;
	if (ps_iattr) {
		struct iattr *ps_iattrs = &ps_iattr->ia_iattr;
		ps_iattrs->ia_ctime = ps_iattrs->ia_mtime = CURRENT_TIME;
	}
 
	return 0;
}
 
struct sysfs_dirent *sysfs_find_dirent(struct sysfs_dirent *parent_sd,
				       const void *ns,
				       const unsigned char *name)
{
	struct sysfs_dirent *sd;
 
	for (sd = parent_sd->s_dir.children; sd; sd = sd->s_sibling) { //遍历父目录
		if (ns && sd->s_ns && (sd->s_ns != ns)) //查找同一命名空间
			continue;
		if (!strcmp(sd->s_name, name)) //同名sysfs数据项
			return sd;
	}
	return NULL;
}
 
static void sysfs_link_sibling(struct sysfs_dirent *sd)
{
	struct sysfs_dirent *parent_sd = sd->s_parent;
	struct sysfs_dirent **pos;
 
	BUG_ON(sd->s_sibling);
 
	for (pos = &parent_sd->s_dir.children; *pos; pos = &(*pos)->s_sibling) {
		if (sd->s_ino < (*pos)->s_ino) //升序排列
			break;
	}

	sd->s_sibling = *pos;
	*pos = sd;
}
```

4.2、内核对象的属性（struct attribute）对应于sysfs文件系统中的文本文件，可使用sysfs_create_file函数来创建，源代码如下所示：

```text
/* fs/sysfs/file.c */
int sysfs_create_file(struct kobject * kobj, const struct attribute * attr)
{
	BUG_ON(!kobj || !kobj->sd || !attr); //三者必须为真
 
	return sysfs_add_file(kobj->sd, attr, SYSFS_KOBJ_ATTR); //数据项类型为SYSFS_KOBJ_ATTR，对应sysfs文件系统中的文本文件
}
 
int sysfs_add_file(struct sysfs_dirent *dir_sd, const struct attribute *attr,
		   int type)
{
	return sysfs_add_file_mode(dir_sd, attr, type, attr->mode); //访问权限由内核对象的属性自身配置
}
 
int sysfs_add_file_mode(struct sysfs_dirent *dir_sd,
			const struct attribute *attr, int type, mode_t amode)
{
	umode_t mode = (amode & S_IALLUGO) | S_IFREG; //文件类型为普通文件
	struct sysfs_addrm_cxt acxt;
	struct sysfs_dirent *sd;
	int rc;

	//分配sysfs数据项并初始化
	sd = sysfs_new_dirent(attr->name, mode, type);
	if (!sd)
		return -ENOMEM;
	sd->s_attr.attr = (void *)attr; //保存内核对象属性
	sysfs_dirent_init_lockdep(sd); //初始化死锁检测模块
 
	sysfs_addrm_start(&acxt, dir_sd); //dir_sd对应于属性文件所在的目录
	rc = sysfs_add_one(&acxt, sd);
	sysfs_addrm_finish(&acxt);
 
	if (rc) //失败则销毁sysfs数据项
		sysfs_put(sd);
 
	return rc;
}
```

sysfs文件系统中的二进制文件通过sysfs_create_bin_file函数来创建，符号链接文件通过sysfs_create_link函数来创建。

------

版权声明：本文为知乎博主「Linux内核库」的原创文章，遵循CC 4.0 BY-SA版权协议，转载请附上原文 出处链接及本声明。

原文链接：https://zhuanlan.zhihu.com/p/558497992