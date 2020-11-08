# 提纲
[toc]

## 简介
YugaByte 

### Proxy
client使用Proxy作为代理来发送远程调用到远端的RPC服务。调用可以是同步的，也可以是异步的，同步调用仅仅是在异步调用的基础上进行了封装（即同步调用在异步调用的基础上会等待异步调用的回调被触发）。

为了执行一个远程调用，用户需要提供：方法名，请求对应的protobuf，响应对应的protobuf，一个RpcController和一个回调方法。

每一个正在调用过程中的调用都唯一对应一个RpcController，在执行RPC调用之前可以配置该RpcController，当前对RpcController的配置只支持关于超时时间的配置，未来可能添加比如优先级，tracing等相关的配置。

### Proxy Cache
Proxy Cache中可能会包含多个Proxies，所以他是一个map：ProxyKey -> Proxy，其中ProxyKey是由HostAndPort + Protocol组成。

```
class ProxyCache {
 public:
  # 构造方法
  explicit ProxyCache(ProxyContext* context)
      : context_(context) {}

  # 给定ProxyKey获取对应的Proxy
  std::shared_ptr<Proxy> Get(const HostPort& remote, const Protocol* protocol);

 private:
  # ProxyKey的定义
  typedef std::pair<HostPort, const Protocol*> ProxyKey;

  # 用于下面的unordered_map的hash方法
  struct ProxyKeyHash {
    size_t operator()(const ProxyKey& key) const {
      size_t result = 0;
      boost::hash_combine(result, HostPortHash()(key.first));
      boost::hash_combine(result, key.second);
      return result;
    }
  };

  # 通常就是Messenger结构
  ProxyContext* context_;
  std::mutex mutex_;
  # ProxyKey到Proxy的映射
  std::unordered_map<ProxyKey, std::shared_ptr<Proxy>, ProxyKeyHash> proxies_;
};
```

### Messenger
Messenger用于收发消息，如果是server端的Messenger，它还有一个Acceptor，用于接收新的连接请求。用户通常并不主动和Messenger打交道，而是和Proxy打交道，但是Proxy是借助于Messenger来实现RPC调用的。

### RPC Server



## 参考
[kudu RPC](https://github.com/cloudera/kudu/blob/master/docs/design-docs/rpc.md)