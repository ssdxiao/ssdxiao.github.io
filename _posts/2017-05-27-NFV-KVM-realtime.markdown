---
layout: post
title:  "NFV场景下优化KVM——低时延"
date:   2017-5-27 21:13:33
categories:
  - linux
tags:
  - linux
  - NFV
---

前言
---
NFV（网络功能虚拟化）由运营商联盟提出，为了加速部署新的网络服务，运营商倾向于放弃笨重昂贵的专用网络设备，转而使用标准的IT虚拟化技术来拆分网络功能模块。在传统CT网络中对于服务质量保证的要求是非常高的，简而言之就是高带宽，高可靠，低时延。这些要求对于虚拟化技术而言本身就具有非常大的挑战。在虚拟化场景下，Hypervisor对vcpu本身要进行调度，而在Guest中vcpu对正在运行的程序也需要调度。这种二层调度使得低时延在虚拟化场景下是一个很难解决的问题。而解决低时延很大程度上需要使用实时操作系统


实时性优化思路
---
实时性就是指能够在规定的时间内已足够快的速度予以处理，这要求操作系统能够及时的响应各种中断事件。并且在调度算法上要求中断处理进程能够得到及时的调度。以此来保证事件的处理。
在虚拟化场景中涉及操作系统有两个。Hypervisor操作系统和Guest操作系统。Hyperviosr操作系统中主要跑vcpu的进程，硬件中断处理程序，以及管理程序等等。为了提高实时性需要修改Hyperviosr调度算法，进程锁，中断屏蔽等机制。在Guest操作系统中也需要使用实时操作系统来进行优化。同时，为了保证Guest的vcpu在Hypervisor上得到及时的调度，需要在Hyperviosr对vcpu进行隔离，物理cpu和vcpu进行绑定，减少vcpu调度到其他cpu上，保证cache命中率。

实时KVM优化
---
Hyperviosr侧的优化

* 首先要将linux修改成实时系统。

  1. PREEMPT_RT

   PREEMPT_RT是针对kernel进行修改，使得kernel成为实时操作系统的patch。它主要优化了spinlocks，interrrupts等，减少了不能够相应中断的操作。

   相应可以在这里找到https://rt.wiki.kernel.org/index.php/Main_Page

* 然后调整内核参数，以满足实时性的要求。

  1. cpu隔离/cpu绑定

   在kernel启动时候通过isolcups进行cpu隔离，使得Hyperviso上的控制程序跑在其他cpu中，隔离出来的cpu主要给Guest使用。然后在创建虚拟机的时候同时使用cpu绑定技术，将Guest的cpu绑定到被隔离的cpu上。

  2. 减少cpu中断

  nohz_full 提供一种动态的无时钟设置,在内核”CONFIG_NO_HZ_FULL=y”的前提下，指定哪些CPU核心可以进入完全无滴答状态。配置了CONFIG_NO_HZ_FULL=y后，当cpu上只有一个任务在跑的时候，不发送调度时钟中断到此cpu也就是减少调度时钟中断，不中断空闲CPU，从而可以减少耗电量和减少系统抖动。默认情况下所有的cpu都不会进入这种模式，需要通过nohz_full参数指定那些cpu进入这种模式。

  rcu_nocbs 当cpu有RCU callbacks pending的时候,nohz_full设置可能不会生效，使用rcu_nocbs来指定cpu进行卸载RCU callback processing

  mce=off 彻底禁用MCE(Machine Check Exception)。MCE是用来报告主机硬件相关问题的一种日志机制.

  3. 强制cpu进入高性能状态

  idle=poll 从根本上禁用休眠功能(也就是禁止进入C-states状态)

  intel_pstate=disable禁用 Intel CPU 的 P-state 驱动(CONFIG_X86_INTEL_PSTATE)，也就是Intel CPU专用的频率调节器驱动P-state表明处理器处于省电模式但仍旧在执行一些工作。

  processor.max_cstate 无视ACPI表报告的值，强制指定CPU的最大C-state值(必须是一个有效值)：C0为正常状态，其他则为不同的省电模式(数字越大表示CPU休眠的程度越深/越省电)。
  
  tsc=reliable 表示TSC时钟源是绝对稳定的，关闭启动时和运行时的稳定性检查。

具体参数见如下设置
{%highlight bash%}
  isolcpus=1-4 nohz_full=1-4 rcu_nocbs=1-4 mce=off idle=poll intel_pstate=disable processor.max_cstate=1 pcie_asmp=off tsc=reliable
{%endhighlight%}


* 关闭Hypervisor侧影响性能的程序
  
  1. 关闭内存交换
  {%highlight bash%}
   swapoff -a
  {%endhighlight%}

  2. 关闭ksm

  {%highlight bash%}
   echo 0 > /sys/kernel/mm/ksm/merge_across_nodes
   echo 0 > /sys/kernel/mm/ksm/run
  {%endhighlight%}
 
  3. 关闭看门狗 
  {%highlight bash%}
  echo 0 > /proc/sys/kernel/watchdog
  echo 0 > /proc/sys/kernel/nmi_watchdog
  {%endhighlight%}

  4. 调整隔离cpu上的ksoftirqd和rcuc的优先级
  
  {%highlight bash%}
  host_isolcpus="1-4"
  startVal=$(echo ${host_isolcpus} | cut -f1 -d-)
  endVal=$(echo ${host_isolcpus} | cut -f2 -d-)
  i=0
  while [ ${startVal} -le ${endVal} ]; do
    tid=`pgrep -a ksoftirq | grep "ksoftirqd/${startVal}$" | cut -d ' ' -f 1`
    chrt -fp 2 ${tid}

    tid=`pgrep -a rcuc | grep "rcuc/${startVal}$" | cut -d ' ' -f 1`
    chrt -fp 3 ${tid}

    cpu[$i]=${startVal}
    i=`expr $i + 1`
    startVal=`expr $startVal + 1`
  done
  {%endhighlight%}
   
  5. 禁止带宽限制
  {%highlight bash%}
   echo -1 > /proc/sys/kernel/sched_rt_period_us
   echo -1 > /proc/sys/kernel/sched_rt_runtime_us
  {%endhighlight%}
   
  6.设置中断亲和性
   将中断都设置到0号cpu上。
  {%highlight bash%}
  for irq in /proc/irq/* ; do
   echo 0 > ${irq}/smp_affinity_list
  done
  {%endhighlight%}

虚拟机的启动设置
 * 将vcpu绑定都内核隔离出来的物理cpu中
   {%highlight bash%}
    <cputune>
       <vcpupin vcpu="0" cpuset="1"/>
       <vcpupin vcpu="1" cpuset="2"/>
       <vcpupin vcpu="2" cpuset="3"/>
       <vcpupin vcpu="3" cpuset="4"/>
    </cputune>
   {%endhighlight%}

* 禁用kvmclock 使用tsc
   {%highlight bash%}
   <clock offset='utc'>
      <timer name='kvmclock' present='no'/>
   </clock>
   {%endhighlight%}

虚拟机内部优化

* 启动项优化
{%highlight bash%}
isolcpus=3 nohz_full=3 rcu_nocbs=3  mce=off idle=poll
{%endhighlight%}

* 虚拟机内部其他优化

{%highlight bash%}
 swapoff -a
 echo 0 > /sys/kernel/mm/ksm/merge_across_nodes
 echo 0 > /sys/kernel/mm/ksm/run
 echo 0 > /proc/sys/kernel/watchdog
 echo 0 > /proc/sys/kernel/nmi_watchdog
 echo -1 > /proc/sys/kernel/sched_rt_period_us
 echo -1 > /proc/sys/kernel/sched_rt_runtime_us
{%endhighlight%}

* 禁止时钟迁移
{%highlight bash%}
echo 0 > /proc/sys/kernel/timer_migration
{%endhighlight%}


实时性测试
---

对于实时性的测试，采用的是cyclictest。
在10分钟内测试下cpu的实时性。具体命令如下
{%highlight bash%}
cyclictest -m -n -p95 -h60 -i200 -D 10m
{%endhighlight%}


测试结果
优化前
{%highlight bash%}
# /dev/cpu_dma_latency set to 0us
policy: fifo: loadavg: 0.00 0.01 0.03 1/129 2407          

T: 0 ( 2404) P:95 I:200 C:2999375 Min:      5 Act:   22 Avg:   16 Max:    1475
{%endhighlight%}

优化后
{%highlight bash%}
# /dev/cpu_dma_latency set to 0us
policy: fifo: loadavg: 0.00 0.01 0.01 1/152 2441

T: 0 ( 2440) P:95 I:200 C:2999967 Min:  6 Act:7 Avg:7 Max:  12
{%endhighlight%}

可以看到优化前在实时性方面原始的KVM还是会出现毛刺。在优化后基本上能达到很好的结果。

作者简介
---
* 肖丁 
烽火云计算高级虚拟化设计师，多年从事云计算产品的架构设计、软件开发与技术方案编制等工作。长期专注于内核、虚拟化、云计算、容器、分布式等方向的研究，尤其对KVM和XEN虚拟化等产品有较深研究。


