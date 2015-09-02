---

layout: post

title:  "Hyper初步分析"

date:   2015-09-02 09:43:31

categories: 
  - 虚拟化
tags:
  - 虚拟化
---

Docker的快速发展为云场景带来了一丝新意，Docker提出的镜像打包方式迎合了Devops的开发方式，
对云服务的快速开发提供了有力的帮助。Docker使用的是container技术，而云平台上传统应用在目前更多的是直接跑在vm中。
对于container和vm之间的区别在[why Hyper]中有了很好的描述，当然也包括了今天的主角Hyper自身的优势。
这里就引出了一个新东西[Hyper]。

什么是Hyper呢？

简而言之，hyper具有了vm级别的隔离性以及docker对镜像封装方式的优点。好了闲话不多开始分析下hyper如何实现的。

Hyper本身就是一个虚拟机，我们传统的vm会在虚拟磁盘上来安装操作系统等其他服务,而Hyper并没有直接使用虚拟磁盘，而是使用了虚拟化中的文件系统虚拟机化，[VirtFS]特性。这样可以让虚拟机通过VirtFS的驱动，直接读取Host上的文件系统。这样就完成了虚拟机直接对Host文件系统的读写。Hyper正是利用了这一特性完成了VM对Docker镜像的读写，从而在此之上实现了VM和Docker的完美结合。


上面描述的原理基本上已经对Hyper的技术基础有了一个大致的分析，但是真正想要实现用VM来模拟Docker的一系列行为，光有这个基础是远远不够的。下来我们就根据Docker提供的功能来分析Hyper究竟还做了什么。

1.*Docker的镜像管理*

这个特性基本上是和VM或者container是分离的，所以Hyper也没有重复造轮子，直接使用了Docker对镜像的一系列操作。

2.*Docker后台守护*

Docker提供了dockerd守护进程对container的生命周期进行了管理，同样hyper也提供了hyperd来对vm进行生命周期管理。

3.*Docker 启动* 

Docker使用了container直接启动,然后执行命令，

{%highlight bash%} 
docker run centos:7 /bin/echo 'Hello world'
{%endhighlight%} 

Hyper命令也是如此，使用

{%highlight bash%} 
hyper run centos:7 /bin/echo 'Hello world'
{%endhighlight%} 

也达到了同样的效果。Hyper是如何实现的呢？
Hyper是一个虚拟机，那么不可避免的就需要先启动操作系统，
然后再运行命令。这样从运行速度上来讲，天生就要和Docker产生差距。
Hyper对此进行了优化，使得虚拟机的启动也就只有500ms以内。

为什么会这么快，这里来对这个特性进行一个解密，

首先描述下一个linux在VM启动的过程，VM启动首先也要经过读取模拟BiOS文件来进行硬件自检，然后读取引导程序(例入grub等)，然后再由引导程序载入kernel和ramdisk进行启动。


下面代码是从[runv-qemu]中贴出的代码。

{%highlight go%}

if boot.Bios != "" && boot.Cbfs != "" {
	params = append(params,
		"-drive", fmt.Sprintf("if=pflash,file=%s,readonly=on", boot.Bios),
		"-drive", fmt.Sprintf("if=pflash,file=%s,readonly=on", boot.Cbfs))
} else if boot.Bios != "" {
	params = append(params,
		"-bios", boot.Bios,
		"-kernel", boot.Kernel, "-initrd", boot.Initrd, "-append", "\"console=ttyS0 panic=1 no_timer_check\"")
} else if boot.Cbfs != "" {
	params = append(params,
		"-drive", fmt.Sprintf("if=pflash,file=%s,readonly=on", boot.Cbfs))
} else {
	params = append(params,
		"-kernel", boot.Kernel, "-initrd", boot.Initrd, "-append", "\"console=ttyS0 panic=1 no_timer_check\"")
}
{%endhighlight%} 

可以看到，Hyper使用了[qboot]对启动进行了优化，


这些文件在/var/lib/hyper/目录下，

{%highlight bash%}
[root@localhost hyper]# ll -h
total 145M
-rwxr-xr-x. 1 root root  64K Aug 31 02:52 bios-qboot.bin
-rw-r--r--. 1 root root 4.0M Aug 31 02:52 cbfs-qboot.rom
-rw-r--r--. 1 root root  10G Sep  1 22:39 data
drwxr-xr-x. 2 root root 4.0K Sep  1 22:39 hyper.db
-rwx------. 1 root root 859K Aug 31 02:52 hyper-initrd.img
-rw-r--r--. 1 root root 3.0M Aug 31 02:52 kernel
-rw-r--r--. 1 root root 128M Sep  1 22:39 metadata
{%endhighlight%} 



所以说Hyper是通过定制的kernel和裁剪到极致的ramdisk，以及优化的bios来实现了毫秒级别的启动。


4.*命令的执行*

这里也是hyper的一个亮点，普通的linux启动后，会调用init程序，也就是pid=1的进程。然后拉起很多自启动
的其他进程等等。但是在Hyper的应用场景，其实和Docker一样，并不需要拉起很多进程。所以Hyper自己实现了
一个新的新的init进程，被称为[hyperstart]，这个进程是用go语言实现的，所以只需要libc就可以直接运行，
这也是ramdisk可以很小的原因。有兴趣的同学可以自己解压hyper-initrd.img来看看这个ramdisk究竟包含了什么。
答案我贴出来啦

{%highlight bash%}
[root@localhost xd]# cpio -div < hyper-initrd 
.
lib
lib/x86_64-linux-gnu
lib/x86_64-linux-gnu/libc.so.6
init
lib64
lib64/ld-linux-x86-64.so.2
4103 blocks
{%endhighlight%} 

hyperstart中主要实现了

a.将VirtFS挂在到/tmp/hyper/share 下

b.通过char设备和hyperd进行了通信，获取用户要执行的命令，以及接受其他管理。

c.通过chroot到VirtFS中执行命令。


5.*hyperd和init的通信*

下面展示了一个Hyper启动时候的qemu命令

{%highlight bash%}
qemu-system-x86_64 -machine pc-i440fx-2.0,accel=kvm,usb=off -global kvm-pit.lost_tick_policy=discard -cpu host 
-drive if=pflash,file=/var/lib/hyper/bios-qboot.bin,readonly=on -drive if=pflash,file=/var/lib/hyper/cbfs-qboot.rom,readonly=on 
-realtime mlock=off -no-user-config -nodefaults -no-hpet -rtc base=utc,driftfix=slew -no-reboot -display none -boot strict=on -m 128 -smp 1 
-qmp unix:/var/run/hyper/vm-vMaTbveMPS/qmp.sock,server,nowait 
-serial unix:/var/run/hyper/vm-vMaTbveMPS/console.sock,server,nowait 
-device virtio-serial-pci,id=virtio-serial0,bus=pci.0,addr=0x2 
-device virtio-scsi-pci,id=scsi0,bus=pci.0,addr=0x3 
-chardev socket,id=charch0,path=/var/run/hyper/vm-vMaTbveMPS/hyper.sock,server,nowait 
-device virtserialport,bus=virtio-serial0.0,nr=1,chardev=charch0,id=channel0,name=sh.hyper.channel.0 
-chardev socket,id=charch1,path=/var/run/hyper/vm-vMaTbveMPS/tty.sock,server,nowait 
-device virtserialport,bus=virtio-serial0.0,nr=2,chardev=charch1,id=channel1,name=sh.hyper.channel.1 
-fsdev local,id=virtio9p,path=/var/run/hyper/vm-vMaTbveMPS/share_dir,security_model=none 
-device virtio-9p-pci,fsdev=virtio9p,mount_tag=share_dir
{%endhighlight%}

Hyper虚拟机使用串口来提供登陆功能。
可以看到Hyper还给虚拟机建立了2个char设备，来进行交互，对应host上分别是hyper.sock 和tty.sock
其中hyper.sock 负责vm的启停控制等功能。

在hyperstart的init.c文件中定义了处理逻辑

hyper的handle。主要对POD进行了操作
{%highlight c%}
static int hyper_channel_handle(struct hyper_event *de, uint32_t len)
{
	struct hyper_buf *buf = &de->rbuf;
	struct hyper_pod *pod = de->ptr;
	uint32_t type = 0;
	int i, ret = 0;

	for (i = 0; i < buf->get; i++)
		fprintf(stdout, "%0x ", buf->data[i]);

	type = hyper_get_be32(buf->data);

	fprintf(stdout, "\n %s, type %" PRIu32 ", len %" PRIu32 "\n",
		__func__, type, len);

	pod->type = type;
	switch (type) {
	case STARTPOD:
		ret = hyper_start_pod((char *)buf->data + 8, len - 8);
		hyper_print_uptime();
		break;
	case STOPPOD:
		ret = hyper_stop_pod(pod);
		break;
	case DESTROYPOD:
		fprintf(stdout, "get DESTROYPOD message\n");
		hyper_shutdown(pod);
		return 0;
	case EXECCMD:
		ret = hyper_exec_cmd((char *)buf->data + 8, len - 8);
		break;
	case PING:
	case GETPOD:
		break;
	case READY:
		ret = hyper_rescan();
		break;
	case WINSIZE:
		ret = hyper_set_win_size((char *)buf->data + 8, len - 8);
		break;
	default:
		ret = -1;
		break;
	}

	if (ret < 0)
		hyper_send_type(de->fd, ERROR);
	else
		hyper_send_type(de->fd, ACK);

	return 0;
}

{%endhighlight%}

tty的处理handle如下，

{%highlight c%}
static int hyper_ttyfd_handle(struct hyper_event *de, uint32_t len)
{
	struct hyper_buf *rbuf = &de->rbuf;
	struct hyper_pod *pod = de->ptr;
	struct hyper_exec *exec;
	struct hyper_buf *wbuf;
	uint64_t seq = 0;
	int size;

	seq = hyper_get_be64(rbuf->data);

	dprintf(stdout, "\n%s seq %" PRIu64", len %" PRIu32"\n", __func__, seq, len - 12);

	exec = hyper_find_exec_by_seq(pod, seq);
	if (exec == NULL) {
		wbuf = &de->wbuf;
		fprintf(stderr, "can't find exec whose seq is %" PRIu64 "\n", seq);

		/* goodbye */
		if (wbuf->get + 12 > wbuf->size)
			return 0;

		hyper_set_be64(wbuf->data + wbuf->get, seq);
		hyper_set_be32(wbuf->data + wbuf->get + 8, 12);
		wbuf->get += 12;

		if (hyper_modify_event(ctl.efd, de, EPOLLIN | EPOLLOUT) < 0) {
			fprintf(stderr, "modify ctl tty event to in & out failed\n");
			return -1;
		}

		return 0;
	}

	dprintf(stdout, "find exec %s pid %d, seq is %" PRIu64 "\n",
		exec->id ? exec->id : "pod", exec->pid, exec->seq);
	wbuf = &exec->e.wbuf;

	size = wbuf->size - wbuf->get;
	if (size == 0)
		return 0;

	/* Data may lost since pts buffer is full. do not allow one exec pts occupy all
	 * of the tty buff. */
	if (size > (len - 12))
		size = (len - 12);

	if (size > 0) {
		memcpy(wbuf->data + wbuf->get, rbuf->data + 12, size);
		wbuf->get += size;
		if (hyper_modify_event(ctl.efd, &exec->e, EPOLLIN | EPOLLOUT) < 0) {
			fprintf(stderr, "modify exec pts event to in & out failed\n");
			return -1;
		}
	}

	return 0;
}

{%endhighlight%}

说明一下，通过tty.sock才可以发送数据到hyperd，而hyper.sock主要负责接收数据。

通过hyperstart的这一系列动作，实现了对docker的最后模拟。完成了命令执行的功能。


Hyper的实现还是相当的简洁，对Hyper感兴趣的童鞋可以访问[Hyper github]


[hyperstart]: https://github.com/hyperhq/hyperstart
[qboot]: https://github.com/bonzini/qboot
[runv-qemu]: https://github.com/hyperhq/runv/blob/master/hypervisor/qemu/qemu.go
[VirtFS]: http://wiki.qemu.org/Documentation/9psetup
[Hyper]: https://hyper.sh/
[Hyper github]: https://github.com/hyperhq
[why Hyper]: https://github.com/hyperhq/hyper


