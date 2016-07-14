---

layout: post

title:  virtio-gpu介绍
date:   2016-7-14 14:10:00
categories: 虚拟化
tags: 
  - libvirt
  - 虚拟化 
  - qemu
---

背景：
---
显卡的提升在虚拟化场景下一直是一个难以解决的问题。目前qemu中提供的显卡有2种
一种是cirrus显卡，一种是vga显卡。这两种显卡都是通过qemu来进行模拟的，也仅仅达到了能够让虚拟机使用的功能。
而对于游戏需要的3D加速能力等，还不能很好的模拟。

显卡本身需要强大的计算能力，这里qemu纯用cpu来模拟gpu的功能明显有些力不从心。

新的功能virtio-gpu的出现给虚拟化的显卡功能带来了一些新的气息。下面就简要的介绍一下virtio-gpu这个功能。


virtio-gpu介绍
---
同所有的virtio设备一样，virtio-gpu也是有这前端显卡和后端显卡组成。
virtio-gpu的前端显卡在kernel 4.2 进入主干，只具有2D功能。在4.4合入了3D功能。
所以要想体验这一功能需要使用kernel 4.4以后版本。


virtio-gpu相关代码主要在kernel的drivers/gpu/drm/virtio目录下。
这里简单就介绍下drm。详细的可以参看[DRM介绍]
DRM可以直接访问DRM clients的硬件。DRM驱动用来处理DMA，内存管理，资源锁以及安全硬件访问。

这样可以看到通过DRM来管理相关的内存信息，这样就可以通过virtio-gpu来将内存信息传递给后端。

接下来就是要看qemu中的后端实现。

virtio-gpu的支持在qemu 2.5中开始支持。
并准备在qemu2.6中对spice显卡进行支持。
作为一个典型的virtio设备，同样需要实现virtio的接口。在qemu的hw/display中包含了virtio-gpu后端的代码。

在这里qemu使用了Virgil 3D 这个工程来进行显卡模拟 参考[Virgil 3D]

Virgil 3D 目的是使用Host的3D加速技术来实现一个虚拟的3D GPU来供给虚拟机使用。


说了这么多当然要自己体验一把virtio-gpu。
在[virtio-gpu]中介绍了如何使用virtio-gpu的过程。
下面也把我测试的过程写在这里。

virtio-gpu测试过程
---

本人使用的硬件环境：


CPU： Intel(R) Core(TM) i5-5200U CPU @ 2.20GHz (2194MHz) x2


GPU： Intel(R) HD Graphics 5500 20.19.15.4331 (1024MB) x1

使用个人PC搭建centos7.2.

安装 
{%highlight bash%}
spice-server-0.13.1-20160524.b122.g70f04bd.x86_64
spice-server-devel-0.13.1-20160524.b122.g70f04bd.x86_64
spice-glib-0.31-20160321.b000.g0a1f9bf.x86_64
spice-vdagent-0.14.0-10.el7.x86_64
spice-gtk3-0.31-20160321.b000.g0a1f9bf.x86_64
virglrenderer-0.5.0-1.20160411git61846f92f.el7.centos.x86_64
virglrenderer-devel-0.5.0-1.20160411git61846f92f.el7.centos.x86_64
{% endhighlight %}

当然现在还需要2.6版本的[qemu]

编译时的配置
{%highlight bash%}
 ./configure --enable-debug --enable-gtk --target-list=x86_64-softmmu
{% endhighlight %}
 
 需要确认 如下支持
 {%highlight bash%}
 virgl support     yes
 spice support     yes (0.12.11/0.13.1)
{% endhighlight %}

这样我们的hypervisor就准备好了。

由于kernel 4.4以上才支持virtio-gpu的驱动，所以选取了
[Fedora-Server-dvd-x86_64-24-1.2] 来作为guest OS 进行测试。

需要安装KDE桌面环境来测试3D效果。


使用virtio-gpu的启动命令，这里使用spice开启gl=on，目前的spice实现只支持本地访问。
还不能以TCP形式提供给远程。

{%highlight bash%}
x86_64-softmmu/qemu-system-x86_64 -m 4000 --enable-kvm -vga virtio -cpu host -smp 2 -drive file=/home/test.img,if=virtio -usb -usbdevice tablet -spice gl=on,unix,addr=/home/spice.sock,disable-ticketing
{% endhighlight %}

连接到虚拟机
{%highlight bash%}
remote-viewer spice+unix:///home/spice.sock 
{% endhighlight %}

这样可以看到带有3D加速的画面了。

如何确认 3D加速已经开启了呢？
使用glxinfo 来查看 direct 是否是Yes
{%highlight bash%}
glxinfo|grep direct
{% endhighlight %}

virtio-gpu到底性能如何？当然还要进行一番测试。
这里选用Unigine_Valley-1.0 显卡性能测试工具。
软件下载

{%highlight bash%}
wget http://ftp.nluug.nl/pub/games/PC/guru3d/benchmark/Unigine_Valley-1.0-[Guru3D.com].run
{% endhighlight %}

安装后，直接可以开始进行测试了。


|         | Host           | Guest  |
| ------------- |:-------------:| -----:|
| FPS    | 10.2 | 1.1 |
| Score      | 425      |   44 |
| Min FPS | 5.4     |    1.0 |
|Max FPS| 20.5 | 1.2|



可以看到virtio-gpu的性能损耗也还是很高的。

国外友人也同样进行了测试[benchmark]。


在这里有一个小发现，在使用-vag qxl 模拟显卡时，查询3D加速，也是Yes的，但是
无法运行3D测试软件。会报有函数找不到。猜测模拟显卡实现了部分3D加速指令。

另外就是不使用spice，直接使用qemu的display也可以开启virtio-gpu的加速功能。

{%highlight bash%}
x86_64-softmmu/qemu-system-x86_64 -m 4000 --enable-kvm -display gtk,gl=on -vga virtio -cpu host -smp 2 -drive file=/home/test.img,if=virtio -usb -usbdevice tablet
{% endhighlight %}


小结：
---
virtio-gpu的优势很明显，这里就不多说了，如果要拿来作为产品，目前还有一些限制：

- 目前能够提供3D加速支持，该能力和本地的显卡性能有关，性能损耗还是略高。
- 目前spice的client端仅仅支持本地使用，还需要提供远程访问能力。
- virtio-gpu驱动只提供给了linux，而桌面系统还是windows居多，需要windows版的驱动。




[DRM介绍]: http://blog.csdn.net/kickxxx/article/details/19188711

[Virgil 3D]: https://virgil3d.github.io/

[virtio-gpu]:https://www.kraxel.org/blog/tag/virtio-gpu/

[benchmark]:http://www.gearsongallium.com/?p=3244

[qemu]: https://github.com/qemu/qemu.git 

[Fedora-Server-dvd-x86_64-24-1.2]: http://mirrors.aliyun.com/fedora/releases/24/Server/x86_64/iso/Fedora-Server-dvd-x86_64-24-1.2.iso


