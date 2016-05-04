---
layout: post
title: "Vert.x模块的基本概念"
date: 2013-10-03 07:47
comments: true
categories:
- Vert.x
- Asynchronize programming
published: true

---

前面在介绍里就已经提到Vert.x是一个微内核,可插拔的设计架构.对外只提供一些标准的API,通过Vert.x的
Platform来加载其他的Verticle,以及相关的模块.这种低耦合的设计可以方便将你的模块随意抽象,也可以注册发布到
公网上模块仓库提供给其他的开发者使用.

## 什么Vert.x的模块 ##
所谓Vert.x模块就是将你常用的,或者规划好verticle代码打包起来,这里的打包起来是打包成vert.x模块指定的目录格式.这样的包格式
为vert.x模块,一般用mod来表示.创建一个vert.x模块有哪些好处呢.

* classpath被vert.x环境自动装载.你不需要手动的设置其classpath.
* 所有的模块依赖都打包单个文件里,比如zip.
* vert.x模块支持Maven仓库,这样你可以从Maven仓库里获取你的模块.
* 模块可以发布到vert.x的模块注册中心里提供给其他人使用.完善的生态圈
* vert.x容器可以自动的下载并安装这些模块.(支持从maven仓库里或者远程二进制文件)

基于以上的意见,强烈推荐将你应用包装成vert.x模块,而不是简单的verticles应用.(通过vertx run运行)

## 模块结构 ##
一个vert.x 模块就是一个zip文件,里面包含一些必须的文件(.class 文件, java文件 或者对应的语言脚本文件, lib目录下的jar资源,以及其他资源文件).
这点特别类似Jar文件结构.但是没有将vert.x做成与Jar文件结构一样的主要原因就是因为,Vert.x是一个多语言的异步开发框架.设想你的项目如果完全使用
Ruby或者Python脚本来运行,你又何必打包成Jar,这么奇怪的格式呢.

### 模块描述文件 mod.json ###
与主流的容器差不多比如Node.js,都是通过一个json文件来描述项目配置.
所有的vert.x模块都必须包含一个mod.json文件在模块的根目录下面.注意,这并不是标准的JS文件,注释只支持C形式的单行注释.
下面来看看mod.json包含哪些字段.

* main
main如同你所理解的一样,程序的入口,这里指定一个模块启动的时候运行的类或者脚本文件.给一些简单的例子.

`"main": "org.mycompany.mymod.MyMain"` Java Class完整类名.

`"main": "app.js"` 指定一个JS文件

`"main": "somedir/starter.rb"` 指定一个ruby文件

默认Vert.x支持的语言有JS Java Groovy Ruby Python.这些都配置在vert.x home目录里的conf下,你会发现里面
有以个lang.properties文件,如果要支持其他语言比如Clojure.必须先添加.

* worker
如果模块被定义为worker,那就必须设置这个字段为true.默认是false.
这里再提醒一下Worker与非Worker的区别. Worker的运行是用的阻塞方式,所有的请求都会按顺序,一个一个的被执行.
标准的verticles则是用的事件线程.非阻塞方式.
`"worker":true`

* multi-threaded
因为worker默认一次只有一线程在跑,如果你恰好是一个可以支持多并发的应用,比如JDBC,其底层本身是支持连接池的方式来连接数据库.
这个时候你可以设置该worker为多线程应用,开启这个功能要非常小心,设置worker instances数目最好与线程池大小一致.

* includes
一个模块可以在包含其他模块,这里的包含意味着被包含的模块的classloaders指向了此模块.这样被包含模块的资源文件以及class文件可以从包含模块访问.
类似于Java的包导入.
`includes`内容可以用逗号隔开
`"includes": "io.vertx~some-module~1.1,org.aardvarks~foo-mod~3.21-beta1"`

* preserve-cwd
默认模块读取资源文件都是从模块的根目录读取.举个栗子.
你的模块有如下目录结构
    /mod.json
    /server.js
    /web/index.html
按要求返回目标html内容.下面是server.js的内容
{% codeblock simple http server - server.js %}
var vertx = require('vertx')
vertx.createHttpServer().requestHandler(function(req) {
   req.response.sendFile('./web/index.html'); // Always serve the index page
}).listen(8080, 'foo.com')
{% endcodeblock %}
默认的`.`可以访问到模块的根目录,但是如果你希望访问的文件不在模块根目录下,则可以设置`preserver-cwd`为`false`.
默认是`false`

* auto-redeploy
设置此参数为`true`的话,Vert.x将自动重新部署模块,如果发现模块资源文件发生变动.
典型的场景就是Web项目,改个CSS, JS文件是很平常的事情.当然静态文件不用Vert.x也可以实现.如果是java class文件,就比较麻烦了.
Vert.x模块重部署会自动监视相关的classpath路径是否发生改变,如果发生改变则自动重新部署.

此外它对IDE以及项目编译工具支持都很好.**需要安装相关的构建工具插件**
你可以通过gradle或者maven生成对应的vert.x项目目录.
`./gradlew runModEclipse` Eclipse下使用gradle
`./gradlew runModIDEA` IDEA下使用gradle

`mvn vertx:runModEclipse` Eclipse下使用maven
`mvn vertx:runModIDEA` IDEA下使用Maven

具体的可以查看官方介绍[auto-redeploy](http://vertx.io/dev_guide.html#auto-redeploy)

* resident
通常Vert.x模块对其他模块的引用都是在内存里,即加载模块到内存里,当此模块不再被引用的时候从内存里则卸载此模块.但是对于特别的模块,比如相关语言实现的模块,频繁的加载卸载会导致
Java堆溢出.(每一次特定语言实现都会被加载一次然后卸载一次,如果存在20个JS文件,那么就是20次,Java堆里可能有20份Rhino的class文件).resident如其名,常驻的意思,即加载完这个模块
后,一直不释放,直到Vert.x实例被终止.一般存在于语言实现上面才重用这个字段,默认是`false`.当然,我的项目里依赖Guice,等其他的Java共用包,我也会把那个模块设置成`"resident":true`

* system
默认模块的安装路径是在模块调用目录下的`mods`文件夹下,当然你可以设置`VERTX_MODS`环境变量来改变其默认设置.但是一些模块,比如语言实现模块,需要被其他模块共享,所以为了避免多次的下载
加载到不同的mods下,你可以配置`"system"":true`.这样就会在VERTX_HOME下的`sys-mod`存在该模块.(此目录下的模块,被所有的Vert.x应用共享),默认是`false`.

* 其他的字段
其他的字段眼睛一瞄你就应该知道什么意思,我就不说了.比如`description`, `licenses`, `author`, `homepage`等.

### 其他 ###
关于Vert.x 模块的类加载以及资源包含,嵌套模块等部分,下一篇将讲解.




















