---
layout: post
title: "关于Dispatch Sources"
date: 2013-11-12 21:11
comments: true
categories: iOS
tags: GCD
---
dispatch source是一个基本数据类型，它用来协调处理low-level的系统事件，其中包括timer dispatch source，signal dispatch source，descriptor sources，process dispatch source，Mach port dispatch source，custom dispatch sources。

当配置一个dispatch source的时候，我们需要指定监控事件的类型，当事件发生时要执行的block code，已经执行code的queue，当一个监控的事件发生时，dispatch source提交block到指定的dispatch queue去执行。dispatch source会retain住其相关联的dispatch queue，阻止其被过早的被release。

为了防止事件在dispatch queue中积压，dispatch sources实现了一个事件聚合机制(event coalescing scheme)，如果一个事件在上一个事件还没有开始处理之前到达，dispatch source会聚合两次事件的data。根据事件的类型，聚合可能是代替旧的事件或者时更新信息。

下面将详细说明Dispatch Sources。
<!--more-->

##1.创建Dispatch Sources

创建一个Dispatch Source分为下面几步：
(1)使用函数`dispatch_source_create`创建一个dispatch source。
(2)配置dispatch source，其中包括给dispatch source赋值一个event handler，对于timer source，需要使用`dispatch_source_set_timer`设置timer信息。
(3)给dispatch source赋值一个cancel handler(可选的)
(4)调用`dispatch_resume`开始。

因为dispatch source在创建之后，需要进行一些配置才能被使用，因此`dispatch_source_create`返回的dispatch source状态为suspended，在suspended状态，一个dispatch source接收到事件，但并不处理。下面主要讲一下dispatch source的配置。

通过函数`dispatch_source_set_event_handler` 或者 `dispatch_source_set_event_handler_f`设置dispatch source的event handler，当event事件到达的时候，dispatch source提交event handler到设定的dispatch queue运行。

event handler有责任处理所有到达的事件。如果一个event handler已经入队等待执行的时候，一个新的event到来，dispatch source会聚合两个事件。event handler通常只能看到最近的event信息。如果一个或者多个event到达的时候，event handler已经在执行，则dispatch source会持有这些events，直到event handler完成，这时会提交新的event handler处理新的events。

下面是一个配置event handler的例子：
```objective-c
dispatch_source_t source = dispatch_source_create(DISPATCH_SOURCE_TYPE_READ,
                                                      myDescriptor, 0, myQueue);
    dispatch_source_set_event_handler(source, ^{
        // Get some data from the source variable, which is captured
        // from the parent context.
        size_t estimated = dispatch_source_get_data(source);
        
        // Continue reading the descriptor...
    });
    dispatch_resume(source);
```

cancel handler在dispatch source被释放的时候用于清理工作。对于大多数dispatch source类型，cancel handler是可选的。对于一些dispatch source，使用了descriptor或者Mach port，则需要提供cancel handler去close descriptor或者release Mach port。

安装cancel handler通过函数 `dispatch_source_set_cancel_handler`或者`dispatch_source_set_cancel_handler_f`，下面例子：
```objective-c
dispatch_source_set_cancel_handler(mySource, ^{
   close(fd); // Close a file descriptor opened earlier.
});
```

和其他的GCD数据类型一样，我们可以使用函数`dispatch_set_context`去设置和dispatch source相关联的custom data。如果我们在context中存储了custom data，则我们应该设置cancel handler，当dispatch source不在需要的时候去释放这些data。

##2.例子
下面将对每一个dispatch source给出一个例子。

(1)Timer
Timer source顾名思义，就是会按照指定的间隔时间周期性的产生事件。当电脑进入sleep模式，所有的timer dispatch source也会被暂停，当电脑wake up的时候，这些dispatch source也会自动的wake。

根据timer source配置的不同，当一个timer暂停之后再次触发的时间可能不同。如果使用`dispatch_time`设置dispatch source，dispatch source使用系统提供的缺省时钟，当程序处于sleep的时候，系统默认时钟是不前进的。相反，当使用`dispatch_walltime`去设置timer dispatch source的时候，timer触发是按照真实时间计算的。
下面看一个例子，首先使用dispatch_walltime去设置timer dispatch source。
```objective-c
- (void)createTimerSource{
    timerSource = dispatch_source_create(DISPATCH_SOURCE_TYPE_TIMER, 0, 0, dispatch_get_main_queue());
    dispatch_source_set_timer(timerSource,
                              dispatch_walltime(NULL, 0),//start
                              30ull * NSEC_PER_SEC,//interval
                              1ull * NSEC_PER_SEC//leeway
                              );
    dispatch_source_set_event_handler(timerSource, ^{
        NSLog(@"timer fire");
    });
    dispatch_resume(timerSource);
}
```

或者将其中的`dispatch_walltime(NULL, 0)`改为`dispatch_time(DISPATCH_TIME_NOW, 0)`，发现两者在iOS上面没有差异，不知道在Mac上怎么样，感兴趣的可以试一试。

(2)从descriptor读取数据
先看例子：
```objective-c
Boolean MyProcessFileData(char *buffer, ssize_t size){
    //todo:
    return YES;
}

dispatch_source_t ProcessContentsOfFile(const char* filename)
{
    // Prepare the file for reading.
    int fd = open(filename, O_RDONLY);
    if (fd == -1)
        return NULL;
    fcntl(fd, F_SETFL, O_NONBLOCK);  // Avoid blocking the read operation
    
    dispatch_queue_t queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
    dispatch_source_t readSource = dispatch_source_create(DISPATCH_SOURCE_TYPE_READ,
                                                          fd, 0, queue);
    if (!readSource)
    {
        close(fd);
        return NULL;
    }
    
    // Install the event handler
    dispatch_source_set_event_handler(readSource, ^{
        size_t estimated = dispatch_source_get_data(readSource) + 1;
        // Read the data into a text buffer.
        char* buffer = (char*)malloc(estimated);
        if (buffer)
        {
            ssize_t actual = read(fd, buffer, (estimated));
            Boolean done = MyProcessFileData(buffer, actual);  // Process the data.
            
            // Release the buffer when done.
            free(buffer);
            
            // If there is no more data, cancel the source.
            if (done)
                dispatch_source_cancel(readSource);
        }
    });
    
    // Install the cancellation handler
    dispatch_source_set_cancel_handler(readSource, ^{close(fd);});
    
    // Start reading the file.
    dispatch_resume(readSource);
    return readSource;
}
```
上面的例子是从一个file中读取数据。其中需要注意的是fcntl设置为非阻塞的，其他的都很简单。

(3)向descriptor写数据
```objective-c
dispatch_source_t WriteDataToFile(const char* filename)
{
    int fd = open(filename, O_WRONLY | O_CREAT | O_TRUNC,
                  (S_IRUSR | S_IWUSR | S_ISUID | S_ISGID));
    if (fd == -1)
        return NULL;
    fcntl(fd, F_SETFL); // Block during the write.
    
    dispatch_queue_t queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
    dispatch_source_t writeSource = dispatch_source_create(DISPATCH_SOURCE_TYPE_WRITE,
                                                           fd, 0, queue);
    if (!writeSource)
    {
        close(fd);
        return NULL;
    }
    
    dispatch_source_set_event_handler(writeSource, ^{
        size_t bufferSize = MyGetDataSize();
        void* buffer = malloc(bufferSize);
        
        size_t actual = MyGetData(buffer, bufferSize);
        write(fd, buffer, actual);
        
        free(buffer);
        
        // Cancel and release the dispatch source when done.
        dispatch_source_cancel(writeSource);
    });
    
    dispatch_source_set_cancel_handler(writeSource, ^{close(fd);});
    dispatch_resume(writeSource);
    return (writeSource);
}
```

(4)监视文件系统对象的变化
如果想要监视一个文件系统对象的变化，创建dispatch source的时候使用类型DISPATCH_SOURCE_TYPE_VNODE，当一个文件删除，写入，或者重命名等等的时候，可以收到通知。
```objective-c
dispatch_source_t MonitorNameChangesToFile(const char* filename)
{
    int fd = open(filename, O_EVTONLY);
    if (fd == -1)
        return NULL;
    
    dispatch_queue_t queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
    dispatch_source_t source = dispatch_source_create(DISPATCH_SOURCE_TYPE_VNODE,
                                                      fd, DISPATCH_VNODE_RENAME, queue);
    if (source)
    {
        // Copy the filename for later use.
        int length = strlen(filename);
        char* newString = (char*)malloc(length + 1);
        newString = strcpy(newString, filename);
        dispatch_set_context(source, newString);
        
        // Install the event handler to process the name change
        dispatch_source_set_event_handler(source, ^{
            const char*  oldFilename = (char*)dispatch_get_context(source);
            MyUpdateFileName(oldFilename, fd);
        });
        
        // Install a cancellation handler to free the descriptor
        // and the stored string.
        dispatch_source_set_cancel_handler(source, ^{
            char* fileStr = (char*)dispatch_get_context(source);
            free(fileStr);
            close(fd);
        });
        
        // Start processing events.
        dispatch_resume(source);
    }
    else
        close(fd);
    
    return source;
}
```
上面的例子是用来监控文件名字的变化。

还有监视signal和监视进程，因为iOS中不能使用，所以在此处不在讲解。

##3.取消一个dispatch source
只有在调用`dispatch_source_cancel`之后，一个dispatch source才会被取消。取消一个dispatch source之后，就会停止收到新的事件，并且是不可撤销的，因此一般在cancel一个source之后，就释放其。
```objective-c
void RemoveDispatchSource(dispatch_source_t mySource)
{
    dispatch_source_cancel(mySource);
    dispatch_release(mySource);
}
```
cancel是一个异步操作，尽管调用cancel函数之后，就停止接收新的事件，但是会等待已经接收的事件处理完，才真正取消。
