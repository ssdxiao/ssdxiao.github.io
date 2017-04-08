---
layout: post
title:  "解决sysctl配置不能持久化"
date:   2017-3-20 21:13:31
categories:
  - linux
tags:
  - linux
---

前言
---
linux下提供了sysctl进行系统参数的配置，这些配置可以及时生效，也可以记录在配置文件/etc/sysctl.conf中，使得重启主机后也可以生效。
但是最近在配置nf_conntrack_max参数时候，发生了重启配置不生效的问题。


问题详述
---
最近在解决问题时，需要对nf_conntrack_max参数进行修改。
先修改配置文件，

在/etc/sysctl.conf中添加如下内容
{%highlight bash%}
net.netfilter.nf_conntrack_max=655360
{% endhighlight %}

然后执行
{%highlight bash%}
sysctl -p
{% endhighlight %}

这样就很简单的修改了配置项信息，但是在重启以后，发现该配置不能生效。


问题定位
---

* 解决方向一


首先求助google。发现网上有人说过类似的事情。
见[Can not automatically apply sysctl.conf when boot up]


这里提到了停止tuned服务。

tuned服务是用来动态调整系统参数的，那么如果我设置的参数被动态调整了所以不生效，也是很有可能。

因此我先查看了tuned服务的配置文件。在/usr/lib/tuned/目录下记录了tuned服务进行管理的配置项，搜索后发现
并没有我设置的nf_conntrack_max参数相关配置。所以感觉并没有找到问题关键。停止tuned服务后

重启主机的确发现配置并没有被修改。

* 解决方向二


只能靠自己怀疑去定位了，我使用的是centos 7.2 系统。该系统的配置管理使用的是systemd-sysctl管理

那么这个问题就会怀疑是不是由于systemctl的机制引入的。
查看/usr/lib/systemd/system/systemd-sysctl.service文件内容

{%highlight bash%}
[Unit]
Description=Apply Kernel Variables
Documentation=man:systemd-sysctl.service(8) man:sysctl.d(5)
DefaultDependencies=no
Conflicts=shutdown.target
After=systemd-readahead-collect.service systemd-readahead-replay.service
After=systemd-modules-load.service
Before=sysinit.target shutdown.target
ConditionPathIsReadWrite=/proc/sys/

[Service]
Type=oneshot
RemainAfterExit=yes
ExecStart=/usr/lib/systemd/systemd-sysctl
{% endhighlight %}

可以看到服务调用的是systemd-sysctl命令进行配置，所以我也就手动重启该服务。
{%highlight bash%}
systemctl restart systemd-sysctl
{% endhighlight %}

结果参数生效了。那么其实验证了如果该服务正常启动后，参数应该是可以生效配置的。那么问题会出在哪里呢。

* 解决方向三


查看/var/log/message 发现，nf_conntrack.ko 加载时候会有打印日志。
{%highlight bash%}
Apr  7 15:33:35 localhost kernel: nf_conntrack version 0.5.0 (16384 buckets, 65536 max)
{% endhighlight %}
这个日志出现在启动快完成的时候，那么会不会是由于systemd并发启动使得启动顺序引入的问题呢。

基于这个怀疑，需要控制启动顺序。在systemd-sysctl.service服务中可以看到，该服务是在systemd-modules-load.service服务之后启动的。
systemd-modules-load.service服务就是用来提前加载需要进入内核的服务。

因此首先配置启动加载nf_conntrack.ko

{%highlight bash%}
echo nf_conntrack > /usr/lib/modules-load.d/net.conf
{% endhighlight %}

重启之后，参数设置成功。至此问题解决了。


总结
---
使用sysctl.conf进行系统参数设置是一件很简单的事情。但是这一次也在其中学到了不少东西。
* 系统参数如果是module中的，那么在sysctl相关服务启动前，module必须加载。
* 就算sysctl相关服务已经进行了修改，还是有可能被其他服务例如tuned等再次修改掉。


[Can not automatically apply sysctl.conf when boot up]:https://www.centos.org/forums/viewtopic.php?t=50704

