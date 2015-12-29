title: Spring Transaction 实例
category: TVDCR
date: 2015-11-20
tags: [spring, transacion, dynamic reconfiguration]
author: Hooting
---
Spring提供了编码式和声明式事务管理的支持。通常，你不需要编码式事务所提供的细粒度控制，而会选择在上下文定义文件中声明事务。本文通过一个简单的应用，介绍Spring声明式事务管理的开发过程。

<!--more-->

---

## 理解Spring对事务管理的支持
Spring提供了编码式和声明式事务管理的支持。编码式事务允许用户在代码中精确定义事务的边界，而声明式事务（基于AOP）有助于用户将操作与事务规则进行结构。选择编码式事务还是声明式事务很大程度上是在细粒度控制和易用性之间进行权衡。当通过编码实现事务控制时，你能够精确控制事务的边界，它们的开始和结束完全取决于你的需要。通常，你不需要编码式事务所提供的细粒度控制，而会选择在上下文定义文件中声明事务。

本文通过一个简单的应用，介绍Spring声明式事务管理的开发过程。

## Java应用实例介绍
本文以多媒体流转码程序为例，介绍Spring事务对该程序的管理。该转码程序的结构如下图所示，它由四个Bean组成：
* Port 接收用户的转码请求
* Decoder 将用户输入的数据转换成内部表示形式
* Encoder 将内部表示形式转换成用户要求的编码格式
* Segmenter 将大数据切割成小块或将小块数据合并成完整数据

![](/images/converter-sample-structure.jpg)

### Port的实现代码如下所示：
```
package tvdcr.sample.component;

public class Port{
	private Encoder encoder;
	private Decoder decoder;
	
	public Port(Encoder encoder, Decoder decoder){
		this.encoder = encoder;
		this.decoder = decoder;
	}
	
	public void start() {
		System.out.println("start Port");
		decoder.decode();
		encoder.encode();
		System.out.println("end Port");
	}
}
```

### Decoder的实现代码如下所示：
```
package tvdcr.sample.component;

public class Decoder {
	private Segmenter seg;
	public Decoder(Segmenter seg){
		this.seg = seg;
	}
	
	public void decode(){
		System.out.println("start decode");
		seg.partition();
		System.out.println("end decode");
	}
}
```

### Encoder的实现代码如下所示：
```
package tvdcr.sample.component;

public class Encoder {
	private Segmenter seg;
	
	public Encoder(Segmenter seg){
		this.seg = seg;
	}
	
	public void encode(){
		System.out.println("start encode");
		seg.combinate();
		System.out.println("end encode");
	}
}

```

### Segmenter的实现代码如下所示：
```
package tvdcr.sample.component;

public class Segmenter {
	public void partition(){
		System.out.println("partition");
	}
	
	public void combinate(){
		System.out.println("combinate");
	}
}
```

> **Note:** 本文关注Spring Transaction对应用程序的事务支持，所以转码程序没有给出具体的实现，只是用`print`代替之。

## Spring 声明式事务
使用Spring的声明式事务，可以有两种方式：
* Spring的tx命名空间
* @Transactional注解

### 在XML中定义事务
使用tx命名空间，会涉及将其添加到Spring XML配置文件中，同时应该将AOP命名空间包括在内。

```
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
	xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:aop="http://www.springframework.org/schema/aop"
	xmlns:tx="http://www.springframework.org/schema/tx"
	xsi:schemaLocation="
        http://www.springframework.org/schema/beans
        http://www.springframework.org/schema/beans/spring-beans.xsd
        http://www.springframework.org/schema/tx
        http://www.springframework.org/schema/tx/spring-tx.xsd
        http://www.springframework.org/schema/aop
        http://www.springframework.org/schema/aop/spring-aop.xsd">
```

这个tx命名空间提供了一些新的XML配置元素，其中最重要的是`<tx:advice>`,通过它可以定义事务性策略(传播性、隔离级别、回滚规则等)。以下的XML片段显示了`<tx:advice>`是如何声明事务性策略的。
```
<tx:advice id="txAdvice" transaction-manager="txManager">
	<!-- the transactional semantics... -->
	<tx:attributes>
		<!-- all methods starting with 'get' are read-only -->
		<tx:method name="get*" propagation="REQUIRED_NEW" />
		<!-- other methods use the default transaction settings (see below) -->
		<tx:method name="*" propagation="REQUIRED" />
	</tx:attributes>
</tx:advice>
```

对于`<tx:advice>`来说，事务属性定义在`<tx:attributes>`元素中，该元素包含了一个或多个的`<tx:method>`元素。`<tx:method>`元素为某个(或某些)`name`属性(使用通配符)指定的方法定义事务性策略。

当使用`<tx:advice>`来声明事务时，还需要一个事务管理器，需要在`transaction-manager`属性中指定事务管理器的`id`，如上述XML片段所示。

> **Note:** 为了简便起见，本文并没有用具体的`transaction-manager`,如`DataSourceTransactionManager`和`HibernateTransactionManager`，而是用Spring自带的用于测试的`transaction-manager`:`CallCountingTransactionManager`。它的实现如下：
```
public class CallCountingTransactionManager extends AbstractPlatformTransactionManager {

	public TransactionDefinition lastDefinition;

	public int begun;

	public int commits;

	public int rollbacks;

	public int inflight;

	@Override
	protected Object doGetTransaction() {
		return new Object();
	}

	@Override
	protected void doBegin(Object transaction, TransactionDefinition definition) {
		this.lastDefinition = definition;
		++begun;
		++inflight;
	}

	@Override
	protected void doCommit(DefaultTransactionStatus status) {
		++commits;
		--inflight;
	}

	@Override
	protected void doRollback(DefaultTransactionStatus status) {
		++rollbacks;
		--inflight;
	}

	@Override
	protected boolean isExistingTransaction(Object transaction)
			throws TransactionException {
		return false;
	}

	public void clear() {
		begun = commits = rollbacks = inflight = 0;
	}
}
```

`<tx:advice>`只是定义了AOP通知，用于把事务边界通知给方法。但是并没有声明哪些`Bean`应该被通知——我们需要一个切点来做这件事，即定义一个通知器(advisor)。以下的XML定义了一个通知器，它使用`txAdvice`通知所有`tvdcr.sample.component`包下的`Bean`。
```
<aop:config>
	<aop:pointcut id="sampleInterceptor"
		expression="execution(* tvdcr.sample.component.*.*(..))" />
	<aop:advisor advice-ref="txAdvice" pointcut-ref="sampleInterceptor" />
</aop:config>
```

### 定义注解驱动的事务
除了`<tx:advice>`元素，`tx`命名空间还提供了`<tx:annotation-driven>`元素。使用`<tx:annotation-driven>`时，只需要一行XML,并通过`transaction-manager`属性指定特定的事务管理器，如下：
```
<tx:annotation-driven transaction-manager='txManager'/>
```

`<tx:annotation-driven>`元素告诉Spring检测上文中所有的`Bean`边查找使用`@Transactional`注解的`Bean`,而不管这个注解是用在类级别上还是方法级别上。对于每一个使用`@Transactional`注解的`Bean`,`<tx:annotation-driven>`会自动为它添加事务通知。通知的事务属性是通过`@Transactional`注解的参数来定义的。下述代码展示了添加`@Transactional`注解后`Segmenter`的实现：
```
package tvdcr.sample.component;

import org.springframework.transaction.annotation.Propagation;
import org.springframework.transaction.annotation.Transactional;

public class Segmenter {
	@Transactional(propagation=Propagation.REQUIRED)
	public void partition(){
		System.out.println("partition");
	}
	
	@Transactional(propagation=Propagation.REQUIRES_NEW)
	public void combinate(){
		System.out.println("combinate");
	}
}
```



