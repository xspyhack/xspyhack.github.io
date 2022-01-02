---
title: Plist Parser
date: 2017-08-10 20:00:00
tags:
	- plist parser
	- parser
	- parser combinator
	- monad
	- functional programming
	- swift
categories: functional programming
---
Plist 是 Apple 家平台上一种很常见的配置文件，常见的存储格式是常见的 XML 格式（还有 Binary 格式），不同于 HTML 的复杂，Plist 只包含了比较少的几种标签（tag），所以实现使用 functional 的 parser combinator 来实现一个简单的 plist parser 也是一件很有意思的事情。

## Plist

一个 Plist 文件内容长这个样子：

```
<dict>
    <key>number</key>
    <integer>0</integer>
    <key>date</key>
    <date>2017-08-05T14:25:14Z</date>
    <key>data</key>
    <data>VGVzdFZhbHVl</data>
    <key>boolean</key>
    <true/>
    <key>array</key>
    <array>
        <string>string</string>
        <false/>
        <integer>0</integer>
    </array>
 </dict>
```

[wikipedia](https://zh.wikipedia.org/zh-hans/属性列表) 上列出了一个详细的 `XML` 标签和 macOS/iOS 中的类型关系以及存储格式。

| Foundation 类 | Core Foundation 类型 | XML 标签 | 储存格式 |
| :--- | :--- | :--- | :--- |
| NSString | CFString | &lt;string&gt; | UTF-8 编码的字符串 |
| NSNumber	| CFNumber | &lt;real&gt;, &lt;integer&gt; | 十进制数字符串 |
| NSNumber | CFBoolean	| &lt;true/&gt;, or &lt;false/&gt; | 无数据（只有标签）|
| NSDate | CFDate | &lt;date&gt; | ISO 8601 格式的日期字符串 |
| NSData | CFData | &lt;data&gt; | Base64 编码的数据 |
| NSArray | CFArray | &lt;array&gt; | 可以包含任意数量的子元素 |
| NSDictionary | CFDictionary | &lt;dict&gt; | 交替包含 &lt;key&gt; 标签和 plist 元素标签 |

根据这个表格，我们可以定义出 Plist 的数据结构。

## Model

```swift
/// The plist data model
public enum PLIST {
    /// <true/> or <false/>
    case bool(Bool)
    
    /// 2017-08-05T14:25:14Z
    case date(Date)
    
    /// <data>VGVzdFZhbHVl</data> (<54657374 56616c75 65>
    case data(Data)
    
    /// <integer>233</integer> or <real>2.33</real>
    case number(Int)
    
    /// <string>string</string>
    case string(String)
    
    /// <array><string>The String</string></array>
    indirect case array([PLIST])
    
    /// <dict><key>The Key</key><string>The String</string></dict>
    indirect case dict([String: PLIST])
}
```

## Parser

根据 Plist data model，想要解析一个 Plist 字符串 得到 `PLIST` 类型，只需要一个 `parser`。

没错，只需要一个 parser，这个 parser 大概长这样：

```swift
let parser: Parser<PLIST>
let result = parser.parse("plist")
```

这个 `let parser: Parser<PLIST>` 的实现才是最关键的。一个 `PLIST` 是由 `Bool` `Date` `Data` `Number` `String` 5 种简单的类型和 `Array<PLIST>` `Dictionary<PLIST>` 2 种容器（nested）类型组成，所以一个 `Parser<PLIST>` 也是由对应的 `Parser<Bool>` `Parser<Date>` `Parser<Data>` `Parser<Number>` `Parser<String>` 5 中简单的 parser 和 `Parser<Array>` `Parser<Dictionary>` 2 种容器类型 parser 组成。

### Bool Parser

在 Plist 中，Bool 类型由两种形式 `<true/>` 和 `<false/>`，所以一个 Bool 类型的 parser 也就是能够解析字符串 `<true/>` 和 `<false/>`。

```swift
let _true = string("<true/>") <&> const(PLIST.bool(true))
let _false = string("<false/>") <&> const(PLIST.bool(false))

let _bool = _true <|> _false
_bool.parse("<false/>")
```

### Date Parser

Plist 中的 Date 类型存储的是 UTC 字符串，如 `<date>2017-08-05T14:25:14Z</date>`。字符串中的开始标签 `<date>` 和结束标签 `</date>` 对于解析的结果来说是没有用的，所以一个 Date 类型的 parser 是要将这个字符串解析成 `PLIST.date(date)`, date 为 2017-08-05T14:25:14Z 通过 format 得到。

```swift
let _date = string("<date>") *> manyTill(_any, string("</date>")) <&> { PLIST.date(String($0).date!) }
_date.parse("<date>2017-08-05T14:25:14Z</date>")

/// UTC Date
public extension String {
    public var date: Date? {
        let formatter = DateFormatter()
        formatter.dateFormat = "yyyy-MM-dd'T'HH:mm:ss'Z'"
        return formatter.date(from: self)
    }
}
```

### Data Parser

Plist 中的 Data 类型存储的是 Base64 编码后的数据，所以实现一个 Data Parser 和 Date Parser 差不多，区别是 tag 和 Data 类型初始化。

```swift
let _data = string("<data>") *> manyTill(_any, string("</data>")) <&> { PLIST.data(Data(base64Encoded: String($0))!) }
let dataString = _data.parse("<data>VGVzdFZhbHVl</data>")
```

### Number Parser

Plist 中的 Number 的存储实际上分两种。一种是整型，一种是浮点型。整型的 `tag` 是 `integer`，浮点型是 `real`。

先看 Integer Parser：

```swift
let _integer = string("<integer>") *> manyTill(_digit, string("</integer>")) <&> { PLIST.number(Int(String($0))!) }
```

### String Parser

String Parser 和 Date Parser 以及 Data Parser 对比起来更简单，实际上就是去掉了最后转换的那一步。

```swift
let _string = string("<string>") *> manyTill(_any, string("</string>")) <&> { PLIST.string(String($0)) }
_string.parse("<string>The String</string>")
```

### Tag Parser

通过对比上面几种除了 Bool Parser 之外不同类型的 Parser，可以发现实现的方式很相似。

* closed tag，成对存在。
* 中间存储的都是字符串，最后把字符串转为具体类型。

把这些相似的 Parser 进行抽象，将相同部分封装成一个函数，不同的部分用传参的形式来实现。

```swift
func tag<A>(_ tag: String, _ p: Parser<A>) -> Parser<[A]> {
    return string("<\(tag)>") *> manyTill(p, string("</\(tag)>"))
}

let _date1 = tag("<date>", _any) <&> { PLIST.date(String($0).date!) }
_date1.parse("<date>2017-08-05T14:25:14Z</date>")

let _string1 = tag("string", _any) <&> { PLIST.string(String($0)) }
_string1.parse("<string>The String</string>")
```

### Array Parser

Array Parser 和 Dictionary Parser 相对比较复杂，因为它们是容器类型，里面可以是任意的 PLIST 类型，包括它们本身。对于 Enum PLIST 来说，可以使用 `indirect` 关键字来表示这种情况，但是在定义 parser 的时候，确没有这些魔法。

但是通过利用 Swift 的一些特性，还是很容易解决这个递归的问题。先忽略 Dictionary 类型。

```swift
let _plist = plist()
func plist() -> Parser<PLIST> {
    return _bool <|> _string <|> _integer <|> _date <|> _data <|> _array
}

let _array = tag("array", _plist) <&> {
    PLIST.array($0)
}
```

### Dictionary Parser

Dictionary Parser 的递归问题和 Array Parser 一样。

```swift
let _plist = plist()
func plist() -> Parser<PLIST> {
    return _bool <|> _string <|> _integer <|> _date <|> _data <|> _array <|> _dict
}

let _dict = tag("dict", ?) <&> {
   /// 转换为 PLIST.dict
}
```

但 Dictionary 和 Array 不一样的地方在于，Array 里面是多个 Plist 的元素，而 Dictionary 是 key-value 对，且必须是 key-value 对，也就是 `tag("dict", _keyValue)`。

先实现一个 Key-Value Parser：

```swift
let _key = string("<key>") *> manyTill(_any, string("</key>")) <&> { String($0) }
let _keyValue = ({ a in { b in (a, b) }} <^> _key <*> (value))
```

然后就可以得到 Dictionary Parser：

```swift
let _dict = tag("dict", _keyValue) <&> { PLIST.dict(atod($0)) }
/// Tuple Array to Dictionary
public func atod<Key: Hashable, Value>(_ tuples: [(Key, Value)]) -> [Key: Value] {
    var dict: [Key: Value] = [:]
    for (key, value) in tuples {
        dict[key] = value
    }
    return dict
}
```

或者换一种写法：

```swift
let _kv = _keyValue <&> { ttod($0) }

let _dict1 = tag("dict", _kv) <&> {
    PLIST.dict(
    		$0.flatMap { $0 }
        .reduce([String: PLIST]()) { d, kv in
            var dict = d
            dict.updateValue(kv.value, forKey: kv.key)
            return dict
        })
}

public func ttod<Key: Hashable, Value>(_ tuple: (Key, Value)) -> [Key: Value] {
    return [tuple.0: tuple.1]
}
```

### Plist Parser

最后

```swift
let _plist = _bool <|> _string <|> _integer <|> _date <|> _data <|> _array <|> _dict

let result = _plist.parse(plist)
dump(result)
```

## Ref

[属性列表](https://zh.wikipedia.org/zh-hans/属性列表)
[Parser Combinator](http://blessingsoft.com/2017/05/28/parser-combinator/)
[解析组合子](https://github.com/nixzhu/dev-blog/blob/master/2017-04-12-json-parser.md)


