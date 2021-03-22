---
description: app提交上来之后，运行过程出现偶发性的空指针异常，导致连续执行两次之后，app 异常结束
---

# 集群繁忙时偶发性空指针导致app执行失败

放在最前面：该问题是社区已知的缺陷，更多参考：[https://issues.apache.org/jira/browse/YARN-3260](https://issues.apache.org/jira/browse/YARN-3260)。本文主要介绍问题定位思路和分析过程。

## 1. 问题描述

app执行日志中异常堆栈如下：

![](../.gitbook/assets/image%20%287%29.png)

从字面意思理解就是AM启动之后向RM注册异常，然后失败了。

通过进一步的了解，我们知道yarn 集群最近做了扩容，app 提交规模也上来了。而且有部分app成功，有部分失败。失败app原因都是如上截图所示，没有明显的node 关联性。

## 2. 问题分析

### 2.1 确认异常位置

从异常堆栈，我们看到是AM向RM注册的过程中，抛出空指针异常导致app异常退出。也就是说不是am 本身的代码逻辑出现空指针，而是RPC调用返回了空指针。进一步说就是RM在处理过程中出现空指针。

我们通过搜索RM日志，并没有相关时间范围内发现有异常信息。

现在我们走到了一个死胡同，我们知道是RM在处理AM注册请求时出现空指针异常，但是在RM日志中找不到异常信息，而且AM异常堆栈中没有打印RM空指针出现的行号。

### 2.2 RM 日志分析

进一步RM日志，我们观察到一个异常信息

```text
attempt.RMAppAttemptImpl (RMAppAttemptImpl.java:rememberTargetTransitionsAndStoreState(1165)) - Updating application attempt appattempt_1616059928557_55186_000002 with final state: FAILED, and exit status: 1
```

这个异常是说AM执行异常了，退出码为1（其实也容易理解，AM向RM注册异常，尝试几次之后，告诉RM我执行不下去了，我要异常退出）。

所以回到RM日志，我们重新梳理了RM处理AM注册部分的逻辑。

重点看如下日志（通过appid 关键字过滤）

```text
2021-03-20 19:13:52,261 INFO  attempt.RMAppAttemptImpl (RMAppAttemptImpl.java:handle(796)) - appattempt_1616059928557_55186_000001 State change from ALLOCATED_SAVING to ALLOCATED
2021-03-20 19:14:04,020 INFO  amlauncher.AMLauncher (AMLauncher.java:run(249)) - Launching masterappattempt_1616059928557_55186_000001
2021-03-20 19:14:04,020 INFO  amlauncher.AMLauncher (AMLauncher.java:launch(105)) - Setting up container Container: [ContainerId: container_e10_1616059928557_55186_01_000001, NodeId: tbds-172-29-15-181:45454, NodeHttpAddress: tbds-172-29-15-181:8042, Resource: <memory:4096, vCores:1>, Priority: 0, Token: Token { kind: ContainerToken, service: 172.29.15.181:45454 }, ] for AM appattempt_1616059928557_55186_000001
2021-03-20 19:14:04,020 INFO  amlauncher.AMLauncher (AMLauncher.java:createAMContainerLaunchContext(187)) - Command to launch container container_e10_1616059928557_55186_01_000001 : $JAVA_HOME/bin/java -Djava.io.tmpdir=$PWD/tmp -Dlog4j.configuration=container-log4j.properties -Dyarn.app.container.log.dir=<LOG_DIR> -Dyarn.app.container.log.filesize=0 -Dhadoop.root.logger=INFO,CLA -Dhadoop.root.logfile=syslog -Dhdp.version=2.2.0.0-2041 -Xmx3072m org.apache.hadoop.mapreduce.v2.app.MRAppMaster 1><LOG_DIR>/stdout 2><LOG_DIR>/stderr 
2021-03-20 19:14:04,020 INFO  security.AMRMTokenSecretManager (AMRMTokenSecretManager.java:createAndGetAMRMToken(195)) - Create AMRMToken for ApplicationAttempt: appattempt_1616059928557_55186_000001
2021-03-20 19:14:04,020 INFO  security.AMRMTokenSecretManager (AMRMTokenSecretManager.java:createPassword(307)) - Creating password for appattempt_1616059928557_55186_000001
2021-03-20 19:14:04,023 INFO  amlauncher.AMLauncher (AMLauncher.java:launch(126)) - Done launching container Container: [ContainerId: container_e10_1616059928557_55186_01_000001, NodeId: tbds-172-29-15-181:45454, NodeHttpAddress: tbds-172-29-15-181:8042, Resource: <memory:4096, vCores:1>, Priority: 0, Token: Token { kind: ContainerToken, service: 172.29.15.181:45454 }, ] for AM appattempt_1616059928557_55186_000001
2021-03-20 19:14:09,528 INFO  ipc.Server (Server.java:saslProcess(1316)) - Auth successful for appattempt_1616059928557_55186_000001 (auth:SIMPLE)
2021-03-20 19:14:09,530 INFO  resourcemanager.ApplicationMasterService (ApplicationMasterService.java:registerApplicationMaster(274)) - AM registration appattempt_1616059928557_55186_000001
2021-03-20 19:14:09,530 INFO  resourcemanager.RMAuditLogger (RMAuditLogger.java:logSuccess(127)) - USER=tdwhive	IP=172.29.15.181	OPERATION=Register App Master	TARGET=ApplicationMasterService	RESULT=SUCCESS	APPID=application_1616059928557_55186	APPATTEMPTID=appattempt_1616059928557_55186_000001
2021-03-20 19:14:16,977 INFO  attempt.RMAppAttemptImpl (RMAppAttemptImpl.java:handle(796)) - appattempt_1616059928557_55186_000001 State change from ALLOCATED to LAUNCHED
2021-03-20 19:14:22,670 INFO  attempt.RMAppAttemptImpl (RMAppAttemptImpl.java:handle(796)) - appattempt_1616059928557_55186_000001 State change from LAUNCHED to RUNNING
2021-03-20 19:14:22,671 INFO  rmcontainer.RMContainerImpl (RMContainerImpl.java:handle(410)) - container_e10_1616059928557_55186_01_000001 Container Transitioned from ACQUIRED to RUNNING
2021-03-20 19:14:31,376 INFO  rmapp.RMAppImpl (RMAppImpl.java:handle(715)) - application_1616059928557_55186 State change from ACCEPTED to RUNNING
2021-03-20 19:14:31,659 INFO  rmcontainer.RMContainerImpl (RMContainerImpl.java:handle(410)) - container_e10_1616059928557_55186_01_000001 Container Transitioned from RUNNING to COMPLETED
```

结合下图（RMAppAttempt 状态流转图，[完整图](https://www.processon.com/view/link/5d10d751e4b024123de6bd47)），我们来分析在AM启动之后，向RM注册过程，会涉及到RMAppAttempt 对象从Allocated到running状态到变化。

![](../.gitbook/assets/image%20%288%29.png)

我们19:14:04,023 看到日志关键字： Done launching 。说明AM已经被发送到NodeManager 。然后接下来第二行日志 （也就是19:14:09）我们看到日志关键字：AM registration 。这就是说在 14分04秒RM下发完成，14分09秒AM就来注册了。接下来神奇的事情发生了，也就是再下面第二行（也就是19:14:16）我们看到RMAppAttempt 状态变化了（change from ALLOCATED to LAUNCHED）。

我们比较一个成功的app 对应的RM处理日志，则没有上面描述的问题

```text
2021-03-22 10:51:17,048 INFO  amlauncher.AMLauncher (AMLauncher.java:run(249)) - Launching masterappattempt_1616340089879_4264_000001
2021-03-22 10:51:17,048 INFO  amlauncher.AMLauncher (AMLauncher.java:launch(105)) - Setting up container Container: [ContainerId: container_e12_1616340089879_4264_01_000001, NodeId: tbds-172-29-0-125:45454, NodeHttpAddress: tbds-172-29-0-125:8042, Resource: <memory:4096, vCores:1>, Priority: 0, Token: Token { kind: ContainerToken, service: 172.29.0.125:45454 }, ] for AM appattempt_1616340089879_4264_000001
2021-03-22 10:51:17,048 INFO  amlauncher.AMLauncher (AMLauncher.java:createAMContainerLaunchContext(187)) - Command to launch container container_e12_1616340089879_4264_01_000001 : $JAVA_HOME/bin/java -Djava.io.tmpdir=$PWD/tmp -Dlog4j.configuration=container-log4j.properties -Dyarn.app.container.log.dir=<LOG_DIR> -Dyarn.app.container.log.filesize=0 -Dhadoop.root.logger=INFO,CLA -Dhadoop.root.logfile=syslog -Dhdp.version=2.2.0.0-2041 -Xmx3072m org.apache.hadoop.mapreduce.v2.app.MRAppMaster 1><LOG_DIR>/stdout 2><LOG_DIR>/stderr 
2021-03-22 10:51:17,048 INFO  security.AMRMTokenSecretManager (AMRMTokenSecretManager.java:createAndGetAMRMToken(195)) - Create AMRMToken for ApplicationAttempt: appattempt_1616340089879_4264_000001
2021-03-22 10:51:17,048 INFO  security.AMRMTokenSecretManager (AMRMTokenSecretManager.java:createPassword(307)) - Creating password for appattempt_1616340089879_4264_000001
2021-03-22 10:51:17,052 INFO  amlauncher.AMLauncher (AMLauncher.java:launch(126)) - Done launching container Container: [ContainerId: container_e12_1616340089879_4264_01_000001, NodeId: tbds-172-29-0-125:45454, NodeHttpAddress: tbds-172-29-0-125:8042, Resource: <memory:4096, vCores:1>, Priority: 0, Token: Token { kind: ContainerToken, service: 172.29.0.125:45454 }, ] for AM appattempt_1616340089879_4264_000001
2021-03-22 10:51:17,052 INFO  attempt.RMAppAttemptImpl (RMAppAttemptImpl.java:handle(796)) - appattempt_1616340089879_4264_000001 State change from ALLOCATED to LAUNCHED
2021-03-22 10:51:17,677 INFO  rmcontainer.RMContainerImpl (RMContainerImpl.java:handle(410)) - container_e12_1616340089879_4264_01_000001 Container Transitioned from ACQUIRED to RUNNING
2021-03-22 10:51:22,251 INFO  ipc.Server (Server.java:saslProcess(1316)) - Auth successful for appattempt_1616340089879_4264_000001 (auth:SIMPLE)
2021-03-22 10:51:22,253 INFO  resourcemanager.ApplicationMasterService (ApplicationMasterService.java:registerApplicationMaster(274)) - AM registration appattempt_1616340089879_4264_000001
2021-03-22 10:51:22,253 INFO  resourcemanager.RMAuditLogger (RMAuditLogger.java:logSuccess(127)) - USER=tdwhive	IP=172.29.0.125	OPERATION=Register App Master	TARGET=ApplicationMasterService	RESULT=SUCCESS	APPID=application_1616340089879_4264	APPATTEMPTID=appattempt_1616340089879_4264_000001
2021-03-22 10:51:22,253 INFO  attempt.RMAppAttemptImpl (RMAppAttemptImpl.java:handle(796)) - appattempt_1616340089879_4264_000001 State change from LAUNCHED to RUNNING
2021-03-22 10:51:22,253 INFO  rmapp.RMAppImpl (RMAppImpl.java:handle(715)) - application_1616340089879_4264 State change from ACCEPTED to RUNNING
```

这说明啥呢，说明RMAppAttempt对象还没有变launched状态，AM的注册就来了。这会导致什么问题？

问题就是[yarn-3260](https://issues.apache.org/jira/browse/YARN-3260)提到的，简要而言就是有如下图两个异步过程，一个过程\(左边\)是向缓存中添加token ,一个过程（右边）是从缓存中获取token。如果获取的过程比添加的过程要早执行，则会拿不到token。最麻烦的是获取的过程在获取返回之后，会执行编码（getEncoded）操作

```text
        LOG.info("Setting client token master key");
        response.setClientToAMTokenMasterKey(java.nio.ByteBuffer.wrap(rmContext
            .getClientToAMTokenSecretManager()
            .getMasterKey(applicationAttemptId).getEncoded())); 
```

如此就会出现NPE（NullPointerException）

![](../.gitbook/assets/image%20%286%29.png)

## 3.解决办法

jira 提到的解决办法就是不要在allocated到launched过程去保存token ,而是把保存token 提前，放到ALLOCATED\_SAVING 到allocated 到过程去，这样保存token 之后，才去拉起的AM，然后才是AM注册。

## 4.补充

这个问题本身比较简单，但是解决的门槛比较高，因为你得了解yarn 处理App的细节流程。最开始我们直接google ，并没有找到有用的信息（都指向yarn-site,yarn-core 的配置），直到后面我们通过在搜索前面加上 jira hadoop 关键字才看到有用信息。所以在直接google 没有什么用的时候，一定要去jira 上面看看

```text
jira hadoop  org.apache.hadoop.mapreduce.v2.app.rm.RMCommunicator: Exception while registering java.lang.NullPointerException: java.lang.NullPointerException at org.apache.hadoop.yarn.server.resourcemanager.ApplicationMasterService.registerApplicationMaster(ApplicationMasterService.java:296)
```

为什么会慢呢？无非是两个原因：1.中央异步处理器缓存队列处理比较慢，或者队列事件比较多。二. AMLaunchedTransition 处理过程慢。具体原因还要具体分析

