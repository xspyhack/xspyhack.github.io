---
title: PATs
tags:
	- swift
	- type erasure
	- type erased
	- pats
	- protocol
	- generic
categories: advanced swift
---
> 更新：随着 Swift 中一些新的提案（如 [SE-0309](https://github.com/apple/swift-evolution/blob/main/proposals/0309-unlock-existential-types-for-all-protocols.md) 和 [SE-0335](https://github.com/apple/swift-evolution/blob/main/proposals/0335-existential-any.md)）的提出，大大的简化了 Swift 中的 Protocol 的使用，文中的一些概念或者观点抑或是写法已经显得落后而不适用。--2022.01.01

> 很久没有写 Swift 了，闲着写几行玩玩的时候，遇到了一个之前没有接触过的问题——Protocol with Associated Types

## Protocol-Oriented Programming

首先 Swift 是一个 **支持** 面向协议编程思想的语言，并且 [Standard Library](https://github.com/apple/swift) 也是大量使用这种思想来实现很多的特性。（这里删掉介绍 POP 以及对应优缺点的几千字

先看看用 POP 的思想，来实现一个 Cache：

```swift
protocol Caching {
    var name: String { get }
}

struct MemoryCache: Caching {
    var name: String {
        return "Memory Cache"
    }
}

struct DiskCache: Caching {
    var name: String {
        return "Disk Cache"
    }
}

let memory: Caching = MemoryCache() // 🙂️
let disk: Caching = DiskCache() // 🙂️
let caches: [Caching] = [memory, disk] // 🙂️
```

到这里，一切都是很熟悉的样子，也能正常的 work，类似这样的**用一个 *protocol* 来进行接口（行为）约束**的代码估计写的也不少。

## Generics

> 作为一个现代语言，泛型是必须支持的。作为一个现代程序员，泛型也是必须会使用的。

Cache  是用来缓存数据的，但有很多数据类型都适合被缓存，比如一张图片，一个视频等等。如果希望不同类型的数据有不同的缓存策略，使用的时候也能直接获取某一种类型的缓存，不需要各种 `as? XXX`，立马想到的就是泛型。

泛型还不简单：

```swift
protocol Caching {
    associatedtype Object

    func store(_ object: Object, forKey key: String)
    func retrieve(forKey key: String) -> Object?
}

struct MemoryCache: Caching {
    typealias Object = UIImage

    func store(_ object: Object, forKey key: String) {
        //
    }
    func retrieve(forKey key: String) -> Object? {
        //
    }
}

struct DiskCache: Caching {
    typealias Object = UIImage

    func store(_ object: Object, forKey key: String) {
        //
    }
    func retrieve(forKey key: String) -> Object? {
        //
    }
}

let memory: Caching = MemoryCache() // 🙃
let disk: Caching = DiskCache() // 🙃
let caches: [Caching] = [memory, disk] // 🙃
```

然后 Xcode 就好很无情的提示你：

> ❗️Protocol 'Caching' can only be used as generic constraint because it has Self or associated type requirements

WTF?

## What is Protocol with Associated Types?

看到这个错误提示，有经验的 Swift  程序员一般会想到，`Caching` 里面关联了一个类型，如果不指定这个关联的类型的具体类型是什么，作为一门静态语言，那可能就无法知道内存是怎么布局的。

既然需要指定类型，马上想到的就是 *泛型参数*。

```swift
let cache: Caching<UIImage> =  ...
// ❗️Protocol 'Caching' can only be used as generic constraint because it has Self or associated type requirements
```

```swift
protocol Caching<Object> {
}
// ❗️Protocols do not allow generic parameters; use associated types instead
```

WTF？

## Protocol as Types

突然发现自己好像一点都不了解 protocol，看看文档介绍[Protocols — The Swift Programming Language (Swift 4.2)](https://docs.swift.org/swift-book/LanguageGuide/Protocols.html)。里面 Protocol as Types 一节有一段话：

> Protocols don’t actually implement any functionality themselves. Nonetheless, any protocol you create will become a fully-fledged type for use in your code.
> Because it’s a type, you can use a protocol in many places where other types are allowed, including:
> * As a parameter type or return type in a function, method, or initializer
> * As the type of a constant, variable, or property
> * As the type of items in an array, dictionary, or other container

为什么 protocol + generic 就这么难用？应该怎么用？

## PATs in Swift Standard Library

既然不会用，那么看看 Standard Library 里面是如何使用的。

```swift
public protocol IteratorProtocol {
    /// The type of element traversed by the iterator.
    associatedtype Element

    /// - Returns: The next element in the underlying sequence, if a next element
    ///   exists; otherwise, `nil`.
    mutating func next() -> Element?
}

public protocol Sequence {
    /// A type representing the sequence's elements.
    associatedtype Element

    /// A type that provides the sequence's iteration interface and
    /// encapsulates its iteration state.
    associatedtype Iterator : IteratorProtocol where Iterator.Element == Element

    /// Returns an iterator over the elements of this sequence.
    func makeIterator() -> Iterator
}
```

然后又发现有一个叫 `AnyIterator` 和 `AnySequence` 的东西。

```swift
public struct AnyIterator<Element> {
    internal let _box: _AnyIteratorBoxBase<Element>

    public init<I : IteratorProtocol>(_ base: I) where I.Element == Element {
        self._box = _IteratorBox(base)
    }

    public init(_ body: @escaping () -> Element?) {
        self._box = _IteratorBox(_ClosureBasedIterator(body))
    }

    internal init(_box: _AnyIteratorBoxBase<Element>) {
        self._box = _box
    }
}

extension AnyIterator: IteratorProtocol {

    public func next() -> Element? {
        return _box.next()
    }
}

public struct AnySequence<Element> {
    internal let _box: _AnySequenceBox<Element>

    public init<I : IteratorProtocol>(_ makeUnderlyingIterator: @escaping () -> I) where I.Element == Element {
        self.init(_ClosureBasedSequence(makeUnderlyingIterator))
    }

    internal init(_box: _AnySequenceBox<Element>) {
        self._box = _box
    }
}

extension AnySequence: Sequence {
    public typealias Iterator = AnyIterator<Element>

    public init<S : Sequence>(_ base: S) where S.Element == Element {
        self._box = _SequenceBox(_base: base)
    }
}
```

这是什么鬼，先看看文档：

> This iterator forwards its next() method to an arbitrary underlying iterator having the same Element type, hiding the specifics of the underlying IteratorProtocol. —[AnyIterator - Swift Standard Library | Apple Developer Documentation](https://developer.apple.com/documentation/swift/anyiterator)

> An instance of AnySequence forwards its operations to an underlying base sequence having the same Element type, hiding the specifics of the underlying sequence. —[AnySequence - Swift Standard Library | Apple Developer Documentation](https://developer.apple.com/documentation/swift/anysequence)

说白了就是包装一层，转发一下，它有个术语叫做 **Type Erasure**

## What is Type Erasure?

首先 Swift 的类型系统里面，有两种类型：

* Concrete Type: Int, Bool…
* Abstract Type: associatedType, <T>

对于抽象类型来说，编译器无法知道这个类型的确切功能。当编译器处理抽象类型的时候，它无法知晓其所占的空间大小；甚至可能会认为这个类型是不存在的。Swift 是静态语言。

> Type erasure is a process in code that makes abstract types concrete.

具体看看 Swift Standard Library 里面，是怎么做到的：

```swift
// 1. abstract base
internal class _AnyIteratorBoxBase<Element> : IteratorProtocol {
    internal init() {}

    internal func next() -> Element? { _abstract() }
}

// 2. private box
internal final class _IteratorBox<Base : IteratorProtocol> : _AnyIteratorBoxBase<Base.Element> {
    internal init(_ base: Base) { self._base = base }

    internal override func next() -> Base.Element? { return _base.next() }

    internal var _base: Base
}

// 3. public wrapper
public struct AnyIterator<Element> {
    internal let _box: _AnyIteratorBoxBase<Element>

    public init<I : IteratorProtocol>(_ base: I) where I.Element == Element {
        self._box = _IteratorBox(base)
    }

    public init(_ body: @escaping () -> Element?) {
        self._box = _IteratorBox(_ClosureBasedIterator(body))
    }

    internal init(_box: _AnyIteratorBoxBase<Element>) {
        self._box = _box
    }
}
```

这个模式概括起来就是三个步骤：

* an abstract base class
* a private box class
* a public wrapper class

（想了解更多相关的理论知识？*Existential* 了解一下

## One more thing

到这里，我以为我已经掌握了如何用 PATs 了，然后有一天，我开始写一个轻量级的日志系统。

```swift
protocol Logging: Hashtable {
}
```

```swift
let loggers: Set<Logging> = []
// ❗️Using 'Logging' as a concrete type conforming to protocol 'Hashable' is no supported
```

```swift
func add(_ logger: Logging) {
}
// ❗️Protocol 'Logging' can only be used as generic constraint because it has Self or associated type requirements
```

看到这熟悉的错误，马上就想到 `Hashable` 其实是继承 `Equatable` 的，然后这个 `Equatable` 的几个方法里面，用了 `Self` 来占位，它其实也是 PATs 的一种。意不意外，惊不惊喜。（其实一点都不意外

然后就是 type erasure 了，真正根据上面的三个步骤来写的时候，发现好像跟之前的又有点不太一样，因为它没有关联别的类型。

```swift
// 1. abstract baes
class _AnyLoggerBoxBase<T> : Logging {
}

// 2. private box
class _LoggerBox<Base : Logging> : _AnyLoggerBoxBase <Base.Self> {
}

// 3. public wrapper
struct AnyLogger<T> {
    init<L : Logging>(_ base: L) where L.Self == Self {
    }
}
```

这三步里面的泛型参数像是多出来的，根本无从下手。

> 遇到不懂，首先看源码总是不会有错。— 圣人

了解 Swift 的都知道，有一个叫做 `AnyHashable` 的东西。（一顿抄，完事

## Ref

* [Alexis Gallagher - Protocols with Associated Types - YouTube](https://www.youtube.com/watch?v=XWoNjiSPqI8&t=2391s)
* [Keep Calm and Type Erase On](https://academy.realm.io/posts/tryswift-gwendolyn-weston-type-erasure/)
* [AnySequence - Swift Standard Library | Apple Developer Documentation](https://developer.apple.com/documentation/swift/anysequence)
* [swift/ExistentialCollection.swift.gyb at master · apple/swift · GitHub](https://github.com/apple/swift/blob/master/stdlib/public/core/ExistentialCollection.swift.gyb)
* [swift/AnyHashable.swift at master · apple/swift · GitHub](https://github.com/apple/swift/blob/master/stdlib/public/core/AnyHashable.swift)