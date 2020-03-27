---
description: YARN 队列配置说明
---

# YARN-队列配置

## 1.队列最大可运行个数
配置来源
```
Integer maxApps = queueMaxApps.get(queue);
return (maxApps == null) ? queueMaxAppsDefault : maxApps;
```
queueMaxAppsDefault 来自 配置 conf/fair-scheduler.xml 对应的 queueMaxAppsDefault 属性   
而 queueMaxApps.get(queue) 来自 队列设置中的 maxRunningApps  

yarn 在处理最大可运行app 个数的判断也非常有意思  
通过if else 来做两个判断，第一个判断失败，表示用户最大可运行次数可以过。然后判断队列最大可运行次数  
```java
if (exceedUserMaxApps(attempt.getUser())) {
  attempt.updateAMContainerDiagnostics(AMState.INACTIVATED,
      "The user \"" + attempt.getUser() + "\" has reached the maximum limit"
          + " of runnable applications.");
  ret = false;
} else if (exceedQueueMaxRunningApps(queue)) {
  attempt.updateAMContainerDiagnostics(AMState.INACTIVATED,
      "The queue \"" + queue.getName() + "\" has reached the maximum limit"
          + " of runnable applications.");
  ret = false;
}
```
在队列判断的过程中 会从子队列向上轮询到跟队列  
```java
public boolean exceedQueueMaxRunningApps(FSQueue queue) {
  // Check queue and all parent queues
  while (queue != null) {
    if (queue.getNumRunnableApps() >= queue.getMaxRunningApps()) {
      return true;
    }
    queue = queue.getParent();
  }

  return false;
}
```
所以在一个队列上提交app ，他会检查每一个父队列正在运行的app 是否超过 最大限制。 如果没有设置，则使用全局参数。当然queueMaxAppsDefault 也变成了全局限制。
