title: Swift运行速度的秘密
date: 2014-10-21 00:58:53
categories: Swift
tags: Swift
---
自从Swift推出之后，官方特别强调的是速度是其一个很大的优势，它比动态语言例如Python、JavaScript快很多，同时也比Objective-C的运行速度快，并且声称在某些情况下比C的速度也要快。但真的是这样的吗？
<!-- more -->

虽然Swift语言应该有更出色的性能，但是当前的编译器优化的并不是很好，Swift很难在性能测试中发挥出优势，大部分原因是因为编译调用了大量的无关的retian/release，希望在以后的版本中对此有优化。这篇文章主要是讲Swift在运行速度上为什么会比Objective-C有优势。

### Faster Method Dispatch

我们都知道，Objective-C方法的每一次调用，都会被转化为调用`objc_msgSend`，此方法负责运行时在class method tables中寻找selector对应的函数。

考虑下面的Swift代码：
```Swift
class Class {
        func testmethod1() { print("testmethod1") }
        @final func testmethod2() { print("testmethod2") }
    }

    class Subclass: Class {
        override func testmethod1() { print("overridden testmethod1") }
    }

    func TestFunc(obj: Class) {
        obj.testmethod1()
        obj.testmethod2()
    }

```
在Swift中，编译器可以利用语言提供的优势--更严格的类型检查，在Objective-C中，我们可以欺骗编译器，对象可能跟我们声明的类型不匹配，但是在Swift中，我们我允许这么做。如果我们说obj是Class类型的实例，那么obj要么是Class类型，要么是其子类类型。

Swift编译器调用函数是通过index在一个叫vtable的数组中获取函数指针，vtable是一个函数指针数组。编译器调用一个方法的过程大概是这样的：
```
methodImplementation = object->class.vtable[indexOfMethod1]
methodImplementation()
```
数组索引机制是比objc_msgSend机制在速度上有优势的。

方法testmethod2的表现更好，因为其被声明为@final，编译器知道子类不能去重写此方法，因此不管什么情况，调用此方法总是去寻找Class对应的实现。因此编译器不需要在vtable中去寻找，其可以直接进行一个普通的函数调用---__TFC9speedtest5Class11testmethod2fS0_FT_T_(这个是我们例子中testmethod2方法实现的mangled name)。

这并不是巨大的优势，objc_msgSend也是很快的，大多数程序不会在这个上面花费大量时间。因此，从这个方面，Swift比Objective-C最多有几个百分点的优势。

































https://mikeash.com/pyblog/friday-qa-2014-07-04-secrets-of-swifts-speed.html

