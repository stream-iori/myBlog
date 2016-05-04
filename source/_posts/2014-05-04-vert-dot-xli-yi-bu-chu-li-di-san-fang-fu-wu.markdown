---
layout: post
title: "Vert.x里异步处理第三方服务"
date: 2014-05-04 09:48:31
categories: Vert.x
published: true

---

有一段时间没有发布新的Vert.x相关信息了.最近忙于一些简单的IOS开发,所以没有太多时间写一些总结.5.1假期后,终于有一点时间可以总结之前半年的Vert.x相关经验.这里会陆续放出来.

## Vert.x里如何引入第三方服务

很多时候我们不可避免的需要与已有的系统做对接,比如将某个服务的一些信息通过MQ发布到消息队列里,让后消息队列的消费端去处理消息.这里的MQ就算是第三方服务.
so,我们怎么在vertx项目里做为生产者将消息发布出去呢,或者vertx本身作为消费者去处理来自MQ的消息呢.

_这里有两个前提_

* 在vertx里所有的行为必须是异步的,及没有返回值
* 线程之间传输值,必须是通过EventBus

我们这里依旧以RabbitMQ为例子,来解释一下如何在vert.x里调用RabbitMQ.

### 用EventBus来包装RabbitMQ
计算机软件里,所有的问题都可以通过包一层来解决.不记得这句牛X的话出自哪里,不过的确是可以通过用bus包装java的RabbitMQ SDK来实现.
具体做法也很简单,我们把所有的对MQ的请求发到一个特定的verticle里就OK了.这个verticle通过eventBus接受MQ相关的信息,然后处理异步的返回消息结果.
so,新建一个verticle,通过继承`BusModBase`,可以简化一些繁琐的步骤.

这个verticle,我们姑且称之为`MQ-verticle`,在`start`方法里,我们需要完成两个工作

* 连接rabbitmq服务
* 注册提供对外服务的bus handler

连接rabbitmq服务很简单,通过`JsonObject`传入配置信息,比如mq的`uri` `user` `password`等,通过SDK直接建立connection就OK了.
下面就是确定对外暴露的bus服务,这里我们提供三个服务`create consumer`,`close consumer`,以及`send message`.通过向`MQ-verticle`发送带有此类的消息, 我们可以确定
消息处理方式.简单点讲通过`swtich case`消息,来确定处理分支. 下面是示意代码

{% codeblock Sstart.java %}

    public void start() {
      super.start();
    final String address = config.getString("address", "vertx.rabbitMQ");
    String uri = getMandatoryStringConfig("uri");
    defaultContentType = ContentType.fromString(getMandatoryStringConfig("defaultContentType"));

    ConnectionFactory factory = new ConnectionFactory();

    try {
      factory.setUri(uri);
    } catch (URISyntaxException e) {
      throw new IllegalArgumentException("illegal uri: " + uri, e);
    } catch (NoSuchAlgorithmException e) {
      throw new IllegalArgumentException("illegal uri: " + uri, e);
    } catch (KeyManagementException e) {
      throw new IllegalArgumentException("illegal uri: " + uri, e);
    }

    try {
      conn = factory.newConnection(); // IOException
    } catch (IOException e) {
      throw new IllegalStateException("Failed to create connection", e);
    }

    // register handlers
    eb.registerHandler(address , new Handler<Message<JsonObject>>() {
      public void handle(final Message<JsonObject> message) {
        JsonObject body = message.body();
        switch (body.getString("method")) {
           case "create-consumer":
             handleCreateConsumer(body);
             break;
             
           case "close-consumer":
             handlerCloseConsumer(body);
             break;

          case "send-message":
             handlerSendMessage(body);
        }
        
       }
      });
    }
{% endcodeblock  %}

#### 如何处理消息的ACK

这里有一个难点就是处理ACK消息.因为consumer创建后,不可以返回方法执行结果,所以只能以replyHandler的方式返回给请求发起发,告知rabbitMQ消息是否已经被ACK.
所以这里创建`consumer`的时候需要带一个`forwardAddress`用于真正消费消息的verticle.当然`MQ-verticle`收到消息且没有异常后,将消息`forward`到目标verticle
然后根据replyHander可以ACK这条消息.

{% codeblock createConsumer.java %}

    eb.send(forwardAddress, body, new Handler<Message<JsonObject>>() {
          @Override
          public void handle(Message<JsonObject> event) {
            //这里只ACK正确处理的消息
            if (event.body() != null && event.body().getString("status").equals("ok")) {
              try {
                //消息消费成功后ACK给RabbitMQ,删除消息
                getChannel().basicAck(deliveryTag, false);
              } catch (IOException e) {
                container.logger().warn(e.getMessage(), e);
              }
            }
          }
        });
        
{% endcodeblock  %}

### 不依赖EventBus,结合第三方服务.
是否可以不通过EventBus来处理第三方服务呢,当然是有办法,比较Hacker,而且需要对Vert.x内部结构了解比较深刻才行.
Vert.x内部在执行业务逻辑的时候,也就是你自己定义的`handler`,是基于一个特定的`环境 Context`.由这个环境来区分是在`EventLoop`里还是
在`worker`里.这里重点讨论`EventLoop`.如果你熟悉Vert.x内部机制,你会知道默认的Context就是EventLoop,他从Netty那里得到`GroupEventLoop`用于
基于`I/O`的请求处理.这样差不多是在模拟CPU时间片段,你可以想象成每个`Handler`同一时间只有被一个`EventLoop`线程处理.处理完成之后就`EventLoop`会去
处理另外的`Handler`.而且只要是基于`I/O`的请求,Vert.x必然会包装成Handler.所以你会觉得用Vert.x写程序好麻烦.
其实Vert.x内部留了一个API给用户,让之去实现基于线程逻辑.怎么讲呢.

{% codeblock Example.java %}

     vertx.runOnContext(new Handler() {
          
     });

{% endcodeblock  %}

如果你有自己的逻辑想在上一个Handler处理完后继续跑在EventLoop下,就封装到上面的Handler里.
Ok,你会问,什么时候需要`runOnContext`呢.其实实际用到的地方比较少,典型的场景就是结合第三包的异步API.
比如上面的RabbitMQ.你可以不包装成`EventBus`,直接在Verticle的start方法里通过RabbitMQ JAVAAPI启动
一个Channel,绑定一个`Consumer`.然后实现一个一直在消费数据的rabbitmq消费端.

{% codeblock Example.java %}

    private void consuming() {
         vertx.runOnContext(new Handler() {
            QueueingConsumer.Delivery delivery = consumer.nextDelivery();
            String message = new String(delivery.getBody());
            System.out.println(" [x] Received '" + message + "'");
            //ACK消息
            channel.basicAck(delivery.getEnvelope().getDeliveryTag(), false);
            //递归调用
            this.consuming();
         });
    }
    
{% endcodeblock %}

通过上面的代码你可以很好讲第三库的API整合到Vertx的`EventLoop`里,但是需要注意的是,第三方库的API不能是
同步的.同步方法会将整个`Verticle`Hang死.


























