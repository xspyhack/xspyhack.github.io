---
title: @_silgen_name
date: 2022-07-16 20:00:00
tags:
    - swift
    - silgen_name
    - router
    - sil
categories: advanced swift
---
> 在设计 `Router` 组件的时候，其中必不可少的一步是路由注册，将 `pattern` 和 `handler` 函数进行绑定。

## 使用 @_silgen_name 来进行注册

```swift
@_silgen_name("/user")
func user(context: Context) -> Bool {
    ...
}
```

## 实现原理

### 什么是 @silgen_name

根据 Swift 的官方手册，`@silgen_name` 有两个作用：

> 1. To specify the symbol name of a Swift function so that it can be called from Swift-aware C. Such functions have bodies.
> 2. To provide a Swift declaration which really represents a C declaration. Such functions do not have bodies.
> 
>  [swift/StandardLibraryProgrammersManual.md at main · apple/swift · GitHub](https://github.com/apple/swift/blob/main/docs/StandardLibraryProgrammersManual.md#_silgen_name)

所以可以利用第2个特性给 `handler` 函数指定一个导出函数符号 `Exported Symbols`，然后通过将 `pattern` 作为导出符号即可绑定对应的 `handler` 函数。

同时为了避免与其它导出符号冲突并且提高查找的效率，可以给导出符号加一个约定的前缀，如 `ditto:`。

### 注册

运行时利用 `_dyld_register_func_for_add_image()` 来监听所有的 `image` 加载，然后遍历带特定前缀的导出函数符号。

```swift
_dyld_register_func_for_add_image { image, slide in
    let mhp = image?.withMemoryRebound(to: MachHeader.self, capacity: 1, { $0 })
    let symbols = getDyldRouteSymbols(prefix: "ditto:", image: mhp!, slide: slide)
}
```

然后通过 `unsafeBitCast(_:to:)` 函数将函数符号转为 `handler` 函数进行注册。

```swift
let routes: [(String, Route<Coordinator>.Handler)] = symbols
    .filter { $0.key.hasPrefix("ditto:") }
    .map { (String($0.key.dropFirst("ditto:".count)), unsafeBitCast($0.value, to: Handler.self)) }
try? register(routes)
```

## 进阶
[Ditto](https://github.com/xspyhack/Ditto) 在设计之初便支持范型，并且可以同时存在多个 `Router`（虽然没太必要）。

### 范型

`_dyld_register_func_for_add_image()` 是一个 `C` 函数，支持传入一个回调函数指针作为参数，定义如下：

```c
void _dyld_register_func_for_add_image(void (*func)(const struct mach_header* mh, intptr_t vmaddr_slide));
```

回调函数并不同于一般的 `closure`，它不能捕获上下文信息：`A C function pointer cannot be formed from a closure that captures context`。所以对于支持范型的 `Router` 来说，就不能直接的调用这个函数。

所以我们需要用过一些其他手段先把遍历到的函数符号先存起来，再重新读取出来即可。

```swift
class SIL {
    private(set) var symbols: [UnsafeMutableRawPointer?] = []
    static let shared = SIL()

    static func install() {
        _dyld_register_func_for_add_image { image, slide in
            let mhp = image?.withMemoryRebound(to: MachHeader.self, capacity: 1, { $0 })
            let symbols = getDyldRouteSymbols(image: mhp!, slide: slide)
            SIL.shared.symbols.append(contentsOf: symbols)
        }
    }
}

extension Router {
    typealias Handler = @convention(thin) (Context<Coordinator>) -> Bool
    public func register() {
        SIL.install()
        DispatchQueue.main.async {
            self.register(symbols: SIL.shared.symbols)
        }
    }

    private func register(symbols: [UnsafeMutableRawPointer?]) {
        let routes: [(String, Route<Coordinator>.Handler)] = symbols
            .map { (String($0.key), unsafeBitCast($0.value, to: Handler.self)) }
        try? register(routes)
    }
}
```

### 模块化

假如同时存在多个 `Router` 的话，那么只靠 `pattern` 无法区分要注册到哪一个。同时由于 `_dyld_register_func_for_add_image()` 函数无法捕获上下文，也无法传递指定的 `silgen_name` 前缀来进行分开遍历。

一个可行但并不优雅的方法依然是在 `silgen_name` 的命名中做文章，如约定特殊的前缀来进行区分。

如果不同的 `Router` 是在不同的模块中（`module`），那么可以通过遍历指定的 `image` （通过 `_dyld_get_image_name()` 方法）来注册当前模块的符号。

[swift/StandardLibraryProgrammersManual.md](https://github.com/apple/swift/blob/main/docs/StandardLibraryProgrammersManual.md#_silgen_name)
[Should anyone be using @_silgen_name outside of the standard library developers?](https://forums.swift.org/t/should-anyone-be-using-silgen-name-outside-of-the-standard-library-developers/19396/23)