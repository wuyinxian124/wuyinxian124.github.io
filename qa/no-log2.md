---
description: app 日志查看异常
---

# yarn UI看不了app 日志2

## 现象

查看日志，出现如下错误提示

![](../.gitbook/assets/image%20%284%29.png)

同时发现正在运行app 日志是可以查看的

## 猜测

正在运行的app 日志可见，也就是本地日志是有的，通过yarn.nodemanager.log-dirs 配置目录我们也确认了正在运行的app 日志存在 所以我们猜测就是日志聚合的时候出现问题。

## 异常

通过container ID 到rm 主节点查询container 运行到NM，再通过container ID查询相关日志，得到如下异常信息

![](../.gitbook/assets/image%20%283%29.png)

通过对异常进一步分析，发现这个异常应该不影响日志聚合。同时我们也在hdfs 查到了聚合之后对日志

![](../.gitbook/assets/image%20%285%29.png)

## 原因分析

