# AsyncDispatcher（中央异步调度器 ）

中央异步调度器   
事件处理跟事件，服务关系密切，如果仅仅从事件处理角度来分析yarn 的事件处理模型可以概括为：
AsyncDispatcher 中央异步调度器将对应的事件分发给事件处理器（handler）或状态机处理，并触发新的事件，直到没有新的事件产生。
如下图所示

AsyncDispatcher 内部有两个临时缓存
BlockingQueue<Event> eventQueue 以及Map<Class<? extends Enum>, EventHandler> eventDispatchers（事件类型和handler处理关系）
AsyncDispatcher 本身是一个服务，启动服务之后会生成一个新的线程”AsyncDispatcher event handler“
“AsyncDispatcher event handler” 线程不断从eventQueue 队列拿出事件，通过 dispatch(event) 方法 将事件分发给eventDispatchers 存储的 handler（EventHandler）
处理逻辑为 （涉及泛型，参考java泛型）


 Class<? extends Enum> type = event.getType().getDeclaringClass();
 EventHandler handler = eventDispatchers.get(type);
 handler.handle(event);


AsyncDispatcher 对外提供 register(Class<? extends Enum> eventType,EventHandler handler)方法
该方法将事件和handler 写入缓存eventDispatchers。为了能够兼容一个事件对应多个handler的情况，创建了MultiListenerHandler 对象（内部缓存交由List<EventHandler<Event>> listofHandlers;）。
AsyncDispatcher 对外提供 getEventHandler方法
该方法方便中央异步调度器的使用者 使用EventHandler.handle(Event event) 将需要中央异步处理器处理的事件 写入 eventQueue 缓存。为了能够处理写入缓存的异常情况，创建了GenericEventHandler对象。
下面以NodeManager 为例，看看AsyncDispatcher如何使用
启动脚本bin/yarn
yarn 指定class
CLASS='org.apache.hadoop.yarn.server.nodemanager.NodeManager'，启动NodeManager。
NodeManager.main() 调用nodeManager.initAndStartNodeManager(conf, false);
进而调用 this.init(conf);
init 方法源于 NodeManager父类CompositeService的父类AbstractService，AbstractService内部调用了serviceInit()，而NodeManager 重写了serviceInit()
在NodeManager.serviceInit(Configuration conf) 方法中
初始化了中央异步处理器
this.dispatcher = new AsyncDispatcher();
并注册了两个事件和对应handler


dispatcher.register(ContainerManagerEventType.class, containerManager);
dispatcher.register(NodeManagerEventType.class, this);


ContainerManagerEventType 事件类型对应的处理（containerManager） 为：


  @Override
  public void handle(ContainerManagerEvent event) {
    switch (event.getType()) {
    case FINISH_APPS:
      CMgrCompletedAppsEvent appsFinishedEvent =
          (CMgrCompletedAppsEvent) event;
      for (ApplicationId appID : appsFinishedEvent.getAppsToCleanup()) {
        .......
        try {
          this.context.getNMStateStore().storeFinishedApplication(appID);
        } catch (IOException e) {
          LOG.error("Unable to update application state in store", e);
        }
        this.dispatcher.getEventHandler().handle(
            new ApplicationFinishEvent(appID,
                diagnostic));
      }
      break;
    case FINISH_CONTAINERS:
      CMgrCompletedContainersEvent containersFinishedEvent =
          (CMgrCompletedContainersEvent) event;
      for (ContainerId container : containersFinishedEvent
          .getContainersToCleanup()) {
          this.dispatcher.getEventHandler().handle(
              new ContainerKillEvent(container,
                  ContainerExitStatus.KILLED_BY_RESOURCEMANAGER,
                  "Container Killed by ResourceManager"));
      }
      break;
    default:
        throw new YarnRuntimeException(
            "Got an unknown ContainerManagerEvent type: " + event.getType());
    }
  }


NodeManagerEventType事件类型对应的处理（this）为
public void handle(NodeManagerEvent event) {
    switch (event.getType()) {
    case SHUTDOWN:
      shutDown();
      break;
    case RESYNC:
      resyncWithRM();
      break;
    default:
      LOG.warn("Invalid shutdown event " + event.getType() + ". Ignoring.");
    }
  }
上面说了中央异步处理模型，并解释了是如何注册事件和handler ，下面说说是如何生成该事件。
NodeManager 在 serviceInit()方法中会通过 createNodeStatusUpdater() 初始化一个NodeStatusUpdaterImpl 服务。该服务在启动过程中（执行serviceStart方法）会调用startStatusUpdater方法，该方法会新建一个线程“Node Status Updater”
”Node Status Updater“不断循环向resourceManager发送心跳，根据放回的结果，决定是否将相关事件发给中央异步处理器
发送方式如下：


dispatcher.getEventHandler().handle(
        new NodeManagerEvent(NodeManagerEventType.SHUTDOWN));
