---

layout: post

title:  "xen热迁移代码分析"

date:   2015-09-18 11:10:01

categories: 虚拟化

tags: 
  - 虚拟化
  - xen
---

xen的热迁移代码使用的是precopy算法，简单描述下算法


使用了3个位图来记录需要迁移的page

to_send：上一次迭代的脏页情况，

to_skip：当前脏页情况。

to_fix：最后停机copy的页

Xen 利用循环迭代的方式将内存页拷贝到目的主机上，在第一轮迭代中，所有的页都被
拷贝，以后每轮迭代只传送上一轮传送过程中被修改过的内存页。

在每轮迭代结束前，调用
影子页表的清空操作(XEN_DOMCTL_SHADOW_OP_CLEAN)一将脏页位图拷贝到 to_send
中，该操作同时清空脏页位图。

在下一轮迭代开始后调用影子页表查看操作
(XEN_DOMCTL_SHADOW_OP_PEEK)将脏页位图拷到 to_skip。并根据 to_send 和 to_skip
判定该页是否作为传送对象。

好了，老规矩直接上代码。
xen的热迁移和save虚拟机调用的是同一个接口，这里和kvm的思路是一样的。save到文件和发送到对端
都是将一段流发送出去。

在tools/libxc/xc_domain_save.c中xc_domain_save函数实现了热迁移的全过程。
由于函数比较大，截关键的地方来说明。

很明显，下面的代码主要调用了xc_shadow_control函数，向xen开启了logdirty形式。
这个xc_shadow_control函数中的具体作用，后面还会详解。也可以看到如果不是live形式，
会直接suspend虚拟机，这样就不会形成脏页了。


{%highlight c%}
//xc_domain_save函数片段

 if ( live )
    {
        /* Live suspend. Enable log-dirty mode. */
        if ( xc_shadow_control(xch, dom,
                               XEN_DOMCTL_SHADOW_OP_ENABLE_LOGDIRTY,
                               NULL, 0, NULL, 0, NULL) < 0 )
        {
            /* log-dirty already enabled? There's no test op,
               so attempt to disable then reenable it */
            frc = xc_shadow_control(xch, dom, XEN_DOMCTL_SHADOW_OP_OFF,
                                    NULL, 0, NULL, 0, NULL);
            if ( frc >= 0 )
            {
                frc = xc_shadow_control(xch, dom,
                                        XEN_DOMCTL_SHADOW_OP_ENABLE_LOGDIRTY,
                                        NULL, 0, NULL, 0, NULL);
            }

            if ( frc < 0 )
            {
                PERROR("Couldn't enable shadow mode (rc %d) (errno %d)", frc, errno );
                goto out;
            }
        }

        /* Enable qemu-dm logging dirty pages to xen */
        if ( hvm && callbacks->switch_qemu_logdirty(dom, 1, callbacks->data) )
        {
            PERROR("Couldn't enable qemu log-dirty mode (errno %d)", errno);
            goto out;
        }
    }
    else
    {
        /* This is a non-live suspend. Suspend the domain .*/
        if ( suspend_and_state(callbacks->suspend, callbacks->data, xch,
                               io_fd, dom, &info) )
        {
            ERROR("Domain appears not to have suspended");
            goto out;
        }
    }

{%endhighlight%}

然后就是要建立迁移需要的bitmap
{%highlight c%}

    /* Setup to_send / to_fix and to_skip bitmaps */
    to_send = xc_hypercall_buffer_alloc_pages(xch, to_send, NRPAGES(BITMAP_SIZE));
    to_skip = xc_hypercall_buffer_alloc_pages(xch, to_skip, NRPAGES(BITMAP_SIZE));
    to_fix  = calloc(1, BITMAP_SIZE);
{%endhighlight%}

接下来会开始进行迭代copy状态
{%highlight c%}
/* Now write out each data page, canonicalising page tables as we go... */
    for ( ; ; )
    {
        unsigned int N, batch, run;
        char reportbuf[80];

        snprintf(reportbuf, sizeof(reportbuf),
                 "Saving memory: iter %d (last sent %u skipped %u)",
                 iter, sent_this_iter, skip_this_iter);

        xc_report_progress_start(xch, reportbuf, dinfo->p2m_size);

        iter++;
        sent_this_iter = 0;
        skip_this_iter = 0;
        N = 0;

        //每一次大迭代，都会从p2m表的0号帧开始，
        while ( N < dinfo->p2m_size )
        {
            //一次迭代开始
            xc_report_progress_step(xch, N, dinfo->p2m_size);

            if ( !last_iter )
            {
                /* Slightly wasteful to peek the whole array every time,
                   but this is fast enough for the moment. */
                //获取当前脏页情况
                frc = xc_shadow_control(
                    xch, dom, XEN_DOMCTL_SHADOW_OP_PEEK, HYPERCALL_BUFFER(to_skip),
                    dinfo->p2m_size, NULL, 0, NULL);
                if ( frc != dinfo->p2m_size )
                {
                    ERROR("Error peeking shadow bitmap");
                    goto out;
                }
            }

            //中间经过了一个大循环主要是建立了一个batch，决定一次传送的页的数量。
            //然后还是用xc_map_foreign_bulk将需要迁移的页记录下来。
            region_base = xc_map_foreign_bulk(
                xch, dom, PROT_READ, pfn_type, pfn_err, batch);
            if ( region_base == NULL )
            {
                PERROR("map batch failed");
                goto out;
            }
 
            //然后对页进行传输。
         }

         //一次迭代后会进行判断是否是最后一次迭代，
     skip:

        xc_report_progress_step(xch, dinfo->p2m_size, dinfo->p2m_size);

        total_sent += sent_this_iter;

        if ( last_iter )
        {
            print_stats( xch, dom, sent_this_iter, &stats, 1);

            DPRINTF("Total pages sent= %ld (%.2fx)\n",
                    total_sent, ((float)total_sent)/dinfo->p2m_size );
            DPRINTF("(of which %ld were fixups)\n", needed_to_fix  );
        }

        if ( last_iter && debug )
        {
            int id = XC_SAVE_ID_ENABLE_VERIFY_MODE;
            memset(to_send, 0xff, BITMAP_SIZE);
            debug = 0;
            DPRINTF("Entering debug resend-all mode\n");

            /* send "-1" to put receiver into debug mode */
            if ( wrexact(io_fd, &id, sizeof(int)) )
            {
                PERROR("Error when writing to state file (6)");
                goto out;
            }

            continue;
        }

        if ( last_iter )
            //最终跳出大循环
            break;

        if ( live )
        {
            if ( ((sent_this_iter > sent_last_iter) && RATE_IS_MAX()) ||
                 (iter >= max_iters) ||
                 (sent_this_iter+skip_this_iter < 50) ||
                 (total_sent > dinfo->p2m_size*max_factor) )
            {
                DPRINTF("Start last iteration\n");
                last_iter = 1;

                if ( suspend_and_state(callbacks->suspend, callbacks->data,
                                       xch, io_fd, dom, &info) )
                {
                    ERROR("Domain appears not to have suspended");
                    goto out;
                }

                DPRINTF("SUSPEND shinfo %08lx\n", info.shared_info_frame);
                if ( (tmem_saved > 0) &&
                     (xc_tmem_save_extra(xch,dom,io_fd,XC_SAVE_ID_TMEM_EXTRA) == -1) )
                {
                        PERROR("Error when writing to state file (tmem)");
                        goto out;
                }

                if ( save_tsc_info(xch, dom, io_fd) < 0 )
                {
                    PERROR("Error when writing to state file (tsc)");
                    goto out;
                }


            }
            //如果是热迁移，则还是要更新最新的脏页情况。
            if ( xc_shadow_control(xch, dom,
                                   XEN_DOMCTL_SHADOW_OP_CLEAN, HYPERCALL_BUFFER(to_send),
                                   dinfo->p2m_size, NULL, 0, &stats) != dinfo->p2m_size )
            {
                PERROR("Error flushing shadow PT");
                goto out;
            }

            sent_last_iter = sent_this_iter;

            print_stats(xch, dom, sent_this_iter, &stats, 1);

        }
    } 

{%endhighlight%}

上面讲了迁移的基本算法，那么一些细节就是接下来描述的。
比如是谁来记录脏页的？还有脏页记录到哪里了？

这就不得不提到EPT，EPT是intel在虚拟化下提供内存访问加速的特性，在热迁移时，需要让EPT来帮忙记录脏页。

上面提到了xc_shadow_control函数，这个函数很简单，其实就是将传递的参数进行封装，然后发给xen来进行接下来

的处理。

{%highlight c%}
int xc_shadow_control(xc_interface *xch,
                      uint32_t domid,
                      unsigned int sop,
                      xc_hypercall_buffer_t *dirty_bitmap,
                      unsigned long pages,
                      unsigned long *mb,
                      uint32_t mode,
                      xc_shadow_op_stats_t *stats)
{
    int rc;
    DECLARE_DOMCTL;
    DECLARE_HYPERCALL_BUFFER_ARGUMENT(dirty_bitmap);

    domctl.cmd = XEN_DOMCTL_shadow_op;
    domctl.domain = (domid_t)domid;
    domctl.u.shadow_op.op     = sop;
    domctl.u.shadow_op.pages  = pages;
    domctl.u.shadow_op.mb     = mb ? *mb : 0;
    domctl.u.shadow_op.mode   = mode;
    if (dirty_bitmap != NULL)
        set_xen_guest_handle(domctl.u.shadow_op.dirty_bitmap,
                                dirty_bitmap);

    rc = do_domctl(xch, &domctl);

    if ( stats )
        memcpy(stats, &domctl.u.shadow_op.stats,
               sizeof(xc_shadow_op_stats_t));

    if ( mb )
        *mb = domctl.u.shadow_op.mb;

    return (rc == 0) ? domctl.u.shadow_op.pages : rc;
}
{%endhighlight%}

xen中的arch_do_domctl实现了对这一调用的后半段处理，可以看到迁移使用的系统调用其实是
在paging_domctl函数中

{%highlight c%}
long arch_do_domctl(
    struct xen_domctl *domctl,
    XEN_GUEST_HANDLE(xen_domctl_t) u_domctl)
{
    long ret = 0;

    switch ( domctl->cmd )
    {

    case XEN_DOMCTL_shadow_op:
    {
        struct domain *d;
        ret = -ESRCH;
        d = rcu_lock_domain_by_id(domctl->domain);
        if ( d != NULL )
        {
            ret = paging_domctl(d,
                                &domctl->u.shadow_op,
                                guest_handle_cast(u_domctl, void));
            rcu_unlock_domain(d);
            copy_to_guest(u_domctl, domctl, 1);
        }
    }
    break;

{%endhighlight%}

paging_domctl函数中就可以看到刚才在迁移中开启logdirty状态，获取dirty页的相关操作。
xen下面对页的虚拟化采用了两种，一种是影子页表形式，主要在xen/arch/x86/mm/shadow目录下，另一种是EPT形式

，在xen/arch/x86/mm/hap目录下。关于如何开启logdirty状态，等细节请自己翻看代码。这里简述下过程。
p2m表中有标记位为logdirty状态，这里开启时候，就遍历所有的p2m表项，然后设置标记位。在虚拟机运行中，如果

遇到标记了logdirty状态的页就会vmexit到xen中，xen在vmx_vmexit_handler中对异常进行处理,调用

ept_handle_violation函数来处理，经过hvm_hap_nested_page_fault调用paging_mark_dirty函数来对页面进行标记

。


{%highlight c%}
int paging_domctl(struct domain *d, xen_domctl_shadow_op_t *sc,
                  XEN_GUEST_HANDLE(void) u_domctl)
{
    int rc;

    if ( unlikely(d == current->domain) )
    {
        gdprintk(XENLOG_INFO, "Tried to do a paging op on itself.\n");
        return -EINVAL;
    }

    if ( unlikely(d->is_dying) )
    {
        gdprintk(XENLOG_INFO, "Ignoring paging op on dying domain %u\n",
                 d->domain_id);
        return 0;
    }

    if ( unlikely(d->vcpu == NULL) || unlikely(d->vcpu[0] == NULL) )
    {
        gdprintk(XENLOG_DEBUG, "Paging op on a domain (%u) with no vcpus\n",
                 d->domain_id);
        return -EINVAL;
    }

    rc = xsm_shadow_control(d, sc->op);
    if ( rc )
        return rc;

    /* Code to handle log-dirty. Note that some log dirty operations
     * piggy-back on shadow operations. For example, when
     * XEN_DOMCTL_SHADOW_OP_OFF is called, it first checks whether log dirty
     * mode is enabled. If does, we disables log dirty and continues with
     * shadow code. For this reason, we need to further dispatch domctl
     * to next-level paging code (shadow or hap).
     */
    switch ( sc->op )
    {

    case XEN_DOMCTL_SHADOW_OP_ENABLE:
        if ( !(sc->mode & XEN_DOMCTL_SHADOW_ENABLE_LOG_DIRTY) )
            break;
        /* Else fall through... */
    case XEN_DOMCTL_SHADOW_OP_ENABLE_LOGDIRTY:
        if ( hap_enabled(d) )
            hap_logdirty_init(d);
        return paging_log_dirty_enable(d);

    case XEN_DOMCTL_SHADOW_OP_OFF:
        if ( paging_mode_log_dirty(d) )
            if ( (rc = paging_log_dirty_disable(d)) != 0 )
                return rc;
        break;

    case XEN_DOMCTL_SHADOW_OP_CLEAN:
    case XEN_DOMCTL_SHADOW_OP_PEEK:
        return paging_log_dirty_op(d, sc);
    }

    /* Here, dispatch domctl to the appropriate paging code */
    if ( hap_enabled(d) )
        return hap_domctl(d, sc, u_domctl);
    else
        return shadow_domctl(d, sc, u_domctl);
}
{%endhighlight%}

paging_mark_dirty函数就是在页被标记位logdirty时候对页进行的处理。
{%highlight c%}
/* Mark a page as dirty */
void paging_mark_dirty(struct domain *d, unsigned long guest_mfn)
{
    unsigned long pfn;
    mfn_t gmfn, new_mfn;
    int changed;
    mfn_t mfn, *l4, *l3, *l2;
    unsigned long *l1;
    int i1, i2, i3, i4;

    gmfn = _mfn(guest_mfn);

    if ( !paging_mode_log_dirty(d) || !mfn_valid(gmfn) ||
         page_get_owner(mfn_to_page(gmfn)) != d )
        return;

    /* We /really/ mean PFN here, even for non-translated guests. */
    pfn = get_gpfn_from_mfn(mfn_x(gmfn));
    /* Shared MFNs should NEVER be marked dirty */
    BUG_ON(SHARED_M2P(pfn));

    /*
     * Values with the MSB set denote MFNs that aren't really part of the
     * domain's pseudo-physical memory map (e.g., the shared info frame).
     * Nothing to do here...
     */
    if ( unlikely(!VALID_M2P(pfn)) )
        return;

    i1 = L1_LOGDIRTY_IDX(pfn);
    i2 = L2_LOGDIRTY_IDX(pfn);
    i3 = L3_LOGDIRTY_IDX(pfn);
    i4 = L4_LOGDIRTY_IDX(pfn);

    /* We can't call paging.alloc_page() with the log-dirty lock held
     * and we almost never need to call it anyway, so assume that we
     * won't.  If we do hit a missing page, we'll unlock, allocate one
     * and start again. */
    new_mfn = _mfn(INVALID_MFN);

again:
    log_dirty_lock(d);

    //获取dirty_bitmap的根节点，
    l4 = paging_map_log_dirty_bitmap(d);
    if ( unlikely(!l4) )
    {
        //如果不存在l4的页表，就要建立
        l4 = paging_new_log_dirty_node(new_mfn);
        d->arch.paging.log_dirty.top = new_mfn;
        new_mfn = _mfn(INVALID_MFN);
    }
    if ( unlikely(!l4) )
        goto oom;

    mfn = l4[i4];
    if ( !mfn_valid(mfn) )
    {
        l3 = paging_new_log_dirty_node(new_mfn);
        mfn = l4[i4] = new_mfn;
        new_mfn = _mfn(INVALID_MFN);
    }
    else
        l3 = map_domain_page(mfn_x(mfn));
    unmap_domain_page(l4);
    if ( unlikely(!l3) )
        goto oom;

    mfn = l3[i3];
    if ( !mfn_valid(mfn) )
    {
        l2 = paging_new_log_dirty_node(new_mfn);
        mfn = l3[i3] = new_mfn;
        new_mfn = _mfn(INVALID_MFN);
    }
    else
        l2 = map_domain_page(mfn_x(mfn));
    unmap_domain_page(l3);
    if ( unlikely(!l2) )
        goto oom;

    mfn = l2[i2];
    if ( !mfn_valid(mfn) )
    {
        l1 = paging_new_log_dirty_leaf(new_mfn);
        mfn = l2[i2] = new_mfn;
        new_mfn = _mfn(INVALID_MFN);
    }
    else
        l1 = map_domain_page(mfn_x(mfn));
    unmap_domain_page(l2);
    if ( unlikely(!l1) )
        goto oom;
    //找到了要标记的页表，对改页表进行标记，为dirty页
    changed = !__test_and_set_bit(i1, l1);
    unmap_domain_page(l1);
    if ( changed )
    {
        PAGING_DEBUG(LOGDIRTY,
                     "marked mfn %" PRI_mfn " (pfn=%lx), dom %d\n",
                     mfn_x(gmfn), pfn, d->domain_id);
        d->arch.paging.log_dirty.dirty_count++;
    }

    log_dirty_unlock(d);
    if ( mfn_valid(new_mfn) )
        paging_free_log_dirty_page(d, new_mfn);
    return;

oom:
    log_dirty_unlock(d);
    new_mfn = paging_new_log_dirty_page(d);
    if ( !mfn_valid(new_mfn) )
        /* we've already recorded the failed allocation */
        return;
    goto again;
}
{%endhighlight%}

接下来看看，在接收到XEN_DOMCTL_SHADOW_OP_PEEK或者XEN_DOMCTL_SHADOW_OP_CLEAN后，如何拿到bitmap
{%highlight c%}
int paging_log_dirty_op(struct domain *d, struct xen_domctl_shadow_op *sc)
{
    int rv = 0, clean = 0, peek = 1;
    unsigned long pages = 0;
    mfn_t *l4, *l3, *l2;
    unsigned long *l1;
    int i4, i3, i2;

    domain_pause(d);
    log_dirty_lock(d);

    clean = (sc->op == XEN_DOMCTL_SHADOW_OP_CLEAN);

    PAGING_DEBUG(LOGDIRTY, "log-dirty %s: dom %u faults=%u dirty=%u\n",
                 (clean) ? "clean" : "peek",
                 d->domain_id,
                 d->arch.paging.log_dirty.fault_count,
                 d->arch.paging.log_dirty.dirty_count);

    sc->stats.fault_count = d->arch.paging.log_dirty.fault_count;
    sc->stats.dirty_count = d->arch.paging.log_dirty.dirty_count;

    if ( clean )
    {
        d->arch.paging.log_dirty.fault_count = 0;
        d->arch.paging.log_dirty.dirty_count = 0;
    }

    if ( guest_handle_is_null(sc->dirty_bitmap) )
        /* caller may have wanted just to clean the state or access stats. */
        peek = 0;

    if ( unlikely(d->arch.paging.log_dirty.failed_allocs) ) {
        printk("%s: %d failed page allocs while logging dirty pages\n",
               __FUNCTION__, d->arch.paging.log_dirty.failed_allocs);
        rv = -ENOMEM;
        goto out;
    }

    pages = 0;
    //获取dirty_bitmap的根节点，然后会遍历整个bitmap树
    l4 = paging_map_log_dirty_bitmap(d);

    for ( i4 = 0;
          (pages < sc->pages) && (i4 < LOGDIRTY_NODE_ENTRIES);
          i4++ )
    {
        l3 = (l4 && mfn_valid(l4[i4])) ? map_domain_page(mfn_x(l4[i4])) : NULL;
        for ( i3 = 0;
              (pages < sc->pages) && (i3 < LOGDIRTY_NODE_ENTRIES);
              i3++ )
        {
            l2 = ((l3 && mfn_valid(l3[i3])) ?
                  map_domain_page(mfn_x(l3[i3])) : NULL);
            for ( i2 = 0;
                  (pages < sc->pages) && (i2 < LOGDIRTY_NODE_ENTRIES);
                  i2++ )
            {
                static unsigned long zeroes[PAGE_SIZE/BYTES_PER_LONG];
                unsigned int bytes = PAGE_SIZE;
                l1 = ((l2 && mfn_valid(l2[i2])) ?
                      map_domain_page(mfn_x(l2[i2])) : zeroes);
                if ( unlikely(((sc->pages - pages + 7) >> 3) < bytes) )
                    bytes = (unsigned int)((sc->pages - pages + 7) >> 3);
                if ( likely(peek) )
                {
                    //将标记位dirty的页放到用户的dirty_bitmap中， 
                    if ( copy_to_guest_offset(sc->dirty_bitmap, pages >> 3,
                                              (uint8_t *)l1, bytes) != 0 )
                    {
                        rv = -EFAULT;
                        goto out;
                    }
                }
                if ( clean && l1 != zeroes )
                    //clean时候，直接清除页表
                    clear_page(l1);
                pages += bytes << 3;
                if ( l1 != zeroes )
                    unmap_domain_page(l1);
            }
            if ( l2 )
                unmap_domain_page(l2);
        }
        if ( l3 )
            unmap_domain_page(l3);
    }
    if ( l4 )
        unmap_domain_page(l4);

    if ( pages < sc->pages )
        sc->pages = pages;

    log_dirty_unlock(d);

    if ( clean )
    {
        /* We need to further call clean_dirty_bitmap() functions of specific
         * paging modes (shadow or hap).  Safe because the domain is paused. */
        //将dirty标记重新再设置一次
        d->arch.paging.log_dirty.clean_dirty_bitmap(d);
    }
    domain_unpause(d);
    return rv;

 out:
    log_dirty_unlock(d);
    domain_unpause(d);
    return rv;
}

{%endhighlight%}


到此，xen的迁移介绍完了。

