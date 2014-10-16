---
layout: post
title: "NSURLSession总结"
date: 2013-10-19 16:27
comments: true
categories: iOS 
tags: iOS7
---
在iOS7中，apple在网络中引入了NSURLSession类，最近一直在研究这个类，觉得挺强大的，最主要的是支持后台下载了，连著名的网络开源库[AFNetworking](https://github.com/AFNetworking/AFNetworking)为了支持此类，都重写了架构，发布了2.0版。本节讲对我这几天对这个类的学习总结一下，[官方文档参考](https://developer.apple.com/library/ios/documentation/Cocoa/Conceptual/URLLoadingSystem/Tasks/UsingNSURLConnection.html).
<!--more-->
##1.NSURLSession的概念

为了使用NSURLSession的API，app中可能创建多个sessions,而没有session都对应一组数据传输的任务（data transfer tasks）。对于每一个session,app可以在给其添加多个task，其中每一个task代表一个具体的URL request.

NSURLSession API已经高度的异步化，使用NSURLSession可以有两个方式：

1.使用系统提供的delegate，也就是无需自己提供delegate，只用提供一个completion handler block,此block会返回下载的data给用户，当下载失败是会返回error信息。 
``` objective-c
+ (NSURLSession *)sessionWithConfiguration:(NSURLSessionConfiguration *)configuration
```
2.使用自己的delegate，这样得到各种想要的回调信息，方便自定制。注意：NSURLSession对delegate是强引用。
``` objective-c
+ (NSURLSession *)sessionWithConfiguration:(NSURLSessionConfiguration *)configuration delegate:(id<NSURLSessionDelegate>)delegate delegateQueue:(NSOperationQueue *)queue
```

一个session中task的行为主要有以下三件事情决定：session的类型（主要由创建session时使用的configuration对象类型决定），task的类型，以及当task创建时app是否在前台。

NSURLSession有三种类型：

* Default session，主要用于从URL下载，cache基于硬盘的，credentials存储在user keychain中。
* Ephemeral session(短暂的session)，不会存储任何数据到硬盘，所有的caches,credentials都存储在RAM中，因此当app把一个session设置为失效的时候，所有的这些都会自动清除
* Background session，和default session有点相似，除了一个单独的进程处理所有的数据传输。

NSURLSession支持三种类型的task：

* Data tasks，使用NSData发送和接收数据，Data tasks主要用于和服务器的进行短的，交互式的请求。因为Data tasks不会把data存储为文件，因此background session不支持此种类型的task。
* Download tasks，以文件的形式获取到数据，支持background session
* Upload tasks，以文件的形式发送数据，background session


##2.NSURLSession流程

在前台的时候，NSURLSession和NSConnection的流程基本一样，再此不再讲述。这里主要讲述在由后台的时候的情况。

在iOS中，当一个后台传输完成或者请求证书的时候，如果你的app已经不在运行，iOS会自动的重启你的app后台运行，并调用app的UIApplicationDelegate的方法`application:handleEventsForBackgroundURLSession:completionHandler:`。在此方法中我们需要把completionHandler存储起来，并用回调提供的identifier创建一个background configuration object，并用configuration object创建一个session。这个新创建的session会和正在进行的background task进行连接。当session完成了最后一个background task的时候，session的delegate会发送一个` URLSessionDidFinishEventsForBackgroundURLSession: `消息，应该在此方法中去调用刚刚存储起来的completionHandler。

当用户重新运行app的时候，如果app上次运行中有session有没有完成的任务，app应当立即使用一个相同的标识（与上一个session）去创建一个background configuration object，并根据其去创建一个新的session，这个新的session会自动的连接正在进行的task。

流程就好用流程图去讲解，下次画一个详细的流程图讲解。


