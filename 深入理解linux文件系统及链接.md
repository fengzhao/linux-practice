## 深入理解linux文件系统及链接



### 1.文件存储

理解文件系统，要先理解文件在计算机中的存储。

众所周知，计算机在处理数据时，文件是以二进制存储在硬盘中，硬盘的最小存储单位叫“扇区”（sectort），每个扇区存储512字节（），

对于硬盘，存储一份文件时，其实需要保存两类信息，一类是文件本身的数据，例如文本文件中的字符内容，还有一类是文件的元数据，比如文件的拥有者和属组，创建时间，修改时间，文件大小等信息。但是元数据不包括文件名称，因为在linux内核中的是通过inode号来标识文件的，下面会讲到文件的访问过程。存储文件元数据的区域叫做inode，中文译名“索引节点”。



#### 1.1 文件访问

操作系统访问文件的流程，大致可以简单分为如下三步：

１


#### 1.1 indoe的内容

在Unix/Linux上，每个文件都有一个inode节点来保存其元数据，inode在系统管理员看来是每一个文件的唯一标识，在系统里面，inode是一个结构，存储了关于这个文件的大部分信息。 但是不包括文件名。

inode包含文件的元信息，具体来说有以下内容：

- 文件的字节数
- 文件拥有者的UserID和文件属组的GroupID
- 文件的读、写、执行权限
- 文件的时间戳，共有三个：ctime指inode上一次变动的时间，mtime指文件内容上一次变动的时间，atime指文件上一次打开的时间。
- 链接数，即有多少文件名指向这个inode文件数据block的位置可以用stat命令，查看某个文件的inode信息：stat example.txt

inode就是一个文件的一部分描述，不是全部，在内核中，inode对应了这样一个实际存在的。inode中存储了一个文件的以下信息:

```c
struct inode {
        struct hlist_node       i_hash;              /* 哈希表 */
        struct list_head        i_list;              /* 索引节点链表 */
        struct list_head        i_dentry;            /* 目录项链表 */
        unsigned long           i_ino;               /* 节点号 */
        atomic_t                i_count;             /* 引用记数 */
        umode_t                 i_mode;              /* 访问权限控制 */
        unsigned int            i_nlink;             /* 硬链接数 */
        uid_t                   i_uid;               /* 使用者id */
        gid_t                   i_gid;               /* 使用者id组 */
        kdev_t                  i_rdev;          	 /* 实设备标识符 */
        loff_t                  i_size;              /* 以字节为单位的文件大小 */
        struct timespec         i_atime;             /* 最后访问时间 */
        struct timespec         i_mtime;             /* 最后修改(modify)时间 */
        struct timespec         i_ctime;             /* 最后改变(change)时间 */
        unsigned int            i_blkbits;           /* 以位为单位的块大小 */
        unsigned long           i_blksize;           /* 以字节为单位的块大小 */
        unsigned long           i_version;           /* 版本号 */
        unsigned long           i_blocks;            /* 文件的块数 */
        unsigned short          i_bytes;             /* 使用的字节数 */
        spinlock_t              i_lock;              /* 自旋锁 */
        struct rw_semaphore     i_alloc_sem;         /* 索引节点信号量 */
        struct inode_operations *i_op;               /* 索引节点操作表 */
        struct file_operations  *i_fop;              /* 默认的索引节点操作 */
        struct super_block      *i_sb;               /* 相关的超级块 */
        struct file_lock        *i_flock;            /* 文件锁链表 */
        struct address_space    *i_mapping;          /* 相关的地址映射 */
        struct address_space    i_data;              /* 设备地址映射 */
        struct dquot            *i_dquot[MAXQUOTAS]; /* 节点的磁盘限额 */
        struct list_head        i_devices;           /* 块设备链表 */
        struct pipe_inode_info  *i_pipe;             /* 管道信息 */
        struct block_device     *i_bdev;             /* 块设备驱动 */
        unsigned long           i_dnotify_mask;      /* 目录通知掩码 */
        struct dnotify_struct   *i_dnotify;          /* 目录通知 */
        unsigned long           i_state;             /* 状态标志 */
        unsigned long           dirtied_when;        /* 首次修改时间 */
        unsigned int            i_flags;             /* 文件系统标志 */
        unsigned char           i_sock;              /* 可能是个套接字吧 */
        atomic_t                i_writecount;        /* 写者记数 */
        void                    *i_security;         /* 安全模块 */
        __u32                   i_generation;        /* 索引节点版本号 */
        union {
                void            *generic_ip;         /* 文件特殊信息 */
        } u;
};
```



纵观整个inode的C语言描述，没有发现关于文件名的东西，也就是说文件名不由inode保存，实际上系统是不关心文件名的，对于系统中任何的操作，大部分情况下你都是通过文件名来做的，但系统最终都要通过找到文件对应的inode来操作文件，由inode结构中 * i_op指向的接口来操作。

文件系统如何存取文件的：
1. 根据文件名，通过Directory里的对应关系，找到文件对应的Inodenumber
2. 再根据Inodenumber读取到文件的Inodetable
3. 再根据Inodetable中的Pointer读取到相应的Blocks

#### 1.2 Inode的大小

inode也会消耗硬盘空间，所以硬盘格式化的时候，操作系统自动将硬盘分成两个区域。一个是数据区，存放文件数据；另一个是inode区（inode table），存放inode所包含的信息。

每个inode节点的大小，一般是128字节或256字节。inode节点的总数，在格式化时就给定，一般是每1KB或每2KB就设置一个inode。假定在一块1GB的硬盘中，每个inode节点的大小为128字节，每1KB就设置一个inode，那么inode table的大小就会达到128MB，占整块硬盘的12.8%。

查看每个硬盘分区的inode总数和已经使用的数量，可以使用df命令:

```shell
[root@li994-163 ~]# df -i
Filesystem      Inodes  IUsed  IFree IUse% Mounted on
/dev/root      1264000 295407 968593   24% /
devtmpfs        124707   1380 123327    2% /dev
tmpfs           125413      1 125412    1% /dev/shm
tmpfs           125413   1280 124133    2% /run
tmpfs           125413     17 125396    1% /sys/fs/cgroup
[root@li994-163 ~]# 
```

查看磁盘的inode详细情况，可以用如下命令：

```shell
[root@li994-163 ~]# dumpe2fs /dev/sda  | more
dumpe2fs 1.42.9 (28-Dec-2013)
Filesystem volume name:   <none>
Last mounted on:          /
Filesystem UUID:          2665379f-bd8a-4760-b1ff-e4dfb3d4d8d2
Filesystem magic number:  0xEF53
Filesystem revision #:    1 (dynamic)
Filesystem features:      has_journal ext_attr resize_inode dir_index filetype needs_recovery extent flex_bg sparse_super large_file huge_file uninit_bg dir_nlink extra_isize
Filesystem flags:         signed_directory_hash 
Default mount options:    user_xattr acl
Filesystem state:         clean
Errors behavior:          Continue
Filesystem OS type:       Linux
Inode count:              1264000
Block count:              5177344
Reserved block count:     258867
Free blocks:              3510627
Free inodes:              1100962
First block:              0
Block size:               4096
Fragment size:            4096
Reserved GDT blocks:      77
Blocks per group:         32768
Fragments per group:      32768
Inodes per group:         8000
Inode blocks per group:   500
Flex block group size:    16
Filesystem created:       Mon Sep 18 17:40:01 2017
Last mount time:          Thu Jun 21 06:16:14 2018
Last write time:          Thu Jun 21 06:16:14 2018
Mount count:              7
Maximum mount count:      -1
Last checked:             Fri Apr 13 14:49:19 2018
Check interval:           0 (<none>)
Lifetime writes:          33 GB
Reserved blocks uid:      0 (user root)
Reserved blocks gid:      0 (group root)
First inode:              11
Inode size:	          256
Required extra isize:     28
Desired extra isize:      28
Journal inode:            8
First orphan inode:       30782
Default directory hash:   half_md4
Directory Hash Seed:      107763c5-7cca-4da9-b405-2ecc4ed346bf


```

 