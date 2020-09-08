---
description: YARN active rm停止，standby rm 切主失败，两个rm 都是standby，YARN 服务异常
---

# YARN RM 无主

## 现象两个RM 都是standby

![](/images/rm1.png)

而且无法通过重启解决  
如果设置如下参数  yarn.resourcemanager.recovery.enabled=false rm 启动正常

## 问题定位
通过上面的现象分析，其实是两个问题第一 为什么active RM 会挂，而触发来主备切换 。第二 RM 主备切换过程为什么会失败。   
第一个问题，可能原因很多，要通过rm 日志具体分析，直接找 error  日志。   
第二个问题怀疑是主备切换过程中，RM recovery app 数据异常。  

### 1. active RM 日志
从active rm 日志中，发现如下异常   

![](/images/rm2.png)

该异常可以看出，rm 挂掉的原因是 提交过来的一个app ,触发来创建队列操作，但是指标却已经存在该队列，因为 APP_ADDED 事件处理是在 rm 入口类 ResourceManager ，所以直接导致 RM 异常退出

### 2. standby RM 日志
从原来到standby rm 日志中 ，发现如下异常

![](/images/rm3.png)

通过该异常可以看出，standby rm 已经被触发尝试做主节点，但是在恢复app 状态过程中出现异常，但是切主失败   

上面两个原因导致整个YARN 集群没有可用的RM

## 原因分析
从上面两个异常，可以看出导致问题的原因是 提交的app 触发rm 创建新的队列，但是这个队列已经在指标中存在了。

### 1. 分析 是不是特定的app 才会触发该问题
从日志中过滤 app ID 发现RM 接收的app 提交命令中 队列指定值有空格  

![](/images/rm4.png)

而这个空格是用户后台提交都hive 命令行中指定的  

![](/images/rm6.png)

该问题可以在测试环境复现，会导致原来的active rm 异常退出

### 2. 代码分析
为什么带空格会触发新建队列？需要通过代码进一步确认  
根据FairScheduler.java 方法 assignToQueue 调用堆栈，我们发现 用户传入的 队列命令 **root. data_enquiry** 会传递到 RM FSQueueMetrics.java 的如下方法：
```
public static FSQueueMetrics forQueue(String queueName,
                                      Queue parent,
                                      boolean enableUserMetrics,
                                      Configuration conf)
```
查看具体方法实现
```
public synchronized
static FSQueueMetrics forQueue(String queueName, Queue parent,
    boolean enableUserMetrics, Configuration conf) {
  MetricsSystem ms = DefaultMetricsSystem.instance();
  QueueMetrics metrics = queueMetrics.get(queueName);
  if (metrics == null) {
    metrics = new FSQueueMetrics(ms, queueName, parent, enableUserMetrics, conf)
        .tag(QUEUE_INFO, queueName);

    // Register with the MetricsSystems
    if (ms != null) {
      metrics = ms.register(
              sourceName(queueName).toString(),
              "Metrics for queue: " + queueName, metrics);
    }
    queueMetrics.put(queueName, metrics);
  }

  return (FSQueueMetrics)metrics;
}
```
传入的参数queueName 为带空格的参数 **root. data_enquiry**  ，但是通过 sourceName(queueName) 调用之后，会转成QueueMetrics,q0=root,q1=data_enquiry ，注意子队列前面的空格已经没有。  
因为切分代码是调用 guava 工具类 Splitter对象已经将字符串前面的空格清理了。
```
Splitter Q_SPLITTER =
        Splitter.on('.').omitEmptyStrings().trimResults();
Q_SPLITTER.split(name);
```
## 解决办法
如果用户没有创建对应的队列，只是在提交app的时候指定带空格的队列是没有问题的，因为rm缓存中能够找到队列缓存   
但是如果用户，先带空格提交app 后不带空格提交app 则依然会有问题  
所以关键还是使得rm 中 队列缓存和指标缓存所使用的关键字是一一对应的。  
即使用Splitter 来切分queueName
有两个办法：   
1. 在fairScheduler.java 修改  
2. 在QueuePlacementRule 修改

第一种方式 不是通用方法，所以我们选择第二种
结合fairScheduler.xml 的配置
```
<queuePlacementPolicy>
    <rule name="specified" />
    <rule name="tbds" />
    <rule name="default" queue="default"/>
</queuePlacementPolicy>
```
我们修改specified 对应的 QueuePlacementRule 代码
```
if (!requestedQueue.startsWith("root.")) {
  requestedQueue = "root." + requestedQueue;
}
requestedQueue = StringUtils.join(Q_SPLITTER.split(requestedQueue),".");
return requestedQueue;
```

### 补充
环境恢复方法（概述）  
1. 先停止YARN   
2. 查看数据   
查看目录ls /rmstore/ZKRMStateRoot/RMAppRoot  
3. 删除   
不为空则使用该命令rmr /rmstore/ZKRMStateRoot/RMAppRoot/* 删除目录文件   
4. 重启YARN   
