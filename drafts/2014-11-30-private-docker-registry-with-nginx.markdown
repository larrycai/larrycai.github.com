---
layout: post
title: 用Nginx来做私有docker registry的安全控制 
---

## 介绍

docker registry就是管理docker镜像的仓库，Docker公司维护的registry就是[http://hub.docker.com](http://hub.docker.com) ，它可以让我们方便的下载别人预先做好的镜像。

    $ docker pull ubuntu

上面的命令就是缺省的从这个源下载。在国内为了加快访问，你可以使用 [docker.cn](http://docker.cn)的服务，他们同步了常用的镜像，使用也非常方便，如：

    $ docker pull docker.cn/docker/ubuntu

大部分公司在推广使用docker时，都会为了使用方便，在公司内部自己架设一个，不仅仅是为了安全，也可以节省大量的带宽。

Docker公司的docker registry也是开源的，我们可以很容易的架设自己的私有docker registry，启动时典型的docker方式，就是：

    $ docker run -d -p 5000:5000 registry

更多的参数，请查看[github docker registry](https://github.com/docker/docker-registry)。

然后你就可以很容易的使用了，下面的命令就是把官方的`ubuntu`镜像放在私有的registry：

    $ docker tag ubuntu company.com:5000/ubuntu
    $ docker push company.com:5000/ubuntu

不过他有以下几个问题：

1. registry没有安全权限的设置，任何人都可以pull、push，这个基本上是不能接受的。
2. 十月发布的docker 1.3.x版本的发布有很多新的功能，但也强制基本鉴定（basic authentication）必须使用https，及其烦人。

这篇博客就把作者做的一些实验分享给大家，解释一下如何可以用nginx来怎么这些问题。（我刚学Nginx，有错误请指正），所有的实验都是在Windows boot2docker的环境下完成，其他docker环境应该也一样。

先会把成功的环境演示一下，后面再把其中的技术稍微解释一下。

以下的几个链接给我提供了很到的帮助和启发：

* [Building private Docker registry with basic authentication](https://medium.com/@deeeet/building-private-docker-registry-with-basic-authentication-with-self-signed-certificate-using-it-e6329085e612)
* [How To Set Up a Private Docker Registry on Ubuntu 14.04](https://www.digitalocean.com/community/tutorials/how-to-set-up-a-private-docker-registry-on-ubuntu-14-04)
* slideshare上的[docker registry basic auth](http://www.slideshare.net/Remotty/dobestan-docker-registryandbasicauth)

## 服务架构说明

让我们先来看看这个registry的服务是怎么组成的。

![](http://larrycaiyu.com/images/docker-registry1-1.png)

`dokk.co` 这是docker服务器的名字也就是你的公司docker私有服务器的地址，因为https的SSL证书不能用IP地址，我就随便找了个名字，做实验没有问题。

registry服务器作为上游服务器处理docker镜像的上传和下载，用的是官方的镜像。

nginx是一个最常见的反向代理服务器，通过模块提供https的ssl的认证和basic authentication，用的是我自己做的一个镜像[larrycai/nginx-auth-proxy](https://github.com/larrycai/nginx-auth-proxy)。

这里可以看到典型的互联网服务，nginx把这些https、认证服务搞定，registry关注docker的镜像就可以了。

可以很方便的启动这些服务，命令如下：

	docker run -d --name registry -p 5000:5000 registry
	docker run -d --name nginx --link registry:registry -p 443:443 larrycai/nginx-auth-proxy

命令稍加解释一下

启动的时候，我加了`--name registry`，这是docker的容器链接(link)技术，为了方便`nginx`容器的访问`registry`（对应`--link registry:registry`)，稍后还有解释。

`-p 5000:5000` `registry`作为上游服务器，这个`5000`端口可以不用映射出来，因为所有的外部访问都是通过前端的nginx来提供，`nginx`可以在私有网络访问`registry`。端口映射出来只是为了方便调试，确保查看`registry`是否独立也能工作。

`-p 443:443` 就是对外服务的https端口。

## 尝试 ##

多说无益，尝试一下，看看到底现在可以能做啥了。

### Registry服务 ###

先验证registry是否正常：

	docker pull hello-world # 这个hello world包很小，适合做实验
	docker tag hello-world localhost:5000/hello-world
    docker push localhost:5000/hello-world

结果很好

	docker@boot2docker:~$ docker push localhost:5000/hello-world
	The push refers to a repository [localhost:5000/hello-world] (len: 1)
	Sending image list
	Pushing repository localhost:5000/hello-world (1 tags)
	511136ea3c5a: Image successfully pushed
	7fa0dcdc88de: Image successfully pushed
	ef872312fe1b: Image successfully pushed
	Pushing tag for rev [ef872312fe1b] on {http://localhost:5000/v1/repositories/hello-world/tags/latest}

### docker的HTTPS服务 ###

现在看看nginx的https服务怎么样，先用curl命令。（`larrycai:passwd`是我预先放置的用户密码）

	curl -i -k https://larrycai:passwd@dokk.co

噢

	curl: (6) Couldn't resolve host 'dokk.co'

对了，这是自己杜撰的域名，需要在`/etc/hosts`的`localhost`后面加上，如下。

	docker@boot2docker:~$ cat /etc/hosts
	127.0.0.1 boot2docker localhost localhost.local dokk.co

再来一次

	docker@boot2docker:~$ curl -i -k https://larrycai:passwd@dokk.co
	HTTP/1.1 200 OK
	Server: nginx/1.6.2
	Date: Sat, 29 Nov 2014 09:21:18 GMT
	Content-Type: application/json
	Content-Length: 28
	Connection: keep-alive
	Expires: -1
	Pragma: no-cache
	Cache-Control: no-cache
	
	"\"docker-registry server\""

说明连通了`nginx`和`registry`。说句实话，以前我没玩过nginx，看到这么简单，还是有点小激动，这么容易。

实际上也打开浏览器访问 [https://192.168.59.103](https://192.168.59.103), 192.168.59.103是boot2docker的VM的IP地址(你可以换成你的docker机器的IP地址）

![](http://larrycaiyu.com/images/docker-registry1-2.png)

忽略他的安全告警后，输入`larrycai:passwd`，还是正确的访问到了上游的registry了。

### docker的login服务 ###

如果我们希望`docker push`需要访问控制的话，在docker中是通过`docker login`先来登陆的，成功后的配置信息存放在`~/.dockercfg`。

先来试试不登陆的情况：

	docker tag hello-world dokk.co/hello-world
    docker push dokk.co/hello-world
    
结果报错。

	docker@boot2docker:~$ docker push dokk.co/hello-world
	WARNING: Invalid Auth config file
	The push refers to a repository [dokk.co/hello-world] (len: 1)
	Sending image list

好吧，试试用docker登陆是否成功

	docker login -u larrycai -p passwd -e test@gmail.com dokk.co

OMG，证书它不相信。 

让我们把我做`dokk.co`的根证书[ca.pem](https://github.com/larrycai/nginx-auth-proxy/blob/master/)下载下来，再加入到boot2docker的证书中。当然为了演示方便，`ca.pem`已经放在容器中了，把它拷贝出来。(`docker copy`是个好命令，别忘了）

	$ docker copy nginx:/ca.pem $PWD
    $ sudo cat ca.pem >> /etc/ssl/certs/ca-certificates.crt

不好意思，证书是docker的daemon需要用到的，docker服务需要重启。先停掉服务是个好习惯。

    $ docker rm -f registry nginx
    $ sudo /etc/init.d/docker restart

然后再次运行registry和nginx服务

	docker run -d --name registry -p 5000:5000 registry
	docker run -d --hostname dokk.co --name nginx --link registry:registry -p 443:443 larrycai/nginx-auth-proxy
	docker login -u larrycai -p passwd -e test@gmail.com dokk.co

结果很棒！

	docker@boot2docker:~$ docker login -u larrycai -p passwd -e test@gmail.com dokk.co
	Login Succeeded

上传试试

	docker tag hello-world dokk.co/hello-world
    docker push dokk.co/hello-world

一切完美，测试结果

现在我们可以试着做一个negative案例，`~/.dockercfg`存放了login的信息，把他删掉（如果还有其他登陆信息，就只删除这一行）

    $ rm ~/.dockercfg
    $ docker push dokk.co/hello-world

结果当然出错。

	docker@boot2docker:~$ docker push dokk.co/hello-world
	WARNING: Invalid Auth config file
	The push refers to a repository [dokk.co/hello-world] (len: 1)
	Sending image list

## 技术讨论 ##

## 摘要

在这篇博客中，我们演示了如何用Nginx作为前端服务器来解决Docker 私有registry的安全认证问题，通过它，你应该可以架设一个可以使用的安全的私有regsitry。

所有的代码你都可以在[github上的nginx-auth-proxy](https://github.com/larrycai/nginx-auth-proxy)上找到。

当然现在常用的`pull`命令下载镜像也要认证了，在公司内部会非常讨厌，最好是可以匿名访问。而且一些出错处理也很简单，这些用nginx是比较容易实现的，有机会的话，我会再写博客讲解，如果你有好的方案的话，也请告知。

Docker可以帮助我们做很多事情，关注新浪微博 [@larrycaiyu](http://weibo.com/larrycaiyu) [@Docker中文社区](http://weibo.com/dockboard) [@infoq的docker专栏](http://www.infoq.com/cn/dockers)
