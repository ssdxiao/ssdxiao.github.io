---

layout: post

title:  libvirt savevm 命令存储内存文件分析
date:   2016-6-30 14:10:00
categories: 虚拟化
tags: 
  - libvirt
  - 虚拟化 
  - qemu
---

在libvirt 使用save命令对虚拟机进行存储后，将libvirt的信息，虚拟机的内存，cpu以及其他设备的信息都存储了起来。

先把文件格式大致的画出来。

{%highlight bash%}
libvirt 存储文件信息
        +-------------+-----+-----+-----+-----+
        |Libvirt-Magic|版本 | xml |运行 |是否 |
        |             |     |长度 |状态 |压缩 |
        |8字节        |4字节|4字节|4字节|4字节|
        +-------------+-----+-----+-----+-----+
        |  保留字段（60字节）                 |
        +-------------------------------------+
        | xml 详细信息（占用xml长度个字节)    |
        +-------------------------------------+
qemu文件开头存储信息。
        +-----+-----+--------+
        |QEMV |版本 |设备信息|
        |     |     |        |
        |4字节|4字节|        |
        +-----+-----+--------+

qemu中savevm_handlers中的设备信息，可能有多个设备。

        设备循环，一个设备信息如下。
        +-------+-------+------+------+------+------+--------+------+
        |section|section|设备名|设备名|实例id|版本id|子设备信|循环  |
        |start  | id    | 长度 |      |      |      | 息     |设备  |
        |1字节  | 4字节 |1字节 |n字节 |4字节 |4字节 |        |      |
        +-------+-------+------+------+------+------+--------+------+

        子设备信息存储 这里以ram设备 为例
                +--------------------------------------+
                |共占用内存字节数 8字节                |
                +------+------+------+--------+--------+
                |设备名|设备名|设备内|循环前面|ram存储 |
                | 长度 |      |存长度|信息    |结束标记|
                |1字节 |n字节 |8字节 |        |8字节   |
                +------+------+------+--------+--------+


        开始存储设备详细信息
        +-------+-------+------+------+
        |section|section|设备详|循环  |
        |part   | id    |细信息|设备  |
        |1字节  |4字节  |n字节 |      |
        +-------+-------+------+------+

        处理存储结束
        +-------+-------+------+------+
        |section|section|设备结|循环  |
        |end   | id     |束信息|设备  |
        |1字节  |4字节  |n字节 |      |
        +-------+-------+------+------+
        存储设备状态
        +-------+-------+------+------+
        |section|section|设备状|循环  |
        |full   | id    |态信息|设备  |
        |1字节  |4字节  |n字节 |      |
        +-------+-------+------+------+

        +-----+
        |结束 |
        |标记 |
        |1字节|
        +-----+


{%endhighlight%}
这里就对这一文件进行分析。

首先将存储文件的2进制信息发出来。

{%highlight bash%}
00000000  4c 69 62 76 69 72 74 51  65 6d 75 64 53 61 76 65  |LibvirtQemudSave|
00000010  02 00 00 00 07 12 00 00  01 00 00 00 00 00 00 00  |................|
00000020  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
00000030  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
00000040  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
00000050  00 00 00 00 00 00 00 00  00 00 00 00 3c 64 6f 6d  |............<dom|

{%endhighlight%}

这是存储文件中的一部分信息，结合代码来看一下这个文件究竟是什么

{%highlight c%}
#define QEMU_SAVE_MAGIC   "LibvirtQemudSave"

typedef enum {
    QEMU_SAVE_FORMAT_RAW = 0,
    QEMU_SAVE_FORMAT_GZIP = 1,
    QEMU_SAVE_FORMAT_BZIP2 = 2,
    /*
     * Deprecated by xz and never used as part of a release
     * QEMU_SAVE_FORMAT_LZMA
     */
    QEMU_SAVE_FORMAT_XZ = 3,
    QEMU_SAVE_FORMAT_LZOP = 4,
    /* Note: add new members only at the end.
       These values are used in the on-disk format.
       Do not change or re-use numbers. */

    QEMU_SAVE_FORMAT_LAST
} virQEMUSaveFormat;

struct _virQEMUSaveHeader {
    char magic[sizeof(QEMU_SAVE_MAGIC)-1]; //libvirt 存储内存文件的魔数LibvirtQemudSave 16个字节
    uint32_t version; //版本信息 上述例子是02
    uint32_t xml_len; //存储的xml配置信息长度0x1207
    uint32_t was_running;//01 表示恢复时运行，00表示恢复时处于暂停状态
    uint32_t compressed;//表示压缩方式，0表示raw，其他格式根据virQEMUSaveFormat定义。
    uint32_t unused[15];
};
{%endhighlight%}
首先，文件最开始存储了_virQEMUSaveHeader 文件的头信息。
头信息总共占用了0x5b个字节。

忽略libvirt存储配置的一部分，然后直接到达qemu开始存储文件的位置。
根据上面头文件信息，可以得知，qemu存储文件的起始位置为0x1207+0x5b=0x1262

{%highlight bash%}
00001250  63 6c 61 62 65 6c 3e 0a  3c 2f 64 6f 6d 61 69 6e  |clabel>.</domain|
00001260  3e 0a 00 51 45 56 4d 00  00 00 03 01 00 00 00 02  |>..QEVM.........|
00001270  03 72 61 6d 00 00 00 00  00 00 00 04 00 00 00 00  |.ram............|
00001280  80 8d 10 04 06 70 63 2e  72 61 6d 00 00 00 00 80  |.....pc.ram.....|
00001290  00 00 00 08 76 67 61 2e  76 72 61 6d 00 00 00 00  |....vga.vram....|
000012a0  00 80 00 00 07 70 63 2e  62 69 6f 73 00 00 00 00  |.....pc.bios....|
000012b0  00 04 00 00 1f 30 30 30  30 3a 30 30 3a 30 33 2e  |.....0000:00:03.|
000012c0  30 2f 76 69 72 74 69 6f  2d 6e 65 74 2d 70 63 69  |0/virtio-net-pci|
000012d0  2e 72 6f 6d 00 00 00 00  00 04 00 00 06 70 63 2e  |.rom.........pc.|
000012e0  72 6f 6d 00 00 00 00 00  02 00 00 14 2f 72 6f 6d  |rom........./rom|
000012f0  40 65 74 63 2f 61 63 70  69 2f 74 61 62 6c 65 73  |@etc/acpi/tables|
00001300  00 00 00 00 00 02 00 00  1b 30 30 30 30 3a 30 30  |.........0000:00|
00001310  3a 30 32 2e 30 2f 63 69  72 72 75 73 5f 76 67 61  |:02.0/cirrus_vga|
00001320  2e 72 6f 6d 00 00 00 00  00 01 00 00 15 2f 72 6f  |.rom........./ro|
00001330  6d 40 65 74 63 2f 74 61  62 6c 65 2d 6c 6f 61 64  |m@etc/table-load|
00001340  65 72 00 00 00 00 00 00  10 00 00 00 00 00 00 00  |er..............|
00001350  00 10 02 00 00 00 02 00  00 00 00 00 00 00 08 06  |................|
00001360  70 63 2e 72 61 6d 53 ff  00 f0 53 ff 00 f0 c3 e2  |pc.ramS...S.....|
{%endhighlight%}

可以看到从0x1263开始就是qemu存储的文件了。接下来就分析qemu存储文件格式。


这是存储qemu虚拟机状态的函数，这里可以看按到大致流程
{%highlight c%}
static int qemu_savevm_state(QEMUFile *f)
{
    int ret;
    MigrationParams params = {
        .blk = 0,
        .shared = 0
    };

    if (qemu_savevm_state_blocked(NULL)) {
        return -EINVAL;
    }

    qemu_mutex_unlock_iothread();
    qemu_savevm_state_begin(f, &params);//存储虚拟机状态主要是有数据的设备存储。
    qemu_mutex_lock_iothread();

    while (qemu_file_get_error(f) == 0) {
        if (qemu_savevm_state_iterate(f) > 0) { //存储具体信息。
            break;
        }
    }

    ret = qemu_file_get_error(f);
    if (ret == 0) {
        qemu_savevm_state_complete(f);//进行有是护具的存储设备结尾，同时对有状态的设备进行存储。
        ret = qemu_file_get_error(f);
    }
    if (ret != 0) {
        qemu_savevm_state_cancel();
    }
    return ret;
}
{%endhighlight%}




{%highlight c%}
#define QEMU_VM_FILE_MAGIC           0x5145564d
#define QEMU_VM_FILE_VERSION         0x00000003

#define QEMU_VM_SECTION_START        0x01

void qemu_savevm_state_begin(QEMUFile *f,
                             const MigrationParams *params)
{
    SaveStateEntry *se;
    int ret;

    trace_savevm_state_begin();
    QTAILQ_FOREACH(se, &savevm_handlers, entry) {
        if (!se->ops || !se->ops->set_params) {
            continue;
        }
        se->ops->set_params(params, se->opaque);
    }

    qemu_put_be32(f, QEMU_VM_FILE_MAGIC);//qemu文件起始QEVM 4字节
    qemu_put_be32(f, QEMU_VM_FILE_VERSION);//qemu文件的版本 4字节

    QTAILQ_FOREACH(se, &savevm_handlers, entry) {
        int len;

        if (!se->ops || !se->ops->save_live_setup) {
            continue;
        }
        if (se->ops && se->ops->is_active) {
            if (!se->ops->is_active(se->opaque)) {
                continue;
            }
        }
        /* Section type */
        qemu_put_byte(f, QEMU_VM_SECTION_START);//设备起始·1字节
        qemu_put_be32(f, se->section_id);//设备的section_id 4字节

        /* ID string */
        len = strlen(se->idstr);
        qemu_put_byte(f, len);//设备字符串长度1字节
        qemu_put_buffer(f, (uint8_t *)se->idstr, len);//设备字符

        qemu_put_be32(f, se->instance_id);//实例id 4字节
        qemu_put_be32(f, se->version_id);//版本id 4字节

        ret = se->ops->save_live_setup(f, se->opaque);//进入具体的设备存储信息，这里首先是ram
        if (ret < 0) {
            qemu_file_set_error(f, ret);
            break;
        }
    }
}

{%endhighlight%}

首先是qemu中的魔数（4字节），qemu文件版本（4字节），
然后就是需要save的设备的信息，首先是section start的标记（1字节）
然后是section_id。（4字节）设备名字长度1字节，设备名字，然后实例id（4字节）（版本id）4字节



{%highlight c%}

#define RAM_SAVE_FLAG_EOS      0x10

static int ram_save_setup(QEMUFile *f, void *opaque)
{
    RAMBlock *block;
    int64_t ram_bitmap_pages; /* Size of bitmap in pages, including gaps */

    mig_throttle_on = false;
    dirty_rate_high_cnt = 0;
    bitmap_sync_count = 0;
    migration_bitmap_sync_init();

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

    qemu_mutex_lock_iothread();
    qemu_mutex_lock_ramlist();
    bytes_transferred = 0;
    reset_ram_globals();

    ram_bitmap_pages = last_ram_offset() >> TARGET_PAGE_BITS;
    migration_bitmap = bitmap_new(ram_bitmap_pages);
    bitmap_set(migration_bitmap, 0, ram_bitmap_pages);

    /*
     * Count the total number of pages used by ram blocks not including any
     * gaps due to alignment or unplugs.
     */
    migration_dirty_pages = 0;
    QTAILQ_FOREACH(block, &ram_list.blocks, next) {
        uint64_t block_pages;

        block_pages = block->length >> TARGET_PAGE_BITS;
        migration_dirty_pages += block_pages;
    }

    memory_global_dirty_log_start();
    migration_bitmap_sync();
    qemu_mutex_unlock_iothread();

    qemu_put_be64(f, ram_bytes_total() | RAM_SAVE_FLAG_MEM_SIZE);//存了8位用来表示内存站了多少个字节 

    QTAILQ_FOREACH(block, &ram_list.blocks, next) {
        qemu_put_byte(f, strlen(block->idstr));//1个字节长度
        qemu_put_buffer(f, (uint8_t *)block->idstr, strlen(block->idstr));//设备名字
        qemu_put_be64(f, block->length);//设备占用长度。//8字节
    }

    qemu_mutex_unlock_ramlist();

    ram_control_before_iterate(f, RAM_CONTROL_SETUP);
    ram_control_after_iterate(f, RAM_CONTROL_SETUP);

    qemu_put_be64(f, RAM_SAVE_FLAG_EOS); //存储结尾8字节

    return 0;
}
{%endhighlight%}

0x127b开始，就是ram的save函数进行记录的信息。
首先是存储了8字节 ram总共占用了多少字节 这里是0x808d1004。这里除了guest的内存还有其他设备的寄存器也在这里保存。
接下来是pc.ram 内存，我们的虚拟机使用了2G内存，使用了0x80000000。
vga.vram 显卡的显存 8M 使用了0x800000
pc.bios bios寄存器 256k 0x40000
virtio-net-pci寄存器 256k 0x40000
acpi表 128k 0x20000
cirrus_vga显存 64k 0x010000
acpi的table-loader  4k 0x1000
存储结尾标记 RAM_SAVE_FLAG_EOS 
这里到达0x1351结束，接下来就是要存储的设备进行一一存储了。

{%highlight c%}
#define QEMU_VM_SECTION_PART         0x02
int qemu_savevm_state_iterate(QEMUFile *f)
{
    SaveStateEntry *se;
    int ret = 1;

    trace_savevm_state_iterate();
    QTAILQ_FOREACH(se, &savevm_handlers, entry) {
        if (!se->ops || !se->ops->save_live_iterate) {
            continue;
        }
        if (se->ops && se->ops->is_active) {
            if (!se->ops->is_active(se->opaque)) {
                continue;
            }
        }
        if (qemu_file_rate_limit(f)) {
            return 0;
        }
        trace_savevm_section_start(se->idstr, se->section_id);
        /* Section type */
        qemu_put_byte(f, QEMU_VM_SECTION_PART);// 开始标记1字节
        qemu_put_be32(f, se->section_id);//section id

        ret = se->ops->save_live_iterate(f, se->opaque);
        trace_savevm_section_end(se->idstr, se->section_id);

        if (ret < 0) {
            qemu_file_set_error(f, ret);
        }
        if (ret <= 0) {
            /* Do not proceed to the next vmstate before this one reported
               completion of the current stage. This serializes the migration
               and reduces the probability that a faster changing state is
               synchronized over and over again. */
            break;
        }
    }
    return ret;
}
{%endhighlight%}

记录了设备的其实信息后，就会调用设备自身来进行数据存储到达0x1356
{%highlight c%}
static int ram_save_iterate(QEMUFile *f, void *opaque)
{
    int ret;
    int i;
    int64_t t0;
    int total_sent = 0;

    qemu_mutex_lock_ramlist();

    if (ram_list.version != last_version) {
        reset_ram_globals();
    }

    ram_control_before_iterate(f, RAM_CONTROL_ROUND);

    t0 = qemu_clock_get_ns(QEMU_CLOCK_REALTIME);
    i = 0;
    while ((ret = qemu_file_rate_limit(f)) == 0) {
        int bytes_sent;

        bytes_sent = ram_find_and_save_block(f, false);//开始存储
        /* no more blocks to sent */
        if (bytes_sent == 0) {
            break;
        }
        total_sent += bytes_sent;
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

    qemu_mutex_unlock_ramlist();

    /*
     * Must occur before EOS (or any QEMUFile operation)
     * because of RDMA protocol.
     */
    ram_control_after_iterate(f, RAM_CONTROL_ROUND);

    bytes_transferred += total_sent;

    /*
     * Do not count these 8 bytes into total_sent, so that we can
     * return 0 if no page had been dirtied.
     */
    qemu_put_be64(f, RAM_SAVE_FLAG_EOS);//8字节的结束标记
    bytes_transferred += 8;

    ret = qemu_file_get_error(f);
    if (ret < 0) {
        return ret;
    }

    return total_sent;
}

{%endhighlight%}
在ram_find_and_save_block中 进行了实际的存储。

{%highlight c%}
static int ram_find_and_save_block(QEMUFile *f, bool last_stage)
{
    RAMBlock *block = last_seen_block;
    ram_addr_t offset = last_offset;
    bool complete_round = false;
    int bytes_sent = 0;
    MemoryRegion *mr;

    if (!block)
        block = QTAILQ_FIRST(&ram_list.blocks);

    while (true) {
        mr = block->mr;
        offset = migration_bitmap_find_and_reset_dirty(mr, offset);//从dirty页中拿出一页
        if (complete_round && block == last_seen_block &&
            offset >= last_offset) {
            break;
        }
        if (offset >= block->length) {
            offset = 0;
            block = QTAILQ_NEXT(block, next);
            if (!block) {
                block = QTAILQ_FIRST(&ram_list.blocks);
                complete_round = true;
                ram_bulk_stage = false;
            }
        } else {
            bytes_sent = ram_save_page(f, block, offset, last_stage);//存储page

            /* if page is unmodified, continue to the next */
            if (bytes_sent > 0) {
                last_sent_block = block;//标记上一次存储的设备
                break;
            }
        }
    }
    last_seen_block = block;
    last_offset = offset;

    return bytes_sent;
}
{%endhighlight%}
ram_save_page 继续执行存储page
{%highlight c%}

#define RAM_SAVE_FLAG_CONTINUE 0x20

static int ram_save_page(QEMUFile *f, RAMBlock* block, ram_addr_t offset,
                         bool last_stage)
{
    int bytes_sent;
    int cont;
    ram_addr_t current_addr;
    MemoryRegion *mr = block->mr;
    uint8_t *p;
    int ret;
    bool send_async = true;

    cont = (block == last_sent_block) ? RAM_SAVE_FLAG_CONTINUE : 0;//这里进行了比较如果新的block，这里标记cont

    p = memory_region_get_ram_ptr(mr) + offset;//获取要存储页的地址

	
    ...
	 } else if (is_zero_range(p, TARGET_PAGE_SIZE)) {//如果页面全0就会采用压缩方式。
        acct_info.dup_pages++;
        bytes_sent = save_block_hdr(f, block, offset, cont,
                                    RAM_SAVE_FLAG_COMPRESS);
        qemu_put_byte(f, 0);//并存入一个字节0
        bytes_sent++;

	...

    /* XBZRLE overflow or normal page */
    if (bytes_sent == -1) {
        bytes_sent = save_block_hdr(f, block, offset, cont, RAM_SAVE_FLAG_PAGE);//存储页的偏移
        if (send_async) {
            qemu_put_buffer_async(f, p, TARGET_PAGE_SIZE);//实际存储数据异步
        } else {
            qemu_put_buffer(f, p, TARGET_PAGE_SIZE);//实际存储数据同步
        }
        bytes_sent += TARGET_PAGE_SIZE;
        acct_info.norm_pages++;
    }

    XBZRLE_cache_unlock();

    return bytes_sent;
}
{%endhighlight%}

这里来看一下如何存储页面偏移。
{%highlight c%}
#define RAM_SAVE_FLAG_PAGE     0x08

static size_t save_block_hdr(QEMUFile *f, RAMBlock *block, ram_addr_t offset,
                             int cont, int flag)
{
    size_t size;

    qemu_put_be64(f, offset | cont | flag); //一个8字节的标记位
    size = 8;

    if (!cont) {//如果是第一次存储，会存储设备的名字。
        qemu_put_byte(f, strlen(block->idstr));
        qemu_put_buffer(f, (uint8_t *)block->idstr,
                        strlen(block->idstr));
        size += 1 + strlen(block->idstr);
    }
    return size;
}
{%endhighlight%}

这样就可以看到 0x1356后接下来是存储8个字节page的标记位
0x00000008 就是offset=0 cont=0 flag=RAM_SAVE_FLAG_PAGE,因为是第一次，所以存储了
设备的名字。
从0x1366开始就是存储4k的页面，
在ram_find_and_save_block中 会循环执行ram_save_page函数，直到全部存储完所有内存。
因此存储模式就是 页偏移+页内容
所以接下来直接查看0x2366的信息。
{%highlight bash%}
00002360  00 00 00 00 00 00 00 00  00 00 00 00 10 22 00 00  |............."..|
00002370  00 00 00 00 00 20 22 00  00 00 00 00 00 00 30 22  |..... ".......0"|
00002380  00 00 00 00 00 00 00 40  22 00 00 00 00 00 00 00  |.......@".......|
00002390  50 22 00 00 00 00 00 00  00 60 28 00 00 00 00 00  |P".......`(.....|
000023a0  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
000023b0  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
000023c0  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
000023d0  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
000023e0  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
000023f0  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
{%endhighlight%}
可以看到第二个页的标记是0x1022，
由于第二个页面是全0页，所以做了压缩处理。第二个页offset=0x10000，cont=0x20，flag=RAM_SAVE_FLAG_COMPRESS。
然后存入1字节0
紧接着0x236e是第三个页面是0x2022 后面的就不过多说明了。
pc.ram 是2G，当pc.ram存储完后，会接着存储其他设备。

{%highlight bash%}
00939ec0  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
00939ed0  00 00 00 02 08 76 67 61  2e 76 72 61 6d 00 00 00  |.....vga.vram...|
00939ee0  00 00 00 00 10 22 00 00  00 00 00 00 00 20 22 00  |....."....... ".|
00939ef0  00 00 00 00 00 00 30 22  00 00 00 00 00 00 00 40  |......0".......@|
00939f00  22 00 00 00 00 00 00 00  50 22 00 00 00 00 00 00  |".......P"......|
{%endhighlight%}

可以看到从0x00939cb开始就是存储vag.ram的信息。由于第一个页就是0页。值是0x00000002.
所以标记offset=0，cont=0，flag=RAM_SAVE_FLAG_COMPRESS。由于第一次存储该设备，所以存入设备名字。


当这些设备全部存储完毕后，就是存储结尾。
{%highlight bash%}
00a72d40  00 00 00 00 00 e0 22 00  00 00 00 00 00 00 f0 22  |......"........"|
00a72d50  00 00 00 00 00 00 00 00  08 15 2f 72 6f 6d 40 65  |........../rom@e|
00a72d60  74 63 2f 74 61 62 6c 65  2d 6c 6f 61 64 65 72 01  |tc/table-loader.|
00a72d70  00 00 00 65 74 63 2f 61  63 70 69 2f 72 73 64 70  |...etc/acpi/rsdp|
{%endhighlight%}

这里从最后一个 设备开始查找，最后一个设备占用4k，所以从0x00a72d6d开始向后4k来查找。

{%highlight bash%}
00a73d50  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................| 
00a73d60  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
00a73d70  00 00 00 00 00 00 10 03  00 00 00 02 00 00 00 00  |................|
00a73d80  00 00 00 10 04 00 00 00  00 05 74 69 6d 65 72 00  |..........timer.|
00a73d90  00 00 00 00 00 00 02 00  00 00 05 9f e1 15 85 00  |................|
00a73da0  00 00 00 00 00 00 00 00  00 00 02 73 7e b6 7e 04  |...........s~.~.|
00a73db0  00 00 00 03 0a 63 70 75  5f 63 6f 6d 6d 6f 6e 00  |.....cpu_common.|
{%endhighlight%}
从0x00a73d6e开始设备信息查找完成，然后存入ram结束信息RAM_SAVE_FLAG_EOS 8字节 0x00000010 。
从0x00a73d77开始 ，就是进行最后一次存储，此时会存储cpu信息。
 

{%highlight c%}
#define QEMU_VM_SECTION_END          0x03
#define QEMU_VM_SECTION_FULL         0x04
#define QEMU_VM_EOF                  0x00

void qemu_savevm_state_complete(QEMUFile *f)
{       
    SaveStateEntry *se;
    int ret;
    
    trace_savevm_state_complete();
        
    cpu_synchronize_all_states();//同步此时的cpu状态。

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

        ret = se->ops->save_live_complete(f, se->opaque);//调用设备的complete函数
        trace_savevm_section_end(se->idstr, se->section_id);
        if (ret < 0) {
            qemu_file_set_error(f, ret);
            return;
        }
    }
    //开始存储具有状态的设备。
    QTAILQ_FOREACH(se, &savevm_handlers, entry) {
        int len;

        if ((!se->ops || !se->ops->save_state) && !se->vmsd) {
            continue;
        }
        trace_savevm_section_start(se->idstr, se->section_id);
        /* Section type */
        qemu_put_byte(f, QEMU_VM_SECTION_FULL); //1字节section full
        qemu_put_be32(f, se->section_id); 

        /* ID string */
        len = strlen(se->idstr);
        qemu_put_byte(f, len);
        qemu_put_buffer(f, (uint8_t *)se->idstr, len);

        qemu_put_be32(f, se->instance_id);
        qemu_put_be32(f, se->version_id);

        vmstate_save(f, se);
        trace_savevm_section_end(se->idstr, se->section_id);
    }

    qemu_put_byte(f, QEMU_VM_EOF);
    qemu_fflush(f);
}
{%endhighlight%}

{%highlight c%}
static int ram_save_complete(QEMUFile *f, void *opaque)
{
    qemu_mutex_lock_ramlist();
    migration_bitmap_sync();

    ram_control_before_iterate(f, RAM_CONTROL_FINISH);

    /* try transferring iterative blocks of memory */
    ram_control_before_iterate(f, RAM_CONTROL_FINISH);

    /* try transferring iterative blocks of memory */

    /* flush all remaining blocks regardless of rate limiting */
    while (true) {
        int bytes_sent;

        bytes_sent = ram_find_and_save_block(f, true);
        /* no more blocks to sent */
        if (bytes_sent == 0) {
            break;
        }
        bytes_transferred += bytes_sent;
    }

    ram_control_after_iterate(f, RAM_CONTROL_FINISH);
    migration_end();

    qemu_mutex_unlock_ramlist();
    qemu_put_be64(f, RAM_SAVE_FLAG_EOS);//存入8字节 ram结束标记。

    return 0;
}
{%endhighlight%}
0x00a73d78 是03 是标记存储结束。sectionid是 0x00000002。然后存入表示RAM_SAVE_FLAG_EOS 8字节0x00000010

从 0x00a73d83开始 存储QEMU_VM_SECTION_FULL 1字节 0x04 然后sectionid 0x00000000
然后是设备的名字，设备instance_id和version_id。然后使用vmstate_save来存储信息。

从0x00a73d89开始是timer设备的信息。

{%highlight c%}
static const VMStateDescription vmstate_timers = {
    .name = "timer",
    .version_id = 2,
    .minimum_version_id = 1,
    .fields = (VMStateField[]) {
        VMSTATE_INT64(cpu_ticks_offset, TimersState),
        VMSTATE_INT64(dummy, TimersState),
        VMSTATE_INT64_V(cpu_clock_offset, TimersState, 2),
        VMSTATE_END_OF_LIST()
    },
    .subsections = (VMStateSubsection[]) {
        {
            .vmsd = &icount_vmstate_timers,
            .needed = icount_state_needed,
        }, {
            /* empty */
        }
    }
};

void vmstate_save_state(QEMUFile *f, const VMStateDescription *vmsd,
                        void *opaque)
{
    VMStateField *field = vmsd->fields;
    
    if (vmsd->pre_save) {    
        vmsd->pre_save(opaque);
    }   
    while (field->name) {
        if (!field->field_exists || 
            field->field_exists(opaque, vmsd->version_id)) {
            void *base_addr = vmstate_base_addr(opaque, field, false);
            int i, n_elems = vmstate_n_elems(opaque, field);
            int size = vmstate_size(opaque, field);//获取state的size

            for (i = 0; i < n_elems; i++) {
                void *addr = base_addr + size * i;

                if (field->flags & VMS_ARRAY_OF_POINTER) {
                    addr = *(void **)addr;
                }
                if (field->flags & VMS_STRUCT) {
                    vmstate_save_state(f, field->vmsd, addr);
                } else {
                    field->info->put(f, addr, size);//这里对state进行存储
                }
            }
        } else {
            if (field->flags & VMS_MUST_EXIST) {
                fprintf(stderr, "Output state validation failed: %s/%s\n",
                        vmsd->name, field->name);
                assert(!(field->flags & VMS_MUST_EXIST));
            }
        }
        field++;
    }
    vmstate_subsection_save(f, vmsd, opaque);
}

{%endhighlight%}

timer设备的state有3个VMSTATE_INT64组成，即24字节。
从00a73d97到00a73dae用存储timer的state信息。

所以对于有状态设备来说，就在这里就进行存储。

后面的设备就不一一分析。

当存储完成后，存入QEMU_VM_EOF。
