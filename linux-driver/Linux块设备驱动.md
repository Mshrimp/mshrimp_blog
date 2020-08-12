---
title: Linux块设备驱动
date: 2020-04-19 10:40:13
tags: Driver
---




## Linux块设备驱动



Linux块设备驱动



未完成

<!--more-->




### 目录


[TOC]


### 0. 简介




字符设备


字符设备是一种顺序的数据流设备，对字符设备的读写是以字节为单位进行的，这些字符连续地形成一个数据流，字符设备没有缓存区，对于字符设备的读写是实时的；




块设备


块设备是一种具有一定结构的随机存取设备，对块设备的读写是以块为单位进行的，块设备使用缓存区来存放数据，待条件满足后，将数据从缓存区一次性写入到设备，或者从设备一次性读取到缓存区；
















为了创建一个块设备驱动程序，实现一个基于内存的块设备驱动程序；




#### 1. 块设备结构


**段（Segments）**：由若干个块组成；是Linux内存管理机制中一个内存页或内存页的一部分；


**块（Blocks）**：由Linux制定对内核或文件系统等数据处理的基本单位；通常通常为4096个字节，由1个或多个扇区组成；


**扇区（Sectors）**：块设备的基本单位，是一个固定的硬件单位，制定了设备最少能够传输的数据量；通常在512字节到32768字节之间，默认：512字节；


块是连续扇区的序列，块长度总是扇区长度的整数倍；块的最大长度，受特定体系结构的内存页长度限制；




块设备使用请求队列，缓存并重排读写数据块的请求，用高效的方式读取数据；块设备的每个设备都关联了请求队列；对块设备的读写请求不会立即执行，这些请求会汇总起来，经过协同之后传输到设备；

![Linux的块IO](Linux块设备驱动/Linux的块IO.png)



#### 2. 块设备驱动框架



![块设备驱动架构图](Linux块设备驱动/块设备驱动架构图.jpg)








### 1. 重要结构及操作




#### 1. 注册块设备驱动程序


向内核注册块设备驱动程序


```c
#include <linux/fs.h>
int register_blkdev(unsigned int major, const char *name);
```



> name：设备名字，是在/proc/devices中显示的名字
>
> major：设备的主设备号，如果major=0，则分配一个主设备号

该函数的调用是可选的，完成的工作：

> 1. 如果需要的话分配一个动态的主设备号
> 2. 在/proc/devices中创建一个入口项



注销块设备驱动程序


```c
#include <linux/fs.h>
void unregister_blkdev(unsigned int major, const char *name);
```




如下用例：


```c
#include <linux/fs.h>
int sbull_major = 0;
sbull_major = register_blkdev(sbull_major, "sbull");
if (sbull_major < 0) {
	printk(KERN_WARNNING "sbull: unable to get major number\n");
	return -EBUSY;
}
```




```c
// include/linux/fs.h
struct block_device {
    dev_t           bd_dev;  /* not a kdev_t - it's a search key */
    int         bd_openers;
    struct inode *      bd_inode;   /* will die */
    struct super_block *    bd_super;
    struct mutex        bd_mutex;   /* open/close mutex */
    void *          bd_claiming;
    void *          bd_holder;
    int         bd_holders;
    bool            bd_write_holder;
#ifdef CONFIG_SYSFS
    struct list_head    bd_holder_disks;
#endif
    struct block_device *   bd_contains;
    unsigned        bd_block_size;
    struct hd_struct *  bd_part;
    /* number of times partitions within this device have been opened. */
    unsigned        bd_part_count;
    int         bd_invalidated;
    struct gendisk *    bd_disk;
    struct request_queue *  bd_queue;
    struct list_head    bd_list;
    /*
     * Private data.  You must have bd_claim'ed the block_device
     * to use this.  NOTE:  bd_claim allows an owner to claim
     * the same device multiple times, the owner must take special
     * care to not mess up bd_private for that case.
     */
    unsigned long       bd_private;

    /* The counter of freeze processes */
    int         bd_fsfreeze_count;
    /* Mutex for freeze */
    struct mutex        bd_fsfreeze_mutex;
};
```








#### 2. 块设备操作




字符设备使用file_operations结构，来告诉系统字符设备驱动的操作接口；


块设备使用block_device_operations结构，来告诉系统块设备驱动的操作接口；


```c
// include/linux/blkdev.h
struct block_device_operations {
    int (*open) (struct block_device *, fmode_t);
    void (*release) (struct gendisk *, fmode_t);
    int (*rw_page)(struct block_device *, sector_t, struct page *, bool);
    int (*ioctl) (struct block_device *, fmode_t, unsigned, unsigned long);
    int (*compat_ioctl) (struct block_device *, fmode_t, unsigned, unsigned long);
	......
    /* ->media_changed() is DEPRECATED, use ->check_events() instead */
    int (*media_changed) (struct gendisk *);
    int (*getgeo)(struct block_device *, struct hd_geometry *);
	......
};
```


和字符设备驱动不同，块设备驱动的block_device_operations操作集中没有负责读和写数据的函数；在块设备驱动中，这些操作是由request函数处理的；




#### 3. 注册磁盘


为了管理独立的磁盘，需要使用struct gendisk结构体，内核使用gendisk结构表示一个独立的磁盘设备，还可以表示分区；


```c
// include/linux/genhd.h
struct gendisk {
    int major;          /* major number of driver */
    int first_minor;
    int minors;                     /* maximum number of minors, =1 for
                                         * disks that can't be partitioned. */
    char disk_name[DISK_NAME_LEN];  /* name of major driver */
    char *(*devnode)(struct gendisk *gd, umode_t *mode);

    struct hd_struct part0;

    const struct block_device_operations *fops;
    struct request_queue *queue;
	......
};
```


major：指定驱动程序的主设备号


first_minor和minors：从设备号的可能范围


disk_name：磁盘名称，在/proc/partitions 和 sysfs 中表示该磁盘




```c
char disk_name[DISK_NAME_LEN];
```


显示在/proc/partitions 和 sysfs 中




对于每一个分区来说，都有一个hd_struct结构体，用于描述该分区


```c
// include/linux/genhd.h
struct hd_struct {
    sector_t start_sect;
    sector_t nr_sects;
    seqcount_t nr_sects_seq;
	......
    struct partition_meta_info *info;
	......
};
```


start_sect和nr_sects：定义了该分区在块设备上的起始扇区和长度，唯一地描述了该分区










拥有了设备内存和请求队列，就可以分配、初始化及安装gendisk结构；在struct gendisk是动态分配的结构，需要内核进行初始化，驱动必须通过alloc_disk分配：


```c
# include <linux/genhd.h>
struct gendisk *alloc_disk(int minors);
```

> minors：是该磁盘使用的从设备号的数目；



卸载磁盘


```c
void del_gendisk(struct gendisk *disk)
```


gendisk是一个引用计数结构，get_disk和put_disk函数负责处理引用计数；调用del_gendisk后，该结构可能继续存在；




为了使gendisk结构的磁盘设备生效，需要初始化结构，并将磁盘或分区信息添加到内核链表；


```c
void add_disk(struct gendisk *gd);
```


调用add_disk后，磁盘设备将被激活，并随时会调用它提供的操作方法，因此在驱动程序完全被初始化并且能够响应对磁盘的请求前，不要调用add_disk；
















#### 4. 请求队列




块设备的读写请求放置在请求队列中，在struct gendisk中，通过struct request_queue *queue指针指向请求队列；请求队列用数据结构struct request_queue表示；


```c
// include/linux/blkdev.h
struct request_queue {
    struct list_head    queue_head;
    struct request      *last_merge;
    struct elevator_queue   *elevator;
	......
    request_fn_proc     *request_fn;
    make_request_fn     *make_request_fn;
	......
    void            *queuedata;

    spinlock_t      __queue_lock;
    spinlock_t      *queue_lock;

    struct list_head    icq_list;

    struct queue_limits limits;

    struct blk_flush_queue  *fq;

    struct list_head    requeue_list;
    spinlock_t      requeue_lock;
    struct delayed_work requeue_work;
	......
};
```


queue_head：表头，用于构建一个IO请求的双链表；链表每个元素代表向块设备读取数据的一个请求；内核会重排该链表，以得到更好的IO性能；






与每个块设备驱动程序相关的I/O请求队列用request_queue结构体描述，而每个request_queue队列中的请求用request结构体描述；


```c
// include/linux/blkdev.h
struct request {
    struct list_head queuelist;

    struct request_queue *q;
    struct blk_mq_ctx *mq_ctx;

    /* the following two fields are internal, NEVER access directly */
    unsigned int __data_len;    /* total data len */
    sector_t __sector;      /* sector cursor */

    struct bio *bio;
    struct bio *biotail;

	struct request *next_rq;
    ......
}
```




request结构体关联了struct bio，struct bio结构体是块I/O操作在页级粒度的底层描述；


```c
// include/linux/blk_types.h
struct bio {
    struct bio      *bi_next;   /* request queue link */
    struct block_device *bi_bdev;
    int         bi_error;

    unsigned short      bi_flags;   /* status, command, etc */
    unsigned short      bi_ioprio;

    struct bvec_iter    bi_iter;

    atomic_t        __bi_remaining;
    bio_end_io_t        *bi_end_io;
    
    unsigned short      bi_vcnt;    /* how many bio_vec's */
    unsigned short      bi_max_vecs;    /* max bvl_vecs we can hold */
    struct bio_vec      *bi_io_vec; /* the actual vec list */
	......
}
```


块数据通过bio_vec结构体数组在内部被表示成I/O向量；每个bio_vec数组元素由三元组组成（即，页、页偏移、长度），表示该块I/O的一个段；

```c
// include/linux/bvec.h
struct bio_vec {
    struct page *bv_page;	// 页指针
    unsigned int    bv_len;	// 传输的字节数
    unsigned int    bv_offset;	// 偏移位置
};
```








块设备驱动程序的核心是请求函数，包含请求处理过程；

```c
// include/linux/blkdev.h
struct request_queue *blk_init_queue(request_fn_proc *rfn, spinlock_t *lock)
```

或

```c
// include/linux/blkdev.h
void blk_queue_make_request(struct request_queue *q, make_request_fn *mfn)
```




### 2. 块设备驱动的初始化



#### 2.1 块设备的注册过程

注册一个块设备驱动，需要以下步骤：



> 创建一个块设备 
>
> 分配一个申请队列
>
> 分配一个gendisk结构体
>
> 设置gendisk结构体成员
>
> 注册gendisk结构体






```mermaid
graph TB
	A("注册设备(register_blkdev)(可选)")-->B("分配磁盘(alloc_disk)")
	B-->C("不使用请求队列(blk_init_queue)")-->E
	B-->D("使用请求队列(blk_alloc_queue)")-->E
	E("设置磁盘属性(gendisk)")-->F("激活磁盘(add_disk)")
```


1. 通过register_blkdev()函数注册设备，是个可选操作；

2. 使用alloc_disk()函数分配通用磁盘gendisk结构体；

3. 根据是否需要I/O调度，分两种情况，一种是使用请求队列进行数据传输，一种是不使用请求队列进行数据传输；

4. 初始化gendisk结构体的数据成员，包括：major、fops、queue等；

5. 使用add_disk()函数激活磁盘设备，调用该函数之前要做好所有的准备工作；






分配一个gendisk结构体


设置一个队列，将访问请求放到队列里


设置gendisk结构体的属性，如：名称、容量、操作集等


添加gendisk结构体


另外分配一块内存空间，当做块设备，在request函数中使用memcpy访问，模仿块设备读写




```c
module_init(sbull_blkdev_init);
module_exit(sbull_blkdev_exit);
```




块设备驱动程序的初始化方法在sbull_blkdev_init()函数中；




##### 1）注册块设备


```c
static int major = 0;
major = register_blkdev(major, "sbull_blkdev");
```


为块设备驱动分配一个未使用的主设备号，并在/proc/devies中添加一个入口；




##### 2）注册请求队列


注册请求队列的操作，将一个请求的操作方法与该设备相关联，通过blk_init_queue()函数实现；


```c
static struct request_queue *sbull_blkdev_request = NULL;
static DEFINE_SPINLOCK(sbull_blkdev_lock);
sbull_blkdev_queue = blk_init_queue(sbull_blkdev_request, &sbull_blkdev_lock);
```


blk_init_queue()函数返回请求队列request_queue；将blkdev_request()函数指针方式关联到设备；而第二个参数是自旋锁，用来保护request_queue队列不被同时访问；




##### 3）设置读写块大小


硬件执行磁盘是以扇区为单位的，而文件系统是以块为单位处理数据；通常，扇区大小为512字节，块大小为4096字节；需要将硬件支持的扇区大小和驱动程序在一次请求中能接收的最大扇区数通知块层；


```c
// include/linux/blkdev.h
int sbull_blkdev_sect_size = 512;
int sbull_blkdev_size = 16 * 1024 * 1024;
// blk_queue_hardsect_size(blkdev_queue, my_blkdev_sect_size);
blk_queue_logical_block_size(sbull_blkdev_queue, sbull_blkdev_sect_size);
```




##### 4）创建磁盘


使用alloc_disk()函数分配一个与设备对应的磁盘gendisk结构体，并初始化其成员；需要初始化的成员有：block_device_operations、存储容量（单位是扇区）、请求队列、主设备号、磁盘名称等；设置存储容量通过set_capacity()函数来完成；


调用add_disk()函数将磁盘添加到块I/O层；


```c
sbull_blkdev_disk = alloc_disk(1);

sprintf(sbull_blkdev_disk->disk_name, "sbull_blkdev_disk");
sbull_blkdev_disk->fops = &sbull_blkdev_fops;
sbull_blkdev_disk->queue = sbull_blkdev_queue;
sbull_blkdev_disk->major = major;
sbull_blkdev_disk->first_minor = 0;
set_capacity(sbull_blkdev_disk, sbull_blkdev_size);

add_disk(sbull_blkdev_disk);
```




到这里，设备/dev/sbull_blkdev_disk就可以使用了，如果设备支持多个磁盘分区，会显示为/dev/sbull_blkdev_diskX，X是分区号；


```c
# ls /dev/sbull_blkdev_disk -l
brw-rw----    1 root     root      253,   0 Oct 21 08:18 /dev/sbull_blkdev_disk
```



##### 5）块设备初始化实例




```c
struct block_device_operations sbull_blkdev_fops = {
    .owner = THIS_MODULE,
    .open = sbull_blkdev_open,
    .release = sbull_blkdev_release,
    .ioctl = sbull_blkdev_ioctl,
};
```



请求处理函数，具体实现内容，后边补充；

```c
void sbull_blkdev_request(struct request_queue *q)
{
    printk("%s: %d\n", __func__, __LINE__);

    return;
}
```




```c
static int major = 0;
static struct request_queue *sbull_blkdev_queue = NULL;
static struct gendisk *sbull_blkdev_disk = NULL;
static DEFINE_SPINLOCK(sbull_blkdev_lock);

int sbull_blkdev_size = 256 * 1024;
int sbull_blkdev_sect_size = 512;

static int sbull_blkdev_init(void)
{
    major = register_blkdev(major, "mcy_blk");
    if (major < 0) {
        printk("%s, register_blkdev failed, major: %d\n", __func__, major);
        goto register_blkdev_err;
    }
    printk("%s, major: %d\n", __func__, major);

    sbull_blkdev_queue = blk_init_queue(sbull_blkdev_request, &sbull_blkdev_lock);
    if (!sbull_blkdev_queue) {
        printk("%s, blk_init_queue failed!\n", __func__);
        goto init_queue_err;
    }

    sbull_blkdev_disk = alloc_disk(1);
    if (!sbull_blkdev_disk) {
        printk("%s, alloc_disk failed!\n", __func__);
        goto alloc_disk_err;
    }

    sprintf(sbull_blkdev_disk->disk_name, "sbull_blkdev_disk");
    sbull_blkdev_disk->fops = &sbull_blkdev_fops;
    sbull_blkdev_disk->queue = sbull_blkdev_queue;
    sbull_blkdev_disk->major = major;
    sbull_blkdev_disk->first_minor = 0;
    set_capacity(sbull_blkdev_disk, sbull_blkdev_size * 2);

    add_disk(sbull_blkdev_disk);
    printk("%s, sbull_blkdev_disk add success!\n", __func__);

    return 0;
alloc_disk_err:
    blk_cleanup_queue(sbull_blkdev_queue);
init_queue_err:
    unregister_blkdev(major, "sbull_blkdev");
register_blkdev_err:
    return -EBUSY;
}
```



#### 2.2 块设备的卸载过程






```mermaid
graph TB
	A("删除gendisk(del_gendisk)")-->
	B("删除gendisk的引用(put_disk)")-->
	C("清除请求队列(blk_cleanup_queue)")-->
	D("注销块设备(unregister_blkdev)")
```


1. 使用del_gendisk()函数删除gendisk设备（磁盘）；

2. 使用put_disk()函数删除gendisk设备的引用；

3. 使用blk_cleanup_queue()函数清除请求队列，释放请求队列占用的资源；

4. 使用unregister_blkdev()函数注销设备，并释放对设备的引用，可选操作，与register_blkdev()函数配合使用；




```c
static void sbull_blkdev_exit(void)
{
    del_gendisk(sbull_blkdev_disk);
    put_disk(sbull_blkdev_disk);
    blk_cleanup_queue(sbull_blkdev_queue);
    unregister_blkdev(major, "sbull_blkdev");
}
```



```c
module_init(sbull_blkdev_init);
module_exit(sbull_blkdev_exit);
MODULE_LICENSE("GPL");
```





#### 2.3 封装块设备信息

为了在一些函数中能够访问块设备驱动的一些信息（比如：请求队列，硬盘分区，硬盘尺寸等）参数，可以定义一个块设备驱动设备结构体，给每一个块设备定义一个信息描述，如下：

```c
typedef struct sbull_dev {
    int major;
    unsigned long size;
    void *data;
    struct request_queue *queue;
    struct gendisk *disk;
    spinlock_t lock;
    struct timer_list timer;
} blkdev_t;

blkdev_t *blkdev = NULL;
```

同时，定义了一些必要的宏定义：

```c
#define SBULL_BLKDEV_SIZE   (16 * 1024 * 1024)	// 指定块设备硬盘的大小为16M
#define SBULL_BLKDEV_SECTOR_SIZE    (512)	// 设置块设备的sector为512字节
#define SBULL_BLKDEV_MAX_PARTITIONS (16)	// 设置块设备最大的分区数
```

这样在块设备中只需要通过blkdev指针就可以获取到设备的所有需要的参数信息；而相应的初始化操作也需要相应地修改：

```c
blkdev = kmalloc(sizeof(blkdev_t), GFP_KERNEL);
if (!blkdev) {
    printk("%s, blkdev kmalloc failed\n", __func__);
    goto blkdev_kmalloc_err;
}
memset(blkdev, 0, sizeof(blkdev_t));

spin_lock_init(&blkdev->lock);
blkdev->queue = blk_alloc_queue(GFP_KERNEL);
blkdev->size = SBULL_BLKDEV_SIZE;
blkdev->data = sbull_blkdev_data;

blkdev->disk = alloc_disk(SBULL_BLKDEV_MAX_PARTITIONS);
sprintf(blkdev->disk->disk_name, SBULL_BLKDEV_NAME);
blkdev->disk->fops = &sbull_blkdev_fops;
blkdev->disk->queue = blkdev->queue;
blkdev->disk->major = major;
blkdev->disk->first_minor = 0;
set_capacity(blkdev->disk, SBULL_BLKDEV_SIZE / SBULL_BLKDEV_SECTOR_SIZE);
add_disk(blkdev->disk);
```

这个块设备的结构体信息，可以通过gendisk结构体中的private_data指针保存，也可以通过request_queue结构体中的queuedata指针保存；

```c
blkdev->queue->queuedata = blkdev;
blkdev->disk->private_data = blkdev;
```

在需要使用这些信息时可以通过queue->queuedata或disk->private_data指针快速获取到块设备信息；



### 3. 队列请求



这主要是由于该版本适用于2.6.29内核，从2.6.31内核开始，一些API发生变化（见linux/include/blkdev.h）；用到的几个API修改如下：

| 老版本内核接口              | 新版本内核接口                      | 意义               |
| --------------------------- | ----------------------------------- | ------------------ |
| request -> sectors          | blk_rq_pos(request)                 | 获取请求的开始扇区 |
| request -> nr_sectors       | blk_rq_nr_sectors(request)          | 获取请求的扇区数   |
| elev_next_request(request)  | blk_fetch_request(request)          |                    |
| end_request(request, error) | blk_end_request_all(request, error) |                    |





每个块设备，都有一个请求队列，当请求队列生成时，请求函数request()就与该队列绑定，这个操作有两种方法实现；

#### 3.1 blk_init_queue

第一种是通过blk_init_queue()函数完成；


```c
// block/blk-core.c
struct request_queue *blk_init_queue(request_fn_proc *rfn, spinlock_t *lock)
{
    return blk_init_queue_node(rfn, lock, NUMA_NO_NODE);
}
EXPORT_SYMBOL(blk_init_queue);
```



blk_init_queue()函数的调用关系：


```c
blk_init_queue
	blk_init_queue_node
		blk_alloc_queue_node
			kmem_cache_alloc_node
			ida_simple_get
			bioset_create
			bdi_init
			percpu_ref_init
			blkcg_init_queue
		blk_init_allocated_queue
			blk_alloc_flush_queue
			blk_init_rl
			blk_queue_make_request
			elevator_init
```



实现方法

```c
blkdev->queue = blk_init_queue(sbull_blkdev_request, &blkdev->lock);
```





```c
void sbull_blkdev_request(struct request_queue *q)
{
    struct request *req = NULL;
    blkdev_t *dev = q->queuedata;
    unsigned long sect_pos = 0;
    unsigned long sect_count = 0;
    unsigned long start = 0;
    unsigned long len = 0;
    void *buffer = NULL;
    int err = 0;

    req = blk_fetch_request(q);
    while (req != NULL) {
        sect_pos = blk_rq_pos(req);
        sect_count = blk_rq_cur_sectors(req);
        start = sect_pos << 9;
        len = blk_rq_cur_bytes(req);

        if (start + len > dev->size) {
            printk("%s, bad access, block: 0x%llx, count: 0x%lx\n", __func__, (unsigned long long)sect_pos, sect_count);
            err = -EIO;
            goto done;
        }

        buffer = bio_data(req->bio);
        if (!buffer) {
            printk("%s, bio_data buffer is null\n", __func__);
            goto done;
        }
        sbull_transfer(dev, sect_pos, sect_count, buffer, rq_data_dir(req) == WRITE);
done:
        if (!__blk_end_request_cur(req, err)) {
            req = blk_fetch_request(q);
        }
    }

    return;
}
```



```c
void sbull_transfer(blkdev_t *dev, unsigned long sector, unsigned long nsect, char *buffer, int write)
{
    unsigned long offset = sector * SBULL_BLKDEV_SECTOR_SIZE;
    unsigned long nbytes = nsect * SBULL_BLKDEV_SECTOR_SIZE;

    if (offset + nbytes > dev->size) {
        printk("%s, over range, size: 0x%lx, offset: 0x%lx, nbytes: 0x%lx", __func__, dev->size, offset, nbytes);
        return;
    }

    if (write) {
        memcpy(dev->data + offset, buffer, nbytes);
    } else {
        memcpy(buffer, dev->data + offset, nbytes);
    }

    return;
}
```









#### 3.2 blk_alloc_queue





实现方法：

```c
blkdev->queue = blk_alloc_queue(GFP_KERNEL);
blk_queue_make_request(blkdev->queue, sbull_blkdev_make_request);
```





```c
unsigned int sbull_blkdev_make_request(struct request_queue *q, struct bio *bio)
{
    blkdev_t *dev = q->queuedata;
    int status = 0;

    status = sbull_xfer_bio(dev, bio);
    bio_endio(bio);

    return 0;
}

int sbull_xfer_bio(blkdev_t *dev, struct bio *bio)
{
    struct bvec_iter iter;
    struct bio_vec bvec;
    sector_t sector = bio->bi_iter.bi_sector;
    char *buffer = NULL;

    bio_for_each_segment(bvec, bio, iter) {
        buffer = __bio_kmap_atomic(bio, iter);
        if (!buffer) {
            printk("%s, buffer is null\n", __func__);
            return -1;
        }
        sbull_transfer(dev, sector, bio_cur_bytes(bio)>>9, buffer, bio_data_dir(bio) == WRITE);
        sector += bio_cur_bytes(bio)>>9;
        __bio_kunmap_atomic(bio);
    }

    return 0;
}
```










```c
// include/linux/blkdev.h
struct request {

    

}
```
















```c
blk_queue_max_hw_sectors(blkdev->queue, 255);
```


blk_queue_max_hw_sectors()函数，用来通知通用块层和I/O调度器，该请求队列的每个请求中能够包含的最大扇区数；


```c
blk_queue_logical_block_size(blkdev->queue, sbull_sect_size);
```


blk_queue_logical_block_size()函数，用于告知该请求队列的逻辑块大小；








bio的一些接口：


```c
// include/linux/fs.h
static inline int bio_data_dir(struct bio *bio)
{
    return op_is_write(bio_op(bio)) ? WRITE : READ;
}

#ifdef CONFIG_BLOCK
static inline bool op_is_write(unsigned int op)
{
    return op == REQ_OP_READ ? false : true;
}
```


return：READ/WRITE




```c
// include/linux/bio.h
#define BIO_MAX_PAGES       256

#define bio_prio(bio)           (bio)->bi_ioprio
#define bio_set_prio(bio, prio)     ((bio)->bi_ioprio = prio)

#define bio_iter_iovec(bio, iter)               \
    bvec_iter_bvec((bio)->bi_io_vec, (iter))

#define bio_iter_page(bio, iter)                \
    bvec_iter_page((bio)->bi_io_vec, (iter))
#define bio_iter_len(bio, iter)                 \
    bvec_iter_len((bio)->bi_io_vec, (iter))
#define bio_iter_offset(bio, iter)              \
    bvec_iter_offset((bio)->bi_io_vec, (iter))

#define bio_page(bio)       bio_iter_page((bio), (bio)->bi_iter)
#define bio_offset(bio)     bio_iter_offset((bio), (bio)->bi_iter)
#define bio_iovec(bio)      bio_iter_iovec((bio), (bio)->bi_iter)

#define bio_multiple_segments(bio)              \
    ((bio)->bi_iter.bi_size != bio_iovec(bio).bv_len)
#define bio_sectors(bio)    ((bio)->bi_iter.bi_size >> 9)
#define bio_end_sector(bio) ((bio)->bi_iter.bi_sector + bio_sectors((bio)))
```




```c
// include/linux/bio.h
static inline unsigned int bio_cur_bytes(struct bio *bio)
{
    if (bio_has_data(bio))
        return bio_iovec(bio).bv_len;
    else /* dataless requests such as discard */
        return bio->bi_iter.bi_size;
}

// 数据缓冲区的内核虚拟地址
static inline void *bio_data(struct bio *bio)
{
    if (bio_has_data(bio))
        return page_address(bio_page(bio)) + bio_offset(bio);

    return NULL;
}
```






```c
// include/linux/bio.h
// 获取给定bio的第i个缓冲区的虚拟地址
#define __bio_kmap_atomic(bio, iter)                \
    (kmap_atomic(bio_iter_iovec((bio), (iter)).bv_page) +   \
        bio_iter_iovec((bio), (iter)).bv_offset)

// 释放缓冲区的虚拟地址
#define __bio_kunmap_atomic(addr)   kunmap_atomic(addr)
```












```c
// include/linux/blkdev.h
#define rq_data_dir(rq)     (op_is_write(req_op(rq)) ? WRITE : READ)

#define REQ_OP_SHIFT (8 * sizeof(u64) - REQ_OP_BITS)
#define req_op(req)  ((req)->cmd_flags >> REQ_OP_SHIFT)

// include/linux/blk_types.h
#define REQ_OP_BITS 3
```






```c
// include/linux/blkdev.h
/*
 * blk_rq_pos()         : the current sector
 * blk_rq_bytes()       : bytes left in the entire request
 * blk_rq_cur_bytes()       : bytes left in the current segment
 * blk_rq_err_bytes()       : bytes left till the next error boundary
 * blk_rq_sectors()     : sectors left in the entire request
 * blk_rq_cur_sectors()     : sectors left in the current segment
 */
static inline sector_t blk_rq_pos(const struct request *rq)
{
    return rq->__sector;
}

static inline unsigned int blk_rq_bytes(const struct request *rq)
{
    return rq->__data_len;
}

static inline int blk_rq_cur_bytes(const struct request *rq)
{
    return rq->bio ? bio_cur_bytes(rq->bio) : 0;
}

static inline unsigned int blk_rq_sectors(const struct request *rq)
{
    return blk_rq_bytes(rq) >> 9;
}

static inline unsigned int blk_rq_cur_sectors(const struct request *rq)
{
    return blk_rq_cur_bytes(rq) >> 9;
}

static inline unsigned int blk_rq_count_bios(struct request *rq)
{
    unsigned int nr_bios = 0;
    struct bio *bio;

    __rq_for_each_bio(bio, rq)
        nr_bios++;

    return nr_bios;
}
```





#### 3.3 两种队列请求的区别




#### 3.4 I/O调度器


I/O调度器可以通过合并请求、重排块设备操作顺序等方式提高块设备访问的顺序；


I/O调度器四种






```c
# cat /sys/block/sbull_disk/queue/scheduler
noop [cfq]
```




对于大多数基于物理磁盘的块设备驱动，使用适合的I/O调度器能提高性能；




无I/O调度器


```c
# cat /sys/block/sbull_disk/queue/scheduler
none
```













































### 8. 测试验证






测试步骤：

> 加载驱动：insmod ramblock.ko
>
> 格式化：mkdosfs /dev/ramblock
>
> 挂载：mount /dev/ramblock /mnt
>
> 读写文件：cd /mnt，创建文件
>
> 卸载：umount /mnt
>
> cat /dev/ramblock > /mnt/ramblock.bin
>
> 在PC上查看/mnt/ramblock.bin，sudo mount -o loop ramblock.bin /mnt





### 9. 总结






### 参考资料


https://www.cnblogs.com/big-devil/p/8590007.html









[回到目录](#目录)

