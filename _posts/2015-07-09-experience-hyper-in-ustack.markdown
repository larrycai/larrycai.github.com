---
layout: post
title: 在Ustack公有云上体验Hyper-一个基于Hypervisor的容器化解决方案
---
# 介绍 #

现在Docker和容器如日中天，大家都在谈论，这些都是基于LXC的。前一阵子在微博上，[马全一讲国内的技术流容器相关的创业项目时，提到了Hyper](http://weibo.com/1642262644/Cpi2p7emf)，兴趣大增。

先阅读了InfoQ的文章[Hyper是一个基于Hypervisor的容器化解决方案][hyperinfoq]，还有官方网站[http://hyper.sh](http://hyper.sh)，都不是特别理解如何在虚拟机上启动docker（是不是笨了好多）和它的效果。

趁上次[UnitedStack公有云][ustack]上给的学习券（coupon）还没用光，决定在公有云上试一试。 （是不是这篇博客又可以给我的coupon呢，哈哈）

这篇博客就是介绍如何在ustack上体验一下Hyper的效果来加深理解，相信你自己用物理机或虚机也是一样的（用公有云省去安装和proxy的问题）。

# 安装 #

实际上Hyper的安装，[官方文档](https://docs.hyper.sh/get_started/install.html)写得挺详细了，只是他是英文，也没针对哪个版本，这里就更详细些。


在ustack上选择Ubuntu 15.04，最小配置，连上公网IP，ssh登陆，一气呵成。

![](http://www.larrycaiyu.com/images/hyper-1.png)

现在就可以安装了，需要docker，qemu-KVM包。 Ubuntu15.04中可以装docker1.5版本，如果需要最新的加个PPA源。

	apt-get update                             # 缺省是国外apt源，有点慢
	apt-get install docker.io                  # docker包是docker.io
	service docker start					   # hyper安装需要docker服务
	apt-get install qemu-kvm
	curl -sSL https://hyper.sh/install | bash  # 见文档，这是直接装hyper二进制包

# 运行 #

看上去好了，试试吧。国外的网站，下docker镜像（image）比较慢，幸好不到，有空可以把他转到国内的docker镜像源（registry mirror）。

	root@hyper1:~# docker pull ubuntu:latest  # 加latest，否则下一坨，
	root@hyper1:~# hyper run ubuntu
	POD id is pod-EljQlbnRAP
	root@ubuntu-5747385737:/# cat /etc/lsb-release
	DISTRIB_ID=Ubuntu
	DISTRIB_RELEASE=14.04
	DISTRIB_CODENAME=trusty

hyper启动Ubuntu容器（`hyper run ubuntu`)的速度比docker稍微有点慢，但是还是飞快，感觉也是一个完整的Ubuntu环境。

退出后，试一下`hyper list`命令

	root@hyper1:~# hyper list
	         POD ID                      POD Name             VM name    Status
	 pod-EljQlbnRAP             ubuntu-5747385737                     succeeded

Bingo，就这么简单。和网站上的效果一样，但是为啥是虚拟机上的容器呢？

# 再看看 #

既然是虚机上的容器，那么底下的linux kernel应该不同。

	root@hyper1:~# uname -a
	Linux hyper1 3.19.0-15-generic #15-Ubuntu SMP Thu Apr 16 23:32:37 UTC 2015 x86_64 x86_64 x86_64 GNU/Linux
	root@hyper1:~# docker run ubuntu uname -a
	Linux b1f5523cb694 3.19.0-15-generic #15-Ubuntu SMP Thu Apr 16 23:32:37 UTC 2015 x86_64 x86_64 x86_64 GNU/Linux

docker运行的容器和主机的linux kernel是一样的`3.19.0-15-generic #15-Ubuntu SMP`,再来看看hyper：

	root@hyper1:~# hyper run ubuntu uname -a
	POD id is pod-ErbgsvxAjz
	Linux ubuntu-8938030742 4.0.4-hyper #16 SMP Fri Jun 19 10:02:22 CST 2015 x86_64 x86_64 x86_64 GNU/Linux

hyper的kernel是`4.0.4-hyper #16 SMP`，是在自己的虚拟核上。

再来看看`qemu`吧，启动ubuntu容器`hyper run ubuntu`,然后在另一个shell中查看`qemu`,果然多了一个qemu虚拟机的进程。
	
	root@hyper1:~# ps -ef | grep qemu
	root       526     1  0 09:05 ?        00:00:00 /usr/sbin/qemu-ga --daemonize -m virtio-serial -p /dev/virtio-ports/org.qemu.guest_agent.0
	root     13642     1 14 09:52 ?        00:00:04 qemu-system-x86_64 -machine pc-i440fx-2.0,usb=off -cpu core2duo -drive if=pflash,file=/var/lib/hyper/bios-qboot.bin,readonly=on -dri
	ve if=pflash,file=/var/lib/hyper/cbfs-qboot.rom,readonly=on -realtime mlock=off -no-user-config -nodefaults -no-hpet -rtc base=utc,driftfix=slew -no-reboot -display none -boot stri
	ct=on -m 128 -smp 1 -qmp unix:/var/run/hyper/vm-pfbFabRstw/qmp.sock,server,nowait -serial unix:/var/run/hyper/vm-pfbFabRstw/console.sock,server,nowait -device virtio-serial-pci,id=
	virtio-serial0,bus=pci.0,addr=0x2 -device virtio-scsi-pci,id=scsi0,bus=pci.0,addr=0x3 -chardev socket,id=charch0,path=/var/run/hyper/vm-pfbFabRstw/hyper.sock,server,nowait -device
	virtserialport,bus=virtio-serial0.0,nr=1,chardev=charch0,id=channel0,name=sh.hyper.channel.0 -chardev socket,id=charch1,path=/var/run/hyper/vm-pfbFabRstw/tty.sock,server,nowait -de
	vice virtserialport,bus=virtio-serial0.0,nr=2,chardev=charch1,id=channel1,name=sh.hyper.channel.1 -fsdev local,id=virtio9p,path=/var/run/hyper/vm-pfbFabRstw/share_dir,security_mode
	l=none -device virtio-9p-pci,fsdev=virtio9p,mount_tag=share_dir

里面具体的你可以自己研究研究吧。

# 总结 #

实验做完后，和[作者王旭 @gnawux](http://weibo.com/gnawux)在微博上又聊了几句，再回去看看[infoq的那篇文章][hyperinfoq]就明白多了（当然还是皮毛）。

Hyper是一个基于Hypervisor的容器化解决方案，非常有新意，VMWare也跟着推出了相类似的[Bonneville项目](http://blogs.vmware.com/cloudnative/introducing-project-bonneville/)，这块可能会有很到的市场。特别是大部分用户已经有了成熟的虚拟化基础架构，他们可能更倾向于这种基于虚拟机的方案。

容器技术还需不断磨合，Hyper现在也还只是0.2.1版本 (2015-07-06)，假以时日，或许他能成为容器中的一颗明珠。

来吧，加入Hyper的社区，一起来点好点子吧。


# 参考 #

* http://www.infoq.com/cn/news/2015/06/Hyper-Hypervisor-Docker 
* http://hyper.sh 

[hyperinfoq]: http://www.infoq.com/cn/news/2015/06/Hyper-Hypervisor-Docker
[ustack]: https://www.ustack.com