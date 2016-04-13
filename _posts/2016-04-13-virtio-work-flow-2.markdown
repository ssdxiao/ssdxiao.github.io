---

layout: post

title:  virtio的工作流程-kernel中virtio-pci初始化(2)
date:   2015-04-13 14:10:00
categories: linux
tags: 
  - linux
  - virtio
---

接上节，这次主要讲virtio-pci设备初始化，以及建立相应的通信通道。
一个virtio-pci设备有2个区域，一个是data区域，一个是config区域
使用 info mtree 可以看到这个结果。

data区域是pci-conf-data
config区域是virtio-pci

其中data区域主要记录了pci设备的 设备号，厂商等等信息。
而config区域就是用来前后协商，以及irq中断通知。

首先，检测到pci设备，会加载virtio-pci驱动，主要就是初始化相关寄存器，
这是一个pci设备。

然后向virtio-bus注册设备，开始实体化我们的virtio设备。


{%highlight c%}
virtio-pci的驱动加载是在寄存器初始化好以后，开始进行的。

  static int virtio_pci_probe(struct pci_dev *pci_dev,
                  const struct pci_device_id *id)
  {
      struct virtio_pci_device *vp_dev;
      int err;
  
      /* We only own devices >= 0x1000 and <= 0x103f: leave the rest. */
      if (pci_dev->device < 0x1000 || pci_dev->device > 0x103f)
          return -ENODEV;
  
      if (pci_dev->revision != VIRTIO_PCI_ABI_VERSION) {
          printk(KERN_ERR "virtio_pci: expected ABI version %d, got %d\n",
                 VIRTIO_PCI_ABI_VERSION, pci_dev->revision);
          return -ENODEV;
      }
  
      /* allocate our structure and fill it out */
      vp_dev = kzalloc(sizeof(struct virtio_pci_device), GFP_KERNEL);
      if (vp_dev == NULL)
          return -ENOMEM;
  
      vp_dev->vdev.dev.parent = &pci_dev->dev;
      vp_dev->vdev.dev.release = virtio_pci_release_dev;
      vp_dev->vdev.config = &virtio_pci_config_ops;
      vp_dev->pci_dev = pci_dev;
      INIT_LIST_HEAD(&vp_dev->virtqueues);
      spin_lock_init(&vp_dev->lock);
  
      /* Disable MSI/MSIX to bring device to a known good state. */
      //操作pci msi相关寄存器
      pci_msi_off(pci_dev);
  
      /* enable the device */
      err = pci_enable_device(pci_dev);
      if (err)
          goto out;
      //标记virtio-pci使用的区域
      err = pci_request_regions(pci_dev, "virtio-pci");
      if (err)
          goto out_enable_device;
  
      vp_dev->ioaddr = pci_iomap(pci_dev, 0, 0);
      if (vp_dev->ioaddr == NULL) {
          err = -ENOMEM;
          goto out_req_regions;
      }
  
      pci_set_drvdata(pci_dev, vp_dev);
      //设置总线DMA模式
      pci_set_master(pci_dev);
  
      /* we use the subsystem vendor/device id as the virtio vendor/device
       * id.  this allows us to use the same PCI vendor/device id for all
       * virtio devices and to identify the particular virtio driver by
       * the subsystem ids */
      vp_dev->vdev.id.vendor = pci_dev->subsystem_vendor;
      vp_dev->vdev.id.device = pci_dev->subsystem_device;
  
      /* finally register the virtio device */
      err = register_virtio_device(&vp_dev->vdev);
      if (err)
          goto out_set_drvdata;
  
      return 0;
  
  out_set_drvdata:
      pci_set_drvdata(pci_dev, NULL);
      pci_iounmap(pci_dev, vp_dev->ioaddr);
  out_req_regions:
      pci_release_regions(pci_dev);
  out_enable_device:
      pci_disable_device(pci_dev);
  out:
      kfree(vp_dev);
      return err;
  }


{%highlight end%}

register_virtio_device主要是注册设备，并且对设备进行状态设置，
VIRTIO_CONFIG_S_ACKNOWLEDGE  表示是发现了设备，这里注册设备到virtio_bus,这里会触发vritio_bus的probe函数。

{%highlight c%}
  int register_virtio_device(struct virtio_device *dev)
  {
      int err;
  
      dev->dev.bus = &virtio_bus;
  
      /* Assign a unique device index and hence name. */
      err = ida_simple_get(&virtio_index_ida, 0, 0, GFP_KERNEL);
      if (err < 0)
          goto out;
      
      //注册了一个virtio设备。
      dev->index = err;
      dev_set_name(&dev->dev, "virtio%u", dev->index);
  
      /* We always start by resetting the device, in case a previous
       * driver messed it up.  This also tests that code path a little. */
      dev->config->reset(dev);
  
      /* Acknowledge that we've seen the device. */
      add_status(dev, VIRTIO_CONFIG_S_ACKNOWLEDGE);
  
      INIT_LIST_HEAD(&dev->vqs);
  
      /* device_register() causes the bus infrastructure to look for a
       * matching driver. */
      err = device_register(&dev->dev);
  out:
      if (err)
          add_status(dev, VIRTIO_CONFIG_S_FAILED);
      return err;
  }
{%highlight end%}

会调用的virtio设备的virtio_dev_probe初始化设备。然后调用具体的vritio设备驱动来初始化。
可以看到virtio_bus主要控制了设备的连接状态，对于真正需要virito前后端建立连接的工作还是
调用了具体的virtio_blk的probe函数继续进行。
VIRTIO_CONFIG_S_DRIVER 发现了设备
VIRTIO_CONFIG_S_DRIVER_OK 驱动初始化完成。

{%highlight c%}
  static int virtio_dev_probe(struct device *_d)
  {
      int err, i;
      struct virtio_device *dev = dev_to_virtio(_d);
      struct virtio_driver *drv = drv_to_virtio(dev->dev.driver);
      u32 device_features;
  
      /* We have a driver! */
      add_status(dev, VIRTIO_CONFIG_S_DRIVER);
  
      /* Figure out what features the device supports. */
      device_features = dev->config->get_features(dev);
  
      /* Features supported by both device and driver into dev->features. */
      memset(dev->features, 0, sizeof(dev->features));
      for (i = 0; i < drv->feature_table_size; i++) {
          unsigned int f = drv->feature_table[i];
          BUG_ON(f >= 32);
          if (device_features & (1 << f))
              set_bit(f, dev->features);
      }
  
      /* Transport features always preserved to pass to finalize_features. */
      for (i = VIRTIO_TRANSPORT_F_START; i < VIRTIO_TRANSPORT_F_END; i++)
          if (device_features & (1 << i))
              set_bit(i, dev->features);
  
      dev->config->finalize_features(dev);
  
      err = drv->probe(dev);
      if (err)
          add_status(dev, VIRTIO_CONFIG_S_FAILED);
      else {
          add_status(dev, VIRTIO_CONFIG_S_DRIVER_OK);
          if (drv->scan)
              drv->scan(dev);
      }
  
      return err;
  }
{%highlight end%}
virtblk_probe 函数主要做了这些，读取配置，初始化ring环。

{%highlight c%}
  static int virtblk_probe(struct virtio_device *vdev)
  {
      struct virtio_blk *vblk;
      struct request_queue *q;
      int err, index;
      int pool_size;
  
      u64 cap;
      u32 v, blk_size, sg_elems, opt_io_size;
      u16 min_io_size;
      u8 physical_block_exp, alignment_offset;
  
      err = ida_simple_get(&vd_index_ida, 0, minor_to_index(1 << MINORBITS),
                   GFP_KERNEL);
      if (err < 0)
          goto out;
      index = err;
  
       //这里读取后端配置。
      /* We need to know how many segments before we allocate. */
      err = virtio_config_val(vdev, VIRTIO_BLK_F_SEG_MAX,
                  offsetof(struct virtio_blk_config, seg_max),
                  &sg_elems);
  
      /* We need at least one SG element, whatever they say. */
      if (err || !sg_elems)
          sg_elems = 1;
  
      /* We need an extra sg elements at head and tail. */
      sg_elems += 2;
      vdev->priv = vblk = kmalloc(sizeof(*vblk) +
                      sizeof(vblk->sg[0]) * sg_elems, GFP_KERNEL);
      if (!vblk) {
          err = -ENOMEM;
          goto out_free_index;
      }
  
      init_waitqueue_head(&vblk->queue_wait);
      vblk->vdev = vdev;
      vblk->sg_elems = sg_elems;
      sg_init_table(vblk->sg, vblk->sg_elems);
      mutex_init(&vblk->config_lock);
  
      INIT_WORK(&vblk->config_work, virtblk_config_changed_work);
      vblk->config_enable = true;
  
      //这里开启了msix中断，并建立了vring
      err = init_vq(vblk);
      if (err)
          goto out_free_vblk;
  
      pool_size = sizeof(struct virtblk_req);
      if (use_bio)
          pool_size += sizeof(struct scatterlist) * sg_elems;
      vblk->pool = mempool_create_kmalloc_pool(1, pool_size);
      if (!vblk->pool) {
          err = -ENOMEM;
          goto out_free_vq;
      }
  
       // 这里开始初始化bdi设备。
      /* FIXME: How many partitions?  How long is a piece of string? */
      vblk->disk = alloc_disk(1 << PART_BITS);
      if (!vblk->disk) {
          err = -ENOMEM;
          goto out_mempool;
      }
  
      q = vblk->disk->queue = blk_init_queue(virtblk_request, NULL);
      if (!q) {
          err = -ENOMEM;
          goto out_put_disk;
      }
  
      if (use_bio)
          blk_queue_make_request(q, virtblk_make_request);
      q->queuedata = vblk;
  
      virtblk_name_format("vd", index, vblk->disk->disk_name, DISK_NAME_LEN);
  
      vblk->disk->major = major;
      vblk->disk->first_minor = index_to_minor(index);
      vblk->disk->private_data = vblk;
      vblk->disk->fops = &virtblk_fops;
      vblk->disk->driverfs_dev = &vdev->dev;
      vblk->index = index;
  
      /* configure queue flush support */
      virtblk_update_cache_mode(vdev);
  
      /* If disk is read-only in the host, the guest should obey */
      if (virtio_has_feature(vdev, VIRTIO_BLK_F_RO))
          set_disk_ro(vblk->disk, 1);
  
      /* Host must always specify the capacity. */
      vdev->config->get(vdev, offsetof(struct virtio_blk_config, capacity),
                &cap, sizeof(cap));
  
      /* If capacity is too big, truncate with warning. */
      if ((sector_t)cap != cap) {
          dev_warn(&vdev->dev, "Capacity %llu too large: truncating\n",
               (unsigned long long)cap);
          cap = (sector_t)-1;
      }
      set_capacity(vblk->disk, cap);
  
      /* We can handle whatever the host told us to handle. */
      blk_queue_max_segments(q, vblk->sg_elems-2);
  
      /* No need to bounce any requests */
      blk_queue_bounce_limit(q, BLK_BOUNCE_ANY);
  
      /* No real sector limit. */
      blk_queue_max_hw_sectors(q, -1U);
  
      /* Host can optionally specify maximum segment size and number of
       * segments. */
      err = virtio_config_val(vdev, VIRTIO_BLK_F_SIZE_MAX,
                  offsetof(struct virtio_blk_config, size_max),
                  &v);
      if (!err)
          blk_queue_max_segment_size(q, v);
      else
          blk_queue_max_segment_size(q, -1U);
  
      /* Host can optionally specify the block size of the device */
      err = virtio_config_val(vdev, VIRTIO_BLK_F_BLK_SIZE,
                  offsetof(struct virtio_blk_config, blk_size),
                  &blk_size);
      if (!err)
          blk_queue_logical_block_size(q, blk_size);
      else
          blk_size = queue_logical_block_size(q);
  
      /* Use topology information if available */
      err = virtio_config_val(vdev, VIRTIO_BLK_F_TOPOLOGY,
              offsetof(struct virtio_blk_config, physical_block_exp),
              &physical_block_exp);
      if (!err && physical_block_exp)
          blk_queue_physical_block_size(q,
                  blk_size * (1 << physical_block_exp));
  
      err = virtio_config_val(vdev, VIRTIO_BLK_F_TOPOLOGY,
              offsetof(struct virtio_blk_config, alignment_offset),
              &alignment_offset);
      if (!err && alignment_offset)
          blk_queue_alignment_offset(q, blk_size * alignment_offset);
  
      err = virtio_config_val(vdev, VIRTIO_BLK_F_TOPOLOGY,
              offsetof(struct virtio_blk_config, min_io_size),
              &min_io_size);
      if (!err && min_io_size)
          blk_queue_io_min(q, blk_size * min_io_size);
  
      err = virtio_config_val(vdev, VIRTIO_BLK_F_TOPOLOGY,
              offsetof(struct virtio_blk_config, opt_io_size),
              &opt_io_size);
      if (!err && opt_io_size)
          blk_queue_io_opt(q, blk_size * opt_io_size);
  
      add_disk(vblk->disk);
      err = device_create_file(disk_to_dev(vblk->disk), &dev_attr_serial);
      if (err)
          goto out_del_disk;
  
      if (virtio_has_feature(vdev, VIRTIO_BLK_F_CONFIG_WCE))
          err = device_create_file(disk_to_dev(vblk->disk),
                       &dev_attr_cache_type_rw);
      else
          err = device_create_file(disk_to_dev(vblk->disk),
                       &dev_attr_cache_type_ro);
      if (err)
          goto out_del_disk;
      return 0;
  
  out_del_disk:
      del_gendisk(vblk->disk);
      blk_cleanup_queue(vblk->disk->queue);
  out_put_disk:
      put_disk(vblk->disk);
  out_mempool:
      mempool_destroy(vblk->pool);
  out_free_vq:
      vdev->config->del_vqs(vdev);
  out_free_vblk:
      kfree(vblk);
  out_free_index:
      ida_simple_remove(&vd_index_ida, index);
  out:
      return err;
  }
{%highlight end%}

vq_init->virtio_find_single_vq，调用了virtio-pci驱动中的vp_find_vqs->vp_try_to_find_vqs。
这里，根据情况 初始化msix或者是irq中断。并注册了callback。
{%highlight c%}
  static int vp_try_to_find_vqs(struct virtio_device *vdev, unsigned nvqs,
                    struct virtqueue *vqs[],
                    vq_callback_t *callbacks[],
                    const char *names[],
                    bool use_msix,
                    bool per_vq_vectors)
  {
      struct virtio_pci_device *vp_dev = to_vp_device(vdev);
      u16 msix_vec;
      int i, err, nvectors, allocated_vectors;
  
      if (!use_msix) {
           //使用的是irq中断
          /* Old style: one normal interrupt for change and all vqs. */
          err = vp_request_intx(vdev);
          if (err)
              goto error_request;
      } else {
          if (per_vq_vectors) {
              /* Best option: one for change interrupt, one per vq. */
              nvectors = 1;
              for (i = 0; i < nvqs; ++i)
                  if (callbacks[i])
                      ++nvectors;
          } else {
              /* Second best: one for change, shared for all vqs. */
              nvectors = 2;
          }
          //使用msix中断，这里申请了2个msix 中断，一个绑定回调config_change用来更新config
          另外一个绑定了vring_interupt 用来调用callback，来处理io
  
          err = vp_request_msix_vectors(vdev, nvectors, per_vq_vectors);
          if (err)
              goto error_request;
      }
  
      vp_dev->per_vq_vectors = per_vq_vectors;
      allocated_vectors = vp_dev->msix_used_vectors;
      for (i = 0; i < nvqs; ++i) {
          if (!names[i]) {
              vqs[i] = NULL;
              continue;
          } else if (!callbacks[i] || !vp_dev->msix_enabled)
              msix_vec = VIRTIO_MSI_NO_VECTOR;
          else if (vp_dev->per_vq_vectors)
              msix_vec = allocated_vectors++;
          else
              msix_vec = VP_MSIX_VQ_VECTOR;
           //通知后端msix中断，以及初始化vring
          vqs[i] = setup_vq(vdev, i, callbacks[i], names[i], msix_vec);
          if (IS_ERR(vqs[i])) {
              err = PTR_ERR(vqs[i]);
              goto error_find;
          }
  
          if (!vp_dev->per_vq_vectors || msix_vec == VIRTIO_MSI_NO_VECTOR)
              continue;
  
          /* allocate per-vq irq if available and necessary */
          snprintf(vp_dev->msix_names[msix_vec],
               sizeof *vp_dev->msix_names,
               "%s-%s",
               dev_name(&vp_dev->vdev.dev), names[i]);
          err = request_irq(vp_dev->msix_entries[msix_vec].vector,
                    vring_interrupt, 0,
                    vp_dev->msix_names[msix_vec],
                    vqs[i]);
          if (err) {
              vp_del_vq(vqs[i]);
              goto error_find;
          }
      }
      return 0;
  
  error_find:
      vp_del_vqs(vdev);
  
  error_request:
      return err;
  }
           false, false);
  }
{%highlight end%}

setup_vq用来初始化vring，并且通知后端msix中断。

{%highlight c%}
  static struct virtqueue *setup_vq(struct virtio_device *vdev, unsigned index,
                    void (*callback)(struct virtqueue *vq),
                    const char *name,
                    u16 msix_vec)
  {
      struct virtio_pci_device *vp_dev = to_vp_device(vdev);
      struct virtio_pci_vq_info *info;
      struct virtqueue *vq;
      unsigned long flags, size;
      u16 num;
      int err;
  
      /* Select the queue we're interested in */
      iowrite16(index, vp_dev->ioaddr + VIRTIO_PCI_QUEUE_SEL);
  
      /* Check if queue is either not available or already active. */
      //读取后端设置的queue的长度，这里一般是128  
      num = ioread16(vp_dev->ioaddr + VIRTIO_PCI_QUEUE_NUM);
      if (!num || ioread32(vp_dev->ioaddr + VIRTIO_PCI_QUEUE_PFN))
          return ERR_PTR(-ENOENT);
  
      /* allocate and fill out our structure the represents an active
       * queue */
      info = kmalloc(sizeof(struct virtio_pci_vq_info), GFP_KERNEL);
      if (!info)
          return ERR_PTR(-ENOMEM);
  
      info->num = num;
      info->msix_vector = msix_vec;
  
      //计算vring的长度。 是 num * vring_desc +num *vring_used_elem 以及其他信息。
      size = PAGE_ALIGN(vring_size(num, VIRTIO_PCI_VRING_ALIGN));
      //分配一段连续的页
      info->queue = alloc_pages_exact(size, GFP_KERNEL|__GFP_ZERO);
      if (info->queue == NULL) {
          err = -ENOMEM;
          goto out_info;
      }
  
      /* activate the queue */
      //通知后端pfn号，这样后端可以通过pfn来直接访问guest的内存。
      iowrite32(virt_to_phys(info->queue) >> VIRTIO_PCI_QUEUE_ADDR_SHIFT,
            vp_dev->ioaddr + VIRTIO_PCI_QUEUE_PFN);
  
      /* create the vring */
      vq = vring_new_virtqueue(index, info->num, VIRTIO_PCI_VRING_ALIGN, vdev,
                   true, info->queue, vp_notify, callback, name);
      if (!vq) {
          err = -ENOMEM;
          goto out_activate_queue;
      }
  
      vq->priv = info;
      info->vq = vq;
  
      if (msix_vec != VIRTIO_MSI_NO_VECTOR) {
          iowrite16(msix_vec, vp_dev->ioaddr + VIRTIO_MSI_QUEUE_VECTOR);
          msix_vec = ioread16(vp_dev->ioaddr + VIRTIO_MSI_QUEUE_VECTOR);
          if (msix_vec == VIRTIO_MSI_NO_VECTOR) {
              err = -EBUSY;
              goto out_assign;
          }
      }
  
      if (callback) {
          spin_lock_irqsave(&vp_dev->lock, flags);
          list_add(&info->node, &vp_dev->virtqueues);
          spin_unlock_irqrestore(&vp_dev->lock, flags);
      } else {
          INIT_LIST_HEAD(&info->node);
      }
  
      return vq;
  
  out_assign:
      vring_del_virtqueue(vq);
  out_activate_queue:
      iowrite32(0, vp_dev->ioaddr + VIRTIO_PCI_QUEUE_PFN);
      free_pages_exact(info->queue, size);
  out_info:
      kfree(info);
      return ERR_PTR(err);
  }
{%highlight end%}








