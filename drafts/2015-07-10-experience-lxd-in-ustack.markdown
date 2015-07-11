---
layout: post
title: (没有写完）在Ustack公有云上体验LXD：一个基于Linux容器上的Hypervisor
---
# 介绍 #

[粗略看了Hyper](http://www.larrycaiyu.com/2015/07/09/experience-hyper-in-ustack.html)以后，我想换个方向，看看Ubuntu的LXD。它也不同于Docker container技术。

国内介绍LXD的文章很少，国外也不多（看参考链接）。虽说存在挺长时间了，但只是在今年五月渥太华的Openstack Summit上Ubuntu讲了[LXD vs KVM](https://www.openstack.org/summit/vancouver-2015/summit-videos/presentation/lxd-vs-kvm)，才引起了不少注意。

这篇博客就是介绍如何在[UnitedStack公有云][ustack]上体验一下LXD的效果来加深理解，相信你自己用物理机或虚机也是一样的（用公有云省去安装和proxy的问题）。

# 安装 #

在ustack上选择Ubuntu 15.04，最小配置，连上公网IP，ssh登陆，一气呵成。

![](http://www.larrycaiyu.com/images/hyper-1.png)

现在就可以安装了，LXD 0.7也已经在Ubuntu 15.04中了，安装比较方便，看[官方文档](https://insights.ubuntu.com/2015/04/28/getting-started-with-lxd-the-container-lightervisor/)。我再借鉴了github上的[LXD项目](https://github.com/lxc/lxd),用PPA更新到最新版本。

	add-apt-repository ppa:ubuntu-lxc/lxd-git-master 
	apt-get update
	apt-get install lxd
	
# 运行 #

缺省服务没起来

	service lxd start

加上LXC的源

	lxc remote add lxc-org images.linuxcontainers.org

查看服务器上的

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
	
	root@lxd:~# lxc launch lxc-org:/ubuntu/trusty/i386 ubuntu-32
	Creating container...

这个就一直停在这儿

	root@lxd:~# lxc image copy lxc-org:/ubuntu/trusty/amd64 local: --alias=ubuntu-64

这儿一直没搞定，返回不同的错

	error: UNIQUE constraint failed: images.fingerprint
	error: Post http://unix.socket/1.0/images: unexpected EOF
	Temporary failure in name resolution
	error: Get https://images.linuxcontainers.org:8443/1.0: read tcp 192.99.34.219:8443: connection reset by peer

	root@lxd:~# lxd-images import lxc ubuntu trusty amd64 --alias ubuntu
	Downloading the GPG key for https://images.linuxcontainers.org
	Downloading the image list for https://images.linuxcontainers.org
	Validating the GPG signature of /tmp/tmp7cwp4e4u/index.json.asc
	Downloading the image: https://images.linuxcontainers.org/images/ubuntu/trusty/amd64/default/20150619_19:15/lxd.tar.xz
	Validating the GPG signature of /tmp/tmp7cwp4e4u/ubuntu-trusty-amd64-default-20150619_19:15.tar.xz.asc
	Image imported as: 04aac4257341478b49c25d22cea8a6ce0489dc6c42d835367945e7596368a37f
	Setup alias: ubuntu
		
	root@lxd:~# lxc launch ubuntu
	Creating container...done
	Starting container...done
	root@lxd:~# lxc list
	+---------------+---------+------------+------+-----------+
	|     NAME      |  STATE  |    IPV4    | IPV6 | EPHEMERAL |
	+---------------+---------+------------+------+-----------+
	| apneic-kamron | RUNNING | 10.0.3.221 |      | NO        |
	+---------------+---------+------------+------+-----------+
	
	root@lxd:~# lxc info apneic-kamron
	Name: apneic-kamron
	Status: RUNNING
	Init: 3627
	Ips:
	  eth0:  IPV4   10.0.3.221
	  lo:    IPV4   127.0.0.1
	  lo:    IPV6   ::1

	root@lxd:~# lxc image list
	+--------------+--------------+--------+-----------------------+--------+------------------------------+
	|    ALIAS     | FINGERPRINT  | PUBLIC |      DESCRIPTION      |  ARCH  |         UPLOAD DATE          |
	+--------------+--------------+--------+-----------------------+--------+------------------------------+
	| ubuntu       | 04aac4257341 | no     |                       | x86_64 | Jul 11, 2015 at 4:31pm (CST) |
	| ubuntu-32    | 230c0c42fa5a | no     | Ubuntu trusty (i386)  | i686   | Jul 10, 2015 at 9:44am (CST) |
	| jessie-amd64 | f86a94578985 | no     | Debian jessie (amd64) | x86_64 | Jul 10, 2015 at 9:45am (CST) |
	+--------------+--------------+--------+-----------------------+--------+------------------------------+
	
	root@lxd:~# lxc exec ub /bin/bash
	error: websocket: bad handshake

# 再看看 #

# 和其他技术的关系 # 
## 和docker、appc的关系 ##
## openstack ##
## lxc ##
## docker下的lxc ##


# 总结 #


# 参考 #

* https://github.com/lxc/lxd
* http://www.ubuntu.com/cloud/tools/lxd
* https://insights.ubuntu.com/2015/04/28/getting-started-with-lxd-the-container-lightervisor/
* https://insights.ubuntu.com/2015/06/30/publishing-lxd-images/
* http://blog.scottlowe.org/2015/05/06/quick-intro-lxd/
* https://github.com/lxc/lxd/issues/756
* https://images.linuxcontainers.org/images

[ustack]: https://www.ustack.com