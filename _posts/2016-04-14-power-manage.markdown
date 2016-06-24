---

layout: post

title:  VM电源管理xen和kvm比较
date:   2016-4-14 14:10:00
categories: linux
tags: 
  - linux
---

电源管理是在学习虚拟化时候比较容易被忽略的一个问题。
最近在碰到一个问题，如何区分，虚拟机是从外部destroy掉还是用户内部shutdown的行为。
所以顺便好好研究了一下VM电源管理这一块。
首先简单描述一下，linux执行poweroff会做的事情。
当执行poweroff后，会执行系统调用kernel_power_off，然后根据不同架构来调用相关的
poweroff实现。

kernel/sys.c
{%highlight c%}
void kernel_power_off(void)
{
    kernel_shutdown_prepare(SYSTEM_POWER_OFF);
    if (pm_power_off_prepare)
        pm_power_off_prepare();
    migrate_to_reboot_cpu();
    syscore_shutdown();
    printk(KERN_EMERG "Power down.\n");
    kmsg_dump(KMSG_DUMP_POWEROFF);
    machine_power_off();                                                                                           
}
{%endhighlight%}

arch/x86/kernel/reboot.c
{%highlight c%}

struct machine_ops machine_ops = {                                                                                 
    .power_off = native_machine_power_off,
    .shutdown = native_machine_shutdown,
    .emergency_restart = native_machine_emergency_restart,
    .restart = native_machine_restart,
    .halt = native_machine_halt,
#ifdef CONFIG_KEXEC
    .crash_shutdown = native_machine_crash_shutdown,
#endif
};
{%endhighlight%}

在现在计算机中，电源管理交给了APM和ACPI，因为APM有着天然缺陷，所以现在一般都会使用ACPI。
使用ACPI需要在kernel中开启选项。值得一提的是APM和ACPI只有一个会生效。
在我测试的发行版中，redhat7，redhat6 已结sles11sp3中，kernel都只开启了ACPI，而没有编译APM，
所以如果禁止了ACPI选项，就会出现没有电源管理驱动加载。

好了，先来看看kvm下的电源管理。
kvm使用qemu来作为设备模拟，所以kvm直接使用了qemu中piix4中初始化的ACPI模块。

hw/i386/pc_piix.c
pc_init1
{%highlight c%}
    if (pci_enabled && acpi_enabled) {
        DeviceState *piix4_pm;
        I2CBus *smbus;

        smi_irq = qemu_allocate_irqs(pc_acpi_smi_interrupt, first_cpu, 1);
        /* TODO: Populate SPD eeprom data.  */
        smbus = piix4_pm_init(pci_bus, piix3_devfn + 3, 0xb100,
                              gsi[9], *smi_irq,
                              kvm_enabled(), fw_cfg, &piix4_pm);
        smbus_eeprom_init(smbus, 8, NULL, 0);

        object_property_add_link(OBJECT(machine), PC_MACHINE_ACPI_DEVICE_PROP,
                                 TYPE_HOTPLUG_HANDLER,
                                 (Object **)&pc_machine->acpi_dev,
                                 object_property_allow_set_link,
                                 OBJ_PROP_LINK_UNREF_ON_RELEASE, &error_abort);
        object_property_set_link(OBJECT(machine), OBJECT(piix4_pm),
                                 PC_MACHINE_ACPI_DEVICE_PROP, &error_abort);
    }
{%endhighlight%}

可以看到，如果开启了ACPI，则会触发piix4_pm 这个芯片的初始化。
这个芯片在 
hw/acpi/piix4.c 中。
这里就不细看了，只需要查看，相关寄存器读写以后会有什么状态触发。

在hw/acpi/core.c 中可以看到在这个acpi-cnt的io空间，如果进行write操作，就会
触发qemu_system_shutdown_request函数。这个函数，就会发起qemu退出信号。然后然后整个
qemu进程就退出了。

{%highlight c%}

 memory_region_init_io(&ar->pm1.cnt.io, memory_region_owner(parent),
                          &acpi_pm_cnt_ops, ar, "acpi-cnt", 2);


static const MemoryRegionOps acpi_pm_cnt_ops = {
    .read = acpi_pm_cnt_read,
    .write = acpi_pm_cnt_write,
    .valid.min_access_size = 2,
    .valid.max_access_size = 2,
    .endianness = DEVICE_LITTLE_ENDIAN,
};



static void acpi_pm1_cnt_write(ACPIREGS *ar, uint16_t val)
{
    ar->pm1.cnt.cnt = val & ~(ACPI_BITMASK_SLEEP_ENABLE);

    if (val & ACPI_BITMASK_SLEEP_ENABLE) {
        /* change suspend type */
        uint16_t sus_typ = (val >> 10) & 7;
        switch(sus_typ) {
        case 0: /* soft power off */                                                                        
            qemu_system_shutdown_request();
            break;
        case 1:
            qemu_system_suspend_request();
            break;
        default:
            if (sus_typ == ar->pm1.cnt.s4_val) { /* S4 request */
                qapi_event_send_suspend_disk(&error_abort);
                qemu_system_shutdown_request();
            }
            break;
        }
    }
}
{%endhighlight%}

以上就是如果用户从VM内部执行poweroff后，qemu中ACPi相关的操作。

刚才提到了其实kernel中ACPI是可选的。所以如果在grub时候，在启动项中加入acpi=off，系统
就不会初始化ACPI，那么会出现什么问题呢？
由于发行版中并没有编译APM的驱动，所以当关闭acpi后，执行poweroff，kvm的VM会打印一个
System Halt.然后卡住了。这也许是一个目前虚拟机中的bug。

下来看看xen的电源管理。
在xen中也使用qemu进行设备模拟，不可避免的也会初始化ACPI设备。
所以上面流程中提到的关于ACPI设备的操作在xen下都会出现。
但是在xen下，由于并不依赖与这个信号退出qemu。所以这个ACPI设备其实没有实际作用。

其实xen的电源管理是自己实现的。
这里我们主要看xl中电源管理的实现。
在xl执行create 命令后，并不会退出，
而是fork到后台后开始执行一个poll。
用来监听消息事件。

其中libxl_evenable_domain_death 注册了回调函数，来监听虚拟机退出事件。

{%highlight c%}
 ret = libxl_evenable_domain_death(ctx, domid, 0, &deathw);
    if (ret) goto out;


 while (1) {
        libxl_event *event;
        ret = domain_wait_event(&event);
        if (ret) goto out;

        switch (event->type) {

        case LIBXL_EVENT_TYPE_DOMAIN_SHUTDOWN:
            LOG("Domain %d has shut down, reason code %d 0x%x", domid,
                event->u.domain_shutdown.shutdown_reason,
                event->u.domain_shutdown.shutdown_reason);
            switch (handle_domain_death(&domid, event,
                                        (uint8_t **)&config_data, &config_len,
                                        &d_config)) {
            case 2:
                if (!preserve_domain(&domid, event, &d_config)) {
                    /* If we fail then exit leaving the old domain in place. */
                    ret = -1;
                    goto out;
                }


{%endhighlight%}

可以看到 监听的key是@releaseDomain ，这个key是一个特殊key，只要有vm退出都会触发信息。
所以需要自己来判断是否是关心的VM触发了退出事件。

{%highlight c%}

int libxl_evenable_domain_death(libxl_ctx *ctx, uint32_t domid,
                libxl_ev_user user, libxl_evgen_domain_death **evgen_out) {
    GC_INIT(ctx);
    libxl_evgen_domain_death *evg, *evg_search;
    int rc;

    CTX_LOCK;

    evg = malloc(sizeof(*evg));  if (!evg) { rc = ERROR_NOMEM; goto out; }
    memset(evg, 0, sizeof(*evg));
    evg->domid = domid;
    evg->user = user;

    LIBXL_TAILQ_INSERT_SORTED(&ctx->death_list, entry, evg, evg_search, ,
                              evg->domid > evg_search->domid);

    if (!libxl__ev_xswatch_isregistered(&ctx->death_watch)) {
        rc = libxl__ev_xswatch_register(gc, &ctx->death_watch,
                        domain_death_xswatch_callback, "@releaseDomain");
        if (rc) { libxl__evdisable_domain_death(gc, evg); goto out; }
    }

    *evgen_out = evg;
    rc = 0;

 out:
    CTX_UNLOCK;
    GC_FREE;
    return rc;
};

{%endhighlight%}

因为xen下的VM并没有依赖ACPI来实现电源管理。所以无论ACPI是否生效，虚拟机都会成功
关闭，当然，如果没有开启ACPI，在VM关机时，还是会看到System Halt. 
