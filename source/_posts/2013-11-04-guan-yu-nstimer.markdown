---
layout: post
title: "关于NSTimer"
date: 2013-11-04 14:49
comments: true
categories: iOS
tags: NSTimer
---

NSTimer是我们经常使用的一个类，相信大家都使用过。但是NSTimer在使用的时候有很多小陷进，以及我们没有考虑到的问题，所以决定用此篇文章记录在使用中遇到的问题。
<!--more-->

##1. fire方法
```objective-c
- (void)fire
```
关于这个方法官方的解释是这样的：
You can use this method to fire a repeating timer without interrupting its regular firing schedule. If the timer is non-repeating, it is automatically invalidated after firing, even if its scheduled fire date has not arrived.

比较难懂，我们举个例子来说吧。对于repeat timer来讲，如果调用fire方法，会直接触发timer的action函数，但原来的按照指定计划时间的action不会受到影响。看一个小小例子：
```objective-c
- (void)viewDidLoad
{
    [super viewDidLoad];
    
    testTimer = [NSTimer timerWithTimeInterval:5.0f target:self selector:@selector(timerAction:) userInfo:nil repeats:YES];
    NSRunLoop *runloop=[NSRunLoop currentRunLoop];
    [runloop addTimer:testTimer forMode:NSDefaultRunLoopMode];
    
    UIButton *btn = [UIButton buttonWithType:UIButtonTypeSystem];
    btn.frame = CGRectMake(100.0f, 100.0f, 100.0f, 30.0f);
    [btn setTitle:@"hit" forState:UIControlStateNormal];
    [btn addTarget:self action:@selector(hitFire:) forControlEvents:UIControlEventTouchUpInside];
    [self.view addSubview:btn];
    NSLog(@"========== add timer over");
}

- (void)hitFire:(UIButton *)btn{
    NSLog(@"hitBtn: ");
    [testTimer fire];
}

- (void)timerAction:(id)timer{
    NSLog(@"======== timerAction start ===========");
}
```
运行结果如下：
```bash
2013-11-04 15:05:33.571 podProject[11554:60b] ========== add timer over
2013-11-04 15:05:38.570 podProject[11554:60b] ======== timerAction start ===========
2013-11-04 15:05:40.821 podProject[11554:60b] hitBtn: 
2013-11-04 15:05:40.823 podProject[11554:60b] ======== timerAction start ===========
2013-11-04 15:05:43.570 podProject[11554:60b] ======== timerAction start ===========
2013-11-04 15:05:48.571 podProject[11554:60b] ======== timerAction start ===========
2013-11-04 15:05:50.405 podProject[11554:60b] hitBtn: 
2013-11-04 15:05:50.407 podProject[11554:60b] ======== timerAction start ===========
2013-11-04 15:05:53.571 podProject[11554:60b] ======== timerAction start ===========
2013-11-04 15:05:58.571 podProject[11554:60b] ======== timerAction start ===========
```
根据打印结果我们可以看到，在运行的过程中，我们出发了两次fire，然后timerAction:立即运行，但是原来的计划不变，隔5秒进行一次。

但是对于non-repeat timer来讲，当在action触发之前去调用fire方法，则timer会自动被置为invalidate。如上面的例子，当我们将创建timer的一句修改为testTimer = [NSTimer timerWithTimeInterval:5.0f target:self selector:@selector(timerAction:) userInfo:nil repeats:NO],则运行结果为：
```bash
2013-11-04 15:09:17.649 podProject[11577:60b] ========== add timer over
2013-11-04 15:09:20.947 podProject[11577:60b] hitBtn: 
2013-11-04 15:09:20.949 podProject[11577:60b] ======== timerAction start ===========
```

这就是fire的效果。


##2.关于timer selector的执行时间
一般情况下，我们的selector都能很快执行完成，但是当selector执行时间很久时，例如，超过了TimeInterval的时候会发生什么情况？我们看看下面的例子：
```objective-c
- (void)viewDidLoad
{
    [super viewDidLoad];
    
    testTimer = [NSTimer timerWithTimeInterval:5.0f target:self selector:@selector(timerAction:) userInfo:nil repeats:YES];
    NSRunLoop *runloop=[NSRunLoop currentRunLoop];
    [runloop addTimer:testTimer forMode:NSDefaultRunLoopMode];
}

- (void)timerAction:(id)timer{
    NSLog(@"======== timerAction start ===========");
    sleep(6.0f);
    NSLog(@"======= end=========");
}
```
运行结果为：
```bash
2013-11-04 15:15:47.720 podProject[11584:60b] ======== timerAction start ===========
2013-11-04 15:15:53.725 podProject[11584:60b] ======= end=========
2013-11-04 15:15:57.720 podProject[11584:60b] ======== timerAction start ===========
2013-11-04 15:16:03.723 podProject[11584:60b] ======= end=========
2013-11-04 15:16:07.720 podProject[11584:60b] ======== timerAction start ===========
2013-11-04 15:16:13.723 podProject[11584:60b] ======= end=========
2013-11-04 15:16:17.720 podProject[11584:60b] ======== timerAction start ===========
2013-11-04 15:16:23.723 podProject[11584:60b] ======= end=========
2013-11-04 15:16:27.720 podProject[11584:60b] ======== timerAction start ===========
2013-11-04 15:16:33.723 podProject[11584:60b] ======= end=========
2013-11-04 15:16:37.720 podProject[11584:60b] ======== timerAction start ===========
2013-11-04 15:16:43.723 podProject[11584:60b] ======= end=========
```
通过结果我们可以很容易的分析出，当timer的selector执行时间超过timeInterval的时候，其会丢弃已经过了的selector，按原计划等待最近的执行。

