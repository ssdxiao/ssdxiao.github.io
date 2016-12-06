---
layout: post
title:  "linux的jiffies"
date:   2016-12-06 21:13:31
categories: 
  - linux
tags:
  - linux
---

在看libvirt如何获取虚拟机的cpu占用率这个问题。计算cpu占用率不可避免的需要直到jiffies的概念。


jiffies是内核中的一个全局变量，用来记录自系统启动一来产生的节拍数。简单描述，就是1s内，内核发起的时钟中断次数。kernel中就使用这个来对程序的运行时间进行统计。这个值实在编译内核时进行设置的。一般有100，250，1000。CONFIG_HZ这个参数就是用来设置1s中的中断次数的。


查看 /proc/stat 可以看到kernel对cpu使用的各项时间进行的统计。
{%highlight bash%}
cat /proc/stat

cpu  1776 10 9610 35061503 4272 0 122 0 0 0
cpu0 359 0 1820 8767364 1812 0 40 0 0 0
cpu1 398 0 1578 8764789 688 0 18 0 0 0
cpu2 593 3 1646 8768085 1076 0 8 0 0 0
cpu3 425 6 4564 8761263 694 0 56 0 0 0

{% endhighlight %}

cpu的使用率一般包含 

user ( 1776 )    从系统启动开始累计到当前时刻，处于用户态的运行时间，不包含 nice值为负进程。

nice (10)      从系统启动开始累计到当前时刻，nice值为负的进程所占用的CPU时间

system (35061503)  从系统启动开始累计到当前时刻，处于核心态的运行时间

idle (4272)  从系统启动开始累计到当前时刻，除IO等待时间以外的其它等待时间

iowait (0) 从系统启动开始累计到当前时刻，IO等待时间

irq (122)          从系统启动开始累计到当前时刻，硬中断时间

softirq (0)      从系统启动开始累计到当前时刻，软中断时间

stealstolen(0)    which is the time spent in other operating systems when running in a virtualized environment

guest(0)        which is the time spent running a virtual  CPU  for  guest operating systems under the control of the Linux kernel

那么如果采样时间为1s中，那么上面所有项加起来应该就是一个固定值。即CONFIG_HZ的值。

计算cpu的使用率，一般使用1 - idle/(user+nice+system+idle+iowait+irq+softirq+stealstolen+guest)


那么对于一个进程实际使用给的cpu时间怎么查看呢?
在/proc/pid/stat 中显示如下

{%highlight bash%}
29341 (nova-compute) S 1 29341 29341 0 -1 4219136 3167457 1467516 0 0 8211 2001 768 804 20 0 22 0 9453342 2382286848 26918 18446744073709551615 4194304 7057652 140733366069296 140733366045760 139709566769683 0 0 16781312 16899 18446744073709551615 0 0 17 6 0 0 0 0 0 9158080 9634804 38973440 140733366070976 140733366071090 140733366071090 140733366071266 0
{% endhighlight %}

这里也记录了程序使用的时间片。
要看懂这些信息可以使用man proc 进行查看。这里只说下和cpu占用率有关的参数。
第14个和第15个 分别代表了程序在用户态使用的时间，和内核态使用的时间。

{%highlight bash%}
 utime %lu   (14) Amount of time that this process has been scheduled in user mode, measured in clock ticks  (divide by  sysconf(_SC_CLK_TCK)).  This includes guest time, guest_time (time spent running a virtual CPU, see below), so that applications that are not aware of the guest time field do  not  lose  that  time  from their calculations.

stime %lu   (15)  Amount  of  time  that  this  process  has been scheduled in kernel mode, measured in clock ticks (divide by sysconf(_SC_CLK_TCK)).
{% endhighlight %}

但是这里却和刚才的jiffies不同，jiffies的次数是用来给kernel进行时间内记录的。
然而在用户态的程序中，使用的却是sysconf(_SC_CLK_TCK)来记录1s中中断的次数。
这个值可以通过如下命令查看。

{%highlight bash%}
getconf CLK_TCK
{% endhighlight %}

因此计算程序的cpu占用率，可以用如下方式
{%highlight c%}
 *cpuTime = 1000ull * 1000ull * 1000ull * (usertime + systime)
            / (unsigned long long)sysconf(_SC_CLK_TCK);

{% endhighlight %}

而这正式libvirt用来统计qemu的cpu占用率所用的公式。
