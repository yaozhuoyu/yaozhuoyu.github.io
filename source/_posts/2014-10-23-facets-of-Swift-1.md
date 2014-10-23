title: Facets of Swift -- Optionals
date: 2014-10-23 21:40:48
categories: Swift
tags: [Swift, Optionals]
---
在Swift中，引入了一个新的类型Optional，关于Optional的概念，已经有很多的文章去介绍了，本文主要是总结Optional的用法，已经我们在使用Optional的时候应该要注意的地方。
<!-- more -->

### 什么是Optionals？

在Objective-C中，是没有明确的可选值的，为了说明问题，举下面一个例子，假设一个方法如下：

```objective-c
- (NSInteger)indexOfItem:(Item *)item {
    // ...
}
```
我们可以在调用方法的时候传入一个Item类型的对象或者传入一个nil，而返回值我们可以返回一个NSInteger类型的数字，或者找不到的时候返回NSNotFound。

对于参数item我们是没有办法明确要求传入一个非nil的值的，而我们也没办法要求返回的值类型一定为数字而不是NSNotFound。

而在Swift中，我们可以：
```swift
func indexOfItem(item: Item) -> Int? {
    // ...
}
```
如果我们在调用方法的时候传入一个值为nil的对象的话，编译器会报错的。

在类型的后面加上问号，就是此类型的可选。

### Unwrapping Optionals？

假设我们写一个方法返回一个item在list中的index，在Objc中，实现应该如下：
```objective-c
- (NSInteger)naturalIndexOfItem:(Item *)item {
    NSInteger indexOrNotFound = [self indexOfItem:item];
    if (indexOrNotFound != NSNotFound) {
        return indexOrNotFound + 1;
    } else {
        return NSNotFound;
    }
}
```
如果在Swift中，我们可能尝试如下写：
```swift
func naturalIndexOfItem(item: Item) -> Int? {
    let indexOrNotFound = indexOfItem(item)
    if indexOrNotFound {
        let index = indexOrNotFound!
        return index + 1
    } else {
        return nil
    }
}
```
其中方法indexOfItem返回的类型Optional，我们可以直接用if语句来判断Optionals，将其作为一个逻辑值。indexOrNotFound后面的感叹号是用来unwrap可选，因此index的类型为Int。

如果indexOrNotFound为nil，我们强制unwrap会在运行时报错。

### Optional Binding

上面的写法太麻烦，我们可以用一个简单的方法--可选绑定来简化代码：
```swift
func naturalIndexOfItem(item: Item) -> Int? {
    let indexOrNotFound = indexOfItem(item)
    if let index = indexOrNotFound {
        return index + 1
    } else {
        return nil
    }
}
```
更简单的写法：
```swift
func naturalIndexOfItem(item: Item) -> Int? {
    if let index = indexOfItem(item) {
        return index + 1
    } else {
        return nil
    }
}
```
我们可以用一个方法`successor()`来代替加1.
```swift
func naturalIndexOfItem(item: Item) -> Int? {
    if let index = indexOfItem(item) {
        return index.successor()
    } else {
        return nil
    }
}
```

### Optional Chaining

因为indexOfItem的返回为可选，所以我们不能这样写，编译器会报错的。
```swift
func naturalIndexOfItem(item: Item) -> Int? {
    return indexOfItem(item).successor()
}
```
我们可以强制unwrap，如下：
```swift
func naturalIndexOfItem(item: Item) -> Int? {
    return indexOfItem(item)!.successor()
}
```
但是当方法indexOfItem返回nil时，在运行的时候程序会crash。我们可以使用?代替！。
```swift
func naturalIndexOfItem(item: Item) -> Int? {
    return indexOfItem(item)?.successor()
}
```

### Implicitly Unwrapped Optionals

对于类型后加上!也是可选，是implicitly unwrapped optional，对于implicitly unwrapped optional我们可以直接取值而不用加上unwrap。

如下对于方法：
```swift
class func currentDevice() -> UIDevice!
```
我们在使用的时候直接调用UIDevice的方法或者属性就可以：
```swift
var name = UIDevice.currentDevice().name
```

以上就是可选的一般用法。

本文翻译自：[Facets of Swift, Part 1: Optionals](https://medium.com/swift-programming/facets-of-swift-part-1-optionals-b8ba5b0051a2)


