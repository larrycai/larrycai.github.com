# 介绍

使用官方的docker image来学习 RabbitMQ和Python客户端的使用

参考： https://www.rabbitmq.com/tutorials/tutorial-one-python.html 

启动RabbitMQ

    docker run -d -P  -e RABBITMQ_NODENAME=rabbit --hostname rabbit --name rabbit rabbitmq:3

启动Python

    docker run -v $PWD:/code -w /code --link=rabbit:rabbit -it python:2 bash
    # pip install pika==0.9.8
    # # 安装最新版有问题
    
安装 pika库，和运行`send.py`的例子，没问题

<pre>
root@7848479ed0d2:/code# pip install pika==0.9.8

Collecting pika==0.9.8
  Downloading pika-0.9.8.tar.gz (56kB)
    100% |################################| 57kB 1.0MB/s
Installing collected packages: pika
  Running setup.py install for pika
Successfully installed pika-0.9.8
root@7848479ed0d2:/code# ./send.py
 [x] Sent 'Hello World!'
</pre>
    
但是运行`receive.py`时，它就挂死在那里了

<pre>
root@7848479ed0d2:/code# ./receive.py
 [*] Waiting for messages. To exit press CTRL+C
</pre>

观察RabbitMQ docker容器正常

<pre>
$ docker logs -f rabbit

=INFO REPORT==== 31-Mar-2015::16:59:38 ===
Server startup complete; 0 plugins started.

=INFO REPORT==== 31-Mar-2015::17:00:57 ===
accepting AMQP connection <0.311.0> (172.17.0.17:35606 -> 172.17.0.16:5672)

=INFO REPORT==== 31-Mar-2015::17:03:23 ===
accepting AMQP connection <0.361.0> (172.17.0.17:35613 -> 172.17.0.16:5672)
</pre>

