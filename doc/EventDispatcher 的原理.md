先说下 EventDispatcher 的原理：

EventDispatcher 核心功能：
1.管理 listener
2.触发事件

Entry 是说将 event 分组，因为每个 event 可能有多个不同的 listener 关注.

AbstractEventListener 表征感兴趣的事件，子类需要重写 interest 和 onEvent 方法.

再说下 listener 在 nacos 中的实现类:

1.LongPollingService

1.1ClientLongPolling 对客户端长轮询的封装.
1.2allSubs 存储的就是客户端的长连接.
1.3com.alibaba.nacos.config.server.service.LongPollingService#onEvent 触发的是 DataChangeTask 任务.


2.AsyncNotifyService

2.1AsyncNotifyService 感兴趣 ConfigDataChangeEvent
2.2为每一个节点都创建了一个 NotifySingleTask
2.3使用 AsyncTask 执行任务. 如果某个节点不健康，则异步执行任务. 否则同步执行.
2.4同步任务地址：http://{0}{1}/communication/dataChange?dataId={2}&group={3}
2.5在 com.alibaba.nacos.config.server.service.dump.processor.DumpChangeProcessor#process 中，如果发现本地缓存的 md5 值和数据库中的不一致，发送 LocalDataChangeEvent 事件. 然后 LongPollingService 监听到该事件通知客户端(会不会重复被通知？).
2.6更新磁盘文件
2.7ConfigCacheService.reloadConfig() 是重新加载 nacos 内置的一些 dataId 的配置：ClientIpWhiteList,AggrWhitelist,SwitchService.



参考：https://blog.csdn.net/u011177064/article/details/104126250