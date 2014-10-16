---
layout: post
title: "如何使用NSOperation"
date: 2013-10-31 13:04
comments: true
categories: iOS
tags: NSOperation
---

我们知道，NSOperation是一个抽象类，在使用的时候我们不能直接创建它，而是要使用NSOperation的子类，系统已经为我们提供了两个现成的子类`NSInvocationOperation`和`NSBlockOperation`方便我们使用，当然我们也可以自定义NSOperation的子类，本篇文章详细讲述这三种方法，如果对NSOperation不了解的，可以参考另一篇文章[NSOperationQueue and NSOperation](http://www.yuzhongleixueren.com/blog/2013/10/29/nsoperationqueue-and-nsoperation/)。
<!--more-->

##1.NSInvocationOperation
NSInvocationOperation是NSOperation的一个子类，它将一个具体的invocation(NSInvocation类的对象)封装成一个任务去执行，我们也可以用一个对象和对象里的一个方法(selector)去初始化operation。NSInvocationOperation是一个non-current operation。
通过上面我们可以知道，NSInvocationOperation有两个初始化方法：
```objective-c
- (id)initWithInvocation:(NSInvocation *)inv
- (id)initWithTarget:(id)target selector:(SEL)sel object:(id)arg
```
如果selector有返回值，则我们可以在operation结束的时候调用方法
```objective-c
- (id)result
```
获取到。
下面是一个例子：
```objective-c
@interface  InvocationOperationObject: NSObject{
    
}
- (NSInvocationOperation *)taskForSquareCal:(NSUInteger)num;
- (NSInvocationOperation *)taskForSquareCalInvocation:(NSUInteger)num;
@end

@implementation InvocationOperationObject

- (NSInvocationOperation *)taskForSquareCal:(NSUInteger)num{
    NSInvocationOperation *theOp = [[NSInvocationOperation alloc] initWithTarget:self
                                                                        selector:@selector(calSquare:)
                                                                          object:[NSNumber numberWithUnsignedInteger:num]];
    return theOp;
}

- (NSInvocationOperation *)taskForSquareCalInvocation:(NSUInteger)num{
    NSMethodSignature *sig= [[self class] instanceMethodSignatureForSelector:@selector(calSquare:)];
    NSInvocation *invocation = [NSInvocation invocationWithMethodSignature:sig];
    [invocation setTarget:self];
    [invocation setSelector:@selector(calSquare:)];
    NSNumber *numObj = [NSNumber numberWithUnsignedInteger:num];
    [invocation setArgument:&numObj atIndex:2];
    
    NSInvocationOperation *theOp = [[NSInvocationOperation alloc] initWithInvocation:invocation];
    return theOp;
}

- (NSUInteger)calSquare:(NSNumber *)numObj{
    NSUInteger result = [numObj unsignedIntegerValue];
    result = result * result;
    NSLog(@"====result %d", result);
    return result;
}

@end

- (void)benginOperation{
    NSOperationQueue *operationQueue = [[NSOperationQueue alloc] init];
    operationQueue.maxConcurrentOperationCount = 1;
    
    InvocationOperationObject *opObj = [[InvocationOperationObject alloc] init];
    
    NSInvocationOperation *invoOp = [opObj taskForSquareCal:9];
    NSInvocationOperation *invoOp1 = [opObj taskForSquareCal:10];
    
    NSInvocationOperation *invoOp2 = [opObj taskForSquareCalInvocation:11];
    NSInvocationOperation *invoOp3 = [opObj taskForSquareCalInvocation:12];
    
    [invoOp addObserver:self
             forKeyPath:@"isFinished"
                options:NSKeyValueObservingOptionNew|NSKeyValueObservingOptionOld
                context:nil];
    
    [operationQueue addOperation:invoOp];
    [operationQueue addOperation:invoOp1];
    [operationQueue addOperation:invoOp2];
    [operationQueue addOperation:invoOp3];
}

- (void)observeValueForKeyPath:(NSString *)keyPath
                      ofObject:(id)object
                        change:(NSDictionary *)change
                       context:(void *)context{
    if ([keyPath isEqualToString:@"isFinished"]) {
        NSNumber *boolNum = [change objectForKey:NSKeyValueChangeNewKey];
        if ([boolNum boolValue]) {
            NSInvocationOperation *operation = (NSInvocationOperation *)object;
            NSValue *resultValue = [operation result];
            NSUInteger result;
            [resultValue getValue:&result];
            NSLog(@"kvo get the reslut is %d", result);
        }
    }
    
}

```
上面的例子中分别使用两个初始化方法创建了4个NSInvocationOperation对象，对于第一个NSInvocationOperation对象，通过result方法获取其结果。运行结果如下：
```bash
2013-10-31 14:42:14.192 AFNetworking iOS Example[8340:3707] ====result 81
2013-10-31 14:42:14.194 AFNetworking iOS Example[8340:3707] kvo get the reslut is 81
2013-10-31 14:42:14.196 AFNetworking iOS Example[8340:3707] ====result 100
2013-10-31 14:42:14.196 AFNetworking iOS Example[8340:3707] ====result 121
2013-10-31 14:42:14.197 AFNetworking iOS Example[8340:3707] ====result 144
```

##2.NSBlockOperation

NSBlockOperation也是NSOperation的一个子类，通过此类可以管理一个或者多个block的并发执行。我们可以使用此类一次执行多个blocks，而不需要针对每一个block单独创建一个operation对象。当执行多个block的时候，只有当所有的blocks都执行完成的时候，此operation才会被设置为finished。要注意，NSBlockOperation也是一个non-current operation。

下面看例子代码：
```objective-c
- (void)benginOperation{
    NSOperationQueue *operationQueue = [[NSOperationQueue alloc] init];
    operationQueue.maxConcurrentOperationCount = 1;
    
    NSBlockOperation *operation = [NSBlockOperation blockOperationWithBlock:^{
        sleep(3);
        NSLog(@"block 1 current thread %@",[NSThread currentThread]);
    }];
    
    [operation addExecutionBlock:^{
        sleep(3);
        NSLog(@"block 2 current thread %@",[NSThread currentThread]);
    }];
    [operationQueue addOperation:operation];
}
```
看输出结果：
```bash
2013-10-31 15:27:10.426 AFNetworking iOS Example[8494:4907] block 1 current thread <NSThread: 0x15565fc0>{name = (null), num = 5}
2013-10-31 15:27:10.426 AFNetworking iOS Example[8494:370b] block 2 current thread <NSThread: 0x1566b450>{name = (null), num = 6}
```
可以看出两个block是同时执行的。

注意：在blockOperation开始执行的时候，不要在向里面添加block，否则会抛出异常
`*** Terminating app due to uncaught exception 'NSInvalidArgumentException', reason: '*** -[NSBlockOperation addExecutionBlock:]: blocks cannot be added after the operation has started executing or finished'`

当没有将blockoperation添加到operation queue中，而是直接启动即调用start方法，则会阻塞当前线程，直到block执行完，看下面例子：
```objective-c
- (void)benginOperation{
    NSOperationQueue *operationQueue = [[NSOperationQueue alloc] init];
    operationQueue.maxConcurrentOperationCount = 1;
    
    NSBlockOperation *operation = [NSBlockOperation blockOperationWithBlock:^{
        sleep(10);
        NSLog(@"block 1 current thread %@",[NSThread currentThread]);
    }];
    
    [operation addExecutionBlock:^{
        sleep(10);
        NSLog(@"block 2 current thread %@",[NSThread currentThread]);
    }];
    [operation start];
    NSLog(@"end, current thread %@", [NSThread currentThread]);
}
```
结果为：
```bash
2013-10-31 15:34:33.361 AFNetworking iOS Example[8532:3707] block 2 current thread <NSThread: 0x176a3450>{name = (null), num = 5}
2013-10-31 15:34:33.361 AFNetworking iOS Example[8532:60b] block 1 current thread <NSThread: 0x17580e90>{name = (null), num = 1}
2013-10-31 15:34:33.365 AFNetworking iOS Example[8532:60b] end, current thread <NSThread: 0x17580e90>{name = (null), num = 1}
```
因此，当我们使用NSBlockOperation的时候，一般都是放入queue中的。


##3. custom operation

当invocation operation和block operation不能满足我们需求的时候，我们就需要直接子类化operation了。
自定义operation包括实现nonconcurrent operation和实现concurrent opertaion。
对于自定义实现nonconcurrent operation，很简单，我们只需要覆盖main方法，并在其中判断当前是否为取消状态就可以了。但是对于实现concurrent operation，则比较麻烦，需要做一些额外的工作，例如在start方法中生成一个新的线程去执行任务或者异步执行任务，添加必要的KVO通知等等。下面我们分开讲述。

###(1) custom nonconcurrent operation
对于nonconcurrent operation，一般最少实现实现方法，一个main方法，我们用来处理任务的地方，还有一个自定义的初始化方法，我们用于初始化operation的一些数据。下面是一个一般模板：
```objective-c
@interface MyNonConcurrentOperation : NSOperation
@property id (strong) myData;
-(id)initWithData:(id)data;
@end
 
@implementation MyNonConcurrentOperation
- (id)initWithData:(id)data {
   if (self = [super init])
      myData = data;
   return self;
}
 
-(void)main {
   //一般开始的时候先判断是否取消，如果是取消的话，直接返回
   if ([self isCancelled])return;
   
   //……处理相应的任务
   
   //进行相应的通知
}
@end
```

###(2) custom concurrent operation

如果我们想手动的调用operation的start，而不是将其放入到一个operation queue中，但又想让此operation异步的运行，则我们必须把此operation定义为concurrent。

将一个operation定义为concurrent的，我们必须至少override下面方法：

```objective-c
- (void)start
```
所有的concurrent operation都必须重写此方法，用自己自定义的实现代替父类缺省的实现。手动执行一个operation的时候，需要调用此方法。因此，需要在此方法中去创建一个线程或者异步的执行任务。实现中不要调用super start方法。

```objective-c
- (BOOL)isConcurrent
```
返回YES，标记此operation是一个concurrent operation。

```objective-c
- (BOOL)isExecuting
- - (BOOL)isFinished
```
concurrent operation需要自己创建执行环境，并向外部报告当前环境的状态。因此，一个concurrent operation需要自己维护一些状态方便知道什么时间任务开始执行，以及什么时间任务finished，operation必须通过这两个方法来报告状态。
我们实现这两个方法的时候一定要保证其在多线程中调用是安全的。当这两个方法返回的状态发生改变的时候，我们也应该根据对应的key path去生成相应的KVO通知。

下面是一个实现concurrent operation的一般模板：
定义一个MyOperation：
```objective-c
@interface MyOperation : NSOperation {
    BOOL        executing;
    BOOL        finished;
}
- (void)completeOperation;
@end
 
@implementation MyOperation
- (id)init {
    self = [super init];
    if (self) {
        executing = NO;
        finished = NO;
    }
    return self;
}
 
- (BOOL)isConcurrent {
    return YES;
}
 
- (BOOL)isExecuting {
    return executing;
}
 
- (BOOL)isFinished {
    return finished;
}
@end
```
下面是start方法的实现：
```objective-c
- (void)start {
   // Always check for cancellation before launching the task.
   if ([self isCancelled])
   {
      // Must move the operation to the finished state if it is canceled.
      [self willChangeValueForKey:@"isFinished"];
      finished = YES;
      [self didChangeValueForKey:@"isFinished"];
      return;
   }
 
   // If the operation is not canceled, begin executing the task.
   [self willChangeValueForKey:@"isExecuting"];
   [NSThread detachNewThreadSelector:@selector(main) toTarget:self withObject:nil];
   executing = YES;
   [self didChangeValueForKey:@"isExecuting"];
}
```
下面是剩下方法的实现：
```objective-c
- (void)main {
   // Do the main work of the operation here.
   [self completeOperation];
}
 
- (void)completeOperation {
    [self willChangeValueForKey:@"isFinished"];
    [self willChangeValueForKey:@"isExecuting"];
 
    executing = NO;
    finished = YES;
 
    [self didChangeValueForKey:@"isExecuting"];
    [self didChangeValueForKey:@"isFinished"];
}
```

##3.operation中的KVO
最后讲一下NSOperation中的KVO。NSOperation中的一些key path是KVO支持的，有：`isCancelled`,`isConcurrent`,`isExecuting`,`isFinished`,`isReady`,`dependencies`,`queuePriority`,`completionBlock`。

如果我们重写了NSOperation的start方法，或者对其他的一些方法进行了自定义（除了main方法），我们必须保证上面的key path仍是KVO支持的。当我们重写start方法的时候，我们需要关心key path中的`isExecuting`和`isFinished`。

除了系统提供的支持依赖其他的operations，如果我们还想要实现支持其他依赖，则需要重写方法isReady，在自定义依赖满足之前强制返回NO（在实现自定义依赖的时候，要确保调用super isReady方法，以便继续支持系统提供的缺省的依赖管理），当operation的准备状态发生改变的时候，我们要自己生成isReady的KVO通知。

对于key path dependencies，除非我们重写了addDependency:和removeDependency:方法，否则我们不需要自己去生成dependencies的KVO通知。


关于NSOperation先讲到这里，相信在平时的使用中，这些就已经足够了。如果有疑问，可以留言。
