---
description: ResourceManager High Available
---

# RM HA

ResourceManager 高可用是基于zookeeper 实现

## 1. RM HA结构

RM 高可用包括哪些部分，相互之间的联系
ActiveStandbyElector,EmbeddedElectorService,ZkFailoverController

## 2. RM HA 切换逻辑

RM 主备切换的过程和逻辑
正常切换过程
Active状态RM 因为GC 或 网络问题 跟zk失联，ARM（ActiveResourceManager）在zk 上的临时节点因为连接断开，而自动被删除。  
因为其他RM 也监听了这个node ，所以其他RM 会抢leader ，从而选举出新的master。  
假设之前Active的RM 为A，后面的选举产生的RM 为B。  
RM B会调用 fence 接口，尝试将RM A  隔离。
隔离RM A的过程为 先通过 A 的RPC 接口，尝试将A变成standby ，这个过程会同时将A在ZK 上创建持久化node 删除。这个过程俗称 优雅停止。
如果rpc 接口异常，则会调用 SSH 或本地 SHELL 尝试将A 终止。这个过程俗称 强制终止（强制终止的方式可以配置多个）。
如果强制终止也失败，那RM B 会断开跟zk 的连接，然后重新参与选举。
如果强制终止或优雅终止成功，那么RM B 会将自己变成active，然后从ZKStatStore 中加载全量的缓存信息，接着将自己的连接信息写到 zk 的持久node 上。

## 3. RM HA 使用过程的注意事项  
1. RMStateStore 选择很重要  
hadoop 提供了四种stateStore:FileSystemRMStateStore,LeveldbRMStateStore,MemoryRMStateStore,NullRMStateStore  
后两者不能持久化 app state ，所以在主备切换过程中会丢失app 状态（YARN会kill 然后重新拉起）  
前两者能够给持久化 APP state，所以主备切换过程不会丢失APP 状态，而且不会kill 再 拉起 APP。

2. FenceMethod 选择很重要 （NameNode提供，RM fence为空-成功）  
hadoop 提供了两种fence 实现：ShellCommandFencer，SshFenceByTcpPort  
SshFenceByTcpPort 需要远程到 RM B节点，执行 ```fuser -v -k -n tcp port``` 命令,所以需要无秘登录的key  
ShellCommandFencer 是在本地执行一个shell 脚本，目的是干掉RM B ，默认情况下 shell 都是成功的。  

3. Active RM 被突然关机，或者被拔掉网线 HA如何切换？  
ZK 发现RM A连接中断，会删除临时lock ，则其他RM 会竞争leader。RM B 选举成功会check 持久化node ，拿到RM A 的连接信息。  
首先尝试通过 RPC 终止 RM A ,因为网络中断 失败。  
然后尝试fence ，因为fence 方法为空，所以会成功。   
所以在切断网线或者突然关机的情况下，RM B 会被唤醒为active 。  
如果这时候 RM A 插上网线或者 恢复（FULL GC ,CPU 爆掉 卡死）分两种情况
 1. 如果跟zk 的连接部分首先被唤醒，则会发现自己不是leader 则主动切为standby.  
 主动切standby 的过程，会重启RMActiveServices ,重新刷 acl,队列等信息，但是没有主动删除 zk 上的持久化节点。

 2. 如果zk 连接部分还卡死了，那就意味着有一个时间窗口是有两个 RM的。


4. 会不会出现脑裂？什么情况下回出现脑裂？

5. 会不会出现RM 无主？什么情况下会出现无主？

6. RM 脑裂会带来什么问题，有没有办法避免？

7. RM 无主会带来什么问题，有没有办法避免？

8. RM HA 存在什么的问题？
  1. 使用ZkRMStateStore 情况下，因为app 状态更新都需要在更新之前创建一个Lock Node所以RM 在状态更新是串行的，并意味着存在一个全局锁。


## 4. RM HA 和 NameNode HA 差异
