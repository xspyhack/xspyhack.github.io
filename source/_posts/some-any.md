---
title: some P and any P
date: 2022-01-01 20:00:00
tags:
    - swift
    - existential
    - protocol
    - generic
    - pats
categories: advanced swift
---
## TL;DR
* `some` 是另一种特殊的 `generic`（符合要求的未知但具体的类型）。它的目的是用来对复杂的类型进行抽象，类型的信息不会被擦除。
* `any` 其实是显式的声明 `existential`。它的目的是用来对值进行抽象，类型信息会被 compiler 擦除。

## some Protocol
写过 SwiftUI 的人对于 `some` 都不陌生，每一个 `View` 的 `body` 的类型都是 `some View`。

```swift
struct ContentView: View {
  var body: some View {
    Text("Hello World")
  }
}
```

为什么需要引进一个新的关键词 `some`，不能直接写 `var body: View { ... }`？原因是 `View` 这个协议是经典的 *PAT*，对就是那个万恶之源 [PAT](https://blessingsoft.com/2018/08/26/pats/) 。

```swift
public protocol View {
    associatedtype Body : View
    var body: Self.Body { get }
}
```

对于 `PAT` 不能直接的把 `View` 当作一个类型来使用，`Protocol ‘View’ can only be used as generic constraint because it has Self or associated type requirements`。`View` 这个 `PAT` 没法自动生成一个 [Existential](https://blessingsoft.com/2019/06/05/existential/)，所以不能直接写 `var body: View { ... }`，必须明确指定 `View` 的具体类型。

```swift
struct ContentView: View {
  // typealias Body = Text
  var body: Text {
    Text("Hello World")
  }
}
```

明确 `Body` 的具体类型，有助于揭示 `ContentView` 的部分实现，同时也使得声明变得脆弱。如果想改变 `body` 的返回类型，必须同时修改对应的类型。在这个场景中，具体的 `body` 返回类型其实并不重要，重要的是它符合 `View` 协议，它是一个 `View`。这时候如果想要抽象出声明的返回值类型，就必须要考虑 `existential` 或者 `type erasure`。

基于这一点，Swift 5.1 中引入了 [SE-0244 opaque result types](https://github.com/apple/swift-evolution/blob/master/proposals/0244-opaque-result-types.md) 这一特性。`some Protocol` 表示一个确定的实现了 `Protocol` 协议的类型。同时它还有一个要求，就是 `some` 修饰 return type 的时候，要求所有的 return 语句返回相同的具体类型。

```swift
protocol P { }
extension Int : P { }
extension String : P { }

func foo(flip: Bool) -> some P {
  if flip { return 17 }
  return "a string" // error: different return types Int and String
}
```

在一个支持范型的语言中，想使用 `PAT` 又不需要指定具体的类型对于编写简洁代码非常重要。考虑一下嵌套的范型，如 SwiftUI 中那些 View 的 body 的真实类型。

> 当然，`some` 只是在编写代码的时候帮助我们进行了简化，类型信息依然存在，`SwiftUI` 非常依赖这些信息进行 View 的更新。

## any Protocol
Swift 中想要把一个 `Protocol` 作为一个类型来用，要求这个 `Protocol` 必须不能是 `PAT`，否则的话就会报错 `Protocol can only be used as generic constraint because it has Self or associated type requirements`。为了缓解这个问题，Swift 引入了 [SE-0309 unlock existential types for all protocols](https://github.com/apple/swift-evolution/blob/main/proposals/0309-unlock-existential-types-for-all-protocols.md)，compiler 帮忙自动的进行 `type erasure`。

```swift
protocol Logging: Hashtable { }

// before
// -func add(_ logger: AnyLogger) {
// after, 🙂️ OK
func add(_ logger: Logging) {
}
```

到这里，`PAT` 的使用成本大大的降低，实用性大大提升。终于可以像使用范型那样的使用 `PAT` 了。

```swift
func add(_ logger: Logging) { ... } // existential type
func add<T: Logging>(_ logger: T) { ... } // generic type

let logger: MemoryLogger
add(logger)
```

但是 `generic type` 和 `existential type` 始终是不同的东西，前者是 `type-level abstraction` 而后者是 `value-level abstraction`，前者强调的是类型以及类型之间的关系，后者关心的是值，类型信息被 compiler 帮忙抹除掉。

为了从语法上区分开两者，Swift 又引入了 [SE-0335 existential any](https://github.com/apple/swift-evolution/blob/main/proposals/0335-existential-any.md)，`existential type` 必须通过 `any` 关键词进行显式声明 。

```swift
let logger: Logging = MemoryLogger() // before
let logger: any Logging = MemoryLogger() // after
```

至此，终于可以一致的对待 Swift 中的所有 protocol 了，而不需要区别它是不是 PAT。

## Type-level and value-level abstraction
要深入了解 `some` 和 `any`，还是要先了解什么是 `type-level abstraction` 和 `value-level abstraction` 。

### Type-level abstraction
Generics（范型）提供了类型层面上的抽象，范型允许函数或者类型与符合给定约束集的任何类型统一使用，同时保留了在任何特定实例中使用的特定类型的标识。泛型函数引入了代表特定类型的类型变量（type variables，感觉更多时候被叫做类型参数？）。 这允许函数声明它接受任何符合协议的值。

```swift
func foo<T: Collection>(x: T) { ... }
```

类型变量 `T` 在类型层面上抽象出特定的 `Collection` 类型。具体的类型标识仍然保留着，因此类型系统可以保留不同值之间的类型关系。

```swift
func bar<T: Collection>(x: T, y: T) -> [T] { ... }
```

### Value-level abstraction
Existential 类型提供了值层面上的抽象。与绑定一些符合约束的现有类型的泛型类型参数相比，existential 类型是一种不同的类型，它可以保存符合一组约束的任何类型的任何值，在值层面上抽象底层的具体类型。Existentials 允许不同的具体类型的值作为相同的存在类型的值互换使用，在值层面上抽象了底层符合类型之间的差异。同一个 existential 类型的不同实例可以持有完全不同的底层类型的值，并且改变一个 existential 类型可以改变该值所持有的底层类型。

### Type-level abstraction is missing for function returns
泛型是 Swift 在函数接口中进行类型层面的抽象的工具，但它们的工作方式基本上在调用者的控制之下，也就是说具体类型由调用方来决定。

```swift
func zim<T: P>() -> T { ... }
```

对于 `zim` 这个范型函数，具体返回的类型 `T`，是由调用方来决定，实现方（callee）只声明了它必须要满足 `P` 约束。

```swift
let x: Int = zim() // T == Int chosen by caller
let y: String = zim() // T == String chosen by caller
```

### "Reverse generics" for return type abstraction
但有时候，实现方真正希望的 `zim` 函数是它返回一个 `P` 类型，但具体是 `Int` 还是 `String` 由实现方来决定。这点与普通的范型刚刚相反，可以认为是反向的范型（`reverse generics`）。

```swift
func zim() -> P { ... }
```

此时的 `P` 不再是用来作为范型约束，`zim` 函数返回的值是一个 existential。 

### Expressing constraints directly on arguments and returns
为了完善 Swift 的类型抽象系统，提出了使用 `some` 这个修饰符来对参数和返回值表达约束，而不需要使用 `existential` 类型。

```swift
func concatenate(a: some Collection, b: some Collection) -> some Collection { ... }
```

甚至可以使用 `where` 来表达它们直接的类型关系。

```swift
func concatenate(a: some Collection, b: some Collection) -> some Collection
  where type(of: a).Element == type(of: b).Element,
        type(of: return).Element == type(of: b).Element
```

## Conclusion
* `some` 是另一种特殊的 `generic`（符合要求的未知但具体的类型）。它的目的是用来对复杂的类型进行抽象，类型的信息不会被擦除。
* `any` 其实是显式的声明 `existential`。它的目的是用来对值进行抽象，类型信息会被 compiler 擦除。

`some` 和 `any` 是对偶的（`duals`）两个修饰词，它们都解决了 `PAT` 所带来的一些不方便的问题。目前 Swift 只支持使用 `some` 来修饰返回值，而 `any` 的使用场景则比较多。

## Ref
[swift-evolution/0244-opaque-result-types.md at master · apple/swift-evolution · GitHub](https://github.com/apple/swift-evolution/blob/master/proposals/0244-opaque-result-types.md)
[swift-evolution/0309-unlock-existential-types-for-all-protocols.md at main · apple/swift-evolution · GitHub](https://github.com/apple/swift-evolution/blob/main/proposals/0309-unlock-existential-types-for-all-protocols.md)
[swift-evolution/0335-existential-any.md at main · apple/swift-evolution · GitHub](https://github.com/apple/swift-evolution/blob/main/proposals/0335-existential-any.md)
[Improving the UI of generics - Discussion - Swift Forums](https://forums.swift.org/t/improving-the-ui-of-generics/22814#heading--clarifying-existentials)