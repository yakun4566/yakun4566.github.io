本文主要介绍了 Spring 事务传播性的相关知识。

Spring 中定义了 7 种事务传播性：

PROPAGATION_REQUIRED

PROPAGATION_SUPPORTS

PROPAGATION_MANDATORY

PROPAGATION_REQUIRES_NEW

PROPAGATION_NOT_SUPPORTED

PROPAGATION_NEVER

PROPAGATION_NESTED

在 Spring 环境中，含有事务的方法嵌套调用，事务是如何传递的规则，以及每种规则是如何开展工作的。文章还提到每种事务传播性是如何使用的，方便读者依据实际的场景，使用不同的事务规则。

一、什么是 Spring 事务的传播性

Spring 事务传播性是指， 在 Spring 的环境中，当多个含有事务的方法嵌套调用时，每个事务方法都处于自己事务的上下文中，其提交或者回滚行为应该如何处理。

通俗讲，就是当一个事务方法调用另外一个事务方法时，事务如何跨上下文传播。

![image](https://github.com/yakun4566/yakun4566.github.io/assets/18443998/71d034b4-c2a5-4472-99e6-cb3997b7fd68)


1）当事务方法 A 调用事务方法 B 时，事务方法 B 是合并到事务方法 A 中，还是开启新事务？

2）当事务方法 B 抛出异常时  ，在合并事务或者开启新的事务的场景中，事务的回滚是如何处理的 ？

以上事务的处理规则，都取决于事务传播级别的设置。

二、事务的传播性都有哪些行为

![image](https://github.com/yakun4566/yakun4566.github.io/assets/18443998/f93f779b-389d-4127-a199-ae58fa2629cc)


事务的传播行为，主要分为三种类型，分别是：支持当前事务、不支持当前事务、嵌套事务。

2.1 支持当前事务

REQUIRED：默认的事务传播级别，表示如果当前方法已在事务内，该方法就在当前事务中执行，否则，开启一个新的事务并在其上下文中执行。

SUPPORTED：当前方法在事务内，则在其上下文中执行该方法，否则，开启一个新的事务。

MANDATORY：必须在事务中执行，否则，将抛出异常。

2.2 不支持当前事务

REQUIRES_NEW：无论当前是否有事务上下文，都会开启一个事务  。如果已经有一个事务在执行 ，则正在执行的事务将被挂起 ，新开启的事务会被执行。

事务之间相互独立，互不干扰。

NOT_SUPPORTED：不支持事务，如果当前存在事务上下文，则挂起当前事务，然后以非事务的方式执行。

NEVER：不能在事务中执行，如果当前存在事务上下文，则抛出异常。

2.3 嵌套事务

NESTED：嵌套事务，如果当前已存在一个事务的上下文中，则在嵌套事务中执行，如果抛异常，则回滚嵌套事务，而不影响其他事务的操作。

三、每种事务的传播性如何工作

3.1 REQUIRED

默认的事务传播行为，保证多个嵌套的事务方法在同一个事务内执行，并且同时提交，或者出现异常时，同时回滚。

这个机制可以满足大多数业务场景。

![image](https://github.com/yakun4566/yakun4566.github.io/assets/18443998/2bc89917-b525-4df8-a516-940f4928dec3)

例子 ：

![image](https://github.com/yakun4566/yakun4566.github.io/assets/18443998/50a50469-7301-41e3-96d6-f180b861af0e)

1）类 TestAService 的方法通过声明式事务的方式，加上了事务注解 @Transactional ，并设置事务的传播性为 REQUIRED。

2）调用者调用 TestAService 的 A 方法时，如果调用者没有开启事务，那么 A 方法会开启一个事务。

A 方法的具体执行过程如下 ：

a. 执行 insert，但没有提交；

b. 调用 TestBServcie 的 B 方法，由于 B 方法也声明了事务，并且传播性是 REQUIRED，所以方法 B 的事务，合并到方法 A 开启的事务中。

c. 方法 B 执行 insert 操作，此时也没有提交。

3）由于这两个方法的操作都在同一个事务中执行，当这两个方法所有操作执行成功之后，提交事务。

嵌套调用链路：

![image](https://github.com/yakun4566/yakun4566.github.io/assets/18443998/112def4e-22e4-4cc2-a525-f2e6c17f9c4d)

当方法 B 执行时抛出了 Exception 异常后，事务是如何处理的 ？

![image](https://github.com/yakun4566/yakun4566.github.io/assets/18443998/7e65ccd4-afaf-46be-9b3d-04cdc7d41524)

1）方法 B 声明了事务，insert 操作会回滚

2）由于方法 A 和方法 B 同属一个事务，方法 A 也会执行回滚，由此说明该规则保证了事务的原子性。

嵌套调用，异常后的链路：

![image](https://github.com/yakun4566/yakun4566.github.io/assets/18443998/747d0b4b-d62f-4a0e-ab59-84599a320894)

如果 方法 B 抛出异常后，方法 A 使用 try-catch 处理了方法 B 的异常（如下代码），并没有向外抛出，此时事务又如何处理的 ？

![image](https://github.com/yakun4566/yakun4566.github.io/assets/18443998/ba553a33-7309-4b57-9955-e7cbf6c2cedc)

方法 A 也会回滚。

从事务的特性我们可知，事务具有原子性。方法 A 和方法 B 同属一个事务，当方法 B 抛出异常，触发回滚操作后，整个事务的操作都会回滚。

因此，Spring 在处理事务过程中，当事务的传播性设置为 REQUIRED，在整个事务的调用链上，任何一个环节抛出的异常都会导致全局回滚。

3.2 REQUIRES_ NEW

每次都开启一 个新的事务。

![image](https://github.com/yakun4566/yakun4566.github.io/assets/18443998/c04c4f6e-278a-4234-9e66-5388decfecce)

例子：

![image](https://github.com/yakun4566/yakun4566.github.io/assets/18443998/30240515-5581-4e31-96d8-4931ad9fb841)

上面例子中，方法 B 的传播性设置为 REQUIRES_NEW，方法 A 仍然是 REQUIRED，当 A 调用 B 时，具体调用链路如下：

![image](https://github.com/yakun4566/yakun4566.github.io/assets/18443998/e7a9a09e-8221-4602-9285-70cc114ee912)

具体执行过程：

方法 A 被执行前，如果调用者没有开启事务，方法 A 开启一个事务 1，然后执行 insert ，此时没有提交；

方法 B 的事务传播性设置为 REQUIRES_NEW，当被方法 A 调用时，此时方法 A 的事务 1 会被挂起，方法 B 开启自己的事务 2，然后执行 insert，此时并没有提交；

当方法 B 执行完毕后，提交事务 2；

恢复事务 1，最终提交。

当 方法 B 执行时抛出了异常，会发生什么？

方法 B 的 insert 操作会被回滚掉，方法 A 不受影响。但这里有个前提，方法 A 需要 try-catch 方法 B 的异常，使其异常不会往上传递，从而导致方法 A 接收到异常，导致回滚。

![image](https://github.com/yakun4566/yakun4566.github.io/assets/18443998/3eb92e2a-1616-4dc3-8206-6f3b50933fe5)

3.3  SUPPORTED

当外层方法 A 存在事务，方法 B 加入到当前事务中，以事务的方式执行。

![image](https://github.com/yakun4566/yakun4566.github.io/assets/18443998/f9ee4955-6db7-4951-afed-3e17b517a890)

当外层方法 A 不存在事务，方法 B 不会创建新的事务，以非事务的方式执行。

![image](https://github.com/yakun4566/yakun4566.github.io/assets/18443998/a6baf60b-4067-487a-8be5-d521606957e6)

例子 1：

![image](https://github.com/yakun4566/yakun4566.github.io/assets/18443998/8813612d-b83b-4c88-907e-7953969d792e)

以上例子，方法 A 没有加事务注解，方法 B 的加了事务注解，并且传播为 SUPPORTS。

具体执行过程：

方法 A 以非事务的方式执行 insert 操作。

方法 B 被调用，由于其外层事务 A 没有开启事务，方法 B 也是以非事务方法执行 insert 操作。

![image](https://github.com/yakun4566/yakun4566.github.io/assets/18443998/2b2ea03d-756b-4982-910c-3b5f80cd9529)

例子 2：

![image](https://github.com/yakun4566/yakun4566.github.io/assets/18443998/5ee9a1b5-8051-4f58-b0ef-3f91b4e45e8c)

以上例子，方法 A 和 B 都加上了事务注解，其中方法 A 的传播性为 REQUIRED，方法 B 的传播性为 SUPPORTS。

具体执行过程：

如果方法 A 的调用方没有开启事务，则方法 A 开启事务，并执行 insert 操作，但没有提交；

方法 B 被调用，由于其外层方法 A 开启了事务，因此方法 B 加入到方法 A 开启的事务中，并执行 insert, 但没有提交；

当事务中的所有操作执行成功后，事务提交。

![image](https://github.com/yakun4566/yakun4566.github.io/assets/18443998/3b3baa36-d27c-41b3-bfbd-43552d3f7a7b)

3.4  NOT_SUPPORTED

不支持事务。

如果外层方法存在事务，则挂起外层事务，以非事务方式执行，执行完毕后，恢复外层事务。

![image](https://github.com/yakun4566/yakun4566.github.io/assets/18443998/d5b651a3-73d9-4922-a4ad-66e44486c1e9)

例子：

![image](https://github.com/yakun4566/yakun4566.github.io/assets/18443998/e60b4c4d-3585-4a32-91ad-20023c7a6abf)

以上例子：方法 A 和 B 都加上了事务注解，方法 A 的传播性为 REQUIRED, 方法 B 为 NOT_SUPPORTED。

具体执行过程：

如 A 的调用方没有开启事务，方法 A 开启事务，并执行 insert，但没有提交。

方法 A 调用方法 B 时，方法 B 的传播性为 NOT_SUPPORTED, 不支持事务，然后挂起外层方法 A 的事务，方法 B 以非事务的方式执行 insert。

方法 B 执行完毕后，恢复方法 A 的事务，最终提交事务。

调用链路过程：

![image](https://github.com/yakun4566/yakun4566.github.io/assets/18443998/ff5c4c19-ac74-41a0-9cdc-6488aa56ac68)

3.5 NEVER

不支持事务

当外层方法 A 开启了事务，方法 B 抛出异常

![image](https://github.com/yakun4566/yakun4566.github.io/assets/18443998/0cd2c17d-2a7a-48f2-b75e-0a12ad88bfb1)

例子：

![image](https://github.com/yakun4566/yakun4566.github.io/assets/18443998/ce04854f-6eb5-4e0e-a616-0fcf10f89f4d)

以上代码，两个方法都打上了事务注解，方法 A 的传播性是 REQUIRED, 方法 B 的传播性是 NEVER。

具体执行过程：

方法 A 开启事务，执行 insert, 没有提交。

含有事务的方法 A 调用方法 B，方法 B 的传播性是 NEVER, 表示不支持事务，因此方法 B 抛出异常。

方法 A 的事务执行回滚。

![image](https://github.com/yakun4566/yakun4566.github.io/assets/18443998/648820e5-6443-4c6e-80ef-a598e605c802)

3.6 MANDATORY

必须在事务中执行。

如果外层方法 A 没有开启事务，方法 B 抛出异常。

![image](https://github.com/yakun4566/yakun4566.github.io/assets/18443998/ac49a1c0-4479-4e81-8fc6-9eba2bd5d9e4)

如果外层方法 A 开启了事务，方法 B 加入事务，方法 A&B 在同一事务中执行。

![image](https://github.com/yakun4566/yakun4566.github.io/assets/18443998/5f748e93-efaf-437d-874b-72f9ed43c2f4)

例子：

![image](https://github.com/yakun4566/yakun4566.github.io/assets/18443998/3d34c7c2-c95b-4fc3-99c6-78b0ceefd10c)

以上例子，方法 A 没有加事务注解，方法 B 的传播性为 MANDATORY。

具体执行过程：

方法 A 的调用方如果本身没有开启事务，方法 A 执行前不会开启事务。

当非事务方法 A 调用方法 B 时，由于方法 B 的传播性为 MANDATORY，必须在事务中执行，条件不满足，抛出异常。

![image](https://github.com/yakun4566/yakun4566.github.io/assets/18443998/b1f283ed-ed56-47ea-9c43-894bcde6a2f5)

3.7 NESTED

嵌套事务

如果外层方法 A 不存在事务，内层方法 B 的规则与 REQUIRED 一致。

如果外层方法 A 存在事务，内层方法 B 做为外层方法 A 事务的子事务执行，两个方法是一起提交，但子事务是独立回滚。

内层方法 B 抛出异常，则会回滚方法 B 的所有操作，但不影响外层事务方法 A。（方法 A 需要 try-catch 子事务，避免异常传递到父层事务）

外层方法 A 回滚，则内层方法 B 也会回滚。

该传播性的特点是可以保存状态点，当回滚时，只会回滚到某一个状态点，保证了子事务之间的独立性，避免嵌套事务的全局回滚。

![image](https://github.com/yakun4566/yakun4566.github.io/assets/18443998/6b2cefd2-a76b-422d-9781-6bf505242962)

例子：

![image](https://github.com/yakun4566/yakun4566.github.io/assets/18443998/ede90b12-be13-4630-b2c7-d53b8174f8da)

以上例子，方法 A 的传播性为 REQUIRED, 方法 B 为 NESTED。

具体执行过程：

方法 A 执行时，如调用方没有开启事务，则开启一个事务。

方法 B 被外层方法 A 调用时，因为方法 B 的传播性为 NESTED，方法 B 在此处建立 savepoint, 标记 insert 行为。

当方法 B 抛出异常，其 insert 操作会回滚，但只会回滚到 savepoint，（前提是方法 A 要 try-catch 方法 B，使方法 B 的异常不会往外传递）。

方法 B 回滚后，方法 A 的事务提交。

调用链路：

![image](https://github.com/yakun4566/yakun4566.github.io/assets/18443998/ff1cf1ca-be3a-405d-af11-84f013327c90)

四、总结

本文解释了 Spring 框架中的事务传播性，即多个业务方法之间调用时事务如何处理的规则。Spring 提供了七种传播级别，如

PROPAGATION_REQUIRED、

PROPAGATION_REQUIRES_NEW 等。

每种级别都有适用场景和限制，本文提供了一些示例，介绍了声明式事务如何使用，每种事务的规则，产生哪种行为，当方法抛出异常时，事务的提交和回滚是如何被处理的。正确处理事务对于任何企业级应用程序都是必要的，了解 Spring 事务传播性是构建高效、可靠和可扩展应用程序的关键。

END