---
title: Parser Combinator
tags:
	- parser
	- parser combinator
	- monad
	- functional programming
	- swift
categories: functional programming
---
解析组合子是由多个解析器为参数并返回一个解析器的高阶函数。

## Parser

将一个数据流解析成结构化的数据的工具，我们称为解析器。比如我们需要将用户输入的表达式字符串解析成 AST，我们就可以使用解析器来达到我们的目的。

`( 4 + 3  )` 就是一个表达式语句，它由字符 `(`  `4` `空格` `+` `3` 和 `)` 组成。我们可以将这个表达式解析成一种 *表达式树* (AST 的一种)。

所以我们的解析器简单的用一个函数来描述就是：
`func parser(_ string: String) -> AST`

我们不是用正则表达式来解析输入的表达式字符串，为了得到表达式树里面的节点，我们需要一步步的解析，每次解析得到不同的节点。所以我们需要将解析器的定义变成解析成功的话，会返回结果值和剩下的字符串。

`func parser(_ string: String) -> (AST, String)`

表达式树的节点都是一些 `4` `+` 这种不同类型的数据，所以为了表示解析 `4` 成功和解析 `+` 成功，我们的返回值可以定义为泛型。并且表达出解析失败的情况，我们可以使用可选值。最终解析器函数就变成了：

`func parser<T>(_ string: String) -> (T, String)?`

所以解析第一个数字 4 的解析器函数为：

```swift
func parser(_ string: String) -> (Int, String)? {
    guard let head = string.characters.first, head == "4" else {
        return nil
    }
    return Optional.some((4, String(string.characters.dropFirst())))
}
```

## Combinator

> One of the distinguishing features of functional programming is the widespread use of combinators to construct programs. *A combinator is a function which builds program fragments from program fragments*; in a sense the programmer using combinators constructs much of the desired program automatically, rather that writing every detail by hand. -- John Hughes

其实 Combinator 很容易理解，就像字面意思那样 —— 组合子。首先定义一系列原子操作，然后定义组合的规则，然后根据组合的规则把这些原子操作组合起来。

## Parser Combinator

回到开头的话：*解析组合子是由多个解析器为参数并返回一个解析器的高阶函数。* 所以我们需要重新定义一下我们的解析器，把它变成一个解析组合子。

```swift
struct Parser<A> {
    let parse: (String) -> (A, String)?
    // (input) -> (result, remaining)
}
```

这是一个解析字符串的解析器，我们把这个函数放到一个结构体 `Parser` 中，作为一个 `parse` 变量。当然我们也可以用类型别名 `typealias Parser<Result> = (String) -> (Result, String)?`。

当然解析组合子不仅仅能解析字符串，所以可以用泛型来把它变得更通用。

```swift
struct Parser<I, O> {
    let parse: (I) -> (O, I)?
    // (input) -> (output, remaining input)
}
```

所以解析数字字符 `4` 的解析器就变成了：

```swift
func character4() -> Parser<Character> {
    return Parser { input in
        guard let head = input.characters.first, head == "4" else {
            return nil
        }
        return ("4", String(input.characters.dropFirst()))
    }
}
```

所以根据 `func character4() -> Parser<Character>` 很容易得到一个能够解析任何字符的解析器：

```swift
func character(_ character: Character) -> Parser<Character> {
    return Parser { input in
        guard let head = input.characters.first, head == character else {
            return nil
        }
        return (head, String(input.characters.dropFirst()))
    }
}
```

根据 `character` 和 `digit` 的区别，很容易又得到能够解析任何数字的解析器：

```swift
func digit() -> Parser<Character> {
    return Parser { input in
        guard let head = input.characters.first, "0"..."9" ~= head else {
            return nil
        }
        return (head, String(input.characters.dropFirst()))
    }
}
```

根据 `character` 和 `digit` 相同和不同，我们可以进一步抽象，把相同部分进行封装，把不同部分作为参数，得到新的解析器：

```swift
func satisfy(_ condition: @escaping (Character) -> Bool) -> Parser<Character> {
    return Parser { input in
        guard let head = input.characters.first, condition(head) else {
            return nil
        }
        return (head, String(input.characters.dropFirst()))
    }
}
```

所以解析数字：

`(satisfy { "0"..."9" ~= $0 }).parse("1abc")`

解析空格：

```swift
func isSpace(_ character: Character) -> Bool {
    return String(character).trimmingCharacters(in: .whitespacesAndNewlines).isEmpty
}
(satisfy(isSpace)).parse(" abc")
```

所以我们只需要给 `satisfy` 函数传入一个是否属于 X 的函数，就可以得到一个能够解析 x 的解析器。

## Next

最基本的 `character` 有了，`digit` 有了，当我们需要解析一个字符串 `alex` 的时候，我们只需要把 `alex` 看成 `a` `l` `e` `x` 4 个字符，然后不断的用 `character` 进行解析，最后把每一步返回的结果合并起来就行了。考虑到解析一个字符串是一个基本功能，为了不用每次写重复的代码，把它封装成用来解析 `string` 的解析器。

```swift
func string(_ str: String) -> Parser<String> {
    let parsers = str.characters.map { character($0) }
    return Parser { input in
        var results: [Character] = []
        var stream = input
        for parser in parsers {
            guard let (result, remainder) = parser.parse(stream) {
                return nil
            }
            results.append(result)
            stream = remainder
        }
        return (String(results), stream)
    }
}
```

观察 parse 函数类型 `(String) -> (A, String)`，解析成功的返回值是解析结果和 **剩余** 的字符串，所以解析 `alex` 的时候：

1. "alex": 'a' -> ('a', "lex")
2. "lex": 'l' -> ('l', "ex")
3. "ex": 'e' -> ('e', "x")
4. "x": 'x' -> ('x', "")
5. Parser<"alex">

可以看到这几步做的事情除了参数不一样，内部逻辑是一样的，而且很容易看出是一个递归的过程，**每次解析成功就 `吃掉` 第一个字符**（留意这句话），然后解析剩下的字符串。

所以我们写一个递归的版本：

```swift
func string(_ str: String) -> Parser<String> {
    return Parser { input in
        guard let (head, tail) = uncons(str.characters) else {
            // 0. 把空字符解析器去解析任何字符串，都认为是解析成功
            return ("", input)
        }
        // 1. 先解析第一个字符
        guard let (_, remainder1) = character(head).parse(input) else {
            return nil
        }
        // 2. 然后解析剩下的所有
        guard let (_, remainder2) = string1(String(tail)).parse(remainder1) else {
            return nil
        }
        // 3. 返回("结果", "剩余的字符串")
        return (str, remainder2)
    }
}

func uncons<C: Collection>(_ xs: C) -> (C.Iterator.Element, C.SubSequence)? {
    guard let head = xs.first else {
        return nil
    }
    return (head, xs.suffix(from: xs.index(after: xs.startIndex)))
}
```

观察 1 和 2，在这两步中，我们都没有使用解析的 **结果**，这两步实现的仅仅是 **每次解析成功就 `吃掉` 结果**！最后在第 3 步一次将结果返回。也就是说我们 1 和 2 这两本并不关心结果，只关心这些要解析道字符存在就行了。

我们把解析成功吃掉结果这一步封装一下：

```swift
func discarding<A, B>(_ x: Parser<A>, _ y: Parser<B>) -> Parser<B> {
    return Parser { input in
        guard let (_, remainder1) = x.parse(input) else {
            return nil
        }
        guard let (result2, remainder2) = y.parse(remainder1) else {
            return nil
        }
        // 只保留右边的解析器的结果 result2，没有 result1
        return (result2, remainder2)
    }
}
```

`discarding` 函数会 **吃掉** 左边第一个参数 `x` 的解析结果，返回值中只保留右边 `y` 的解析结果。用 `discarding` 函数重写一下上面的 `string` 函数：

```swift
func string(_ str: String) -> Parser<String> {
    return Parser { input in
        guard let (head, tail) = uncons(str.characters) else {
            // 把空字符解析器去解析任何字符串，都认为是解析成功
            return ("", input)
        }
        // 1 吃掉 character(head) 的结果
        guard let (_, remainder) = discarding(character(head), string2(String(tail))).parse(input) else {
            return nil
        }
        // 2 返回 ("结果", "剩余的字符串")
        return (str, remainder)
    }
}
```

这次改版的 `string` 里面第 1 步中的解析结果还是被忽略了，所以是否可以继续用 `discarding` 来简化？但是 `discarding` 函数需要两个解析器，但函数内只有 `lift` 返回的一个解析器，所以没办法继续简化了？

仔细看 `string` 函数体的第一行 `return Parser {}` 就是一个解析器，能否把这个解析器利用上呢？`discarding` 是在 `Parser {}` 里面的，所以只要能想办法把它展平，那么就能再次利用上 `lift`，而且展平后的解析器需要做为 `string` 函数的返回值，所以它肯定是做为 `discarding` 的右边的参数。

```swift
func string(_ str: String) -> Parser<String> {
    let lhs = Parser {}
    let rhs = Parser<String> {}
    return lift(x, y)
}
```

根据第 2 步的 `return (str, remainder)` 可以知道，最终的返回结果是 `(输入的，lhs 吃剩的)`，所以很容易得到 `let rhs = Parser<String> { (str, $0) }`。所以可以推出 lhs 要做的只是负责吃掉一部分。也就是上面的第 1 步所做的。所以：

```swift
func string(_ str: String) -> Parser<String> {
    guard let (head, tail) = uncons(str.characters) else {
        // 1 把空字符解析器去解析任何字符串，都认为是解析成功
        return ("", input)
    }
    // 2 吃掉
    let lhs = discarding(character(head), string2(String(tail)))
    // 3 结果和剩下的
    let rhs = Parser<String> { (str, $0) }
    // 4 返回
    return discarding(lhs, rhs)
}
```

函数的返回值是 `Parser`，由于外面没有 Parser {} ，展开后 1 那里需要返回一个 `Parser`。

```swift
func string(_ str: String) -> Parser<String> {
    guard let (head, tail) = uncons(str.characters) else {
        // 1 把空字符解析器去解析任何字符串，都认为是解析成功
        return Parser<String> { ("", $0) }
    }
    // 2
    let lhs = discarding(character(head), string2(String(tail)))
    // 3
    let rhs = Parser<String> { (str, $0) }
    // 4
    return discarding(lhs, rhs)
}
```

明眼人可以看到 1 和 3 只有 `""` 和 `str` 不一样，剩下的一模一样，虽然代码不长，但我们还是把它相同部分封装成一个函数，然后把不同的部分做为参赛。

```swift
func string(_ str: String) -> Parser<String> {
    guard let (head, tail) = uncons(str.characters) else {
        // 1 把空字符解析器去解析任何字符串，都认为是解析成功
        return pure("")
    }
    // 2
    let lhs = discarding(character(head), string2(String(tail)))
    // 3
    let rhs = pure(str)
    // 4
    return discarding(lhs, rhs)
}

// Lift a value
func pure<A>(_ x: A) -> Parser<A> {
    return Parser<A> { (x, $0) }
}
```

所以封装一个简洁的 `string` 解析器，花了很多功夫，但抛开性能，它比迭代的版本更简洁易懂。

## Combine

观察表达式 `( 4 + 3  )`，里面 `(` 和 `4` 之间有 1 个空格，数字 `3` 和 `)` 中间是有 2 个空格，在做加法运算的时候，这些 *many* 个空格是没有意义的，所以需要 *skip* 掉。

#### Many

首先需要解析空格的解析器，前面已经有实现过：

```swift
func space() -> Parser<Character> {
    return satisfy(isSpace)
}
```

由于空格数量未知，可能有 *many* 个，假如有一个解析器，能够解析 *many* 个 parser。用一个 loop 不断去解析就能实现：

```swift
func many<A>(_ x: Parser<A>) -> Parser<[A]> {
    return Parser { input in
        var results: [A] = []
        var stream = input
        while let (result, remainder) = x.parse(stream) {
            results.append(result)
            stream = remainder
        }
        return (results, stream)
    }
}
```

任意个空格就是：

```swift
func spaces() -> Parser<[Character]> {
    return many(space())
}
```

`spaces` 得到的是一个 `Parser<[Character]>` 类型的 parser，但是按照理解更希望得到一个 `Parser<String>` 类型的 parser。在 Swift 中，`String([Character])` 就能够将 `[Character]` 拍扁成 `String` 类型。所以把 `many` 稍微修改一下：

```swift
func many<Character>(_ x: Parser<Character>) -> Parser<String> {
    return Parser { input in
        var results: [Character] = []
        var stream = input
        while let (result, remainder) = x.parse(stream) {
            results.append(result)
            stream = remainder
        }
        return (String(results), stream)
    }
}
```

因为不是任何泛型 A 类型，都能用 String 拍扁，也不一定能通过其他类型进行拍扁，所以这里把泛型 A 去掉，直接用 Character 代替。但是这样做并不理想，因为 `many` 解析器从一个泛型解析器，变成了一个只能解析 Character 类型的解析器，变成了 `manyCharacter`。后面考虑解析这个问题，重新把 `many` 变成通用的解析器。

#### Skip

接着实现一个通用的 `skip` 解析器，它要做的事情很简单，输入什么吃掉什么，返回剩下的，和上面吃掉左边的 `discarding` 很像，不一样的是 `skip` 只有一个参数。

```swift
func skip<A>(_ x: Parser<A>) -> Parser<Void> {
    return Parser { input in
        guard let (_, remainder) = x.parse(input) else {
            return ((), input)
        }
        
        return ((), remainder)
    }
}
```

把 `skip` 和 `spaces` 进行组合：

```swift
func skipSpaces() -> Parser<Void> {
    return skip(spaces)
}
```

#### Many1

前面实现的 `digit` 解析器，它只能解析个位数，这是没有什么卵用的。相比 `digit`，更加需要的是一个 `number` 解析器。一个 *number* 实际上也是由 *many* 个 *digit* 组成。

```swift
func number() -> Parser<[Character]> {
    return many(digit())
}

number().parse("123abc") // (["1", "2", "3"], "abc")
number().parse("abc") // ([], "abc")
```

等等！`number().parse("abc")` 也解析成功了，结果是空数组。这并不是想要的结果，一个 *string* 可以是空的，*space* 甚至也可以是空的，但一个 *number* 不能是空的。所以需要另外一个只是有一个的 `many`。这其实很常见，比如正则表达式中有 `*` 和 `+`，一个 {0, +} 一个是 {1, +}。

```swift
func many1<A>(_ x: Parser<A>) -> Parser<[A]> {
    return Parser { input in
        // 多加一个判断，第一个值必须满足条件
        guard let (_, _) = x.parse(input) else {
            return nil
        }
        var results: [A] = []
        var stream = input
        while let (result, remainder) = x.parse(stream) {
            results.append(result)
            stream = remainder
        }
        return (results, stream)
    }
}
```

所以正确的 `number` 解析器就变成：

```swift
func number() -> Parser<[Character]> {
    return many1(digit())
}
```

这里 `number` 解析器和 `spaces` 解析器遇到了同样的问题，`number` 解析器的结果应该是 `Int`（暂不考虑浮点数），而不是 `[Character]`。解决方法可以类似 `manyCharacter`，但是这显示是很有问题的，抽象抽象抽象！

程序员要有抽象思维，要学会用更高的层次的思维去看待问题，发现不同问题的共同点。`[Character]` 可以用 `String([Character])` 变成一个 `String`。对于 `digit` character，同样的也是用 `String([Character])` 拍扁，然后用 `Int(String)` 得到一个 `number`。

结合 Swift 的 OOP（面向协议编程），可以定义一个协议，暂且叫做 `Combinable` :D

```swift
protocol Combinable {
    static func combine(_ xs: [Character]) -> Self
}

extension Int: Combinable {
    
    static func combine(_ xs: [Character]) -> Int {
        return Int(String(describing: xs))!
    }
}

extension String: Combinable {
    static func combine(_ xs: [Character]) -> String {
        return String(describing: xs)
    }
}

func many1<A: Combinable>(_ x: Parser<Character>) -> Parser<A> {
    return Parser { input in
        guard let (_, _) = x.parse(input) else {
            return nil
        }
        var results: [Character] = []
        var stream = input
        while let (result, remainder) = x.parse(stream) {
            results.append(result)
            stream = remainder
        }
        return (A.combine(results), stream)
    }
}
```

然后就可以得到：

```swift
func number() -> Parser<Int> {
    return many1(digit())
}

func spaces() -> Parser<String> {
    return many1(space()) // 忽略 spaces 可以为空的情况
}
```

实际上 `skipSpaces` 还可以用另外一个角度来拆分，上面先解析 *many* 个空格，然后一次 *skip* 掉。还可以每次 *skip* 一个空格，然后进行 *many* 次。不同的地方是 `skip` 和 `many` 两个 parser 的调用次序不一样，甚至还可以定义一个叫做 `skipMany` 的解析器，这也说明了 **Combinator** 的强大。通过定义一系列基础的 parser，进行不同的排列组合操作，最后覆盖所有的 case。（理想状态

#### Zip

合并两个 parser 的解析器 `zip` 的实现也很简单：

```swift
func zip<A, B>(_ x: Parser<A>, _ y: Parser<B>) -> Parser<(A, B)> {
    return Parser { input in
        guard let (result1, remainder1) = x.parse(input) else {
            return nil
        }
        guard let (result2, remainder2) = y.parse(remainder1) else {
            return nil
        }
        return ((result1, result2), remainder2)
    }
}
```

#### Choice

接下来还需要解析几个简单的一元运算符 `+` `-` `*` `/`。去掉空格后，两个数中间必须是其中一个运算符那么表达式就是合法的。

```swift
func opt() -> Parser<Character> {
    let opts = ["+", "-", "*", "/"].map { character($0.characters.first!) }
    return choice(opts)
}
```

```swift
// 也可以叫 one(of:)
func choice<A, S: Sequence>(_ xs: S) -> Parser<A> where S.Iterator.Element == Parser<A> {
    return xs.reduce(empty(), { $0 <|> $1 })
}

func empty<A>() -> Parser<A> {
    return Parser { _ in nil }
}
```

## Transform

上面利用 Protocol 实现的 `number` 和 `space` 解析器其实并不是很优雅，费了很大劲把 `many` 变得 ~~通用~~，结果却并不是很 **通用**，因为要求结果的类型必须实现 `Combinable` 协议。但它做的工作却很少，只是把传入的 `Parser<A>` 循环解析得到的结果 `[A]` 在 `many` **内部** 组合成最终的类型，实现把 `Parser<[A]` 转换为 `Parser<B>`。正由于它是在 `many` 内部做的操作，所以依赖于传入的类型，使得 `many` 不再那么通用。

再看 `func character(_ character: Character) -> Parser<Character>` 的定义，假如调用 `character("4")`，那么返回的是一个 `Parser<Character>` 类型的解析器，这个解析器调用 `parse` 方法，返回的结果是 `Character` 类型的值。在解析表达式 `4 + 3` 的时候，需要将解析到的 `4` 和 `3` 当作一个整数然后相加，才能得到最终的结果，所以不想要 `Character` 类型的值，而是想要 `Int` 类型的值，那么需要将 `Parser<Character>` 转换为 `Parser<Int>` 解析器。

所以，假如能实现一个函数，可以将任意 `Parser<A>` 转换为 `Parser<B>` 解析器，就完美了。`many` 只负责将 `Parser<A>` 解析得到 `Parser<[A]>`，然后由 `number` 自己将 `Parser<[A]>` 转换为 `Parser<Int>`。

### Functor

回忆 Swift 中 Optional 类型中的 map 方法：

```swift
let a: Optional<Int> = 1
let b: Optional<String> = a.map { String($0) }
```

它将一个 `Optional<Int>` 转换为 `Optional<String>`，仔细一看，把 `Optional` 换成 `Parser`，就是我们所需要的转换解析器的方法。

`Optional` 的函数签名是 `func map<U>(_ transform: (Wrapped) -> U) -> U?`，所以依葫芦画瓢，我们可以得到：

```swift
struct Parser<A> {
    func map<B>(_ transform: (A) -> B) -> Parser<B> {
        return Parser { input in
            guard let (result, remainder) = self.parse(input) else { return nil }
            return (transform(result), remainder)
        }
    }
}
```

像上面的 `satisfy` 和其他函数一样，把 `map` 方法从结构图内移出来，则得到：

```swift
func map<A, B>(_ x: Parser<A>, _ f: @escaping (A) -> B) -> Parser<B> {
    return Parser { input in
        guard let (result, remainder) = x.parse(input)else {
            return nil
        }
        return (f(result), remainder)
    }
}
```

由于值是在 `Parser` 中包裹着的，想把返回的 `Parser<Character` 变成 `Parser<Int>`，需要把 `Parser<Character>` 解开取出里面<Character>的值，然后把它变成<Int>类型，然后重现包装起来。对于不同的类型转换，解包重新包装的步骤是一样的，不同的地方是把结果从一种类型变成另一种类型，函数的作用就是把相同的封装起来，把不同做为参赛传进去，所以在 `map` 函数的实现中，只需要在返回前，给外部将这个结果进行一次转换机会，所以需要一个参赛，能够将解开后得到的值变成另一种类型的值，也就是提供一个函数 `(Character) -> Int`。

重新实现 `number` 和 `spaces`：

```swift
func number() -> Parser<Int> {
    return map(many1(digit()), { Int(String($0))! })
}

func spaces() -> Parser<String> {
    return map(many(space()), { String($0) })
}
```

两种不同的结构体 `Optional<T>` 和 `Parser<A>`，都可以给它实现一个 `map` 方法，使得它变成一个不同类型的结构体。而支持这种 `map` 方法的结构体，我们称把它为 `Functor`。

> 简单来说，所谓的 `Functor` 就是可以把一个函数应用于一个 **封装过的值** 上，得到一个新的 **封装过的值**

`Functor` 最早出自于代数拓扑，这里说的 `Functor` 一般是指范畴论（Category Theory）中的 `Functor`，它被用来描述各种范畴间的关系。更多 Functor 的理解 [Group Theory and Category Theory](http://blessingsoft.com/2017/06/12/group-theory-and-category-theory/)。

### Applicative

```swift
func pure<A>(_ x: A) -> Parser<A> {
    return Parser<A> { (x, $0) }
}
```

实际上前面的 `pure` 和 `discarding` 函数就是一种 Applicative。像 `discarding` 一样有时候只关心这些要解析道字符存在就行了，上面定义的 `discarding` 解析器作用是忽略第一个 parser 参数的解析结果，同样地，可以定义一个忽略第二个 parser 参数的解析器。比如当解析出现在右边的 symbol 的时候就很有用，`discarding2(parser, string(")"))` 的作用就是确保存在闭合的右括号 ")"。

```swift
/// 吃掉右边的结果
func discarding2<A, B>(_ x: Parser<A>, _ y: Parser<B>) -> Parser<B> {
    return Parser { input in
        guard let (result1, remainder1) = x.parse(input) else {
            return nil
        }
        guard let (_, remainder2) = y.parse(remainder1) else {
            return nil
        }
        // 只保留左边的解析器的结果 result1，没有 result2
        return (result1, remainder2)
    }
}
```

```swift
/// 吃掉左边的结果
func discarding1<A, B>(_ x: Parser<A>, _ y: Parser<B>) -> Parser<B> {
    return Parser { input in
        guard let (_, remainder1) = x.parse(input) else {
            return nil
        }
        guard let (result2, remainder2) = y.parse(remainder1) else {
            return nil
        }
        // 只保留右边的解析器的结果 result2，没有 result1
        return (result2, remainder2)
    }
}
```

这两个 `discarding` 函数长的很像，如果有办法把它们抽象一下，把相似的地方提取出来就好了。

对于两个结果，忽略其中一个，实际上很简单：

```swift
/// 忽略 B
func f<A, B>(_ a: A, _ b: B) -> A {
    return a
}
```

```swift
/// 忽略 A
func f<A, B>(_ a: A, _ b: B) -> B {
    return b
}
```

```swift
/// 吃掉左边的结果
func left<A, B>(_ x: Parser<A>, _ y: Parser<B>) -> Parser<B> {
    
    func discarding<A, B>(_ a: A, _ b: B) -> A {
        return b
    }
    
    return Parser { input in
        guard let (result1, remainder1) = x.parse(input) else {
            return nil
        }
        guard let (result2, remainder2) = y.parse(remainder1) else {
            return nil
        }
        return (discarding(result1, result2), remainder2)
    }
}
```

然后这个函数并没有卵用。

### Monad

观看 `map` 函数 `func map<A, B>(_ x: Parser<A>, _ f: (A) -> B) -> Parser<B>`

它要求传入两个参数，一个是 `Parser<A>`，一个是函数 `A -> B`，第二个参数对标题中的 **Combinator** 并不是很友好，**Parser Combinator** 的思想是组合一系列的 `Parser` 得到结果。上面定义了有很多小的 parser，比如 `func string(_ str: String) -> Parser<String>`，函数签名是 `(String) -> Parser<String>`，由于 `map` 函数的第二个参数的签名是 `(A) -> B`，而非 `(A) -> Parser<B>`，所以假如存在一个与 `map` 功能相似，但第二个参数的签名是 `(A) -> Parser<B>`，则能够使得之前定义的很多小的 `parser` 能够直接作为一个参数，直接得到一个新类型的 `Parser`，大概这样：

```swift
func flatMap<A, B>(_ x: Parser<A>, _ f: (A) -> Parser<B>) -> Parser<B>
```

使用的时候：

```swift
let parser = flatMap(stringParser, string("alex"))
```

具体实现与 `map` 也很像：

```swift
func flatMap<A, B>(_ x: Parser<A>, _ f: @escaping (A) -> Parser<B>) -> Parser<B> {
    return Parser { input in
        guard let (result, remainder) = x.parse(input)else {
            return nil
        }
        return f(result).parse(remainder)
    }
}
```

[Group Theory and Category Theory](http://blessingsoft.com/2017/06/12/group-theory-and-category-theory/)

### Alternative

```swift
func empty<A>() -> Parser<A> {
    return Parser { _ in nil }
}
```

```swift
func choice<A>(_ x: Parser<A>, _ y: Parser<A>) -> Parser<A> {
    return Parser { input in
        return x.parse(input) ?? y.parse(input)
    }
}
```

Alternative 类似于 `Swift Standard Library` 中定义的运算符 `??`，它有两个同类型的参数，第一个参数是偏爱的 `parser`，第二个参数是默认的 `parser`。它首先尝试使用第一个 `parser` 来进行解析，如果成功，则返回。如果不成功，则使用默认的 `parser` 进行解析。它的返回值类型也是同类型的 `Parser`。

作用是假如有 Int, String, Bool 三个类型的 `parser`，而一个 scalar 类型的 `parser` 只要能够解析 Int, String, Bool 任意一种类型，则算解析成功。换句话说就是 scalar 是 Int, String, Bool 的父集。一种简单的从 `Parser<Int>`, `Parser<String>`, `Parser<Bool>` 三种已有实现的 parser 得到 `Parser<Scalar>` 的方法是逐个进行 parse，如果成功则马上返回。

```swift
let scalar = parserInt <|> parserString <|> parserBool
```

从这个例子看有点 one of 的意思，但实际上更加准确的说法是 choice。

### Applicative & Monad

Applicative 和 Monad 的区别在于：

Applicative 的两个 parser 是相互独立的，组合后的新 parser 是可以静态分析其行为的。而对于 Monad，在不知道输入的情况下，是不能确定其行为，也就是说 Monad 是依赖于计算结果。

```swift
func alex(_ x: Parser<String>) -> Bool {
    if let (_, _) = x.parse("alex.huo") {
        return true
    }
    return false
}

let af: Parser<(String) -> String> = pure(id)
let ax = string("alex")
alex(af <*> ax) // true

let mf: (String) -> Parser<String> = { string($0) }
let mx = string("alex")
alex(mx >>- mf) // false
```

## Ref

[Jiffy](https://github.com/hlian/jiffy)
[Parser combinators](https://news.realm.io/news/tryswift-yasuhiro-inami-parser-combinator/)
[Monadic Parser Combinators](http://www.cs.nott.ac.uk/~pszgmh/monparsing.pdf)


