---
layout: post
title: "NSOperation深入研究"
date: 2013-10-31 20:12
comments: true
categories: iOS
tags: NSOperation
---
这篇文章是[如何使用NSOpertion](/2013/10/31/ru-he-shi-yong-nsoperation/)的后续，将详细深入的总结NSOperation。
<!--more-->

##1.自定义operation对象的执行行为

当我们创建好一个operation对象之后，在将其加入到queue之前，可以对此operation进行一些配置。下面我们描述的对operation对象的配置适用于所有的operations类型，不管其是operation的自定义子类，还是系统提供的NSInvocationOperation、NSBlockOperation。

###(1) 配置依赖关系
添加依赖是按序执行一组opertaion对象的常用方法。建立两个operation对象之间的依赖关系通过NSOperation的addDependcy:方法。这个依赖关系意味着当前的operation不会被执行，直到依赖的operation执行完成。两个有依赖关系的operations可以是在不同的operation queue中，因为operation是自己管理自己的依赖的，和queue无关。但是要注意，不要去创建循环依赖，这将导致死锁。

另外一点，我们应该在operation运行之前或者加入operation queue之前，给其添加依赖。如果给一个正在运行的operation添加依赖，将没有任何效果。

###(2) 改变operation的execution priority
当operations加入到queue中之后，其执行顺序主要有两个条件决定：首先，取决于operation是否准备就绪，其次，是operation优先级。一个operation的是否准备就绪主要是由其依赖的operations是否执行完成来决定的，而其优先级是其自己的一个属性。缺省情况下，所有的operation对象都有一个normal优先级，但是我们可以通过调用operation的setQueuePriority:方法去设置改变其优先级。

operation的优先级只对同一个queue中的operations才有效。因此，一个queue中低优先级的operation可能先于先于另一个queue中高优先级的operation执行。

operation的优先级并不是依赖的一个替代品，只有在两个operation都处于准备就绪状态的时候，operation queue才根据优先级的大小去决定先执行哪一个operation。如果一个高优先级的operation没有准备好，而一个低优先级的operation已经准备好了，则queue会先执行低优先级的operation。因此，如果想要一个operation在另外的operation执行完成之后再执行，必须使用dependency。

###(3) 改变相关线程的priority
从OX v10.6和iOS4开始，可以配置运行operation的线程的优先级。在系统中，线程的优先级一般是由内核来管理。一般高优先级的线程会比低优先级的线程赋予更多的执行机会。在operation对象中，可以设置通过浮点数设置线程的优先级，从0.0到1.0。0.0是最低的优先级，而1.0是最高优先级。如果不设置，则运行operation的线程优先级默认为0.5。

如果要设置一个operation的线程优先级，则必须在operation对象添加到queue之前通过调用方法setThreadPriority:来设置。当operation准备执行的时候，缺省的start方法会使用此值去设置执行任务的线程的优先级。此新的优先级只会影响到operation的main方法，其他的方法包括operation的completion block都运行在缺省的优先级下。当我们创建concurrent operation，则我们必须自己在重写的start方法中设置线程优先级。

###(4) 设置Completion Block
从OX v10.6和iOS4开始，一个operation在其task执行完成之后，可以执行一个completion block。例如，我们可以使用此block通知外部operation已经完成。通过operation的方法setCompletionBlock:来设置。

##2.实现Operation的注意事项

尽管一个operation对象很容易实现，但是在我们写代码的时候有几项也是需要考虑的，下面将详细说明。

###(1) 避免使用线程存储（pre-thread storage）
对于大多数nonconcurrent operation来说，其都是在单独的线程上运行的，而这些线程都是有operation queue提供的。如果operation queue为我们提供一个线程，我们一定要注意此线程是属于queue而不是operation的。具体来讲，就是不要将operation中的数据关联到thread上，因为thread是有operation queue管理的，依赖于当前的系统。因此，如果通过thread存储在operations之间传递数据是不和靠的，很可能失败。

在operation对象中，在任何情况下都不要去使用线程存储。当我们创建一个operation对象的时候，就应该提供其所需要的数据了。
(关于线程存储：每一个线程都有一个可以访问的字典，我们可以使用这个字典存储我们想要的保存信息，可以通过线程NSThread的threadDictionary方法获取到)

###(2)要保持对operation对象的引用
因为大多数operation对象都是异步运行的，我们不应该在创建它们，然后加入queue之后，就认为结束了。当在operation完成之后，我们需要从operation中得到结果数据，则在之前保持一个对operation的引用是很重要的。

当我们将一个operation对象加入到queue中之后，虽然可以通过queue的方法operations获取到operation对象，但是当一个operation运行完成之后，其会从queue中移除，所以通过方法operations也是获取不到的。因此保持对operation对象的引用是很有必要的。

###(3)处理错误和异常
因为一个operation在一个应用中本质上是一个离散的实体，因此我们有责任去处理operation运行中遇到的error和exception。从OS X v10.6和iOS4开始，NSOperation缺省的start方法就没有提供异常的捕获，我们自己的代码应该去捕获和处理这些异常。

##4.其他

###(1)关于性能
尽管我们可以向一个operation queue中添加任意数量的operation对象，但是我们也要考虑其他因素。例如，我们向一个queue中添加成千上万个operation，但operation中只进行了很少量的工作，则会发现大量的时间都耗费在了operation的分发上了。

因此，关键是一个权衡。例如，我们的应用需要创建100个operation对象，每个operation进行相同的任务，并返回值，则一个比较好的处理办法是创建10个operation对象，每一个对象返回10个不同的结果值。

我们也要避免同一时刻向queue中添加大量的operation对象，避免加入operation数量的速度超过queue执行operation数量的速度。一种比较好的解决办法是分批量加入operation对象，当一批operation对象执行完成之后，我们可以通过block通知系统可以在创建一批operation对象加入。

###(2)手动执行operation
尽管将一个operation加入到queue中，让queue管理其执行非常方便，但是有些情况下，我们可能会选择手动执行一个operation，这时我们就应该注意，此operation必须在准备就绪之后才能运行，我们必须通过调用其start方法使其运行。

下面是一个例子，在我们手动执行一个operation的时候应该调用，如果这个方法返回NO，则我们应该启动一个定时器，到时间的时候再次调用，直到其返回YES。当返回YES的时候，我们也需要判断是否是被取消了。

```objective-c
- (BOOL)performOperation:(NSOperation*)anOp
{
   BOOL ranIt = NO;
 
   if ([anOp isReady] && ![anOp isCancelled])
   {
      if (![anOp isConcurrent])
         [anOp start];
      else
         [NSThread detachNewThreadSelector:@selector(start)
                   toTarget:anOp withObject:nil];
      ranIt = YES;
   }
   else if ([anOp isCancelled])
   {
      // If it was canceled before it was started,
      //  move the operation to the finished state.
      [self willChangeValueForKey:@"isFinished"];
      [self willChangeValueForKey:@"isExecuting"];
      executing = NO;
      finished = YES;
      [self didChangeValueForKey:@"isExecuting"];
      [self didChangeValueForKey:@"isFinished"];
 
      // Set ranIt to YES to prevent the operation from
      // being passed to this method again in the future.
      ranIt = YES;
   }
   return ranIt;
}
```

关于NSOperation的内容先到这里，有问题大家可以留言交流。
