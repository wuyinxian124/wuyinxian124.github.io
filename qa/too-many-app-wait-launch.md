# YARN 资源充足，但app等待调度排长队

通常app 排队常见原因有三个  
1. 队列资源满了  
 1. 比较内存和core   
比较 Max Resources 的 vcore 和 memory 与 Used Resources的 vcore 和 memory 。如果used Resources的大，则是资源队列满了  
 2. 比较app 运行个数  
 比较 Max Running Applications 和 Num Active Applications。如果相等，则表示是最大运行个数限制了
2. 集群最大运行个数限制  
在配置文件conf/fair-scheduler.xml 有 queueMaxAppsDefault 了集群最大可运行app 个数，如果Apps Running（yarn UI -> Cluster Metrics） 等于设置的最大个数，则表示是集群最大可运行个数的限制
3. 资源预留  
 看yarn UI -> Cluster Metrics 中的 Reserved，包括core ,memory 。只要有预留，就说明有有一个或者多个节点的资源被预留了，而且一个节点只能预留给一个container，如果有一个节点被预留了，那么这个节点其他资源是不能再被分配的



：
