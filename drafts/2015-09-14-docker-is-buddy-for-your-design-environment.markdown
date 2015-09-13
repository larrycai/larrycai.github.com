---
layout: post
title: Docker挺适合用于软件开发环境
---
# 介绍 #

前几天在微信中看到一篇翻译的文章[Docker根本不适合用于本地开发环境](http://dockone.io/article/660)，翻译得很流畅，一下子看懂了，但对于原作者的观点实在不敢苟同。我有点不敢确认是否作者真心研究过docker对于软件开发环境带来的好处。因为我从去年年初开始用docker以后的感触是：部署docker行不行还有待商榷，但是用在开发中是极其适合的、应极力推广的。

这篇博客用我实际工作中的一个例子让大家跟着我一起体会一下，我是如何舒舒服服地用docker来作为开发环境的，也对原文中提到的几个问题一一进行解释。

# 一个开发环境的例子 #

最近我一直在和Jenkins、[Dashing](dashing.io)打交道，前几周，有人提需求要把Jenkins中在等待构建队列（Queue）中超过一定时间的Job显示在Dashing的Widget中，以便于他们及时发现解决。

Jenkins是Java写的，Dashing是Ruby的，为了完成这个任务，我决定使用Groovy。

Docker开始派上大用场了

## 开发环境 ##
我上班使用的是Windows 7，为了使用Docker，它和MacOS一样需要安装[Docker Toolbox](https://www.docker.com/toolbox)（原来叫Boot2docker）, MobaXterm是一个终端软件比PuTTY好用（季浩小盆友推荐的），它用来处理Docker环境。

![](https://www.docker.com/sites/default/files/products/tbox.jpg)

## 源代码 ##

代码当然是用Git来处理，我习惯在Windows上运行Git，如果你习惯可以用在docker中。

![](http://git-scm.com/images/logo@2x.png)
![](http://static.oschina.net/uploads/img/201111/14154628_Ydd9.jpg)

	cd ~/git
    git pull larrycai/docker-dev-demo

里面两个目录dashing、jenkins代表了工作的目录。

启动Docker Toolbox，然后使用MobaXterm登陆到docker主机上，你就可以发现在docker主机中能看到克隆下来的文件了


	ls /c/User/larrcai/docker-dev-demo
    ...


**技巧一**：把代码克隆到用户目录下，这样就可以映射到docker主机中被访问，对应的目录是`/c/User/<id>`,

为了方便起见，我在做个软链接（soft link）放在docker用户下，方便使用 (`~/git`)。 具体方法见stackoverflow 

	docker@default:~$ ls -al
	..
	lrwxrwxrwx    1 root     staff           20 Sep 11 09:19 git -> /c/Users/larrycai/git/
	lrwxrwxrwx    1 root     staff           21 Sep 11 09:19 m2 -> /c/Users/larrycai/.m2/

**技巧二**：建立一些软链接到缺省的`docker`用户下，不然这个目录下的文件重启就重置了。

## Docker镜像 ##

Jenkins和Dashing都做了相应的Docker镜像（嘚瑟一下，[第一本Docker书](http://book.douban.com/subject/26285268/)中的Jenkins镜像代码还是用了我提供的一个版本）

	docker pull larrycai/jenkins
	docker pull larrycai/dashing

## 开始工作 ##

编辑器我就用[Notepad++](https://notepad-plus-plus.org/), sublime太贵，一直没舍得买 ；-）。

	cd ~/git/docker-dev-demo
	docker run $PWD/ ...

打个`docker exec`命令进入容器看看，bingo，代码在容器中已经妥妥的了。

	$ docker exec -it jenkins bash
    # ls $JENKINS_HOME/jobs/sample/workspace
    ....


**技巧三**：熟练掌握`--volume`命令参数和`docker exec`命令

实际上一般我都在目录下放个`README.md`和`docker.sh`,方便copy/paste和直接启动容器（关键我打字太慢）

**技巧四**：多写些脚本替代冗长的`docker`命令，容器依赖多的话，可以考虑`docker-compose`

现在可以在Windows浏览器中，blabla 看效果了

如果有问题，很简单，直接在`notepad++`的修改就可以了，这一步，你自己琢磨吧。

## 再来看docker镜像 ##

为啥不用标准镜像的呢？为了自己的开发方便，我们要不断打磨需要的docker镜像，把常见的依赖都提前包含在里面，繁杂配置都一次设好。

方便分享的，放在github上，自动生成docker image在docker hub上，否则就放在自己内部的docker私有镜像服务器中。

**技巧五**：要不断打磨需要的docker镜像，使得使用最方便。

# 总结 #

看了上面的实践后，我再总结一些，我认为原来的文章中主要有三个方面没解决好，结果导致了错误的结论。

1. 文件的共享问题, 也就是如何在Windows/Mac和docker主机、docker主机和docker容器共享文件，这个分别实际有Docker Toolbox（使用Virtualbox的Guest Addition）和docker --volume很好的解决了。
2. 太拘泥于在容器中运行了:基于上面的文件共享技术，完全可以继续使用原来Windows的IDE（如Sublime）和Git，搭配好MobaXterm来处理docker环境和容器内相关命令的执行。
3. 把简单问题复杂化了：一般Shell脚本就可以解决了，不熟练的话，刚开始还是少用docker-compose。

上面用到的是脚本，实际上对编译型的开发环境也一样。有一阵子在学习陈迪豪的[Seagull海鸥项目](https://github.com/tobegit3hub/seagull)，程序是用golang写的，我也是完完全全用docker环境来开发的，很顺畅。有问题时，因为环境一样，重现也很方便。

虽然我不能打包票你的开发环境也一定很适合用Docker，但是如果你碰到了问题，我很希望一起见面聊聊，看看怎么来解决。如果能在上海凌空SOHO附近的见面的话，一起来喝杯咖啡，如果我解决不了，我请。反之，你懂的。

## 参考 ##



