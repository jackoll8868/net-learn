# 基于 Netty 的项目如何在异步模式下同步请求

[基础知识](#)

​	 [同步调用](#同步调用)

​	[异步调用](#异步调用)

[网络通信中的同步和异步应用场景](#网络通信中的同步和异步应用场景)

[如何在异步模式实现同步](#)

​	[连接池模式](#连接池模式)

​	[滑动窗口模式](#滑动窗口模式)

## 基础知识

### <a id="synccall">同步调用</a>

同步调用对于广大程序员来说应该是习以为常，司空见惯的了。简单来说就是语句 A、B、C 的执行顺序是 A—>B—>C，那么在 A 没有执行完成(返回值或者抛出异常)之前，B 和 C 是无法执行的。

```java
public void method() throws Exception{
  //校验参数
  Assert.notNull(***);
  //查询数据库或者访问 OPENAPI
  T response = x.invoke();
  //对查询到的东西作进一步的加工
  handle(response);
}
```

上述代码示例中，只有校验参数通过了，才能去查询数据库或者调用 OPENAPI 获取相关资源，在获取相关资源成功之后才能对得到的值进行进一步的加工。这就是我们日常编程中经常使用的，就不在赘述。

### <a id="synccall">异步调用</a>

*asynchronous call* 异步调用：一个可以无需等待被调用函数的返回值就让操作继续进行的方法。

也就是说异步调用允许在当前调用返回值还未返回之前，继续做想做的事情。返回值返回或者抛出异常之后怎么处理需要在后续的逻辑中声明。例如，在一个分布式环境中，进程 A 中的模块 X，需要不定时的从 N 个服务节点中获取节点健康相关的信息，讲得到的信息稍加处理用于分析节点是否正常或者做一些舆情分析。由于获取节点健康信息需要通过网络调用来完成，再加之服务器的节点数 N 可能足够大，如果按照我们通常的同步逻辑来说可能会出现如下的代码:

```java
public class A{
  static final ConcurrentHashMap<String,URI> NODE_CACHE = ...;
  
  public void collect() throws Exception{
    //获取 NODE_CACHE 中的数据试图然后循环
    Set<Entry<String,URI>> view = NODE_CACHE.entrySet();
    Iterator<Entry<String,URI>> it = view.iterator();
    while(it.hasNext()){
      //根据 URI 发起 HTTP 请求，并得到响应
      Entry<String,URI> node = it.next();
      T resppnse = invoke(node.getValue());
      //分析得到的请求
      handle(response);
    }       
  }
}
```

这种方式虽然能达到最终效果，但可想而知没做一次的耗时都是很长的，这可能导致应用实际已经挂了，但是等了很久才会被发现。

为了解决这种问题才有了异步调用的方式，JAVA 中的异步调用方式可以实现 Callable 或者 Runnable 接口，然后通过 JDK 提供Executors 来执行异步任务。大致的代码片段如下:

```java
public class A{
  static final ConcurretHashMap<String,URI> NODE_CACHE = ...;//
  static final ExecutorService POOL = Executors.newCachedThreadPool();
  public void asynCollect() throws Exception{
    List<Futur> buffer = ...;//用于存放异步调用的结果
    //获取 NODE_CACHE 中的数据试图然后循环
    Set<Entry<String,URI>> view = NODE_CACHE.entrySet();
    Iterator<Entry<String,URI>> it = view.iterator();
    while(it.hasNext()){
      //根据 URI 发起 HTTP 请求，并得到响应
      final Entry<String,URI> node = it.next();
      Future<T> response = POOL.submit(new Callable<T>() {
            public T call() throws Exception {
                return invoke(node.getValue());
            }
        });
      buffer.add(response);
      //分析得到的请求
      handle(buffer);
    }     
  }
}
```

在异步调用中，我们通常会得到一个代表任务执行任务的结果的一个代表值(**表值**)，也就是 Future，Future 仅仅代表这个任务的可能返回结果或者异常。所以得到 Future 之后通常会有如下的代码片段:

```java
Future<T> response = pool.submit(...);
while(response.isDone()){
  //作别的事情或者睡眠
}
if(response.isSuccess())//如果成功
	T result = response.get();//得到最终
else//执行过程中有异常
  throw new Exception(response.getCause());
```

### <a id="网络通信中的同步和异步应用场景"> 网络通信中的同步和异步应用场景</a>

这里所说的网络通信场景特指 client-server 模型中使用长连接通信的应用场景。

在这中场景中分为两个大的模块：

- Server Side
- Client Side

#### SeverSide

在服务端通常是对外开放一个端口，然后接受客户端的请求(a)—>接受数据(b)—>处理数据(c)—>将处理的最终结果返回给客户端(d)

在这个通信过程中，如果后台使用了非 BIO 模式那么以上的 a、b、c、d 都有可能是异步的，对于 a、b、c 这个组合也有可能是同步调用的，但是 d 这个操作通常是采用异步写的方式，如果关注写完成的状态通常会使用回调函数的模式;

#### Client Side

在客户端，通常要连接服务器(a)—>发送请求到服务器(b)—>等待服务器响应(c)—>根据响应做相应处理(d);

在客户端的这个执行流程中，通常是会采用基础知识单元里讲到的*同步调用*，对于某些特定的场景则会使用异步回调的方式。例如如下的场景：

网络爬虫将获取到的数据写入到缓冲区，供后续的分析模块分析。为了提升爬取的速率可以采用异步回调的方式.

```java
Promise<String> page = getPage(URI);
page.addListener(x -> onSuccess(){writeToBuffer(x)});
```

## 如何在异步模式实现同步

### <a id="线程池模式">线程池模式</a>

使用线程池模式来实现一个请求发出到响应回来之前的阻塞过程通常的模式如下:

1. 客户端在启动时同服务器端建立 N 个有效的长连接，缓存在连接池中；
2. 客户端发起请求的时候从连接池获取一个连接 X；
3. 在 X 上将请求数据发送到服务端(这是一个异步请求)然后得到 Future future;
4. 采用 future.await(offset,TimeUnit.MILLISECONDS)的方式阻塞最大offset 的时长；
5. 阻塞被唤醒之后根据 future 的状态来判断是返回值还是抛出异常。

需要注意的是第四步中的等待是指：

- 要么 offset 时长内响应提前到达被唤醒；
- 要么 offset 时长内响应没有到达被唤醒(通常所说的调用超时)。

其代码片段大致如下:

```java
public Response<T> request(request,T.class) throws Exceptiom{
  //从连接池获取连接
  Channel ch = poll.aquire(offset,TimeUnit.MILLISECONDS);
  //发送请求
  ChannelFuture future = ch.write(codec(request,ENCODE));
  boolean result = writeFuture.awaitUninterruptibly(timeout, TimeUnit.MILLISECONDS);
  if (result && writeFuture.isSuccess()) {
    response.addListener(new FutureListener() {
      @Override
      public void operationComplete(Future future) throws Exception {
        if (future.isSuccess()) {
            // 成功的调用         
    		return response;
        }
  }
	//失败的调用
  writeFuture.cancel();
  throw new Exception(future.getCause());
}
```

### <a id="滑动窗口模式">滑动窗口模式</a>

连接池模式使用了一个客户端同时持有 N 个到服务端的连接，通过这种协定，在通过一个 request 对应一个连接的方式来实现了阻塞网络调用的模式。这种模式通常叫做单工模式。

对于双工通信的互联网程序来说，存在一种对连接数极为苛刻的要求。例如如下的场景：

> 一个电信短信网关，同时要服务重得多的客户。为了保证连接尽可能高效的利用，做出了如下的限定：
>
> 1. 每个短信网关的客户每次只能有且仅有一条活动的长连接同短信网关通信；
> 2. 通信过程采用全双工模式；
> 3. 客户端数据的收发均是异步的模式；

对于上述的这个场景，如果还是使用连接池的模式，那么显然是行不通的。因为连接池模式的处理方式会因为 A 在调用的时候 B C D 这些请求都要等待 A 处理完成之后才能使用仅用的一条长连接。要达到高效的读写必须采用如下的方式：

1. 发起请求和接受请求必须分开来做，并且收发之间相互互不影响；
2. 为了保证数据的安全性，一次性不能有太多的数据发送请求最大值为 X；

上述的模式大致如下图:

![一步读写](https://ww4.sinaimg.cn/large/006tKfTcgy1fdtn36uuy0j30r80ts75e.jpg)

从图可知数据的写和读取都是异步的也就是写只负责写的部分，读只负责读的部分。这样会导致如下几个问题：

1. 数据写出去之后，响应回来的时候，如何将读到的响应和已经写出去的请求一一对应；
2. 底层是异步的读写对于上层调用者来说却要求做到感觉不到底层是异步的，上层调用者为了开发简单必须是同步的调用。

对于这种模式通常会用到 Promise，Promise 是一个可读写的 Future，也就是说可以对 Promise 进行写数据和唤醒的操作。Promise 可以实现数据的传递和唤醒工作；

要实现收到的响应和发出去的请求一一对应需要采用类似唯一 ID 的方式，也就是说每个请求都有一个唯一的 ID 标识，这个标识传递到服务端之后，服务端子给出响应的时候同样要把这个 ID 回写回来，通过这个 ID 就能做到请求和响应的一一对应。

有了以上两点知识我们不难得到一个如下的处理流程:

1. 调用者发起 request 请求；
2. 底层将 request 请求异步写到网络连接中，同时得到一个 Promise;
3. 底层将 Promise 返回，同时将{requestId:Promise}的键值对存放到 MAP  X中，然后释放网络连接给别的调用者使用；
4. 上层获得 Promise 之后阻塞 offset 的时间；
5. 但底层接收到请求之后通过{request:response} 的 key 去X 中得到 Promise，然后将 response 写入到 Promise，再调用 Promise 的 sucess 方法唤醒正在等待的上层调用者。然后删除 X 中的 requestid 对应的数据。

实际的应用场景中为了限制 X 的大小避免 OOM 或者太多的并发碰上网络异常或者程序崩溃导致的大量数据丢失或者数据重复发送。所以 X 是一个定长的线程安全的 MAP。

