### [Android高频面试专题 - 架构篇（二）okhttp面试必知必会](https://cloud.tencent.com/developer/article/1601358)

+ #### Http的报文结构？请求报文、响应报文

+ #### Http协议的版本？最新的Http2有什么优势？

+ #### OKHttp的优势？怎样减少延时、处理失败请求和避免重复请求？

+ #### 看过OKhttp源码吗？简单介绍一下？

+ #### `dispatcher`内部有几个队列、几个线程池？他们的作用是什么？

+ #### OkHttp最多可以并发执行多少个请求(maxRequests)？每个host可以最多并发执行几个请求(maxRequestsPerHost)？

+ #### 默认拦截器有几种？每一种的功能是什么？

+ #### 拦截器的调用顺序？

+ #### OkHttp同步/异步执行的过程？

+ #### `RetryAndFollowUpInterceptor`的功能？

+ #### `BridgeInterceptor`的功能？

+ #### `CacheInterceptor`的功能？

+ #### `ConnectInterceptor`的功能？

+ #### `CallServerInterceptor`的功能？

+ #### OKHttp使用的是socket还是URLConnection？如何创建、连接、输入输出流操作？

+ #### OKHttp中运用了那些设计模式？举例说明

-----------

### [OkHttp 3.7源码分析（一）——整体架构](https://developer.aliyun.com/article/78105)

+ #### OKHttp有哪些优点？（协议、效率、设计实现）

+ #### 一个RealCall实例可以多次调用吗？为什么？

+ #### OKHttp的整体架构大致如何？分为几层？每层干什么？

+ #### 连接池的工作过程？call/realCall、stream/StreamAllocation、RealConnection、连接池之间的关系？

------------

### [OkHttp 3.7源码分析（二）——拦截器&一个实际网络请求的实现](https://developer.aliyun.com/article/78104)

### [OkHttp 3.7源码分析（三）——任务队列](https://developer.aliyun.com/article/78103)

+ #### 线程池的有点？

+ #### 有几种任务队列？每一种任务队列的作用？

+ #### 异步请求最大请求数是多少？单一host的最大请求数是多少？

+ #### Application Interceptors和Network Interceptors¶有什么不同？[Interceptors](https://square.github.io/okhttp/interceptors/)

------------

### [浏览器 HTTP 协议缓存机制详解](https://my.oschina.net/leejun2005/blog/369148)

+ #### 缓存的分类有几种？

+ #### 浏览器第一次请求信息的大致流程是？

+ #### 浏览器再次请求信息的流程是？

+ #### Expire/Cache-control/last-modified/etag之间有什么关系和不同？

+ #### 既然已经有了last-modified，为何还要使用etag？

### [OKHTTP之缓存配置详解](https://www.jianshu.com/p/9b2366f5e97a)

+ #### Cache-Control一般有那些值？莓种植的作用是什么？

+ #### Pragma指令是什么作用？

+ #### OKHttp的response类型有哪些？

+ #### 使用拦截器进行缓存有什么缺点？如何单独为每一次请求设置特定缓存？

### [OkHttp缓存原理](https://juejin.cn/post/6850418120729985038)

+ #### 说一说常用的Http协议请求字段/响应字段？

+ #### cacheInterceptor能缓存post/patch/put/delete方法吗？

+ #### CacheStrategy的执行策略？cache中使用什么作为不同缓存的key值？

+ #### CacheStrategy如何获取缓存cacheResponse中的头信息？尝试描述一下头信息解析之后通过get()/compute()构建CacheCategory的过程？

+ #### 拿到CacheStrategy后，cacheInterceptor如何处理？

+ #### 说一说常用的Http返回码？

+ #### 配置了cache后，okhttp的缓存文件有几种？有什么不同？

+ #### 磁盘上的日志文件是如何关联Cache并支持增删改查的呢？

+ #### DiskLruCache.Entry与dirty/clean文件以及DIRAY/CLEAN in .journal文件之间的关系？与.0、.1文件之间的关系？

+ #### DiskLruCache.Editor如何通过DiskLruCache.Entry以及fileSystem读写缓存文件内容？

+ #### cache如何保存缓存内容到本地文件(put方法)？ （cache-put-Cache.entry-DiskLruCache.Editor-entry.writeTo-RealCacheRequest-okio

+ #### 没有服务端支持是否可以实现纯客户端缓存以及如何实现？

### [OkHttp 3.7源码分析（四）——缓存策略](https://developer.aliyun.com/article/78102)

+ #### Http的缓存策略是什么？

+ #### DiskCacheLru初始化时如何创建cache？（DiskLruCache.readJournalLine,DiskLruCache.processJournal）

+ #### 为避免日志文件不断膨胀，如何对日志进行重建精简？（DiskLruCache.rebuildJournal）

+ #### 当冗余日志过多时，如何重建日志？（cleanUpRunnable线程）

+ #### 什么是cache的SnapShot？每次获取SnapShot后，会修改日志文件吗？会重建日志文件吗？

+ #### OKHttp是如何保证Cache原子性与可用性的？

+ #### OKHttp如何防止日志过分膨胀？

------------

### 关于Transmitter/ExchangeFinder/Exchange/route/routeSelector/proxy/StreamAllocation/HttpCodec之间的具体分析，请看下面几篇文章：

+ ### [OkHttp：源码详解之连接拦截器（五）](https://blog.csdn.net/baidu_32237719/article/details/109743702)

  + #### HttpUrl：对URL/URI进行一层封装，可以获得uri的详细信息，scheme/username/password/host/port等

  + #### OKHttp如何实现TCP+TLS握手实现的？（简述）
  
+ ### [OkHttp源码解析 （三）——代理和路由](https://www.jianshu.com/p/63ba15d8877a)

+ ### [懂了，原来 OkHttp 是这样建立 HTTPS 连接的](https://www.jianshu.com/p/f7972c30fc52)

+ ### [okhttp源码学习笔记（二）-- 连接与连接管理](https://www.jianshu.com/p/671a123ec163)

+ ### [OkHttp 源码学习笔记（三） 数据交换的流 HTTPCodec](https://www.jianshu.com/p/92ce01caa8f0)
