---
description: 状态机
---

# stateMachine
为了让复杂逻辑更容易维护，代码更容易清晰，明了。YARN 使用了事件触发。事件触发有两个核心概念：中央异步调度器和状态机。   
前文介绍了[中央异步处理器](./AsyncDispatcher.md)，本文将介绍YARN 另一核心概念-状态机。

## 1. 简介
状态机是为了处理单一对象的输入，输出，状态转换逻辑的抽象。在YARN中状态机都跟对象强绑定，代码实现表现为每个对象中都有一个状态机属性。  

## 2. 功能
状态机在功能上能够抽象为：某个对应在某个状态下，因为某个输出，触发某个某个操作，进而变成另一个状态。
## 3. 设计和实现
### 3.1 概要
每个对象都有一个属性stateMachine 处理状态转换逻辑  
```
private final StateMachine<RMAppState, RMAppEventType, RMAppEvent>
                                                               stateMachine;
```
状态机将对象的输入，状态转换，输出过程梳理为如下五个过程：
![](/images/stateMachine2.png)  
1. 在prevState状态，接收到事件event，变成postState状态   
2. 在prevState状态，接收到事件event，触发hook操作，变成postState状态  
3. 在prevState状态，接收到事件event1或event2或event3，变成postState状态
4. 在prevState状态，接收到事件event1或event2或event3，触发hook操作，变成postState状态  
5. 在prevState状态，接收到事件event，触发hook操作，变成postState1或postState2或postState3状态  
通过StateMachineFactory 对象实现上面对应过程。

### 3.1 缓存
StateMachineFactory 其内部维护两个缓存:  
1. Map<STATE, Map<EVENTTYPE,Transition<OPERAND, STATE, EVENTTYPE, EVENT>>> stateMachineTable  
2. final TransitionsListNode transitionsListNode  

**stateMachineTable**  
是两层map ，存放状态和事件类型对应的转移操作（很大程度上是为了检索更加方便），表示某个状态碰见某个事件类型，触发什么转换操作。  
其存储结构可以通过下图来表示  
![](/images/stateMachine3.png)

**transitionsListNode**  
是一个状态转移操作对象（ApplicableSingleOrMultipleTransition）列表，如下图所示  
![](/images/stateMachine4.png)
ApplicableSingleOrMultipleTransition 有三个属性，preState,事件类型，触发操作。
stateMachineTable 需要在 transitionsListNode支持下才能构建。

### 3.2 核心逻辑
### 3.3 执行堆栈
