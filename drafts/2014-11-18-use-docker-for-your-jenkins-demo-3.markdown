---
layout: post
title: 使用docker来提升你的Jenkins演示 - 3
---
## 回顾



* 第一篇[使用docker来提升你的Jenkins演示 - 1](http://www.larrycaiyu.com/2014/11/04/use-docker-for-your-jenkins-demo-1.html)，我们把需要的Jenkins软件、插件和任务配置放到了Jenkins docker容器中，使得你很容易演示，别人也很轻松的可以下载自己尝试。
* 第二篇[使用docker来提升你的Jenkins演示 - 2，](http://www.larrycaiyu.com/2014/11/16/use-docker-for-your-jenkins-demo-2.html)我们演示了如何更进一步docker化你的Jenkins应用程序，学到了如何管理配置文件和巧妙地使用`docker run --volume`和`docker exec`来获取容器内的数据

前两篇还是很简单，搭了个架子而已，在Jenkins的持续集成部署中，它的一个重要的使用方式就是构建的实际任务放在Slave（从属）机器上，Jenkins Master（主机）主要负责调度。这样能确保干净的构建环境和整体性能。

![jenkins-demo1-main](http://larrycaiyu.com/images/jenkins-demo1-arch.png)

这就是这个博客系列的第三篇文章，我会用一些例子一步一步来说明如何用docker实现这种演示，如果你对Jenkins的这种使用还不熟悉的话，正好可以一起学习。


## 演示环境 ##

废话少说，直接运行上基于docker的演示环境，如果没有docker的运行环境，请自己安装。Windows/Mac用户推荐[boot2docker](http://boot2docker.com)（也可以使用在线的环境如ustack.com的CoreOS环境，看最后)。

    $ docker run -t -p 8080:8080 larrycai/jenkins-demo3

它会下载`larrycai/jenkins-docker-demo1`这个docker image并运行。Jenkins服务的端口是`8080`，如果是`boot2docker`，那就是 http://192.168.59.103:8080

![jenkins-demo1-main](http://larrycaiyu.com/images/jenkins-demo1-main.png)

你可以直接运行里面的演示任务（job）：`docker-demo`，启动后它会自动下载配套的`larrycai/ubuntu-slave`作为Slave来运行，里面是一个C++的编译环境，非常简洁。

![jenkins-demo1-console](http://larrycaiyu.com/images/jenkins-demo1-console.png)

## 在Jenkins中使用Docker ##

在Jenkins中常见有三种方式可以使用：

1. 拿Docker container作为简单的Slave机器，像管理一般的机器一样管理docker的从属机。
2. 在Jenkins中用配置从属机，自己写脚本管理Docker container，自动读取IP地址和端口。
3. 用[Docker插件][jenkins-docker-plugin]。

前两种比较繁琐，不太Docker化，接下来主要讲如何用[Docker插件][[jenkins-docker-plugin]来使用Docker，这也是上面的演示环境中如何使用Docker的。


## 更加Dockerize

软件开发离不开重构，通过不断的学习用新的技术来提升这个Dockerfile。

## 用docker容器作为从属机器

现在我们考虑来个C++编译的Jenkins任务，在Jenkins中为了不污染主机和提高性能，一般都把具体的累活在从属机上实现。

既然jenkins主机已经用docker了，从属机当仁不让的也用docker了，这个已经有很多文章了，我就用了我以前做的一个镜像larrycai/jenkins-slave-ubuntu

先把它启动后监听在2222端口。

再创建好任务craft2，里面通过下载github outhome仓库，安排在从属机上运行，具体的就不讨论了。

那么在演示时怎么能够为了自动启动从属机器，一般有以下种办法

1. 用脚本或编排（orechstration）工具如fig来启动多docker镜像
2. 在Jenkins任务里面启动docker从属节点
3. 在docker容器启动docker从属节点
4. 用Jenkins docker插件

第一种用脚本的方式肯定可以，但是感觉有点老土，而且还要下载其他脚本，何况也不能确定fig等工具是否能运行。

第四种情况听起来不错，但是坑很多，留待下一篇博客专门来批判。

第二、第三种都需要要用到docker里的docker技术，其中第二种需要改变jenkins里的任务脚本，不够简洁，下面着重讨论第三种。

* $JENKINS_HOME/credentials.xml # 这是一些Jenkins的认证信息，如从属节点的用户认证。

### 结果

老样子，让我们来先来看看效果，或许你也可能只是对这个功能感兴趣。

    docker run –p 8080:8080 –t larrycai/jenkins-demo2

![](http://larrycaiyu.com/images/jenkins-demo1-3.png)

在控制台窗口中上Jenkins已经被启动，然后可以打开浏览器访问`8080`端口。

![](http://larrycaiyu.com/images/jenkins-demo1-4.png)

看起来相当不错，一个叫`craft2`的任务（job）也存在了，Jenkins显示是最新的LTS版本1.580.1

点击`craft`任务，并运行它，然后检查`console`。太棒了，部分结果可以有颜色显示了，这就是我们要的。

![](http://larrycaiyu.com/images/jenkins-demo1-5.png)

然后回过头来看看它是如何配置。

![](http://larrycaiyu.com/images/jenkins-demo1-6.png)

现在演示完毕，可以学习怎么做到的。

### docker里的docker

### 在docker容器启动docker从属节点

### 如何更优雅的控制docker里的docker

## 摘要

在这篇博客中，我们演示了如何dockerize你的Jenkins应用程序，它包含了必须的插件和配置和实例任务。 这将会很容易让你的听众了解你想演示的功能。

所有的代码你都可以在[github上的jenkins-demo1](https://github.com/larrycai/docker-images/tree/master/jenkins-demo1)上找到。

现在，您可以把您的漂亮的Jenkins新功能打包到Docker到处演示。

在接下来的博客中，我将展示如何更好地组织Jenkins目录。

Docker可以帮助我们做很多事情。
