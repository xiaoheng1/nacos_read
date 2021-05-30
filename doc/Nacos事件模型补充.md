Nacos 事件模型补充：

在 Nacos 中，事件模型模块，出镜最高的是 NotifyCenter. 我们先从这个入手，看下其是如何设计的.

NotifyCenter 代码注释第一行：统一事件通知中心.

其中有一个很重要的属性：

private static BiFunction<Class<? extends Event>, Integer, EventPublisher> publisherFactory = null; 创建 EventPublisher.

// event 对应的 publisher.
private final Map<String, EventPublisher> publisherMap = new ConcurrentHashMap<String, EventPublisher>(16) 

EventPublisher 是如何加载的了？

通过 SPI 加载其扩展实现.

目前 EventPublisher 有两个默认实现，一个是 DefaultPublisher，另一个是 DefaultSharePublisher.

我们先看下 EventPublisher 的接口定义.

public interface EventPublisher extends Closeable {
    
    void init(Class<? extends Event> type, int bufferSize);
    
    long currentEventSize();
    
    void addSubscriber(Subscriber subscriber);
    
    void removeSubscriber(Subscriber subscriber);
   
    boolean publish(Event event);
    
    void notifySubscriber(Subscriber subscriber, Event event);
}

总的来说就是注册监听，发布事件，然后通知.

DefaultPublisher 首先实现了 EventPublisher 接口，并继承 Thread. 这将表明 DefaultPublisher 的工作方式，有一个后台线程会一直跑，然后通知.

DefaultPublisher {
    1.subscribers -> set
    2.queue -> blockingQueue
}

然后我们关注下其 run 方法. 在其 run 方法中调用了 openEventHandler 方法，这是一个死循环，一直处理队列中的 event.


// 主要是从队列中拿出一个 event，然后通知 subscriber.
void openEventHandler() {
    try {
        
        // This variable is defined to resolve the problem which message overstock in the queue.
        int waitTimes = 60;
        // To ensure that messages are not lost, enable EventHandler when
        // waiting for the first Subscriber to register
        for (; ; ) {
            if (shutdown || hasSubscriber() || waitTimes <= 0) {
                break;
            }
            ThreadUtils.sleep(1000L);
            waitTimes--;
        }
        
        for (; ; ) {
            if (shutdown) {
                break;
            }
            final Event event = queue.take();
            receiveEvent(event);
            UPDATER.compareAndSet(this, lastEventSequence, Math.max(lastEventSequence, event.sequence()));
        }
    } catch (Throwable ex) {
        LOGGER.error("Event listener exception : {}", ex);
    }
}

如何通知 subscriber 了？

如果 subscriber 中有 executor 的话，那么让这个 executor 执行通知任务(调用 subscriber.onEvent 事件). 否则同步调用 subscriber.onEvent 事件.

然后关注下其 publish 方法:

public boolean publish(Event event) {
    // 检查 publisher 是否启动了.
    checkIsStart();
    // 消息入队.
    boolean success = this.queue.offer(event);
    if (!success) {
        LOGGER.warn("Unable to plug in due to interruption, synchronize sending time, event : {}", event);
        // 不成功直接消费这条消息.
        receiveEvent(event);
        return true;
    }
    return true;
}

问题：每一个 event 都要通知所有 subscriber 吗？

这个目前是在各个 subscriber 的 onEvent 中进行处理的.


DefaultSharePublisher 注释第一行写到：为慢 event 设计的共享事件发布者.

在 DefaultSharePublisher 中重写了 receiveEvent. 首先会基于 slowEventType 去 subMapping 中捞一遍 subscriber 集合，然后调用父类的 notifySubscriber 方法.


Subscriber 所有订阅者的抽象接口.

public abstract class Subscriber<T extends Event> {
    
    public abstract void onEvent(T event);
    
    // 有了 subscribeType，其实可以优化 DefaultPublisher，先基于类型做判断，然后再通知 subscriber，这样就不用全量通知了.
    public abstract Class<? extends Event> subscribeType();
    
    public Executor executor() {
        return null;
    }
    
    public boolean ignoreExpireEvent() {
        return false;
    }
}

上面的是发布订阅模式，其实还可以衍生出监听模式.

一个思路就是在 subscirber 中注册 listener，然后在 subscriber 的 onEvent 中进行 listener 通知. 这就形成了三级结构.

NotifyCenter -> Publisher/Subscriber -> Listener.

参考：com.alibaba.nacos.client.naming.event.InstancesChangeNotifier#onEvent