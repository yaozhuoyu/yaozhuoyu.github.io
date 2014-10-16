---
layout: post
title: "NSOperationQueue and NSOperation"
date: 2013-10-29 17:36
comments: true
categories: iOS
tags: NSOperationQueue NSOperation
---

最近一直在看开源库AFNetworking，很值得学习，在看的过程中发现自己在HTTP协议方面很薄弱，有时间一定要好好的看看，感觉很有用。AFNetworking库中是主要使用NSOperationQueue和NSOperation两个类来处理网络request，发现在其中有些概念自己以前理解有误，所以决定写一篇好好总结一下，主要摘自苹果官网文档。
<!--more-->

##1.NSOperationQueue

NSOperationQueue类主要管理一组NSOperation对象的执行。当NSOperation对象添加到queue中之后，NSOperation对象一直在queue中，直到此operation被cancel或者完成其任务。Operaions（不包含已经执行中的operations）在queue中是按照operaion优先级和operaions内部直接的依赖属性组织的，并执行。一个app可以创建多个NSOperationQueue.

Operation内部之间的依赖关系为operation的执行提供了绝对的执行顺序，即使这些operations在不同的queue中。只有在所有依赖的operations都执行完成之后，operation才会考虑准备(ready)去执行。对于准备执行（ready to execute）的operations，operation queue总是执行优先级高的operation。

当向一个queue中添加一个operation之后，我们不能直接将其从queue中移除。一个operation会一直在queue中，直到其汇报其task 结束(finish)。一个operation的任务结束并不意味着其任务完成，也可能是此operation被取消。

取消一个opertaion不会立即将此operation从queue中删除的，取消一个operation的行为取决于当前operation的状态。如果当前operation已经在执行，这意味着只有operation自己在执行代码中判断cancell状态，然后决定是否立即结束；如果operation在队列中，但是并没有开始执行，队列仍然必须(会)调用operation的start方法，以便它可以处理取消事件，并标记为结束(finish)(isFinish方法返回YES)。

下面举一个例子：
```objective-c
@interface MyOperation : NSOperation
@property (nonatomic, assign) int num;
@end

@implementation MyOperation
- (void)main{
    NSLog(@"current operation %d begin", self.num);
    sleep(5);
    NSLog(@"current operation %d end", self.num);
}

- (void)start{
    NSLog(@"========call start %d", self.num);
    [super start];
}
@end

- (void)benginOperation{
    NSOperationQueue *operationQueue = [[NSOperationQueue alloc] init];
    operationQueue.maxConcurrentOperationCount = 1;
    
    MyOperation *operation1 = [[MyOperation alloc] init];
    operation1.num = 1;
    MyOperation *operation2 = [[MyOperation alloc] init];
    operation2.num = 2;
    [operationQueue addOperation:operation1];
    [operationQueue addOperation:operation2];
    
    NSLog(@"operation1 status isExecuting %d  isReady %d", operation1.isExecuting, operation1.isReady);
    NSLog(@"operation2 status isExecuting %d  isReady %d", operation2.isExecuting, operation2.isReady);
    
    [operationQueue cancelAllOperations];
}

```

log结果每次不一定一样，因为operation加入到queue中之后开始的时候不固定。但是每次start都是会调用的。

```bash
2013-10-29 20:11:38.571 AFNetworking iOS Example[6680:60b] operation1 status isExecuting 0  isReady 1
2013-10-29 20:11:38.574 AFNetworking iOS Example[6680:60b] operation2 status isExecuting 0  isReady 1
2013-10-29 20:11:38.574 AFNetworking iOS Example[6680:3707] ========call start 1
2013-10-29 20:11:38.576 AFNetworking iOS Example[6680:3707] ========call start 2
```

```bash
2013-10-29 20:13:16.777 AFNetworking iOS Example[6696:60b] operation1 status isExecuting 0  isReady 1
2013-10-29 20:13:16.777 AFNetworking iOS Example[6696:3707] ========call start 1
2013-10-29 20:13:16.782 AFNetworking iOS Example[6696:60b] operation2 status isExecuting 0  isReady 1
2013-10-29 20:13:16.783 AFNetworking iOS Example[6696:3707] current operation 1 begin
2013-10-29 20:13:21.788 AFNetworking iOS Example[6696:3707] current operation 1 end
2013-10-29 20:13:21.790 AFNetworking iOS Example[6696:3707] ========call start 2
```
```bash
2013-10-29 20:14:20.024 AFNetworking iOS Example[6701:4303] ========call start 1
2013-10-29 20:14:20.026 AFNetworking iOS Example[6701:4303] current operation 1 begin
2013-10-29 20:14:20.023 AFNetworking iOS Example[6701:60b] operation1 status isExecuting 0  isReady 1
2013-10-29 20:14:20.029 AFNetworking iOS Example[6701:60b] operation2 status isExecuting 0  isReady 1
2013-10-29 20:14:25.027 AFNetworking iOS Example[6701:4303] current operation 1 end
2013-10-29 20:14:25.029 AFNetworking iOS Example[6701:4303] ========call start 2
```
等等。

operation queue自己提供线程去运行operations。因此，operations总是在独立的线程中去执行，无论operation是否被指定为并发或非并发操作的。在下一节介绍NSOperation的时候在详细介绍并发operation和非并发operation的区别。

NSOperationQueue类是兼容KVC和KVO的，在开发过程中我们可以观察下列属性：
- operations，只读属性
- operationCount，只读
- maxConcurrentOperationCount，可读可写
- suspended，可读可写
- name，可读可写

尽管我们可以观察(observers)这些属性，但是有一点我们要记住，operation queue上连接的kvo notififcation 可能在任何线程发生，并不一定主线程。

在多个线程中访问同一个NSOperationQueue对象时安全的，不需要创建多余的locks去同步访问。

一个operation queue执行其排队的operations主要是基于operations的优先级和是否准备就绪(readiness)，如果所有排队的operations有相同的优先级，并且都准备就绪准备执行（operation的isReady方法返回YES），则operations的执行顺序按照其添加到operation queue中的顺序。对于一个maxConcurrentOperationCount设置为1的operation queue，其相当于一个线性queue。但是，我们永远不应该依赖operation对象的串行执行,改变operation是否准备就绪(readiness)会改变operations的执行顺序。

最后讲一下NSOperaion的suspended方法：
```objective-c
- (void)setSuspended:(BOOL)suspend
```
这个方法会暂停或者恢复operations的执行。暂停一个queue会阻止queue开始(start)其他的operations。换句话说，queue中尚未执行的operations，以及后续添加到queue中的operations都会被阻止去开始(start)运行，直到queue恢复。暂停一个queue不会去停止已经在运行(running)的operations。
只有在operation结束(finish)的时候，operation才会被从queue中移除。然而，一个operation想要finish executing，其首先要start。因此，因为一个暂停的queue不会开始(start)任何新的operations，所以他也不会删除或者取消任何正在排队但是没有执行的operations。


##2.NSOperation

这一节主要详细讲NSOperation。NSOperation是一个抽象类，可以使用其分装代码和数据并和一个task相关联。因为NSOperation是抽象类，因此我们不能直接使用它，通常是子类或者使用系统定义的子类(NSInvocationOperation 或者 NSBlockOperation)去进行任务。尽管NSOperation是抽象的，但是NSOperation的实现包裹了一些重要的逻辑，用于协调任务安全的执行，内置逻辑的存在主要是让我们集中的关心实际任务的实现，而不用写一些代码确保其正确的执行。

operation对象时一个单次（single-shot）对象，它执行任务一次，不能用来重复使用再执行一次。通常我们通过将其添加一个operation queue去执行。

如果不想使用operation queue，在代码中我们可以直接调用operation的start方法去执行，但是手动的执行一个operations会给代码增加负担，因为start一个没有处于ready状态的operation会引发一个异常。

dependencies是一个方便的方式用于按特定的顺序执行operations。可以通过operation的方法`addDependency:`和`removeDependency:`来给operation添加或者删除依赖。缺省情况下，一个有dependencies的operation不会是ready的，直到它依赖的所有operations finish。一旦最后一个dependent的opertaion完成，operaion会变成ready，可以去执行。

NSOperation的dependencies是不会区分dependent operation是成功还是失败的finish的（换句话，cancel一个operation也会把其标记为finish的）。当dependent operation被取消或者没有成功的完成task的时候，这依赖于我们去决定dependencies operation是否应该去执行。这可能需要我们在operation 中包含一些额外的错误跟踪能力。

NSOperation类的一些属性也是KVO和KVC兼容的。有如下一些属性：
- isCancelled 只读
- isConcurrent 只读
- isExecuting 只读
- isFinished  只读
- isReady 只读
- dependencies 只读
- queuePriority 可读可写
- completionBlock 可读可写

operation的KVO通知可能在任何一个线程中，所以跟UI相关的操作记得要判断。如果提供上述属性的自定义实现，则在实现中必须保持KVO和KVC兼容。如果在子类NSOperaion中定义了多余的属性，则推荐这些属性也满足KVO和KVC。

NSOperation类是支持多线程的，因此不必创建锁去同步对operation的访问，在多个线程中直接调用NSOperation的方法是安全的。这种行为是必要的，因为通常operation运行在一个单独的线程,和创建并监视它的线程不同。

当子类一个NSOperation的时候，我们要确保我们重写的方法在多个线程中访问时仍然是安全的。如果是自定义方法，例如像一些custom data accessors，我们也必须确保这些方法是线程安全的，因此在operation中访问任何数据变量都需要同步，确保安全。

如果我们计划手动的执行一个operation，而不是添加到一个queue中，则我们可以设计此operation是并发还是非并发的方式执行。一个operation对象缺省情况下是非并发的。一个非并发的operation，其task是同步进行的，operation不会创建一个单独的线程去运行task。因此当你在代码中直接调用non-concurrent operation的start方法，此operation会立即在当前线程中执行。当operation的start调用完成，返回给调用者的时候，operation的task已经完成。

而并发operation(concurrent operation)是异步运行的。当调用一个并发operation的start方法之后，此方法会在task完成之前返回。这可能是因为operation对象创建了一个新的线程去执行task，或者operation对象调用了一个异步函数。但实际上是怎么的并不重要。

如果我们计划是将operations加入到queue中去执行的话，则简单的定义这些operations为non-concurrent就可以。如果打算手动的执行operation的时候，则我们可能需要定义operation对象为concurrent的，来保证其异步的执行。定义一个concurrent operation需要更多的工作，因为我们需要监视task进行的状态，报告其变化。

在OS X v10.6中，operation queue会忽略operation对象isConcurrent的返回值，总是在一个单独的线程中调用operation对象的start方法。然而，在OSX v10.5中，只有operation对象isConcurrent方法返回NO时，operation queue才为其创建一个单独的线程。但总体上来说，如果我们总是在operation queue中使用operations的时候，不必将其标记为concurrent。

NSOperation类提供了基本的逻辑用于追踪operation的执行状态，但是需要子类化去定义要完成的任务。如何子类化取决去operation设计是用来并发还是非并发。

对于非并发operation，我们正常情况下只需要重写方法
```objective-c
- (void)main
```
当然我们也可以定义自定义的初始化方法，可以方便的创建自定义opertaion。我们也应该定义访问operation中自定义data的setter和getter方法，以便其在多线程访问中安全。

如果需要创建concurrent operation，则至少需要重写下面方法
```objective-c
- (void)start
- (BOOL)isConcurrent
- (BOOL)isExecuting
- (BOOL)isFinished
```
在concurrent operation中，start方法有责任保证一个operation以一种异步的方式start，在此方法中你可以生成一个线程，或者调用异步函数。在start方法中我们应该更新operation的执行状态，并通过isExecuting key path发送KVO通知，让感兴趣的知道operation已经running。

在AFNetworking开源库中，AFURLConnectionOperaion就是一个concurrent operation，我们以此为例子说明。

下面是摘抄里面的部分代码：
```objective-c

static inline NSString * AFKeyPathFromOperationState(AFOperationState state) {
    switch (state) {
        case AFOperationReadyState:
            return @"isReady";
        case AFOperationExecutingState:
            return @"isExecuting";
        case AFOperationFinishedState:
            return @"isFinished";
        case AFOperationPausedState:
            return @"isPaused";
        default:
            return @"state";
    }
}

- (void)setState:(AFOperationState)state {
    if (!AFStateTransitionIsValid(self.state, state, [self isCancelled])) {
        return;
    }
    
    [self.lock lock];
    NSString *oldStateKey = AFKeyPathFromOperationState(self.state);
    NSString *newStateKey = AFKeyPathFromOperationState(state);
    
    [self willChangeValueForKey:newStateKey];
    [self willChangeValueForKey:oldStateKey];
    _state = state;
    [self didChangeValueForKey:oldStateKey];
    [self didChangeValueForKey:newStateKey];
    [self.lock unlock];
}

- (void)start {
    [self.lock lock];
    if ([self isReady]) {
        self.state = AFOperationExecutingState;
        
        [self performSelector:@selector(operationDidStart) onThread:[[self class] networkRequestThread] withObject:nil waitUntilDone:NO modes:[self.runLoopModes allObjects]];
    }
    [self.lock unlock];
}

```

在重写的start方法中，我们可以看到，首先判断状态，当状态为isReady，则改变state为isExecuting，并调用`willChangeValueForKey:`和`didChangeValueForKey`。

当任务完成或者取消的时候，concurrent operation必须对isExecuting和isFinished进行KVO通知（当取消的时候，仍然要更新isFinished，进行KVO通知，因为放在queue中的operation在被从queue中移除之前，必须报告其finished）。除了生成KVO通知，我们也应该重写isExecuting和isFinished方法，以便返回正确的状态。

```objective-c
- (BOOL)isExecuting {
    return self.state == AFOperationExecutingState;
}

- (BOOL)isFinished {
    return self.state == AFOperationFinishedState;
}

- (BOOL)isConcurrent {
    return YES;
}
```
**注意：**一定不要在自己的start方法中去调用super start。当我们定义一个concurrent operation的时候，我们有责任提供一个和默认start方法相同行为start方法，其中包括开始task，生成恰当的KVO通知。我们的start方法在开始执行task的时候，应该检查operation是否被取消。

对于concurrent operation，你覆盖上面的方法就可以了。但是如果你自定义operation的dependency功能，则需要覆盖更多的方法来提供KVO通知。对于dependency这种情况，我们要提供isReady的KVO通知。

operation对象内部维持了自己的状态信息，来确定其安全的执行，并通过其生命周期通知外部进展。当自定义子类的时候，我们必须同样要维持状态信息，确保operation正确的执行。下面将详细讲述我们在自定义operation子类的时候应该管理的key path（KVO通知 key）。

 (1) isReady :isReady key path让外部知道一个operation什么时间准备执行。当isReady方法返回YES的时候，表示这个operation现在已经准备好去执行，如果其依赖的operations有没有完成的，则返回NO。在大多数情况下，我们不需要管理此key path的状态，如果operation的准备状态除了依赖dependent operations，还依赖一些外部条件，则我们需要自己实现isReady方法，自己跟踪operation的ready状态。但如果在外部条件满足的时候在创建operation，往往是比较简单的。在OS X v10.6开始，当取消一个正在等待依赖operations完成的operation，它的这些依赖会被忽略掉，isReady属性也会被更新，表明其准备运行，这种行为给一个operation queue迅速清理掉cancel operations的机会。
(2) isExecuting :isExecuting key path 让外部知道一个operation是否正在进行。当isExecuting方法返回YES的时候，表示operation的task正在进行，当返回NO的时候，表示没有。当operation重写了start方法之后，我们也必须重写isExecuting方法，并在operation 执行状态发生改变的时候生成KVO通知。
(3) isFinished :isFinished key path 让外部知道此operation是成功的完成了task，或者是取消并退出的。一个operation queue 直到一个operation的isFinished方法返回YES的时候，才让其出队列。因此，标记一个operation为finished是很重要的。
(4) isCancelled :isCancelled key path 让外部直到operation被请求取消。是否支持取消是自愿的，但是鼓励这样做。我们的代码不应该去发送此种KVO通知。


一旦一个operation添加到operation queue中之后，此operation已经不再需要你去控制了。queue会全权帮你控制并计划运行此operation的task。但是你突然决定不再想去执行此operation，你可以去取消此operation，通过调用此operation的cancel方法，或者调用NSOpertaionQueue类的cancelAllOperations方法。

取消一个operation对象不会立即强制其停止。我们的代码中需要明确的检查isCancelled方法的返回值来决定是否中断。NSOperation缺省的实现已经包含了取消状态的检查。例如，当在一个operation的start方法调用之前取消，则start方法不去执行task，直接退出。

应对cancel最恰当的做法是在main中，如果遇到耗时的操作的时候，首先调用isCancelled方法，检查是否被取消。
