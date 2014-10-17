title: Swift会调用哪个函数？(Ⅰ)
date: 2014-10-17 18:09:09
categories:
- Swift
tags:
- Swift
- function
---

我们Swift中的函数是做为一等公民的身份出现的，因为其功能极其强大，但是功能的强大注定为其带来复杂，本文以及接下来的几篇文章主要详细的探讨当有多个函数选择的时候，Swift会去调用哪个函数。

我们知道，Swift中运算符`...`是有三个定义的：

```swift
/// Returns a closed interval from `start` through `end`
func ...<T : Comparable>(start: T, end: T) -> ClosedInterval<T>


/// Forms a closed range that contains both `minimum` and `maximum`.
func ...<Pos : ForwardIndexType>(minimum: Pos, maximum: Pos) -> Range<Pos>


/// Forms a closed range that contains both `start` and `end`.
/// Requres: `start <= end`
func ...<Pos : ForwardIndexType where Pos : Comparable>(start: Pos, end: Pos) -> Range<Pos>
```

当我们写语句`let r = 1...5`的时候，Swift选择最后一个函数调用，我们会得到一个`Range`，Swift是为何这样选择呢？

<!-- more -->

### Differing Return Values

在Swift中，我们是不能声明两个完全相同的函数的：
```swift
func f() {  }
 
// error: Invalid redeclaration of f()
func f() {  }
```
但是和其他语言不一样，我们可以在Swift中定义参数和名字都相同但是返回值不同的函数，如下是合法的：
```swift
struct A { }
struct B { }
 
func f() -> A { return A() }
 
// this second f is fine, even though it only differs
// by what it returns:
func f() -> B { return B() } 
 
// note, typealiases are just aliases not different types,
// so you can't do this:
typealias L = A
// error: Invalid redeclaration of 'f()'
func f() -> L { return A() }
```
当我们在调用这样的函数的时候，我们要给Swift提供足够的信息，不让其产生歧义，如下：
```swift
// error: Ambiguous use of 'f'
let x = f()
 
// instead you must specify which return value you want:
let x: A = f()    // calls the A-returning version
                  // x is of type A
 
// or if you prefer this syntax:
let y = f() as B  // calls the B-returning version
                  // y is of type B
 
// or, maybe you're passing it in as the argument to a function:
func takesA(a: A) { }
 
// the A-returning version of f will be called
takesA(f())
```
如果我们想声明一个变量引用函数f，我们需要指定完整的类型：
```swift
// g will be a reference to the A-returning version
let g: ()->A = f
 
// h will be a reference to the B-returning version
let h = f as ()->B
 
// calling g doesn't need further info, it references 
// a specific f, and z will be of type A:
let z = g()
```

OK，现在我们在回过头看`...`运算符，它会返回`ClosedInterval`或者`Range`，对于`let r = 1...5`没有声r的类型，却得到`Range`类型的返回值？

为了回答这个问题，我们需要看函数的参数，我们从单个参数的函数开始。

### Single Arguments

和返回值不同，如果相同函数名的函数的参数不同，Swift会自动的帮我们选择最好的匹配，然后调用。

做为一个经验法则，Swift总是挑选最"具体"的函数，Swift喜欢那些有精确类型的函数。

如果不能找到需要精确类型的函数，接下来的首选方案是继承的类或者协议，子类或者协议优先与父类。

如下例子：
```swift
protocol P { }
protocol Q: P { }
 
struct S: Q { }
 
// this is the function that will be picked
// if called with type S
func f(s: S) { print("S") }
 
// if that weren't defined, this would be the next
// choice for a call passing in an S, since Q is
// more specialized than P:
func f(p: Q) { print("Q") }
 
// and then finally this
func f(p: P) { print("P") }
 
f(S())  // prints S
 
class C: P { }
class D: C { }
class E: D { }
 
// this is the function that will be picked
func f(d: D) { print("D") }
 
// if that weren't defined, this would be the next choice
func f(c: C) { print("C") }
 
// if neither were defined, the version above for the 
// protocol P would be called
 
f(E())  // prints D
```








http://airspeedvelocity.net/2014/09/20/which-func/
http://airspeedvelocity.net/2014/09/24/which-function-does-swift-call-part-2-single-arguments/

