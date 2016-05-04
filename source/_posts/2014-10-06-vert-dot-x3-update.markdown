---
layout: post
title: "Vert.x3 Update"
date: 2014-10-06 14:34:49 +0800
comments: true
categories:
- Vert.x
published: true

---

Vert.x新版本在3月份的时候社区里进行了功能上的讨论,5月份就开始动工,在9月底的时候基本核心功能以及API已经冻结.也就是说,目前基本上可以发布alpha版.Vert.x开发团队的整体效率还是挺高的.

那这牛X的货又多了哪些新玩意呢.我大致从官方wiki以及社区的邮件列表里得到下面的一些信息.贴出来分享一下.

首先底层库的更新,Hazelcast更新到3.2系列,Netty依旧是最新的4系列版本.
依赖JDK8,(这个必须的,没有lambad表达式,写callback简直就是折磨人).
构建工具换成了Maven.据说是为了更好的兼容非Java语言实现.

### 降低模块系统复杂度

Vert.x2的模块系统需要你定义一个mod.json文件,然后将项目按指定的目录结构打包,这样才能跑起来,V3版本开始取消这个限制,所有的模块都打包成标准的Jar包,然后Push到Maven仓库里,这样通过artifact来定位项目,这里与V2有一个明显的区别,你可以在编译期间依赖项目.而不是在运行期间.
另外一些非Java语言的Verticles也可以打包成jars.Vert.x团队也打算支持npm.估计会在3.0正式发布之前.
Vert.x团队现在推荐将项目打包成一个可执行的jar文件.(在V3版本已经取消Platform).这样你的Vertx项目实际上是从一个标准的Main方法启动的.可以得到更多的控制.

### 目标语言代码自动生成
在V2版本里,如果要实现一个非Java语言的API.你需要按照现有的JavaAPI去包装成目标语言.
目前Vert.x有7种语言的支持,这也意味着你要写7遍.如果API发生变动,那么一样涉及到的API要全部更新.
这个代价比较大,而且对开发人员来说比较痛苦.所以团队里现在提出使用模板生成API.

具体做法就是根据核心的JavaAPI通过MAEL2表达式来生成对应语言的源码.
目前Vert.x已经通过这种方法生成了JavaScript与Groovy语言.当然Redhat首推的语言Ceylon也有人在做.
[codegen项目地址](https://github.com/vert-x3/codegen)

PS:我估计Clojure要等到明年才会考虑发布Vert.x3的版本.

### 使用Maven作为构建工具
可能是因为Maven是其他构建工具的基石吧,Vert.x3已经切到Maven下面.当然你自己基于Vert.x的项目用什么构建工具无所谓.
比如我这里的Clojure就是使用Lein.但是不管是lein,还是gradle还是sbt都依赖Maven.我觉得切到Maven下面可以让更多的人上手项目.

### 只支持JDK8
这个没什么好说的.因为Vert.x3的特性就是异步回调,所以Lambdas非常适合.当你在Java8项目里使用Vert.x的时候.你会觉得整个开发过程更加的流程.另外一个使用Java8的原因是Nashorn.众所周知Rhino引擎是Java8之前版本推荐的JS引擎.因为年久失修,其性能与稳定性都很差.Mozila已不打算花精力去维护它了.现在Java8里默认包含了性能更好的Nashorn,有什么理由不用呢.

### 扁平化的类路径
以前部署verticles时,类路径是有层级的概念,即ModA包含ModB,那么ModB的jar以及资源可以通过ModA的ClassLoader加载到.
现在vertcle部署的时候采用扁平化的类路径(扁平化一直很火嘛).扁平化的类路径有一个明显的好处——做嵌入式或者Debug的时候会非常的方便,因为你的资源不再是允许时加载进来的.当然如果你希望保留V2那种层级加载(可以做一定的Class隔离)可以通过参数配置.

### 移除platform manager
PlatformManager这个API会被移除,所有部署相关的接口已经移到Vertx Interface里.现在部署一个Verticle非常easy.所以Vert.x3做嵌入式更简单.我估计我的V3项目都会直接从Main方法里起来.

### 简化Verticle工厂
现在没有了Platform.也没有了langs.properties这个文件.所有的Verticle加载方式都通过标准的service loader模式.另外Verticle工厂也可以通过代码注册或者取消注册(V3所有的参数都可以通过代码来注入)

### 内置Block警告
V3版本里多了一个人性化的功能,阻塞警告.玩过Vert.x都知道你经常会碰到Handler长时间不回调的问题.产生这个问题的原因有很多.也许是你的代码问题，也许是第三方库的问题.你没有办法定位具体的位置.现在V3会把EventLoop事件阻塞写到日志里.另外如果你长时间的阻塞worker线程,也会写到日志里.

### 集群动态管理
在V2版本里,你无法动态的设定hazelcast参数.现在V3里可以了,一切参数化,这样你可以动态的指定集群IP地址等参数,或者动态的加入到另一个集群里.

### 跨集群共享数据
V2里shareData只能在实例之间共享数据,而Vert.x3里则不一样了.通过HazleCast可以在跨物理机的实例之间共享数据.
同时还提供了分布式Lock,以及Counter.这些常用的分布式工具.V3整体又向前跨了一步.由于cluster是暴露的SPI.本人考虑在未来实现ZooKeeper版.

### 重写Http Client
V2的HTTP Client一个Host只能复用一个连接,这个在V3里得到了改进,团队把HTTP Client全部重写了一遍.
同时也优化了HTTP连接关闭与打开相关代码(这个在V2是个坑),并且优化了pipe和keep alive.

### 改进WebSocket API
V3改进了WebSocket API,现在可以发送并接受WebSocket frames.

### 改进SSL/TLS
V2版本里配置cerificates只能通过Java key stores.现在可以通过方式了,比如字符串与Buffer.这点与Node.js很像
同时也支持特定的加密方式,甚至废除Key.


### EventBus 改进
EventBus改进算是比较大了,现在通过一个Registration对象来管理注册的Bus地址,同时也通过这个对象去UnRegister掉.
最激动人心的莫过于提供EventBus消息序列化功能,通过自定义Codec可以很方便接入自己的消息解码与加码方法.而且不要求发送与接受的对象类型一致.
比如你可以send一个BSON对象,然后接受一个POJO.

### EventBus 代理
有一些场景,你希望暴露给用户的是接口,而不是Bus地址和接受参数的类型.这个时候使用EventBus proxy是最合适的.EventBus Proxy实质上是在EventBus基础了做了一层接口的封装.
这样调用端,直接依赖接口传递方法参数就可以了.这个我认为是杀手级的功能.

### 剥离一些项目使之成为ext扩展
之前在Vert.x2里面的RouterMactch以及SockJS之类的功能,现在都被提取出来放到`ext`项目里.这样vert.x核心jar包只有两个`core`,与`hazelcast-cluster`.
如果你想增强Vert.x的相关功能,可以在这个`ext`里按需选择.目前官方提供了`RouterMatcher`,`MongoService`,`ReactiveStreams`,`Rx`等.其余的会慢慢加进来.
下面会介绍两个个相关的ext.
[项目地址](https://github.com/vert-x3/vertx-examples)

#### MongoService
MongoService使用其中一个stack,使用mongodb提供的异步API.你可以以verticle方式直接部署.让后通过eventBus Proxy方式与之交互.

{% codeblock example.java%}

    void save(String collection, JsonObject document, WriteOptions options, Handler<AsyncResult<String>> resultHandler);

    void insert(String collection, JsonObject document, InsertOptions options, Handler<AsyncResult<String>> resultHandler);

{% endcodeblock  %}

可以看到上面提供的API,都是异步的,而且访问的方式也变了不需要通过Bus send对象.而是直接调用方法.
妈妈再也不要担心我写一堆的switch case了.

#### ReactiveStreams
ReactiveStreams是干什么的呢,其实你只要看到Reactive就知道是干什么的了.具体可以参考[官方文档](http://www.reactive-streams.org/)

### 使用Option类作为配置选项
默认V3所有的参数都是可以配置的,这里包括集群配置,以及Net配置等.所有的配置都是通过Option类作为配置参数
这也意外着很多接口会有一堆的setter方法.NetClient,NetServer,HttpClient等类会被彻底的移除.

### 简单的Vert.x3项目例子
最后官方给了一个简单的基于Maven的Vert.x3项目例子,有兴趣的可以去[看看](https://github.com/vert-x3/example-proj).这个例子使用了SockJS以及REST API.通过JS main verticle去启动一个基于Java的WebServer.非常easy.

##最后
Vert.x3官方没有给出准确的发布时间,我估计会在明年年初.个人会密切关注Vert.x动向,一定稳定了考虑迁到Vert.x3.到时候会分享更多的关于Vert.x3项目经验. :)









