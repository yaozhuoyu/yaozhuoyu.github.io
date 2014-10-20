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

























http://get.jobdeer.com/6009.get

https://developer.apple.com/library/ios/documentation/Swift/Conceptual/Swift_Programming_Language/Initialization.html#//apple_ref/doc/uid/TP40014097-CH18-XID_339