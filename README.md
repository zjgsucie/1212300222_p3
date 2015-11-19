##1212300222_p3

钟楠 1212300222 unix系统分析作业3

分析一个字符设备驱动程序结构。 [参考blog](http://www.cnblogs.com/myblesh/articles/2364962.html)

+ 字符设备结构体描述：cdev

```
　　struct cdev{

　　　　struct kobject kobj;/*内嵌的kobject对象*/

　　　　strcut module *owner;/*所属模块*/

　　　　struct file_operations *ops;/*文件操作结构体*/

　　　　struct list_head list;

　　　　dev_t dev;/*设备号，dev_t实质是一个32位整，12位为主设备号，20位为次设备号，

　　　　　　　　　　　　提取主次设备号的方法：MAJOR（dev_t dev），MINOR（dev_t dev），生成dev_t的方法：MKDEV（int major，int minor）*/

　　　　unsigned int count;

　　};
```

+ linux2.6内核提供了一组函数来操作cdev结构体

```
　　void cdev_init(struct cdev *,struct file_operations *);/*初始化cdev的成员，并且建立cdev与file_operation的连接*/

　　struct cdev *cdev_alloc(void);/*动态申请一个cdev的内存空间*/

　　void cdev_put(struct cdev *p);

 

　　int cdev_add(struct cdev *,dev_t,unsigned);/*添加一个cdev，完成字符的注册*/

　　void cdev_del(struct cdev *);/*删除一个cdev，完成字符的注销*/

　　在调用cdev_add函数向系统注册字符设备之前，应该先调用register_chrdev_region()函数或是alloc_chrdev_region()函数向系统申请设备号；模型为：

　　int register_chrdev_region(dev_t from,unsigned count,const char *name);

　　int alloc_chrdev_region(dev_t *dev,unsigned baseminor,unsigned count,const char *name)

　　在系统调用cdev_del函数从系统注销字符设备后，unregister_chrdev_region()应该释放之前申请的设备号，该函数原型为：

　　unregister_chrdev_region(dev_t from,unsigned count)
```

+ 以下代码基于虚拟的globalmem设备进行分析，具体在linux中设备驱动远比这个复杂。

```
/*
* A globalmem driver as an example of char device drivers
*
* The initial developer of the original code is Barry Song
* <author@linuxdriver.cn>. All Rights Reserved.
*/

#include <linux/module.h>
#include <linux/types.h>
#include <linux/fs.h>
#include <linux/errno.h>
#include <linux/mm.h>
#include <linux/sched.h>
#include <linux/init.h>
#include <linux/cdev.h>
#include <asm/io.h>
#include <asm/system.h>
#include <asm/uaccess.h>

#define GLOBALMEM_SIZE 0x1000 /*全局内存最大4K字节*/
#define MEM_CLEAR 0x1 /*清0全局内存*/
#define GLOBALMEM_MAJOR 250 /*预设的globalmem的主设备号*/

static int globalmem_major = GLOBALMEM_MAJOR;
/*globalmem设备结构体*/
struct globalmem_dev {
struct cdev cdev; /*cdev结构体*/
unsigned char mem[GLOBALMEM_SIZE]; /*全局内存*/
};

struct globalmem_dev *globalmem_devp; /*设备结构体指针*/
/*文件打开函数*/
int globalmem_open(struct inode *inode, struct file *filp)
{
/*将设备结构体指针赋值给文件私有数据指针*/
filp->private_data = globalmem_devp;
return 0;
}
/*文件释放函数*/
int globalmem_release(struct inode *inode, struct file *filp)
{
return 0;
}

/* ioctl设备控制函数 */
static int globalmem_ioctl(struct inode *inodep, struct file *filp, unsigned
int cmd, unsigned long arg)
{
struct globalmem_dev *dev = filp->private_data;/*获得设备结构体指针*/

switch (cmd) {
case MEM_CLEAR:
memset(dev->mem, 0, GLOBALMEM_SIZE);
printk(KERN_INFO "globalmem is set to zero\n");
break;

default:
return - EINVAL;
}

return 0;
}

/*读函数*/
static ssize_t globalmem_read(struct file *filp, char __user *buf, size_t size,
loff_t *ppos)
{
unsigned long p = *ppos;
unsigned int count = size;
int ret = 0;
struct globalmem_dev *dev = filp->private_data; /*获得设备结构体指针*/

/*分析和获取有效的写长度*/
if (p >= GLOBALMEM_SIZE)
return 0;
if (count > GLOBALMEM_SIZE - p)
count = GLOBALMEM_SIZE - p;

/*内核空间->用户空间*/
if (copy_to_user(buf, (void *)(dev->mem + p), count)) {
ret = - EFAULT;
} else {
*ppos += count;
ret = count;

printk(KERN_INFO "read %u bytes(s) from %lu\n", count, p);
}

return ret;
}

/*写函数*/
static ssize_t globalmem_write(struct file *filp, const char __user *buf,
size_t size, loff_t *ppos)
{
unsigned long p = *ppos;
unsigned int count = size;
int ret = 0;
struct globalmem_dev *dev = filp->private_data; /*获得设备结构体指针*/

/*分析和获取有效的写长度*/
if (p >= GLOBALMEM_SIZE)
return 0;
if (count > GLOBALMEM_SIZE - p)
count = GLOBALMEM_SIZE - p;

/*用户空间->内核空间*/
if (copy_from_user(dev->mem + p, buf, count))
ret = - EFAULT;
else {
*ppos += count;
ret = count;

printk(KERN_INFO "written %u bytes(s) from %lu\n", count, p);
}

return ret;
}

/* seek文件定位函数 */
static loff_t globalmem_llseek(struct file *filp, loff_t offset, int orig)
{
loff_t ret = 0;
switch (orig) {
case 0: /*相对文件开始位置偏移*/
if (offset < 0) {
ret = - EINVAL;
break;
}
if ((unsigned int)offset > GLOBALMEM_SIZE) {
ret = - EINVAL;
break;
}
filp->f_pos = (unsigned int)offset;
ret = filp->f_pos;
break;
case 1: /*相对文件当前位置偏移*/
if ((filp->f_pos + offset) > GLOBALMEM_SIZE) {
ret = - EINVAL;
break;
}
if ((filp->f_pos + offset) < 0) {
ret = - EINVAL;
break;
}
filp->f_pos += offset;
ret = filp->f_pos;
break;
default:
ret = - EINVAL;
break;
}
return ret;
}

/*文件操作结构体*/
static const struct file_operations globalmem_fops = {
.owner = THIS_MODULE,
.llseek = globalmem_llseek,
.read = globalmem_read,
.write = globalmem_write,
.ioctl = globalmem_ioctl,
.open = globalmem_open,
.release = globalmem_release,
};

/*初始化并注册cdev*/
static void globalmem_setup_cdev(struct globalmem_dev *dev, int index)
{
int err, devno = MKDEV(globalmem_major, index);

cdev_init(&dev->cdev, &globalmem_fops);
dev->cdev.owner = THIS_MODULE;
err = cdev_add(&dev->cdev, devno, 1);
if (err)
printk(KERN_NOTICE "Error %d adding LED%d", err, index);
}

/*设备驱动模块加载函数*/
int globalmem_init(void)
{
int result;
dev_t devno = MKDEV(globalmem_major, 0);

/* 申请设备号*/
if (globalmem_major)
result = register_chrdev_region(devno, 1, "globalmem");
else { /* 动态申请设备号 */
result = alloc_chrdev_region(&devno, 0, 1, "globalmem");
globalmem_major = MAJOR(devno);
}
if (result < 0)
return result;

/* 动态申请设备结构体的内存*/
globalmem_devp = kmalloc(sizeof(struct globalmem_dev), GFP_KERNEL);
if (!globalmem_devp) { /*申请失败*/
result = - ENOMEM;
goto fail_malloc;
}

memset(globalmem_devp, 0, sizeof(struct globalmem_dev));

globalmem_setup_cdev(globalmem_devp, 0);
return 0;

fail_malloc:
unregister_chrdev_region(devno, 1);
return result;
}

/*模块卸载函数*/
void globalmem_exit(void)
{
cdev_del(&globalmem_devp->cdev); /*注销cdev*/
kfree(globalmem_devp); /*释放设备结构体内存*/
unregister_chrdev_region(MKDEV(globalmem_major, 0), 1); /*释放设备号*/
}

MODULE_AUTHOR("Barry Song <21cnbao@gmail.com>");
MODULE_LICENSE("Dual BSD/GPL");

module_param(globalmem_major, int, S_IRUGO);

module_init(globalmem_init);
module_exit(globalmem_exit);
```