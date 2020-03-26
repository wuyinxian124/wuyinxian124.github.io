# RM HA 基础

结合代码和[http://hackershell.cn/?p=1174](http://hackershell.cn/?p=1174) 可以确定 RM 并没有在切换过程中处理fence，而是将隔离操作赋权给RMStateStore ，推荐的方式是：ZKRMStateStore

RM 是如何 用ZKRMStateStore来避免脑裂  
正常情况下，zk 上存储的RM 信息格式

```text
[zk: tbds-10-1-3-18:2181,tbds-10-1-3-30:2181,tbds-10-1-3-20:2181(CONNECTED) 10] ls /rmstore
[ZKRMStateRoot]
[zk: tbds-10-1-3-18:2181,tbds-10-1-3-30:2181,tbds-10-1-3-20:2181(CONNECTED) 11] ls /rmstore/ZKRMStateRoot
[RMAppRoot, AMRMTokenSecretManagerRoot, EpochNode, RMDTSecretManagerRoot, RMVersionNode]
```

通过/rmstore/ZKRMStateRoot 节点的权限，可知同时只能有一个用户能进行写操作

```text
[zk: tbds-10-1-3-18:2181,tbds-10-1-3-30:2181,tbds-10-1-3-20:2181(CONNECTED) 4] getAcl /rmstore/ZKRMStateRoot
'world,'anyone
: rwa
'digest,'tbds-10-1-3-26:vqMG85jYBFVbN5G2KRaOrqZVeqc=
: cd
```

获取zookeeper 节点权限 getAcl node zookeeper 权限信息说明 参考：[http://www.yoonper.com/post.php?id=47](http://www.yoonper.com/post.php?id=47)

因为强依赖zk,所以有必要了解一下 node 属性信息，如 dataVersion 等，参考：[https://blog.csdn.net/lihao21/article/details/51810395](https://blog.csdn.net/lihao21/article/details/51810395)

RM HA更多参考：https://docs.cloudera.com/documentation/enterprise/6/6.3/topics/cdh_hag_rm_ha_config.html#xd_583c10bfdbd326ba--43d5fd93-1410993f8c2--7f77
