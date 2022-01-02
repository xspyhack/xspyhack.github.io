---
title: ∃ Existential
date: 2019-06-05 20:00:00
tags:
    - swift
    - existential
    - protocol
    - generic
    - pats
categories: advanced swift
---
最初看到 **existential** 这个词，是在 [Swift Associated Types, cont.](http://www.russbishop.net/swift-associated-types-cont) 这篇文章中（第一次知道 `typealias Any = protocol<>` 也是在这篇文章），但当时并没有对这个陌生名称留下什么深刻印象。

后来陆陆续续应该也看到过一些，比如 [Understanding Swift Performance - WWDC 2016](https://developer.apple.com/videos/play/wwdc2016/416/)。

真正的想要去了解它，是之前写 [PATs](http://blessingsoft.com/2018/08/26/pats/) 的时候，那时候 **existential** 这个词很高频的出现，甚至是和 PATs 息息相关，所以进行了一个初步的了解。

但是呢，只是片面的了解，而没有建立起立体的认识，记忆很快就会开始模糊，直到最后忘记掉。所以才有了这次的 **existential** 认知之旅。

> 注：这篇大多数概念、观点、片段都来自于官方文档或者参考文章，小部分自己的认知理解，并且不保证理解的正确。

## Existential Type

想理解 **existential**，必须要先了解 **existential values**, **existential containers** 和 **witness tables** 的概念。

在类型论中， [existential type](https://en.wikipedia.org/wiki/Type_system#Existential_types) 描述了抽象类型的接口。当对象的类型是 `protocol` 时，就会用到 **existential type**，因为存储或传递一个 `protocol` 类型的对象意味着对象在运行时的真实类型是不透明的（也就是编译期不可知的，因此我们也无法确定这类对象的布局）。

一个遵从了特定 `protocol` 的类型一定包含其约定的所有方法，但是这些方法的地址是无法在编译期确定的，因为我们只有在运行时，才能确定这个 `protocol` 对应的真实类型。这和 `non-final class` 引用是类似的（因为可能被 override），因此也使用了 [类似的技术手段](https://github.com/apple/swift/blob/master/docs/ABIStabilityManifesto.md#method-dispatch) 来解决。`Protocol` 中约定的每一个被实现的方法的地址，都被保存在 **witness table** 中。

### Existential Value

显然的，**existential type** 的值，就是 **existential value**。:P

### Existential Container

```swift
protocol Foo {}
let foos: [Foo] = ... // What's the memory storage looks like?
```

简单来说 **existential of a protocol** 就是一个编译器生成的盒子 box，用来存放遵从这个 `protocol` 的值，这个盒子，也叫做 **Existential Container**，盒子里面的东西，就叫做 **witness**。

关于 **existential container** 的内存布局，这些值怎么存储，要分几种情况来说。因为值类型和引用类型的处理方式不一样，值比较小和比较大也可以为了性能用不同的策略。[Understanding Swift Performance - WWDC 2016](https://developer.apple.com/videos/play/wwdc2016/416/) 和 [ABIStabilityManifesto · GitHub](https://github.com/apple/swift/blob/master/docs/ABIStabilityManifesto.md) 中都有详细的描述。

简单来说就是 5 个字节的大小，前三个连续的字节叫做 *value buffer*，用来存放对象的值或者指针。值类型如果放得下，就直接内联放在 *value buffer* 里面，如果放不下，就在存放在堆上，把指针地址存放在 *value buffer* 里；引用类型直接放指针。
第四个字节存放 `vwt` (*value witness table*) 指针。
第五个字节存放 `pwt` (*protocol witness table*) 指针。

对于那些限定了只能是 `class` 实现的 `protocol`，`containers` 中则会忽略 `vwt` 指针（因为对象自身包含指向自己类型信息的指针）以及多余的内连 buffer。并且，这里还有一个特例 `Any`，由于它没有遵从任何 `protocol`，因此 `Any` 对象的 `containers` 中没有 *witness table* 指针。（没错，`Any` 也是一个 **existential** ！即使 Swift 3 之后把 `Any` 当作了 keyword，但估计和之前的 `protocol <>` 差不多的实现，所以依然是 **existential** 。）

### VWT

每一个具体类型（concrete type）都有一张 *value witness table*，用来存放这个类型的有关内存布局和操作它的值的信息。当一个值类型具有不透明布局的时候，因为值编译的时候没办法知道实际类型，所以只能通过查询这个表来知道这些有关信息（metadata）。

### PWT

*Protocol witness table* 是 `protocol` 接口的一张函数表。如果有 associated type，它还会存储 associated type 的 metadata。

> 所以什么是 **existential** 是什么？就是一个 `protocol` 类型的值。

## Protocol

当一个 `protocol` 作为类型而不是具体的类型约束的时候，它就是一个 **existential**。

```swift
protocol Foo {}

func bar<T: Foo>(_ foo: T) {} // This requires a concrete T that conforms to Foo
func baz(_ foo: Foo) {} // This requires a variable of type Foo (pedantically: "a Foo existential")

let foo: Foo = ... // existential of protocol `Foo`
bar(foo) // 😢 Protocol type 'Foo' cannot conform to 'Foo' because only concrete types can conform to protocols
baz(foo) // 😊
```

所以当看到 "a protocol doesn't conform to itself" 的时候，它实际上是指 "the existential of a protocol doesn't conform to that protocol"。 

## Generic

**Existentials** 不是真正的泛型（`generic`），但由于它们相互依赖于 `protocol`，这两个系统紧密地交织在一起。

> While protocols create existential ("there exists") types, generics create universal ("for all") types. 

先回顾一下泛型的一些常见概念：

* 泛型函数 `func swap<T>(_ a: inout T, _ b: inout T)`
* 类型参数 `<T>`
* 泛型类型 `Queue<T>`
* 类型约束 `<T: Protocol, U: Class>`
* 关联类型 `associatedtype T`
* 泛型从句 `func foo<T: P, U: P>(_ a: T, _ b: T) where T: Equatable, T.Item == U.Item`

当使用泛型作为类型约束的时候，会涉及到 **existentials**。

```swift
func bar<T: Foo>(_ foo: T) {}
let foo: Foo = ...
bar(foo) // Protocol type 'Foo' cannot conform to 'Foo' because only concrete types can conform to protocols
```

## PATs
既然有 **existentials** 了，那为什么还需要 **type eraser** 呢？

先回过头来看看之前在 [PATs](http://blessingsoft.com/2018/08/26/pats/) 中遇到的几个问题。

> Protocol 'Caching' can only be used as generic constraint because it has Self or associated type requirements

它的意思是，`Caching`  这个 PATs 没有（无法自动生成）一个 **existential**.

> Using 'Logging' as a concrete type conforming to protocol 'Hashable' is no supported

它的意思是，这个 `Logging` 的 **existential** 没有实现 `Hashable` 这个协议。

为什么无法为 PATs 生成一个 **existential** 呢？实际上是可以的，但它很复杂。它可以通过一种叫做 **generalized existentials** 的技术，生成一个 **implicit existential**。即使这样，它还有很多问题需要解决。

对于 **existential** 的自动生成，首先 **existential** 是运行时的（泛型 `generic` 是编译时的），它是通过在运行时，把 `protocol` 的一些信息存放在 **existential container** 里面。当 `protocol` 里面存在有 *associated types* 或者有 `Self` 约束的时候，它没办法针对任意类型（Any）自动生成填充这个 **existential container**。（Swift 是静态语言，对于泛型需要在编译时就进行泛型特化，**generic specialization**，除非把泛型当作是 `Any` 来处理。还有一种方式就是对 PATs 进行约束，`let strings: Any<Sequence where .Iterator.Element == String> = ["a", "b", "c"]`，也就是 `AnySequence<String>` ）

理解这一点非常重要，可能会有点晕，再来捋一下。首先编译器把存储或者传递的  `protocol` 类型，先替换成 `existential container`（生成代码），然后再编译成目标代码。当编译器发现这个 `protocol` 是 PATs 时，它如果不通过 *generic specialization* 的话，无法生成不带泛型的代码。那为什么说 **existential** 是运行时的呢？因为存储或传递一个 `protocol` 类型的对象意味着对象在运行时的真实类型是不透明的（也就是编译期不可知的，因此我们也无法确定这类对象的布局）。

还有一些类型是不适合自动生成 **existential** 的，编译器没法满足有 `init` 和 `static` 的要求。比如 `Decodable` 这样的没有实例方法的协议，**existential** 没有任何意义。

```swift
public protocol Decodable {
    init(from decoder: Decoder) throws
}

struct Model: Decodable {
    let x: String
}

func decode(_ decodable: Decodable) {
}
let decodable: Decodable = Model(x: "x")
decode(decodable)
// 上面对代码编译起来没有任何问题，也就是能自动生成 *existential*
// 但对于 decode(_:) 函数，根本无从下手，因为 Decoder 需要的是一个遵守 Decodable 协议的类型，而不是值。

func decode(_ type: Decodable.Type) {
    let decodable = JSONDecoder().decode(type, from: data)
    // let decodable = JSONDecoder().decode(Decodable.self, from: data)
    // Protocol type 'Decodable' cannot conform to 'Decodable' because only concrete types can conform to protocols
}
// 最终还是那个 `Protocol type 'Decodable' cannot conform to 'Decodable' because only concrete types can conform to protocols`
```

其实 **type eraser** 和 **existentials** 这两种是对偶的（ duals ），泛型（ generic ）的 `Any<T>` 等同于 协议（ protocol ）的一种 **explicit existential**。

## Existential in Other Language

### Existential type in Java

Java 泛型中的 **Wildcards** 其实就是一种 existential type，比如 `java.util.List<?>`。

在 Java 中由于有[类型擦除](h) 的存在，泛型的参数类型信息在运行时会丢失，在运行时无法根据已知的类型信息区分 `List[Int]` 和 `List[String]`。

```java
List foo = new ArrayList();
foo.add("foo");
foo.get(0); // "foo"
```

当没有给出类型参数的时候，通过使用 existential 来解决。

```java
List<?>
List<? extends Number>
List<? super Integer>
```

### Existential type in Kotlin

Kotlin 中没有 **existential type**。它有一个概念叫着 **The Existential** 的概念。

```kotlin
abstract class Animal()
class Dog()
class Bar<T> {
}
var dogBar: Bar<Dog> = Bar()
var animalBar: Bar<Animal> = dogBar // 😢
```

```kotlin
class Bar<out T> {
}
var dogBar: Bar<Dog> = Bar()
var animalBar: Bar<Animal> = dogBar // 😊
```

### Existential type in Scala

`ArrayList() == List[]`

```scala
object Trait {
  def foo(seq: Seq[String]): Seq[String]
  def foo(seq: Seq[Int]): Seq[Int]
}
// Error, have the same type after erasure

List[_] // List[T] forSome { type T }
trait List[+T]
```

### Existential type in Rust

 `fn foo() -> impl Trait`

核心在于 `impl Trait`，和 Swift 5.1 [Opaque Result Types](https://github.com/apple/swift-evolution/blob/master/proposals/0244-opaque-result-types.md) 中的 `func foo() -> some P` 一样。

## Conclusion

**Existential** 是什么？**Existential** 就是 `protocol` 类型的值。这是编译层面相关的概念，平时写代码不需要知道它意味着什么或者是什么，只需要知道它会跟你想象中一样 work 就行了。

## Ref

[Swift Associated Types, cont. - Russ Bishop](http://www.russbishop.net/swift-associated-types-cont)
[ABIStabilityManifesto · GitHub](https://github.com/apple/swift/blob/master/docs/ABIStabilityManifesto.md)
[GenericsManifesto · GitHub](https://github.com/apple/swift/blob/master/docs/GenericsManifesto.md)
[Improving the UI of generics - Swift Forums](https://forums.swift.org/t/improving-the-ui-of-generics/22814)
[Protocols III: Existential Spelling - Cocoaphony](http://robnapier.net/existential-spelling)
[Existential types - Wikipedia](https://en.wikipedia.org/wiki/Type_system#Existential_types)
[Understanding Swift Performance - WWDC 2016](https://developer.apple.com/videos/play/wwdc2016/416/)
[0244-opaque-result-types - GitHub](https://github.com/apple/swift-evolution/blob/master/proposals/0244-opaque-result-types.md)