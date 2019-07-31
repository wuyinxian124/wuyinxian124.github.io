# 客户端提交app

## 基础
### 1. 通信协议
client 通过ApplicationClientProtocol 协议 与RM 交互。  
其包括两个部分  
ApplicationClientProtocol类 和protobuf 文件applicationclient_protocol.proto  

### 2. 协议封装
#### 2.1 第一层封装
为了简化client 跟 RM 的交互，yarn  提供抽象类org.apache.hadoop.yarn.applications.distributedshell.Client.YarnClient （具体实现为YarnClientImpl）  
YarnClient 继承自 AbstractService 是一个服务，起内部封装了一个对象timelineClient。  
YarnClient 内部有一个封装了ApplicationClientProtocol 协议的rmClient 对象。相关的协议操作都通过rmClient 对象进行。
#### 2.2 第二层封装
rmClient 对象当然不是直接new 一个实现都协议就行了。  
比如要解决一下RM是HA，你如何确保是跟主RM通信呢？RM 主备切换了，连接怎么切换呢？所有有第二层封装。  
不过这里先暂时跳过这一层，假设RM 不是HA 。   
RM 非HA的 ApplicationClientProtocol 协议调用栈为如下：  
```java
rmClient = ClientRMProxy.createRMProxy(getConfig(),ApplicationClientProtocol.class);  
                          | |
                          \ /
RMProxy.createRMProxy(final Configuration configuration,final Class<T> protocol, RMProxy instance);
          | |
          \ /
RMProxy.newProxyInstance(final YarnConfiguration conf,
      final Class<T> protocol, RMProxy<T> instance, RetryPolicy retryPolicy)
          | |
          \ /            
YarnRPC.create(conf).getProxy(protocol, rmAddress, conf);
```    
#### 2.3 第三层封装
核心类 YarnRPC 是rpc框架对外开放的操作接口，其对应实现类为 HadoopYarnProtoRPC。  
YarnRPC.create(conf) 获取的就是 HadoopYarnProtoRPC 对象  
所以前面提到的 YarnRPC.create(conf).getProxy(protocol, rmAddress, conf) 实现的逻辑就是  
HadoopYarnProtoRPC.getProxy(protocol, rmAddress, conf) ，其具体实现如下：
RpcFactoryProvider.getClientFactory(conf).getClient(protocol, 1, addr, conf)  

getClient 方法，是通过反射获得的对象    
pbClazz = localConf.getClassByName(getPBImplClassName(protocol))  
getPBImplClassName 方法是获取 protocol 对应的实现类，该实现类在固定位置   
假设protocol 是XXX ，所属包名是YYY ，则实现类在 YYY.impl.pb.client.XXXPBClientImpl  
比如 设置的协议是 ApplicationClientProtocol 。其获取的实现类就是  ApplicationClientProtocolPBClientImpl .  

到了这里 我们就知道  
通过  
```java
ClientRMProxy.createRMProxy(getConfig(),
          ApplicationClientProtocol.class);
```
方式创建的 rmClient  其实就是获得 ApplicationClientProtocolPBClientImpl对象.  

## 执行过程
### 1. 客户端处理
以提交app 方法为例，说明客户端提交app的请求是如何到达 resourceManager 的    
#### 1.1 ApplicationClientProtocolPBClientImpl 对象
提交app 的方法是
```java
  public SubmitApplicationResponse submitApplication(
      SubmitApplicationRequest request)
  throws YarnException, IOException;
```
上面已经知道 调用 rmClient.submitApplication  方法实际执行的是 ApplicationClientProtocolPBClientImpl.submitApplication 方法  
而 ApplicationClientProtocolPBClientImpl.submitApplication 实际执行的是
```java
proxy.submitApplication(null,requestProto)
```

#### 1.2 RPC 对象
要了解 proxy.submitApplication 执行过程，必须先研究 RPC 对象,因为 proxy 初始化过程如下：
```java
    RPC.setProtocolEngine(conf, ApplicationClientProtocolPB.class,
      ProtobufRpcEngine.class);
    proxy = RPC.getProxy(ApplicationClientProtocolPB.class, clientVersion, addr, conf);
```    
RPC.getProxy 方法实现为
```java
getProtocolProxy(protocol, clientVersion, addr, conf).getProxy();
```
虽然RPC.getProxy相关方法有很多 ，但最终执行的代码为：
```java
return getProtocolEngine(protocol, conf).getProxy(protocol, clientVersion,
addr, ticket, conf, factory, rpcTimeout, connectionRetryPolicy,
fallbackToSimpleAuth)
```
这里 getProtocolEngine 返回的就是 WritableRpcEngine ，所以proxy 对象取值就是 WritableRpcEngine.getProxy 方法返回的结果  

#### 1.3 WritableRpcEngine.getProxy 方法
WritableRpcEngine.getProxy 方法实现
```java
public <T> ProtocolProxy<T> getProxy(Class<T> protocol, long clientVersion,
                         InetSocketAddress addr, UserGroupInformation ticket,
                         Configuration conf, SocketFactory factory,
                         int rpcTimeout, RetryPolicy connectionRetryPolicy,
                         AtomicBoolean fallbackToSimpleAuth)
    throws IOException {
    T proxy = (T) Proxy.newProxyInstance(protocol.getClassLoader(),
        new Class[] { protocol }, new Invoker(protocol, addr, ticket, conf,
            factory, rpcTimeout, fallbackToSimpleAuth));
    return new ProtocolProxy<T>(protocol, proxy, true);
}
```
Proxy.newProxyInstance 为JDK 函数，其方法有三个参数，分别是类，接口，和invoke  
方法最终返回的是 ProtocolProxy 对象

#### 1.4 动态代理
JDK 动态代理
```JAVA
T proxy = (T) Proxy.newProxyInstance(protocol.getClassLoader(),
new Class[] { protocol }, new Invoker(protocol, addr, ticket, conf,
factory, rpcTimeout, fallbackToSimpleAuth));
```
Proxy.newProxyInstance()方法有三个参数：
1. 类加载器(Class Loader)  
2. 需要实现的接口数组  
3. InvocationHandler接口。  
通过下面这个例子 [动态代理](https://blog.csdn.net/Heyeverybody/article/details/50707683) 我们就可以知道 ：   
所有动态代理类的方法调用,都会交由InvocationHandler接口实现类里的invoke()方法去处理,这是动态代理的关键所在。  
所以实际只要关心 InvocationHandler.invoke 方法和 protocol（ApplicationClientProtocolPB） 的实现  

#### 1.5 动态代理Invoker 对象
前面说到动态代理 其中最终要的就是invoke 方法，我们看看 作为WritableRpcEngine 内部静态类的 Invoker.invoke 方法执行内容
```JAVA
    public Object invoke(Object proxy, Method method, Object[] args)
      throws Throwable {
      ...省略
      ObjectWritable value;
      try {
        value = (ObjectWritable)
          client.call(RPC.RpcKind.RPC_WRITABLE, new Invocation(method, args),
            remoteId, fallbackToSimpleAuth);
      } finally {
        if (traceScope != null) traceScope.close();
      }
      ...省略
      return value.get();
    }
```

#### 1.6 执行过程梳理  
ApplicationClientProtocolPBClientImpl.submitApplication() 方法  
proxy.submitApplication(null,requestProto)  
ProtocolProxy.submitApplication()  

#### 1.7 进入核心类 WritableRpcEngine  
要搞清 通过动态代理获取的 ProtocolProxy 对象调用 submitApplication()方法的执行逻辑，需要知道一点 WritableRpcEngine 对象的内容    
WritableRpcEngine 入口包含一个静态方法 ensureInitialized 和new Invoker（继承InvocationHandler）对象  
a. 静态方法  
ensureInitialized()，初始化一个 对象 WritableRpcInvoker，并加入到 rpcKindMap临时缓存中  
b. Invoker 初始化包括  
先获取一个 Client.ConnectionId 对象  
然后初始化一个 Client 对象，并将其放到clients 缓存中  
#### 1.8 如果执行proxy 某个方法，则会触发 WritableRpcEngine.Invoker.invoke()方法
进而调用 client.call ,进而创建一个 Call 对象  
实现逻辑如下  
![](/images/clientRm1.png)

![](/images/clientRpc11.png)

![](/images/clientRpc12.png)

#### 1.9 Client对象


### 2. 服务器处理
