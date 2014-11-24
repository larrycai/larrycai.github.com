---
layout: post
title: 使用docker来提升你的Jenkins演示 - 3
---
## 系列介绍

使用docker来提升你的Jenkins演示会安排五篇文章循序渐进地学习：

* 第一篇[使用docker来提升你的Jenkins演示 - 1](http://www.larrycaiyu.com/2014/11/04/use-docker-for-your-jenkins-demo-1.html)：把需要的Jenkins软件、插件和作业配置放到了Jenkins docker容器中，使得你很容易演示，别人也很轻松的可以下载自己尝试。
* 第二篇[使用docker来提升你的Jenkins演示 - 2](http://www.larrycaiyu.com/2014/11/16/use-docker-for-your-jenkins-demo-2.html)：演示了如何更进一步docker化你的Jenkins应用程序，学到了如何管理配置文件和巧妙地使用`run --volume`和`exec`来获取容器内的数据。
* 第三篇：用docker中的docker来解决Jenkins主从机的演示
* 第四篇：巧用ENTRYPOINT/CMD配合脚本来解决复杂的环境
* 第五篇：讨厌的Jenkins docker插件

## 回顾

前两篇还是很简单，搭了个架子而已，在Jenkins的持续集成部署中，它的一个重要的使用方式就是构建的实际任务放在Slave（从属）机器上，Jenkins Master（主机）主要负责调度。这样能确保干净的构建环境和整体性能。

![jenkins-demo1-main](http://larrycaiyu.com/images/jenkins-demo3-1.png)

这就是这个博客系列的第三篇文章，我会用一些例子一步一步来说明如何用docker实现这种演示，如果你对Jenkins的这种使用还不熟悉的话，正好可以一起学习。

## 用docker容器作为从属机器

现在我们考虑来个C++编译的Jenkins任务，在Jenkins中为了不污染主机和提高性能，一般都把具体的累活在从属机上实现。

既然jenkins主机已经用docker了，从属机当仁不让的也用docker了，这个已经有很多文章了，我就用了我以前做的一个镜像`larrycai/jenkins-slave-ubuntu`，可以查看[Dockerfile](https://github.com/larrycai/jenkins-docker-demo1/blob/master/jenkins-slave-ubuntu/Dockerfile)，里面主要有以下几个：

1. `openjdk-7-jdk` 包，jenkins从属机运行环境
2. sshd 服务
3. ssh的公钥（public key)，当然密码也有。
4. 为了演示方便，镜像里面已经预先下载好了一个小的c++程序[outhome](https://github.com/larrycai/out2html/archive/master.zip)仓库。

先把它启动后监听在2222端口。

	docker@boot2docker:~$ docker run -d -p 2222:22 larrycai/jenkins-slave-ubuntu 
	docker@boot2docker:~$ docker ps
	CONTAINER ID        IMAGE                                  COMMAND                CREATED             STATUS              PORTS                    NAMES
	4d1189c2c7be        larrycai/jenkins-slave-ubuntu:latest   "/usr/sbin/sshd -D"    16 minutes ago      Up 16 minutes       0.0.0.0:2222->22/tcp     boring_morse

## 创建新环境 craft3 ##
 
把上次的[larrycai/jenkins-demo2](https://github.com/larrycai/docker-images/tree/master/jenkins-demo2)构建目录拷贝一份到新的工作环境，这次叫`larrycai/jenkins-demo3`，重新构建，启动（别忘了上次怎么做`JENKINS_HOME`）。

	docker@boot2docker:$ docker build -t larrycai/jenkins-demo3 .
	docker@boot2docker:$ docker run -v ~/jenkins:/data -p 8080:8080 -it larrycai/jenkins-demo3

先在系统中配好从节点，如下：

![jenkins-demo1-main](http://larrycaiyu.com/images/jenkins-demo3-2.png)

`Host`要选`172.17.42.1`这是boot2docker的主机IP地址，每个docker容器都能访问，端口就是映射出的`2222`端口。这里也可以直接访问从属机的IP地址，那么端口就是标准的`22`了。

其中还要配好ssh访问的私钥。

![jenkins-demo1-main](http://larrycaiyu.com/images/jenkins-demo3-3.png)

最后创建好新的工作craft3。

![jenkins-demo1-main](http://larrycaiyu.com/images/jenkins-demo3-4.png)

里面配好从属机和工作脚本，运行一下看看

![jenkins-demo1-main](http://larrycaiyu.com/images/jenkins-demo3-5.png)

一级棒，它工作了。

用上一篇博客教的方式把数据拷贝出来放到镜像里，重新构建镜像实验一下。其中这次要多加一个配置文件

* $JENKINS_HOME/credentials.xml # 这是一些Jenkins的认证信息，如从属节点的用户认证。

## 自动启动从属节点 ##

现在从属节点是手工下载和启动的，那么在演示时怎么能够自动启动从属节点呢，一般有以下种办法

1. 用脚本或编排（orechstration）工具如fig来启动多docker镜像
2. 在Jenkins任务里面启动docker从属节点
3. 在docker容器启动docker从属节点
4. 用[Jenkins Docker插件](https://wiki.jenkins-ci.org/display/JENKINS/Docker+Plugin)

第一种用脚本的方式肯定可以，但是感觉有点老土，而且还要下载其他脚本，何况也不能确定fig等工具是否能运行。

第四种情况听起来不错，但是坑很多，留待下一篇博客专门来批判。

第二、第三种都需要要用到docker里的docker技术，其中第二种需要改变jenkins里的任务脚本，不够简洁，下面着重讨论第三种。

### docker里的docker

Docker是有客户端和服务器Daemon端组成，具体请看Infoq上的[Docker源码分析（一）：Docker架构](http://www.infoq.com/cn/articles/docker-source-code-analysis-part1)。

所以我们可以在镜像装好docker客户端，很简单就是一个二进制文件。

	RUN curl https://get.docker.io/builds/Linux/x86_64/docker-latest -o /usr/local/bin/docker
	RUN chmod +x /usr/local/bin/docker

为了访问，我们必须传递访问的套接字`/var/run/docker.sock`，这个用我们学过的标记`-v`就可以了。

我们把这段写在已有的`start.sh`脚本中，放在启动jenkins之前。

	#!/bin/bash
	
	export DOCKER_HOST=unix://docker.sock
	docker run -d -p 2222:22 larrycai/jenkins-slave-ubuntu
	
	exec java -jar /opt/jenkins/jenkins.war

重新构建运行一下（别忘了`docker.sock`）。

	docker run -v /var/run/docker.sock:/docker.sock -p 8080:8080 -it larrycai/jenkins-demo3

就这么简单！是的，docker就是简洁，永远的v5。

## 摘要

在这篇博客中，我们演示了如何dockerize你的Jenkins应用程序，它包含了必须的插件和配置和实例任务。 这将会很容易让你的听众了解你想演示的功能。

所有的代码你都可以在[github上的jenkins-demo1](https://github.com/larrycai/docker-images/tree/master/jenkins-demo1)上找到。

现在，您可以把您的漂亮的Jenkins新功能打包到Docker到处演示。

在接下来的博客中，我将展示如何更好地组织Jenkins目录。

Docker可以帮助我们做很多事情。
