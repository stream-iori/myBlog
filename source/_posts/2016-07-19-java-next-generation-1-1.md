title: 次时代Java编程(一):续 vertx-sync实践
date: 2016-07-19 18:40:02
tags:
---

### 次时代Java编程(一):续 vertx-sync实践

本来打算另起一篇，写其他方面的东西，但是最近比较忙，就先写一篇实践方面的文章。  

#### vertx-sync是什么
上一篇我们已经讲了 **Fiber** 相关的知识，想必大家对Java实现类似Golang的coroutine已经有印象了，既然Java世界里有第三方提供了这么好的库，
那我们就看看怎么跟 **vert.x** 结合起来使用。  

vert.x官方为了解决异步代码编写的困难，使之更加同步化对开发人员更友好，便基于quasar包装了一个的同步库，[vertx-sync](https://github.com/vert-x3/vertx-sync/)，该库的作者同样也是vert.x的原作者，所以完成度还是很高的。  
vertx-sync 对外只是暴露了几个简单的静态API，来完成对vert.x体系内一系列的操作包装，其实主要也就是三静态API而已。  

* Sync.fiberHandler 如果你希望你的handler里有一些逻辑需要在Fiber里运行，则你的handler必须用这个方法包一下。  
* Sync.awaitEvent 从Handler里返回一个事件结果(同步的)，且 **不会阻塞EventLoop**  
* Sync.awaitResult 从Handler里返回一个异步事件结果(同步的)，且 **不会阻塞EventLoop**  

直接看介绍可能有点不直观，下面跑几个例子。  

#### 使用vertx-sync  
之前介绍过quasar，如果你希望在项目里使用coroutine的话，需要在JVM里设置一个参数，用于应用启动前修改字节码(注入一些中断方法)，从而可以达到协程的目的。
具体方法也很简单。
```
-javaagent:/path/to/the/quasar-core-0.7.5-jdk8.jar
```

如果是基于Maven跑单元测试，那只需要引用quasar instrument的插件就可以里  

```Xml
<plugin>
	<groupId>com.vlkan</groupId>
	<artifactId>quasar-maven-plugin</artifactId>
	<version>0.7.3</version>
	<configuration>
		<check>true</check>
		<debug>true</debug>
		<verbose>true</verbose>
	</configuration>
	<executions>
		<execution>
			<goals>
				<goal>instrument</goal>
			</goals>
		</execution>
	</executions>
	<dependencies>
		<dependency>
			<groupId>co.paralleluniverse</groupId>
			<artifactId>quasar-core</artifactId>
			<version>0.7.5</version>
		</dependency>
	</dependencies>
</plugin>
```

上面是一些非常必要的准备工作，否则你无法使用quasar以及vertx-sync。

##### vertx定时器例子
之前通过vert.x调用定时器，需要传一个回调handler，然后所有的代码逻辑都包在里面。  

```Java
vertx.setTimer(1000L, h -> {
	System.out.println("time's up");
});
```

现在我们来重新塑造一下三观。  
```Java
awaitEvent(h -> vertx.setTimer(1000L, h));
System.out.println("time's up");
```
这里定时器会阻塞在awaitEvent这一行，直到一秒后才会执行下面的一行。有点类似执行 *Thread.sleep(1000L)*，但是并不会阻塞 **EventLoop** 因为quasar会在EventLoop基础之上再开启一个fiber。  
下面我看个稍微复杂点的例子。  

##### HTTP Client请求例子
我们先用传统的回调方式使用vert.x的HttpClient API。  

```java
HttpClientRequest httpClientRequest = vertx.createHttpClient().get("leapcloud.cn");
httpClientRequest.handler(response -> {
	response.handler(responseBody -> {
		System.out.println(responseBody.toString());
	});
}).end();
```
这里有两层回调嵌套，一层是得到Http的Response，另一层是从Response里得到详细的body。因为有lambda表达式才使得Java现在看起来并不是那么恶心。但是如果我们需要根据body的内容进一步做判断去继续请求其他页面，则嵌套会变的非常的深。下面尝试改造成sync方式看看。  

```java
HttpClientRequest httpClientRequest = vertx.createHttpClient().get("leapcloud.cn");
HttpClientResponse response = awaitEvent(Sync::fiberHandler);
Buffer body = awaitEvent(response::handler);
System.out.println(body.toString());
```
额，是不是感觉看着很舒服，无嵌套，直接顺序下来，非常的直观，加上Java8特有的方法引用，会让代码更精简。  

##### 通过vertx-sync使用Vert.x JDBC
写过vert.x同学肯定知道其vertx-jdbc-client为了使其兼容异步开发模型，将JDBC的底层线程池用异步方式包装了一下，也就是说JDBC层还是通过线程池去访问数据库的，但是是通过vert.x的context做了层封装，使其可以将结果放到对应的 **EventLoop** 里，这样比较符合vert.x的开发风格。但是带来的弊端就是嵌套太深。  
```java
final JDBCClient client = JDBCClient.createShared(vertx, new JsonObject()
.put("url", "jdbc:hsqldb:mem:test?shutdown=true")
.put("driver_class", "org.hsqldb.jdbcDriver")
.put("max_pool_size", 30));

client.getConnection(conn -> {
	if (conn.failed()) {
		System.err.println(conn.cause().getMessage());
		return;
	}

	final SQLConnection connection = conn.result();
	connection.execute("create table test(id int primary key, name varchar(255))", res -> {
		if (res.failed()) {
			throw new RuntimeException(res.cause());
		}
		// insert some test data
		connection.execute("insert into test values(1, 'Hello')", insert -> {
			// query some data
			connection.query("select * from test", rs -> {
				for (JsonArray line : rs.result().getResults()) {
					System.out.println(line.encode());
				}

				// and close the connection
				connection.close(done -> {
					if (done.failed()) {
						throw new RuntimeException(done.cause());
					}
				});
			});
		});
	});
});
```

上面代码可以是不是有点恶心呢？尝试改造一下吧。  

```java
final JDBCClient client = JDBCClient.createShared(vertx, new JsonObject()
.put("url", "jdbc:hsqldb:mem:test?shutdown=true")
.put("driver_class", "org.hsqldb.jdbcDriver")
.put("max_pool_size", 30));

try (SQLConnection conn = awaitResult(jdbc::getConnection)) {
	awaitResult(h -> conn.execute("create table test(id int primary key, name varchar(255))", h));
	awaitResult(h -> conn.execute("insert into test values(1, 'Hello')", h));
	ResultSet query = awaitResult(h -> conn.query("select * from test", h));
	for (JsonArray line : query.result.getResults()) {
		System.out.println(line.encode());
	}
	AsyncResult done = awaitResult(h -> conn.close(h));
	if (done.failed()) {
		throw new RuntimeException(done.cause())
	}
} catch (Exception e) {
	e.printStackTrace();
}
```

除了一个try catch，其他都没有嵌套，整体逻辑的可读性非常高，完全是线性的。

##### 如何将逻辑放倒Fiber里
你也许会发现我们似乎一直都没有用到 **fiberHandler** 这个静态方法，上面虽然写了定义，可能大家还是不能够理解，这里结合场景也许能更好理解。  
我们尝试实现一个操作很耗时的逻辑然后包到fiber里，避免 **EventLoop** 被阻塞。这里你也许会很好奇，既然 Fiber 这么廉价开启10万8万的无所谓啊，恩，这里再提一下quasar的重点部分: **fiber可以很廉价的被创造出来，但是他本质上还是跑在一个线程上面，如果其中一个fiber执行了非常耗时的操作，则后面的fiber会一直等待，从而造成整个线程阻塞。** 也就是说一个fiber不能执行非常耗时的操作，比如计算100万以内的素数之和，对于这种操作，我们可以通过直接将逻辑放到vert.x的worker线程里单独去跑，然后通过fiber包装一下就可以了。
```java
AsyncResult<Long> result = awaitResult(fiberHandler(h -> vertx.executeBlocking((Handler<Future<Long>>) event -> {
	//求百万以内素数之和,这里的逻辑会在vert.x的worker线程里跑。随便耗时多久,都不会阻塞EventLoop
	long sum = sumOfPrime(1, 000, 000);
	event.complete(sum);
}, h)));
//打印结果
System.out.println(result.result());
```  

这里你会注意到 awaitReslt 里用了 *fiberHandler* ，因为executeBlocking里的 *handler* 逻辑本身并没有跑在fiber体系下，所以会导致无效，而fiberHandler的作用就是将一段vert.x的handler包到 fiber 里。使之后续的await可以将其结果返回，这里使用awaitResult返回结果。  
我们再深入一点看看 fiberHandler 方法里到底干了什么。  
```java
@Suspendable
public static <T> Handler<T> fiberHandler(Handler<T> handler) {
  FiberScheduler scheduler = getContextScheduler();
  return p -> new Fiber<Void>(scheduler, () -> handler.handle(p)).start();
}
```
这里获取Fiber的调度器，然后直接new了一个 **Fiber** ，避免了我们自己对逻辑做Fiber包装。是不是很简单呢。

### 总结
相比较传统的回调Handler，vertx-sync的包装十分优雅，基本可以恢复到同步方法调用级别，很好的减轻了异步回调带来的心智负担。  
但是这个毕竟不是JVM级别的实现，所以或多或少还是有点门槛的，比如部署的时候，需要通过设置JVM参数来修改部分字节码，同时还要注意一些
需要挂起的方法上面加注释或者强行让其抛出可中断异常。个人建议在一些不重要的工具级项目里使用，非常重要的项目不推荐使用，当然了如果你觉得你的业务只需要依赖vert.x那么强烈你推荐你使用，只要记得打开 **BlockingChecker** 就好，可以即时的发现潜在的阻塞逻辑。  
另外7.24号上海会有一场关于Vert.x的[聚会](http://www.huodongxing.com/event/9342097827100?utm_source=%E6%90%9C%E7%B4%A2%E9%A1%B5&utm_medium=&utm_campaign=searchpage)，有兴趣的同学我们可以当面聊聊。
