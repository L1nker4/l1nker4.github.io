# Linux文件管理模块分析


## 硬盘文件系统



### inode与块的存储



硬盘读写时以扇区为单位，文件系统中读写数据最小单位为块，一个块（簇）内部是相邻的几个扇区，在Linux中的ext文件系统，默认大小为4K。



文件的元数据存放在inode中，ext4中定义如下：

```c
struct ext4_inode {
	__le16	i_mode;		/* File mode */
	__le16	i_uid;		/* Low 16 bits of Owner Uid */
	__le32	i_size_lo;	/* Size in bytes */
	__le32	i_atime;	/* Access time */
	__le32	i_ctime;	/* Inode Change time */
	__le32	i_mtime;	/* Modification time */
	__le32	i_dtime;	/* Deletion Time */
	__le16	i_gid;		/* Low 16 bits of Group Id */
	__le16	i_links_count;	/* Links count */
	__le32	i_blocks_lo;	/* Blocks count */
	__le32	i_flags;	/* File flags */
......
	__le32	i_block[EXT4_N_BLOCKS];/* Pointers to blocks */
	__le32	i_generation;	/* File version (for NFS) */
	__le32	i_file_acl_lo;	/* File ACL */
	__le32	i_size_high;
......
};


#define	EXT4_NDIR_BLOCKS		12
#define	EXT4_IND_BLOCK			EXT4_NDIR_BLOCKS
#define	EXT4_DIND_BLOCK			(EXT4_IND_BLOCK &#43; 1)
#define	EXT4_TIND_BLOCK			(EXT4_DIND_BLOCK &#43; 1)
#define	EXT4_N_BLOCKS			(EXT4_TIND_BLOCK &#43; 1)
```

`i_block`中存放文件所在的磁盘块地址，可以通过多层寻址增加文件大小上限。

但是这样的结构存在很严重的性能问题：大文件需要多次进行磁盘IO才能找到相应的块，而磁盘IO的时间往往很慢。

ext4对其进行优化，引入了一个新的概念**Extents**，将多个连续的块存放在一个Extents中。

![image-20220723145824483](https://blog-1251613845.cos.ap-shanghai.myqcloud.com/image-20220723145824483.png)



















































---

> Author:   
> URL: http://localhost:1313/posts/linux/linux%E6%96%87%E4%BB%B6%E7%AE%A1%E7%90%86%E6%A8%A1%E5%9D%97%E5%88%86%E6%9E%90/  

