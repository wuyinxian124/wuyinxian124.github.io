# TEZ 资源不释放问题分析

TEZ 资源不释放问题分析

## 现象

YARN UI 显示 APP 还在运行，但是任务实例已经显示成功。

## 分析
### 业务进程
#### 实例进程
首先怀疑是我们后台进程异常，导致runner进程挂了，但是beeline进程还在  
通过后台进程关键字确认：
任务实例runner 结束，对应beeline 进程不存在

![](https://qqadapt.qpic.cn/txdocpic/0/4e9962cac0c00e788262f744786439f8/0)

#### 任务实例日志

确认任务实例日志中没有明显异常信息，beeline 进程正常结束，beeline 标准输出有success 关键字

![](https://qqadapt.qpic.cn/txdocpic/0/4f9a553bb060892b0d289f9b64f011dd/0)

### YARN

#### 查看不结束container

YARN UI 查询 container 001 (AM container)

#### 确定container 执行节点

通过RM 日志找到001 号container 执行节点

#### 查看container 执行状况

在datanode 通过APPid 过滤三个进程，其中孙子进程（267609）应该就是不结束原因

![](https://qqadapt.qpic.cn/txdocpic/0/cde0cdb68bf5ec94faaf6cc2bd88f34f/0)

#### 查看孙子进程日志

查看进程信息中log.dir 指定目录

![](https://qqadapt.qpic.cn/txdocpic/0/b38e0a3b34307594eb5f1593fa35c267/0)

#### 日志中异常信息

如下：

![](https://qqadapt.qpic.cn/txdocpic/0/321a4fb5464c075fa035bcc98ae6232e/0)

![](https://qqadapt.qpic.cn/txdocpic/0/56b53f7d11f0aecb453513c9deabd818/0)

## 解决

### 临时解决方案

 确认app 其他container 正常结束的情况下，通过异常日志确认业务逻辑已经执行成功，但是状态同步异常，可以直接终止死循环的session 进程。如执行 kill -9 267609

 终止session进程之后，RM 会拉起另一个（02号）AM 继续执行。状态同步一次之后会忽略异常，使得02 AM 结束退出。

### 最终解决方案

修改TEZ 在hive server 节点 /etc/tez/conf/tez-site.xml 文件如下内容

![](https://qqadapt.qpic.cn/txdocpic/0/6c0f57790b255a6c7e36978fd2e9bbb8/0)

对应value 替换为 org.apache.tez.dag.history.logging.impl.SimpleHistoryLoggingService
