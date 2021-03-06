# Table of contents

* [介绍](README.md)
* [YARN Service](yarn-service.md)
* [事件处理](yarn-evnet/README.md)
  * [AsyncDispatcher](yarn-evnet/asyncdispatcher.md)
  * [stateMachine](yarn-evnet/statemachine.md)
* [YARN RPC](rpc/README.md)
  * [RPC 设计](rpc/rpc_struct.md)
  * [RPC 实现](rpc/rpc_core.md)
* [交互协议](jiao-hu-xie-yi.md)
* [提交app的过程](submit_app/README.md)
  * [提交APP概要](submit_app/ti-jiao-app-gai-yao.md)
  * [客户端提交app](submit_app/client2rm.md)
  * [RM接收请求](submit_app/rm_app.md)
  * [RM分配App](submit_app/rmallocateapp.md)
  * [RM下发App](submit_app/rmissueapp.md)
  * [NM拉起APP](submit_app/nm_app.md)
* [YARN 高可用](yarn-ha/README.md)
  * [RM HA 基础](yarn-ha/rm-ha-ji-chu.md)
  * [RM HA](yarn-ha/rm-ha.md)
* [YARN调度策略](yarn-scheduler/README.md)
  * [Untitled](yarn-scheduler/untitled.md)
  * [YARN-队列配置](yarn-scheduler/yarn-queue.md)
* [问题分析总结](qa/README.md)
  * [集群繁忙时偶发性空指针导致app执行失败](qa/app-failed-rm-exit.md)
  * [YARN 资源充足，但app等待调度排长队](qa/too-many-app-wait-launch.md)
  * [YARN UI看不了app 日志](qa/no_log.md)
  * [YARN UI看不了app 日志2](qa/no-log2.md)
  * [TEZ 资源不释放问题分析](qa/resource-no-relase.md)
  * [NM频繁挂掉](qa/nm-pin-fan-gua-diao.md)
  * [container执行异常](qa/container-exe-failed.md)
  * [YARN RM主备切换异常](qa/rm-fail-over-failed.md)

