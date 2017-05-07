---
layout: post
title:  "LinuxKit初探"
date:   2017-4-25 21:13:31
categories:
  - linux
tags:
  - linux
  - docker
---

前言
---
Docker 在DockerCon 2017大会上发布了一个自己的操作系统，宣称LinuxKit，安全，精简，强移植性。经过研究后发现LinuxKit就是kernel+busybox实现的一个微缩linux系统，其中直接安装了containerd和runc服务。其他服务全部都使用容器启动。一句话概括，LinuxKit就是希望在操作系统层面除了containerd之外其他服务都跑在容器中，这是一个几乎全部服务都在容器中的操作系统。

LinuxKit的使用
---
先介绍下LinuxKit是如何使用的。

项目地址：https://github.com/linuxkit/linuxkit

LinuxKit工程提供了工具来构建我们自己的操作系统。

* moby 通过yml文件的描述，构建自定义的操作系统。
* linuxkit 用来帮忙在不同平台启动镜像的工具。

首先安装moby和linuxkit工具和下载linuxkit代码

{%highlight bash%}
go get -u github.com/linuxkit/linuxkit/src/cmd/moby
go get -u github.com/linuxkit/linuxkit/src/cmd/linuxkit
git clone https://github.com/linuxkit/linuxkit
{% endhighlight %}

然后开始编译定制化的linux操作系统，我们使用LinuxKit提供的linuxkit.yml来进行编译

{%highlight bash%}
cd linuxkit
moby build linuxkit.yml
{% endhighlight %}

编译完成后，我们可以看到新出现的文件，这就是通过linuxkit.yml编译出来的内核和ramdisk

{%highlight bash%}
	linuxkit-bzImage  编译出的kernel
	linuxkit-cmdline  启动时候的grub需要带的参数。
	linuxkit-efi.iso  EFI启动镜像
	linuxkit-initrd.img 一个ramdisk
	linuxkit.iso  BIOS启动镜像
{% endhighlight %}

接下来可以通过linuxkit工具来启动这个系统。

{%highlight bash%}
linuxkit run linuxkit
{% endhighlight %}

linuxkit 会根据系统自动选择合适的hypervisor来启动镜像。
由于我在linux上，所以启动的时候使用了kvm来启动虚拟机。

{%highlight bash%}
Welcome to LinuxKit 

                        ##. 
                  ## ## ## == 
               ## ## ## ## ## === 
           / "" "" "" "" "" "" "" "" "" \ ___ / = = = 
      ~~~ { 
​~~~~~~~~~~~~~~~~~ / === --~~~ 
           \ ______ o __ / 
             \ \ __ / 
              \ ____ \ _______ / 

{% endhighlight %}

启动时候可以看到我们可爱的小鲸鱼。这时候我们订制的专门跑容器的系统就完成了。

这里看一下系统启动了那些程序

{%highlight bash%}
/ # pstree
/ # pstree
init-+-containerd-+-containerd-shim---nginx---nginx
     |-containers---ctr
     |-sh---pstree
     `-sh
{% endhighlight %}
使用默认的linuxkit.yml 其实就是启动了一个nginx服务。而且这个系统中的init程序直接启动了containerd。以后所有的服务都是通过容器启动的。

linuxkit分析
---

通过刚才的使用介绍可以看到，其实lilnuxkit是给用户提供了一种通过yml来订制自己的微缩操作系统的方式。那么我们还是先从这个linuxkit.yml文件来看。

{%highlight bash%}
#指定了需用的内核版本，以及启动参数
kernel:
  image: "linuxkit/kernel:4.9.x"
  cmdline: "console=ttyS0 console=tty0 page_poison=1"
#指定init程序
init:
  - linuxkit/init:42fe8cb1508b3afed39eb89821906e3cc7a70551
  - linuxkit/runc:b0fb122e10dbb7e4e45115177a61a3f8d68c19a9
  - linuxkit/containerd:60e2486a74c665ba4df57e561729aec20758daed
  - linuxkit/ca-certificates:eabc5a6e59f05aa91529d80e9a595b85b046f935
#指定了启动时候的会运行的系统容器，执行一次就会停止的。
onboot:
  - name: sysctl
    image: "linuxkit/sysctl:2cf2f9d5b4d314ba1bfc22b2fe931924af666d8c"
    net: host
    pid: host
    ipc: host
    capabilities:
     - CAP_SYS_ADMIN
    readonly: true
  - name: binfmt
    image: "linuxkit/binfmt:8881283ac627be1542811bd25c85e7782aebc692"
    binds:
     - /proc/sys/fs/binfmt_misc:/binfmt_misc
    readonly: true
  - name: dhcpcd
    image: "linuxkit/dhcpcd:48e249ebef6a521eed886b3bce032db69fbb4afa"
    binds:
     - /var:/var
     - /tmp/etc:/etc
    capabilities:
     - CAP_NET_ADMIN
     - CAP_NET_BIND_SERVICE
     - CAP_NET_RAW
    net: host
    command: ["/sbin/dhcpcd", "--nobackground", "-f", "/dhcpcd.conf", "-1"]
#在镜像生命周期中都要启动的系统服务
services:
  - name: rngd
    image: "linuxkit/rngd:c42fd499690b2cb6e4e6cb99e41dfafca1cf5b14"
    capabilities:
     - CAP_SYS_ADMIN
    oomScoreAdj: -800
    readonly: true
  - name: nginx
    image: "nginx:alpine"
    capabilities:
     - CAP_NET_BIND_SERVICE
     - CAP_CHOWN
     - CAP_SETUID
     - CAP_SETGID
     - CAP_DAC_OVERRIDE
    net: host
#镜像中包含的文件及内容
files:
  - path: etc/docker/daemon.json
    contents: '{"debug": true}'
trust:
  image:
    - linuxkit/kernel

#输出的文件，kernel，initrd，bios-iso和efi-iso
outputs:
  - format: kernel+initrd
  - format: iso-bios
  - format: iso-efi

{% endhighlight %}
文件比较简单，通过阅读文件可以看到，通过指定不同的容器镜像，最终可以帮助我们生成想要的
镜像。

生成了镜像，我们自然要研究一下这个linuxkit操作系统和现在发行版linux有哪些区别。

在使用linuxkit启动时，可以看到qemu相关的参数。

{%highlight bash%}
/usr/bin/docker-current run -i --rm -v /home/git/linuxkit:/tmp -w /tmp --device /dev/kvm linuxkit/qemu:17f052263d63c8a2b641ad91c589edcbb8a18c82 qemu-system-x86_64 -device virtio-rng-pci -smp 1 -m 1024 -enable-kvm -machine q35,accel=kvm:tcg -kernel linuxkit-bzImage -initrd linuxkit-initrd.img -append console=ttyS0 console=tty0 page_poison=1 -nographic
{% endhighlight %}
观察后可以发现，在启动虚拟机时候，并没有挂载硬盘，而是使用了指定kernel和ramdisk的方式来进行的。

这样看来linuxkit其实就是一个使用kernel和ramdisk直接启动的操作系统。 而ramdisk就包含了服务的容器镜像，在init程序启动之后，就通过containerd来拉起服务。linuxkit的init程序是用shell编写，因此我们这里也打开来看看。
{%highlight bash%}
#!/bin/sh

setup_console() {
	tty=${1%,*}
	speed=${1#*,}
	inittab="$2"
	securetty="$3"
	line=
	term="linux"
	[ "$speed" = "$1" ] && speed=115200

	case "$tty" in
	ttyS*|ttyAMA*|ttyUSB*|ttyMFD*)
		line="-L"
		term="vt100"
		;;
	tty?)
		line=""
		speed="38400"
		term=""
		;;
	esac
	# skip consoles already in inittab
	grep -q "^$tty:" "$inittab" && return

	echo "$tty::once:cat /etc/issue" >> "$inittab"
	echo "$tty::respawn:/sbin/getty -n -l /bin/sh $line $speed $tty $term" >> "$inittab"
	if ! grep -q -w "$tty" "$securetty"; then
		echo "$tty" >> "$securetty"
	fi
}

#在/mnt 下挂载了tmpfs
/bin/mount -t tmpfs tmpfs /mnt

#将根目录下的所有内容都copy到了/mnt下
/bin/cp -a / /mnt 2>/dev/null

/bin/mount -t proc -o noexec,nosuid,nodev proc /proc
for opt in $(cat /proc/cmdline); do
	case "$opt" in
	console=*)
		setup_console ${opt#console=} /mnt/etc/inittab /mnt/etc/securetty;;
	esac
done
#进行操作系统根目录的切换 /mnt 这里的/sbin/init 其实是软连接到busybox
exec /bin/busybox switch_root /mnt /sbin/init
{% endhighlight %}

文件内容依然简单可以看到，init程序给/mnt挂载了tmpfs系统，这是一个内存文件系统。然后将ramdisk中的所有内容都copy到了/mnt下，然后切换根目录到 /mnt。然后交给busybox继续启动。
busybox接着就开始启动containerd，并通过runc启动系统应用容器等。

linuxkit使用场景
---
从官方的描述来看，Linuxkit为每种容器提供了一个基于容器的方法，生成轻量级Linux子系统。当为特定硬件或者拥有特定功能定制系统时非常有用。每个LinuxKit子系统都有其自己的Linux核心，而每个系统守护进程或者系统服务都是一个容器。这些Linux子系统一旦被打包成ISO镜像，就可以用来启动物理机或者虚拟环境。Docker以提供的服务方式维护这些子系统。

这里其实可以看到Linuxkit解决了docker一直以来的问题容器和host共享内核，以至于有些应用不能够很好的在docker上使用的问题。
linuxkit具有的特点：
* Linuxkit作为一个操作系统，本身可以在物理机或者虚拟机上启动。
* linuxkit是只读文件系统，并且相当轻量级只有60M左右，可以被应用在无盘启动系统中。
* linuxkit的理念同docker理念一脉相承。不同的镜像对应不同的应用，灵活，轻量，无状态。

可以预见到，可以直接在裸机上通过pxe来启动对应的linuxkit镜像，或者使用虚拟机快速启动linuxkit镜像，用户已经无需再预先搭建docker平台来启动容器。可以说linuxkit给容器化带来了更广阔的前景。


