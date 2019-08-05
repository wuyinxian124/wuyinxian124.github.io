# RM下发App

本文描述的是 ResourceManager 如何下发一个 执行app的命令给Node Manager

## 1. 基本对象
RmApp： app 在 ResourceManager 的抽象。  
RmAppAttempt ： app 在 resourceManager 不同参数运行次数的抽象  
container：app 资源抽象  
ApplicationMaster ： app 执行大脑

本文描述的下发App 详细描述的是 resourceManager 向 Node Manager 发送一个启动 ApplicationMaster 的命令。

## 2. ResourceManager 下发
ResourceManager 下发applicationMaster的入口在  
AMLauncher.launch\(\), AMLaunch.launch先在connect\(\)中拿到对应node的rpc客户端containerMgrProxy，然后构造request，最后调用rpc函数startContainers\(\)并返回response。 具体逻辑是：

```java
private void launch() throws IOException, YarnException {
  connect();
  ContainerId masterContainerID = masterContainer.getId();
  ApplicationSubmissionContext applicationContext =
      application.getSubmissionContext();
  LOG.info("Setting up container " + masterContainer
      + " for AM " + application.getAppAttemptId());
  ContainerLaunchContext launchContext =
      createAMContainerLaunchContext(applicationContext, masterContainerID);

  StartContainerRequest scRequest =
      StartContainerRequest.newInstance(launchContext,
        masterContainer.getContainerToken());
  List<StartContainerRequest> list = new ArrayList<StartContainerRequest>();
  list.add(scRequest);
  StartContainersRequest allRequests =
      StartContainersRequest.newInstance(list);

  StartContainersResponse response =
      containerMgrProxy.startContainers(allRequests);
  if (response.getFailedRequests() != null
      && response.getFailedRequests().containsKey(masterContainerID)) {
    Throwable t =
        response.getFailedRequests().get(masterContainerID).deSerialize();
    parseAndThrowException(t);
  } else {
    LOG.info("Done launching container " + masterContainer + " for AM "
        + application.getAppAttemptId());
  }
}
```

通过rpc 接口将调用发送给 resourceManager

```java
public StartContainersResponse
    startContainers(StartContainersRequest requests) throws YarnException,
        IOException {
  StartContainersRequestProto requestProto =
      ((StartContainersRequestPBImpl) requests).getProto();
  try {
    return new StartContainersResponsePBImpl(proxy.startContainers(null,
      requestProto));
  } catch (ServiceException e) {
    RPCUtil.unwrapAndThrowException(e);
    return null;
  }
}
```
