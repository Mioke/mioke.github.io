---
layout:     post
title:      "Codable introduced in Swift 4.0"
date:       2018-06-13 10:55 +0800
categories: Tech
summary:    Codable 在 Swift 4.0 中的用法和事项。
---

### Swift Codable 

`Codable` 其实是组合协议：

```swift
typealiase Codable = Encodable & Decodable
```

Swift 基础库中的 `Codable`、`JSONEncoder`、`PropertiesListEncoder` 给我们提供了日常所用的基础功能。使用起来也很简单，以 JSONEncoder 为例。

```swift
struct Cat: Codable {
     let name: String
     var age: Int
}

let sahra = Cat(with: "Sahra", age: 1)
if let result = try? JSONEncoder.encode(aCat) {
    let stringValue = String.init(data: result, encoding: .utf8)
    print(stringValue)
}

// Output:
"{"name":"Sahra","age":1}"
```

### 专用容器

Codable 的默认实现是把当前类型中的变量都默认转换，如果你定义的类型中，要含有一些其他属性，而不希望这些属性自动转换，我们可以定义一个容器来使用。

```swift
struct CatContainer {
    let cat: Cat
    init(with cat: Cat) { self.cat = cat }
    
    // Other properties
    var storedID: String = ""
}
```

依实际使用的情况而定，也可以使用泛型的容器。

```swift
struct Container<T: Codable> {
    let element: T
    init(with element: T) { self.element = element }
}
```

### 重定义 CodingKeys

我们可以用另外一种方法忽略部分属性或者改变编码或解码对应的 key 值。在结构体或类中定义枚举类型 `CodingKeys`，将需要编码的属性包含其中。如果需要改变编码后的命名，则用枚举 `String` raw value 来表示相应属性。普通情况下，用 `Int` 类型会节省开销。

```swift
struct Cat: Codable {
    let name: String
    var age: Int = 0 // 如果没有初始值，编译器会报错。
     
    enum CodingKeys: String, CodingKey {
        case name = "pet_name"
    }
}

// Output:
"{"pet_name":"Sahra"}"
```

`CodingKeys` 中的元素名必须与属性名一一对应。除此之外，还可以更进一步自定义编解码动作，比如 nested 型结构的编解码，可以在这篇[官方文档](https://developer.apple.com/documentation/foundation/archives_and_serialization/encoding_and_decoding_custom_types)中找到更多信息。

### Tips

- Optional 类型

系统默认对于 Optional 的实现是，如果为 nil 的情况下，不会将该属性编码。如果你的业务方需要在属性为空时，JSON 对象中表示 `null`的话，需要定义 CodingKeys 并指定含有这个属性即可。

- 关于 `[Hashable: Any]` 类型的编解码

对于代码迁移的工程来说，有很多部分代码还使用例如 `[String: Any]` 的字典类型来传递数据。`Any` 并不符合 Codable 协议，所以是时候使用 `[Hashable: Codable]` 重构这部分代码了:)

### 问题

- [ ] 没有找到 Codable 的协议默认实现，却可以不用实现协议方法。
- [ ] `CodingKeys` 的实现原理，结合编译信息，猜测是编译时期工作。通过代码及注释信息，其实是分辨不出来 `CodingKeys` 的用法的，需要参考官方文档。个人觉得这是缺点之一，一是觉得实现原理不透明，不太容易理解；二是没有显式说明 Codable 和 CodingKeys 的关系。比如类型中需要编解码的属性其实编译时期都会校验是否都是 Codable 的，这点在编码的时候是体现不出来的。

---

