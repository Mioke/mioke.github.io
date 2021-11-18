---
title: Compare between RxSwift and ReactiveSwift
author: Klein
date: 2021-11-10 18:32:00 +0800
categories: [Tech, FRP]
tags: [FRP, Swift, ReactiveSwift]
summary: The different part between RxSwift and ReactiveSwift, and learn how to understand them.
---

If you asking me which framework should we prefer to use now, my answer is `RxSwift`🙈.


1. RxSwift 社区更为活跃，并且 RxSwift 实现的是 Rx 通用接口（加上一点点🤏语言特性功能），所以更容易找到文档或查找到相关讨论（包括目前 RxSwift 已经开始开发 concurrency 相关的 feature 了， ReactiveSwift 一点动静都没有。。）。
1. ReactiveSwift 冷热信号在一些细节上比较难以理解，比如 signal.producer 产生的 producer 和 signal 似乎没什么差别。
1. RxSwift 的订阅方法返回值 disposable 是不能默认忽略的，所以更强调了生命周期管理。
1. ReactiveSwift 没有 share、replay 方法，想在订阅的时候拿到之前的值目前我个人会把它从 `Signal` 换成用 `Property`，再用 `Property.producer` 来获取到最近的一个值。一点点奇怪🤏，可能是我没掌握真正的方式😂。
1. RxSwift 中 `flatMap` 和 `flatMapLatest` 定义跟 ReactiveSwift 中的 `flatMap(strategy:)` 设计角度似乎完全不一样，目前我也没法说哪个设计的更好，不过 ReactiveSwift 中的似乎更好理解和简单。 RxSwift demo 中的 `flatMap & flatMapLaest` 的例子设计都很复杂，不过看起来更加“响应式”。

```swift
// ReactSwift re-write RxSwift flatMap demo
scopedExample("signal map signal") {
    struct People {
        let age: MutableProperty<Int>
        
        init(age: Int) {
            self.age = MutableProperty(wrappedValue: age)
        }
    }
    
    let (people, pInput) = Signal<People, Never>.pipe()
    let some = People(age: 10)
    pInput.send(value: some)
    
    people.flatMap(.latest) { p in
        return p.age.producer
    }.observeValues { age in
        print(age)
    }

    some.age.value = 11 // here won't print, different from RxSwift.
    
    let some2 = People(age: 20)
    pInput.send(value: some2)
    some2.age.value = 22
}
```
