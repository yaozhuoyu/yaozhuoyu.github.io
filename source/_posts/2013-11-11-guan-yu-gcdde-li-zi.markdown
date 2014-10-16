---
layout: post
title: "关于GCD的例子"
date: 2013-11-11 21:21
comments: true
categories: iOS
tags: GCD
---

在iOS开发中，相信很多人都用过gcd，关于gcd的概念以及具体用法网上有很多可以看的资料，方便学习。但是，gcd中包含很多函数，其中有一些我们不常见的函数、概念，本篇文章主要对这一部分内容做分析。
<!--more-->

##1.为Queue提供一个final处理函数
当我们创建一个串行Queue的时候，有些时候想在queue释放的时候，做一些处理，比如分配内存的释放等等，这时就需要给Queue设置一个finalizer函数，通过方法dispatch_set_finalizer_f设置，下面是一个例子：
```objective-c
typedef struct MyDataContext{
    int length;
    char *data;
}MyDataContext;

void myFinalizerFunction(void *context)
{
    MyDataContext* theData = (MyDataContext*)context;
    // Clean up the contents of the structure
    
    // Now release the structure itself.
    free(theData);
    NSLog(@"======= finalizer function");
}

@implementation FirstViewController

- (void)viewDidLoad
{
    [super viewDidLoad];
    MyDataContext*  data = (MyDataContext*) malloc(sizeof(MyDataContext));
    dispatch_queue_t myQueue = dispatch_queue_create("my.queue", NULL);
    
    if (myQueue) {
        dispatch_set_context(myQueue, data);
        dispatch_set_finalizer_f(myQueue, &myFinalizerFunction);
    }
    
}

@end
```
因为创建的Queue作用域在viewDidLoad中，当函数结束的时候，queue会被释放，调用函数myFinalizerFunction，运行结果为：
```bash
2013-11-11 21:39:08.039 podProject[4021:1803] ======= finalizer function
```

##2.disptach_apply
```c
void dispatch_apply(
   size_t iterations,
   dispatch_queue_t queue,
   void (^block)(
   size_t));
```
函数dispatch_apply，可以将一个for循环改造成并发运行，当所有的block运行完成之后，此函数才会返回。一般传入的queue都是并发的queue，传入串行queue就失去意义了。下面是一个例子：
```objective-c
- (void)viewDidLoad
{
    [super viewDidLoad];
    dispatch_queue_t currentQueue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
    
    dispatch_apply(5, currentQueue, ^(size_t index){
        int result = 0;
        for (int i = 0; i < 100; i++) {
            result += i + index;
        }
        NSLog(@"current index %zu result %d", index, result);
    });
    NSLog(@"=========end");
}
```
结果为：
```bash
2013-11-11 21:53:19.199 podProject[4077:60b] current index 0 result 4950
2013-11-11 21:53:19.202 podProject[4077:3707] current index 1 result 5050
2013-11-11 21:53:19.203 podProject[4077:3707] current index 3 result 5250
2013-11-11 21:53:19.203 podProject[4077:60b] current index 2 result 5150
2013-11-11 21:53:19.206 podProject[4077:3707] current index 4 result 5350
2013-11-11 21:53:19.208 podProject[4077:60b] =========end
```
使用起来非常简单，但是要注意一种情况，dispatch_apply函数中传入的queue一定不能和dispatch_apply运行的queue为同一个queue，否则会死锁。下面的例子注定死锁。
```objective-c
- (void)viewDidLoad
{
    [super viewDidLoad];
    dispatch_apply(5, dispatch_get_main_queue(), ^(size_t index){
        int result = 0;
        for (int i = 0; i < 100; i++) {
            result += i + index;
        }
        NSLog(@"current index %zu result %d", index, result);
    });
    NSLog(@"=========end");
}
```

##3.使用dispatch semaphore
dispatch semaphore和系统的semaphore相似，但是比系统的更有效，因为dispatch semaphore只会在资源数量不够用的时候才去调用系统内核处理。

下面是一个简单的例子：
```objective-c
- (void)viewDidLoad
{
    [super viewDidLoad];
    dispatch_queue_t queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
    dispatch_semaphore_t semaphore = dispatch_semaphore_create(1);
    
    NSMutableArray *array = [[NSMutableArray alloc] init];
    for (int i = 0; i < 10; ++i) {
        dispatch_async(queue, ^{
            dispatch_semaphore_wait(semaphore, DISPATCH_TIME_FOREVER);
            [array addObject:[NSNumber numberWithInt:i]];
            dispatch_semaphore_signal(semaphore);
            NSLog(@"current %d", i);
        });
    }
    NSLog(@"viewdidLoad end.......");
}
```
运行结果为：
```bash
2013-11-12 10:38:18.964 podProject[4591:60b] viewdidLoad end.......
2013-11-12 10:38:18.965 podProject[4591:3707] current 1
2013-11-12 10:38:18.966 podProject[4591:3707] current 2
2013-11-12 10:38:18.965 podProject[4591:1803] current 0
2013-11-12 10:38:18.971 podProject[4591:3707] current 3
2013-11-12 10:38:18.971 podProject[4591:1803] current 4
2013-11-12 10:38:18.974 podProject[4591:3d03] current 5
2013-11-12 10:38:18.976 podProject[4591:3707] current 6
2013-11-12 10:38:18.977 podProject[4591:3e03] current 7
2013-11-12 10:38:18.977 podProject[4591:1803] current 8
2013-11-12 10:38:18.978 podProject[4591:3d03] current 9
```

##4.使用dispatch group
使用disptach group可以实现监听一组任务完成，在一组任务完成之后在通知执行其他操作。其中经常使用到的函数有：
```objective-c
void dispatch_group_async(
   dispatch_group_t group,
   dispatch_queue_t queue,
   dispatch_block_t block);
```
dispatch_group_async函数会提交block到指定的queue，并使其和group连接起来。

```objective-c
void dispatch_group_notify(
   dispatch_group_t group,
   dispatch_queue_t queue,
   dispatch_block_t block);
```
dispatch_group_notify函数会提交一个block到指定的queue，只有在先前提交到group的blocks都执行完成之后，此block还会执行。

看一个例子：
```objective-c
- (void)viewDidLoad
{
    [super viewDidLoad];
    dispatch_queue_t queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
    
    dispatch_group_t group = dispatch_group_create();
    
    NSLog(@"add block 1");
    dispatch_group_async(group, queue, ^{
        NSLog(@"block 1 start sleep");
        sleep(1);
        NSLog(@"block 1 finish sleep");
    });
    
    NSLog(@"add block 2");
    dispatch_group_async(group, queue, ^{
        NSLog(@"block 2 start sleep");
        sleep(2);
        NSLog(@"block 2 finish sleep");
    });
    
    NSLog(@"add block 3");
    dispatch_group_async(group, queue, ^{
        NSLog(@"block 3 start sleep");
        sleep(1);
        NSLog(@"block 3 finish sleep");
    });
    
    NSLog(@"add block notify");
    dispatch_group_notify(group, queue, ^{
        NSLog(@"dispatch group notify exe");
    });
    
    NSLog(@"viewdidLoad end.......");
}
```
结果如下：
```bash
2013-11-12 11:05:25.359 podProject[4660:60b] add block 1
2013-11-12 11:05:25.361 podProject[4660:1803] block 1 start sleep
2013-11-12 11:05:25.361 podProject[4660:60b] add block 2
2013-11-12 11:05:25.364 podProject[4660:60b] add block 3
2013-11-12 11:05:25.364 podProject[4660:3707] block 2 start sleep
2013-11-12 11:05:25.365 podProject[4660:60b] add block notify
2013-11-12 11:05:25.366 podProject[4660:3c03] block 3 start sleep
2013-11-12 11:05:25.367 podProject[4660:60b] viewdidLoad end.......
2013-11-12 11:05:26.364 podProject[4660:1803] block 1 finish sleep
2013-11-12 11:05:26.369 podProject[4660:3c03] block 3 finish sleep
2013-11-12 11:05:27.367 podProject[4660:3707] block 2 finish sleep
2013-11-12 11:05:27.369 podProject[4660:3707] dispatch group notify exe
```

看另外一个函数
```objective-c
long dispatch_group_wait(
   dispatch_group_t group,
   dispatch_time_t timeout);
```
dispatch_group_wait函数会等待先前和group相关联的block执行完成，或者超时返回。下面是一个例子：
```objective-c
- (void)viewDidLoad
{
    [super viewDidLoad];
    dispatch_queue_t queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
    
    dispatch_group_t group = dispatch_group_create();
    
    NSLog(@"add block 1");
    dispatch_group_async(group, queue, ^{
        NSLog(@"block 1 start sleep");
        sleep(1);
        NSLog(@"block 1 finish sleep");
    });
    
    NSLog(@"add block 2");
    dispatch_group_async(group, queue, ^{
        NSLog(@"block 2 start sleep");
        sleep(2);
        NSLog(@"block 2 finish sleep");
    });
    
    NSLog(@"add group wait");
    dispatch_group_wait(group, DISPATCH_TIME_FOREVER);
    
    NSLog(@"viewdidLoad end.......");
}
```
结果为：
```bash
2013-11-12 11:20:57.957 podProject[4725:60b] add block 1
2013-11-12 11:20:57.959 podProject[4725:60b] add block 2
2013-11-12 11:20:57.961 podProject[4725:60b] add group wait
2013-11-12 11:20:57.961 podProject[4725:3a03] block 2 start sleep
2013-11-12 11:20:57.959 podProject[4725:1803] block 1 start sleep
2013-11-12 11:20:58.963 podProject[4725:1803] block 1 finish sleep
2013-11-12 11:20:59.963 podProject[4725:3a03] block 2 finish sleep
2013-11-12 11:20:59.965 podProject[4725:60b] viewdidLoad end.......
```
可以看出，dispatch_group_wait会阻塞当前线程。

最后看两个函数
```objective-c
void dispatch_group_enter(
   dispatch_group_t group);
void dispatch_group_leave(
   dispatch_group_t group);
```
这两个函数是成对调用的，dispatch_group_enter函数表示一个block进入group，调用此函数会增加group中task的数量。dispatch_group_leave则相反。
dispatch_group_async函数其实就等价于如下实现：
```objective-c
void
     dispatch_group_async(dispatch_group_t group, dispatch_queue_t queue, dispatch_block_t block)
     {
             dispatch_retain(group);
             dispatch_group_enter(group);
             dispatch_async(queue, ^{
                     block();
                     dispatch_group_leave(group);
                     dispatch_release(group);
             });
     }
```
使用dispatch_group_enter函数和dispatch_group_leave函数可以方便的把系统提供的异步函数加入到一个group中。例如对于NSURLConnection的函数
```objective-c
+ (void)sendAsynchronousRequest:(NSURLRequest *)request 
        queue:(NSOperationQueue *)queue 
        completionHandler:(void (^)(NSURLResponse*, NSData*, NSError*))handler
```
我们可以在写一个能将其加入group的函数
```objective-c
+ (void)withGroup:(dispatch_group_t)group 
        sendAsynchronousRequest:(NSURLRequest *)request 
        queue:(NSOperationQueue *)queue 
        completionHandler:(void (^)(NSURLResponse*, NSData*, NSError*))handler
{
    if (group == NULL) {
        [self sendAsynchronousRequest:request 
                                queue:queue 
                    completionHandler:handler];
    } else {
        dispatch_group_enter(group);
        [self sendAsynchronousRequest:request 
                                queue:queue 
                    completionHandler:^(NSURLResponse *response, NSData *data, NSError *error){
            handler(response, data, error);
            dispatch_group_leave(group);
        }];
    }
}
```

##5.dispatch_after
```objective-c
void dispatch_after(
   dispatch_time_t when,
   dispatch_queue_t queue,
   dispatch_block_t block);
```
dispatch_after函数实现延迟执行某动作，时间并不是很精确，实际上是过多久将Block追加到Queue中，而不是执行该动作，如果此时queue中的任务很多，没有执行完毕，那么新添加的这个动作就要继续推迟。

看下面的例子：
```objective-c
- (void)viewDidLoad
{
    [super viewDidLoad];
    
    NSLog(@"sleep start");
    dispatch_async(dispatch_get_main_queue(), ^{
        sleep(10);
        NSLog(@"sleep 10s finish");
    });
    
    NSLog(@"start dispatch after");
    dispatch_time_t delayTime = dispatch_time(DISPATCH_TIME_NOW, 5ull * NSEC_PER_SEC);
    dispatch_after(delayTime, dispatch_get_main_queue(), ^{
        NSLog(@"after 5s , execute!");
    });
    
    NSLog(@"end viewdidload");
}
```
执行结果为：
```bash
2013-11-12 12:23:36.113 podProject[4783:60b] sleep start
2013-11-12 12:23:36.115 podProject[4783:60b] start dispatch after
2013-11-12 12:23:36.116 podProject[4783:60b] end viewdidload
2013-11-12 12:23:46.128 podProject[4783:60b] sleep 10s finish
2013-11-12 12:23:46.131 podProject[4783:60b] after 5s , execute!
```
因为queue sleep了10秒，所以造成了这样的结果。

##6.dispatch_set_target_queue
看函数
```objective-c
void dispatch_set_target_queue(
   dispatch_object_t object,
   dispatch_queue_t queue);
```
 
所有的用户队列都有一个目标队列概念。从本质上讲，一个用户队列实际上是不执行任何任务的，但是它会将任务传递给它的目标队列来执行。通常，目标队列是默认优先级的全局队列。

用户队列的目标队列可以用函数 dispatch_set_target_queue来修改。我们可以将任意dispatch queue传递给这个函数，甚至可以是另一个用户队列，只要别构成循环就行。这个函数可以用来设定用户队列的优先级。比如我们可以将用户队列的目标队列设定为低优先级的全局队列，那么我们的用户队列中的任务都会以低优先级执行。高优先级也是一样道理。

有一个用途，是将用户队列的目标定为main queue。这会导致所有提交到该用户队列的block在主线程中执行。这样做来替代直接在主线程中执行代码的好处在于，我们的用户队列可以单独地被挂起和恢复，还可以被重定目标至一个全局队列，然后所有的block会变成在全局队列上执行（只要你确保你的代码离开主线程不会有问题）。

还有一个用途，是将一个用户队列的目标队列指定为另一个用户队列。这样做可以强制多个队列相互协调地串行执行，这样足以构建一组队列，通过挂起和暂停那个目标队列，我们可以挂起和暂停整个组。

下面一个例子，将一个创建的串行queue的目标队列设置为mian queue。
```objective-c
- (void)viewDidLoad
{
    [super viewDidLoad];
    NSLog(@"========current thread %@", [NSThread currentThread]);
    dispatch_queue_t myQueue = dispatch_queue_create("my queue", NULL);
    dispatch_set_target_queue(myQueue, dispatch_get_main_queue());
    dispatch_async(myQueue, ^{
        NSLog(@"*******current thread %@", [NSThread currentThread]);
    });
    NSLog(@"end viewdidload");
}
```
运行结果：
```bash
2013-11-12 14:42:00.509 podProject[5158:60b] ========current thread <NSThread: 0x16526f60>{name = (null), num = 1}
2013-11-12 14:42:00.512 podProject[5158:60b] end viewdidload
2013-11-12 14:42:00.524 podProject[5158:60b] *******current thread <NSThread: 0x16526f60>{name = (null), num = 1}
```

##7.dispatch barriers
dispatch barriers和dispatch group很相似。dispatch barriers主要用在并发queue的同步中。用来阻拦之后添加的block，在barriers之前天机的block执行完成之后，才会执行barriers之后添加的block。

看下面的一个例子：
```objective-c
- (void)viewDidLoad
{
    [super viewDidLoad];
    
    dispatch_queue_t myQueue = dispatch_queue_create("my concurrent queue", DISPATCH_QUEUE_CONCURRENT);
    
    dispatch_async(myQueue, ^{
        NSLog(@"block 1");
    });
    dispatch_async(myQueue, ^{
        NSLog(@"block 2");
    });
    dispatch_async(myQueue, ^{
        NSLog(@"block 3");
    });
    dispatch_barrier_async(myQueue, ^{
        NSLog(@"***barrier***");
    });
    dispatch_async(myQueue, ^{
        NSLog(@"block 4");
    });
    dispatch_async(myQueue, ^{
        NSLog(@"block 5");
    });
    NSLog(@"viewDidLoad finish");
}
```
运行结果为：

```bash
2013-11-12 15:39:16.545 podProject[5296:60b] viewDidLoad finish
2013-11-12 15:39:16.545 podProject[5296:1803] block 1
2013-11-12 15:39:16.549 podProject[5296:1803] block 3
2013-11-12 15:39:16.545 podProject[5296:370b] block 2
2013-11-12 15:39:16.550 podProject[5296:370b] ***barrier***
2013-11-12 15:39:16.555 podProject[5296:370b] block 4
2013-11-12 15:39:16.555 podProject[5296:1803] block 5
```

如果将dispatch_barrier_async修改为dispatch_barrier_sync，则运行结果就变为如下：
```bash
2013-11-12 15:41:25.336 podProject[5308:1803] block 1
2013-11-12 15:41:25.336 podProject[5308:3a03] block 2
2013-11-12 15:41:25.340 podProject[5308:1803] block 3
2013-11-12 15:41:25.342 podProject[5308:60b] ***barrier***
2013-11-12 15:41:25.343 podProject[5308:60b] viewDidLoad finish
2013-11-12 15:41:25.343 podProject[5308:3a03] block 5
2013-11-12 15:41:25.343 podProject[5308:1803] block 4
```
使用起来比dispatch group简单。

关于dispatch的使用先介绍这7个，下篇文章将讲一下dispatch I/O。
