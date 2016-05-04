---
layout: post
title: "让Storm飞起来"
date: 2013-11-11 09:11
comments: true
categories:
- Storm
published: true

---

一年前就开始关注Storm,不过当时因为其他事情一直没有机会深入了解Storm,当时也在使用ZeroMQ发现很容易造成泄漏问题,而且ZeroMQ开始出现版本
分裂(ZeroMQ 3系列好像是需要收费的),就没这么研究下去,精力全都去配置Emacs,学Clojure去了.
最近故地重游,发现Yahoo对Storm做了点贡献,传输层可插拔,添加了Netty的支持,很Nice.这里有一遍对替换后的性能测试文章.

目前看来对Java开发者而言,用Netty替换ZeroMQ最大的好处是,取消了对原生的库的依赖,做了真正的跨平台,测试与部署开发方便性得到了极大的提高.
但是,最最让人Happy的是性能提高了两倍,消息传输速度提高了两倍.使对带宽利用更加充分.

### 测试

Yahoo内部已经使用Netty代替ZeroMQ作为消息传输,但是在使用之前他们自己写了一个直接让ZeroMQ达到极限的[例子](https://github.com/yahoo/storm-perf-test).
这个例子充分的展现了Storm传输消息是多么的迅速.

这里一开始部署配置了一个非常小的拓扑,3个workers,3个bolt和3个spout,然后慢慢增长到408worker,上千个bolt和spout消息的大小也是不一样的.
所有的测试中消息的ACK也是启用的,每个spout允许挂起1000个消息,因此这里也有一些流控系统.Yahoo做了多次测试，每次测试的浮动大约在5%.

### 结果

第一个测试是一个非常小得拓扑结果,3个worker工作在3个独立的节点上,所以这里没有资源竞争.每个worker有1个spout 1个bolt和1个acker tasker
从下面的图中可以看出,Netty在这种条件下性能完全高于zeroMQ,差不多高个40%-100%的消息量,这个取决于消息的大小.

<img alt="Figure 1" src="https://lh5.googleusercontent.com/ofF2IcR8CmMac62YjMFgHP30PmmZ4QspU2BSID54fYN8lJRdrNd_0wPH6UVSPiiTcZMIL7hkdky8BaXmQRRgwdD38HR1wBATFbcnwxr-lK6XeLUWkVwTetQy"/>

貌似看起来Netty很牛叉,但是在大的拓扑结构上存在更多竞争的条件下会怎么样呢?
下面的图是在100worker,100spout,500bolt分布在100个spout,每个100个bolt.同时还有100个acker,基本上是之前的100倍.
下面图是不是看起来很奇怪,随着消息的增多,Netty的消息处理量趋向于zeroMQ.

<img alt="Figure 2" src="https://lh5.googleusercontent.com/0ne5JXPaoRF4YbhHpB88LRXb41vbvC_mHDsP2P0AVmgfFDFWFbbmWDJP2QVSlJDlu7_nj9OErXd6H_8zPkAgyNhX7LrPk2olL-bP7sJ2O0Ac-1I5TArDuKUC"/>


为什么每秒处理能力会随着消息的大小而上升? 这完全是违反直觉的哇.仔细挖掘一下每个单独的节点.
图中的Worker处理曲线随着CPU使用率上升而上升,基本上CPU是800%，所有的Core都被充分利用了,但是load值平均在0.2.
这里很明显因为CPU忙于上下文线程切换,并没有真正的忙于处理消息.Netty是一个异步IO框架,每条消息会被回调到OS,然后线程挂起,等待被唤醒去做其他的事情.
一旦数据完成回调到用户进程里,一些线程就被唤醒去看看能不能处理那条消息.CPU频繁的在休眠与唤醒之间切换，浪费了大量的资源.

随着消息量增多,线程切换越来越少,但是很奇怪消息量还可以提升呢.这种趋势一直持续到每条消息在2.5KB左右.曲线开始在4.6G/s的消息量上下降.最后曲线与zeroMq汇集在
4KB大小的消息上。

如果真的是因为CPU切换的原因,那我们减少线程不就可以了?默认netty工作线程数目为CPU物理核数目.如果有多个Worker在一台机器上,这样Netty就会创建更多的线程.从而造成
频繁的CPU切换.这里有两种方法解决:
* 每台物理机只跑一个worker.也就是说一台机器一个JVM.
* 设置线程数目为1,这样就与ZeroMQ一样了.每台物理机器一个线程.这里有一个[pull request](https://github.com/nathanmarz/storm/pull/693)

但是Netty默认设置并不能很好的处理大量的短消息,即使设置了每个节点一个worker.但是如果我们限制线程数目为1的时候,可以得到111%-85%的每秒消息量提升。
因为短小的消息JAVA处理起来非常非常的快,于是线程切换又一次影响了性能.

# 结论

默认的Netty设置已经超过zeroMQ两倍的性能.而且已经用在Yahoo内部.
最后有相关的配置与命令,以及配置信息,这里给出贴出[原文](http://yahooeng.tumblr.com/post/64758709722/making-storm-fly-with-netty).






