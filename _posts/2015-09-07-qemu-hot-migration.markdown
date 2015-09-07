---

layout: post

title:  "走读qemu代码热迁移流程"

date:   2015-09-07 11:10:01

categories: 虚拟化
tags : 
  - 虚拟化
  - qemu
---

热迁移的概念已经不陌生了，在虚拟化发展中，热迁移也越来越多的应用在商用场景。今天就来分析下热迁移的代码，详细了解下这一过程如何实现的。

首先，从官网上翻译下关于[热迁移的算法]描述。

vmA：要迁移的主机

vmB：接受迁移的主机

1.*准备阶段*

启动vmB，vmA连接到vmB，开启内存脏页日志和其他

2.*转移内存*

vmA持续运行，第一次传送所有内存到vmB，然后递归传输脏页到vmB。(每次传输需要耗费一定时间，该时间会产生新的脏页)

3.*停止虚拟机*

vmA suspend,执行sync，同步vmA磁盘信息。

4.*传输状态*

所有vmA的设备状态和脏页被传输。

5.*继续运行*

vmB开始运行，虚拟机迁移成功。


下面开始从代码看看这一流程的实现

位于migration.c的函数qmp_migrate就是迁移的入口函数


{%highlight c%}
void qmp_migrate(const char *uri, bool has_blk, bool blk,
                 bool has_inc, bool inc, bool has_detach, bool detach,
                 Error **errp)
{
    Error *local_err = NULL;
    MigrationState *s = migrate_get_current();
    MigrationParams params;
    const char *p;

    params.blk = has_blk && blk;
    params.shared = has_inc && inc;

    if (s->state == MIGRATION_STATUS_ACTIVE ||
        s->state == MIGRATION_STATUS_SETUP ||
        s->state == MIGRATION_STATUS_CANCELLING) {
        error_set(errp, QERR_MIGRATION_ACTIVE);
        return;
    }

    if (runstate_check(RUN_STATE_INMIGRATE)) {
        error_setg(errp, "Guest is waiting for an incoming migration");
        return;
    }

    //检查在否存在不能迁移的设备
    if (qemu_savevm_state_blocked(errp)) {
        return;
    }

    if (migration_blockers) {
        *errp = error_copy(migration_blockers->data);
        return;
    }

    s = migrate_init(&params);

    //选择指定的协议建立连接，并建立migrate线程。
    if (strstart(uri, "tcp:", &p)) {
        tcp_start_outgoing_migration(s, p, &local_err);
#ifdef CONFIG_RDMA
    } else if (strstart(uri, "rdma:", &p)) {
        rdma_start_outgoing_migration(s, p, &local_err);
#endif
#if !defined(WIN32)
    } else if (strstart(uri, "exec:", &p)) {
        exec_start_outgoing_migration(s, p, &local_err);
    } else if (strstart(uri, "unix:", &p)) {
        unix_start_outgoing_migration(s, p, &local_err);
    } else if (strstart(uri, "fd:", &p)) {
        fd_start_outgoing_migration(s, p, &local_err);
#endif
    } else {
        error_set(errp, QERR_INVALID_PARAMETER_VALUE, "uri", "a valid migration protocol");
        s->state = MIGRATION_STATUS_FAILED;
        return;
    }

    if (local_err) {
        migrate_fd_error(s);
        error_propagate(errp, local_err);
        return;
    }
}
{%endhighlight%}

无论哪个协议，最终都会调用到migrate_fd_connect函数。该函数主要创建了迁移线程。

{%highlight c%}
void migrate_fd_connect(MigrationState *s)
{
    s->state = MIGRATION_STATUS_SETUP;
    trace_migrate_set_state(MIGRATION_STATUS_SETUP);

    /* This is a best 1st approximation. ns to ms */
    s->expected_downtime = max_downtime/1000000;
    s->cleanup_bh = qemu_bh_new(migrate_fd_cleanup, s);

    qemu_file_set_rate_limit(s->file,
                             s->bandwidth_limit / XFER_LIMIT_RATIO);

    /* Notify before starting migration thread */
    notifier_list_notify(&migration_state_notifiers, s);

    qemu_thread_create(&s->thread, "migration", migration_thread, s,
                       QEMU_THREAD_JOINABLE);
}
{%endhighlight%}

ram设备在vm初始化时注册的各种handle，这个用处下面会详细说。

{%highlight c%}
static SaveVMHandlers savevm_ram_handlers = {
    .save_live_setup = ram_save_setup,
    .save_live_iterate = ram_save_iterate,
    .save_live_complete = ram_save_complete,
    .save_live_pending = ram_save_pending,
    .load_state = ram_load,
    .cancel = ram_migration_cancel,
};
{%endhighlight%}

线程主要分为3个部分，准备阶段，循环执行迁移，最后处理。

{%highlight c%}
static void *migration_thread(void *opaque)
{
    MigrationState *s = opaque;
    int64_t initial_time = qemu_clock_get_ms(QEMU_CLOCK_REALTIME);
    int64_t setup_start = qemu_clock_get_ms(QEMU_CLOCK_HOST);
    int64_t initial_bytes = 0;
    int64_t max_size = 0;
    int64_t start_time = initial_time;
    bool old_vm_running = false;

    //第一阶段 分别调用了savevm_handlers中注册的迁移设备准备接口，以ram_save_setup为例
    qemu_savevm_state_begin(s->file, &s->params);

    s->setup_time = qemu_clock_get_ms(QEMU_CLOCK_HOST) - setup_start;
    //设置迁移状态为MIGRATION_STATUS_ACTIVE
    migrate_set_state(s, MIGRATION_STATUS_SETUP, MIGRATION_STATUS_ACTIVE);

    while (s->state == MIGRATION_STATUS_ACTIVE) {
        int64_t current_time;
        uint64_t pending_size;

        if (!qemu_file_rate_limit(s->file)) {
            //第二阶段 循环调用savevm_handlers中的各项，以ram_save_pending为例
            pending_size = qemu_savevm_state_pending(s->file, max_size);
            trace_migrate_pending(pending_size, max_size);
            //进行临界条件判断是继续，还是准备结束。
            if (pending_size && pending_size >= max_size) {
                //继续进行迭代迁移，以ram_save_iterate为例
                qemu_savevm_state_iterate(s->file);
            } else {
                int ret;
                //开始第三阶段
                qemu_mutex_lock_iothread();
                start_time = qemu_clock_get_ms(QEMU_CLOCK_REALTIME);
                qemu_system_wakeup_request(QEMU_WAKEUP_REASON_OTHER);
                old_vm_running = runstate_is_running();
                
                //设置虚拟机为迁移完成态
                ret = vm_stop_force_state(RUN_STATE_FINISH_MIGRATE);
                if (ret >= 0) {
                    qemu_file_set_rate_limit(s->file, INT64_MAX);
                //进行调用最后处理
                    qemu_savevm_state_complete(s->file);
                }
                qemu_mutex_unlock_iothread();

                if (ret < 0) {
                    migrate_set_state(s, MIGRATION_STATUS_ACTIVE,
                                      MIGRATION_STATUS_FAILED);
                    break;
                }

                if (!qemu_file_get_error(s->file)) {
                    migrate_set_state(s, MIGRATION_STATUS_ACTIVE,
                                      MIGRATION_STATUS_COMPLETED);
                    break;
                }
            }
        }

        if (qemu_file_get_error(s->file)) {
            migrate_set_state(s, MIGRATION_STATUS_ACTIVE,
                              MIGRATION_STATUS_FAILED);
            break;
        }
        
        current_time = qemu_clock_get_ms(QEMU_CLOCK_REALTIME);
        if (current_time >= initial_time + BUFFER_DELAY) {
            uint64_t transferred_bytes = qemu_ftell(s->file) - initial_bytes;
            uint64_t time_spent = current_time - initial_time;
            double bandwidth = transferred_bytes / time_spent;
            max_size = bandwidth * migrate_max_downtime() / 1000000;

            s->mbps = time_spent ? (((double) transferred_bytes * 8.0) /
                    ((double) time_spent / 1000.0)) / 1000.0 / 1000.0 : -1;

            trace_migrate_transferred(transferred_bytes, time_spent,
                                      bandwidth, max_size);
            /* if we haven't sent anything, we don't want to recalculate
               10000 is a small enough number for our purposes */
            if (s->dirty_bytes_rate && transferred_bytes > 10000) {
                s->expected_downtime = s->dirty_bytes_rate / bandwidth;
            }

            qemu_file_reset_rate_limit(s->file);
            initial_time = current_time;
            initial_bytes = qemu_ftell(s->file);
        }
        if (qemu_file_rate_limit(s->file)) {
            /* usleep expects microseconds */
            g_usleep((initial_time + BUFFER_DELAY - current_time)*1000);
        }
    }

    qemu_mutex_lock_iothread();
    if (s->state == MIGRATION_STATUS_COMPLETED) {
        int64_t end_time = qemu_clock_get_ms(QEMU_CLOCK_REALTIME);
        uint64_t transferred_bytes = qemu_ftell(s->file);
        s->total_time = end_time - s->total_time;
        s->downtime = end_time - start_time;
        if (s->total_time) {
            s->mbps = (((double) transferred_bytes * 8.0) /
                       ((double) s->total_time)) / 1000;
        }
        runstate_set(RUN_STATE_POSTMIGRATE);
    } else {
        if (old_vm_running) {
            vm_start();
        }
    }
    qemu_bh_schedule(s->cleanup_bh);
    qemu_mutex_unlock_iothread();

    return NULL;
}

{%endhighlight%}

第一阶段 
在迁移前，先调用了ram_save_setup函数，用来进行迁移初始化，比如压缩算法，用来记录脏页的bitmap等。

{%highlight c%}
static int ram_save_setup(QEMUFile *f, void *opaque)
{
    RAMBlock *block;
    int64_t ram_bitmap_pages; /* Size of bitmap in pages, including gaps */

    mig_throttle_on = false;
    dirty_rate_high_cnt = 0;
    bitmap_sync_count = 0;
    migration_bitmap_sync_init();

    //如果支持使用xbzrle算法，进行的初始化。
    if (migrate_use_xbzrle()) {
        XBZRLE_cache_lock();
        XBZRLE.cache = cache_init(migrate_xbzrle_cache_size() /
                                  TARGET_PAGE_SIZE,
                                  TARGET_PAGE_SIZE);
        if (!XBZRLE.cache) {
            XBZRLE_cache_unlock();
            error_report("Error creating cache");
            return -1;
        }
        XBZRLE_cache_unlock();

        /* We prefer not to abort if there is no memory */
        XBZRLE.encoded_buf = g_try_malloc0(TARGET_PAGE_SIZE);
        if (!XBZRLE.encoded_buf) {
            error_report("Error allocating encoded_buf");
            return -1;
        }

        XBZRLE.current_buf = g_try_malloc(TARGET_PAGE_SIZE);
        if (!XBZRLE.current_buf) {
            error_report("Error allocating current_buf");
            g_free(XBZRLE.encoded_buf);
            XBZRLE.encoded_buf = NULL;
            return -1;
        }

        acct_clear();
    }

    /* iothread lock needed for ram_list.dirty_memory[] */
    qemu_mutex_lock_iothread();
    qemu_mutex_lock_ramlist();
    rcu_read_lock();
    bytes_transferred = 0;
    reset_ram_globals();

    //根据内存大小申请一个bitmap
    ram_bitmap_pages = last_ram_offset() >> TARGET_PAGE_BITS;
    migration_bitmap = bitmap_new(ram_bitmap_pages);
    bitmap_set(migration_bitmap, 0, ram_bitmap_pages);

    /*
     * Count the total number of pages used by ram blocks not including any
     * gaps due to alignment or unplugs.
     */
    migration_dirty_pages = ram_bytes_total() >> TARGET_PAGE_BITS;

    //开始统计脏页
    memory_global_dirty_log_start();
    migration_bitmap_sync();
    qemu_mutex_unlock_ramlist();
    qemu_mutex_unlock_iothread();

    qemu_put_be64(f, ram_bytes_total() | RAM_SAVE_FLAG_MEM_SIZE);

    QLIST_FOREACH_RCU(block, &ram_list.blocks, next) {
        qemu_put_byte(f, strlen(block->idstr));
        qemu_put_buffer(f, (uint8_t *)block->idstr, strlen(block->idstr));
        qemu_put_be64(f, block->used_length);
    }

    rcu_read_unlock();

    ram_control_before_iterate(f, RAM_CONTROL_SETUP);
    ram_control_after_iterate(f, RAM_CONTROL_SETUP);

    qemu_put_be64(f, RAM_SAVE_FLAG_EOS);

    return 0;
}

{%endhighlight%}

memory_global_dirty_log_start函数，开始标记要开始进行脏页统计了。
宏MEMORY_LISTENER_CALL_GLOBAL主要调用memory_listenesrs中注册的各个内存模块的log_global_start函数。kvm在这里重新注册了新的memory_region来设置了dirty_log标记。
{%highlight c%}
void memory_global_dirty_log_start(void)
{
    global_dirty_log = true;
    MEMORY_LISTENER_CALL_GLOBAL(log_global_start, Forward);
}
  
{%endhighlight%}

在kvm下，通过memory_listener_register函数注册了内存变化监听。

{%highlight c%}
memory_listener_register(&kvm_memory_listener, &address_space_memory);                             
memory_listener_register(&kvm_io_listener, &address_space_io);
{%endhighlight%}


通知kvm后，要调用migration_bitmap_sync函数来设置脏页。

{%highlight c%}
static void migration_bitmap_sync(void)
{
    RAMBlock *block;
    uint64_t num_dirty_pages_init = migration_dirty_pages;
    MigrationState *s = migrate_get_current();
    int64_t end_time;
    int64_t bytes_xfer_now;
    static uint64_t xbzrle_cache_miss_prev;
    static uint64_t iterations_prev;

    //这个全局变量记录了同步了多少次
    bitmap_sync_count++;

    if (!bytes_xfer_prev) {
        bytes_xfer_prev = ram_bytes_transferred();
    }

    if (!start_time) {
        start_time = qemu_clock_get_ms(QEMU_CLOCK_REALTIME);
    }

    trace_migration_bitmap_sync_start();
    //这里主要调用了kvm_log_sync来从kvm向qemu对bitmap进行更新。
    address_space_sync_dirty_bitmap(&address_space_memory);

    rcu_read_lock();
    
    QLIST_FOREACH_RCU(block, &ram_list.blocks, next) {
        //统计需要sync的脏页有多少
        migration_bitmap_sync_range(block->mr->ram_addr, block->used_length);
    }
    rcu_read_unlock();

    trace_migration_bitmap_sync_end(migration_dirty_pages
                                    - num_dirty_pages_init);
    num_dirty_pages_period += migration_dirty_pages - num_dirty_pages_init;
    end_time = qemu_clock_get_ms(QEMU_CLOCK_REALTIME);

    /* more than 1 second = 1000 millisecons */
    //大于1秒需要看是否支持auto_converge以及xbzrle的数据统计
    if (end_time > start_time + 1000) {
        if (migrate_auto_converge()) {
            /* The following detection logic can be refined later. For now:
               Check to see if the dirtied bytes is 50% more than the approx.
               amount of bytes that just got transferred since the last time we
               were in this routine. If that happens >N times (for now N==4)
               we turn on the throttle down logic */
            bytes_xfer_now = ram_bytes_transferred();
            if (s->dirty_pages_rate &&
               (num_dirty_pages_period * TARGET_PAGE_SIZE >
                   (bytes_xfer_now - bytes_xfer_prev)/2) &&
               (dirty_rate_high_cnt++ > 4)) {
                    trace_migration_throttle();
                    mig_throttle_on = true;
                    dirty_rate_high_cnt = 0;
             }
             bytes_xfer_prev = bytes_xfer_now;
        } else {
             mig_throttle_on = false;
        }
        if (migrate_use_xbzrle()) {
            if (iterations_prev != 0) {
                acct_info.xbzrle_cache_miss_rate =
                   (double)(acct_info.xbzrle_cache_miss -
                            xbzrle_cache_miss_prev) /
                   (acct_info.iterations - iterations_prev);
            }
            iterations_prev = acct_info.iterations;
            xbzrle_cache_miss_prev = acct_info.xbzrle_cache_miss;
        }
        s->dirty_pages_rate = num_dirty_pages_period * 1000
            / (end_time - start_time);
        s->dirty_bytes_rate = s->dirty_pages_rate * TARGET_PAGE_SIZE;
        start_time = end_time;
        num_dirty_pages_period = 0;
        s->dirty_sync_count = bitmap_sync_count;
    }
}
{%endhighlight%}



开始第二阶段：迁移

ram_save_pending比较简单，就是调用了migration_bitmap_sync函数。

{%highlight c%}
static uint64_t ram_save_pending(QEMUFile *f, void *opaque, uint64_t max_size)
{
    uint64_t remaining_size;

    remaining_size = ram_save_remaining() * TARGET_PAGE_SIZE;

    if (remaining_size < max_size) {
        qemu_mutex_lock_iothread();
        rcu_read_lock();
        migration_bitmap_sync();
        rcu_read_unlock();
        qemu_mutex_unlock_iothread();
        remaining_size = ram_save_remaining() * TARGET_PAGE_SIZE;
    }
    return remaining_size;
}
{%endhighlight%}

ram_save_iterate真正开始进行迁移的函数。主要调用ram_find_and_save_block对page进行迁移。


{%highlight c%}
static int ram_save_iterate(QEMUFile *f, void *opaque)
{
    int ret;
    int i;
    int64_t t0;
    int pages_sent = 0;

    rcu_read_lock();
    if (ram_list.version != last_version) {
        reset_ram_globals();
    }

    /* Read version before ram_list.blocks */
    smp_rmb();

    ram_control_before_iterate(f, RAM_CONTROL_ROUND);

    t0 = qemu_clock_get_ns(QEMU_CLOCK_REALTIME);
    i = 0;
    while ((ret = qemu_file_rate_limit(f)) == 0) {
        int pages;

        pages = ram_find_and_save_block(f, false, &bytes_transferred);
        /* no more pages to sent */
        if (pages == 0) {
            break;
        }
        pages_sent += pages;
        acct_info.iterations++;
        check_guest_throttling();
        /* we want to check in the 1st loop, just in case it was the 1st time
           and we had to sync the dirty bitmap.
           qemu_get_clock_ns() is a bit expensive, so we only check each some
           iterations
        */
        if ((i & 63) == 0) {
            uint64_t t1 = (qemu_clock_get_ns(QEMU_CLOCK_REALTIME) - t0) / 1000000;
            if (t1 > MAX_WAIT) {
                DPRINTF("big wait: %" PRIu64 " milliseconds, %d iterations\n",
                        t1, i);
                break;
            }
        }
        i++;
    }
    rcu_read_unlock();

    /*
     * Must occur before EOS (or any QEMUFile operation)
     * because of RDMA protocol.
     */
    ram_control_after_iterate(f, RAM_CONTROL_ROUND);

    qemu_put_be64(f, RAM_SAVE_FLAG_EOS);
    bytes_transferred += 8;

    ret = qemu_file_get_error(f);
    if (ret < 0) {
        return ret;
    }

    return pages_sent;
}

{%endhighlight%}

ram_find_and_save_block真正的传输函数。

{%highlight c%}
static int ram_find_and_save_block(QEMUFile *f, bool last_stage,
                                   uint64_t *bytes_transferred)
{
    RAMBlock *block = last_seen_block;
    ram_addr_t offset = last_offset;
    bool complete_round = false;
    int pages = 0;
    MemoryRegion *mr;

    if (!block)
        block = QLIST_FIRST_RCU(&ram_list.blocks);

    while (true) {
        mr = block->mr;
        offset = migration_bitmap_find_and_reset_dirty(mr, offset);
        if (complete_round && block == last_seen_block &&
            offset >= last_offset) {
            break;
        }
        if (offset >= block->used_length) {
            offset = 0;
            block = QLIST_NEXT_RCU(block, next);
            if (!block) {
                block = QLIST_FIRST_RCU(&ram_list.blocks);
                complete_round = true;
                ram_bulk_stage = false;
            }
        } else {
            //通过f对data进行发送
            pages = ram_save_page(f, block, offset, last_stage,
                                  bytes_transferred);

            /* if page is unmodified, continue to the next */
            if (pages > 0) {
                last_sent_block = block;
                break;
            }
        }
    }

    last_seen_block = block;
    last_offset = offset;

    return pages;
}

{%endhighlight%}

qemu_savevm_state_complete函数完成了最后的操作。

{%highlight c%}
void qemu_savevm_state_complete(QEMUFile *f)
{
    QJSON *vmdesc;
    int vmdesc_len;
    SaveStateEntry *se;
    int ret;

    trace_savevm_state_complete();
    //对cpu状态同步
    cpu_synchronize_all_states();

    QTAILQ_FOREACH(se, &savevm_handlers, entry) {
        if (!se->ops || !se->ops->save_live_complete) {
            continue;
        }
        if (se->ops && se->ops->is_active) {
            if (!se->ops->is_active(se->opaque)) {
                continue;
            }
        }
        trace_savevm_section_start(se->idstr, se->section_id);
        /* Section type */
        qemu_put_byte(f, QEMU_VM_SECTION_END);
        qemu_put_be32(f, se->section_id);
        //调用各个device的最后操作
        ret = se->ops->save_live_complete(f, se->opaque);
        trace_savevm_section_end(se->idstr, se->section_id, ret);
        if (ret < 0) {
            qemu_file_set_error(f, ret);
            return;
        }
    }

    vmdesc = qjson_new();
    json_prop_int(vmdesc, "page_size", TARGET_PAGE_SIZE);
    json_start_array(vmdesc, "devices");
    QTAILQ_FOREACH(se, &savevm_handlers, entry) {
        int len;

        if ((!se->ops || !se->ops->save_state) && !se->vmsd) {
            continue;
        }
        trace_savevm_section_start(se->idstr, se->section_id);

        json_start_object(vmdesc, NULL);
        json_prop_str(vmdesc, "name", se->idstr);
        json_prop_int(vmdesc, "instance_id", se->instance_id);

        /* Section type */
        qemu_put_byte(f, QEMU_VM_SECTION_FULL);
        qemu_put_be32(f, se->section_id);

        /* ID string */
        len = strlen(se->idstr);
        qemu_put_byte(f, len);
        qemu_put_buffer(f, (uint8_t *)se->idstr, len);

        qemu_put_be32(f, se->instance_id);
        qemu_put_be32(f, se->version_id);
        
        //会调用各自的vmsd对状态进行保存
        vmstate_save(f, se, vmdesc);

        json_end_object(vmdesc);
        trace_savevm_section_end(se->idstr, se->section_id, 0);
    }

    qemu_put_byte(f, QEMU_VM_EOF);

    json_end_array(vmdesc);
    qjson_finish(vmdesc);
    vmdesc_len = strlen(qjson_get_str(vmdesc));

    if (should_send_vmdesc()) {
        qemu_put_byte(f, QEMU_VM_VMDESCRIPTION);
        qemu_put_be32(f, vmdesc_len);
        qemu_put_buffer(f, (uint8_t *)qjson_get_str(vmdesc), vmdesc_len);
    }
    object_unref(OBJECT(vmdesc));

    qemu_fflush(f);
}

{%endhighlight%}


ram_save_complete做了最后的处理，传输剩下的脏页。

{%highlight c%}
static int ram_save_complete(QEMUFile *f, void *opaque)
{
    rcu_read_lock();
    
    //进行最后的sync
    migration_bitmap_sync();

    ram_control_before_iterate(f, RAM_CONTROL_FINISH);

    /* try transferring iterative blocks of memory */

    /* flush all remaining blocks regardless of rate limiting */
    while (true) {
        int pages;

        pages = ram_find_and_save_block(f, true, &bytes_transferred);
        /* no more blocks to sent */
        if (pages == 0) {
            break;
        }
    }

    ram_control_after_iterate(f, RAM_CONTROL_FINISH);
    migration_end();

    rcu_read_unlock();
    qemu_put_be64(f, RAM_SAVE_FLAG_EOS);

    return 0;
}

{%endhighlight%}

vmstate_save主要是调用vmstate_save_state，以及为了以前的方式做了兼容处理。
{%highlight c%}
static void vmstate_save(QEMUFile *f, SaveStateEntry *se, QJSON *vmdesc)
{
    trace_vmstate_save(se->idstr, se->vmsd ? se->vmsd->name : "(old)");
    if (!se->vmsd) {
        vmstate_save_old_style(f, se, vmdesc);
        return;
    }
    vmstate_save_state(f, se->vmsd, se->opaque, vmdesc);
}

{%endhighlight%}

vmstate_save_state 根据vmsd中定义的handle开始一系列处理 

{%highlight c%}
void vmstate_save_state(QEMUFile *f, const VMStateDescription *vmsd,
                        void *opaque, QJSON *vmdesc)
{
    VMStateField *field = vmsd->fields;

    if (vmsd->pre_save) {
        //这里如果是存储cpu的寄存器等，调用了cpu_pre_save
        vmsd->pre_save(opaque);
    }

    if (vmdesc) {
        json_prop_str(vmdesc, "vmsd_name", vmsd->name);
        json_prop_int(vmdesc, "version", vmsd->version_id);
        json_start_array(vmdesc, "fields");
    }

    while (field->name) {
        if (!field->field_exists ||
            field->field_exists(opaque, vmsd->version_id)) {
            void *base_addr = vmstate_base_addr(opaque, field, false);
            int i, n_elems = vmstate_n_elems(opaque, field);
            int size = vmstate_size(opaque, field);
            int64_t old_offset, written_bytes;
            QJSON *vmdesc_loop = vmdesc;

            for (i = 0; i < n_elems; i++) {
                void *addr = base_addr + size * i;

                vmsd_desc_field_start(vmsd, vmdesc_loop, field, i, n_elems);
                old_offset = qemu_ftell_fast(f);

                if (field->flags & VMS_ARRAY_OF_POINTER) {
                    addr = *(void **)addr;
                }
                if (field->flags & VMS_STRUCT) {
                    vmstate_save_state(f, field->vmsd, addr, vmdesc_loop);
                } else {
                    field->info->put(f, addr, size);
                }

                written_bytes = qemu_ftell_fast(f) - old_offset;
                vmsd_desc_field_end(vmsd, vmdesc_loop, field, written_bytes, i);

                /* Compressed arrays only care about the first element */
                if (vmdesc_loop && vmsd_can_compress(field)) {
                    vmdesc_loop = NULL;
                }
            }
        } else {
            if (field->flags & VMS_MUST_EXIST) {
                error_report("Output state validation failed: %s/%s",
                        vmsd->name, field->name);
                assert(!(field->flags & VMS_MUST_EXIST));
            }
        }
        field++;
    }

    if (vmdesc) {
        json_end_array(vmdesc);
    }

    vmstate_subsection_save(f, vmsd, opaque, vmdesc);
}


{%endhighlight%}

至此，基本代码走读结束。


[热迁移的算法]:http://www.linux-kvm.org/page/Migration
