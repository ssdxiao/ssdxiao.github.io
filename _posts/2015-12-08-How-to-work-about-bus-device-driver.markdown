---

layout: post

title:  linux的总线，设备，驱动是怎么工作的
date:   2015-12-08 14:10:00
categories: linux
tags: 
  - linux
---

linux的设备是按照总线，设备，驱动联系起来的。简单来说就是任何设备都是挂在在一个总线上，而设备也有相应的驱动，才能正确的运行。

在理解这些概念前，首先要介绍一下linux中的对象模型。

kobject:每一个对象都必须是一个kobject。所有的设备时间通过uevent来驱动。比如kobj的增加，删除等。

kset:就是kobject的集合。

举个例子，在系统中bus是一个根节点，bus中包含各种各样的总线，比如xen，pci，scsi等等。xen，pci，scsi就属于在kset中。

linux为了方便管理这些设备。这些设备会加入到sysfs中。
sysfs是一个特殊的文件系统，专门用来呈现设备模型。

今天，就从代码级别来看一下他们具体是如何被组织起来的。

从xen-bus前端总线的驱动来看总线是怎么工作的。

{%highlight c%}
  static struct xen_bus_type xenbus_frontend = {
      .root = "device",
      .levels = 2,        /* device/type/<id> */
      .get_bus_id = frontend_bus_id,
      .probe = xenbus_probe_frontend,
      .otherend_changed = backend_changed,
      .bus = {
          .name       = "xen",
          .match      = xenbus_match,
          .uevent     = xenbus_uevent_frontend,
          .probe      = xenbus_frontend_dev_probe,
          .remove     = xenbus_dev_remove,
          .shutdown   = xenbus_dev_shutdown,
          .dev_groups = xenbus_dev_groups,
  
          .pm     = &xenbus_pm_ops,
      },
  };


{%endhighlight%}
可以看到，在系统初始化时就会被执行bus_register来注册xenbus。
{%highlight c%}
  static int __init xenbus_probe_frontend_init(void)
  {
      static struct notifier_block xenstore_notifier = {
          .notifier_call = frontend_probe_and_watch
      };
      int err;
  
      DPRINTK("");
  
      /* Register ourselves with the kernel bus subsystem */
      err = bus_register(&xenbus_frontend.bus);
      if (err)
          return err;
  
      register_xenstore_notifier(&xenstore_notifier);
  
      if (xen_store_domain_type == XS_LOCAL) {
          xenbus_frontend_wq = create_workqueue("xenbus_frontend");
          if (!xenbus_frontend_wq)
              pr_warn("create xenbus frontend workqueue failed, S3 resume is likely to fail\n");
      }
  
      return 0;
  }
  subsys_initcall(xenbus_probe_frontend_init);
  
{%endhighlight%}

打开bus_register来看可以看到整个bus的处理过程。注册一个bus到系统总线中。

{%highlight c%}

int bus_register(struct bus_type *bus)
{
    int retval;
    //包含了bus的私有信息。
    struct subsys_private *priv;
    struct lock_class_key *key = &bus->lock_key;
    
   
    priv = kzalloc(sizeof(struct subsys_private), GFP_KERNEL);
    if (!priv)
        return -ENOMEM;

    priv->bus = bus;
    bus->p = priv;

    BLOCKING_INIT_NOTIFIER_HEAD(&priv->bus_notifier);
    
    //这里初始化了bus本身的kobj模型。
    retval = kobject_set_name(&priv->subsys.kobj, "%s", bus->name);
    if (retval)
        goto out;
    //这里的bus_kset就是bus的一个集合。这个集合在bus总线驱动加载时候进行了初始化。

    priv->subsys.kobj.kset = bus_kset;
    priv->subsys.kobj.ktype = &bus_ktype;
    priv->drivers_autoprobe = 1;

    //将总线注册到kset中，注意，如果kset中的kobj没有parent那么这里就会注册自己到parent。
    //由于kset已经初始化过了parent为bus，所以这里注册的总线类型就会成为bus的子节点。
    retval = kset_register(&priv->subsys);
    if (retval)
        goto out;
    //生成sys中相应的文件。
    retval = bus_create_file(bus, &bus_attr_uevent);
    if (retval)
        goto bus_uevent_fail;

    priv->devices_kset = kset_create_and_add("devices", NULL,
                         &priv->subsys.kobj);
    if (!priv->devices_kset) {
        retval = -ENOMEM;
        goto bus_devices_fail;
    }

    priv->drivers_kset = kset_create_and_add("drivers", NULL,
                         &priv->subsys.kobj);
    if (!priv->drivers_kset) {
        retval = -ENOMEM;
        goto bus_drivers_fail;
    }

    INIT_LIST_HEAD(&priv->interfaces);
    __mutex_init(&priv->mutex, "subsys mutex", key);
    klist_init(&priv->klist_devices, klist_devices_get, klist_devices_put);
    klist_init(&priv->klist_drivers, NULL, NULL);

    retval = add_probe_files(bus);
    if (retval)
        goto bus_probe_files_fail;

    retval = bus_add_groups(bus, bus->bus_groups);
    if (retval)
        goto bus_groups_fail;

    pr_debug("bus: '%s': registered\n", bus->name);
    return 0;

{%endhighlight%}

{%highlight c%}
int __init buses_init(void)
{
    //这里就是初始化bus_kset并且初始化了kset，把bus这个kobj注册到自己身上
    bus_kset = kset_create_and_add("bus", &bus_uevent_ops, NULL);
    if (!bus_kset)
        return -ENOMEM;


    return 0;
}
{%endhighlight%}   

上面介绍了一个总线对象的初始化过程。创建了kobj，并加入到kset中。然后再sysfs上注册相应信息。

对于设备的发现，也是通过总线扫描来做到的。xenbus是通过
xenbus_thread来对xenstore进行监控，当检测到有信息变化时候，执行了xenbus_probe_node然后调用

device_register来进行设备的注册。

{%highlight c%}
int device_register(struct device *dev)
{   
    //初始化dev
    device_initialize(dev);
    //类似bus_register建立kobj和kset父子间关系，然后再sys下生成相应文件
    return device_add(dev); 
}
{%endhighlight%}   

{%highlight c%}
void device_initialize(struct device *dev)
{
     //添加到设备集合中
    dev->kobj.kset = devices_kset;
    //初始化kobj
    kobject_init(&dev->kobj, &device_ktype);
    INIT_LIST_HEAD(&dev->dma_pools);
    mutex_init(&dev->mutex);
    lockdep_set_novalidate_class(&dev->mutex);
    spin_lock_init(&dev->devres_lock);
    INIT_LIST_HEAD(&dev->devres_head);
    device_pm_init(dev);
    set_dev_node(dev, -1);
}
{%endhighlight%}   

device_add函数并没有过多需要说明的，也就是类似bus_register进行了sysfs相关文件建立，确定了父子kobj之间的关系，加入到device的集合等操作。在device_add中加入了bus_probe_device函数，该函数是用来把device加入到相应的总线中。

{%highlight c%}
void bus_probe_device(struct device *dev)
{
    struct bus_type *bus = dev->bus;
    struct subsys_interface *sif;

    if (!bus)
        return;

    if (bus->p->drivers_autoprobe)
        //就是调用了__device_attach
        device_initial_probe(dev);

    mutex_lock(&bus->p->mutex);
    list_for_each_entry(sif, &bus->p->interfaces, node)
        if (sif->add_dev)
            sif->add_dev(dev, sif);
    mutex_unlock(&bus->p->mutex);
}

{%endhighlight%} 



{%highlight c%}
static int __device_attach(struct device *dev, bool allow_async)
{
    int ret = 0;

    device_lock(dev);
    if (dev->driver) {
        if (klist_node_attached(&dev->p->knode_driver)) {
            ret = 1;
            goto out_unlock;
        }
        ret = device_bind_driver(dev);
        if (ret == 0)
            ret = 1;
        else {
            dev->driver = NULL;
            ret = 0;
        }
    } else {
        struct device_attach_data data = {
            .dev = dev,
            .check_async = allow_async,
            .want_async = false,
        };
        //扫描总线，然后匹配合适的驱动进行初始化
        ret = bus_for_each_drv(dev->bus, NULL, &data,
                    __device_attach_driver);
        if (!ret && allow_async && data.have_async) {
            /*
             * If we could not find appropriate driver
             * synchronously and we are allowed to do
             * async probes and there are drivers that
             * want to probe asynchronously, we'll
             * try them.
             */
            dev_dbg(dev, "scheduling asynchronous probe\n");
            get_device(dev);
            async_schedule(__device_attach_async_helper, dev);
        } else {
            pm_request_idle(dev);
        }
    }
out_unlock:
    device_unlock(dev);
    return ret;
}

{%endhighlight%} 


{%highlight c%}
static int __device_attach_driver(struct device_driver *drv, void *_data)
{
    struct device_attach_data *data = _data;
    struct device *dev = data->dev;
    bool async_allowed;

    /*
     * Check if device has already been claimed. This may
     * happen with driver loading, device discovery/registration,
     * and deferred probe processing happens all at once with
     * multiple threads.
     */
    if (dev->driver)
        return -EBUSY;
    //调用到驱动注册时的match函数
    if (!driver_match_device(drv, dev))
        return 0;
    //调用驱动注册时的probe函数
    async_allowed = driver_allows_async_probing(drv);

    if (async_allowed)
        data->have_async = true;

    if (data->check_async && async_allowed != data->want_async)
        return 0;

    return driver_probe_device(drv, dev);
}

{%endhighlight%} 

上面描述了一个设备的插入的过程。可以看到从总线扫描到一个设备，然后建立设备和驱动之间的关联，从而将总线
，设备，驱动联系在了一起。



