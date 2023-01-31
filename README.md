# CaiCache
## 基于Go的分布式缓存框架实现

### 架构世界，缓存为王
分布式缓存解决了什么样的问题？

**缓存**：对于某种耗时操作的频繁请求，将该操作的结果暂存，再次遇到改请求，则直接返回相应结果。

缓存的引入可以快速返回请求结果，但也会带来其他的问题。

缓存一般保存在内存，内存不够了怎么办？-> 合理的淘汰策略

缓存结果的写入和读取一般是并行的，并发读写的冲突如何解决？-> 加锁

单机性能不够如何解决？-> 水平扩展：分布式缓存，垂直扩展：增加单节点性能

CaiCache，在考虑资源控制、淘汰策略、并发、分布式节点通信等各个方面的问题下的分布式缓存系统实现。

参考：memcached, groupcache

支持特性：
- 单机缓存和基于HTTP通信的分布式缓存
- 最近最少访问LRU(Least Recently Used)缓存策略
- 基于Go锁机制的缓存击穿解决方案
- 基于一致性哈性的负载均衡实现
- 基于Protobuf的节点通信
- ...

前置知识点：
- Go语言语法基础
- Go test单元测试基础
- Go Protobuf基础
- Go Web服务基础

### LRU缓存淘汰策略
为提高读取效率，缓存全部存在内存，但内存有限，无法无限制增加数据，必须有合理的淘汰策略移除无用的数据。

补充：三种常用的淘汰策略，FIFO，LFU，LRU
- FIFO: 先进先出，越早添加越先淘汰
  - 实现基于队列，新增记录添加到队尾，每次内存不够时，淘汰队首。
  - 部分数据最早添加，也最长被访问，但基于FIFO会出现频繁的添加，淘汰现象，降低缓存命中率。
- LFU: 最少使用，淘汰缓存中访问频率最低的记录
  - 实现基于一个按照访问次数排序的队列，每次访问，访问次数加1，队列重新排序，淘汰访问次数最少的即可。
  - 维护记录的访问次数，对内存消耗较高；受到历史数据的影响较大。
- LRU: 最近最少使用，如果数据最近被访问过，那么将来被访问的概率也会更高
  - 实现基于维护一条队列，如果某条记录被访问了，则移动到队尾，每次淘汰队首元素即可。

#### LRU算法实现
Cache 数据结构：存储键-值映射关系的map，基于双向链表的队列。
基于Cache的 `New(), Get(), RemoveOldest(), Add()` 方法。
参考代码 `/lru/lru.go`

### 单机并发缓存 && 交互实现
基于Cache的LRU算法实现，无法满足并发读写的需要，这里使用`sync.Mutex`来实现单机缓存的并发读写。

补充：多个协程同时读写同一个变量，在并发度较高的情况下，会发生冲突，确保一次只有一个协程可以访问该变量以避免冲突，称为"互斥"。
`sync.Mutex` 是一个互斥锁，可以由不同的协程加锁和解锁。 加锁，释放锁: `Lock(), Unlock()`

这里抽象一个只读数据结构`ByteView`用来表示缓存值，实现新的支持并发读写的cache数据结构(基于Cache，同时增加mutex锁)
实现: 实例化 lru，封装 get 和 add 方法，并添加互斥锁 mu。
（add 方法中引入了延迟初始化 lazy initialization, 即先判断c.lru是否为nil, 再创建实例，个对象的延迟初始化意味着该对象的创建将会延迟至第一次使用该对象时，
可以用于提高性能，并减少程序内存要求。）

这里梳理下用户使用框架中的查询，缓存值的存储和获取流程。 
```text
                          是
接收 key --> 检查是否被缓存 -----> 返回缓存值 ⑴
                 |  否                        是
                 |-----> 是否应当从远程节点获取 -----> 与远程节点交互 --> 返回缓存值 ⑵
                               |  否
                               |-----> 调用`回调函数`，获取值并添加到缓存 --> 返回缓存值 ⑶

```
对于缓存不存在的情况，我们这里应当给定一个回调函数，让程序员调用该函数，得到源数据。（即无需支持多种数据源的配置）

对于每个缓存空间，我们都进行唯一性命名，并保存在`Group`中，包含`name, getter Getter, mainCache cache`
`getter`用来缓存未命中时获取源数据的回调, `mainCache`为一开始实现的并发缓存。

### Http服务端构建
上述已经实现了单机版的并发缓存以及交互，这里建立基于HTTP的通信机制实现节点间通信，将单机缓存升级为分布式缓存。

Go 语言提供了`net/http`标准库, 可以非常方便的搭建HTTP服务端和客户端。
`http.ListenAndServe("localhost:8989", &s)`

> http.ListenAndServe 接收 2 个参数，第一个参数是服务启动的地址，
> 第二个参数是 Handler，任何实现了 ServeHTTP 方法的对象都可以作为 HTTP 的 Handler。
> (接口型函数的具体应用)

这里创建`HTTPPool`， 作为承载节点间HTTP通信的核心数据结构。

### 一致性哈希
为了实现分布式缓存，相同的数据请求应尽可能访问相同节点。（数据要么已缓存在该节点，要么需要重新从数据源获取）
简单的对key取hash值，值作为节点号，但无法应对节点数量变化的场景，容易引发缓存雪崩。
> 缓存雪崩：缓存在同一时刻全部失效，造成瞬时DB请求量大、压力骤增，引起雪崩。常因为缓存服务器宕机，或缓存设置了相同的过期时间引起。


