nacos-client 源码分析


IConfigRequest 的作用，我理解是配置请求.

void putParameter(String key, Object value);

Object getParameter(String key);

IConfigContext getConfigContext();


ConfigRequest 中的 param 和 contextConfig 有啥区别？


IConfigFilterChain filter 链.

ConfigFilterChainManager 实现 IConfigFilterChain 接口，在构造 ConfigFilterChainManager 的时候，通过 SPI 加载 IConfigFilter 的所有实现, 并将 filter 加入到 filters 中.

filter 链的设计：

1.将 filter 构造成用指针连接起来的结构，相当于俄罗斯套娃那种结构，例如 dubbo.

经典代码：

private static <T> Invoker<T> buildInvokerChain(final Invoker<T> invoker, String key, String group) {
    Invoker<T> last = invoker;
    List<Filter> filters = ExtensionLoader.getExtensionLoader(Filter.class).getActivateExtension(invoker.getUrl(), key, group);

    if (!filters.isEmpty()) {
        for (int i = filters.size() - 1; i >= 0; i--) {
            final Filter filter = filters.get(i);
            last = new FilterNode<T>(invoker, last, filter);
        }
    }

    return last;
}

这个相当于 invoker 是一条主线，将 filter 封装成 invoker，然后 invoker 套 invoker.

2.还有一种就是将 filter 统一放到一个地方管理起来，然后传递 position，例如：nacos.

经典代码：

public void doFilter(IConfigRequest request, IConfigResponse response) throws NacosException {
    new VirtualFilterChain(this.filters).doFilter(request, response);
}

private static class VirtualFilterChain implements IConfigFilterChain {
    
    private final List<? extends IConfigFilter> additionalFilters;
    
    private int currentPosition = 0;
    
    public VirtualFilterChain(List<? extends IConfigFilter> additionalFilters) {
        this.additionalFilters = additionalFilters;
    }
    
    @Override
    public void doFilter(final IConfigRequest request, final IConfigResponse response) throws NacosException {
        if (this.currentPosition != this.additionalFilters.size()) {
            this.currentPosition++;
            IConfigFilter nextFilter = this.additionalFilters.get(this.currentPosition - 1);
            nextFilter.doFilter(request, response, this);
        }
    }
}

核心点在于 currentPosition 的自增. 还有就是 filter 中需要手动调用 chain.doFilter(request, response)

新增 filter 的测试代码：




ConfigRequest|ConfigResponse 代码优化


待定

HttpAgent http 请求代理类.


public interface HttpAgent extends Closeable {
    
    // start to get nacos ip list.
    void start() throws NacosException;
    
    
    HttpRestResult<String> httpGet(String path, Map<String, String> headers, Map<String, String> paramValues,
            String encoding, long readTimeoutMs) throws Exception;
    
    HttpRestResult<String> httpPost(String path, Map<String, String> headers, Map<String, String> paramValues,
            String encoding, long readTimeoutMs) throws Exception;
    
    HttpRestResult<String> httpDelete(String path, Map<String, String> headers, Map<String, String> paramValues,
            String encoding, long readTimeoutMs) throws Exception;
    
    String getName();
    
    String getNamespace();
    
    String getTenant();
    
    String getEncode();
}

AK/SK 

AK：Access Key Id 用于标示用户
SK: Secret Access Key 是用户用于加密认证字符串和用来验证认证字符串的密钥，其中 SK 必须保密.

STS 临时凭证：AK/SK 是永久授权方式，STS 临时凭证是说一段时间有效. 

1.客户服务器向存储服务器请求 STS 服务临时凭证
2.客户服务器将临时凭证发送到移动端
3.移动端使用临时凭证访问存储服务器.

参考：https://doc.bscstorage.com/doc/s2/sts/sts.html

ServerHttpAgent 代码优化


MetricsHttpAgent 包含统计信息的代理.


NacosConfigService 主要功能是 publishConfig, getConfig, addListener 等.

addListener 是为 CacheData 添加 listener.

// listener for watch config.
public interface Listener {
    
    Executor getExecutor();
    
    void receiveConfigInfo(final String configInfo);
}

AbstractConfigChangeListener 监听配置变更的 listener.

PropertiesListener 监听配置文件的 listener.

ConfigChangeParser 字面意思，就是配置改变解析器. 那么它所隐含的含义比如说是数据变更后以文本的方式推送？

待确定.


CacheData

CacheData 是配置文件缓存，应用从 CacheData 中获取文件内容，CacheData 由客户端线程不断拉取更新.

CacheData 还和 listener 相关.

ClientWorker 客户端的入口.


参考：https://www.codenong.com/cs105500734/