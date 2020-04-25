---
title: How to design a lightweight Logging System
tags:
    - swift
    - logger
    - logging
categories: advanced swift
---
> 之前项目一直都是使用 [CocoaLumberjack](https://github.com/CocoaLumberjack/CocoaLumberjack) 来定制 Log 系统，这次整个工程使用 Pure-Swift 来开发，而且作为一个初创项目，对于 Log 系统的需求并没有那么高，虽然 [CocoaLumberjack](https://github.com/CocoaLumberjack/CocoaLumberjack) 在 Swift 项目中直接使用也比较友好，但是感觉还是 ~~太重了~~ 。所以为何不直接设计一个比较轻量级的日志系统呢？

## Logging System

在 iOS 中，可以用 Swift 中的 `print()` 和 `debugPrint()` 函数来向 Xcode Console 来打印信息，也可以使用 Foundation 中的 `NSLog()` 来打印，更新的就是 [os_log](https://developer.apple.com/documentation/os/logging) 了。这三种不同的方式有不同的特点。

* **print**/**debugPrint**: Swift 语言层面提供的实现，可以输出到 Xcode Console，但不能输出到 Apple System Logs（Mac Console.app）。
* **NSLog**: Foundation 中的实现，除了能在 Xcode Console 中输出外，还会往 Console.app 发，并且有较大的性能损害。
* **os_log**: 待补充 [Unified Logging and Activity Tracing - WWDC 2016 - Videos - Apple Developer](https://developer.apple.com/videos/play/wwdc2016/721/) [Measuring Performance Using Logging - WWDC 2018 - Videos - Apple Developer](https://developer.apple.com/videos/play/wwdc2018/405/)

Xcode Console 和 Apple System Logs 都是需要物理接触设备才能看到 log。但是在项目中，如果想要看到线上用户的 log 信息，必须要把这些 log 写到本地文件中，或者实时/定时发到远端服务器上。这时候直接使用内置的 API 是无法满足需求的。

对于一个相对比较合理的日志系统，一般有几点要求：

* 在 release 下禁止输出日志到 Xcode Console
* ~~在 release 下~~ 禁止输出日志到 Apple System Logs
* 提供输出日志到本地文件中的能力
* 能够方便的扩展，如直接输出到 Web
* 对于日志根据重要性划分为不同的等级
* 可以根据日志等级，过滤日志

## Design

### Level

Level 有两个作用：

* 定义一条日志的重要性
* 对日志进行过滤，比如过滤掉某个等级以下的日志（实际是有包含的关系

这里通过两个类型来实现 level 的作用。

```swift
public enum Level: Int {
    case off = 0
    case error = 1 // Flag.error | Level.off
    case warning = 3 // Flag.warning | Level.error
    case info = 7 // Flag.info | Level.warning
    case debug = 15 // Flag.debug | Level.info
}

public struct Flag: OptionSet {
    public let rawValue: Int

    public static let error = Flag(rawValue: 1 << 0) // 1
    public static let warning = Flag(rawValue: 1 << 1) // 2
    public static let info = Flag(rawValue: 1 << 2) // 4
    public static let debug = Flag(rawValue: 1 << 3) // 8

    public init(rawValue: Int) {
        self.rawValue = rawValue
    }
}
```

### Message & Formatter

一条日志如果只有正文部分，很难帮助定位具体的位置和发生的时间点，所以一条更加有意义的日志，会带上所在的文件、函数、行数以及时间戳等信息。

首先定义一个 `Message` 的数据结构来定义一条日志，这些信息最终具体如何 format 成一条字符串，需要提供一个 `Formatter` 来实现。

既然是 Pure-Swift，所以这里 `Message` 使用 `struct` （Value Type），而不是 `class` （Reference Type）。（`Codable`, `CustomStringConvertible`, `Equatable` 什么的暂时不需要考虑。

对于 `Formatter`，可能不同的 logger 需要不同的 format，比如输出到本地文件的 logger 需要更详细的信息，比如时间戳，才好帮助日后还原 app 当时运行的情况，而输出到 Xcode Console 的日志一般是在开发的时候看的，所以时间戳可能就没那么重要。（为什么 `Formatter` 使用 `protocol`？可以先想想，后面再解释。

```swift
public struct Message {

    public let message: String

    public let level: Level

    public let flag: Flag

    public let context: Int

    public let file: String

    public let function: StaticString

    public let line: UInt

    public let timestamp: Date
}

public protocol Formatter {
    func format(message: Message) -> String
}
```

### Logging

根据日志输出的目标不同，可以划分为不同类型的 logger，比如 `ConsoleLogger`、`FileLogger` 以及 `WebLogger` 等。每一种不同的 logger 都有一些同样的接口，所以第一反应有两种不同的方式来实现类型的划分以及相同接口（行为）的约束。

使用面向对象的思想，通过继承来实现。

```swift
open class Logging {
    public enum Type {
        case console
        case file
        case web
    }

    open var type: Type {
        return .console
    }
    
    open func log(_ message: String) {
        fatalError("must override this method in subclass.")
    }
}

open class ConsoleLogger: Logging {
    override open var type: Type {
        return .console
    }
    override open func log(_ message: String) {
        //
    }
}
open class FileLogger: Logging {
    override open var type: Type {
        return .file
    }
    override open func log(_ message: String) {
        //
    }
}
```

使用面向协议的思想，通过协议来约束行为。

```swift
protocol Logging {
    var type: Type { get }

    func log(_ message: String)
}

struct ConsoleLogger: Logging {
    var type: Type {
        return .console
    }

    func log(_ message: String) {
        //
    }
}

struct FileLogger: Logging {
    var type: Type {
        return .file
    }

    func log(_ message: String) {
        //
    }
}
```

两种不同的实现，体现的是两种不同的思想：
* 一种是使用面向对象的思想，通过一个基类来提供相同的接口，然后子类重写这些接口来提供不同的能力；
* 另一种是使用面向协议的思想，通过一个协议来对接口进行约束，每一个具体的实现都必须实现这些接口来提供不同都能力。

而对于不同类型的 logger 都共有的行为，前一种方式可以直接在基类中实现，后一种方式可以通过 `protocol extension` 来提供默认实现。

两种方式各有优缺点，如果你也不喜欢前一种 **需要运行时才能知道子类必须重写父类的某个方法**，完全不能体现出 Swift 作为一门有着强大类型安全的静态语言的优势，那么这里毫不犹豫的选择后一种方式。（不解释

对于这里的 `Type`，虽然使用 `enum` 有着很好的强类型信息，但这样写有着很大的约束，就是一开始就必须定义好所有的 Type，对于扩展性来说，非常不友好。

所以综合可扩展性和 `Type` 的作用考虑，这里通过添加一个 `String` 类型的属性 `name` 来简单的区分。（很方便于 debug

### Interface

最终得到一整个 Logging 相关的接口定义：

```swift
public protocol Logging {

    var formatter: Formatter { get }

    var name: String { get }

    var level: Level { get }

    func log(message: Message)

    func flush()

    func start()

    func teardown()
}

public extension Logging {

    var name: String {
        return "Unified"
    }

    func flush() {}

    func start() {}

    func teardown() {}
}
```

### Logger

前面定义了每个不同 `logger` 的接口（行为），但是使用的时候，如果需要手动调用每个 `logger` 的 `log(message:)` 方法，那就太没有意义了。所以需要一个数据结构，来管理所有的 `logger`，并且将消息转发到每一个 `logger`。

```swift
public class Logger {
    public static let shared = Logger()

    private var queue = DispatchQueue(label: "com.xspyhack.logger.queue")

    public private(set) var loggers: Set<AnyLogger> = []

    deinit {
        loggers.forEach {
            $0.teardown()
        }

        loggers = []
    }
}

public extension Logger {
   public func add(_ logger: Logging) {
        loggers.update(with: logger)
    }
}

public extension Logger {
    public func log(message: Message, asynchronous: Bool) {
        let work = DispatchWorkItem {
            self.loggers.forEach { logger in
                guard message.flag.rawValue & logger.level.rawValue != 0 else {
                    return
                }
                logger.log(message: message)
            }
        }

        if asynchronous {
            queue.async(execute: work)
        } else {
            queue.sync(execute: work)
        }
    }

    public func start() {
        loggers.forEach {
            $0.start()
        }
    }

    public func flush() {
        let work = DispatchWorkItem {
            self.loggers.forEach { logger in
                logger.flush()
            }
        }

        queue.sync(execute: work)
    }
}
```

这里只是比较粗糙的实现，很多细节还没有处理，比如 `loggers` 的线程安全问题、以及 `logger` 的删除等等。

### Log

有了 `Logging` 来定义每一种不同作用的 `logger`，以及一个管理所有 `logger` 的管理器 `Logger`（至于这个让人懵逼的命名，实际是因为懒，取一个别的名字比较适合，比如 ~~Charmander~~），还需要考虑最终如何简单的使用这个 Logging System。

现在如果要使用这个系统，首先需要实现自己的多种 `loggers` 和对应的 `Formatter`，然后添加到 `Logger` 里面，然后在需要打 log 的地方，初始化 一个 `Message`，调用 `Logger.shared.log(message:)` 方法。

这里每次初始化一个 `Message` 太麻烦了。如何简化？默认参数啊。

```swift
public static func log(_ message: @autoclosure () -> String, level: Level, flag: Flag, context: Int = 0, file: String = #file, function: StaticString = #function, line: UInt = #line, asynchronous: Bool = false) {

    let message = Message(message: message(), level: level, flag: flag, context: context, file: file, function: function, line: line, timestamp: Date())

    Logger.shared.log(message: message, asynchronous: asynchronous)
}
```

对于 `level` 和 `flag` 如果能提供默认参数，那么在调用的时候，就可以像 `print` 一样，直接只关注要 log 的内容就好了。一个比较简单直接的方法，就是针对这几种 `level` 暴露多个方法。

```swift
Log.d("This is a debug level log")
Log.i("This is an info level log")
Log.w("This is a warning level log")
Log.e("This is an error level log")
```

> 一个思考，上面的 `log` 函数中参数 message 的类型为什么使用 `@autoclosure`？

> 第二个思考，`Swift.print` 函数的定义你知道吗？

## Implementation

[GitHub - xspyhack/Keldeo: A lightweight logging library written in Swift.](https://github.com/xspyhack/Keldeo)

### Why AnyLogger

对于引入 `AnyLogger`，是因为我希望能够提供移除一个 `logger` 的能力，用处就是当我在脱离 Xcode 的时候，可以通过一种可以输出到浏览器的 `logger` 来实时看到日志输出。而这个名为 `WebLogger` 的 `logger` 平时并不会用到，所以它是需要的时候才添加进去，用完之后便移除，所以就涉及到 `logger` 必须实现 `Equatable`，实现从 `loggers` 里面移除它。

> Why `protocol Logging: Equatable {}` needs `AnyLogger`？见另一篇 [PATs](http://blessingsoft.com/2018/08/26/pats/)

## Ref

* [GitHub - CocoaLumberjack/CocoaLumberjack: A fast & simple, yet powerful & flexible logging framework for Mac and iOS](https://github.com/CocoaLumberjack/CocoaLumberjack)
* [Kingfisher/ImageCache.swift at master · onevcat/Kingfisher · GitHub](https://github.com/onevcat/Kingfisher/blob/master/Sources/ImageCache.swift)
* [swift/AnyHashable.swift at swift-3.0-branch · apple/swift · GitHub](https://github.com/apple/swift/blob/swift-3.0-branch/stdlib/public/core/AnyHashable.swift)



