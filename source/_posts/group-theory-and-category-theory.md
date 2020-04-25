---
title: Group Theory and Category Theory
tags:
	- group theory
	- category theory
	- semigroup
	- monoid
	- functor
	- applicative
	- monad
	- functional programming
	- swift
categories: functional programming
---
想理解函数式编程中的一些高大上的概念，比如 Functor, Monad 等，必须要先理解范畴论。

## Group

> 群，是一种代数结构，由一个集合（G）以及一个二元运算符（·）所组成。[wikipedia](https://zh.wikipedia.org/wiki/群)

一个群必须满足一些称为 *群公理* 的条件，也就是 **封闭性**、**结合律**、**单位元** 和 **逆元**。如整数配备上加法运算就形成一个群。

* 封闭性（Closure）：对于任意 a,b∈G，a·b∈G。
* 结合律（Associativity）：对于任意 a,b,c∈G，(a·b)·c = a·(b·c)。
* 单位元（Identity）：G 中存在一个元素 e，使得任意 a∈G，a·e = e·a = a。
* 逆元：对于任意 a∈G，存在 b∈G，使得 a·b = b·a = e。

群并不要求这个二元运算符（·）具体做什么，它只要求这个二元运算符存在，所以很多数学结构都是群。比如我们可以把整数当作一个群，把 `+` 作为二元运算符。

* 封闭性：对于任意两个整数 a,b，a+b 依然是一个整数。
* 结合律：对于任意整数 a,b,c，(a+b)+c = a+(b+c)。
* 单位元：存在元素 0，使得 a+0 = 0+a = a。
* 逆元：对于任意整数 a，当 b=-a 时，a+b = b+a = e。

所以我们可以说 `(整数, +)` 是一个群。如果把 `*` 当作二元运算符，把 `1` 作为单位元的时候，整数就形成了另一个群。

除了整数，还有很多数学结构是群。

### Semigroup

满足封闭性和结合律的群，称为半群（semigroup）。

```swift
infix operator <>: AdditionPrecedence

protocol Semigroup {
    static func <>(lhs: Self, rhs: Self) -> Self
}

extension Int: Semigroup {
    static func <>(lhs: Int, rhs: Int) -> Int {
        return lhs + rhs
    }
}

extension Array: Semigroup {
    static func <>(lhs: Array, rhs: Array) -> Array {
        return lhs + rhs
    }
}

// 折叠 fold
func concat<S: Semigroup>(_ xs: [S], _ initial: S) -> S {
    return xs.reduce(initial, <>)
}
```

半群的结合律特性使得我们可以进行并行运算，`1 <> 2 <> 3 <> 4`。

### Monoid

在抽象代数中，有一类简单的抽象结构被称为 Monoid（幺半群）。许多数学结构都是幺半群，因为成为幺半群的要求非常低。

存在单位元的半群，称为含幺半群，或者幺半群，或者单群，或者独异点（monoid）。

```swift
protocol Monoid: Semigroup {
    static var e: Self { get }
}

extension Int: Monoid {
    static var e = 0
}

extension Array: Monoid {
    static var e: Array {
        return []
    }
}

func concat<M: Monoid>(_ xs: [M]) -> M {
    return xs.reduce(M.e, <>)
}
```

`concat` 是对 Monoid 的一种应用，它可以利用 Monoid 的定义（ 二元操作 `<>` 和 单位元 `e` ）进行折叠操作。

## Category Theory

> 范畴论是数学的一门学科，以抽象的方法来处理数学概念，将这些概念形式化成一组组的“物件”及“态射”。[wikipedia](https://zh.wikipedia.org/wiki/范畴论)

### Category

一个范畴 C 包括：

* 一个由对象（object）组成的类 ob(C)。（注：这里把“物件”成为“对象“更有助于从计算机的角度理解
* 对象间态射（morphism，->）组成的类 hom(C)：每个态射 f 都只有一个「源对象」a 以及一个「目标对象」b（其中 a,b 都在 ob(C) 内），称之为 **从 a 到 b 的态射**，记为 f: a->b。（注：identity 态射即自己映射到自己的特殊态射，f: a->a，简单记为 id[a]）
* 一个二元运算符（·），用于态射组合，如 h=g·f。

满足公理：

* 结合律：f: a->b, g: b->c, h: c->d，h·(g·f) = (h·g)·f。
* 单位元：id[a]·f = id[b]·f = f。
* 封闭性：f: a->b, g: b->c, h: a->c, h = f·g。

范畴举例：

* 范畴 C 有 Int 类型和 String 类型对象。
* 存在态射 f: Int->String。

划重点：幺半群可以视为一类特殊的范畴。幺半群运算满足的公理同于范畴中 **从一个对象到自身的态射**。换言之：
幺半群实质上是只有单个对象的范畴。

### Functor

> 在范畴论中，函子是范畴间的一类映射。函子也可以解释为小范畴为成员的范畴内的态射。 [wikipedia](https://zh.wikipedia.org/wiki/函子)

在当代数学中，函子被用来描述各种范畴间的关系。对范畴论者来说，函子则是个特别类型的函数。

设 C 和 D 为范畴，从 C 至 D 的函子为一映射 F:

* 将每个对象 x∈C 映射至一对象 F(x)∈D 上。
* 将每个态射 f: x->y∈C 映射至一态射 F(f): F(x)->F(y)∈D 上，

使之满足：

* 对任何对象 x∈C，恒有 F(id[x]) = id[F(x)]。
* 对任何态射 f: x->y, g: y->z，恒有 F(f·g) = F(f)·F(g)。

换言之，函子会保持单位态射与态射的复合。
一个由一范畴映射至其自身的函子称之为 **自函子（Endofunctor）**。

#### 可以把范畴当作一组类型的集合

如范畴 C 有 `Int` 类型和 `String` 类型对象，以及 `Int -> String` 的态射；范畴 D 有 `Array<Int>` 类型和 `Array<String>` 类型对象，以及 `Array<Int> -> Array<String>` 的态射。两个范畴之间的映射 F：

* `Int` 映射至 `Array<Int>` 上，`String` 映射至 `Array<String>` 上。
* 态射 `Int -> String` 映射至 `Array<Int> -> Array<String>` 上。

翻译成代码：C: `Int`, `String`, f: `Int -> String`, D: `Array<Int>`, `Array<String>`, f: `Array<Int> -> Array<String>`。

* x: `Int` -> F(x): `Array<Int>`，`String` -> F(x): `Array<String>`
* f: `(Int -> String)` -> F(f): `(Array<Int> -> Array<String>)`

范畴是不涉及具体类型的，所以用泛型表示：

```swift
func tmap<T>(x: T) -> F<T>
func fmap<A, B>(f: A -> B) -> (F<A> -> F<B>)
```

简化一下变成：

```swift
// Swift 中把 Int 映射到 Array<Int> 由 Array 的初始化方法提供，
// 所以可以不写。
// 由于 fmap 实际上是 (F<A>, A -> B) -> F<B> 的 currying 版本，
// 所以两者是等价的。
func map<A, B>(x: F<A>, f: A -> B) -> F<B>
```

再来看看 Swift 中的 `Array` 和 `Optional`。如果把 Swift 中所有的类型 `A, B` 当作对象，以及 Swift 中所有的函数当作态射 `A -> B`，那么这些类型和函数就组成一个范畴 A。把 `Array` 类型当作对象 `Array<A>, Array<B>`，`Array` 上所有的函数当作态射 `Array<A> -> Array<B>`，那么也组成一个范畴 B。而 A 到 B 之间的函子是 `Array`，因为函子 `Array` 能将任意类型 `T` 转换为 `Array<T>`。`Optional` 同理。

很多库对 `Functor` 的支持直接在类型构造器（Type Constructor）的定义中实现 `map` 方法，比如 Swift 中的 `Array` 和 `Optional` 就是需要一个泛型作为参数来构建具体类型的类型构造器，它在定义中实现了 `map` 方法。这些类型构造器相当于同时具备了类型和函数的映射。在 Haskell 里把这个行为称为 `Lift`，相当于把类型和函数放到容器里面。所以一个带有 `map` 方法的类型构造器就是一个函子。

范畴与高阶类型：如果忽略范畴中的态射，范畴其实就是对特定类型的抽象，即高阶类型（类型构造器）。对于范畴 D，它的所有类型都是 `Array<T>` 的特定类型。而对于范畴 C，可以看作是一个 Identity 类型的构造器（id[T] = T）。

注意⚠️：函子不是容器，函子不是容器，函子不是容器。
如 `typealias Parser<A> = (String) -> (A, String)?` 我们可以实现 `func map<A, B>(x: Parser<A>, f: (A) -> B) -> Parser<B>` 函数，所以我们可以说 `Parser<A>` 是一个函子，但它不是容器。

### Endofunctor

> A functor that maps a category to itself。一个由一范畴映射至其自身的函子称之为 **自函子（Endofunctor）**。

先看自函数的概念：将一个类型映射到自身类型，如 `Int -> Int`。

```swift
func f(x: Int) -> Int { return x + 1 }
```

单位函数（Identity Function）的概念：什么都不做，传入什么就返回什么。属于自函数的特例。

```swift
func id(x: Int) -> Int { return x }
```

**自函子不是单位函子（Identity Functor）**。还是上面的范畴 C，为了区分自函子和单位函子，多加一种态射 g: `String -> Int`，那么：

自函子：对于函子 F，对于 `F(Int)` 结果是 `String`，`F(String)` 结果是 `Int`，对于 `F(f: Int -> String)` 结果是 g: `String -> Int`。那么这个函子就是自函子。

单位函子（Identity Functor）：对于函子 F，对于 `F(Int)` 结果还是 `Int`，对于 `F(String)` 结果还是 `String`，对于 `F(f: Int -> String)` 结果还是 `f: Int -> String`，对于 `F(g: String -> Int)` 结果还是 g: `String -> Int`。那么这个函子就是单位函子。 

## Applicative

> An applicative is a monoid in the category of endofunctors, what's the problem?

虽然在 Haskell 中 Monad 是 Applicative 的一种，但是 Applicative 的出现却在 Monad 之后。

[Applicative](https://www.reddit.com/r/haskell/comments/2lompe/where_do_the_applicative_laws_come_from/)

## Monad

> A monad is a monoid in the category of endofunctors -- Philip Wadler

自函子说穿了就是把一个范畴映射到自身的函子，自函子范畴说穿了就是从小范畴映射到自身的函子所构成的以自函子为对象以自然变换为态射的范畴，幺半群说穿了就是只有单个对象的范畴，给定了一个幺半群则可构造出一个仅有单个对象的小范畴使其态射由幺半群的元素给出而合成由幺半群的运算给出，而单子说穿了就是自函子范畴上的这样一个幺半群。（这都不理解么亲连这种最基本的概念都不理解还学什么编程！

一系列 Endofunctor 组成的范畴，成为 **自函子范畴**。

* X 上的自函子：F：`X -> X`
* 单位自函子 id[X] 到函子 F 的自然转换：`id[X] -> F` (pure
* 函子 F 的张量积 F⊗F 到函子 F 的自然转换：`F⊗F -> F` (join

代码表示：

* `func unit<T>(x: T) -> F<T> // x = id[x]` 
* `func join<T>(a: F<F<T>>) -> F<T>`

此处函子的张量积 ⊗ 可以看作为组合（Composition）；
注意结合 Monoidal Category 理解，`unit` 和 `join` 满足 Monoid 的定律，所有形成了 Monoid。

也就是说：单子（Monad）是自函子的 Monoidal 范畴上的一个幺半群，该 Monoidal 范畴的张量积（Tensor Product，⊗：F×F -> F）是自函子的复合（Composition），单位元是 Id Functor。

`bind` 或者说 `flatMap` 或者 `>>=` 其实等于 `map + join`。（见 [apple/swift](https://github.com/apple/swift) 中的 `stdlib/public/core/FlatMap.swift`

```swift
func >>=<A, B>(x: F<A>, f: A -> F<B>) -> F<B>
// Currying 前的样子
func >>=<A, B>(x: F<A>) -> (f: A -> F<B>) -> F<B>
```

## Ref

Too much


