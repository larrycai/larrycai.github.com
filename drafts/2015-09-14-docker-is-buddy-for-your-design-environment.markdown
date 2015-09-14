---
layout: post
title: Docker挺适合用于软件开发环境
---
# 介绍 #

前几天在微信中看到一篇翻译的文章[Docker根本不适合用于本地开发环境](http://dockone.io/article/660)，翻译得很流畅，一下子看懂了，但对于原作者的观点实在不敢苟同。我有点不敢确认是否作者真心研究过docker对于软件开发环境带来的好处。因为我从去年年初开始用docker以后的感触是：部署docker行不行还有待商榷，但是用在开发中是极其适合的、应极力推广的。

这篇博客用我实际工作中的一个例子让大家跟着我一起体会一下，我是如何舒舒服服地用docker来作为开发环境的，并给出我使用的一些小技巧，最后也对原文中提到的几个问题一一进行解释。

# 真实案例 #

最近我一直在和Jenkins、[Dashing](dashing.io)打交道，前几周，有人提需求要把Jenkins中在等待构建队列（Queue）中超过一定时间的Job显示在Dashing的Widget中，以便于他们及时发现解决。

Jenkins是Java写的，Dashing是Ruby的，为了完成这个任务，我决定使用Groovy来写个Jenkins的Job定期检测并发到dashing上。

Docker开始派上大用场了。

## 开发环境 ##
我上班使用的是Windows 7，所以就以Windows环境为例子，Mac类似，Linux更方便点。为了在Win上使用Docker，它和MacOS一样需要安装[Docker Toolbox](https://www.docker.com/toolbox)（原来叫Boot2docker）, [MobaXterm](http://mobaxterm.mobatek.net/)是一个终端软件比PuTTY好用（季浩小盆友推荐的），它用来处理Docker环境。

![](https://www.docker.com/sites/default/files/products/tbox.jpg)

## 源代码 ##

代码当然是用Git来处理，我习惯在Windows上运行Git，如果你习惯，也可以用在docker中。


**技巧一**：把代码克隆到用户目录下，这样就可以映射到docker主机中被访问，对应的目录是`/c/User/<id>`,

	cd ~/git/docker
    git clone https://github.com/larrycai/docker-dev-demo.git

里面两个目录`dashing`、`jenkins`代表了我要编程的工作目录，第一版工作的代码已经在里面了。

启动Docker Toolbox，然后使用MobaXterm登陆到docker主机上。

	docker@default:~$ ls /c/Users/larrycai/git/docker/docker-dev-demo/
	README.md   dashing/    dashing.sh  jenkins/    jenkins.sh  start.sh    stop.sh

你可以发现在docker主机中能看到克隆下来的文件了。它说明了Windows目录下的文件能够共享到docker主机中。这个不是docker做的，它只是集成进了Virtualbox的共享文件夹功能。


**技巧二**：建立一些软链接到缺省的`docker`用户下，不然这个目录下的文件重启就重置了。

为了方便访问，我一般做个软链接（soft link）如`git`放在docker用户下，这样方便使用 (`~/git`)。 具体方法见[stackoverflow](http://stackoverflow.com/questions/26639968/boot2docker-startup-script-to-mount-local-shared-folder-with-host/) 

	docker@default:~$ ls -al
	..
	lrwxrwxrwx    1 root     staff           20 Sep 11 09:19 git -> /c/Users/larrycai/git/
	lrwxrwxrwx    1 root     staff           21 Sep 11 09:19 m2 -> /c/Users/larrycai/.m2/

## Docker镜像 ##

Jenkins和Dashing都做了相应的Docker镜像（嘚瑟一下，[第一本Docker书](http://book.douban.com/subject/26285268/)中的Jenkins镜像代码还是用了我提供的一个版本）

	docker pull larrycai/jenkins
	docker pull larrycai/dashing

## 启动docker应用容器 ##

先把工作的容器启动起来，我的代码也会共享进去。看不懂的建议买本书温习一下。

	$ cd ~/git/docker/docker-dev-demo
	$ docker run --rm --name dashing -it -p 3030:3030 \
		-v $PWD/dashboards:/dashing/dashboards \
		-v $PWD/widgets:/dashing/widgets \
		larrycai/dashing dashing start
	Thin web server (v1.6.3 codename Protein Powder)
	Maximum connections set to 1024
	Listening on 0.0.0.0:3030, CTRL+C to stop

这样就启动了Dashing容器，接着再在MobaXTerm开启另一个Shell来启动Jenkins环境

	$ docker run --rm -it --name jenkins -p 8080:8080 \
		--link dashing:dashing \
		-v $PWD/jenkins:/opt/jenkins/data/jobs/longjobs/workspace \
	 	larrycai/jenkins
	Running from: /opt/jenkins/jenkins.war
	webroot: EnvVars.masterEnvVars.get("JENKINS_HOME")
	...
	INFO: Jenkins is fully up and running

再开个Shell窗口，打个`docker exec`命令进入容器看看，bingo，你可以看到Windows上的代码在容器中也已经妥妥的了。

	$ docker exec -it jenkins bash
	root@3e026f490f2a:/# ls $JENKINS_HOME/jobs/longjobs/workspace
	docker.sh  longjobs_to_dashing.groovy

**技巧三**：要熟练掌握`--volume`命令参数和`docker exec`命令来理解你的代码能够共享到哪里了

实际上一般我都在目录下放个`README.md`和`docker.sh`,方便copy/paste和直接启动容器（关键我打字太慢），这儿实际上我写了两个脚本`jenkins.sh`和`dashing.sh`

**技巧四**：多写些脚本替代冗长的`docker`命令，容器依赖多的话，可以考虑`docker-compose`

## 配置运行 ##

现在可以在Windows浏览器中配置一下，看效果了。常用的端口我已经在Virtualbox中映射好了，一般我用`localhost`访问

http://localhost:3030/sample （用chrome/firefox打开，不支持早期IE版本）是dashing的，可以看到监控等待队列的Widget已经在那儿了

![](http://www.larrycaiyu.com/images/docker-dev-demo-2.png)

http://localhost:8080 是jenkins的，创建一个新Job叫`longjobs`，然后配好系统的Groovy脚本。这个`longjobs_to_dashing.groovy`就是在git repo中的，现在已经在jenkins容器中了。

![](http://www.larrycaiyu.com/images/docker-dev-demo-1.png)

保存运行，看看输出的console log，应该dashing widget没有大的变化，只有标题变成了`Waited in queue (demo)`，这就是传上去的。

![](http://www.larrycaiyu.com/images/docker-dev-demo-3.png)

## 调试代码 ##

到现在为止，还没有改一行代码，实际上你就是在用docker环境很方便的验证我写的代码！！

来点变化吧，简单的代码编辑器我就用[Notepad++](https://notepad-plus-plus.org/), sublime太贵，一直没舍得买 ；-）。

假设对那个标题不满意，那就在Windows上打开文件`longjobs_to_dashing.groovy`，把大概59行改成你喜欢的标题吧。

	  title: "Waited in queue (demo)",

再把那个job运行一下，一下子就搞定了吧。

如果有想玩dashing，没问题，很简单，可以试试修改`dashing/dashboards/sample.erb`，这一步，你自己琢磨吧。

## 再来看docker镜像 ##

为啥不用标准镜像的呢？一般来说为了自己的开发方便，我们要不断打磨需要的docker镜像，标准镜像一般满足不了需求。我们要把常见的依赖都提前包含在里面，繁杂配置都一次设好。

方便分享的，放在github上，自动生成docker image在docker hub上，否则就放在自己内部的docker私有镜像服务器中。

**技巧五**：要不断打磨需要的docker镜像，使得使用最方便。

# 总结 #

上面的工作方式也是我常用的，很流畅吧。里面的技巧还需要自己体会。

再来看看原来的文章，我认为原来的文章中主要有三个方面没解决好，结果导致了错误的结论。

1. 文件的共享问题, 也就是如何在Windows/Mac和docker主机、docker主机和docker容器共享文件，这个分别实际有Docker Toolbox（使用Virtualbox的共享文件夹技术和`docker run --volume`很好的解决了。
2. 太拘泥于在容器中运行了:基于上面的文件共享技术，完全可以继续使用原来Windows的IDE（如Sublime）和Git，搭配好MobaXterm来处理docker环境和容器内相关命令的执行。
3. 把简单问题复杂化了：一般Shell脚本就可以解决了，不熟练的话，刚开始还是少用`docker-compose`。

上面用到的是脚本，实际上对编译型的开发环境也一样。有一阵子在学习陈迪豪的[Seagull海鸥项目](https://github.com/tobegit3hub/seagull)，程序是用golang写的，我也是完完全全用docker环境来开发的，很顺畅。有问题时，因为环境一样，重现也很方便。

docker用来支撑你的开发环境真的很棒，我们还用在了很多其他地方，有机会再分享。

虽然我不能打包票你的开发环境也一定很适合用Docker，但是如果你碰到了问题，我很希望一起见面聊聊，看看怎么来解决。如果能在上海凌空SOHO附近的见面的话，一起来喝杯咖啡，如果我解决不了，我买单。反之，你懂的。



