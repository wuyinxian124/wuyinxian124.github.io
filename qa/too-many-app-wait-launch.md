# YARN 资源充足，但app等待调度排长队

通常app 排队常见原因有三个  
1. 队列资源满了  
1.1. 比较内存和core  
比较 Max Resources 的 vcore 和 memory 与 Used Resources的 vcore 和 memory 。如果used Resources的大，则是资源队列满了  
1.2. 比较app 运行个数  
比较 Max Running Applications 和 Num Active Applications。如果相等，则表示是最大运行个数限制了 2. 集群最大运行个数限制  
在配置文件conf/fair-scheduler.xml 有 queueMaxAppsDefault 了集群最大可运行app 个数，如果Apps Running（yarn UI -&gt; Cluster Metrics） 等于设置的最大个数，则表示是集群最大可运行个数的限制 3. 资源1.3 资源被预留  
看yarn UI -&gt; Cluster Metrics 中的 Reserved，包括core ,memory 。只要有预留，就说明有有一个或者多个节点的资源被预留了，而且一个节点只能预留给一个container，如果有一个节点被预留了，那么这个节点其他资源是不能再被分配的

2. 待运行的app资源需求超高

如果有一个accept的app 资源要求很高（超过队列设置的最大内存和最大core）也会导致后面提交的app都会被阻塞（pending）

3. RM 处理慢

RM 中央异步处理器处理速度比较慢（可能存在，但是没有具体遇到过）



：

