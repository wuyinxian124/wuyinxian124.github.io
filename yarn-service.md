# yarn service 分析

---
description: >-
  yarn 将生命周期较长的对象，用基于服务化的模式管理。服务化并不是将服务线程化或者是进程化，亦或者是docker化，它可以简单的理解为一种
  管理生命周期的抽象方式。
---

### 1. 什么是服务化
yarn 将生命周期较长的对象，用基于服务化的模式管理。  
服务化并不是将服务线程化或者是进程化，亦或者是docker化，它可以简单的理解为：一种管理生命周期的方式。
#### 2. 服务化的基本结构
服务化的架构很简单，就是一个接口(Service)和一个抽象类(AbstractService)以及一个组合服务(AbstractService)。
（对任意服务进行组合就是组合服务）
### 3. 服务化基本操作
service 提供基本的操作接口  
比如init,start,stop  
AbstractService 实现了service接口，并向外提供了更多操作  
比如： serviceInit，serviceStart，serviceStop  

上面的方法跟service 接口的关系是？比如init跟serviceInit ，AbstractService 实现了init方法，并根据服务的状态选择性的执行AbstractService.serviceInit()

组合服务CompositeService 实际就是在CompositeService 内部维护一个List<Service> serviceList 集合，并继承AbstractService。也即是说组合服务本身也是一个服务，这个服务同时维护着多个服务。
### 4. 服务状态维护
每个服务除了对应操作还有对应的状态,状态列表如下所示：
```java
/** Constructed but not initialized */
NOTINITED(0, "NOTINITED"),
/** Initialized but not started or stopped */
INITED(1, "INITED"),
/** started and not stopped */
STARTED(2, "STARTED"),
/** stopped. No further state transitions are permitted */
STOPPED(3, "STOPPED");
```
如果已经调用了init方法，那状态就变成INITED，如果调用了start方法，服务的状态就会变成START。那服务的状态转换控制如何实现呢？ServiceStateModel就登场了  
其通过一个二维数组来确认不同状态之间是否可以转换
```Java
/**
 * Map of all valid state transitions
 * [current] [proposed1, proposed2, ...]
 */
private static final boolean[][] statemap =
  {
    //                uninited inited started stopped
    /* uninited  */    {false, true,  false,  true},
    /* inited    */    {false, true,  true,   true},
    /* started   */    {false, false, true,   true},
    /* stopped   */    {false, false, false,  true},
  };
```
### 5. 如何使用
下图展示了Nodemanager 基于服务化的基本结构   
NodeManager 是一个组合服务，其中还包括了NodeStatusUpdaterImpl，NodeStatusUpdaterImpl是一个单一服务
![](/images/yarn-service1.png)
NodeManager 启动过程如下：
先初始化NodeManager对象（因为nodeManager 继承了 AbstractService ，所以本身也是一个服务），接着调用service提供的两个方法
```java
this.init(conf);
this.start();
```
以init方法为例
init 实际调用的是AbstractService.init()，而AbstractService.init() 内部会调用serviceInit，因为AbstractService.serviceInit是一个抽象方法，实际执行的是NodeManager.serviceInit().在NodeManager.serviceInit方法中会执行addService方法接入其他server ，并最后调用super.serviceInit(conf);方法来触发通过addService方法添加的service执行serviceInit


**值得注意的是**
1. AbstractService 虽然提供了两个监听器集合 listeners和globalListeners。但globalListeners 到目前还没有用到。
2. CompositeService 通过addService方法和addIfService方法 向组合服务中添加服务，通过removeService方法移除服务。这些方法皆是线程安全的

yarn 服务化概览  
整个yarn 中有48个服务,nodeManger 有组合服务4个,可以看出service yarn中一个很重要的业务抽象
![](/images/yarn-service3.png)

![](/images/yarn-service4.png)
