title: Swift Failable Initializers
date: 2014-10-20 08:47:37
categories: Swift
tags: [Swift,Initializers]
---
在Swift中，为class,structure,enumeration定义一个可能失败的初始化函数是很有用的。初始化函数的失败可能由于无效的初始化参数，缺少所需的外部资源，或者其他一些阻止初始化成功的条件。

为了应对可能的初始化失败，我们要给class，structure，enumeration定义一个或者多个**failable initializers**,我们通过在关键字init后面添加问号(init?)来声明一个**failable initializers**。

<!-- more -->

一个**failable initializers**会创建一个可选值，当初始化失败的条件被触发的时候，会返回一个nil。

严格意义上讲，initializers是没有返回值的，其作用是确保self在初始化函数的最后被完全和正确的初始化。虽然我们通过写`return nil`去触发初始化失败，但是我们不能通过return关键字表示初始化成功。

下面是一个例子，定义了一个结构体，名为 Animal，并且定义了一个**failable initializers**，有一个名为species的参数。**failable initializers**会检查species是否为空，如果为空，则初始化失败，返回nil。
```swift
struct Animal {
    let species: String
    init?(species: String) {
        if species.isEmpty { return nil }
        self.species = species
    }
}
```
我们可以使用**failable initializers**去尝试初始化一个新的Animal实例，并检查初始化是否成功：
```swift
let someCreature = Animal(species: "Giraffe")
// someCreature is of type Animal?, not Animal
 
if let giraffe = someCreature {
    println("An animal was initialized with a species of \(giraffe.species)")
}
// prints "An animal was initialized with a species of Giraffe"
```
如果传递一个空的字符串，则会初始化失败：
```swift
let anonymousCreature = Animal(species: "")
// anonymousCreature is of type Animal?, not Animal
 
if anonymousCreature == nil {
    println("The anonymous creature could not be initialized")
}
// prints "The anonymous creature could not be initialized"
```

### Failable Initializers for Enumerations

**failable initializers**也可以应用到枚举中，我们可以通过**failable initializers**通过一个或者多个参数去选择恰当的枚举成员返回，如果通过提供的参数没有合适的枚举成员匹配，则初始化失败。

下面是一个例子，定义了一个枚举类型TemperatureUnit，可能有三个状态值，当根据symbol没有找到对应的值时，返回nil。
```swift
enum TemperatureUnit {
    case Kelvin, Celsius, Fahrenheit
    init?(symbol: Character) {
        switch symbol {
        case "K":
            self = .Kelvin
        case "C":
            self = .Celsius
        case "F":
            self = .Fahrenheit
        default:
            return nil
        }
    }
}
```
如下使用：
```swift
let fahrenheitUnit = TemperatureUnit(symbol: "F")
if fahrenheitUnit != nil {
    println("This is a defined temperature unit, so initialization succeeded.")
}
// prints "This is a defined temperature unit, so initialization succeeded."
 
let unknownUnit = TemperatureUnit(symbol: "X")
if unknownUnit == nil {
    println("This is not a defined temperature unit, so initialization failed.")
}
// prints "This is not a defined temperature unit, so initialization failed."
```

### Failable Initializers for Enumerations with Raw Values

带有raw value的枚举类型会自动的得到一个**failable initializers** --`init?(rawValue:)`，此方法有一个叫rawValue的参数，为raw-value类型的，通过传入的参数，会选择对应的枚举成员，如果没有找到，则**failable initializers**失败，返回nil。如下：
```swift
enum TemperatureUnit: Character {
    case Kelvin = "K", Celsius = "C", Fahrenheit = "F"
}
 
let fahrenheitUnit = TemperatureUnit(rawValue: "F")
if fahrenheitUnit != nil {
    println("This is a defined temperature unit, so initialization succeeded.")
}
// prints "This is a defined temperature unit, so initialization succeeded."
 
let unknownUnit = TemperatureUnit(rawValue: "X")
if unknownUnit == nil {
    println("This is not a defined temperature unit, so initialization failed.")
}
// prints "This is not a defined temperature unit, so initialization failed."
```

### Failable Initializers for Classes

对于值类型(structure 和 enumeration)，**failable initializers**可以在initializer实现的任何点触发初始化失败，返回nil。在上面的Animal类的例子中，初始化失败的触发是在实现的开始，在species属性设置之前。

对于class，**failable initializers**触发失败之前，其所有的存储型属性必须要设置一个初始值，并且初始化委托发生完成(见下一节Propagation of Initialization Failure)。例如下面的例子：
```swift
class Product {
    let name: String!
    init?(name: String) {
        if name.isEmpty { return nil }
        self.name = name
    }
}
```
Product的定义和结构体Animal的定义很相似，但是我们看到name是一个implicitly unwrapped optional string type (String!)。因为他是一个optional type，这意味着name属性在被赋值一个具体值的之前，有一个缺省值nil。这个缺省值nil表示Product类的每一个属性都有一个有效的初始值，因此，Product的**failable initializers**可以在初始化函数开始的时候就触发失败。

因为name属性是一个常量，在初始化成功之后，我们可以放心的使用name属性而不用去检查其是否为nil。
```swift
if let bowTie = Product(name: "bow tie") {
    // no need to check if bowTie.name == nil
    println("The product's name is \(bowTie.name)")
}
// prints "The product's name is bow tie"
```

### Propagation of Initialization Failure

下面的例子定义了Product的子类--CartItem，CartItem引入了一个常量属性quantity，确保此属性的值至少为1：
```swift
class CartItem: Product {
    let quantity: Int!
    init?(name: String, quantity: Int) {
        super.init(name: name)
        if quantity < 1 { return nil }
        self.quantity = quantity
    }
}
```
CartItem的**failable initializers**首先委到父类的`init(name:)`初始化函数。这满足了要求：**failable initializers**在触发初始化失败之前，其必须进行完初始化委托(其实我的理解就是必须在调用完父类的初始化方法之后)。

如果父类的初始化函数失败(name为空)，则整个初始化过程失败，立即结束，不会在执行。如果父类的初始化函数成功，CartItem会进一步验证quantity的合法性。

如果CartItem实例有一非空的name，并且quantity的值大于1，则初始化成功：
```swift
if let twoSocks = CartItem(name: "sock", quantity: 2) {
    println("Item: \(twoSocks.name), quantity: \(twoSocks.quantity)")
}
// prints "Item: sock, quantity: 2"
```
如果创建一个quantity值为0的实例，则CartItem初始化失败：
```swift
if let zeroShirts = CartItem(name: "shirt", quantity: 0) {
    println("Item: \(zeroShirts.name), quantity: \(zeroShirts.quantity)")
} else {
    println("Unable to initialize zero shirts")
}
// prints "Unable to initialize zero shirts"
```

### Overriding a Failable Initializer

和其他initialize一样，子类可以重写父类的**failable initializers**。并且，子类可以是non-failable initializer。注意如果用一个non-failable initializer来覆盖父类的failable initializer，子类实现中不能去调用父类的failable initializer。一个non-failable initializer永远都不能委托到一个failable initializer上。

下面例子定义了一个类Document：
```swift
class Document {
    var name: String?
    // this initializer creates a document with a nil name value
    init() {}
    // this initializer creates a document with a non-empty name value
    init?(name: String) {
        if name.isEmpty { return nil }
        self.name = name
    }
}
```
又定义了Document类的子类--AutomaticallyNamedDocument，用一个non-failable initializer(`init(name:)`)覆盖了父类的failable initializer(`init?(name:)`)，
```swift
class AutomaticallyNamedDocument: Document {
    override init() {
        super.init()
        self.name = "[Untitled]"
    }
    override init(name: String) {
        super.init()
        if name.isEmpty {
            self.name = "[Untitled]"
        } else {
            self.name = name
        }
    }
}
```
