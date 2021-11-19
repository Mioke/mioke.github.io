---
layout: post
title:  "Learning ReactiveSwift - Part 2"
date:   2017-07-04 20:00:00 +0800
categories: [Tech, FRP]
tags:       [Swift, FRP, ReactiveSwift]
summary: The second part of learning ReacitveSwift.
---

### 数据流
FRP之前一篇已经介绍过，是一种抽象的数据流型的编程思想。对于数据流，开始一个数据流后，会产生很多事件。在ReactiveSwift中，定义的事件有Event、Error。对应Observer则会有observe value、failed、complete、interrupted。

对于冷信号：
```swift
let observer = Signal<Int, NSError>.Observer(
    value: { print("value: \($0)") },
    failed: { print("failed with \($0)")},
    completed: { print("complete") },
    interrupted: { print("interupted") }
)

SignalProducer<Int, NSError>([1,2,3,4]).start(observer)
/*
print:
value: 1
value: 2
value: 3
value: 4
complete
 */

 // or

let producer = SignalProducer<Int, NSError>([1, 2, 3, 4])

producer.startWithSignal { (signal, disposable) in
    signal.observe({ (event) in
        switch event {
        case .value(let value):
            print("value: \(value)")
        case .failed(let error):
            print("failed with \(error)")
        case .interrupted:
            print("interrupted")
        case .completed:
            print("completed")
        }
    })
}
```
对于热信号：

```swift
let (signal, observer) = Signal<String, TestError>.pipe()
signal.observe { event in 
	 switch event {
        case .value(let value):
            print("value: \(value)")
        case .failed(let error):
            print("failed with \(error)")
        case .interrupted:
            print("interrupted")
        case .completed:
            print("completed")
    }
}
```
需要注意的是，在signal接收到failed, interrupted, complete事件后，该通道就相当于关闭了，之后的事件则不会再接收到。

### Map

`map`就像一般函数式编程中map用法一样，对sequence内对象进行transform，如下面例子，对数组中的数进行乘2：

```swift
SignalProducer<Int, NoError>([1, 2, 3, 4])
    .map { $0 * 2 }
    .startWithValues { print($0) }
// print: 
// 2 4 6 8
```
`mapError`是对failed情况进行transform：

```swift
enum TestError: Error {
    case code(Int)
}

SignalProducer<Int, NSError>(error: NSError(domain: "test.domain", code: 100, userInfo: nil))
    .mapError { return TestError.code($0.code) }
    .startWithFailed { (error) in
    switch error {
    case let .code(code):
        print(code)
    }
}

// print: 100
```

`flatMap`是对SignalProducer进行转换

```swift
SignalProducer<Int, NoError>([ 1, 2, 3, 4 ])
    .flatMap(.latest) { SignalProducer(value: $0 + 3) }
    .startWithValues { value in
        print(value)
}
/*
print:
4
5
6
7
*/
```

### Filter

对Value进行过滤，只有符合条件的value才能触发事件:

```swift
SignalProducer<Int, NoError>([ 1, 2, 3, 4 ])
	.filter { $0 > 3 }
	.startWithValues { value in
		print(value)
	}
// print: 4
```

### Take
取事件的前N项（take(first:)）或后N项(take(last:))

```swift 
let producer = SignalProducer<Int, NSError>([1, 2, 3, 4])
producer
    .take(last: 2)
    .startWithResult { print($0) }
// .success(3)
// .success(4)
```

### Combine

`combineLatest`是把第一个producer的最后一项与第二个producer的value进行绑定：

```swift
let producer1 = SignalProducer<Int, NoError>([ 1, 2, 3, 4 ])
let producer2 = SignalProducer<Int, NoError>([ 1, 2 ])

producer1
    .combineLatest(with: producer2)
    .startWithValues { value in
        print("\(value)")
}
// print:
// (4, 1)
// (4, 2)
```

`combinePrevious`是每次value都绑定前一次event的value
```swift
SignalProducer<Int, NoError>([ 1, 2, 3, 4 ])
    .combinePrevious(42)
    .startWithValues { value in
        print("\(value)")
}
/* 
print :
(42, 1)
(1, 2)
(2, 3)
(3, 4)
*/
```

### 小结

对于SignalProducer和Signal，还有很多方便的小工具对其进行操作或者转换，在实际使用过程中根据需求的不同进行使用。
