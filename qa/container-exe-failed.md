---
description: 用户提交app的app之后，如果container 被提交到特定到nm 执行，则一定会异常（could not find or load main class org.apache.hadoop.mapred.YarnChild），进而导致app执行失败
---

# container执行异常
## 背景
用户磁盘挂了，恢复磁盘之后，重启系统，重启该节点所安装的服务正常。用户提交app，如果有container 被分配到恢复到节点，则会出现异常，异常信息如下：  
![](/images/exe1.png)

## 排查思路
1. 有异常日志，说明container 执行命令已经执行（nodeManager已经接收到请求，并准备好数据）
2. 确认 org.apache.hadoop.mapred.YarnChild 在 hadoop-mapreduce-client-app*.jar
3. 打印container 执行命令或classpath 确认是否有对应jar

## 定位步骤
### 关闭日志聚合确认执行脚本和执行资源
#### 关闭YARN 日志聚合
日志聚合是指一个app执行完之后，会将本地执行日志copy到hdfs ，并在HDFS 保存一段时间，同时清理container 在本地的执行资源（保存执行脚本，依赖jar ,配置文件，执行日志）  
 1. 修改配置关闭日志聚合   
默认配置路径 hadoop/conf/yarn-site.xml    
```
 <property>
     <name>yarn.log-aggregation-enable</name>
     <value>false</value>
   </property>
```

 2. 重启NM
```
[yarn]$ $HADOOP_YARN_HOME/sbin/yarn-daemons.sh --config $HADOOP_CONF_DIR stop/start nodemanager
```

#### 检查container 执行资源
 1. container 执行资源默认位置在   
/data/hadoop/yarn/local/usercache/【USER】/appcache/application_ID/container_ID/    
其中包括 执行脚本和jar 依赖 ，执行入口是 default_container_executor.sh  
 2. container 日志目录   
/data/hadoop/yarn/log/application_ID/container_ID 有对应 out 和 err 日志
 3. 有一部分执行资源会被缓存到本地用来复用  
比如 /data/hadoop/yarn/local/usercache/【USER】/filecache/【id】   
和  /data/hadoop/yarn/local/filecache/【id】/  

#### 手动检查执行执行资源
1. 获取CLASSPATH   
修改执行脚本launch_container.sh 打印 CLASSPATH   
![](/images/exe2.png)

2. 从 CLASSPATH 结果路径中查询是否有hadoop-mapreduce-client-app*.jar   
结果是没有发现对应文件

3. 如果 CLASSPATH 确认没有对应 jar ,那应该存放哪里呢？   
我们确认一下container 执行目录资源   
![](/images/exe3.png)  

从子目录详情看 最有可能在 mr-framework 目录（软连目录）  
查看正常情况下的目录情况  
![](/images/exe5.png)  

而实际情况，我们看到的是   
![](/images/exe4.png)  

也就是这里是一个不断软连的目录，没有真实的资源

#### 解决方案
通过上面的定位，我们已经确认了问题原因是本地缓存的可复用的资源为空。  
先说解决方案：删除本地缓存 /data/hadoop/yarn/local/filecache 目录,重跑app 问题解决。  
要深入理解就涉及Hadoop 分布式缓存概念。
即 /data/hadoop/yarn/local/filecache/【ID】/ 路径的数据从哪里来，什么时候触发。
