---
layout: post
title: 关于CGContextSaveGState、CGContextRestoreGState 和UIGraphicsPushContext、UIGraphicsPopContext
date: 2014-01-12 23:20
comments: true
categories: iOS
---
在iOS绘图开发中，经常会看到成对的函数CGContextSaveGState/CGContextRestoreGState 和 UIGraphicsPushContext/UIGraphicsPopContext出现，一些初学者容易把这两对函数弄混淆。所以决定写一篇文章把其总结一下，高手可以略过。。

<!--more-->
##1.CGContextSaveGState/CGContextRestoreGState
首先来看CGContextSaveGState/CGContextRestoreGState这一对函数，他们属于Quartz函数。主要用于保存上下文的绘制状态，并在之后恢复状态。
我们知道context就像一套绘制工具一样，当我们把绘制的线条颜色设置为一种颜色之后，如果在后续不去修改其颜色，它的线条颜色会一直保持下去。

```objective-c
[[UIColor redColor] setStroke]; 
CGContextSaveGState(UIGraphicsGetCurrentContext()); 
[[UIColor blackColor] setStroke]; 
CGContextRestoreGState(UIGraphicsGetCurrentContext()); 
UIRectFill(CGRectMake(10, 10, 100, 100)); // Red
```

看上面的例子，就是一个CGContextSaveGState/CGContextRestoreGState的简单例子。
CGContextSaveState会保持当前context的状态，当在调用CGContextRestoreGState之后，会恢复到之前保存的状态。

CGContextSaveGState/CGContextRestoreGState自始至终是对同一个context来讲的。

##2.UIGraphicsPushContext/UIGraphicsPopContext

UIGraphicsPushContext/UIGraphicsPopContext函数是用来切换context的，因此，和CGContextSaveGState/CGContextRestoreGState完全不是一个概念。

UIGraphicsPushContext的目的不是用来保存当前context的状态的（线的颜色，宽度等等），它是用来切换整个context。

UIGraphicsPushContext/UIGraphicsPopContext是属于UIKit函数，最经常用到这两个函数的地方是在一个bitmap context中使用UIKit函数。
因为所有的UIKit draw相关的函数都是没有CGContext参数的，他们会默认把当前的context作为context。在UIView的drawRect方法里面，UIGraphicsGetCurrentContext()就是当前context。但是在自己创建的bitmap context中，当前的context就不是bitmap context了，需要我们用UIGraphicsPushContext去设置，下面举一个简单的例子：

```objective-c
- (UIImage *)drawImageContext{
    CGColorSpaceRef colorSpace = CGColorSpaceCreateDeviceRGB();
    if (colorSpace == NULL) {
        return nil;
    }
    size_t width = 100;
    size_t height = 100;
    CGContextRef context = CGBitmapContextCreate(NULL, width, height, 8, width *4, colorSpace, (CGBitmapInfo)kCGImageAlphaPremultipliedFirst);
    
    if (context == NULL) {
        CGColorSpaceRelease(colorSpace);
        return nil;
    }
    
    //begin draw
    UIBezierPath *path = [UIBezierPath bezierPathWithOvalInRect:CGRectMake(20.0f, 20.0f, 50, 50)];
    path.lineWidth = 4;
    [[UIColor redColor] setStroke];
    [path stroke];
    
    
    CGImageRef imageRef = CGBitmapContextCreateImage(context);

    UIImage *image = [UIImage imageWithCGImage:imageRef];
    
    CGColorSpaceRelease(colorSpace);
    CGContextRelease(context);
    CFRelease(imageRef);
    
    return image;
}
```
上面的函数是不会生成一个椭圆的image的，因为[[UIColor redColor] setStroke];[path stroke];这些函数当前的context不是CGBitmapContextCreate。如果想要达到生成一个椭圆image的效果需要加上UIGraphicsPushContext/UIGraphicsPopContext，如下：

```objective-c
- (UIImage *)drawImageContext{
    CGColorSpaceRef colorSpace = CGColorSpaceCreateDeviceRGB();
    if (colorSpace == NULL) {
        return nil;
    }
    size_t width = 100;
    size_t height = 100;
    CGContextRef context = CGBitmapContextCreate(NULL, width, height, 8, width *4, colorSpace, (CGBitmapInfo)kCGImageAlphaPremultipliedFirst);
    
    if (context == NULL) {
        CGColorSpaceRelease(colorSpace);
        return nil;
    }
    
    //begin draw
    UIGraphicsPushContext(context);
    UIBezierPath *path = [UIBezierPath bezierPathWithOvalInRect:CGRectMake(20.0f, 20.0f, 50, 50)];
    path.lineWidth = 4;
    [[UIColor redColor] setStroke];
    [path stroke];
    UIGraphicsPopContext();
    
    CGImageRef imageRef = CGBitmapContextCreateImage(context);

    UIImage *image = [UIImage imageWithCGImage:imageRef];
    
    CGColorSpaceRelease(colorSpace);
    CGContextRelease(context);
    CFRelease(imageRef);
    
    return image;
}

```

上面是我对这两对函数的理解，如果有疑问可以给我留言。


