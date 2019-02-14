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

参考
- [https://www.cnblogs.com/raoshaoquan/articles/6636067.html](https://www.cnblogs.com/raoshaoquan/articles/6636067.html)