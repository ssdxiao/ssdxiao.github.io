---

layout: post

title:  kprobe使用实例
date:   2015-12-10 14:10:00
categories: linux
tags: 
  - linux
---

linux提供了kprobe机制来方便我们进行调试。
kprobe的原理是通过替换函数表，使得调用函数前后，会调用在kprobe注册的函数。

下来就来说明下，kprobe的用法。

首先需要编译内核时开启CONFIG_KPROBES选项。
然后编写probe模块。


{%highlight c%}
#include <linux/kernel.h>
#include <linux/module.h>
#include <linux/kprobes.h>


/* 对于每个探测，用户需要分配一个kprobe对象*/
static struct kprobe kp = {
    .symbol_name    = "blkdev_ioctl",
};
 
/* 在被探测指令执行前，将调用预处理例程 pre_handler，用户需要定义该例程的操作*/
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
{
   
    /* 在这里可以调用内核接口函数dump_stack打印出栈的内容*/
    dump_stack();
    return 0;
}
 
/* 在被探测指令执行后，kprobe调用后处理例程post_handler */
static void handler_post(struct kprobe *p, struct pt_regs *regs,
                unsigned long flags)
{

}
 
/*在pre-handler或post-handler中的任何指令或者kprobe单步执行的被探测指令产生了例外时，会调用fault_handler*/
static int handler_fault(struct kprobe *p, struct pt_regs *regs, int trapnr)
{
    printk(KERN_DEBUG "fault_handler: p->addr = 0x%p, trap #%dn",
        p->addr, trapnr);
    /* 不处理错误时应该返回*/
    return 0;
}
 
/*初始化内核模块*/
static int __init kprobe_init(void)
{
    int ret;
    kp.pre_handler = handler_pre;
    kp.post_handler = handler_post;
    kp.fault_handler = handler_fault;
 
    ret = register_kprobe(&kp);  /*注册kprobe*/
    if (ret < 0) {
        printk(KERN_DEBUG "register_kprobe failed, returned %d\n", ret);
        return ret;
    }
    printk(KERN_DEBUG "Planted kprobe at %p\n", kp.addr);
    return 0;
}

static void __exit kprobe_exit(void)
{
    unregister_kprobe(&kp);
    printk(KERN_DEBUG "kprobe at %p unregistered\n", kp.addr);
}

module_init(kprobe_init)
module_exit(kprobe_exit)
MODULE_LICENSE("GPL");

{%endhighlight%}

上面是一个简单的probe模块例子。
在使用kprobe时最主要是需要明白pt_regs结构体中的各项。这样方便我们可以动态获取函数调用时的参数等信息。
pt_regs结构体存储了函数调用中的寄存器的值，所以需要我们明白参数的含义，来帮助我们debug

{%highlight c%}
struct pt_regs {
/*
 * C ABI says these regs are callee-preserved. They aren't saved on kernel entry
 * unless syscall needs a complete, fully filled "struct pt_regs".
 */
    unsigned long r15;
    unsigned long r14;
    unsigned long r13;
    unsigned long r12;
    unsigned long bp;
    unsigned long bx;
/* These regs are callee-clobbered. Always saved on kernel entry. */
    unsigned long r11;
    unsigned long r10;
    unsigned long r9;
    unsigned long r8;  
    unsigned long ax;  
    unsigned long cx;  //函数调用的第四个参数
    unsigned long dx; //函数调用的第三个参数
    unsigned long si; //函数调用的第二个参数
    unsigned long di; //函数调用的第一个参数
/*
 * On syscall entry, this is syscall#. On CPU exception, this is error code.
 * On hw interrupt, it's IRQ number:
 */
    unsigned long orig_ax;
/* Return frame for iretq */
    unsigned long ip;
    unsigned long cs;
    unsigned long flags;
    unsigned long sp;   //栈顶指针，有了这个指针，可以打印堆栈信息，来获得信息
    unsigned long ss;
/* top of stack page */
};
{%endhighlight%}

这里举个使用的pt_regs的例子,通过获取第一个参数，读取相关变量值。

比如想获取函数调用中queue_ra_show 的q->backing_dev_info.ra_pages的值。

{%highlight c%}
static ssize_t queue_ra_show(struct request_queue *q, char *page)
  {
      unsigned long ra_kb = q->backing_dev_info.ra_pages <<
                      (PAGE_CACHE_SHIFT - 10);                                        return queue_var_show(ra_kb, (page));
  }

{%endhighlight%}

我们可以再probe注册函数中这样写

{%highlight c%}
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
{
    struct request_queue *q = regs->di;
          
    printk("ra_pages =%d\n", q->backing_dev_info.ra_pages);
  
    return 0;
}

{%endhighlight%}

这样就可以从dmesg中拿到想要看到的信息。

