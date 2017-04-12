---
layout: post
title:  "libvirt使用sanlock做磁盘锁揭秘"
date:   2017-4-12 21:13:31
categories:
  - linux
tags:
  - linux
---

前言
---
sanlock是redhat基于Paxos协议实现的分布式锁管理器。用来在不同主机之间保证进程能够单活。在libvirt中可以配置使用sanlock来保证虚拟机单活，即我们常说的防脑裂功能。网上大都有sanlock的原理介绍，可以参考[sanlock 原理介绍及应用]本文就不在详述了。这里从sanlock的使用方式以及libvirt的使用方式来揭秘libvirt如何sanlock的。


sanlock对象
---
sanlock其实就是分布式锁服务，那么既然是分布式锁就需要有：
* 集群 
  由一群可用的主机组成一个集群，他们是选举的参与者。在sanlock中每一个应用就需要一个集群的。因此，对于每一个应用都会有一个叫做locksapce的空间，它负责让所有能够启动该应用的主机进行注册，并维护他们的心跳。
  
* 锁
  在分布式场景下用来让唯一的个体获得所有权。在snalock中如果要启动应用，就需要拿到这样的锁。这个锁在sanlock里面就叫做resource。只要获得他的所有权就相当于获得了启动应用的权限。

sanlock对象的格式

上面提到的重要对象，在sanlock中有着严格的格式规定。
{%highlight bash%}
LOCKSPACE = <lockspace_name>:<host_id>:<path>:<offset>
  <lockspace_name>	name of lockspace
  <host_id>		local host identifier in lockspace
  <path>		disk to storage reserved for leases
  <offset>		offset on path (bytes)

RESOURCE = <lockspace_name>:<resource_name>:<path>:<offset>[:<lver>]
  <lockspace_name>	name of lockspace
  <resource_name>	name of resource
  <path>		disk to storage reserved for leases
  <offset>		offset on path (bytes)
  <lver>        optional leader version or SH for shared lease


{% endhighlight %}

举例说明比较明显：
对于一个应用来说，我们需要定义一个LOCKSPACE。比如test应用。
那么我们可以定义我们的LOCKSPACE
* lockspace_name ：test
* host_id ：1（sanlock的host_id支持1-2000之间，0一般用来初始化时使用）
* path : /mnt/nfs/test_lockspace （用来存储LOCKSPACE的空间,这个空间需要是在共享存储上）
* offset ： 0（用来表示这个文件的起始位置）

那么合起来这个LOCKSPACE就是
{%highlight bash%}
test:1:/mnt/nfs/test_lockspace：0
{% endhighlight %}

RESOURCE的格式和LOCKSPACE类似，这里不再举例。

sanlock的使用
---

我们使用nfs来进行sanlock使用的介绍。
首先找两个主机，在两个主机上都将nfs挂载到/mnt/nfs下，为了描述清楚，我们分别定义主机为
hostA : host_id 是1， hostB : host_id 是2。我们的lockspace名为LS，resource名为RS



* 初始化 
 在任一主机上进行初始化的动作，我们选择hostA。
 创建LOCKSPACE文件和RESOURCE文件，并给予这2个文件sanlock权限。并进行初始化
{%highlight bash%}
dd if=/dev/zero bs=1048576 count=1 of=/mnt/nfs/test_lockspace
dd if=/dev/zero bs=1048576 count=1 of=/mnt/nfs/test_resource
chown sanlock:sanlock /mnt/nfs/test_lockspace
chown sanlock:sanlock /mnt/nfs/test_resource
sanlock client init -s LS:0:/mnt/nfs/test_lockspace：0
sanlock client init -r LS:RS:/mnt/nfs/test_resource：0
{% endhighlight %}


* 接下来要分别在两个主机将主机加入到sanlock中
{%highlight bash%}
#hostA
   sanlock client add_lockspace -s  LS:1:/mnt/nfs/test_lockspace：0
#hostB
   sanlock client add_lockspace -s  LS:2:/mnt/nfs/test_lockspace：0   
{% endhighlight %}

* 可以使用锁服务了
比如在HostA 主机上对于我们需要加锁的进程和resource文件关联，并进行注册
{%highlight bash%}
  sanlock client acquire -r LS:RS:/mnt/nfs/test_resource:0 -p (pid)
{% endhighlight %}
那么在HostB 再次执行该命令 就会被告知锁文件被占用，指定的进程讲被kill掉。

这就是snalock进行加锁的流程。

Libvirt中的配置
---
接下来就是今天的正题，libvirt是如何使用sanlock的。

libvirt对于sanlock的使用，实现了两种方式。

1. 自动加锁，即对每一个虚拟机的磁盘都自动进行加锁。这个方式只需要在配置文件中进行配置，就自动生效了。
2. 通过xml中的配置来主动控制加锁。

接下来就对两种方式都加以介绍。

* 对于自动加锁的方式，只需要添加配置文件 /etc/libvirt/qemu-sanlock.conf。
{%highlight bash%}
auto_disk_leases = 1（1为开启自动加锁，0为关闭自动加锁）
host_id = 1
disk_lease_dir = "/var/lib/libvirt/sanlock"
{% endhighlight %}
这样libvirt就会在启动时候，自动初始化__LIBVIRT__DISKS__这个LOCKSPACE文件，并加入的sanlock中，在创建虚拟机时候，也会自动创建RESOURCE文件，对这个虚拟机的磁盘加锁。
* 对于手动加锁，首先需要自己去配置LOCKSPACE文件和RESOURCE文件，然后在xml中进行配置。

首先在配置文件中配置auto_disk_leases=0，关闭该功能，并重启libvirt。

因为libvirt中默认的LOCKSPACE文件所在目录是/var/lib/libvirt/sanlock，所以我们在该目录下进行LOCKSPACE文件和RESOURCE文件的创建。

{%highlight bash%}
dd if=/dev/zero bs=1048576 count=1 of=/var/lib/libvirt/sanlock/LS
dd if=/dev/zero bs=1048576 count=1 of=/var/lib/libvirt/sanlock/RS
chown sanlock:sanlock /var/lib/libvirt/sanlock/LS
chown sanlock:sanlock /var/lib/libvirt/sanlock/RS
sanlock client init -s LS:0:/var/lib/libvirt/sanlock/LS：0
sanlock client init -r LS:RS:/var/lib/libvirt/sanlock/RS：0
{% endhighlight %}

接下来将所在主机的LOCKSPACE文件加入到sanlock中

{%highlight bash%}
#hostA
   sanlock client add_lockspace -s  LS:1:/var/lib/libvirt/sanlock/LS：0
#hostB
   sanlock client add_lockspace -s  LS:2:/var/lib/libvirt/sanlock/LS：0   
{% endhighlight %}

接下来在libvirt中配置xml
{%highlight bash%}
 <lease>
        <lockspace>LS</lockspace>
        <key>RS</key>
        <target path='/var/lib/libvirt/sanlock/RS' offset='0'/>
     </lease>
{% endhighlight %}

lockspace中填的内容，既是LOCKSPACE的名字，也是LOCKSPACE文件的名字。

key是RESOURCE的名字

path是RESOURCE文件的路径。

offset是RESOURCE在磁盘上的偏移。因为我们使用单独文件，偏移就是0

这样也可以使用sanlock的功能。





[sanlock 原理介绍及应用]:https://www.ibm.com/developerworks/cn/linux/1404_zhouzs_sanlock/

