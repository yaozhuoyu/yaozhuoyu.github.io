title: swift-optionals-map
date: 2014-11-29 13:01:16
categories: Swift
tags: [Swift, Optional]
---
Swift中的Optional可以说是让人又喜欢又讨厌，特别是在 optional binding的时候，我们需要添加`if let`语句，使代码变得比较杂乱。

例如下面一个方法，目的是将一个可选值加一，并返回：

```swift
func increment(someNumber: Int?) -> Int? {
    if let number = someNumber {
        return number + 1
    } else {
        return nil
    }
}
 
increment(5)   // Some 6
increment(nil) // nil
```

<!-- more -->

这会使我们的代码被if else语句包围。其实，在Optional的定义中，有一个map方法，可以帮我们解决部分问题：

```swift
enum Optional<T> : Reflectable, NilLiteralConvertible {
    case None
    case Some(T)

    /// Construct a `nil` instance.
    init()

    /// Construct a non-\ `nil` instance that stores `some`.
    init(_ some: T)

    /// If `self == nil`, returns `nil`.  Otherwise, returns `f(self!)`.
    func map<U>(f: (T) -> U) -> U?

    /// Returns a mirror that reflects `self`.
    func getMirror() -> MirrorType

    /// Create an instance initialized with `nil`.
    init(nilLiteral: ())
}
```

根据Optional map的定义，我们可以将increment方法改写为：

```swift
func increment(someNumber: Int?) -> Int? {
    return someNumber.map { number in number + 1 }
}
 
increment(5)   // Some 6
increment(nil) // nil
```

这个map方法可以工作在各种可选类型下，例如String也可以：

```swift
func hello(someName: String?) -> String? {
    return someName.map { name in "Hello, \(name)"}
}
 
hello("ioser") // Some "Hello, ioser"
hello(nil) // nil
```


翻译自：[原文地址](http://natashatherobot.com/swift-using-map-to-deal-with-optionals/)
















