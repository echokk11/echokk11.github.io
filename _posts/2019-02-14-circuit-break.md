---
author: Wuk
layout: post
title: "熔断，限流，降级"
subtitle: "微服务中的一些概念"
catalog: true
header-style: text
tags:
  - "circuit break"
  - "current limiting"
---

## 名词解释
### consumer
表示服务调用方

### provider
表示服务提供方

### 超时（timeout)
在接口调用过程中，consumer调用provider的时候，provider在响应的时候，有可能会慢，如果provider 10s响应，那么consumer也会至少10s才响应。如果这种情况频度很高，那么就会整体降低consumer端服务的性能。
这种响应时间慢的症状，就会像一层一层波浪一样，从底层系统一直涌到最上层，造成整个链路的超时。
所以，consumer不可能无限制地等待provider接口的返回，会设置一个时间阈值，如果超过了这个时间阈值，就不继续等待。
这个超时时间选取，一般看provider正常响应时间是多少，再追加一个buffer即可。

### 重试（retry）
超时时间的配置是为了保护服务，避免consumer服务因为provider 响应慢而也变得响应很慢，这样consumer可以尽量保持原有的性能。
但是也有可能provider只是偶尔抖动，那么超时后直接放弃，不做后续处理，就会导致当前请求错误，也会带来业务方面的损失。
那么，对于这种偶尔抖动，可以在超时后重试一下，重试如果正常返回了，那么这次请求就被挽救了，能够正常给前端返回数据，只不过比原来响应慢一点。
重试时的一些细化策略：
重试可以考虑切换一台机器来进行调用，因为原来机器可能由于临时负载高而性能下降，重试会更加剧其性能问题，而换一台机器，得到更快返回的概率也更大一些。
#### 幂等(idempotent)
如果允许consumer重试，那么provider就要能够做到幂等。
即，同一个请求被consumer多次调用，对provider产生的影响(这里的影响一般是指某些写入相关的操作) 是一致的。
而且这个幂等应该是服务级别的，而不是某台机器层面的，重试调用任何一台机器，都应该做到幂等。

### 熔断（circuit break）
重试是为了应付偶尔抖动的情况，以求更多地挽回损失。
可是如果provider持续的响应时间超长呢?
如果provider是核心路径的服务，down掉基本就没法提供服务了，那我们也没话说。 如果是一个不那么重要的服务，却因为这个服务一直响应时间长导致consumer里面的核心服务也拖慢，那么就得不偿失了。
单纯超时也解决不了这种情况了，因为一般超时时间，都比平均响应时间长一些，现在所有的打到provider的请求都超时了，那么consumer请求provider的平均响应时间就等于超时时间了，负载也被拖下来了。
而重试则会加重这种问题，使consumer的可用性变得更差。
因此就出现了熔断的逻辑，也就是，如果检查出来频繁超时，就把consumer调用provider的请求，直接短路掉，不实际调用，而是直接返回一个mock的值。
等provider服务恢复稳定之后，重新调用。
>典型的有SpringCloud中Netflix提供的`Hystrix`组件

### 限流(current limiting)
上面几个策略都是consumer针对provider出现各种情况而设计的。
而provider有时候也要防范来自consumer的流量突变问题。
这样一个场景，provider是一个核心服务，给N个consumer提供服务，突然某个consumer抽风，流量飙升，占用了provider大部分机器时间，导致其他可能更重要的consumer不能被正常服务。
所以，provider端，需要根据consumer的重要程度，以及平时的QPS大小，来给每个consumer设置一个流量上线，同一时间内只会给A consumer提供N个线程支持，超过限制则等待或者直接拒绝。

#### 资源隔离
provider可以对consumer来的流量进行限流，防止provider被拖垮。 
同样，consumer 也需要对调用provider的线程资源进行隔离。 这样可以确保调用某个provider逻辑不会耗光整个consumer的线程池资源。

#### 服务降级
降级服务既可以代码自动判断，也可以人工根据突发情况切换。
##### consumer端
consumer 如果发现某个provider出现异常情况，比如，经常超时(可能是熔断引起的降级)，数据错误，这时，consumer可以采取一定的策略，降级provider的逻辑，基本的有**直接返回固定的数据**。
##### provider端
当provider 发现流量激增的时候，为了保护自身的稳定性，也可能考虑降级服务。 
比如，1，直接给consumer**返回固定数据**，2，需要实时写入数据库的，先**缓存到队列**里，异步写入数据库。



## 单独说说限流

限流的目的是通过对并发访问/请求进行限速或者一个时间窗口内的请求进行限速来保护系统，一旦达到限速速率则可以拒绝服务（定向到错误页或告知资源没有了）、排队或等待（比如秒杀、评论、下单）、降级（返回兜底数据或者默认数据，如商品详情页库存默认有货）。在**压测**时，我们能找出每个系统的处理峰值，然后通过设定峰值阈值，当系统过载时，通过拒绝过载的请求来保障系统可用。另外，也可以根据系统的吞吐量、响应时间、可用率来动态调整限流阈值。

一般开发高并发系统场景的限流有:

- **限制总并发数**（比如数据库连接池、线程池）
- **限制瞬时并发数**（如Nginx的limit_conn模块，用来限制瞬间并发连接数）
- **限制时间窗口内的平均速率**（如Guava的RateLimiter、Nginx的limit_req模块，用来限制每秒的平均速率）

### 限流算法

#### 令牌桶算法

令牌桶算法，是一个存放固定容量令牌的桶，按照固定速率往桶里添加令牌。令牌桶算法的描述如下：

- 假设限制2r/s，则按照500毫秒的固定速率往桶内添加令牌。
- 桶中最多存放b个令牌，当桶满时，新添加的令牌会被丢弃或拒绝。
- 当一个n个字节大小的数据包到达，将从桶中删除n个令牌，接着数据包被发送到网络上。
- 如果桶中的令牌不足n个，则不会删除令牌，且该数据包被限流（要么丢弃，要么在缓冲区等待）。

#### 漏桶算法

漏桶作为计量工具时，可以用于流量整形和流量控制，漏桶算法的描述如下：

- 一个固定容量的漏桶，按照常量固定速率流出水滴。
- 如果桶是空的，则不需流出水滴。
- 可以以任意速率流入水滴到漏桶。
- 如果流入水滴超过了桶的容量，则流入的水滴溢出了（被丢弃），而漏桶容量是不变的。

#### 令牌桶和漏桶对比

- 令牌桶是按照固定速率往桶中添加令牌，请求是否被处理需要看桶中令牌是否足够，当令牌数减为零时，则拒绝新的请求。
- 漏桶则是按照常量固定速率流出请求，请求流入速率任意，当流入的请求数累积到漏桶容量时，则新流入的请求被拒绝。
- 令牌桶限制的是平均流入速率（允许突发请求，只要有令牌就可以处理，支持一次拿多个令牌），并允许一定程序的突发流量。
- 漏桶限制的是常量流出速率（即流出速率是一个固定常量值，比如都是1的速率流出，而不能一次是1，下次又是2），从而平滑突发流入速率。
- 令牌桶允许一定程序的突发，而漏桶主要目的是平滑流入速率。
- 两个算法实现可以一样，但是方向是相反的，对于相同的参数得到的限流效果是一样的。

### 应用级限流

#### 接口的总并发/请求数

如果接口可能会有并发流量，但又担心访问量太大造成奔溃，那么久需要限制这个接口的总并发/请求数了。因为粒度比较细，可以为每个接口设置相应的阈值。可以使用Java中的AtomicLong或者Semaphore进行限流。Hystrix在信号量模式下也使用`Semaphore`限制每个接口的总请求数。

如：

```java
try {
    if (atomic.incrementAndGet() > limit) {
        //拒绝请求
    }
    //处理请求
} finally {
    atomic.decrementAndGet();
}
```

```java
Semaphore semaphore = new Semaphore(5);
try {
    semaphore.acquire();
    //处理请求
    semaphore.release();
}
```



#### 单位时间的限流

限制每秒的请求数，可以使用Guava的Cache来存储计数器，设置过期时间为2S（保证能记录1S内的计数）。下面代码使用当前时间戳的秒数作为key进行统计，这种限流的方式也比较简单。

```java
LoadingCache<Long, AtomicLong> counter =
        CacheBuilder.newBuilder()
        .expireAfterWrite(2, TimeUnit.SECONDS)
        .build(new CacheLoader<Long, AtomicLong>() {
            @Override
            public AtomicLong load(Long aLong) throws Exception {
                return new AtomicLong(0);
            }
        });

long limit = 1000;
while (true) {
    long currentSeconds = System.currentTimeMillis() / 1000;
    if (counter.get(currentSeconds).incrementAndGet() > limit) {
        logger.info("被限流了:{}", currentSeconds);
        continue;
    }
    //业务处理
}
```



上面介绍的限流方案都是对于**单机**接口的限流，当系统进行多机部署时，就无法实现整体对外功能的限流了。当然这也看具体的应用场景，如果平行的应用服务器需要共享限流阀值指标，可以使用`Redis`作为`共享的计数器`。



使用Redis实现，存储两个key，一个用于计时，一个用于计数。请求每调用一次，计数器增加1，若在计时器时间内计数器未超过阈值，则可以处理任务，如发送短信api的qps为400

```java
//用于计时
private static final String API_WEB_TIME_KEY = "time_key";
//用于计数
private static final String API_WEB_COUNTER_KEY = "counter_key";

//1秒的时间已经过了，重新对time_key赋值，同时初始化计数变量
if(!cacheDao.hasKey(API_WEB_TIME_KEY)) {
    cacheDao.putToValue(API_WEB_TIME_KEY,0,(long)1, TimeUnit.SECONDS);
	cacheDao.putToValue(API_WEB_COUNTER_KEY,0,(long)2, TimeUnit.SECONDS);//时间到就重新初始化为0
}

//如果大于最大请求数量，直接打logger,返回
if(cacheDao.hasKey(API_WEB_TIME_KEY)&&cacheDao.incrBy(API_WEB_COUNTER_KEY,(long)1) > (long)400) {
    LOGGER.info("调用频率过快");
	return;
}

//短信发送逻辑
```





#### 平滑限流

Guava RateLimiter提供的令牌桶算法可用于平滑突发限流（SmoothBursty）和平滑预热限流（SmoothWarmingUp）实现。

1. 平滑突发限流（SmoothBursty）

   平滑突发限流顾名思义，就是允许突发的流量进入，后面再慢慢的平稳限流。

   ```java
   // 创建了容量为5的桶，并且每秒新增5个令牌，即每200ms新增一个令牌
   RateLimiter limiter = RateLimiter.create(5); 
   while (true) {
       // 获取令牌（可以指定一次获取的个数），获取后可以执行后续的业务逻辑
       // acquire()返回的是需要等待的时间，0代表获取到了令牌
       System.out.println(limiter.acquire());
   }
   ```

   上面while循环中执行的limiter.acquire()，当没有令牌时，此方法会阻塞。实际应用当中应当使用tryAcquire()方法，如果获取不到就直接执行拒绝服务。如发送短信api的qps为400

   ```java
   private RateLimiter rateLimiter = RateLimiter.create(400);//400表示每秒允许处理的量是400
   if(rateLimiter.tryAcquire()) {
   	//短信发送逻辑可以在此处
   
   }
   ```

   

2. 平滑预热限流（SmoothWarmingUp）

   平滑突发限流有可能瞬间带来了很大的流量，如果系统扛不住的话，很容易造成系统挂掉。这时候，平滑预热限流便可以解决这个问题。

   ```java
   // permitsPerSecond表示每秒钟新增的令牌数，warmupPeriod表示从冷启动速率过渡到平均速率所需要的时间间隔
   RateLimiter.create(double permitsPerSecond, long warmupPeriod, TimeUnit unit)
   ```

   ```java
   RateLimiter limiter = RateLimiter.create(5, 1000, TimeUnit.MILLISECONDS);
   for (int i = 1; i < 5; i++) {
       System.out.println(limiter.acquire());
   }
   Thread.sleep(1000L);
   for (int i = 1; i < 50; i++) {
       System.out.println(limiter.acquire());
   }
   ```

   平滑预热限流的耗时是慢慢趋近平均值的。


参考
- [https://www.cnblogs.com/raoshaoquan/articles/6636067.html](https://www.cnblogs.com/raoshaoquan/articles/6636067.html)
- [https://www.cnblogs.com/duanxz/p/3465559.html](https://www.cnblogs.com/duanxz/p/3465559.html)