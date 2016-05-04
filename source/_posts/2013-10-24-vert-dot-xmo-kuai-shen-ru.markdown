---
layout: post
title: "Vert.x模块深入"
date: 2013-10-24 17:18
comments: true
categories:
- Vert.x
- Asynchronize programming
published: true

---

最近小忙了一点,一直拖着没有总结Vert.x模块其他内容,今天在下班前应该可以写完模块这一部分的总结。

## 模块包含 ##
有时候你会发现几个模块之间用了共同的Jar文件,或者资源文件.比如我们常用的Apache Commons Lang等.这个时候多个模块,会在JVM里加载多份,导致很多重复的引用,而且会有可能导致JVM里的方法区爆满.这个时候推荐将这些资源文件打包到一个模块里.这种模块称为资源模块.然后在其他模块里使用`include`去包含他.我们举个栗子.
我们有一个`commons-lang3.jar`,这个jar包希望用在两个模块下面,比如`com.xx.module1-v1.0`和`com.xx.module2-v1.0`.为了包含这个`commons-lang3.jar`我们需要创建一个模块.比如`com.xx.share-v1.0`,在这个模块的目录下建立一个`lib`文件夹,然后把`commons-lang3.jar`放到这个目录里.看起来应该是

    |--com.xx.share-v1.0
    |----lib
    |------commons-lang3.jar
    
最后我们在`com.xx.module1-v1.0 与 module2-v1.0`里面包含这个模块就行啦.
`"includes":"com.xx.share-v1.0"`,具体关于`mod.json`可以看我上一篇文章.如果你要包含多个模块,记得用逗号隔开.这里还有一点需要注意的是,被包含的模块必须是`非运行模块`.啥叫非运行模块呢.很简单就是`mod.json`里没有`main`这个声明.典型的就是语言模块.
模块目录结构具体的部分可以查看官网目录资料[典型模块目录结构](http://vertx.io/mods_manual.html#examples)

## 模块类路径与模块类加载器 ##
这一块是Vert.x的比较复杂的部分,与很多修改类加载方式的框架差不多.Vert.x有自己的一套类加载管理方式.官网给的资料不一定是正确的,因为master线对类加载做了一些修改.

### 模块类路径 ###
每一个模块都有自己的类加载器。这个类加载器负责加载此模块跟目录下的所有文件,包括`jar`,`zip`,`xml`,`properties`等各种格式。根路径就是模块目录.注意一点。这些被加载的文件只包含在模块的类加载器里。其他模块不能访问。

### 模块类加载器 ###
模块有自己的类路径,加载类路径下的资源的时候依赖自己类加载器.因为vert.x启动的时候会给每一个模块一个类加载器,注意这里不管你部署多个模块相同的实例,都只有对应的一个模块类加载器。
`vertx runmod com.xx.module-v1.0 -instances 100`即是你这里声明100个实例,还是只有一个类加载器.这样也就意味着模块之间的类加载是互相隔离的.
同时你也不能使用静态变量在模块之间传输数据.但是这些静态变量还是可以在模块实例之间传输的.
另外你也可以对同一个模块设定版本号同时跑在多个Vert.x实例上.具体的场景就是模块动态替换.比如v1.0版本发现有一个Bug,这个时候你部署一个V2.0的版本.然后慢慢的等v1.0的模块不再被访问的时候,可以执行模块卸载命令卸载掉.

模块的加载器们是按层级分布的,一个模块加载器会含有0个或多个父类加载器.当一个模块加载资源或者文件的时候,模块加载器首先通过自己的类加载器来加载资源,如果加载不到,则会委托其父加载器去加载资源.这点恰好与标准的Java类加载机制相反.

如果所有父类模块都加载不到资源的话,那么Vert.x的Platform 类加载器会去搜索,Platform是Vert.x的启动者,他会在Vert.x的Home目录里去搜索.

一般而言直接通过命令行启动一个verticle,不会有其父模块类加载器.除非一个verticle被其一另一个模块部署,那么部署他的模块就是其父模块.这一点其实也非常好理解.被包者的都是子集.同时父加载器可以加载子集的资源.

## 启动模块 ##

### 从命令行 ###

`vertx runmod org.myorg~mymod~3.2`

直接跑zip包也可以
`vertx runzip my-mod-3.2.zip`

### 从程序里启动 ###
这里还是以JS作为例子

`vertx.deployModule('org.myorg~mymod~3.2');`

## 如何定位模块 ##
当尝试包含一个模块的时候,Vert.x会做以下几件事情.
* 尝试查找`mods`目录.
* `VERTX_MODS`环境变量所指定的目录.
* `sys-mods`目录.

如果以上都不存在,Vert.x 会尝试读取`conf`下的`repos.txt`文件.里面包含了一些路径.

    #本地Maven仓库
    mavenLocal:~/.m2/repository

    #Maven 公网
    maven:http://repo2.maven.org/maven2

    # Sonatype 仓库
    maven:http://oss.sonatype.org/content/repositories/snapshots

    # 一个二进制网站.你可以在这里发布二进制文件
    bintray:http://dl.bintray.com

当然你也可以添加自己公司的私服地址,这样就可以在局域网内分享了.

Vert.x模块有自己的文件格式定义,这与NodeJS的package.json一样.

    GroupID: com.mycompany
    ArtifactID: my-mod
    Version: 1.2-beta

上面的定义与Maven很像,它会生成一个Vert.x模块文件名:`com.mycompany~my-mod~1.2-beta`.这就是Vert.x模块唯一字符串.如果你想从公网上直接下载zip模块.你可以尝试使用`dl.bintray.com`.

## 其他命令 ##
另外还两个不怎么常用的vert.x模块命令

* vertx pulldeps <module_name> 拉下该模块所有的相关依赖
* vertx install <module_name> 在Vert.x容器启动前,先下载并安装该模块到`mods`目录
* vertx uninstall <module_name> 与上面的命令相反

关于Vert.x模块方面的东西,暂时先讲这么多.后期我会介绍几个Vert.x社区里特别流行的模块.


