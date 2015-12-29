title: 基于Spring的动态更新实现 设计草稿
category: TVDCR
date: 2015-12-13
tags: [spring, transacion, dynamic reconfiguration]
author: Hooting
---
通过前段时间的学习，初步的构想是：自定义TransactionManager，利用Spring的事务管理框架，继承AbstractPlatformTransactionManager类，管理Spring应用程序中Bean之间的调用历史。利用ZooKeeper的特点，解决Spring应用程序在并行执行事务时的一致性问题。

<!--more-->

---

## 框架
通过前段时间的学习，初步的构想是：自定义TransactionManager，利用Spring的事务管理框架，继承AbstractPlatformTransactionManager类，管理Spring应用程序中Bean之间的调用历史。利用ZooKeeper的特点，解决Spring应用程序在并行执行事务时的一致性问题。希望实现的框架支持如下功能：

* **一致性** 保证同一个事务上下文由同一个版本的Bean完成
* **最小化更新时受关联Bean** 根据程序开发人员设定的事务传播属性，确定更新时受关联组件集合（传播属性为Require_new的Bean和它直接或间接依赖的Bean构成受关联组件集合。传播属性为Require的Bean对应的方法操作属于一个子事务）

![](/images/framework.jpg)
### Refresh
在应用启动时，启动一个Refresh Bean，监听`ZK`特定的节点(Refresh节点)，当`ZK`中创建了Refresh节点后，执行Spring Context的更新工作。

### TxManager

`TxManager`管理事务的调用关系，每次创建、完成事务时，更新保存在`ZK`的计数器。它的主要实现包括:
* doBegin 当需要创建一个上下文时被调用，创建一个连接`ZK`的connection，并使`ZK`的计数器加1
* doCommit 当一个上下文完成时被调用，使`ZK`的计数器减1，并断开连接
* doGetTransaction 当开始一个子事务时被调用，更新与当前线程绑定的结构体：TxInfo(纪录当前上下文已调用的组件)
* doRollback 当调用的方法抛出异常时被调用，更新与当前线程绑定的结构体：TxInfo
* isExistingTransaction 当需要创建一个事务时被调用，若返回true则需要创建一个新的上下文，否则加入当前事务中

### ZK
`ZK`维护一个类似文件系统的数据结构，它的目录结构如下。

![](/images/zk-structure.jpg)

Spring应用程序中，每一个scope<sup>[1](#scope)</sup>都对应`ZK`的一个永久节点：Scope_n。 该节点的内容纪录该scope下当前并行运行上下文的数量。每个Scope_n节点下，分布三个节点Children、Request、Wait。其中Children是一个永久节点，当更新请求发生时，每一个上下文告知其是否曾经调用了待更新组件， Request和Wait节点均是临时节点，它们在需要更新组件时由`Update`来创建。

<a name="scope">1</a>: 每一个propagation属性值为REQUIRES_NEW的方法，与该方法依赖的直接或间接方法所组成的集合构成一个scope。
### Update
`Update`是发起更新请求的入口。当需要更新Spring应用程序某个组件时，它在`ZK`上对应的Scope_n节点上创建Request节点。`TxManager`中的connection在创建时注册了`ZK`的isExist Watcher，所以当Request节点创建后，每个connection会收到通知。收到通知后，`TxManager`先阻塞当前运行的事务，并根据它对应的TxInfo，在`ZK`的目录/Scope_n/Children下创建节点，节点名称是(Y/N)_Con_n，表示该上下文曾经调用了待更新组件（Y）或未调用（N）。

当`Update`发现／Scope_n/Children下节点数量与/Scope_n保存的计数器值一致时，可以保证每个当前运行的上下文都已给出了答复。此时，`Update`可以根据节点的名称(Y/N)确定需要等待哪些上下文结束。同时，`Update`在/Scope_n下创建/Wait节点，通知`TxManager`继续上下文的执行。

当上下文结束调用doCommit时，会断开与`ZK`的连接，因为/Scope_1/Children下创建的子节点是Ephemeral类型，所以对应的子节点也被删除。`Update`在/Scope/Children注册了 getChildren Watcher，所以每次变化都会被通知。当`Update`发现所有曾经调用的上下文(Y_Con_n)都结束时，即可以进行更换新组件的工作。即待更新组件进入free状态

当待更新组件进入free状态时，`Update`向`ZK`创建Refresh节点，`Refresh`收到isExist通知，读取保存在节点Refresh的值，该值是新Bean的地址，然后加其加载至内存并替换已创建的Bean实例。

## TODO Refresh
Spring的AbstractApplicationContext 提供了refresh方法，可以更新application的configuration，但是只能在下次getBean时起作用，而refresh之前创建的bean实例，并不能被更新。

> [Note: when deploying a bean in the prototype mode, the lifecycle of the bean changes slightly. By definition, Spring cannot manage the complete lifecycle of a non-singleton/prototype bean, since after it is created, it is given to the client and the container does not keep track of it at all any longer. You can think of Spring's role when talking about a non-singleton/prototype bean as a replacement for the 'new' operator. Any lifecycle aspects past that point have to be handled by the client.](http://docs.spring.io/spring-framework/docs/1.2.7/reference/beans.html)

由此可见，要想更新已创建的Bean实例，通过Spring Container实现不可行。




