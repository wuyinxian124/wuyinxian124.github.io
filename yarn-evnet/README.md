# 事件处理

## 1. 什么是事件处理

众所周知，YARN中，特别是resourceManager 内部实现逻辑复杂，涉及的对象多，同时对处理性能要求比较高。为了解决这些问题，yarn提出采用事件处理器的方式来提高性能，梳理复杂逻辑。

## 2. 事件处理实现结构

YARN 中事件处理器主要包括两个部分：状态机，中央异步调度器。  
状态机主要用来处理对象内部的操作和状态转换，比如app状态转换和对应操作。  
中央异步调度器，处理不同状态机之间的沟通。

## 3. 事件处理的优势

异步处理，并发度更高。  
调度被高度抽象，逻辑更加简洁，明了。

## 4. 中央异步调度器

## 5. 状态机

## 6. nodeManager 中事件处理用例

