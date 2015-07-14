---
layout: post
title: 在Ustack公有云上体验LXD：一个基于Linux容器上的Hypervisor
---
# 介绍 #

[粗略看了Hyper](http://www.larrycaiyu.com/2015/07/09/experience-hyper-in-ustack.html)以后，我想换个方向，看看Ubuntu的LXD。它不同于Docker/CoreOS container技术。

国内介绍LXD的文章很少，国外也不多（看参考链接）。虽说存在挺长时间了，但只是在今年五月渥太华的Openstack Summit上Ubuntu讲了[LXD vs KVM](https://www.openstack.org/summit/vancouver-2015/summit-videos/presentation/lxd-vs-kvm)，才引起了更多注意。

这篇博客就是介绍如何在[UnitedStack公有云][ustack]上体验一下LXD的效果来加深理解，相信你自己用物理机或虚机也是一样的（用公有云省去安装和proxy的问题）。

# 安装 & 运行 #

## 安装 ##

在ustack上选择Ubuntu 15.04，最小配置，连上公网IP，ssh登陆，一气呵成。

![](http://www.larrycaiyu.com/images/lxd-1.png)

现在就可以安装了，LXD 0.7也已经在Ubuntu 15.04中了，安装比较方便，看[官方文档](https://insights.ubuntu.com/2015/04/28/getting-started-with-lxd-the-container-lightervisor/)。

我再借鉴了github上的[LXD项目](https://github.com/lxc/lxd)，用PPA更新到最新版本。 别忘了更新`lxd-client`，它包含了使用到的`lxc`命令。 否则会出现不一致的问题：`websocket: bad handshake`，参见[#832](https://github.com/lxc/lxd/issues/832)

	add-apt-repository ppa:ubuntu-lxc/lxd-git-master 
	apt-get update
	apt-get install lxd
	apt-get install lxd-client
	
## 运行 ##

缺省服务没起来

	service lxd start

LXC的系统源都在[https://images.linuxcontainers.org](https://images.linuxcontainers.org)，加上LXC的源，方便访问。`lxc-org`只是个别名。

	lxc remote add lxc-org images.linuxcontainers.org

查看服务器上的列表。

	root@lxd:~# lxc image list lxc-org:
	+--------------------------------+--------------+--------+-------------------------+---------+-------------------------------+
	|             ALIAS              | FINGERPRINT  | PUBLIC |       DESCRIPTION       |  ARCH   |          UPLOAD DATE          |
	+--------------------------------+--------------+--------+-------------------------+---------+-------------------------------+
	|                                | 8d552ca0de3c | yes    | Centos 6 (amd64)        | x86_64  | Jun 17, 2015 at 11:17am (CST) |
	|                                | f7d9e7940fbb | yes    | Centos 6 (amd64)        | x86_64  | Jun 18, 2015 at 11:17am (CST) |
	| centos/6/amd64 (1 more)        | afae698680fc | yes    | Centos 6 (amd64)        | x86_64  | Jun 19, 2015 at 11:17am (CST) |
	|                                | 59a75a69c16e | yes    | Centos 6 (i386)         | i686    | Jun 17, 2015 at 11:19am (CST) |
	|                                | 824c9996bb00 | yes    | Centos 6 (i386)         | i686    | Jun 18, 2015 at 11:20am (CST) |
	| centos/6/i386 (1 more)         | 3266fd8b7b6e | yes    | Centos 6 (i386)         | i686    | Jun 19, 2015 at 11:20am (CST) |
	
把实验用的`ubuntu:trusty`版本`import`到本地。这个国内访问速度极慢，我弄了一个周末才成功，而且没有提示，参见[#833](https://github.com/lxc/lxd/issues/833)。看上去他们会很快解决。（小技巧：有问题，要提交issue，这才是开源的力量）

	root@lxd:~# lxd-images import lxc ubuntu trusty amd64 --alias ubuntu
	Downloading the GPG key for https://images.linuxcontainers.org
	Downloading the image list for https://images.linuxcontainers.org
	Validating the GPG signature of /tmp/tmp7cwp4e4u/index.json.asc
	Downloading the image: https://images.linuxcontainers.org/images/ubuntu/trusty/amd64/default/20150619_19:15/lxd.tar.xz
	Validating the GPG signature of /tmp/tmp7cwp4e4u/ubuntu-trusty-amd64-default-20150619_19:15.tar.xz.asc
	Image imported as: 04aac4257341478b49c25d22cea8a6ce0489dc6c42d835367945e7596368a37f
	Setup alias: ubuntu
		
你也可以多下载几个，然后看看结果。

	root@lxd:~# lxc image list
	+--------------+--------------+--------+-----------------------+--------+------------------------------+
	|    ALIAS     | FINGERPRINT  | PUBLIC |      DESCRIPTION      |  ARCH  |         UPLOAD DATE          |
	+--------------+--------------+--------+-----------------------+--------+------------------------------+
	| ubuntu       | 04aac4257341 | no     |                       | x86_64 | Jul 11, 2015 at 4:31pm (CST) |
	| ubuntu-32    | 230c0c42fa5a | no     | Ubuntu trusty (i386)  | i686   | Jul 10, 2015 at 9:44am (CST) |
	| jessie-amd64 | f86a94578985 | no     | Debian jessie (amd64) | x86_64 | Jul 10, 2015 at 9:45am (CST) |
	+--------------+--------------+--------+-----------------------+--------+------------------------------+

然后是启动`lxc launch`，取别名`ub`

	root@lxd:~# lxc launch ubuntu ub
	Creating container...done
	Starting container...done
	root@lxd:~# lxc list
	+---------------+---------+------------+------+-----------+
	|     NAME      |  STATE  |    IPV4    | IPV6 | EPHEMERAL |
	+---------------+---------+------------+------+-----------+
	| ub            | RUNNING | 10.0.3.221 |      | NO        |
	+---------------+---------+------------+------+-----------+

运行`lxc exec`	

	root@lxd:~# lxc exec ub /bin/bash
	root@ub:~# uname -a
	Linux ub 3.19.0-15-generic #15-Ubuntu SMP Thu Apr 16 23:32:37 UTC 2015 x86_64 x86_64 x86_64 GNU/Linux	
	
# 简单分析 #

现在我们简单分析一下lxd和docker的相同点和区别。这只是初步的，关于更深层次的分析，可以查其他的资料。

## 都是linux容器 ##

首先他们都是基于linux容器的技术，所以在容器中看到的内核都是服务器的`3.19.0-15-generic #15-Ubuntu SMP`。

	root@lxd:~# uname -a
	Linux lxd 3.19.0-15-generic #15-Ubuntu SMP Thu Apr 16 23:32:37 UTC 2015 x86_64 x86_64 x86_64 GNU/Linux
	root@lxd:~# lxc exec ub /bin/bash
	root@ub:~# uname -a
	Linux ub 3.19.0-15-generic #15-Ubuntu SMP Thu Apr 16 23:32:37 UTC 2015 x86_64 x86_64 x86_64 GNU/Linux	

## 启动模式 ##

启动模式，这是最大区别，`docker`启动后，里面看到就是一个process


	docker@boot2docker:~$ docker run -it ubuntu bash
	root@4a08576d739b:/# ps -ef
	UID        PID  PPID  C STIME TTY          TIME CMD
	root         1     0  0 09:24 ?        00:00:00 bash
	root        15     1  0 09:25 ?        00:00:00 ps -ef

而在lxd中，他是一个比较完整的系统，这是他叫基于Linux容器的Hypervisor的原因。LXD是想和KVM PK的。

	root@ub:~# ps -ef
	UID        PID  PPID  C STIME TTY          TIME CMD
	root         1     0  0 01:45 ?        00:00:14 /sbin/init
	root       539     1  0 01:46 ?        00:00:01 upstart-udev-bridge --daemon
	root       615     1  0 01:46 ?        00:00:00 /lib/systemd/systemd-udevd --daemon
	root       796     1  0 01:46 ?        00:00:00 dhclient -1 -v -pf /run/dhclient.eth0.pid -lf /var/lib/dhcp/dhclient.eth0.leases eth0
	syslog     866     1  0 01:46 ?        00:00:00 rsyslogd
	root      1046     1  0 01:46 ?        00:00:00 cron
	root      1110     1  0 01:46 console  00:00:00 /sbin/getty -8 38400 console
	root      1209     1  0 01:46 ?        00:00:01 upstart-socket-bridge --daemon
	root      1210     1  0 01:46 ?        00:00:01 upstart-file-bridge --daemon
	root      2397     0  0 01:57 ?        00:00:00 /bin/bash
	root      7210     0  0 02:17 ?        00:00:00 /bin/bash
	root      9515     1  0 02:50 ?        00:00:00 /sbin/getty -8 38400 tty4
	root      9517     1  0 02:50 ?        00:00:00 /sbin/getty -8 38400 tty2
	root      9518     1  0 02:50 ?        00:00:00 /sbin/getty -8 38400 tty3
	root      9519     1  0 02:50 ?        00:00:00 /sbin/getty -8 38400 tty1
	root      9527  7210  0 02:50 ?        00:00:00 ps -ef

## 程序运行 ##

那如果在容器里运行程序会怎么样呢。

lxd中：

	root@lxd:~# lxc exec ub bash
	root@ub:~# sleep 100
	root@lxd:~# ps -ef | grep sleep
	100000    3580  3544  0 12:27 pts/12   00:00:00 sleep 100

docker中：

	docker@boot2docker:~$ docker run -it ubuntu bash
	root@2fbd16a0b1ad:/# sleep 100
	docker@boot2docker:~$ ps -ef | grep sleep
	root     25621 25603  0 11:06 pts/5    00:00:00 sleep 100

在容器外面服务器上，都能看到这个运行的进程。

# 总结 #

通过几个简单的实验，可以看出LXD是在一个基于container的Linux容器虚拟机，结合了linux容器和虚拟机的优点，目标是云里替换KVM虚拟机。因此他还有一个openstack的组件`nova-compute-lxd`。

我对底层技术不在行，这里只是搭了个环境，理解一下他的简单区别。希望有高手有进一步分析。

LXD一再强调他自己是系统级的container，docker/CoreOS是业务层的App container，两者互补，并不冲突。不过docker和CoreOS像是并不买它的账，毕竟没有这一层，容器也可以运行的很好。

技术就是这样，不是你说好就是好。

## 碰到的问题 ##

中间还碰到了各种问题

    root@lxd:~# lxc launch lxc-org:/ubuntu/trusty/i386 ubuntu-32
	Creating container...

这个就一直停在这儿，估计就是网速慢的原因，后来我用`lxd-images import`命令，问题看得清楚一些。

	root@lxd:~# lxc image copy lxc-org:/ubuntu/trusty/amd64 local: --alias=ubuntu-64

这儿网速慢时，会返回下面各种不同的错，反正我也用`lxd-images import`命令替换，功能都一样。

	error: UNIQUE constraint failed: images.fingerprint
	error: Post http://unix.socket/1.0/images: unexpected EOF
	Temporary failure in name resolution
	error: Get https://images.linuxcontainers.org:8443/1.0: read tcp 192.99.34.219:8443: connection reset by peer

## 参考 ##

下面是我做实验时用到的一下资料。

* https://github.com/lxc/lxd
* http://www.ubuntu.com/cloud/tools/lxd
* https://insights.ubuntu.com/2015/04/28/getting-started-with-lxd-the-container-lightervisor/
* https://insights.ubuntu.com/2015/06/30/publishing-lxd-images/
* http://blog.scottlowe.org/2015/05/06/quick-intro-lxd/
* https://github.com/lxc/lxd/issues/756
* https://images.linuxcontainers.org/images
* https://github.com/lxc/lxd/issues/833
* https://github.com/lxc/lxd/issues/832
* https://github.com/lxc/lxd/blob/master/scripts/lxd-images 

[ustack]: https://www.ustack.com