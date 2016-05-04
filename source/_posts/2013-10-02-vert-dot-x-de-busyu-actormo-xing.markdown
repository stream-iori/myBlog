---
layout: post
title: "Vert.x 的Bus与Actor模型"
date: 2013-10-02 20:25
comments: true
categories: 
- Vert.x
- Asynchronize programming
published: true

---

Vert.x 框架自身的最大的卖点就是基于消息传输的Actor模型.理解Vert.x消息传输与Actor实现,对使用
Vert.x框架非常重要.

## Actor 模型 ##
所谓Actor,就是将目标看成一个个独立的个体,每个Actor接受到消息后,单独的做处理,处理完后,可以再发给其他的
Actor.所以你会发现Actor之间的耦合性非常非常的低,只依赖消息,映射到面向对象体系里只是依赖参数调用.

刚才提到面向对象,这里尝试做个简单的对比.人开车这个模型怎么建立.
OOP思想里会分析出两个基本的类,Person, Car. 开车这个行为由Person触发,调用Car的启动这个方法.
于是Car要暴露出至少一个启动接口给Person.这里产生了比较弱的耦合,因为两个类的生命周期或不影响.
如果这个时候Person要打电话呢,抽个烟呢.虽然Phone Cigarette也是单独的类,但是他们必须存在Person
类里面,或者说Person类 has Phone Cigarette and Car.同时你还要关注相关接口的名称与参数.

Actor模型相比较而言会简单一点,Actor之间不存在方法调用,只存在消息传输,简单点讲,定义个JSON格式的数据发送
过去就可以了.你不需要关注其执行成功失败,消耗多久,只需静等消息返回就可以了.期间你可以继续发消息给其他的Actor,
这样可以将对象并发之间的同步,锁给避免掉.

业界最有名的Actor实现者就是Erlang,天然分布式,高容错,热部署,这些都在语言级别上解决了.当然如果你能接受其
怪异的语法,你可以尝试一下Erlang.

## Vert.x的Bus ##
Vert.x里的Actor其实就是每一个Verticle实例.当你用Java继承了Verticle对象,并完成了其相关的逻辑代码的时候
你就定义好了一个Actor in Vert.x 那么怎么处理接受消息呢.

当当当,EventBus登场.EventBus是Vert.x里的神经系统.负责消息在不同的Vert.x实例之间传输消息.

### Bus地址 ###
每一个Bus都必须有一个地址,在Erlang用PID标识.在Vert.x里地址就是一个可以用来标识唯一性的字符串.
可以是UUID,也可以是自己定义的格式,或者类似Java package的命名空间.

### Handlers ###
一个地址绑定到一个Handler接口上,用于接受消息.多个Handler可以注册到不同的verticles上,单个handler
也可以绑定多个地址.你可以简单的理解出单多单(一对一通信),或者单对多(发布消息)

### 发布与P2P ###
EventBus支持P2P发送消息,即使一个地址注册了多个Handler,Vert.x也只Round Robin其中一个来处理.
P2P后你也可以在发送方注册一个reply Handler来处理回复的消息,这个Handler是可选的.只存在消息会回复的情况下.
当然处理完回复的消息,你还可以在send回去一个消息.这个场景非常类似Request-Response pattern.典型的就是HTTP协议.

广播/发布消息也可以通过EventBus 来实现,你只需要指定一个地址,所有的绑定在这个地址上的Handler都会收到这个消息.
这点非常类似发布/订阅模式

Vert.x内置几种消息类型,默认是不支持Java对象的,建议传输格式采用Json对象.**主要是为了支持其他的弱类型语言**

### EventBus API ###
这里不想大段大段的贴代码.各位如果有兴趣可以自己看官方的文档与例子
[EventBus Java API](http://vertx.io/core_manual_java.html#event-bus-api)


## 集群与分布式 ##
刚才一开始就提到了Actor可以避免显示的锁,支持分布式.如果EventBus不能分布式,那也太没意思了.
Vert.x实例本身是可以运行在不同的JVM里的,所以你可以在不同的JVM启动Vert.x实例,然后注册Handler到指定的Address上.
接受处理消息,只需要在启动的时候加个 `-cluster` 参数即可.

比如你机器A运行 `vertx run handler.js -cluster -cluster-port 25501 -cluster-host 192.168.1.2`
机器B运行 `vertx run sender.js -cluster -cluster-port 25502 -cluster-host 192.168.1.3`
这样会在两台机器上启动两个vert.x实例同时运行两个verticle actor来处理逻辑.
具体如何实现跨JVM通信,留到以后vert.x源码分析的时候再讲.

具体的命令行可以参考官方文档 [Vert.x 命令行](http://vertx.io/manual.html#using-vertx-from-the-command-line)


























