---
layout: post
title: 使用docker来提升你的Jenkins演示 - 2 
---
## 回顾

在上一篇[使用docker来提升你的Jenkins演示 - 1](http://www.larrycaiyu.com/2014/11/04/use-docker-for-your-jenkins-demo-1.html)，我们把需要的Jenkins软件、插件和作业配置放到了Jenkins docker容器中，使得你很容易演示，别人也可以很轻松地下载自己尝试。

如果你真的捧场尝试过得化，你可能已经发现了一些问题：

1. 那个作业的配置文件`config.xml`那样运行容器通过命令取到还是繁琐，不太直观。
2. 如果还要调整系统的配置文件和其他配置内容时，一堆文件在`Dockerfile`中被`ADD`进去，还是不干净。

那就对了，这就是这个博客系列的第二篇文章，我会用一些例子一步一步来说明如何实现这这两个目标。

## 更加Docker化（Dockerize）

软件开发离不开重构，现在继续通过不断的研磨来提升这个Dockerfile。

### 把零散的文件放在一起

先来解决第二个问题，与其把文件一个个加上去，为什么不把整个目录在本地准备好，直接加入目录岂不更好。

在Jenkins运行时，它本身就是把运行的配置和其他数据都放在`$JENKINS_HOME`下，我们就顺势建立一个对应的`JENKINS_HOME`存放这些数据。具体的内容参见[What’s in the Jenkins Home Directory?](http://jenkins-le-guide-complet.batmat.cloudbees.net/html/sec-hudson-home-directory-contents.html)。

常用的如下：

* `$JENKINS_HOME/jobs`        # 各个作业的配置和历史记录，一般拷贝`config.xml`即可
* `$JENKINS_HOME/config.xml`  # 这是jenkins全局配置文件，如从属节点的信息

按照`JENKINS`的目录结构，把`config.xml`放在作业下面，顺势把启动脚本也搁在里面，这样`Dockerfile`看上去及其清爽

	$ find JENKINS_HOME/
	JENKINS_HOME/
	JENKINS_HOME/jobs
	JENKINS_HOME/jobs/craft
	JENKINS_HOME/jobs/craft/config.xml
	JENKINS_HOME/start.sh

升级版的`Dockerfile`如下：

	# ADD JENKINS_HOME 
	ADD JENKINS_HOME $JENKINS_HOME
	RUN chmod +x $JENKINS_HOME/start.sh
	
	EXPOSE 8080
	
	CMD [ "/opt/jenkins/data/start.sh" ]

试着`docker build .`再运行一下，结果和上次博客中的一样，v5。

### 调试时，共享目录来传递运行数据构建JENKINS_HOME目录

`JENKINS_HOME`下的数据现在还是上次博客中介绍的，怎么才能最方便的准备好这个目录下的内容呢？

第一个想到的办法就是在容器里面启动`ssh`服务，然后`scp`拷贝出来，这样会“污染”docker镜像（有多余的内容）。

现在介绍一种办法不用`ssh`登陆把运行中的`$JENKINS_HOME`下的数据提取出来。（如果你有其他好办法，请告知）

这儿用到docker的两个技术：

1. 通过docker run中的`--volume/-v`参数使得容器中的内容能够共享和暴露在外面主机。
2. 结合docker 1.3开始支持的`docker exec`可以让我们直接登陆到运行中的容器而无需`ssh`或者[nsenter](https://github.com/jpetazzo/nsenter)命令，在容器中把需要的数据拷贝到共享目录中。

    $ docker build -t larrycai/jenkins-demo2 .
    $ mkdir -p ~/jenkins

如上，构建好docker镜像，建立本地目录来共享。

    $ docker run -v ~/jenkins:/data -P larrycai/jenkins-demo2

启动刚做好的镜像`larrycai/jenkins-demo2`，并且在前台运行方便查看输出的日志，再另一个终端看看容器的Id。

    $ docker ps # 得到容器的ID
	docker@boot2docker:~$ docker ps
	CONTAINER ID        IMAGE                           COMMAND                CREATED             STATUS              PORTS                     NAMES
	f20b2f3c09eb        larrycai/jenkins-demo2:latest   "/opt/jenkins/data/s   2 minutes ago       Up 2 minutes        0.0.0.0:49154->8080/tcp   backstabbing_mccarthy

现在就可以进入容器看看`$JENKINS_HOME`下的数据。

    $ docker exec -it f20b2f3c09eb bash #登入容器，你要替换你自己启动的容器Id
	root@f20b2f3c09eb:/# cd $JENKINS_HOME
	root@f20b2f3c09eb:/opt/jenkins/data# ls
	hudson.model.UpdateCenter.xml   nodeMonitors.xml          secrets
	hudson.plugins.git.GitTool.xml  plugins                   start.sh
	identity.key.enc                secret.key                userContent
	jobs                            secret.key.not-so-secret  war

现在容器中的`/data`目录就是docker主机上的`$HOME/jenkins`目录。

## 创建第二个作业和一个系统配置

活学活用，现在先在系统环境中设置变量`DOCKER_HOST`

![](http://larrycaiyu.com/images/jenkins-demo2-1.png)

然后创建第二个作业`craft2`，其中打印出环境变量

![](http://larrycaiyu.com/images/jenkins-demo2-2.png)

假设这就是我们要的演示环境，包括构建日志，我们可以查看一下容器内的数据，确认是我们需要的。

	root@f20b2f3c09eb:/opt/jenkins/data# find jobs
	jobs
	jobs/craft
	jobs/craft/nextBuildNumber
	jobs/craft/builds
	jobs/craft/builds/lastFailedBuild
	jobs/craft/builds/lastSuccessfulBuild
	jobs/craft/config.xml
	jobs/craft2
	jobs/craft2/lastStable
	jobs/craft2/config.xml
	jobs/craft2/workspace
	jobs/craft2/lastSuccessful
	jobs/craft2/nextBuildNumber
	jobs/craft2/builds
	jobs/craft2/builds/lastStableBuild
	jobs/craft2/builds/lastUnstableBuild
	jobs/craft2/builds/1
	jobs/craft2/builds/lastUnsuccessfulBuild
	jobs/craft2/builds/lastFailedBuild
	jobs/craft2/builds/lastSuccessfulBuild
	jobs/craft2/builds/2014-11-16_18-00-52
	jobs/craft2/builds/2014-11-16_18-00-52/build.xml
	jobs/craft2/builds/2014-11-16_18-00-52/log
	jobs/craft2/builds/2014-11-16_18-00-52/changelog.xml
	root@f20b2f3c09eb:/opt/jenkins/data# cat config.xml #显示部分内容
    ..
    <hudson.slaves.EnvironmentVariablesNodeProperty>
      <envVars serialization="custom">
        <unserializable-parents/>
        <tree-map>
          <default>
            <comparator class="hudson.util.CaseInsensitiveComparator"/>
          </default>
          <int>2</int>
          <string></string>
          <string></string>
          <string>DOCKER_HOST</string>
          <string>unix:///docker.sock</string>
        </tree-map>
      </envVars>
    </hudson.slaves.EnvironmentVariablesNodeProperty>
    ...

我们就在docker容器中可以把这些需要拷贝到`/data`目录，它对应的就是主机的`$HOME/jenkins`目录。

	root@f20b2f3c09eb:/opt/jenkins/data# cp -rf jobs /data
	root@f20b2f3c09eb:/opt/jenkins/data# cp -rf config.xml /data 
   
然后在主机docker镜像构建目录中再把它们拷过去。

	docker@boot2docker:~/git/jenkins-demo2/JENKINS_HOME$ cp ~/jenkins/jobs . 
	docker@boot2docker:~/git/jenkins-demo2/JENKINS_HOME$ cp ~/jenkins/config.xml .

最后重新构建并运行，发现新的内容也打包在里面了，bingo！

一切验证通过后，就可以上传你的Dockerfile自动产生出镜像分享给大家了。 

## 摘要

在这篇博客中，我们演示了如何更进一步docker化你的Jenkins应用程序，学到了如何管理配置文件和巧妙地使用`docker run --volume`和`docker exec`来获取容器内的数据。演示的内容也增加了。

所有的代码你都可以在[github上的jenkins-demo2](https://github.com/larrycai/docker-images/tree/master/jenkins-demo2)上找到。

在接下来的博客中，我将展示如何更好地用docker来做为jenkins的从属节点，并且简单发布。

Docker可以帮助我们做很多事情，关注新浪微博 [@larrycaiyu](http://weibo.com/larrycaiyu) [@Docker中文社区](http://weibo.com/dockboard) [@infoq的docker专栏](http://www.infoq.com/cn/dockers)
