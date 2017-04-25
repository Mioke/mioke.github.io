---
layout: post
title:  "Learning ReactiveSwift - Part 1"
date:   2017-04-24 19:18:00 +0800
categories: Tech
summary: The first part of learning ReacitveSwift, knowledge of basic usages.
---
# Learning ReactiveSwift - Part 1

> RAC 4.0发布以后发现，RAC进行了大量的更新，其中最主要的是使用了ReactiveSwift作为基础库，替换掉了以前OC版的库，RAC则是对于RAS和Cocoa框架的封装。

## 入门
对于以前的RAC教程，理论部分大部分还是通用的，但是ReactiveSwift在某些方面进行了重构和修改，而且现有教程缺乏更新，不得已我们重头来梳理遍ReactiveSwift的入门教程。

ReactiveSwift的[文档](http://reactivecocoa.io/reactiveswift/docs/latest/)是一篇不错的入门介绍，本文部分直接翻译自原文。

## 什么是ReactiveSwift
ReactiveSwift提供了可组合的、宣言式的、灵活的基元，用来构建**数据流（streams of values）**，这种形式是Reative Programming的重要表现之一。ReactiveSwift使用的是Functional Reactive Programming(FRP)，FRP是两种概念的结合，Reactive Programming和Functional Programming。

Reactive Programming是专注于监听异步数据流，并且可以作出相应反馈的一种编程方式。可以查看[这篇文章](https://gist.github.com/staltz/868e7e9bc2a7b8c1f754)的详细介绍。

Functional Programming强调使用数学模型的函数来计算，特点是不可变、而又表达明确清晰，最小化使用变量和状态。

## 理解
先来看看ReactiveSwift的几个重要基元：Signal, SignalProducer, Event, Observer, Action.
当一个对象持有一个Signal，可以看做，一家电视台有在某个频道的电视节目，Signal就是这个频道（通道）。Event，可以看做是这个频道上的数据。Observer则是电视观众，他们订阅（subscribe或observe）了这个频道（signal)，则可以从频道中获取内容（events）。Action则是一种动作，对Signal产生影响，却不对Signal的持有者产生影响，就像：换台（sendInterrupt）。

而Signal和SignalProducer则是RAC更新中的一个变动，他俩的定义就像“冷、热信号”。在**Rx**和**RAC 2.x**以前他们是统一一起的。之所以分开，官方文档的解释是：
> … it is impossible to tell whether subscribing to (observing) that IObservable will involve side effects. If it does involve side effects, it’s also impossible to tell whether each subscription has a side effect, or if only the first one does.

> This example is contrived, but it demonstrates a real, pervasive problem that makes it extremely hard to understand Rx code (and pre-3.0 ReactiveCocoa code) at a glance.

> ReactiveSwift addresses this by distinguishing side effects with the separate Signal and SignalProducer types. Although this means there’s another type to learn about, it improves code clarity and helps communicate intent much better.

> In other words, ReactiveSwift’s changes here are simple, not easy.

简单来说，监听或订阅一个信号，可能会产生副作用，为了区分是否产生副作用，ReactiveSwift区分了两个类型。

## 简单使用ReactiveSwift

> 了解基础用法，其实可以下载ReactiveCocoa源码工程，在单元测试中可以找到API的各种用法。下面源码大部分取自单元测试代码。
### Signal

```swift
// 使用一段代码（运行）
let disposable = SimpleDisposable()
let signal: Signal<Int, NoError> = Signal { observer in
    // do something
    return disposable
}
signal.observe { event in
    switch event {
        case let .value(number):
            // do something
        case .completed:
            // do completion
        default:
            break
    }
}

// 使用pipe()
let (signal, observer) = Signal<String, TestError>.pipe()
// 同样可使用observe
signal.observe { event in 
    // blabla
}
// observer可以send events
observer.send(value: "a string")
observer.sendCompleted()
```

### SignalProducer
SignalProducer是触发式的
```swift
var disposable: Disposable!
let producer = SignalProducer<Int, NoError> { observer, innerDisposable in
    disposable = innerDisposable
    innerDisposable += {
        observer.send(value: 1)
    }
    observer.send(value: 0)
}
producer.startWithSignal { signal, disposable in
    signal.observeValues { print($0) }
}
disposable.dispose()

// print:
// 0
// 1
```
### Disposable
看到上面SignalProducer和Signal的例子，用到`Disposable`的场景。Disposable是由协议`Disposable`描述的，拥有内存管理和取消机制一种对象。

当启动一个`Signal producer`，会返回一个disposable对象，这个对象可被用作取消被启动的工作，清理所有临时资源，然后发送一个`interrupted`事件（Event）给被创建的`Signal`。

监听（Observer）一个`Signal`也会返回一个disposable对象，用来使监听者停止从`Signal`接收`events`，但是不会对`Signal`产生影响。

### Events
事件，描述一个发生的事情。在ReactiveSwift中，事件是通讯的核心部分。一些事件生成器可以通过`Signal`发送事件给注册的观察者们。

### Unidirectional Binding
单向绑定，通过一个数据源（BindingSource）绑定到一个目标（BindingTargetProvider）上。
```swift
public func <~<Provider, Source>(provider: Provider, source: Source) -> Disposable? 
    where Provider : BindingTargetProvider, Source : BindingSource, 
        Source.Error == Result.NoError, 
        Source.Value == Provider.Value
```
有定义可见，只要Source和Provider的Value是同一类型，就可以进行绑定。举个例子：
```swift
var value: Int? = nil
        
let (lifetime, _) = Lifetime.make()
let target = BindingTarget<Int>(lifetime: lifetime) { value = $0 }
let property = MutableProperty.init(1)
        
target <~ property
// value = 1
property.value = 2
// value = 2
```

> 这例子是单元测试的例子，但是我自己试的时候value一直是nil...不知道那出了问题。


### Lifetime
生命周期的监控，比如：
```swift
let object = MutableReference(TestObject())

var isEnded = false
var isCompleted = false

object.value!.ended.observeCompleted { isCompleted = true }
object.value!.lifetime.observeEnded { isEnded = true }
expect(isCompleted) == false
expect(isEnded) == false

object.value = nil
expect(isCompleted) == true
expect(isEnded) == true
```

## Take a break
这篇主要是讲一些ReactiveSwift的基础用法，下篇主要讲他的函数式部分，比如map、filter等。