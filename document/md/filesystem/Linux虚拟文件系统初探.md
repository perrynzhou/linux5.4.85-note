## Linux虚拟文件系统初探

| 作者                 | 时间       | QQ技术交流群 |
| -------------------- | ---------- | ------------ |
| perrynzhou@gmail.com | 2020/12/01 | 672152841    |

#### 什么是VFS?

- Linux内核使用工厂的设计模式抽象出实际文件系统统一接口，这个就是虚拟文件系统(VFS)，根据应用程序调用虚拟文件系统接口，根据不同的文件系统类型(xfs/zfs/ext4)来调用实际文件系统的接口
- VFS自身仅仅存在于内存，VFS自身定义了几个重要的数据结构:inode、dentry、super_block，通过这几个重要的结构将真是硬盘的文件系统抽象到内存，通过dentry、inode这几个对象来进行文件的读写操作。

#### 什么是super block(超级块)？

- 超级块代表了整个文件系统，对应文件系统自身的控制块结构。超级块保存文件系统设定文件块大小、超级块操作函数，同时文件系统中所有的inode都链接到超级块的表头。
- 超级块的内容需要读取具体文件系统在磁盘上的超级块获得，因此超级块是具体文件系统超级块的内存抽象，所以如果磁盘上的超级块坏了，文件系统就坏了。
```
// linux 5.4.85/include/linux/fs.h 取出super_block核心字段
struct super_block {
    // 文件系统块大小
	unsigned long		s_blocksize;
	//文件系统支持的最大文件
    loff_t			s_maxbytes;	/* Max file size */
	// 文件系统的类型
	struct file_system_type	*s_type;
	// 超级块的操作函数
	const struct super_operations	*s_op;
	// 文件系统根目录的指针
	struct dentry		*s_root;
	// 文件系统所有的inode
	struct list_head	s_inodes;	/* all inodes */
	// 对应的块设备指针
	struct block_device	*s_bdev;
}
```

- 每个文件系统都有一个超级块结构，每个超级块都要链接到一个超级块链表。文件系统内的每个文件打开时候都需要在内存分配一个inode结构，这些inode结构都要链接到超级块

#### 什么是dentry(目录项)？

- 在文件系统中，文件和目录一般按照树状结构保存，比如找一个位于/a/b/c/1.txt文件，文件系统会从a开始一层一层的查找直到找到c目录下的1.txt文件。文件系统中的dentry就是反应这里的树状关系
- 在linux中每个文件都有一个dentry，这个dentry链接到上层目录的dentry.根目录有一个dentry结构，根目录中的文件和目录的dentry都链接到根目录的dentry.
- linux内核中为了加快dentry查找，使用hash表来缓存dentry(dentry cache)。对于一个文件查找一般先查找dentry cache中进行

```
// linux 5.4.85/include/linux/dcache.h 取出dentry核心字段
struct dentry {
	// d_hash是链接到dentry cache的hash链表
	struct hlist_bl_node d_hash;	
	// 上层目录的dentry
	struct dentry *d_parent;	
	// 目录名称
	struct qstr d_name;
	// dentry对应的indoe，用dentry和inode来描述文件或者目录
	struct inode *d_inode;		

	// dentry的操作函数
	const struct dentry_operations *d_op;
	// 超级块的dentry树
	struct super_block *d_sb;	
	// 文件系统私有数据
	void *d_fsdata;		
	// d_child是dentry自身的链表头，需要链接到父dentry中的d_subdirs中
	struct list_head d_child;
    // 所有的子目录或者子文件都要链接到这个d_subdirs上
	struct list_head d_subdirs;	
} __randomize_layout;

```

- 每个文件的dentry链接到父目录的dentry,形成了文件系统的结构树，所有的dentry存储在dentry_hashtable这个哈希数组中。如果某个文件已经被打开过，内存中就应该有该文件的dentry的结构，并且这个dentry被链接到dentry_hashtable的数组中，后续在访问该文件时候，直接从hashtable中查找，避免再次读磁盘，这个dentry_hashtable也是linux 的dentry cache.

#### 什么是inode?

- inode表示linux中的文件，inode保存了文件大小、文件创建、修改、访问时间、文件对应磁盘的位置等信息，同时inode也保存了文件的读写函数、读写缓存等信息。使用linux 文件链接可以导致一个真实文件可以包括多个dentry，而inode只有一个。

```
// linux 5.4.85/include/linux/fs.h 取出inode核心字段
struct inode {
	// 文件的权限信息
	umode_t			i_mode;
	// 操作文件的uid
	kuid_t			i_uid;
	// 操作文件的gid
	kgid_t			i_gid;
   // inode标志，可以是S_SYNC,S_NOATIME,S_DIRSYNC等
   unsigned int		i_flags;
  	//如果inode代表设备，i_rdev表示该设备的设备号
	dev_t			i_rdev;
	// 所属的超级块
	struct super_block	*i_sb;
	//文件的长度
	loff_t			i_size;
	// 文件的创建、修改、访问时间
	struct timespec64	i_atime;
	struct timespec64	i_mtime;
	struct timespec64	i_ctime;
	spinlock_t		i_lock;	/* i_blocks, i_bytes, maybe i_size */
	// 文件中位于最后一个块的字节数
	unsigned short          i_bytes;
	// 以bit为单位的块的大小
	u8			i_blkbits;
	// 文件使用块的数目
	blkcnt_t		i_blocks;
	// inode的编号
	unsigned long		i_ino;

#ifdef __NEED_I_SIZE_ORDERED
	seqcount_t		i_size_seqcount;
#endif

	/* Misc */
	unsigned long		i_state;
	struct rw_semaphore	i_rwsem;

	unsigned long		dirtied_when;	/* jiffies of first dirtying */
	unsigned long		dirtied_time_when;

	struct hlist_node	i_hash;
	struct list_head	i_io_list;	/* backing dev IO list */
#ifdef CONFIG_CGROUP_WRITEBACK
	struct bdi_writeback	*i_wb;		/* the associated cgroup wb */

	/* foreign inode detection, see wbc_detach_inode() */
	int			i_wb_frn_winner;
	u16			i_wb_frn_avg_time;
	u16			i_wb_frn_history;
#endif
	// 当前inod lru链表
	struct list_head	i_lru;		/* inode LRU list */
	// i_sb_list用于链接到超级块中的inode链表
	struct list_head	i_sb_list;
	// 同一个文件的多个dentry需要链接到i_dentry中
	union {
		struct hlist_head	i_dentry;
		struct rcu_head		i_rcu;
	};
	atomic64_t		i_version;
	atomic64_t		i_sequence; /* see futex */
	atomic_t		i_count;
	atomic_t		i_dio_count;
	//记录有多少个进程以可写的方式打开此文件
	atomic_t		i_writecount;
	//记录有多少个进程以可读的方式打开此文件

#if defined(CONFIG_IMA) || defined(CONFIG_FILE_LOCKING)
	atomic_t		i_readcount; /* struct files open RO */
#endif
	// i_fop是一个file_operations类型指针，提供文件的读写函数和异步IO函数
	union {
		const struct file_operations	*i_fop;	/* former ->i_op->default_file_ops */
		void (*free_inode)(struct inode *);
	};
	//指向各种不同设备的指针
	union {
		//如果文件是一个管道则使用i_pipe
		struct pipe_inode_info	*i_pipe;
		//如果文件是一个块设备则使用i_bdev
		struct block_device	*i_bdev;
		//如果文件是一个字符设备这使用i_cdev
		struct cdev		*i_cdev;
		char			*i_link;
		unsigned		i_dir_seq;
	};
	// 文件系统的私有数据
	void			*i_private; /* fs or device private pointer */
} __randomize_layout;
```

- linux 内核提供一个inode_hashtable，功能和dentry_hashtable类似。inode结构中的i_mode代表了该文件类型，一般有块设备、字符设备、目录、socket、FIFO。同时inode还有一个重要的作用就是缓存文件的数据内容，这个是通过i_mapping来实现。

#### 什么是文件？

- 文件对象是描述进程和文件交互的关系，磁盘上并不存在以为文件结构，当进程打开一个文件，内核就动态创建爱你一个文件对象。同一个文件，在不同的进程有不同的文件对象。
- 内核为每个打开的文件申请一个文件对象(fd)同时返回文件号。

