---
layout: post
title: 使用docker来提升你的Jenkins演示 - 1 
---
## 什么是Jenkins

Jenkins ([http://jenkins-ci.org](http://jenkins-ci.org/)) 几乎是CI（持续集成）的代名词。 它有一个大的社区，有很多插件一起提供了很强大的功能。为了学习这些东西，最好的办法将是通过设置环境来实践它。

此外如果你想给别人介绍jenkins的新功能，你很想为这些功能快速创建一个演示环境。

如何可以轻松地完成？ 我最喜欢用Docker来实现这一点。

![](http://larrycaiyu.com/images/jenkins-demo1-1.png)

在这个博客系列，我会用一些例子一步一步来说明如何实现这一目标。

## 演示Jenkins的小功能 - AnsiColor插件

[Jenkins AnsiColor plugin](https://wiki.jenkins-ci.org/display/JENKINS/AnsiColor+Plugin) 是我最喜欢的小插件之一，它可以把你控制台日志看起来更好。

![](http://larrycaiyu.com/images/jenkins-demo1-2.png)

所以我想给大家一个演示环境来可以尝试，而无需在本地Jenkins上安装。我一般推荐在正式部署之前尝试一下。

### 结果

让我们来立即来看看效果，或许你也可能只是对这个功能感兴趣。

    docker run –p 8080:8080 –t larrycai/jenkins-demo1

![](http://larrycaiyu.com/images/jenkins-demo1-3.png)

在控制台窗口中上Jenkins已经被启动，然后可以打开浏览器访问`8080`端口。

![](http://larrycaiyu.com/images/jenkins-demo1-4.png)

看起来相当不错，一个叫`craft`的任务（job）已经存在了，Jenkins显示是最新的LTS版本1.580.1

点击`craft`任务，并运行它，然后检查`console`。太棒了，部分结果可以有颜色显示了，这就是我们要的。

![](http://larrycaiyu.com/images/jenkins-demo1-5.png)

然后回过头来看看它是如何配置。

![](http://larrycaiyu.com/images/jenkins-demo1-6.png)

现在演示完毕，可以学习怎么做到的。

### 它是如何工作的

这里是`Dockerfile`, 参见[github上的源代码](https://github.com/larrycai/docker-images/blob/master/jenkins-demo1/Dockerfile)

	FROM ubuntu:trusty
	
	MAINTAINER Larry Cai <larry.caiyu@gmail.com>
	
	ENV REFRESHED_AT 2014-11-03
	
	RUN apt-get update  && apt-get install -qqy curl openjdk-6-jdk
	
	ENV JENKINS_HOME /opt/jenkins/data
	ENV JENKINS_MIRROR http://mirrors.jenkins-ci.org
	
	# install jenkins.war and plugins
	
	RUN mkdir -p $JENKINS_HOME/plugins $JENKINS_HOME/jobs/craft
	RUN curl -sf -o /opt/jenkins/jenkins.war -L $JENKINS_MIRROR/war-stable/latest/jenkins.war
	
	RUN for plugin in chucknorris greenballs scm-api git-client ansicolor description-setter \
	    envinject job-exporter git ws-cleanup ;\
	    do curl -sf -o $JENKINS_HOME/plugins/${plugin}.hpi \
	       -L $JENKINS_MIRROR/plugins/${plugin}/latest/${plugin}.hpi ; done
	
	# ADD sample job craft
	
	ADD craft-config.xml $JENKINS_HOME/jobs/craft/config.xml
	
	# start script
	
	ADD ./start.sh /usr/local/bin/start.sh
	RUN chmod +x /usr/local/bin/start.sh
	
	EXPOSE 8080
	
	CMD [ "/usr/local/bin/start.sh" ]

开始安装`openjdk`/`curl`包，并设置Jenkins启动时所需要的相关环境。

Jenkins 的应用程序 (`.war`)可以在[http://jenkins-ci.org/](http://jenkins-ci.org/)找到, 你可以选择最新的版本或LTS（长期支持）版本稳定，这里我选择LTS版本[http://mirrors.jenkins-ci.org/war-stable/latest/jenkins.war](http://mirrors.jenkins-ci.org/war-stable/latest/jenkins.war)

所有的插件可以在镜像站点上找到：[http://mirrors.jenkins-ci.org/](http://mirrors.jenkins-ci.org/), 你需要找到你的插件Plugin Id像`ansicolor`，它会映射到[http://mirrors.jenkins-ci.org/plugins/ansicolor/latest/ansicolor.hpi](http://mirrors.jenkins-ci.org/plugins/ansicolor/latest/ansicolor.hpi)

![](http://larrycaiyu.com/images/jenkins-demo1-7.png)

在jenkins中，任务的配置保存为`config.xml`。这里我们提前做好了，把它放在docker镜像里的`$JENKINS_HOME/jobs/craft`目录下。

	<?xml version='1.0' encoding='UTF-8'?>
	<project>
	  <actions/>
	  <description></description>
	  <keepDependencies>false</keepDependencies>
	  <properties/>
	  <scm class="hudson.scm.NullSCM"/>
	  <canRoam>true</canRoam>
	  <disabled>false</disabled>
	  <blockBuildWhenDownstreamBuilding>false</blockBuildWhenDownstreamBuilding>
	  <blockBuildWhenUpstreamBuilding>false</blockBuildWhenUpstreamBuilding>
	  <triggers/>
	  <concurrentBuild>false</concurrentBuild>
	  <builders>
	    <hudson.tasks.Shell>
	      <command>#!/bin/bash
	env
	echo -e "\e[1;31;42m Using docker to demo is awful, v5 \e[0m"
	echo see more in http://misc.flogisoft.com/bash/tip_colors_and_formatting
	</command>
	    </hudson.tasks.Shell>
	  </builders>
	  <publishers/>
	  <buildWrappers>
	    <hudson.plugins.ansicolor.AnsiColorBuildWrapper plugin="ansicolor@0.4.0">
	       <colorMapName>xterm</colorMapName>
	    </hudson.plugins.ansicolor.AnsiColorBuildWrapper>
	  </buildWrappers>
	</project>

最简单的方法是直接从运行的Jenkins得到这个`config.xml`文件（在你的任务URL后面追加`config.xml`即可）。 

![](http://larrycaiyu.com/images/jenkins-demo1-8.png)

而在最后，加上一个小脚本`start.sh`，它将在启动时启动Jenkins。

    exec java -jar /opt/jenkins/jenkins.war

然后，你可以建立自己的docker镜像了，就这么简单。

    docker build –t larrycai/jenkins-demo1 .

### 如何公开分享

你可以把你的项目放到[github](http://github.com)上或者[bitbucket](http://bitbucket.com) ，并在[http://hub.docker.com](http://hub.docker.com/)运行在您的构建 ，然后其他人可以简单的运行docker的命令来运行它（您可以自己搜索具体怎么做）。

![](http://larrycaiyu.com/images/jenkins-demo1-9.png)

## 摘要

在这篇博客中，我们演示了如何dockerize你的Jenkins应用程序，它包含了必须的插件和配置和实例任务。 这将会很容易让你的听众了解你想演示的功能。

所有的代码你都可以在[github上的jenkins-demo1](https://github.com/larrycai/docker-images/tree/master/jenkins-demo1)上找到。

现在，您可以把您的漂亮的Jenkins新功能打包到Docker到处演示。

在接下来的博客中，我将展示如何更好地组织Jenkins目录。

Docker可以帮助我们做很多事情。
