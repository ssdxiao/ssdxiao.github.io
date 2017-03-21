---
layout: post
title:  "Virtio-Balloon超详细分析"
date:   2017-3-20 21:13:31
categories:
  - linux
tags:
  - linux
---

前言
---
Virtio-Balloon驱动即内存气泡，可以用来动态调整内存，这个驱动很早就已经出来了，
最近才发现这个驱动居然还有监控内存指标的功能，所以再一次review了一遍代码。这里就把这个驱动详细的介绍一遍。


功能介绍
---
Virtio-Balloon驱动就是通过在虚拟机中申请内存，然后将申请的内存通知给qemu，然后qemu侧再将内存释放掉，可以让其他虚拟机使用，达到内存复用的能力。

要开启这个功能，就是在启动时候需要添加Virtio-Balloon后端设备，并且要在Guest里面安装
Virtio-Balloon驱动。


安装
---
* 添加设备
如果用libvirt启动 在libvirt的虚拟机配置侧添加如下xml
{%highlight bash%}
 <devices>
 <memballoon model='virtio'>
      <alias name='balloon0'/>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x06' function='0x0'/>
      <stats period='10'/>
    </memballoon>
  </devices>
{% endhighlight %}

在qemu中启动中添加如下设备
{%highlight bash%}
-device virtio-balloon-pci,id=balloon0,bus=pci.0,addr=0×4
{% endhighlight %}

* 安装驱动
windows驱动这里省略了在网上教程很多。
linux下在kernel 很早就加入了Virtio-Balloon驱动，所以在主流linux发行版中一般都具有这个驱动。
{%highlight bash%}
[root@localhost _posts]# modinfo virtio-balloon
filename:       /lib/modules/3.10.0-327.el7.x86_64/kernel/drivers/virtio/virtio_balloon.ko
license:        GPL
description:    Virtio balloon driver
rhelversion:    7.2
srcversion:     F2D65C53D0AFD06A3668942
alias:          virtio:d00000005v*
depends:        virtio,virtio_ring
intree:         Y
vermagic:       3.10.0-327.el7.x86_64 SMP mod_unload modversions 
signer:         CentOS Linux kernel signing key
sig_key:        79:AD:88:6A:11:3C:A0:22:35:26:33:6C:0F:82:5B:8A:94:29:6A:B3
sig_hashalgo:   sha256
{% endhighlight %}


使用
---
有了上面的准备，启动虚拟机后就可以体验到内存气泡的功能。

* 查看内存：
在libvirt侧使用 
{%highlight bash%}
virsh # dommemstat test

{% endhighlight %}
在qemu的hmp中查看内存
{%highlight bash%}
info balloon

{% endhighlight %}

* 设置目标虚拟机的内存，注意这里是直接设置虚拟机当前可以使用的内存。
比如起始的时候内存是8G，然后想缩减2G，实际这里操作要设置成目前为6G。

libvirt侧
{%highlight bash%}
virsh # setmem test 4096

{% endhighlight %}
在qemu的hmp中查看内存
{%highlight bash%}
balloon 4096
{% endhighlight %}

代码分析
---
从代码侧来具体分析Virtio-Balloon的。先从linux驱动侧看起吧
相对其他virtio驱动 ，内存气泡的驱动相对简单好看。
它的代码全部哎driver/virtio/virtio_balloon.c中

首先来看 virtio_balloon_driver驱动的定义，
这里很容易看到驱动的处理函数并不多virtballoon_probe，在驱动加载时候，
virtballoon_remove在驱动卸载时候，剩下只有virtballoon_changed那么肯定所有的功能都是在这个里面来实现。

{%highlight c%}
static struct virtio_driver virtio_balloon_driver = {
    .feature_table = features,
    .feature_table_size = ARRAY_SIZE(features),
    .driver.name =  KBUILD_MODNAME,
    .driver.owner = THIS_MODULE,
    .id_table = id_table,
    .probe =    virtballoon_probe,
    .remove =   virtballoon_remove,
    .config_changed = virtballoon_changed,
#ifdef CONFIG_PM_SLEEP
    .freeze =   virtballoon_freeze,
    .restore =  virtballoon_restore,
#endif
};
{% endhighlight %}

virtballoon_changed的函数也相当简单，就是启动了一个work
这个work主要来执行update_balloon_size_work 这个回调。
{%highlight c%}
static void virtballoon_changed(struct virtio_device *vdev)
{
    struct virtio_balloon *vb = vdev->priv;
    unsigned long flags;

    spin_lock_irqsave(&vb->stop_update_lock, flags);
    if (!vb->stop_update)
        queue_work(system_freezable_wq, &vb->update_balloon_size_work);
    spin_unlock_irqrestore(&vb->stop_update_lock, flags);
}
{% endhighlight %}

为了弄清楚这个work我们还是需要看一下virtio_balloon的初始化函数virtballoon_probe。
截取片段来看，初始化时候就注册了stats_request 这个callback，用来响应后端发来的stats请求。定义了2个任务 update_balloon_size_func是用来修改内存，
而update_balloon_stats_func这个是用来更新内存信息的，这个也是再次浏览代码时候的发现。


{%highlight c%}
static int virtballoon_probe(struct virtio_device *vdev)
{
  struct virtqueue *vqs[3];
    vq_callback_t *callbacks[] = { balloon_ack, balloon_ack, stats_request };
    static const char * const names[] = { "inflate", "deflate", "stats" };
    int err, nvqs;

    nvqs = virtio_has_feature(vb->vdev, VIRTIO_BALLOON_F_STATS_VQ) ? 3 : 2;
    err = vb->vdev->config->find_vqs(vb->vdev, nvqs, vqs, callbacks, names,
            NULL);

 INIT_WORK(&vb->update_balloon_stats_work, update_balloon_stats_func);
 INIT_WORK(&vb->update_balloon_size_work, update_balloon_size_func);
}
{% endhighlight %}

先来介绍基本功能update_balloon_size_func，代码非常好理解，拿到diff和本身现在的内存进行比较，然后开始增加或者减少内存，接着update_balloon_size更新下当前内存的信息。

{%highlight c%}
static void update_balloon_size_func(struct work_struct *work)
{
    struct virtio_balloon *vb;
    s64 diff;

    vb = container_of(work, struct virtio_balloon,
              update_balloon_size_work);
    diff = towards_target(vb);

    if (diff > 0)
        diff -= fill_balloon(vb, diff);
    else if (diff < 0)
        diff += leak_balloon(vb, -diff);
    update_balloon_size(vb);

    if (diff)
        queue_work(system_freezable_wq, work);
}
{% endhighlight %}

fill_balloon和leak_alloon其实很相似，这里就看下fill_balloon的实现。
调用balloon_page_enqueue函数进行内存的添加，然后tell_host去更新表项。
balloon_page_enqueue函数是kernel自己实现的堆内存进行增减的功能。
所以无论是kvm还是xen的气球驱动，最终都是调用这个函数去进行实现。
该函数在mm/balloon_compaction.c。这里就不继续往下追了，本编只是介绍virtio-balloon驱动原理。要想更细一步了解内存的实现，在以后专门讲内存操作时会再次提及该函数。

{%highlight c%}
static unsigned fill_balloon(struct virtio_balloon *vb, size_t num)
{
    struct balloon_dev_info *vb_dev_info = &vb->vb_dev_info;
    unsigned num_allocated_pages;

    /* We can only do one array worth at a time. */
    num = min(num, ARRAY_SIZE(vb->pfns));

    mutex_lock(&vb->balloon_lock);
    for (vb->num_pfns = 0; vb->num_pfns < num;
         vb->num_pfns += VIRTIO_BALLOON_PAGES_PER_PAGE) {
        struct page *page = balloon_page_enqueue(vb_dev_info);

        if (!page) {
            dev_info_ratelimited(&vb->vdev->dev,
                         "Out of puff! Can't get %u pages\n",
                         VIRTIO_BALLOON_PAGES_PER_PAGE);
            /* Sleep for at least 1/5 of a second before retry. */
            msleep(200);
            break;
        }
        set_page_pfns(vb, vb->pfns + vb->num_pfns, page);
        vb->num_pages += VIRTIO_BALLOON_PAGES_PER_PAGE;
        if (!virtio_has_feature(vb->vdev,
                    VIRTIO_BALLOON_F_DEFLATE_ON_OOM))
            adjust_managed_page_count(page, -1);
    }

    num_allocated_pages = vb->num_pfns;
    /* Did we get any? */
    if (vb->num_pfns != 0)
        tell_host(vb, vb->inflate_vq);
    mutex_unlock(&vb->balloon_lock);

    return num_allocated_pages;
}

{% endhighlight %}

这个基本功能介绍完了，回到本篇最初的问题，Virtio-Balloon驱动顺便还实现了一个内存的定时监控信息。
刚才在初始化的时候可以看到当后端主动调用stat指令时候，stats_request就会触发另一个work update_balloon_stats_func。

{%highlight c%}
static void stats_request(struct virtqueue *vq)
{
    struct virtio_balloon *vb = vq->vdev->priv;

    spin_lock(&vb->stop_update_lock);
    if (!vb->stop_update)
        queue_work(system_freezable_wq, &vb->update_balloon_stats_work);
    spin_unlock(&vb->stop_update_lock);
}

{% endhighlight %}

update_balloon_stats_func 这里主要调用stats_handle_request
{%highlight c%}
static void update_balloon_stats_func(struct work_struct *work)
{
    struct virtio_balloon *vb;

    vb = container_of(work, struct virtio_balloon,
              update_balloon_stats_work);
    stats_handle_request(vb);
}
{% endhighlight %}

然后update_balloon_stats这个函数将会收集所有的信息到达vb这个结构体，然后将vb发送给后端。
{%highlight c%}
static void stats_handle_request(struct virtio_balloon *vb)
{
    struct virtqueue *vq;
    struct scatterlist sg;
    unsigned int len;

    update_balloon_stats(vb);

    vq = vb->stats_vq;
    if (!virtqueue_get_buf(vq, &len))
        return;
    sg_init_one(&sg, vb->stats, sizeof(vb->stats));
    virtqueue_add_outbuf(vq, &sg, 1, vb, GFP_KERNEL);
    virtqueue_kick(vq);
}
{% endhighlight %}

update_balloon_stats这个函数就是实际采集guest连内存使用的情况。
{%highlight c%}
static void update_balloon_stats(struct virtio_balloon *vb)
{
    unsigned long events[NR_VM_EVENT_ITEMS];
    struct sysinfo i;
    int idx = 0;
    long available;

    all_vm_events(events);
    si_meminfo(&i);

    available = si_mem_available();

    update_stat(vb, idx++, VIRTIO_BALLOON_S_SWAP_IN,
                pages_to_bytes(events[PSWPIN]));
    update_stat(vb, idx++, VIRTIO_BALLOON_S_SWAP_OUT,
                pages_to_bytes(events[PSWPOUT]));
    update_stat(vb, idx++, VIRTIO_BALLOON_S_MAJFLT, events[PGMAJFAULT]);
    update_stat(vb, idx++, VIRTIO_BALLOON_S_MINFLT, events[PGFAULT]);
    update_stat(vb, idx++, VIRTIO_BALLOON_S_MEMFREE,
                pages_to_bytes(i.freeram));
    update_stat(vb, idx++, VIRTIO_BALLOON_S_MEMTOT,
                pages_to_bytes(i.totalram));
    update_stat(vb, idx++, VIRTIO_BALLOON_S_AVAIL,
                pages_to_bytes(available));
}
{% endhighlight %}

到此，Vritio-Balloon的linux驱动的相关实现都描述完了。
Windows驱动的实现和linux有些区别，这里还是有必要介绍一下。

在收到qemu的请求后，驱动会返回内存的值，这里值得注意的是，
sg.physAddr这里通过MmGetPhysicalAddress地址转换直接得到了内存使用率。

{%highlight c%}
VOID
BalloonMemStats(
    IN WDFOBJECT WdfDevice
    )
{
    VIO_SG              sg;
    PDEVICE_CONTEXT     devCtx = GetDeviceContext(WdfDevice);

    TraceEvents(TRACE_LEVEL_INFORMATION, DBG_HW_ACCESS, "--> %s\n", __FUNCTION__);

    sg.physAddr = MmGetPhysicalAddress(devCtx->MemStats);
    sg.length = sizeof(BALLOON_STAT) * VIRTIO_BALLOON_S_NR;

    WdfSpinLockAcquire(devCtx->StatQueueLock);
    if (virtqueue_add_buf(devCtx->StatVirtQueue, &sg, 1, 0, devCtx, NULL, 0) < 0)
    {
        TraceEvents(TRACE_LEVEL_ERROR, DBG_HW_ACCESS, "<-> %s :: Cannot add buffer\n", __FUNCTION__);
    }
    virtqueue_kick(devCtx->StatVirtQueue);
    WdfSpinLockRelease(devCtx->StatQueueLock);

    TraceEvents(TRACE_LEVEL_INFORMATION, DBG_HW_ACCESS, "<-- %s\n", __FUNCTION__);
}
{% endhighlight %}

这个内存使用率的真正采集者其实是BLNSRV.exe这个进程。
这个进程中启动了定时任务，一直在刷新内存的使用率。

{%highlight c%}
StatWorkItemWorker(
    IN WDFWORKITEM  WorkItem
    )
{
    WDFDEVICE       Device = WdfWorkItemGetParentObject(WorkItem);
    PDEVICE_CONTEXT devCtx = GetDeviceContext(Device);
    NTSTATUS        status = STATUS_SUCCESS;

    do
    {
        TraceEvents(TRACE_LEVEL_INFORMATION, DBG_HW_ACCESS,
            "StatWorkItemWorker Called! \n");
        status = GatherKernelStats(devCtx->MemStats);
        if (NT_SUCCESS(status))
        {
#if 0
            size_t i;
            for (i = 0; i < VIRTIO_BALLOON_S_NR; ++i)
            {
                TraceEvents(TRACE_LEVEL_INFORMATION, DBG_HW_ACCESS,
                    "st=%x tag = %d, value = %08I64X \n\n", status,
                    devCtx->MemStats[i].tag, devCtx->MemStats[i].val);
            }
#endif
        } else {
            RtlFillMemory (devCtx->MemStats, sizeof (BALLOON_STAT) * VIRTIO_BALLOON_S_NR, -1);
        }
        BalloonMemStats(Device);
    } while(InterlockedDecrement(&devCtx->WorkCount));
    return;
}
{% endhighlight %}

GatherKernelStats 实际采集内存指标
{%highlight c%}
NTSTATUS GatherKernelStats(BALLOON_STAT stats[VIRTIO_BALLOON_S_NR])
{
    SYSTEM_BASIC_INFORMATION basicInfo;
    SYSTEM_PERFORMANCE_INFORMATION perfInfo;
    ULONG outLen = 0;
    NTSTATUS ntStatus;
    ULONG idx = 0;
    UINT64 SoftFaults;

    RtlZeroMemory(&basicInfo,sizeof(basicInfo));
    RtlZeroMemory(&perfInfo,sizeof(perfInfo));

    ntStatus = ZwQuerySystemInformation(SystemBasicInformation, &basicInfo, sizeof(basicInfo), &outLen);
    if(!NT_SUCCESS(ntStatus))
    {
        TraceEvents(TRACE_LEVEL_ERROR, DBG_HW_ACCESS,
            "GatherKernelStats (SystemBasicInformation) failed 0x%08x (outLen=0x%x)\n", ntStatus, outLen);
        return ntStatus;
    }

    if ((!bBasicInfoWarning)&&(outLen != sizeof(basicInfo))) {
        bBasicInfoWarning = TRUE;
        TraceEvents(TRACE_LEVEL_WARNING, DBG_HW_ACCESS,
            "GatherKernelStats (SystemBasicInformation) expected outLen=0x%08x returned with 0x%0x",
            sizeof(basicInfo), outLen);
    }

    ntStatus = ZwQuerySystemInformation(SystemPerformanceInformation, &perfInfo, sizeof(perfInfo), &outLen);
    if(!NT_SUCCESS(ntStatus))
    {
        TraceEvents(TRACE_LEVEL_ERROR, DBG_HW_ACCESS,
            "GatherKernelStats (SystemPerformanceInformation) failed 0x%08x (outLen=0x%x)\n", ntStatus, outLen);
        return ntStatus;
    }

    if ((!bPerfInfoWarning)&&(outLen != sizeof(perfInfo))) {
        bPerfInfoWarning = TRUE;
        TraceEvents(TRACE_LEVEL_WARNING, DBG_HW_ACCESS,
            "GatherKernelStats (SystemPerformanceInformation) expected outLen=0x%08x returned with 0x%0x",
            sizeof(perfInfo), outLen);
    }

    #define UpdateNoOverflow(x) UpdateOverflowFreeCounter(&Counters[_##x],perfInfo.##x)
    UpdateStat(&stats[idx++], VIRTIO_BALLOON_S_SWAP_IN,  UpdateNoOverflow(PageReadCount) << PAGE_SHIFT);
    UpdateStat(&stats[idx++], VIRTIO_BALLOON_S_SWAP_OUT,
        (UpdateNoOverflow(DirtyPagesWriteCount) + UpdateNoOverflow(MappedPagesWriteCount)) << PAGE_SHIFT);
    SoftFaults = UpdateNoOverflow(CopyOnWriteCount) + UpdateNoOverflow(TransitionCount) +
                 UpdateNoOverflow(CacheTransitionCount) + UpdateNoOverflow(DemandZeroCount);
    UpdateStat(&stats[idx++], VIRTIO_BALLOON_S_MAJFLT,   UpdateNoOverflow(PageReadCount));
    UpdateStat(&stats[idx++], VIRTIO_BALLOON_S_MINFLT,   SoftFaults);
    UpdateStat(&stats[idx++], VIRTIO_BALLOON_S_MEMFREE,  U32_2_S64(perfInfo.AvailablePages) << PAGE_SHIFT);
    UpdateStat(&stats[idx++], VIRTIO_BALLOON_S_MEMTOT,   U32_2_S64(basicInfo.NumberOfPhysicalPages) << PAGE_SHIFT);
    #undef UpdateNoOverflow

    return ntStatus;
}
{% endhighlight %}

到此windows的驱动也讲完了。

qemu后端代码比较简单了，首先来看气球初始化时候，
通过qemu_add_balloon_handler，注册了在调用qmp时候的回调函数，用来发起请求。
初始化了队列，ivq，dvq，svq用来接收相关的信息，然后调用回调进行相应操作。
{%highlight c%}
static void virtio_balloon_device_realize(DeviceState *dev, Error **errp)
{
    VirtIODevice *vdev = VIRTIO_DEVICE(dev);
    VirtIOBalloon *s = VIRTIO_BALLOON(dev);
    int ret;

    virtio_init(vdev, "virtio-balloon", VIRTIO_ID_BALLOON,
                sizeof(struct virtio_balloon_config));

    ret = qemu_add_balloon_handler(virtio_balloon_to_target,
                                   virtio_balloon_stat, s);

    if (ret < 0) {
        error_setg(errp, "Adding balloon handler failed");
        virtio_cleanup(vdev);
        return;
    }

    s->ivq = virtio_add_queue(vdev, 128, virtio_balloon_handle_output);
    s->dvq = virtio_add_queue(vdev, 128, virtio_balloon_handle_output);
    s->svq = virtio_add_queue(vdev, 128, virtio_balloon_receive_stats);
    reset_stats(s);

    register_savevm(dev, "virtio-balloon", -1, 1,
                    virtio_balloon_save, virtio_balloon_load, s);

    object_property_add(OBJECT(dev), "guest-stats", "guest statistics",
                        balloon_stats_get_all, NULL, NULL, s, NULL);

    object_property_add(OBJECT(dev), "guest-stats-polling-interval", "int",
                        balloon_stats_get_poll_interval,
                        balloon_stats_set_poll_interval,
                        NULL, s, NULL);
}

{% endhighlight %}

这里以刷新内存信息为例往下讲，内存增减功能类似。
在上面注册了guest-stats-polling-interval属性，这个是设置查询周期的，在设置了周期后，就可以看到启动了一个定时任务，balloon_stats_poll_cb来实时查询内存的信息。
{%highlight c%}
static void balloon_stats_set_poll_interval(Object *obj, struct Visitor *v,
                                            void *opaque, const char *name,
                                            Error **errp)
{
    VirtIOBalloon *s = opaque;
    Error *local_err = NULL;
    int64_t value;

    visit_type_int(v, &value, name, &local_err);
    if (local_err) {
        error_propagate(errp, local_err);
        return;
    }

    if (value < 0) {
        error_setg(errp, "timer value must be greater than zero");
        return;
    }

    if (value > UINT32_MAX) {
        error_setg(errp, "timer value is too big");
        return;
    }

    if (value == s->stats_poll_interval) {
        return;
    }

    if (value == 0) {
        /* timer=0 disables the timer */
        balloon_stats_destroy_timer(s);
        return;
    }

    if (balloon_stats_enabled(s)) {
        /* timer interval change */
        s->stats_poll_interval = value;
        balloon_stats_change_timer(s, value);
        return;
    }

    /* create a new timer */
    g_assert(s->stats_timer == NULL);
    s->stats_timer = timer_new_ms(QEMU_CLOCK_VIRTUAL, balloon_stats_poll_cb, s);
    s->stats_poll_interval = value;
    balloon_stats_change_timer(s, 0);
}
{% endhighlight %}

而virtio_balloon_receive_stats主要是从vq中取到前面讲的内存信息，然后用balloon_stats_enabled进行更新。

{%highlight c%}
static void virtio_balloon_receive_stats(VirtIODevice *vdev, VirtQueue *vq)
{
    VirtIOBalloon *s = VIRTIO_BALLOON(vdev);
    VirtQueueElement *elem = &s->stats_vq_elem;
    VirtIOBalloonStat stat;
    size_t offset = 0;
    qemu_timeval tv;

    if (!virtqueue_pop(vq, elem)) {
        goto out;
    }

    /* Initialize the stats to get rid of any stale values.  This is only
     * needed to handle the case where a guest supports fewer stats than it
     * used to (ie. it has booted into an old kernel).
     */
    reset_stats(s);

    while (iov_to_buf(elem->out_sg, elem->out_num, offset, &stat, sizeof(stat))
           == sizeof(stat)) {
        uint16_t tag = virtio_tswap16(vdev, stat.tag);
        uint64_t val = virtio_tswap64(vdev, stat.val);

        offset += sizeof(stat);
        if (tag < VIRTIO_BALLOON_S_NR)
            s->stats[tag] = val;
    }
    s->stats_vq_offset = offset;

    if (qemu_gettimeofday(&tv) < 0) {
        fprintf(stderr, "warning: %s: failed to get time of day\n", __func__);
        goto out;
    }

    s->stats_last_update = tv.tv_sec;

out:
    if (balloon_stats_enabled(s)) {
        balloon_stats_change_timer(s, s->stats_poll_interval);
    }
}

{% endhighlight %}

总结
---
到此可以看到Virtio-Balloon的相关实现，和周期性查询内存信息的功能。
Virtio-Balloon进行内存复用本身存在一些问题
* Guest对内存变化会进行感知，Balloon特性本身是kernel所具备的，本身是通过修改识别的内存信息来限制Guest中的使用。所以不是很友好。
* 内存复用需要实时监控，发现客户虚拟机内存使用过多还要及时归还内存，对于系统本身做这个功能有很大的局限性。
* Balloon特性的内存复用并不是本质上进行内存的冗余复用，仅仅是借东墙补西墙，当虚拟机都大量使用内存时候，并不能实际突破物理内存上限。



