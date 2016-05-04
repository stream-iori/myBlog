---
layout: post
title: "vert-x内部线程分析"
date: 2013-11-21 16:04
comments: true
categories: vert.x源码分析
published: true

---

Vert.x是一个纯粹的异步事件架构,与NodeJS一样,一个业务需要大量的回调函数来完成.
如果没有好的框架以及模式(Promise),整体代码都会陷入到回调的恶梦里(callback hell).
这篇我先不聊如何避免这类情况，我们先Look Look如何构建基于回调机制的框架.

### 一个Handler的故事
如果你已经用Vert.x写过一些项目,我估计你最大的感受就是满屏的Handler,做任何事情都需要用Handler来处理.
如果没有Handler你在Vert.x里寸步难行,那么Vert.x到底是如何运作一个Handler的呢.
我们这里以Vert.x最具代表的API- EventBus为例,来看看一个Handler是如何在Vert.x里运作的.

直观上来看调用EventBus的send方法,然后最后一个参数设定为一个Handler的实现者,或者匿名类就可以让这个流程转起来,那么这个Handler的这send方法里是怎么存活的呢.如果并发的多次调用,又是怎么区分的呢,他们会不会乱掉,会不会认错彼此呢.下面我先得认识一些Vert.x内部的接口

#### 重要的接口
* `org.vertx.java.core.Handler`

Vert.x运行时的核心接口,你大部分的逻辑代码都会在这里执行.所以无需解释.

* `org.vertx.java.core.Context`

前面我们已经说过Verticle的概念,这个名词的含义就是一个基于Vert.xAPI搭建起来的可运行的程序.
这里我们就简单的理解继承了`org.vertx.java.platform.Verticle`的Java类或者其他语言的脚本.
那`Context`接口就代表着一次可执行单元的上下文,这里的上下文只干一件事情就是处理Handler里的内容
`void runOnContext(Handler<Void> action);`,在Vert.x里有两种上下文,即EventLoop与Worker,而Worker又会分按顺序执行的Worker与多线程Worker.这里我们就先看成两类EventLoop与Worker.so,什么是EventLoop呢.

这是个好问题,详细的资料你可以参考[WIKI](http://en.wikipedia.org/wiki/Event_loop)我这里给出我的解释,在Vert.x里所有的事件包括IO都是依赖于Netty的EventLoop接口,而这个接口在Netty里会一定的频率调用.即当发生IO事件时,Netty会按时间比率分配CPU资源去响应这个事件.

在Vert.x里你可以简单的理解为IO相关的事件就可以了,用了一个特定的线程池来响应这类请求.
而Worker在Vert.x里默认是一套按顺序执行的Handler,即按照先来先到的顺序依次执行,此类的请求是另一个线程池执行.

* `org.vertx.java.core.Vertx`

这个其实就是API,非Vert.x扩展者,能用到的所有的东西都在这里了.
基于上面的三个接口,其实就能抽象出一个异步的模型,通过Vertx接口调用一个API,API内部会持有一个Context,在API本身的非业务逻辑执行完后,将Handler传入Context执行.这大概就是整个Vert.x内部执行的流程,三个接口抽象出一个世界这便是软件设计的哲学.

那如何丰满这三个接口,使之正常的跑起来呢



### Vert.x线程模型与执行上下文
* 线程模型

我们先来看看Vert.x怎么实现Context的,这是一个没有暴露给我们的接口.在内部维护EventLoop与Worker两种线程池,且有一个抽象实现者`org.vertx.java.core.impl.DefaultContext`
内部维护着这两种线程池,分别是`EventLoop` 与 `orderedBgExec`,`EventLoop`是基于Netty的NioEventLoop产生,线程数目默认与CPU核数一致,而`orderedBgExec`的线程数目是固定的,模式20个，当然我们可以通过修改参数`vertx.pool.worker.size`来提升线程数目.这里需要注意的是orderBgExec内部是依赖一个`LinkedList<Runnable>`来维护顺序的,而且实现的非常巧妙.这里其实是parent Executor的一个委托.

{% codeblock OrderedExecutor OrderExec.java%}
    public OrderedExecutor(Executor parent) {
      this.parent = parent;
      runner = new Runnable() {
        public void run() {
          for (; ; ) {
            final Runnable task;
            //tasks 是一个 LinkedList
            synchronized (tasks) {
              task = tasks.poll();
              if (task == null) {
                running = false;
                return;
              }
            }
            try {
              task.run();
            } catch (Throwable t) {
              log.error("Caught unexpected Throwable", t);
            }
          }
        }
      };
    }
{% endcodeblock  %}

* Context具体行为

我们知道Context的分类后,来看看它在Vert.x内部主要干了哪些工作,在这之前还有一个内部接口一直没有介绍,`VertxInternal`.
这其实是一个内部接口,外部API是用不到这些接口的,它主要封装了两种Context的生命周期,包括建立,获取,以及重新设置,里面还有两个`SharedData`.这个留到
后面再讲.我们回到Context的抽象类`DefaultContext`,会发现默认的构造函数里必须传入VertxInternal以便对两个Context的生命周期进行管理.
`Context`接口只有一个方法需要实现`public void runOnContext(final Handler<Void> task)`,但是针对不同的Context有不同的执行方式,所以这里的DefaultContext
抽象类只是将一些共用的代码提取出来复用.这里面最主要的方法其实是`wrapTask(final Runnable task)`,他将接口传进来的Handler进行包装,成为`Runnable`后放入相关的线程池里
去执行.

{% codeblock WrapTask  WrapTask.java%}
    protected Runnable wrapTask(final Runnable task) {
    return new Runnable() {
      public void run() {
        //执行前要得到当前线程的名字,一般会是EventLoop或者Worker前缀
        Thread currentThread = Thread.currentThread();
        String threadName = currentThread.getName();
        try {
          //执行前,要设置Context,这其实是一个ThreadLocal绑定,下次会详细讲
          vertx.setContext(DefaultContext.this);
          task.run();
        } catch (Throwable t) {
          reportException(t);
        } finally {
          if (!threadName.equals(currentThread.getName())) {
            currentThread.setName(threadName);
          }
        }
        if (closed) {
          //即使Context被关了,Task还是可以运行的,但是还会去执行unsetContext(),避免泄露
          //这里其实是remove一个ThreadLocal的绑定 
          unsetContext();
        }
      }
    };
    }
{% endcodeblock  %}


* Context在Vert.x内部的调度

`VertxInternal`承载着整个Vert.x的内部运行,包括Actor节点的建立,线程池的创建,Socket服务在内部Verticle之间的共享,乃至集群的实现.
他像一个胶水一样,粘着外部API到Context之间.了解这个接口的实现,对Vert.x整体运行的过程会很有帮助,当你写代码的时候脑子里会不由自主的产生相关规划以及规约,
这其实就是一种框架带给你的哲学观.

这里不一一列举其内部的实例变量,这里只讨论重点的,即如何处理一个`Handler`.以后逃了的服务启动,EventBus,以及集群的时候我们会回头再来看它.
`private final ThreadLocal<DefaultContext> contextTL = new ThreadLocal<>();`

这个ThreadLocal是执行Handler的重点,他到底是怎么工作的呢.
因为每个Handler接口都是用户自己的业务逻辑,当Vert.x执行的时候会尝试得到当前线程的Context,如果发现当前线程没有Context的话,会直接创建一个新的.

{% codeblock getOrCreateContext.java%}
    public DefaultContext getOrCreateContext() {
        DefaultContext ctx = getContext();
        if (ctx == null) {
        // Create a context
            ctx = createEventLoopContext();
        }
        return ctx;
    }
{% endcodeblock  %}

这里的createEventLoopContext,就是创建一个基于DefaultContext的EventLoop.有了Context之后,只需将Handler放入Context执行即可.注意这里都是EventLoopContext.
除非你指定你的模块为Worker,否则默认Vert.x都是基于EventLoop线程运行你的Handler.这也说明了为什么不要让一个Handler长时间运行的原因,因为你阻塞了IO的调用.导致Socket里的
数据不能及时被处理.

### 异步传说
至此,一个Handler的完整的旅行就结束了,让我们来回顾一下看看.
你通过Vert.x的API,传入一个包含了你的业务的Handler,Vert.x内部API调用createContext,来创建一个基于EventLoop的Context来运行这个Handler.
当下次再调用到这个线程的时候,不需要再次创建Context,直接从ThreadLocal里拿出来就可以了.

当然其中的细节,远远没有你想的那么简单.下次我们来深度了解一下EventBus的相关事情.







