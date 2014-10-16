---
layout: post
title: "iOS中关于UIView布局的总结"
date: 2013-10-20 13:11
comments: true
categories: iOS
tags: UIView AutoLayout 
---
在iOS开发中，关于UIView布局的相关知识一直没有系统的总结过，所以今天决定好好的总结一下。
<!--more-->
##1.Laying out Subviews
关于布局UIView的subviews，系统提供了相关的三个api：

``` objective-c
- (void)layoutSubviews
```
在iOS5.1和之前的版本，此方法的缺省实现不会做任何事情(实现为空)，iOS5.1之后(iOS6开始)的版本，此方法的缺省实现是使用你设置在此view上面的constraints(Autolayout)去决定subviews的position和size。
UIView的子类如果需要对其subviews进行更精确的布局，则可以重写此方法。只有在`autoresizing`和`constraint-based behaviors of subviews`不能提供我们想要的布局结果的时候，我们才应该重写此方法。可以在此方法中直接设置subviews的frame。
我们不应该直接调用此方法，而应当用下面两个方法。

``` objective-c
- (void)setNeedsLayout
```
此方法会将view当前的layout设置为无效的，并在下一个upadte cycle里去触发layout更新。

``` objective-c
- (void)layoutIfNeeded
```
使用此方法强制立即进行layout,从当前view开始，此方法会遍历整个view层次(包括superviews)请求layout。因此，调用此方法会强制整个view层次布局。


##2.Triggering Constraint-Based Layout
关于autolayout中的一些方法：

``` objective-c
- (void)setNeedsUpdateConstraints
```
当一个自定义view的某个属性发生改变，并且可能影响到constraint时，可需要调用此方法去标记constraints需要在未来的某个点更新，系统然后调用`updateConstraints`.

``` objective-c
- (BOOL)needsUpdateConstraints
```
constraint-based layout system使用此返回值去决定是否需要调用`updateConstraints`作为正常布局过程的一部分。

``` objective-c
- (void)updateConstraints
```
自定义view应该重写此方法在其中建立constraints.
注意：要在实现在最后调用`[super updateConstraints]`


##3.Layout Process

与使用springs and struts(autoresizingMask)比较，Auto layout在view显示之前，多引入了两个步骤：updating constraints 和laying out views。每一个步骤都依赖于上一个。display依赖layout，而layout依赖updating constraints。
`updating constraints->layout->display`

第一步：updating constraints，被称为测量阶段，其从下向上(from subview to super view),为下一步layout准备信息。可以通过调用方法`setNeedUpdateConstraints`去触发此步。constraints的改变也会自动的触发此步。但是，当你自定义view的时候，如果一些改变可能会影响到布局的时候，通常需要自己去通知Auto layout。说到自定义view了，通常可以重写updateConstraints方法，在其中可以添加view需要的局部的contraints。

第二步：layout，其从上向下(from super view to subview)，此步主要应用上一步的信息去设置view的center和bounds。可以通过调用setNeedsLayout去触发此步骤，此方法不会立即应用layout。如果想要系统立即的更新layout，可以调用layoutIfNeeded。另外，自定义view可以重写方法layoutSubViews来在layout的工程中得到更多的控制。

第三步：display，此步时把view渲染到屏幕上，它与你是否使用Auto layout无关，其操作是从上向下(from super view to subview)，通过调用setNeedsDisplay触发，

因为每一步都依赖前一步，因此一个display可能会触发layout，当有任何layout没有被处理的时候，同理，layout可能会触发updating constraints，当constraint system更新改变的时候。

需要注意的是，这三步不是单向的，constraint-based layout是一个迭代的过程，layout过程中，可能去改变constraints,有一次触发updating constraints,进行一轮layout过程。这可以被用来创建高级的自定义视图布局，但是如果你每一次调用自定义layoutSubviews都会导致另一个布局传递，那么你将会陷入一个无限循环中。
如下图：

{% img http://yuzhongleixueren.u.qiniudn.com/20131020_1.png 369 492 %}





