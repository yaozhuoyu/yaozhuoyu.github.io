title: Swift会调用哪个函数？
date: 2014-10-17 18:09:09
categories:
- Swift
tags:
- Swift
- function
---

我们Swift中的函数是做为一等公民的身份出现的，因为其功能极其强大，但是功能的强大注定为其带来复杂，本文接下来主要详细的探讨当有多个函数选择的时候，Swift会去调用哪个函数。
<!-- more -->

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

## Differing Return Values

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

## Single Arguments

和返回值不同，如果相同函数名的函数的参数不同，Swift会自动的帮我们选择最好的匹配，然后调用。
做为一个经验法则，Swift总是挑选最"具体"的函数，Swift喜欢那些有精确类型的函数。
如果不能找到精确类型匹配的函数，接下来的首选方案是继承的类或者协议，子类或者子协议优先于父类和父协议。

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
这看起来很像多态，但是我们要知道，这不是多态，函数的选择是在编译的时候就确定的。

## Overloaded functions are chosen at compile time

在上面的所有情况里，函数调用的确定都是静态非动态的，都是在编译期而非运行期。这表达的意思就是函数调用的确定根据的是变量的类型而不是变量所指的类型。
```swift
class C { }
class D: C { }
 
// function that takes the base class
func f(c: C) { print("C") }
// function that takes the child class
func f(d: D) { print("D") }
 
// the object x references is of type D,
// but x is of type C
let x: C = D()
 
// which function to call is determined by the
// type of x, and not what it points to
f(x)    // Prints "C" _not_ "D"
 
// the same goes for protocols...
protocol P { }
struct S: P { }
 
func f(s: S) { print("S") }
func f(p: P) { print("P") }
 
let p: P = S()
 
// despite p pointing to an object of type S,
// the version that takes a P will be called:
f(p)    // Prints "P" _not_ "S"
```
如果我们想要多态的效果，则应该用方法去替换函数，如下：
```swift
class C {
    func f() { print("C") }
}
 
class D: C {
    override func f() { print("D") }
}
 
// function that takes the child class
 
// the object x references is of type D,
// but x is of type C:
let x: C = D()
 
// which method to call is determined by 
// what x points to, and not what type it is
x.f()    // Prints "D" _not_ "C"
```
另一种常见的例子是，我们可以重载采用不同协议的函数，在编译期检查协议的满足性，但是在调用方法的时候会有动态的行为，但是有一个缺点：

##Functions that take Protocols can be ambiguous

使用协议的时候，可能产生歧义：
```swift
protocol P { }
protocol Q { }
 
struct S: P, Q { }
let s = S()
 
func f(p: P) { print("P") }
func f(p: Q) { print("Q") }
 
// error: argument could be either a P or a Q
f(s)
 
// so you need to disambiguate, either:
f(s as P)  // prints P
 
// or:
let q: Q = S()
f(q)       // prints Q
 
// the same happens with classes vs protocols:
class C  { }
class D: C, P { }
 
func f(c: C) { print("C") }
 
// error: argument could be either a C or a P
f(D())
// you must specify, either:
f(D() as P)
// or:
f(D() as C)
```

##Defaulting to ClosedInterval for Integers

因此，我们可以得出一个结论：采用strcut类型参数的函数是优先于采用protocol类型参数的函数，我们可以自定义写一个新的...运算符的函数，参数为struct(Int)类型，则其优先级将高于当前参数为protocols的函数版本(Comparable to create a ClosedInterval, 或者 ForwardIndex to create a Range)
```swift
func ...(start: Int, end: Int) -> ClosedInterval<Int> {
    return ClosedInterval(start, end)
}
 
// x is now of type ClosedInterval
let x = 1...5
```
但是如果传入的参数类型是其他整数类型，我们仍然是得到一个Range:
```swift
let i: Int32 = 1, j: Int32 = 5
 
// r will be of type Range still:
let r = i...j
```
但是这些还是没有解释最初的问题，为什么选择ranges而不是closed intervals。Comparable和ForwardIndex也没有继承关系。

##Protocol Composition
要知道 `1...5`选择返回Range而不是ClosedInterval的原因，我们首先要看看generics(泛型)是如何匹配的。

在这之前，我们要引入protocol composition的概念。
Protocols可以有0个或者多个protocols合成，有<>包含起来。Protocol composition并不是定义了新的protocol类型，相反，他们只是定义了一个临时的本地协议，要求满足所有的协议要求。

通过这个，我们可能想象声明protocol composition如<P,Q>很像声明一个匿名的协议 Tmp:P,Q{}，然后使用Tmp不用给出名字。但是并不是完全这样，特别是当protocol composition作为函数参数的时候。

首先，我们回顾一个正常protocol的行为，假定定义四个protocols，P,Q，和A，B，其中A，B满足协议P和Q：
```swift
protocol P { }
protocol Q { }
 
protocol A: P, Q { }
protocol B: P, Q { }
```
虽然A和B的功能相同，但是他们却是两个不同的protocols，这意味着我们可以写两个函数，一个把A作为参数，一个把B作为参数，这两个函数是不同的：
```swift
struct S: A, B { }
let s = S()
 
func f(a: A) {
    print("A")
}
 
func f(b: B) {
    print("B")
}
 
// error: Ambiguous use of 'f'
f(s)
```
但是如果我们使用protocol<P,Q>代替B，我们就会得到不同的结果：
```swift
func g(a: A) {
    print("A")
}
 
func g(p: protocol<P,Q>) {
    print("protocol<P,Q>")
}
 
// prints "A"
g(s)
```
如果 protocol<P,Q>真的是声明了一个匿名协议满足P和Q，则我们应该会得到和上面相同的错误。相反，protocol<P,Q>的意思像是说“一个参数同时满足P和Q”。

这里选择参数为A的g函数，因为A比P和Q更具体，这和我们上面总结的经验"Swift总是挑选最"具体"的函数"是一致的。

另外一个更具体的情况是更多的协议，下面的例子说明了三个协议比两个协议更具体：
```swift
protocol R { }
extension S: R { }
 
// you can assign an alias for any
// protocol<> definition if you prefer
typealias Two = protocol<P,Q>
typealias Three = protocol<P,Q,R>
 
func h(p: Two) {
    print("Two")
}
 
func h(p: Three) {
    print("Three")
}
 
// prints "Three"
h(s)
 
// you can still force the other to be
// called if you want: this prints "Two"
h(s as Two)
```
最后，如果要在继承协议和多个协议之间选择(即广度和深度)，Swift就无能为力了：
```swift
func o(p: protocol<A,B>) {
    print("A and B")
}
 
func o(p: protocol<P,Q,R>) {
    print("P, Q and R")
}
 
// error: Ambiguous use of 'o'
o(s)
 
// prints "A and B"
o(s as protocol<A,B>)
// prints "P, Q and R"
o(s as protocol<P,Q,R>)
```

因此，对于最开始的问题，我们知道返回Range的原因了，因为协议FrowardIndexTyep比Comparable更具体，所以在没有声明具体类型的时候，编译器就帮我们选择了这个。



参考文章 [1](http://airspeedvelocity.net/2014/09/20/which-func/)[2](http://airspeedvelocity.net/2014/09/24/which-function-does-swift-call-part-2-single-arguments/)[3](http://airspeedvelocity.net/2014/10/07/which-function-does-swift-call-part-3-protocol-composition/)





