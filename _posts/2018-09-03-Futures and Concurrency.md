---
layout:     post
title:      Futures and Concurrency
date:       2018-09-03
author:     qiqi
header-img: img/post-bg-cook.jpg
catalog: true
tags:
    - Scala,Future
---


- 在`Scala`中的`Future`，可以在没有计算完成的时候就进行转换，执行`Future`的线程是由一个隐式的`execution context`决定的，因此在`Scala`上的`Future`运算就是在不同的`Future`之间进行转换，从而不必去关心共享内存和锁的问题。
- 在`Scala`中可以使用`Java`中的并发原语，比如`wait，notify，notifyAll`以及`sychronized`等，在`Scala`中`sychronized`的实现是一个方法，但是使用共享内存和锁的机制保证数据的正确性是非常困难的，必须对数据的状态进行推断并避免死锁的出现。  并且测试也是困难的，因为线程的运行具有不确定性。在`Java`中，`java.util.concurrent`包中提供了一些高级的方法，但是依然建立在共享内存和锁的基础上，因此这个模型使用起来依然很麻烦。

### 异步执行
- `Scala`中的`Future`是在额外的线程异步执行的，所以对`Future`进行操作的时候，需要引入一个额外的隐式参数，`executation context`用来异步执行这些`Future`，可以使用`future`的`isComplete`方法来检查`Future`是否执行完毕，可以使用`value`方法得到`Future`的值，`value`方法值返回的是一个`Option[Try[T]]`对象，单独靠`Option`的`Some`和`None`表示`Future`是否计算完成，对于计算完成的计算结果，需要额外的类来进行辅助，`Try`有两个子类，一个是`Success`，一个是`Failure`，`Success`表示计算成功，`Failure`表示计算失败。因为产生异常的线程已经去处理别的工作了，所以需要把异常信息传回给主线程。

### 使用Future进行工作
- 可以对`Future`进行变换得到一个新的`Future`。变换的方法主要有：

#### 1.使用Map函数
- 在`Map`函数中，生成新`Future`的`f`转换函数必须要等原来的`Future`计算完成之后才能进行。

#### 2.使用for表达式
- 有多个`Future`表达式的情况下，可以使用`for`表达式，将这些结果串联起来，因为`for`拆来就是`flatMap`，`flatMap`也是需要等到当前`Future`的结束才能进行转换操作。所以一般是将`Future`的创建放在`for`的外部，在`for`中只需要引用这个变量就可以。不然嵌套的`future`全部变成了串行运行的。

### 创建Future
- 使用 `Future.failed, Future.successful, Future.fromTry`, 和`Promises`来创建`Future`对象。除了使用`Future.apply`方法之外，可以使用`Future`对象中的`failed，successful，fromTry`来创建对象，在这些创建过程中，不需要引入额外的`ExecutionContext`。另外一种方式是使用`Promise`进行创建，可以通过获取`promise`中的`future`来创建`Future`对象。

#### 使用filter和collect
- 使用`filter`，如果`future`的结果符合`filter`，则返回原来的`future`；否则的话，会返回`Failure(NoSuchElementException)`。在`Future`中存在`withFilter`方法，所以可以在`for`表达式中使用`if`表达式进行筛选
- `collect`的作用相当于是`filter`和`map`的使用，但是`collect`的入参是一个部分函数。

#### 处理Future中的失败情况
- 使用`failed，fallBackTo，recover，recoverWith`。
```scala
scala > val failure = Future {
    42 / 0
}
failure: scala.concurrent.Future[Int] =...
scala > failure.value
res23: Option[scala.util.Try[Int]] = Some(Failure(java.lang.ArithmeticException: / by zero))
scala > val expectedFailure = failure.failed 
expectedFailure: scala.concurrent.Future[Throwable] =...
scala > expectedFailure.value
res25: Option[scala.util.Try[Throwable]] = Some(Success(java.lang.ArithmeticException: / by zero))
```
`failed`方法可以将`Future`中返回的结果由`Some(Failed)`改为`Some(Success)`，但是`Option[Success[T]]`中的类型也是需要变化的，是`Throwable`类型，但是如果一个`Future`是计算成功的，调用`failed`方法则会返回`Option[Failed[NoSuchElementExecption]`，使用`fallbackTo`方法，可以将失败的`Future`设为到其他的`Future，val failure = fallbackTo(success)`。根据`fallbackTo`的逻辑，则`origin.fallbackTo(target)`，如果`origin`是成功的，直接返回；`origin`是失败的，如果`target`是成功的，返回`target`，如果`target`是失败的，直接返回`origin`。

#### recover
- `recover`接收一个部分函数，可将失败的`Future`转为成功的`Future`，使用的是偏函数，使用`case`进行匹配进行，`case e: Exception => 1`这样的偏函数，将失败的`Future`转换为成功的`Future`。`recover`最后调用的是`Try[T]`中的`recover`函数，对于成功的`Future`，`Success`中定义的`recover`函数将直接返回这个`Try[T]`，如果在`recover`中的`case`没有匹配到这个`Exception`时，这个`Exception`直接透传；如果在`recover`中的`case`匹配上了，但是在处理的过程中出现了新的异常，那么这个新的异常就会传到最后的结果中。

#### recoverWith 
- `recoverWith`和`recover`很像，但是`recover`中只能返回一个值，在`recoverWith`的偏函数中返回的是一个`Future`。和`recover`类似，对于成功的`Future`，直接返回；失败的`Future`，如果没有匹配上，则直接透传，如果匹配上出现了异常，则将该异常进行透传。

#### transform
- `transform`可以对`Success`和`Failure`中的值进行转换，只能将成功的`Future`转换为成功的`Future`，失败的`Future`转换为失败的`Future`，而不能在这两者之间进行转换。在`Scala2.12`中进行了修改，失败的`Future`也可以转换为成功的`Future`。

#### Futures之间的组合

##### zip,fold,reduce,sequence,traverse
- 1. `zip`将两个`Future`串联成为一个二元组，如果其中有失败的`Future`，则结果为`Some(Failure(exception))`。
- `fold`和`reduce`和集合中的操作是类似的，但是操作的都是`Future`中的值，而不是`Future`本身。如果有失败的`Future`，则进行透传。
- 3. `sequence`将`List[Future[T]]`转换为`Future[List[T]]`
- 4. `traverse`将`List[T]`转换为`Future[List[T]]`。

#### 使用Future的副作用,foreach, onComplete, andThen
- 有时候需要在一个`Future`完成之后使用其副作用，主要的有提到的以上三种。`foreach`会被重写成为`for`形式的`map`和`flatMap`。还有回调函数`onComplete`，`future`不保证注册到`onComplete`的函数按顺序执行。如果需要控制执行的顺序，需要使用`andThen`函数，`andThen`复制了原始的`future`。

#### 使用Futurn进行测试
- 在`JVM`上的线程是有限的，如果线程的占用率很高，那么后期线程之间的切换会造成效率的大大降低。
- 使用`Await.result(future, duration)`可以阻塞`future`一直到其完成，如果超过`duration`的时间还是没有完成，则会抛出`Timeout`异常。







