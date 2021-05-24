先看下 Client 端 NacosConfigService 的核心组件：


两大核心组件：ServerHttpAgent 和 ClientWorker.

需要注意的是，在 ServerHttpAgent 中，有一个 ServerListManager 成员变量. 这个类就是为了获取 nacos server 服务列表.

现在支持两种方式，一种是固定 ip 的，另一种是从服务器获取 nacos server 列表.

需要注意的是，在 ServerHttpAgent 中，有一个 ServerListManager 成员变量. 这个类就是为了获取 nacos server 服务列表.

现在支持两种方式，一种是固定 ip 的，另一种是从服务器获取 nacos server 列表.


在 ClientWorker 中，起了一个定时任务，每 10 毫秒执行一次，调用 checkConfigInfo 方法.

对比 client 端缓存的配置和 server 端的配置是否一致，默认情况下一次对比 3000 条.

那么问题来了？ClentWorker 中的 cacheMap 是什么时候更新的了？

一步步追踪，最后发现是在 addListener 的时候.

public void addListener(String dataId, String group, Listener listener) throws NacosException {
    worker.addTenantListeners(dataId, group, Arrays.asList(listener));
}

那么问题来了？如果客户端不注册 listener，能否收到 server 主动推送的 config change 通知了？

按照现有逻辑，应该是只能依赖 checkConfigInfo 这个方法的检查才能收到通知.

addListener/removeListener 会导致 ClientWorker 中 cacheMap 的变化.

所以，对于 client config server 而言，核心点有两个，一个是访问 server 进行发布配置、删除配置等操作，另一个是 ClientWorker.

ClientWorker.run(){
    
    // 1.check failover config
    // 优先用 failover file 来更新 cacheData.
    checkLocalConfig()
    // 如果 md5 发生改变，则通知 listener.
    checkListenerMd5()

    // 2.check server config
    // 从 cacheData 中找 isUseLocalConfigInfo=false 的数据，拼接 dataId、group 等, 调用 checkUpdateConfigStr 方法.
    checkUpdateDataIds()
    // 在这个方法中，会向 nacos server 发送 http 请求. /v1/cs/configs/listener. 那么问题来了？向那一台发起请求了？
    // 在 ServerListManager 中有一个属性：currentServerAddr. 向这台服务器发送请求.
    // 如果请求 nacos server 成功，那么会解析结果, 返回更新的集合.
    checkUpdateConfigStr()

    // 3.拿到 2 的 changedGroupKeys 后，进行遍历.
    // 向 /v1/cs/configs 发起请求，请求成功后，会将配置保存到 snapshot.
    // 一个配置项一个文件.
    // 这里使用的是 agent.getName() 和 failover 的那个名称是一致的.
    // LocalConfigInfoProcessor.saveSnapshot(agent.getName(), dataId, group, tenant, result.getData());
    // 更新 cacheData.
    // 通知 cacheData 上的 listener.

    // 4.将这个任务再加入到 executorService 中.
}


问题：config failover file 是在哪保存的？存储格式是什么？

1.存储格式很简单，就是一个配置项一个文件.
2.存储文件名：${envName}_nacos/snapshot/group/dataId

接着看下 com.alibaba.nacos.config.server.controller.ConfigController#listener

如果支持长轮询的话，就等待，否则就直接比较.

com.alibaba.nacos.config.server.utils.MD5Util#compareMd5 只要 client 的某个 key 的 md5 和 server 的 md5 不一致，说明发生了改变.

传送到 server 端的 key：

dataId|group|md5|tenant,xxxxxx

clientMd5Map key is group key(dataId|groupId|tenant), value is md5.

compareMd5OldResult & compareMd5ResultString 应该是处理兼容问题.
