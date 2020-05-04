



## Linux字符设备驱动






字符设备驱动


设备对数据的处理是按照字节流的形式进行的；




### 1. 字符设备注册


#### 1.1 早期字符设备的注册




```c
#include <linux/fs.h>
static inline int register_chrdev(unsigned int major,					//主设备号
                                  const char *name,						//设备结点名称
                                  const struct file_operations *fops);	//设备操作集
```


函数返回值：注册的设备号






该函数是老版本的设备注册函数，可以实现静态注册和动态注册，通过major参数是否为0来区别，参数major为0时，是动态注册，即在设备注册时会自动分配主设备号。


注册方法：


```c
int major = 0;
static struct class *pclass;
major = register_chrdev(0, DRV_NAME, &fops);
pclass = class_create(THIS_MODULE, "test_dev");
device_create(pclass, NULL, MKDEV(major, 0), NULL, "test_dev");
```




#### 1.2 新版本字符设备的注册




```c
register_chrdev_region(dev_t first,unsigned int count,char *name);
alloc_chrdev_region(dev_t *dev, unsigned baseminor, unsigned count, const char *name);
```




```c
static struct cdev *pcdev;
static struct class *pclass;
dev_t devnum;

major = MAJOR(devnum);
pcdev = cdev_alloc();
cdev_init(pcdev, &fops);
cdev_add(pcdev, DEV_NAME, CNT);
pclass = class_create();
device_create();
```






##### 1）静态注册设备号


使用register_chrdev_region()函数静态注册设备号，支持注册


```c
// fs/char_dev.c
#include <linux/fs.h>
int register_chrdev_region(dev_t from, unsigned count, const char *name)
{
    struct char_device_struct *cd;
    dev_t to = from + count;
    dev_t n, next;

    for (n = from; n < to; n = next) {
        next = MKDEV(MAJOR(n)+1, 0);
        if (next > to)
            next = to;
        cd = __register_chrdev_region(MAJOR(n), MINOR(n),
                   next - n, name);
        if (IS_ERR(cd))
            goto fail;
    }
    return 0;
fail:
    to = n;
    for (n = from; n < to; n = next) {
        next = MKDEV(MAJOR(n)+1, 0);
        kfree(__unregister_chrdev_region(MAJOR(n), MINOR(n), next - n));
    }
    return PTR_ERR(cd);
}
```

> 参数说明：
>
> ​	from：
>
> ​	count：
>
> ​	name：
>
> 返回值
>
> ​	成功，返回0
>
> ​	失败，返回负值



##### 2）动态注册设备号


使用alloc_chrdev_region()函数注册




```c
// fs/char_dev.c
#include <linux/fs.h>
int alloc_chrdev_region(dev_t *dev, unsigned baseminor, unsigned count,
            const char *name)
{
    struct char_device_struct *cd;
    cd = __register_chrdev_region(0, baseminor, count, name);
    if (IS_ERR(cd))
        return PTR_ERR(cd);
    *dev = MKDEV(cd->major, cd->baseminor);
    return 0;
}
```






#### 1.3 最后使用版本




```c
#define XXX_DEV_NUM 0

struct xxx_dev {
  struct cdev cdev;
  int major;
  int minor;
  const char* name;
};

static struct xxx_dev g_dev;
static class *pclass;

static struct file_operations xxx_fops = {
    .owner      = THIS_MODULE,
	......
};

static int __init xxx_init(void)
{
	dev_t dev;
	int ret;

	memset(&g_dev, 0, sizeof(struct g_dev));
    g_dev.name = "xxx";

	if (XXX_DEV_NUM) {
		g_dev.major = XXX_DEV_NUM;
		g_dev.minor = 0;
		dev = MKDEV(g_dev.major, g_dev.minor);
		ret = register_chrdev_region(dev, 1, g_dev.name);
	} else {
		ret = alloc_chrdev_region(&dev, g_dev.minor, 1, g_dev.name);
		g_dev.major = MAJOR(dev);
		g_dev.minor = MINOR(dev);
	}
	if (ret < 0) {
		printk("register chrdev error!\n");
		goto REG_DEV_ERR;
	}

	cdev_init(&g_dev.cdev, &xxx_fops);
	g_dev.cdev.owner = THIS_MODULE;
	ret = cdev_add(&g_dev.cdev, dev, 1);
	if (ret) {
		printk("cdev_add failed!\n");
		goto CDEV_ADD_ERR;
	}

	if (!XXX_DEV_NUM) {
		pclass = class_create(THIS_MODULE, "xxx_dev");
		if (IS_ERR(pclass)) {
			printk("class_create failed!\n");
			ret = PTR_ERR(pclass);
			goto CLS_CRT_ERR;
		}
		device_create(pclass, NULL, MKDEV(g_dev.major, g_dev.minor), NULL, "xxx_dev");
	}

	return 0;

CLS_CRT_ERR:
	cdev_del(&g_dev.cdev);
CDEV_ADD_ERR:
	unregister_chrdev_region(dev, 1);
REG_DEV_ERR:
	return ret;
}

static void __exit xxx_exit(void)
{
	cdev_del(&g_dev.cdev);
	unregister_chrdev_region(dev, 1);
}

module_init(xxx_init);
module_exit(xxx_exit);
```








