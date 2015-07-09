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

看上去好了，试试吧。国外的网站，下docker镜像（image）很慢，有空把他转到国内的docker镜像源（registry mirror）。

	# docker pull ubuntu:latest  # 加latest，否则下一坨，
	# hyper run ubuntu /bin/bash

比docker稍微有点慢，但是还是飞快，感觉也是一个完整的Ubuntu环境。

	# hyper list

Bingo，就这么简单。和网站上的效果一样，但是为啥是虚拟机上的容器呢？

# 再看看 #

	service docker start
	docker pull ubuntu:latest




# 总结 #

做完后，和[作者王旭 @gnawux](http://weibo.com/gnawux)在微博上又聊了几句，再回去看看[infoq的那篇文章][hyperinfoq]就明白多了（当然还是皮毛）。

Hyper是一个基于Hypervisor的容器化解决方案，非常有新意，VMWare也跟着推出了相类似的[Bonneville项目](http://blogs.vmware.com/cloudnative/introducing-project-bonneville/)，这块可能会有很到的市场。特别是大部分用户已经有了成熟的虚拟化基础架构，他们可能更倾向于这种基于虚拟机的方案。

容器技术还需不断磨合，Hyper现在也还只是0.2.1版本 (2015-07-06)，假以时日，或许他能成为容器中的一颗明珠。

来吧，加入Hyper的社区，一起来点好点子吧。


# 参考 #

* http://www.infoq.com/cn/news/2015/06/Hyper-Hypervisor-Docker 
* http://hyper.sh 

[hyperinfoq]: http://www.infoq.com/cn/news/2015/06/Hyper-Hypervisor-Docker
[ustack]: https://www.ustack.com