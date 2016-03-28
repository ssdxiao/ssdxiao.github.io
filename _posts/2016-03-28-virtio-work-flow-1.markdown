---

layout: post

title:  virtio的工作流程-qemu中virtio-backend初始化(1)
date:   2015-12-10 14:10:00
categories: linux
tags: 
  - linux
  - virtio
---

在kvm下，virtio-blk的实现特性有virtio-blk，dataplane，vhost-blk。其中前2个已经进入了社区。而vhost-blk仅仅提了patch，并没有通过。
在开发中需要对比这3种特性下，block的io性能，所以找到了vhost-blk的patch，但是patch直接合入后不能成功使用。所以在解决该问题时候，
刚好又看了一次virtio-blk的初始化，以及io路径，这里为了避免忘记总结一下。这里分2次来讲，先将一下virtio-pci设备的正常通信流程。
再来说vhost-blk。

一 总览

1.简单来说，virtio的设备模拟都是基于virtio-pci设备来进行的。在guest中可以使用lspci看到设备名称。
这里可以看到使用的驱动都是virtio-pci。

{%highlight bash}
00:03.0 Ethernet controller: Red Hat, Inc Virtio network device
	Subsystem: Red Hat, Inc Device 0001
	Physical Slot: 3
	Flags: bus master, fast devsel, latency 0, IRQ 11
	I/O ports at c040 [size=32]
	Memory at febd1000 (32-bit, non-prefetchable) [size=4K]
	Expansion ROM at feb80000 [disabled] [size=256K]
	Capabilities: [40] MSI-X: Enable+ Count=3 Masked-
	Kernel driver in use: virtio-pci

00:04.0 SCSI storage controller: Red Hat, Inc Virtio block device
	Subsystem: Red Hat, Inc Device 0002
	Physical Slot: 4
	Flags: bus master, fast devsel, latency 0, IRQ 10
	I/O ports at c000 [size=64]
	Memory at febd2000 (32-bit, non-prefetchable) [size=4K]
	Capabilities: [40] MSI-X: Enable+ Count=2 Masked-
	Kernel driver in use: virtio-pci

{%endhighlight}

2.在kvm下基于virtio-pci最终实现了virtiobus。这里让所有的设备最终挂在virtiobus上。这一细节要放到对virtio-pci前端驱动分析时候来讲。

{%highlight bash}
/sys/bus/virtio/devices
[root@localhost devices]# ls
virtio0  virtio1
{%endhighlight}

3.通信流程。通过virtio-pci的中断事件来达到virtio前后端中断的通信。然后通过共享内存的读写，达到数据的传递。所以可以看到半虚驱动的实现基本点就是 1 中断。2共享内存。


二 virtio-block后端的初始化。
在qemu启动时，会对相应设备进行初始化。这里，可以看下hw/virtio/virtio-pci.c
我们需要初始化virtio-block， 在该文件中
这里就从virtio-pci初始化开始讲起。
virtio-pci，也是一个pci设备。所以初始化时候会调用到TYPE_PCI_DEVICE的初始化

{%highlight c}

 static const TypeInfo virtio_pci_info = {
      .name          = TYPE_VIRTIO_PCI,
      .parent        = TYPE_PCI_DEVICE,                                                                                                                              
      .instance_size = sizeof(VirtIOPCIProxy),
      .class_init    = virtio_pci_class_init,
      .class_size    = sizeof(VirtioPCIClass),
      .abstract      = true,
  };

{%endhighlight}

在pci设备初始化时候，调用pci_device_class_init函数。
{%highlight c}
 static const TypeInfo pci_device_type_info = {
      .name = TYPE_PCI_DEVICE,
      .parent = TYPE_DEVICE,
      .instance_size = sizeof(PCIDevice),
      .abstract = true,
      .class_size = sizeof(PCIDeviceClass),
      .class_init = pci_device_class_init,
  };
{%endhighlight}

pci_device_class_init函数定义了初始化pci_qdev_init 。

{%highlight c}
  static void pci_device_class_init(ObjectClass *klass, void *data)
  {
      DeviceClass *k = DEVICE_CLASS(klass);
      k->init = pci_qdev_init;
      k->unplug = pci_unplug_device;
      k->exit = pci_unregister_device;
      k->bus_type = TYPE_PCI_BUS;
      k->props = pci_props;
  }
{%endhighlight}

这样，在virio-pci初始化时候，就会触发pci_qdev_init 函数，从而把pci设备加入到qdev tree中，同时也在I/O设备空间的内存中进行了注册。


{%highlight c}
static int virtio_pci_init(PCIDevice *pci_dev)
{
    VirtIOPCIProxy *dev = VIRTIO_PCI(pci_dev);
    VirtioPCIClass *k = VIRTIO_PCI_GET_CLASS(pci_dev);
    virtio_pci_bus_new(&dev->bus, sizeof(dev->bus), dev);
    //这里调用了pci_qdev_init
    if (k->init != NULL) {
        return k->init(dev);
    }
    return 0;
}
{%endhighlight}

pci_qdev_init主要添加了pci rom。以及调用do_pci_register_device函数对pci设备在内存进行了注册。

{%highlight c}
 static int pci_qdev_init(DeviceState *qdev)
  {
      PCIDevice *pci_dev = (PCIDevice *)qdev;
      PCIDeviceClass *pc = PCI_DEVICE_GET_CLASS(pci_dev);
      PCIBus *bus;
      int rc;
      bool is_default_rom;
  
      /* initialize cap_present for pci_is_express() and pci_config_size() */
      if (pc->is_express) {
          pci_dev->cap_present |= QEMU_PCI_CAP_EXPRESS;
      }
  
      bus = FROM_QBUS(PCIBus, qdev_get_parent_bus(qdev));
      pci_dev = do_pci_register_device(pci_dev, bus,
                                       object_get_typename(OBJECT(qdev)),
                                       pci_dev->devfn);
      if (pci_dev == NULL)
          return -1;
      if (qdev->hotplugged && pc->no_hotplug) {
          qerror_report(QERR_DEVICE_NO_HOTPLUG, object_get_typename(OBJECT(pci_dev)));
          do_pci_unregister_device(pci_dev);
          return -1;
      }
      if (pc->init) {
          rc = pc->init(pci_dev);
          if (rc != 0) {
              do_pci_unregister_device(pci_dev);
              return rc;
          }
      }
  
      /* rom loading */
      is_default_rom = false;
      if (pci_dev->romfile == NULL && pc->romfile != NULL) {
          pci_dev->romfile = g_strdup(pc->romfile);
          is_default_rom = true;
      }
      pci_add_option_rom(pci_dev, is_default_rom);
  
      if (bus->hotplug) {
          /* Let buses differentiate between hotplug and when device is
           * enabled during qemu machine creation. */
          rc = bus->hotplug(bus->hotplug_qdev, pci_dev,
                            qdev->hotplugged ? PCI_HOTPLUG_ENABLED:
                            PCI_COLDPLUG_ENABLED);
          if (rc != 0) {
              int r = pci_unregister_device(&pci_dev->qdev);
              assert(!r);
              return rc;
          }
      }
      return 0;
  }
{%endhighlight}

do_pci_register_device中初始化了pci相关config配置。

{%highlight c}
 static PCIDevice *do_pci_register_device(PCIDevice *pci_dev, PCIBus *bus,
                                           const char *name, int devfn)
  {
      PCIDeviceClass *pc = PCI_DEVICE_GET_CLASS(pci_dev);
      PCIConfigReadFunc *config_read = pc->config_read;
      PCIConfigWriteFunc *config_write = pc->config_write;
  
      if (devfn < 0) {
          for(devfn = bus->devfn_min ; devfn < ARRAY_SIZE(bus->devices);
              devfn += PCI_FUNC_MAX) {
              if (!bus->devices[devfn])
                  goto found;
          }
          error_report("PCI: no slot/function available for %s, all in use", name);
          return NULL;
      found: ;
      } else if (bus->devices[devfn]) {
          error_report("PCI: slot %d function %d not available for %s, in use by %s",
                       PCI_SLOT(devfn), PCI_FUNC(devfn), name, bus->devices[devfn]->name);
          return NULL;
      }
      pci_dev->bus = bus;
      if (bus->dma_context_fn) {
          pci_dev->dma = bus->dma_context_fn(bus, bus->dma_context_opaque, devfn);
      } else {
          /* FIXME: Make dma_context_fn use MemoryRegions instead, so this path is
           * taken unconditionally */
          /* FIXME: inherit memory region from bus creator */
          memory_region_init_alias(&pci_dev->bus_master_enable_region, "bus master",
                                   get_system_memory(), 0,
                                   memory_region_size(get_system_memory()));
          memory_region_set_enabled(&pci_dev->bus_master_enable_region, false);
          address_space_init(&pci_dev->bus_master_as, &pci_dev->bus_master_enable_region);
          pci_dev->dma = g_new(DMAContext, 1);
          dma_context_init(pci_dev->dma, &pci_dev->bus_master_as, NULL, NULL, NULL);
      }
      pci_dev->devfn = devfn;
      pstrcpy(pci_dev->name, sizeof(pci_dev->name), name);
      pci_dev->irq_state = 0;
      pci_config_alloc(pci_dev);
  
      pci_config_set_vendor_id(pci_dev->config, pc->vendor_id);
      pci_config_set_device_id(pci_dev->config, pc->device_id);
      pci_config_set_revision(pci_dev->config, pc->revision);
      pci_config_set_class(pci_dev->config, pc->class_id);
  
      if (!pc->is_bridge) {
          if (pc->subsystem_vendor_id || pc->subsystem_id) {
              pci_set_word(pci_dev->config + PCI_SUBSYSTEM_VENDOR_ID,
                           pc->subsystem_vendor_id);
              pci_set_word(pci_dev->config + PCI_SUBSYSTEM_ID,
                           pc->subsystem_id);
          } else {
              pci_set_default_subsystem_id(pci_dev);
          }
      } else {
          /* subsystem_vendor_id/subsystem_id are only for header type 0 */
          assert(!pc->subsystem_vendor_id);
          assert(!pc->subsystem_id);
      }
      pci_init_cmask(pci_dev);
      pci_init_wmask(pci_dev);
      pci_init_w1cmask(pci_dev);
      if (pc->is_bridge) {
          pci_init_mask_bridge(pci_dev);
      }
      if (pci_init_multifunction(bus, pci_dev)) {
          pci_config_free(pci_dev);
          return NULL;
      }
  
      if (!config_read)
          config_read = pci_default_read_config;
      if (!config_write)
          config_write = pci_default_write_config;
      pci_dev->config_read = config_read;
      pci_dev->config_write = config_write;
      bus->devices[devfn] = pci_dev;
      pci_dev->irq = qemu_allocate_irqs(pci_set_irq, pci_dev, PCI_NUM_PINS);
      pci_dev->version_id = 2; /* Current pci device vmstate version */
      return pci_dev;
  }
{%endhighlight}

完成后，调用virtio_blk_pci_init初始化

{%highlight c}
 static int virtio_blk_device_init(VirtIODevice *vdev)
  {
      DeviceState *qdev = DEVICE(vdev);
      VirtIOBlock *s = VIRTIO_BLK(vdev);
      VirtIOBlkConf *blk = &(s->blk);
      static int virtio_blk_id;
  
      if (!blk->conf.bs) {
          error_report("drive property not set");
          return -1;
      }
      if (!bdrv_is_inserted(blk->conf.bs)) {
          error_report("Device needs media, but drive is empty");
          return -1;
      }
  
      blkconf_serial(&blk->conf, &blk->serial);
      if (blkconf_geometry(&blk->conf, NULL, 65535, 255, 255) < 0) {
          return -1;
      }
  
      virtio_init(vdev, "virtio-blk", VIRTIO_ID_BLOCK,
                  sizeof(struct virtio_blk_config));
  
      s->vhost_started = false;
      s->vhost_stopping = false;
      s->bs = blk->conf.bs;
      if (s->bs->use_vhost)
          s->vhost_blk = vhost_blk_init();
      s->conf = &blk->conf;
      memcpy(&(s->blk), blk, sizeof(struct VirtIOBlkConf));
      s->rq = NULL;
      s->sector_mask = (s->conf->logical_block_size / BDRV_SECTOR_SIZE) - 1;
  
      s->vq = virtio_add_queue(vdev, 128, virtio_blk_handle_output);
  #ifdef CONFIG_VIRTIO_BLK_DATA_PLANE
      if (!virtio_blk_data_plane_create(vdev, blk, &s->dataplane)) {
          virtio_cleanup(vdev);
          return -1;
      }
  #endif
  
      s->change = qemu_add_vm_change_state_handler(virtio_blk_dma_restart_cb, s);
      register_savevm(qdev, "virtio-blk", virtio_blk_id++, 2,
                      virtio_blk_save, virtio_blk_load, s);
      bdrv_set_dev_ops(s->bs, &virtio_block_ops, s);
      bdrv_set_buffer_alignment(s->bs, s->conf->logical_block_size);
  
      bdrv_iostatus_enable(s->bs);
  
      add_boot_device_path(s->conf->bootindex, qdev, "/disk@0,0");                                                                                                   
      return 0;
  }
{%endhighlight}


调用virtio_pci_device_plugged。添加到virtio_bus上。

{%highlight c}
  static void virtio_pci_device_plugged(DeviceState *d)
  {
      VirtIOPCIProxy *proxy = VIRTIO_PCI(d);
      VirtioBusState *bus = &proxy->bus;
      uint8_t *config;
      uint32_t size;
  
      config = proxy->pci_dev.config;
      if (proxy->class_code) {
          pci_config_set_class(config, proxy->class_code);
      }
      pci_set_word(config + PCI_SUBSYSTEM_VENDOR_ID,
                   pci_get_word(config + PCI_VENDOR_ID));
      pci_set_word(config + PCI_SUBSYSTEM_ID, virtio_bus_get_vdev_id(bus));
      config[PCI_INTERRUPT_PIN] = 1;
  
      if (proxy->nvectors &&
          msix_init_exclusive_bar(&proxy->pci_dev, proxy->nvectors, 1)) {
          error_report("unable to init msix vectors to %" PRIu32,
                       proxy->nvectors);
          proxy->nvectors = 0;
      }
  
      proxy->pci_dev.config_write = virtio_write_config;
  
      size = VIRTIO_PCI_REGION_SIZE(&proxy->pci_dev)
           + virtio_bus_get_vdev_config_len(bus);
      if (size & (size - 1)) {
          size = 1 << qemu_fls(size);
      }
  
      memory_region_init_io(&proxy->bar, OBJECT(proxy), &virtio_pci_config_ops,
                            proxy, "virtio-pci", size);
      pci_register_bar(&proxy->pci_dev, 0, PCI_BASE_ADDRESS_SPACE_IO,
                       &proxy->bar);
  
      if (!kvm_has_many_ioeventfds()) {
          proxy->flags &= ~VIRTIO_PCI_FLAG_USE_IOEVENTFD;
      }
  
      virtio_add_feature(&proxy->host_features, VIRTIO_F_NOTIFY_ON_EMPTY);
      virtio_add_feature(&proxy->host_features, VIRTIO_F_BAD_FEATURE);
      proxy->host_features = virtio_bus_get_vdev_features(bus,
                                                        proxy->host_features);
  }
 {%endhighlight} 

这里初始化完成。
在main函数中 所有设备初始化完成，会调用一次qemu_devices_reset对应pci设备。这个动作复位了pci设备的状态等信息。


{%highlight c}
  static void virtio_pci_reset(DeviceState *qdev)
  {
      VirtIOPCIProxy *proxy = VIRTIO_PCI(qdev);
      VirtioBusState *bus = VIRTIO_BUS(&proxy->bus);
      virtio_pci_stop_ioeventfd(proxy);
      virtio_bus_reset(bus);
      msix_unuse_all_vectors(&proxy->pci_dev);                                                                     
      proxy->flags &= ~VIRTIO_PCI_FLAG_BUS_MASTER_BUG;
  }
 {%endhighlight} 

在VM状态变化时候，会调用virtio_vmstate_change函数，来改变virtio的状态。
这里需要在VM处于runing状态，和virtio驱动都初始化以后，才开始改变状态。

{%highlight c}
  static void virtio_vmstate_change(void *opaque, int running, RunState state)
  {
      VirtIODevice *vdev = opaque;
      BusState *qbus = qdev_get_parent_bus(DEVICE(vdev));
      VirtioBusClass *k = VIRTIO_BUS_GET_CLASS(qbus);
      bool backend_run = running && (vdev->status & VIRTIO_CONFIG_S_DRIVER_OK);
      vdev->vm_running = running;
  
      if (backend_run) {
          virtio_set_status(vdev, vdev->status);
      }
  
      if (k->vmstate_change) {
          k->vmstate_change(qbus->parent, backend_run);
      }
  
      if (!backend_run) {
          virtio_set_status(vdev, vdev->status);
      }
  }
 {%endhighlight} 

现在就等待virtio-pci的驱动开始生效时候，和virtio-back来进行信息协商。






