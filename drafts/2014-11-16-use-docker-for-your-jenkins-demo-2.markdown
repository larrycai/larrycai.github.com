---
layout: post
title: 使用docker来提升你的Jenkins演示 - 2 
---
## 回顾

在上一篇[使用docker来提升你的Jenkins演示 - 1](http://www.larrycaiyu.com/2014/11/04/use-docker-for-your-jenkins-demo-1.html)，我们把需要的Jenkins软件、插件和任务配置放到了Jenkins docker容器中，使得你很容易演示，别人也很轻松的下载自己尝试。

如果你真的捧场尝试过得化，你会发现还是有些问题

1. 那个任务的配置文件`config.xml`那样运行容器通过命令取到还是繁琐。
2. 当考虑到系统的配置文件和其他配置内容时，一堆文件在`Dockerfile`中被`ADD`进去，还是不干净。
3. 常见的主从模拟（master/slave）需要配置从节点也没有提到

那就对了，这就是这个博客系列的第二篇文章，我会用一些例子一步一步来说明如何实现这些目标。

## 更加Dockerize

软件开发离不开重构，通过不断的学习用新的技术来提升这个Dockerfile

### 把零散的文件放在一起

先来解决第二个问题，与其把文件一个个加上去，为什么不把整个目录在本地准备好，直接加入目录岂不更好。

	# ADD JENKINS_HOME 
	ADD JENKINS_HOME $JENKINS_HOME
	RUN chmod +x $JENKINS_HOME/start.sh
	
	EXPOSE 8080
	
	CMD [ "/opt/jenkins/data/start.sh" ]

然后按照`JENKINS`的目录结构，把`config.xml`放在任务下面，顺势把启动脚本也搁在里面，这样`Dockerfile`看上去及其清爽

	$ find JENKINS_HOME/
	JENKINS_HOME/
	JENKINS_HOME/jobs
	JENKINS_HOME/jobs/craft
	JENKINS_HOME/jobs/craft/config.xml
	JENKINS_HOME/start.sh

试着docker build再运行一下，结果和上次一样，v5。

### 调试时，共享目录来传递运行数据

`JENKINS_HOME`下的数据还是上次的，我们现在通过docker run中的`-v`参数使得容器中的内容能够暴露在外面，在结合docker 1.3开始支持的`docker exec`可以让我们直接把数据拷贝出来。

    docker build -t larrycai/jenkins-demo2 .
    mkdir -p $PWD/jenkins
    docker run -v $PWD/jenkins:/data -P larrycai/jenkins-demo2
    docker ps # 得到容器的ID
    docker exec -it <容器的id> bash

构建启动后，通过exec命令进入容器。现在容器中的`/data`目录就是docker主机上的`$PWD/jenkins`目录。你就可以把需要的文件倒腾出来，一般来说是：

* $JENKINS_HOME/jobs # 各个任务的配置和历史记录，一般拷贝`config.xml`即可
* $JENKINS_HOME/config.xml # 这是jenkins全局配置文件，如从属节点的信息
* $JENKINS_HOME/credentials.xml # 这是一些Jenkins的认证信息，如从属节点的用户认证。


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
