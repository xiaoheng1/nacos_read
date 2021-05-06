NacosConfigServer 初始化流程.

1.NacosConfigService 的构造
    1.1初始化 namespace
    1.2初始化 HttpAgent(一个关于发起 http 请求的代理，httpAgent 有关于 token 续期的处理)
    1.3HttpAgent 中有一个 ServerListManager，它是管理 nacos ip 列表的，分为固定方式和动态方式. 调用其 start 方法会开启定时任务周期的从 url 请求 nacos ip 列表.
    1.4初始化 ClientWorker
        1.4.1 executor 线程池没 10 毫秒执行一次 checkConfigInfo，处理配置变更逻辑.


LongPollingRunnable 
1.本地检查
    1.1首先取出与该 taskId 相关的 CacheData，然后对 CacheData 进行检查，包括本地配置检查和监听器的 md5 检查，本地检查主要是做一个故障容错，当服务端挂掉后，nacos 客户端可以从本地的文件系统中获取相关的配置信息.
2.服务端检查
    2.1从当服务端获取哪些值发生了变化的 dataId 列表，通过 getServerConfig 方法，根据 dataId 到服务端获取最新的配置，接着将最新的配置信息保存到 CacheData 中，最后调用 CacheData 的 checkListenerMd5 方法.


CacheData 是啥？

属性：dataId, group, content, taskId，listener，md5 


Nacos 并不是通过推的方式将服务端最新的配置信息发送给客户端，而是客户端维护了一个长轮询的任务，定时去拉取发生变更的配置信息，然后将最新的数据推送给 Listener 的持有者.

Nacos 采用的 Pull 模式，是结合了 Push 和 Pull 两者的优势. 客户端采用长轮训的方式定时发起 pull 请求，去检查服务端配置信息是否发生了变更，如果发生了变更，则客户端会根据变更的数据获得最新的配置.
如果客户端发起 pull 请求后，发现服务端的配置和客户端的配置保持一致，那么服务端会先 Hold 住这个请求，也就是服务端拿到这个连接后在指定的时间内一直不返回结果，知道这段时间内配置发生变化，服务端会把原来
 Hold 住的请求进行返回. 如果一直没有发生变更，会设置一个定时任务，延迟 29.5s，并把当前的客户端长轮询连接加入到 allSubs 队列.

所以客户端长轮询返回的时机：
1.等待 29.5 秒触发自动检查机制.
2.在 29.5 秒内的任意一个时刻，配置项有变更，监听到该事件的任务会遍历 allSubs 队列，找到发生变更的配置项对应的 ClientLongPolling 任务，将变更的数据通过该任务中的连接机械能返回，完成这一次的推送. 

publishConfig 的流程.

发起 http 请求到 com.alibaba.nacos.config.server.controller.ConfigController#publishConfig 中进行处理.

对应的就是数据入库处理.

需要注意：
1.ConfigChangePublisher.notifyConfigChange 通知订阅者配置发生了变更.
2.参见：com.alibaba.nacos.config.server.service.dump.processor.DumpChangeProcessor#process, 这个会转到 com.alibaba.nacos.config.server.controller.CommunicationController 中.
3.总的来说，就是某一台服务器收到更新请求后，先更新本地数据库的数据，然后发布一个 ConfigDataChangeEvent 事件.该事件会让每个服务收到某个配置更改了，然后每个服务开始执行流程，每个服务会新建一个 task 任务放入自己的任务队列中，每个服务后台的线程会总队列中执行该任务.任务包括如果为非单机或者 mysql，会刷新本地缓存文件，同时会更新内存中的 CacheItem 缓存内容信息，最后触发 LocalDataChangeEvent.


getConfig 的流程.
1.优先从 failover 获取配置
2.从 nacos 服务器获取配置
3.从本地快照文件中获取.


参考：
1.https://blog.csdn.net/god_86/article/details/106394522    
2.https://blog.csdn.net/liyanan21/article/details/89161313
3.https://blog.51cto.com/u_15091660/2604721
4.https://blog.csdn.net/qq_19414183/article/details/112366177
5.https://www.sfanonline.cn/Nacos%E9%85%8D%E7%BD%AE%E4%B8%AD%E5%BF%83%E8%AE%BE%E8%AE%A1%E6%80%9D%E6%83%B3/
6.https://www.jianshu.com/p/1dd59c113287