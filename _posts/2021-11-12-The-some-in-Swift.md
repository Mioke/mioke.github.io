---
title: The some in Swift
author: Klein
date: 2021-11-12 15:20:00 +0800
categories: [Tech, Swift]
tags: [Swift]
summary: 
---

> Swift 5 introduced the "Opaque Type" and see how wrong did I think about it.

## 我的思路遇到了什么问题

最先开始我以为 `some` 是一个类似于 ObjC 中 `__kindof` 的标识，意识到我的错误是在一段时间之后了。

说明这个过程和 `some` 的用法我们需要从我先入为主的乱用开始说起（开始废话了）。

我先定义个 `Flyable` 的协议，然后有 `Bird` `Dragonfly` 实现它。

```swift
protocol Flyable {}
class Bird: Flyable {}
class Dragonfly: Flyable {}
```
这时候我们有个方法可以随机返回一个可飞行生物：
```swift
func getSomeFlyableCreatures() -> Flyable {
    let someRandomCondition = arc4random() % 2 == 1
    return someRandomCondition ? Bird() : Dragonfly()
}
```
直到现在，所有功能完美运行。然后我们现在尝试在 `getSomeFlyableCreatures` 返回值类型前加个 `some`:
```swift
func getSomeFlyableCreatures() -> some Flyable {
    let someRandomCondition = arc4random() % 2 == 1
    return someRandomCondition ? Bird() : Dragonfly()
}
```
然后我们得到一个编译错误：
> Result values in '? :' expression have mismatching types 'Bird' and 'Dragonfly'

意思是返回值表达式中 `Bird` 和 `Dragonfly` 类型不匹配。我们只能返回 Bird 或 Dragonfly 其中类型之一。

## 分析
so，现在我们需要分析为什么会又这个错误，以及从此分析编译器的逻辑。

当返回值不加 `some` 时，我们的返回值类型时确定的 Protocol 类型 `Flyable`，所以此时不论是返回 Bird 还是 Dragonfly 编译器通过协议推断都是符合的，所以是可以通过编译的。

当我们添加 `some` 之后为什么不行了呢？通过刚才的结果，我们只能返回其中之一的确定类型，说明编译器在处理 `some` 时，会自动推断方法内返回的类型，并把它替换到返回类型上。

有点点类似范型方法（可以这么理解，但是实际不一样）
```Swift
func getSomeFlyableCreatures<T: Flyable>() -> T
```
所以在添加 `some` 之后方法体里返回的必须是固定的一种类型。

## 问题更进一步
现在我们给 `Flyable` 加一个定义，定义翅膀材质。
```swift
class Muscle { }
class Cutin { }

protocol Flyable {
    associatedtype wingsMaterial
}

class Bird: Flyable {
    typealias wingsMaterial = Muscle
}

class Dragonfly: Flyable {
    typealias wingsMaterial = Cutin
}
```
然后呢，我们的 `getSomeFlyableCreatures` 方法就报错了。
> Protocol 'Flyable' can only be used as a generic constraint because it has Self or associated type requirements

这个报错原因大家应该都不陌生，因为方法参数、返回值中的类型必须是确定的类型，在一个 protocol 中包含 associated type 时，其实编译器时无法确定具体类型的。

以前我们唯一能做的就是把这个方法改成范型方法，并且放弃使用 `Flyable` 这个协议类型。但是我们现在可以试着在返回值前面添加 `some` 来看这个方法行为变成了什么。

```swift
func getSomeFlyableCreatures() -> some Flyable {
    return Bird()
}
```

在修改方法体返回值为统一的类型后，可以编译通过了～但是实际上这个方法表现等同于
```swift
func getSomeFlyableCreatures() -> Bird {}
```
与之前我们分析的编译器逻辑相符。

## Then
分析了这么多，其实为之前遇到的 protocol 中的 associated type 问题并没有提供什么帮助。

`some` 更多的还是类似于 SwiftUI 中的 `some View`，提供了 Opaque Type 的类型推断，方便代码书写和隐藏不关键的类信息，详细可参考这篇[提案](https://github.com/apple/swift-evolution/blob/main/proposals/0244-opaque-result-types.md)。

## Final
关于类型推断，我个人觉得目前 Swift 有个 bug。。
```Swift
protocol Flyable {}
class Bird: Flyable {}

protocol Builder {
    associatedtype Result: Flyable
    func build() -> Result
}

class NatureBuilder: Builder {
    func build() -> some Flyable {
        return Bird()
    }
    
    func bar() -> Result {
        return Bird() // Error: Cannot convert return expression of type 'Bird' to return type 'some Flyable'
    }
}
```
按正常语言逻辑来推断的话，在 `build()` 方法返回 `Bird()` 的时候就相当于编译器已知道 `Builder.Ressult` = `Bird`。

但是在其他方法返回值类型为 Result 的方法中，不论返回什么都会报错。当然在上面的例子上我们可以把 `bar()` 的返回值改成 Bird 以规避问题。

在范型类中同样也有这个问题。

```Swift
class NatureBuilder<T: Flyable>: Builder {
    func build() -> some Flyable {
        return T()
    }
    
    func bar() -> Result {
        return T() // Error:...
    }
}
```
我们同样需要把 `bar()` 的返回值改成 `T` 才可以，换句话说这种情况下 `Builder.Result` 只能于内部或相关方法了，不能扩展其他方法。这应该算个 bug 吧lol？
