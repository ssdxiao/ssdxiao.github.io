---

layout: post

title:  "如何用zerocopy机制实现内核访问用户空间"

date:   2015-08-31 22:50:06

categories: 
  - linux
tags:
  - linux
---



在32位系统下，linux的地址空间划分以0xC0000000为分界线，以下为用户空间，以上为系统空间。64位下是使用0xffffffff80000000来进行划分。
每个程序运行的时候，都有自己独立的用户地址空间，内核空间则是共用的。如下图所示：

![progress]({{ site.baseurl }}/image/progress.png )



linux提供了标准的函数 copy_from_user,copy_to_user，来提供内核对用户空间的读写。
但是这种方式使用了memcpy来将数据从用户空间copy到内核空间，或者相反的操作，这样在大量数据交换时就会产生很大的性能损耗。
那么如何能够zerocopy来进行数据共享？

首先要注意的是内核是可以访问到全局每一个地址的。
所以如果进程通过syscall陷入到内核中时，内核有了进程的上下文，可以直接访问进程空间的任一逻辑地址。
验证程序如下，

程序主要实现了一个char设备，然后通过文件操作write来传递相关信息。内核直接对用户空间地址进行读写操作。

用户态程序，
{%highlight c%} 
#include <stdio.h>
#include <string.h>
#include <fcntl.h>

char buf[]="write in uspace";
int main()
{
    int fd;

    fd = open("/dev/hello", O_RDWR, S_IRUSR|S_IWUSR);

    if (fd > 0)
    {
        write(fd, buf, strlen(buf)+1);

        printf("read from kernel  %s\n", buf);

    }

    close(fd);
}

{% endhighlight %}

内核程序

{%highlight c%} 
#include <linux/module.h>
#include <linux/types.h>
#include <linux/fs.h>
#include <linux/errno.h>
#include <linux/mm.h>
#include <linux/sched.h>
#include <linux/init.h>
#include <linux/cdev.h>
#include <asm/io.h>
#include <linux/device.h>
#include <linux/uaccess.h>
#include <linux/kthread.h>
#include <linux/highmem.h>
#include <linux/slab.h>
#include <linux/kernel.h>
#include <linux/pagemap.h>

#define DEVICE_MAJOR 222

struct class *my_class;
static char buffer[ ]= "write in kspace";
struct _my_dev
{
    struct cdev cdev;
} my_dev;

int my_cdev_open(struct inode *node, struct file *filp)
{
   return 0;
}

int my_cdev_write(struct file *file,  char __user *buf, size_t count, loff_t *f_pos)
{

    printk("from_user read %s\n", buf);
    memcpy(buf, buffer, sizeof(buffer));

}

static const struct file_operations my_cdev_fops =
{
    .owner = THIS_MODULE,
    .open = my_cdev_open,
    .write = my_cdev_write,

    
};


static int __init cdev_init_demo(void)
{
    int result;
    
    dev_t cdev_demo = MKDEV(DEVICE_MAJOR, 0);
        
    printk("module init\n");
    
    result = register_chrdev_region(cdev_demo, 1, "mycdev" );
    
    if (result < 0)
    {
        return result;
    }

    cdev_init(&my_dev.cdev, &my_cdev_fops);
    my_dev.cdev.owner = THIS_MODULE;
   
    result = cdev_add(&my_dev.cdev, cdev_demo, 1);

    if (result)
    {
        printk(" Error %d cdev_add\n", result);
        return result;
    }

    my_class = class_create(THIS_MODULE, "myclass");

    if(IS_ERR(my_class))
    {
        printk("Err: failed in creating class\n");
        return -1;
    }

    device_create(my_class, NULL, MKDEV(DEVICE_MAJOR, 0), NULL, "hello");

    return 0;
}
   
static void __exit cdev_exit_demo(void)
{
    printk("module exit\n");
    device_destroy(my_class, MKDEV(DEVICE_MAJOR, 0));
    class_destroy(my_class);
    cdev_del(&my_dev.cdev);
    unregister_chrdev_region( MKDEV(DEVICE_MAJOR, 0), 1);
}


MODULE_AUTHOR("Ding Xiao");
MODULE_LICENSE("Dual BSD/GPL");

module_init(cdev_init_demo);
module_exit(cdev_exit_demo);


{% endhighlight %}


这样的访问方式有着很大的局限性，这个内核态必须要具备用户空间的信息，而且只能在返回之前使用，内核中其他进程就无法通过这个地址来访问用户空间的地址。


函数get_user_pages提供了从用户空间的地址获取其物理页的方法。
所以这样我们来实现一个更加通用的读写方式，先获取物理page列表，然后再把page用kmap进行映射重新获取新的地址，来访问用户空间的信息。

内核程序

{%highlight c%} 

#include <linux/module.h>
#include <linux/types.h>
#include <linux/fs.h>
#include <linux/errno.h>
#include <linux/mm.h>
#include <linux/sched.h>
#include <linux/init.h>
#include <linux/cdev.h>
#include <asm/io.h>
#include <linux/device.h>
#include <linux/uaccess.h>
#include <linux/kthread.h>
#include <linux/highmem.h>
#include <linux/slab.h>
#include <linux/kernel.h>
#include <linux/pagemap.h>

#define DEVICE_MAJOR 222
#define MAX_MAP_PAGE 20

struct class *my_class;

struct _data
{
    struct page **p;
    int res;
    unsigned long offset;
} data;
static char buffer[] = "write in kspace";

struct task_struct *task=NULL;

struct _my_dev
{
    struct cdev cdev;
} my_dev;

int my_cdev_open(struct inode *node, struct file *filp)
{
   return 0;
}

int my_cdev_write(struct file *file,  char __user *buf, size_t count, loff_t *f_pos)
{

    unsigned long end=((unsigned long)buf + count + PAGE_SIZE -1) >> PAGE_SHIFT;
    unsigned long start = (unsigned long)buf >> PAGE_SHIFT;
    unsigned long offset = (unsigned long)buf - (start << PAGE_SHIFT);
    const int nr_pages = end -start;
    struct page **pages;

    int res, i;
    
    if ((pages= kmalloc(MAX_MAP_PAGE*sizeof(*pages), GFP_KERNEL)) == NULL)
        return -1;

    down_read(&current->mm->mmap_sem);
    
    res = get_user_pages(current, current->mm, buf, nr_pages, 1,0, pages, NULL);
    if (res < nr_pages)
    {
        goto out_unmap;
    }

    up_read(&current->mm->mmap_sem);

    data.res = res;
    data.offset =offset;
    data.p = pages;
   
    while(1)
    {
        if (!data.p)
        {
            break;
        }
        schedule_timeout(HZ*5);
    }
    return 0;

out_unmap:
    if(res > 0)
    {
        for(i=0; i < res; i++)
        {
            page_cache_release(pages[i]);
            res=0;
        }

    }
    kfree(pages);
    return -1;
}

static const struct file_operations my_cdev_fops =
{
    .owner = THIS_MODULE,
    .open = my_cdev_open,
    .write = my_cdev_write,

    
};


int kthread_demo(void *data)
{
    struct _data *d=( struct _data *)data;
    int i;
    char * addr_map_from_user;
    while(!kthread_should_stop())
    {
        if (d->p)
        {

            addr_map_from_user = ((char*)kmap(*(d->p)) + d->offset);
            printk ("from_user read %s\n ", addr_map_from_user);
            memcpy(addr_map_from_user, buffer, sizeof(buffer));
             
            kunmap(*(d->p));

            for(i=0; i<d->res; i++)
            {
                page_cache_release(d->p[i]);
            }
            kfree(d->p);
            d->p = NULL;
            d->res = 0;
        
        }

        set_current_state(TASK_INTERRUPTIBLE);
        schedule_timeout(5*HZ);
    }
}

static int __init cdev_init_demo(void)
{
    int result;
    
    dev_t cdev_demo = MKDEV(DEVICE_MAJOR, 0);
        
    printk("module init\n");
    
    result = register_chrdev_region(cdev_demo, 1, "mycdev" );
    
    if (result < 0)
    {
        return result;
    }

    cdev_init(&my_dev.cdev, &my_cdev_fops);
    my_dev.cdev.owner = THIS_MODULE;
   
    result = cdev_add(&my_dev.cdev, cdev_demo, 1);

    if (result)
    {
        printk(" Error %d cdev_add\n", result);
        return result;
    }

    my_class = class_create(THIS_MODULE, "myclass");

    if(IS_ERR(my_class))
    {
        printk("Err: failed in creating class\n");
        return -1;
    }

    device_create(my_class, NULL, MKDEV(DEVICE_MAJOR, 0), NULL, "hello");

    data.p=NULL;
    task = kthread_create(kthread_demo, (void*)&data, "map-kthread");
    wake_up_process(task);
    return 0;
}
   


static void __exit cdev_exit_demo(void)
{
    printk("module exit\n");
    device_destroy(my_class, MKDEV(DEVICE_MAJOR, 0));
    class_destroy(my_class);
    cdev_del(&my_dev.cdev);
    unregister_chrdev_region( MKDEV(DEVICE_MAJOR, 0), 1);

    if (task)
    {
        kthread_stop(task);
        task=NULL;
    }
}


MODULE_AUTHOR("Ding Xiao");
MODULE_LICENSE("Dual BSD/GPL");

module_init(cdev_init_demo);
module_exit(cdev_exit_demo);


{% endhighlight %}


上面只是实现了单页面跨内核线程的zerocopy，对于多页面该怎么做呢？
需要注意一点，在用户空间连续的地址空间，如果超过一页，在内核是会被取到多个page，
这样如果要读取page信息需要分别kamp页面，并通过各自的偏移来读取信息。





