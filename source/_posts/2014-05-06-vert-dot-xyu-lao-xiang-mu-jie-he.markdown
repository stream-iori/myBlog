---
layout: post
title: "Vert.x与老项目结合"
date: 2014-05-06 21:59:13
categories:
- Vert.x
published: true

---

已经很多人问过我,关于Vert.x如何与现有的项目结合在一起使用.具体的场景莫过于标准的Java三层结构中融合高性能的Vert.x
我这里结合自己的项目总结一下.

### 作为Web前端接入层
在Java框架界里,Web层框架可选的不多,老牌的Apache旗下的`struts2`, 万能的Spring下有`SpringMVC`,以及`Jersey`基于标准的REST风格的框架.
他们目前都是基于`Servlet`标准实现,所以可以在主流的JavaWeb容器中随意切换.

而`Vert.x`本身就是一个Web容器,同时提供简单的REST风格框架实现.所以如果你的项目不需要考虑Session以及Cookie.可以直接使用Vert.x作为Web前端请求接入.
我的项目里只有一个Web入口作为OpenAPI服务,所以不使用Session 与 Cookie验证请求方.

另外如果你的Web服务是对内部人员使用,只是查看service上的一些性能数据,也可以直接使用Vert.x自带的REST框架.
如果你想Vert.x作为一个完整的Web容器以及支持标准的HTTP请求,且能基于HTTP开发框架的话,这里推荐两个

* [mod-jersey](https://github.com/englishtown/vertx-mod-jersey)
这个应该很流行,懂jersey,就可以直接使用.目前也支持Annotation.适合传统的JavaWeb开发者

* [Yoke](https://github.com/pmlopes/yoke)
这个与NodeJS里的Express很像,扩展都是基于middleware来实现.适合习惯异步编程,NodeJS经验者

* 另外如果你使用Clojure,推荐本人的[ring-vertx-adapter](https://github.com/stream1984/ring-vertx-adapter)
这个项目尚需进一步优化

### 作为服务层处理逻辑
这一层基本上对应的传统Java里的领域模型处理层.各种Service调用,以及对数据层的调用.差不多是一个承上启下的一层.
传统的模型里,这一层基本上都是同步调用,即使有异步调用,也是与业务逻辑分离的异步.如果全异步会导致业务逻辑碎乱.代码很难描述清楚.
到这里你会发现Vert.x其实不太好融合到业务性很强的服务层里.其主要原因如下

* 自身是异步体系,不适合描述顺序逻辑性强的业务
* 由于异步的问题,访问数据层也必须是异步,导致业务模型进一步碎片化.

说实话,的确很难处理这一层,异步特别适合处理那种I/O消耗的应用,比如Web请求,磁盘的请求等.如果用他来处理那些本来就在内存里的事务,那
会显得有点蹩脚.这里举一个'栗子':银行转账,传统的做法是在service的方法里,通过访问DAO得到两个账户,然后进行简单的加减即可.

{% codeblock example.java %}

    public void transfer() {
      Account sourceAccount = accountDAO.getAccount(1);
      Account targetAccount = accountDAO.getAccount(2);
      sourceAccount.money--;
      targetAccount.money++;
    }

{% endcodeblock  %}

但是在Vert.x里因为要访问的DAO不是同步的方式导致逻辑嵌套,显得代码很难理解.

{% codeblock example.java %}

    //开启事务
    public void handler(Message<JsonObject> message) {
      //操作DAO对账户1做操作
      eb.send("mysql.address", obj, new Handelr<Message<JsonObject>>(Message message){
        //在回调的Handler里继续操作2
        eb.send("mysql.address" obj new Handler<Message<JsonObject>>(Message message){
        //这里可以做事务上的提交
        })
      })
    }

{% endcodeblock  %}

Ugly,我们不能拿着斧子看什么都是钉子,所以我们保留对数据库操作,对其包装成一个EventBus,接受请求.具体可以产考我的上一篇文章
[Vert.x里异步处理第三方服务](http://www.streamis.me/blog/2014/05/04/vert-dot-xli-yi-bu-chu-li-di-san-fang-fu-wu/)
这样就是拿Vert.x去处理I/O密集性的请求.传统的事务性请求还是同步处理.
具体做法大概就两种
* 用Vert.x加载Spring获取其他Ioc容器,然后通过解析消息,来分发到对应的数据处理方法上.
* 老的项目不做改动,但是启动的时候IoC容器去实例化一个嵌入式的Vert.x作为EventBus来接受请求.

但是个人不推荐这么做,如果你想全栈都使用Vert.x的话.上面的方法只是过渡使用,而且并没有彻底解放同步请求数据库的问题.
关于如何全栈使用Vert.x的问题我留到下一篇细琢.

### 访问Mysql以及Nosql
Vert.x官方提供MYSQL,mongoDB,Redis等主流的数据层访问模块.
上面已经说了,数据库的访问其实是很恶心的,全异步的代码大部分人看起来很不舒服.
但是如果你一定要使用Redis,mongoDB的mod,建议再加一个模块,RxJava or vertx-mod-when其实这两个库应该是贯穿整个Vert.x项目的.
没有这两个库你几乎没有办法优雅的写程序.我们对比一下就知道了.这里偷懒就这件改上面的例子了

{% codeblock example.java %}
    
    //step 1:定义消息体
    JsonObject sourceAccountOperator = new JsonObject();
    msg.putString("action", "update")
            .putString("sql", "update account set money = ? where id = ?")
            .putArray("values", new JsonArray([1,1]));

    JsonObject targetAccountOperator = new JsonObject();
    msg.putString("action", "update")
            .putString("sql", "update account set money = ? where id = ?")
            .putArray("values", new JsonArray([2,2]));

    //step 2:发送消息
    Observable<Message<JsonObject>> accountOne =  eb.send("mysql.address", sourceAccountOperator);

    //step 3:串联消息,必须在step2结束后才执行
    Observable<Message<JsonObject>> accountTwo = accountOne.flatMap(new Func1<Message, Observable<Message>>() {
      @Override
      public Observable<RxMessage> call(Message message) {
        eb.send("mysql.address", targetAccountOperator);
      });
    });

    //step 4:处理结果
    accountTwo.subscribe(new Action1<Message>() {
      @Override
      public void call(Message message) {
        //提交事务
      }
    });

{% endcodeblock  %}
上面的代码没有超过两层的嵌套,通过Rx模式,避免了callback Hell.但是上面的代码还是有点乱.借助第三方框架还能更加优雅的处理异步回调问题.
下次介绍我的rxBus.让Vert.x基于EventBus实现RPC调用.







