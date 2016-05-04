---
layout: post
title: "Vert.x介绍"
date: 2013-09-30 13:35
comments: true
categories:
- Vert.x
- Asynchronize programming
published: true

---

自去年使用[Vert.x](http://vertx.io)已经有一年多,从千疮百孔的1.3.1版本到现在2.0.1,其发展也算是有不小的进步.
使用Vert.x是由于一次项目的技术选型,因为需求里可能会遇到Web请求以及Socket请求,所以想找一个
同时能够支持这两种连接的框架,还希望保持轻巧,灵活,支持模块化的开发与定制,部署更新方便.最好能
天然支持分布式(类似Erlang的Actor模型).

于是乎,在经历长时间的google后,对比了各种框架,发现一个还属于襁褓中的Vert.x是一个比较有潜力,
且符合我胃口的框架,同时个人觉得深度挖掘一个不是很成熟的框架并与之一起成长是一件非常有意义的事情.

## Vert.x是神马 ##

按官方的解释 Polyglot, Simplicity, Scalability, Concurrency.

* Polyglot(多种语言支持)
简单点讲,就是只要是JVM上的主流语言,都可以直接编写基于Vert.x的应用.
目前官方推出的有Javascript, Groovy, Ruby, Python, 以及 Java本身.
另外处于beta阶段的 Clojure, Scala也在开发中.
本人有幸参与Clojure的部分移植.

* Simplicity(简单)
这里的简单意味着你编写的代码是完全基于异步事件的,类似Node.JS,与此同时.你不需要关注线程上的
同步,与锁之类的概念,所有的程序都是异步执行并且通信是无阻塞的.

* Scalability(扩展性)
因为基于Actor模型,所以你的程序都是一个点一个点的单独在跑,一群点可以组成一个服务,某个点都是可以
水平扩展,动态替换,这样你的程序,基本就可以达到无限制的水平扩展.

* Concurrency(并发性)
此并发并非JDK库的并发,当你Coding的时候,不再关注其锁,同步块,死锁之类的概念,你就可以随性所欲的写自己的业务逻辑了.Vert.x本身内置三种线程池帮你处理.

* 模块化
Vert.x本身内置强大的模块管理机制,当你写完一个Vert.x业务逻辑的时候,你可以将其打包成module,然后部署到基于Maven的仓库里,与现有主流的开发过程无缝结合.so easy...

* 分布式消息传输
Vert.x基于分布式Bus消息机制实现其Actor模型,简单点讲每个Vert.x实例都可以与其他节点内置的通信接口,底层上实现框架内部的消息同步与传输,类似Erlang的ping pong自检消息.
而我们的业务逻辑如果依赖其他Actor则通过Bus简单的讲消息发送出去就可以了.

* 支持WebSocket
支持WebSocket协议兼容SockJS.

* 可嵌入式使用
你可以不依赖Vert.x自身的系统,嵌入到现有的项目里.当然个人不推荐这么使用

* 与现有IDE无缝兼容
官方已经给出基于gradle与Maven的配置,你完全可以生成与之相关的项目结构.

## 基本概念 ##

### Verticle ###
基于Vert.x框架实现的代码包,就是一个Verticle,简单点说,一个可以被Vert.x框架执行的代码调用了
Vert.xAPI的代码就是一个Verticle.他可以用Scala Clojure JS Ruby等语言实现.
多个Verticle实例可以并行的被执行.一个基于Vert.x的服务也许需要多个verticles来实现,而且要部署在多台服务器上.
他们之间通过vert.x事件进行通信.你可以之间通过vert.x命令启动,也可以将verticle包装成vert.x modules.(推荐这么做)

### Module ###
Vert.x应用由一个或多个modules来实现.一个模块呢由多个verticles来实现.你可以把module想象出一个个Java package.
里面可能是特定业务的实现,或者公共的服务实现(那些可以重用的服务).Vert.x编写好的module,可以发布到maven的仓库里.
以zip包装成二进制格式.或者发布到vert.x module 注册中心.实际上这种以模块方式的开发,支撑着整个Vert.x生态系统.
Module更多的信息,我需要单独开一个系列来讲解.

### Vert.x 实例 ###
Verticles 其实是跑在 Vert.x实例上的.所谓Vert.x实例其实就是一个运行在某一个JVM上的Vert.x对象实例.
可以将多个Verticles运行在一个Vert.x实例上,而vert.x实例可以跑在单个JVM上,也可以跑在其他JVM上,通过分布式event bus
来维持通信.注意vert.x实例其实是由vertx命令行启动的.你可以指定实例数目在单个JVM上.

### Event Loops ###
上面提到Vert.x实例,每个Vert.x实例内部维持几个线程,线程数目基本与CPU核数一致.这些线程在Vert.x内部叫做事件循环(Event Loop)
这个思想在很多事件驱动的架构都有,典型的就是IOS事件,它操作系统内部也有一个事件监控线程,不停捕捉外部的事件,比如touch,多点触摸等.
然后分配到指定的处理函数上,在vert.x里这些处理函数是Handler接口.
在Vert.x里这些事件可以是从Socket里读到数据,或者是一个定时器触发,亦或是一个HTTP请求接受到.
一个部署好的verticle都会得到一个event loop,来处理相关的事件.相关的后续的处理都会在这个event loop解决掉(也就是一个线程里)
,注意在同一个时间里有且只有一个线程处理.即Handler接口里是线程同步的.这点非常类似 reactor pattern.

不要在Event Loops写一些阻塞代码,因此下面code不应该存在

- `Thread.sleep()`
- `Object.wait()`
- `CountDownLatch.await()`
而且如果是长时间的计算也不应该存在.更不可能发生长时间的IO堵塞,典型的就是JDBC查询

### 阻塞处理 ###
事件处理之外肯定会发生其长时间数据处理请求.比如处理一个图片上传,然后转存到磁盘上等.或者一次长时间的排序计算等.
在Verticle类型里,有一种特别的verticle叫做Worker.不同于标准的verticle,他不使用event loop.而是采用vert.x内部的
另一个线程池叫做worker pool.

其区别在于,worker verticles不会并行的执行Handler.而是阻塞式的,等待上一个Handler处理完了,才会再执行后面的请求.你可以
想象出队列的方式执行某些请求.

所以为了支持标准的与阻塞式的worker verticles, Vert.x 提供了一种混合线程模型,你可以选择适当的模型用于你的应用.
__worker verticle是一种阻塞式的方法,但是不可以做到并发水平扩展的__

### 共享数据 ###
消息通过Bus可以在各个Vert.x实例直接传输.但是如果多个Verticle在一个Vert.x实例内,是可以避免进行消息传输的.比如单个JVM内,你不会
通过Socket互相在两个Java 对象之间传输消息吧.但是因为实例隔离,因为Actor模型,所以对象数据如果要传到Handler里,必须通过消息传输.
Vert.x提供了一个简单的共享Map与Set来解决这个问题.数据被存储到一个不可变的数据结构了,各个实例直接通过此API获取数据.(看例子更容易)

### API ###
Vert.x本身由两块组成,Vert.x Platform与Core.
Platform提供相关的部署API,可以通过此类API动态的部署相关的Verticle实例.
相关API使用参考官方网站.[命令行操作Vert.x](http://vertx.io/manual.html#using-vertx-from-the-command-line)

Core API提供

- TCP/SSL servers and clients
- HTTP/HTTPS servers and clients
- WebSockets servers and clients
- The distributed event bus
- Periodic and one-off timers
- Buffers
- Flow control
- File-system access
- Shared map and sets
- Accessing configuration
- SockJS


## 安装 ##

Vert.x安装很简单,下载好binary文件,设置其Home即可.
注意: Vert.x依赖其JDK1.7

解压下载好的包
`tar -zxf ~/Downloads/vert.x-2.0.1-final.tar.gz`

检查版本
`vertx version`

## Hello World ##

我们跑一个简单的HTTP服务.这里使用JS语法.

{% codeblock simple http server - httpServer.js %}
    var vertx = require('vertx');
    vertx.createHttpServer().requestHandler(function(req) {
    req.response.end("Hello World!");
    }).listen(8080, 'localhost');
{% endcodeblock %}
保存为server.js然后run起来

`vertx run server.js`

打开浏览器 localhost:8080
Hello World
你可以在官网首页上,看到其他语言的实现.__选择自己喜欢的语言作为开发,是一件非常开心的事情.__

## Vert.x的缺点 ##
框架之所以称之为框架,是因为你一旦用了他,你就被框住了.一旦被框住,你也就被局限了.
所以,所有的框架都会存在其局限.就我使用一年多得经验来看.缺陷如下

* 异步编程模型
如果你是Node.js开发者,亦或高级前端重度Ajax使用者,你会发现其异步编程模型带来的弊端.
事务代码离散,大量的闭包.如果你使用JS作为Vert.x开发语言,你可以使用Q.js等其他promise
模式来简化异步编程带来的 callback hell问题,以及异步事件串联问题.
但如果你使用Java. 你会发现其无比痛苦.当然也有一些框架可以简化带来的弊端.以后会讲到.
**传统JAVA程序员很难接受其这种思想**

* 阻塞问题
Vert.x本身优势之一就是避免在代码里考虑阻塞.但是如果你不了解Vert.x的内部事件循环机制.
你是无法做到Handler的粒度控制.如果你的业务过于复杂,占用一个Handler时间过长,会导致其
业务的线性阻塞,所以编写Vert.x程序前,最好能估算其执行效率,充分的分离计算密集型请求,与
IO密集型请求.

* 与其他框架的关联
Vert.x对其他框架的集成不是很方便,比如Spring,Hibernate等这些传统的Java框架.Vert.x
本身的Classloader机制与他们冲突,如果想深度集成进来会比较麻烦.我个人使用Guice作为注入
框架.

* 基于Json数据传输协议
这个缺点其实仁者见仁智者见智,如果你是Java程序员,你肯定希望你代码里都是Java Bean,而不是
一堆字符串组织起来的Json.Vert.x内部如果在bus上发送消息且是夸语言的.最好能够走Json.虽然
字节流也支持,但是你不会在JS代码里在再转成Java Bean吧.个人实现一个基于Json的传输,但是在
Java里会转成其对应的Java Bean的模块.后期完善考虑开源.

## 总结 ##
总得来说,个人还是非常喜欢这个轻巧强大的框架.出色的底层架构,微内核可插拔的设计.可以
满足各种工程需求.个人偏向FP语言,因为FP语言特别适合Actor模型.Vert.x社区也是非常的热闹.
基本上你提的问题会立刻回答.当然你也可以直接问我.


























































