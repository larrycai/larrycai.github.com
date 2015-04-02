---
layout: post
title: 用Docker镜像来学习RabbitMQ和Python客户端使用例子
---
# 介绍

这篇博客介绍如何使用官方的docker镜像来学习RabbitMQ和Python客户端的使用，中间碰到问题也比较容易寻求帮助和定位，非常方便。

我的环境是Windows 7下的boot2docker 1.5版本。

## RabbitMQ 服务器 ##
启动RabbitMQ docker容器，这里使用`rabbitmq:3-management`，如果用`rabbitmq:3`就没有管理界面（15672端口），不适合初学者。

	docker run -d -e RABBITMQ_NODENAME=rabbit --name rabbit -p 8080:15672 rabbitmq:3-management    

打开浏览器，查看界面

![rabbitmq-1](http://www.larrycaiyu.com/images/rabbitmq-docker-1.png)

## RabbitMQ Python客户端 ##

例子都来自[rabbit python的入门][rabbitmqtut]，这里不讲解了。

启动Python docker容器，同docker的name/link方式，访问rabbit容器通信

    docker run -v $PWD:/code -w /code --link=rabbit:rabbit -it python:2 bash
    
### 发送端 ###

安装 pika库，和运行`send.py`的例子（源程序在我的docker主机上编辑，共享进去）。

<pre>
root@7848479ed0d2:/code# pip install pika
root@7848479ed0d2:/code# ./send.py
 [x] Sent 'Hello World!'
</pre>

你可以在管理界面上看到消息已收到。

源程序来自[rabbit python的入门][rabbitmqtut]，RabbitMQ主机名从`localhost`改为`rabbit`

<pre>
#!/usr/bin/env python
import pika

connection = pika.BlockingConnection(pika.ConnectionParameters(
        host='rabbit'))
channel = connection.channel()

channel.queue_declare(queue='hello')

channel.basic_publish(exchange='',
                      routing_key='hello',
                      body='Hello World!')
print " [x] Sent 'Hello World!'"
connection.close()
</pre>
    
### 接受端 ###

你可以用`docker exec`命令进入发送端的python容器，也可以再启动一个。

运行`receive.py`，前面的消息就收到了。

<pre>
root@7848479ed0d2:/code# ./receive.py
 [*] Waiting for messages. To exit press CTRL+C
 [x] Received 'Hello World!'
</pre>

源程序来自[rabbit python的入门][rabbitmqtut]：

<pre>
#!/usr/bin/env python
import pika

connection = pika.BlockingConnection(pika.ConnectionParameters(
        host='rabbit'))
channel = connection.channel()

channel.queue_declare(queue='hello')

print ' [*] Waiting for messages. To exit press CTRL+C'

def callback(ch, method, properties, body):
    print " [x] Received %r" % (body,)

channel.basic_consume(callback,
                      queue='hello',
                      no_ack=True)

channel.start_consuming()
</pre>


## 碰到的问题 ##

### 收不到消息 ###

一开始`receive.py`程序运行后，收不到消息（但连接正常 `accepting AMQP connection ...`）。

<pre>
$ docker logs -f rabbit

=INFO REPORT==== 31-Mar-2015::16:59:38 ===
Server startup complete; 0 plugins started.

=INFO REPORT==== 31-Mar-2015::17:00:57 ===
accepting AMQP connection <0.311.0> (172.17.0.17:35606 -> 172.17.0.16:5672)
</pre>

以为这个程序错了，后来启动admin管理界面，在界面上发送消息后，立马就收到了。

### 发送不了消息 ###

`send.py`程序运行后，正常退出，一开始以为是发送成功了，后来发现接受正确后，继续研究。

突然发现界面上面有红色的，提示空间不够

![rabbitmq-2](http://www.larrycaiyu.com/images/rabbitmq-docker-2.png)

再检查，果然是boot2docker虚拟机中的空间又不够了，而且也发现在[rabbit python的入门][rabbitmqtut]中最后提到了这一点

![rabbitmq-3](http://www.larrycaiyu.com/images/rabbitmq-docker-3.png)

# 总结 #

用docker特别方便学东西，容易架设。而且很容易重现，当我发现其他人运行正确后，我能确保不是docker镜像和网络通信的问题。

# 参考 #

* https://www.rabbitmq.com/tutorials/tutorial-one-python.html 
* https://registry.hub.docker.com/_/rabbitmq/ 

[rabbitmqtut]: https://www.rabbitmq.com/tutorials/tutorial-one-python.html