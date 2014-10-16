---
layout: post
title: "关于NSRunLoop的几个例子"
date: 2013-11-05 13:09
comments: true
categories: iOS
tags: NSRunLoop
---
关于NSRunLoop，在一般的开发中我们很少用到，相对底层的类。网上介绍NSRunLoop的文章也很多，但是大部分都是摘抄和翻译官网文档的，晦涩难懂，并且也没有例子。所以本篇文章主要是以例子为基础，详细介绍NSRunLoop的每一个细节，关于NSRunLoop的基本概念在此就不再讨论了，如果不了解的可以参考其他的文章，或者官方文档[链接](https://developer.apple.com/library/mac/documentation/cocoa/Conceptual/Multithreading/RunLoopManagement/RunLoopManagement.html)。
<!--more-->

##1.NSRunLoop添加observer
在一个runloop的执行过程中，我们可以给runloop添加一个oberver来接收runloop状态变化的通知。
当runloop运行到下面6个状态的时候，oberver会收到通知：
(1)The entrance to the run loop.
(2)When the run loop is about to process a timer.
(3)When the run loop is about to process an input source.
(4)When the run loop is about to go to sleep.
(5)When the run loop has woken up, but before it has processed the event that woke it up.
(6)The exit from the run loop.

```objective-c
/* Run Loop Observer Activities */
typedef CF_OPTIONS(CFOptionFlags, CFRunLoopActivity) {
    kCFRunLoopEntry = (1UL << 0),
    kCFRunLoopBeforeTimers = (1UL << 1),
    kCFRunLoopBeforeSources = (1UL << 2),
    kCFRunLoopBeforeWaiting = (1UL << 5),
    kCFRunLoopAfterWaiting = (1UL << 6),
    kCFRunLoopExit = (1UL << 7),
    kCFRunLoopAllActivities = 0x0FFFFFFFU
};
```
上面的枚举类型定义分别对应上面的六种状态。
写一个小例子，说明什么时候是什么状态，例子如下：
```objective-c
- (void)viewDidLoad
{
    [super viewDidLoad];
    [self createRunLoopThread];
}

- (void)createRunLoopThread{
    NSThread *runLoopThread = [[NSThread alloc] initWithTarget:self selector:@selector(threadStart:) object:nil];
    [runLoopThread start];
}

- (void)threadStart:(id)object{
    @autoreleasepool {
        NSRunLoop *myRunLoop = [NSRunLoop currentRunLoop];
        
        //create runloop observer
        CFRunLoopObserverContext context = {0, NULL, NULL, NULL};
        CFRunLoopObserverRef observer = CFRunLoopObserverCreate(kCFAllocatorDefault,
                                                                kCFRunLoopAllActivities,
                                                                YES,
                                                                0,
                                                                &myRunLoopObserver,
                                                                &context);
        if (observer) {
            CFRunLoopRef cfLoop = [myRunLoop getCFRunLoop];
            CFRunLoopAddObserver(cfLoop, observer, kCFRunLoopDefaultMode);
        }
        
        [NSTimer scheduledTimerWithTimeInterval:5 target:self
                                       selector:@selector(doFireTimer:) userInfo:nil repeats:YES];
        
        NSInteger loopCount = 2;
        do
        {
            [myRunLoop runUntilDate:[NSDate dateWithTimeIntervalSinceNow:10]];
            NSLog(@"current loop count ======== %d", loopCount);
            loopCount--;
        }
        while (loopCount);
        
        if (observer) {
            CFRelease(observer);
        }
        NSLog(@"thread end=================");
    }
}

- (void)doFireTimer:(id)object{
    NSLog(@"************timer doFireTimer");
}
```
上面代码首先创建一个thread，此线程的方法为threadStart:，在此方法中，我们给此线程的runloop添加了一个observer，observer的处理方法为myRunLoopObserver。然后添加一个timer，最后使用方法runUntilDate启动runloop。
NSRunLoop的方法runUntilDate:会在在指定的时间内重复的调用runMode:beforeDate:，直到到达过期时间。
运行打印结果如下：
```bash
2013-11-05 14:23:18.353 podProject[12480:370b] my run loop observer  1
2013-11-05 14:23:18.356 podProject[12480:370b] my run loop observer  2
2013-11-05 14:23:18.358 podProject[12480:370b] my run loop observer  4
2013-11-05 14:23:18.359 podProject[12480:370b] my run loop observer  32
2013-11-05 14:23:23.354 podProject[12480:370b] my run loop observer  64
2013-11-05 14:23:23.356 podProject[12480:370b] ************timer doFireTimer
2013-11-05 14:23:23.359 podProject[12480:370b] my run loop observer  2
2013-11-05 14:23:23.360 podProject[12480:370b] my run loop observer  4
2013-11-05 14:23:23.361 podProject[12480:370b] my run loop observer  32
2013-11-05 14:23:28.354 podProject[12480:370b] my run loop observer  64
2013-11-05 14:23:28.357 podProject[12480:370b] ************timer doFireTimer
2013-11-05 14:23:28.359 podProject[12480:370b] my run loop observer  128
2013-11-05 14:23:28.360 podProject[12480:370b] current loop count ======== 2
2013-11-05 14:23:28.361 podProject[12480:370b] my run loop observer  1
2013-11-05 14:23:28.363 podProject[12480:370b] my run loop observer  2
2013-11-05 14:23:28.364 podProject[12480:370b] my run loop observer  4
2013-11-05 14:23:28.364 podProject[12480:370b] my run loop observer  32
2013-11-05 14:23:33.354 podProject[12480:370b] my run loop observer  64
2013-11-05 14:23:33.357 podProject[12480:370b] ************timer doFireTimer
2013-11-05 14:23:33.359 podProject[12480:370b] my run loop observer  2
2013-11-05 14:23:33.360 podProject[12480:370b] my run loop observer  4
2013-11-05 14:23:33.361 podProject[12480:370b] my run loop observer  32
2013-11-05 14:23:38.354 podProject[12480:370b] my run loop observer  64
2013-11-05 14:23:38.356 podProject[12480:370b] ************timer doFireTimer
2013-11-05 14:23:38.359 podProject[12480:370b] my run loop observer  2
2013-11-05 14:23:38.360 podProject[12480:370b] my run loop observer  4
2013-11-05 14:23:38.361 podProject[12480:370b] my run loop observer  32
2013-11-05 14:23:38.364 podProject[12480:370b] my run loop observer  64
2013-11-05 14:23:38.366 podProject[12480:370b] my run loop observer  128
2013-11-05 14:23:38.368 podProject[12480:370b] current loop count ======== 1
2013-11-05 14:23:38.369 podProject[12480:370b] thread end=================
```
其中数字1，2，4，32，64，128分别对应CFRunLoopActivity的定义。

##2.停止runloop

我们知道，在线程中启动runloop的方法有：
```objective-c
- (void)run
- (BOOL)runMode:(NSString *)mode beforeDate:(NSDate *)limitDate
- (void)runUntilDate:(NSDate *)limitDate
```
CF中启动runloop的方法为：
```c
void CFRunLoopRun (
   void
);
SInt32 CFRunLoopRunInMode (
   CFStringRef mode,
   CFTimeInterval seconds,
   Boolean returnAfterSourceHandled
);
```
如果我们使用run方法去启动一个runloop是没有办法去停止此runloop的。因此如果想要一个runloop停止，不要使用此方法启动runloop。
而方法runMode:beforeDate:和runUntilDate:启动的runloop，会在指定时间到的时候或者有输入源事件发生时自动stop。

使用方法CFRunLoopRun和CFRunLoopRunInMode启动的runloop可以使用方法CFRunLoopStop停止。

看下面的例子：
```objective-c
- (void)viewDidLoad
{
    [super viewDidLoad];
    [self createRunLoopThread];
}

- (void)createRunLoopThread{
    NSThread *runLoopThread = [[NSThread alloc] initWithTarget:self selector:@selector(threadStart:) object:nil];
    [runLoopThread start];
}

- (void)threadStart:(id)object{
    @autoreleasepool {
        NSRunLoop *myRunLoop = [NSRunLoop currentRunLoop];
        [NSTimer scheduledTimerWithTimeInterval:1 target:self
                                       selector:@selector(doFireTimer:) userInfo:nil repeats:YES];
        [myRunLoop runUntilDate:[NSDate dateWithTimeIntervalSinceNow:5]];
    }
}

- (void)doFireTimer:(id)object{
    NSRunLoop *myRunLoop = [NSRunLoop currentRunLoop];
    NSLog(@"************timer doFireTimer");
    CFRunLoopStop([myRunLoop getCFRunLoop]);
}
```
运行结果为：
```bash
2013-11-05 16:09:38.115 podProject[12681:3807] ************timer doFireTimer
2013-11-05 16:09:39.114 podProject[12681:3807] ************timer doFireTimer
2013-11-05 16:09:40.115 podProject[12681:3807] ************timer doFireTimer
2013-11-05 16:09:41.115 podProject[12681:3807] ************timer doFireTimer
2013-11-05 16:09:42.115 podProject[12681:3807] ************timer doFireTimer
```
可以看到，通过runUntilDate启动的runloop是没有方法通过方法CFRunLoopStop停止的。
如果将threadStart:改为如下：
```objective-c
- (void)threadStart:(id)object{
    @autoreleasepool {
        [NSTimer scheduledTimerWithTimeInterval:1 target:self
                                       selector:@selector(doFireTimer:) userInfo:nil repeats:YES];
        CFRunLoopRun();
    }
}
```
运行结果为：
```bash
2013-11-05 16:12:06.699 podProject[12694:3807] ************timer doFireTimer
```
通过CFRunLoopRun启动的runloop，可以直接通过CFRunLoopStop停止。

##3.创建并添加自定义Input Source
在一些场合下，我们需要创建添加自定义输入源，很简单，看下面代码：
```objective-c
#import "FirstViewController.h"
#import "AppDelegate.h"

void RunLoopSourceScheduleRoutine (void *info, CFRunLoopRef rl, CFStringRef mode)
{
    FirstViewController *firstCon = (__bridge FirstViewController *)info;
    [firstCon sourceSchedule];
}

void RunLoopSourcePerformRoutine (void *info)
{
    FirstViewController *firstCon = (__bridge FirstViewController *)info;
    [firstCon sourceFired];
}

void RunLoopSourceCancelRoutine (void *info, CFRunLoopRef rl, CFStringRef mode)
{
    FirstViewController *firstCon = (__bridge FirstViewController *)info;
    [firstCon sourceCancel];
}

@interface FirstViewController (){
    CFRunLoopRef threadRunLoop;
    CFRunLoopSourceRef runLoopSource;
    NSTimer *testTimer;
}

@end

@implementation FirstViewController

- (void)sourceSchedule{
    NSLog(@"soucre schedule..........");
}

- (void)sourceFired{
    NSLog(@"soucre fired..........");
}

- (void)sourceCancel{
    NSLog(@"soucre cancel..........");
}

- (void)viewDidLoad
{
    [super viewDidLoad];
    [self createRunLoopThread];
    testTimer = [NSTimer scheduledTimerWithTimeInterval:3.0f target:self selector:@selector(timerStart:) userInfo:nil repeats:YES];
}

- (void)timerStart:(id)obj{
    static int flag = 0;
    flag ++;
    if (flag > 5) {
        [self removeSource];
    }else{
        [self fireSource];
    }
}

- (void)createRunLoopThread{
    NSThread *runLoopThread = [[NSThread alloc] initWithTarget:self selector:@selector(threadStart:) object:nil];
    [runLoopThread start];
}

- (void)threadStart:(id)object{
    @autoreleasepool {
        threadRunLoop = CFRunLoopGetCurrent();
        CFRunLoopSourceContext context = {
            0,//version
            (__bridge void *)(self),//info
            NULL,//retain
            NULL,//release
            NULL,//copyDescription
            NULL,//equal
            NULL,//hash
            &RunLoopSourceScheduleRoutine,//schedule
            &RunLoopSourceCancelRoutine,//cancel
            &RunLoopSourcePerformRoutine//perform
        };
        
        runLoopSource = CFRunLoopSourceCreate(NULL, 0, &context);
        CFRunLoopAddSource(threadRunLoop, runLoopSource, kCFRunLoopDefaultMode);
        CFRunLoopRun();
        
        CFRelease(runLoopSource);
        NSLog(@"thread end........");
    }
}

- (void)removeSource{
    CFRunLoopRemoveSource(threadRunLoop, runLoopSource, kCFRunLoopDefaultMode);
    CFRunLoopStop(threadRunLoop);
    [testTimer invalidate];
}

- (void)fireSource
{
    CFRunLoopSourceSignal(runLoopSource);
    CFRunLoopWakeUp(threadRunLoop);
}
@end
```

运行结果如下：
```bash
2013-11-05 21:08:12.348 podProject[13000:3807] soucre schedule..........
2013-11-05 21:08:15.349 podProject[13000:3807] soucre fired..........
2013-11-05 21:08:18.349 podProject[13000:3807] soucre fired..........
2013-11-05 21:08:21.349 podProject[13000:3807] soucre fired..........
2013-11-05 21:08:24.349 podProject[13000:3807] soucre fired..........
2013-11-05 21:08:27.349 podProject[13000:3807] soucre fired..........
2013-11-05 21:08:30.349 podProject[13000:60b] soucre cancel..........
2013-11-05 21:08:30.351 podProject[13000:3807] thread end........
```

