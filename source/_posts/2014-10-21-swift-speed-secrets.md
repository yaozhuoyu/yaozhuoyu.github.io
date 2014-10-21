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

### More Intelligent Method Calls

不同的方法调用语法可以在速度上比dispatch技术有一个改进。因为编译器比我们更清楚控制流程，其可以很好的优化代码。它可以将方法line调用，甚至完全消除方法调用。

在上面的例子中，如果删除testmethod2方法体：
```
@final func testmethod2() {}
```
编译器是足够聪明的，当它看到调用的testmethod2方法什么也没有做时，会优化掉对方法testmethod2的调用。

如果有足够的本地信息，即使方法没有被标记为@final，编译器也有可能会优化掉对方法testmethod2的调用。例如下面的代码：
```
let obj = Class()
obj.testmethod1()
obj.testmethod2()
```
因为编译器可以看到obj对象的创建，知道其是Class的实例，而不是其子类，这舍得编译器能完全消除dynamic dispatch，对于方法testmethod1直接调用其实现。

考虑一个极端情况，如下代码：
```
for i in 0..1000000 {
    obj.testmethod2()
}
```
在Objective-C中，这段代码将发送一百万的消息。在Swift中，编译器会看到方法testmethod2什么都没有做，可以优化掉此循环，意味着此代码一次都不会运行。

### Less Memory Allocation

如果编译器知道足够多的信息，它可以优化掉一些不必要的内存分配。在上面的例子中，编译器看到对象的创建和使用从来没有超出局部作用域，编译器会有heap allocation代替stack allocation，因为stack上分配内存更快。在极端的情况下，如果一个对象方法的调用没有使用此对象(the methods invoked on the object never actually use the object)，编译器会直接调过allocation。考虑下面的Objective-C代码：
```
for(int i = 0; i < 1000000; i++)
        [[[NSObject alloc] init] self];
```
这将发送300万次的消息，分配和销毁100万个对象。如果是Swift代码，编译器编译之后什么都会没有，因为编译器确定self方法不会做任何有用的事情，并且生成的对象在出了作用域外都没有使用到。

### More Efficient Register Usage

每一个Objective-C方法都会带有两个隐性的参数：self和_cmd。对于大多数架构（包括x86-64的，ARM和ARM64），函数的最初几个参数在寄存器中传递，其余的通过在堆栈中。寄存器是比堆要快得多，所以在寄存器传递参数可以有一个很大的性能优势。

隐性参数_cmd很少需要，当完全采用动态消息发送的时候才会用到，而99%的OC代码是不会用到的。但是在每次函数调用的时候，_cmd都要占用寄存器，剩下很少的寄存器可以使用。ARM仅使用四个寄存器传递参数，x86-64的使用六个，而ARM64采用8。

Swift省略了参数_cmd，从而释放了一个额外的参数寄存器，用于更有用的目的。对于需要多个参数（三个或更多个的ARM，五个或更多的x86-64的，和七个或更多的ARM64）的方法，这意味着在每次函数调用的时候因为多一个参数寄存器使用从而能有一个小的性能优势。

### Aliasing

上面都是Swift比Objective-C运行速度有优势的原因，但是C语言呢？Aliasing可能是一个Swift超越C语言的方面。

Aliasing是你有多个指针指向同一块内存，例如：
```
int *ptrA = malloc(100 * sizeof(*ptrA));
int *ptrB = ptrA;
```
这是一个棘手的情况，因为对ptrA的修改会影响到ptrB的读取，反之也一样。这会影响到编译器的优化能力。

考虑下面标准库函数memcpy的实现：
```
 void *mikememcpy(void *dst, const void *src, size_t n) {
  char *dstBytes = dst;
  const char *srcBytes = src;
  
  for(size_t i = 0; i < n; i++)
    dstBytes[i] = srcBytes[i];
  
  return dst;
}
```
每次都copy一个字节，效率是非常低效的，我们通常想要copy更大的块。SIMD指令允许每次copy 16或者32字节，会使其速度变快。理论上，编译器应该能够分析这个循环，并为我们这样做的。然而，因为Aliasing，编译器停止优化。要理解为什么，考虑下面的代码：
```
char *src = strdup("hello, world");
char *dst = src + 1;
mikememcpy(dst, src, strlen(dst));
```
对于函数memcpy，这样做是不合法的。其不允许传递重叠指针。然而，这里是方法mikememcpy，当其接受重叠指针的时候，行为会变得异常。

在第一次循环的时候，srcBytes[i]为h，而dstBytes[i]为e，在第一个copy之后，字符串为"hhllo, world"，第二次循环之后，字符串变为"hhhlo, world"，继续重复这个过程，最后结果为"hhhhhhhhhhhh"。

这就是memcpy不允许使用重叠指针的原因。memmove函数是足够只能的，可以处理想这样的重叠指针情况，但是是以牺牲性能为代价的。通过忽略重叠指针，memcpy的处理速度会更快。

但是，编译器并不知道这些场景，他不知道mikememcpy函数假定参数不重叠。因此编译器很难去优化。

clang在对于这个函数的时候做了一些令人欣喜的工作。其在生成代码的时候会去检查参数指针是否重叠，如果没有重叠会进行代码优化。此举将编译器缺少上下文环境的造成的性能损失大大降低，但并不是完全没有损失。

C语言中这是一个普遍的问题，大多数代码写的时候假定指针没有别名(alias)，但是编译器生成的代码假定我们可能有，这就使得C代码很难去优化。

在C99中，引入了新的关键字restrict，关键字restrict只用于限定指针；该关键字用于告知编译器，所有修改该指针所指向内容的操作全部都是基于(base on)该指针的，即不存在其它进行修改操作的途径；这样的后果是帮助编译器进行更好的代码优化，生成更有效率的汇编代码。

举个简单的例子:
```
int foo (int* x, int* y)
...{
*x = 0;
*y = 1;
return *x;
}
```
很显然函数foo()的返回值是0，除非参数x和y的值相同。可以想象，99％的情况下该函数都会返回0而不是1。然而编译起必须保证生成100%正确的代码，因此，编译器不能将原有代码替换成下面的更优版本
```
int f (int* x, int* y)
...{
*x = 0;
*y = 1;
return 0;
}
```
现在我们有了restrict这个关键字，就可以利用它来帮助编译器安全的进行代码优化了
```
int f (int *restrict x, int *restrict y)
...{
*x = 0;
*y = 1;
return *x;
}
```
此时，由于指针 x 是修改x的唯一途径，编译起可以确认 “*y=1; ”这行代码不会修改x的内容，因此可以安全的优化为
```
int f (int *restrict x, int *restrict y)
...{
*x = 0;
*y = 1;
return 0;
}
```

在Swift代码中，在一般情况下，你不采取引用任意实例变量和数组的语义使他们无法重叠。这能让Swift编译器可以更好的为我们优化代码，减少过多的载入和存储，从而优化性能。


本文主要翻译自[Secrets of Swift's Speed](https://mikeash.com/pyblog/friday-qa-2014-07-04-secrets-of-swifts-speed.html)，同时也参考了博文[restrict关键字用法 ](http://blog.chinaunix.net/uid-22197900-id-359209.html)
