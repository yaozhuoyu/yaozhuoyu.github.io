---
layout: post
title: "Building Concurrent User Interfaces on iOS 总结"
date: 2013-11-01 19:59
comments: true
categories: iOS
tags: NSOperation WWDC NSOperationQueue
---

今天抽了一点时间看了WWDC 2012中的一集视频 -- Building Concurrent User Interfaces on iOS，觉得很有用，在此总结一下。

全文主要解决了一个问题，如何让tableview滚动起来更流畅。如何解决呢？通过将繁重的绘制工作从主线程提出来，放在其他线程做，而解放主线程。文字表达不好，来看代码吧。
<!--more-->
```objective-c

- (void)tableView:(UITableView *)tableView didEndDisplayingCell:(UITableViewCell *)cell forRowAtIndexPath:(NSIndexPath *)indexPath{
    NSString *stockName = [[_stocks objectAtIndex:[indexPath row]] name];
    NSOperation *operation = [_stockNamesToRenderingOperations objectForKey:stockName];
    if (operation) {
        [operation cancel];
        [_stockNamesToRenderingOperations removeObjectForKey:stockName];
    }
}

- (UITableViewCell *)tableView:(UITableView *)tableView cellForRowAtIndexPath:(NSIndexPath *)indexPath {
    static NSString *CellIdentifier = @"Cell";
    
    UITableViewCell *cell = [tableView dequeueReusableCellWithIdentifier:CellIdentifier];
    if (!cell) {
        cell = [[UITableViewCell alloc] initWithStyle:UITableViewCellStyleDefault reuseIdentifier:CellIdentifier];
    }
    
    CRStock *stock = [_stocks objectAtIndex:[indexPath row]];
    
    CRStockRenderer *renderer = [_stockNamesToRenderers objectForKey:[stock name]];
    if ([renderer hasRendered]) {
        [[cell imageView] setImage:[renderer renderedGraphOfSize:ROW_IMAGE_SIZE]];
    }else{
        if (!renderer) {
            renderer = [[CRStockRenderer alloc] initWithStock:stock];
            [_stockNamesToRenderers setObject:renderer forKey:[stock name]];
            
            NSBlockOperation *operation = [NSBlockOperation blockOperationWithBlock:^{
                UIImage *renderedImage = [renderer renderedGraphOfSize:ROW_IMAGE_SIZE];
                [[NSOperationQueue mainQueue] addOperationWithBlock:^{
                    [[[tableView cellForRowAtIndexPath:indexPath] imageView] setImage:renderedImage];
                }];
            }];
            [_queue addOperation:operation];
            [_stockNamesToRenderingOperations setObject:operation forKey:[stock name]];
        }
        [[cell imageView] setImage:[renderer placeholderImageOfSize:ROW_IMAGE_SIZE]];
    }
    
    return cell;
}

- (void)viewDidDisappear:(BOOL)animated{
    [super viewDidDisappear:animated];
    [_queue cancelAllOperations];
}

```
```objective-c
@interface CRStockRenderer : NSObject{
    UIImage *_renderedGraph;
}
- (UIImage *)renderedGraphOfSize:(CGSize)desiredSize;
@end

@implementation CRStockRenderer
- (UIImage *)renderedGraphOfSize:(CGSize)desiredSize{
    if (_renderedGraph)
        return _renderedGraph;
    
    UIGraphicsBeginImageContextWithOptions(desiredSize, NO, 0.0f);
    
    //do a lot work
    
    _renderedGraph = UIGraphicsGetImageFromCurrentImageContext();
    UIGraphicsEndImageContext();
    
    return _renderedGraph;
}

- (BOOL)hasRendered{
    return _renderedGraph != nil;
}

@end
```

如果对NSOperationQueue和NSOperation了解的话，上面的代码很容易理解了。

