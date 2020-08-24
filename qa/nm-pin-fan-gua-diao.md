---
description: NM 频繁挂掉问题分析和解决
---

# NM频繁挂掉

NM 频繁挂掉问题分析和解决

## 一.现象

1. NM 频繁挂掉，而且出现NM 启动异常 重启异常日志如下：

   ```text
   Caught java.lang.OutOfMemoryError: unable to create new native thread. One possible reason is that ulimit setting of 'max user processes' is too low. If so, do 'ulimit -u &lt;largerNum&gt;' and try again.
   ```

   ![](https://qqadapt.qpic.cn/txdocpic/0/3dd06b5a9ee219602cf3709c259a12cd/0?w=2560&h=2560)

2. 节点内存足够（而且清理部分进程，使得空闲内存变大，问题依旧） ![](https://qqadapt.qpic.cn/txdocpic/0/4fb6945e230ff6bf54ea0877ee0caef6/0?w=2560&h=2560) ![](https://qqadapt.qpic.cn/txdocpic/0/2da50e6d8b9e739b1126ce11e1be09c7/0?w=2560&h=2560)

## 二.分析

如果不是内存不足引起的问题，而且是部分节点才出现的问题，怀疑是文件句柄被占用。如果是文件句柄的问题，解决思路有两个： 1. 节点设置的句柄数太小 2. 句柄被异常消耗

## 三.句柄情况

查看节点文件句柄设置

###  1. /proc/sys/vm/max\_map\_count

![](https://qqadapt.qpic.cn/txdocpic/0/f85518eee0cbd85d329452d89ecc18cf/0?w=1972&h=120)

### 2./etc/sysctl.conf

![](https://qqadapt.qpic.cn/txdocpic/0/780a46a4fccaa461a7e8e3bf403ba168/0?w=1992&h=1626)

### 3./proc/sys/kernel/pid\_max

![](https://qqadapt.qpic.cn/txdocpic/0/ac8414916d03df6fe808dc94108f84f7/0?w=1292&h=108)

### 4./etc/security/limits.conf

![](https://qqadapt.qpic.cn/txdocpic/0/ca326685db7566f87ff799760eeca7a7/0?w=720&h=704)

### 5./etc/security/limits.d/yarn.conf

![](https://qqadapt.qpic.cn/txdocpic/0/0d0e80fa4a13a4117756fd2af7f24aee/0?w=1650&h=970)

从相关设置看，文件句柄符合系统要求

## 四.异常情况

在分析NM 进程的时候，我们注意到在NM 已经终止的节点中有几十个container 执行进程。 ![](https://qqadapt.qpic.cn/txdocpic/0/f76dd51f0563cae428f628fdc5cca477/0?w=2608&h=1560)

如上图所示，有部分进程在NM 终止后，持续了相当长的时间（有超过一个月）。 而且还有一个特征是，遗留的container 都是非001 号container

## 五.问题分析

从container的输出日志看，container 一直在重连，应该是重连 AM 我们进一步分析 其对应的 001号 container 对应的输出日志，发现日志中有大量 终止 container 异常日志 ![](https://qqadapt.qpic.cn/txdocpic/0/85021b12816f2567c079f40d834abb86/0?w=2628&h=1498)

基于此 我们分析 在nodemanager 挂掉后（比如跟RM 连接超时），因为 001 container kill 其他container 也是通过nodemanager 来操作的，而且kill 不管是否成功都，都认为 container 被kill 了，然后001 退出，001 拉起的其他container 自然就附着到1 号进程了。从而遗留了大量container 孤儿进程消耗了文件句柄，从而导致上述问题。

而其他没有问题的节点，基本不出现这个问题是因为这两个节点已经分配了长时间运行的spark 任务，所以新的tez 任务不会分配到这两个节点，所以不会出现container 孤儿 ，因此是稳定的。

## 六.解决办法

后续出现此类问题 先在 nodemanager 执行命令

```text
ps -eo cmd --sort=start\_time\|grep -v grep \|grep container\_executor \|sed 's/^.\*\/container\_e..\_//g'\|sed 's/\_0.\_000...\/default.\*$//g'\|uniq\|sed 's/^/yarn application -status application\_/'
```

对扫描出遗留的container 孤儿对应的 APP，然后再通过下面命令，确认 APP 已经执行完 yarn application -status application

![](https://qqadapt.qpic.cn/txdocpic/0/7a6c9f43f96861cf6d6f5cfb0d62fcf8/0?w=2548&h=1414)

如果state 为 finished 则可以用

```text
ps -ef\|grep -v grep \|grep 1574737015641\_15941 \|awk '{print $2}'\|xargs  kill -15
```

终止。

## 七、写在最后

综上所述，该问题导致的根本原因是有遗留的container 孤儿进程，根本的解决办法是有三

1. AM 需要确保 其拉起的container 能被真正终止
2. container 重连AM 应该是有上限，达到重连上限应该自动终止
3. NM 应该对其节点上的container 的状态负责

## 八、参考

YARN如何终止container ：[https://www.jianshu.com/p/47e2fd3ad636](https://www.jianshu.com/p/47e2fd3ad636)

查看子进程和负载：[https://blog.csdn.net/lvqingyao520/article/details/82144413](https://blog.csdn.net/lvqingyao520/article/details/82144413)

linux ps 命令：[https://jaminzhang.github.io/linux/using-ps-to-view-process-started-and-elapsed-time-in-linux/](https://jaminzhang.github.io/linux/using-ps-to-view-process-started-and-elapsed-time-in-linux/)

jcmd 命令：[https://www.jianshu.com/p/388e35d8a09b](https://www.jianshu.com/p/388e35d8a09b)

/proc/pid 有什么：[https://blog.csdn.net/enweitech/article/details/53391567](https://blog.csdn.net/enweitech/article/details/53391567)

进程句柄：[https://www.ibm.com/developerworks/cn/linux/l-cn-handle/](https://www.ibm.com/developerworks/cn/linux/l-cn-handle/)

## 九. 补充

### 1.查看所有用户创建的进程数,使用命令：

```text
ps h -Led -o user \| sort \| uniq -c \| sort -n
```

![](https://qqadapt.qpic.cn/txdocpic/0/a884e63bc8ffc2fc1dcc7fd39654b18e/0?w=524&h=199)

### 2.查看hfds用户创建的进程数，使用命令:

```text
ps -o nlwp,pid,lwp,args -u hdfs \| sort -n
```

![](https://qqadapt.qpic.cn/txdocpic/0/cad8dde6d3b4a22b072473718662f36f/0?w=483&h=92)

### 3.查看进程启动的精确时间和启动后所流逝的时间

![](https://qqadapt.qpic.cn/txdocpic/0/d5ddfe0a4395bd9fef1d5893368c5bca/0?w=1588&h=210)

### 4.grep 报错Binary file \(standard input\) matches cat 文件名 \| grep -a 特定条件

```text
比如： cat xxxx \| grep -a 12345
```

### 5.进程按启动时间排序

```text
ps aux --sort=start\_time\|grep Full\|grep -v grep
```

