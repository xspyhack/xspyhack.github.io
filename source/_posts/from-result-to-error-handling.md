---
title: From Result to Error Handling
date: 2018-11-19 20:00:00
tags:
    - swift
    - error handling
    - result
categories: advanced swift
---
> 最近关于 [Add Result to the Standard Library](swift-evolution/0235-add-result.md at master · apple/swift-evolution · GitHub) 的提案正在激烈的[讨论中](SE-0235 - Add Result to the Standard Library - Proposal Reviews - Swift Forums)，讨论的内容从命名到异步错误处理，再到是否应该有一个 `Either` 类型等等。

## Result

对于在项目中使用过 Swift 的人来说，`Result` 类型应该再熟悉不过了，在 community 中有着非常广泛的应用。最早看到对于 `Result` 的应用是在[Alamofire](https://github.com/Alamofire/Alamofire) 中，然后是有订阅的博客开始介绍 `Result` 是如何帮忙解决非必要的错误值/可选值检查，明确的区分成功和失败。再后来就是大家都开始在项目中使用 `Result` 类型来进行 **异步错误处理**。

为什么说 **异步错误处理** 呢，因为最早接触到 `Result` 这个类型的使用案例，就是用来处理异步错误的，并且它非常的适合，如果不从语言设计上考虑的话，它可以说是非常完美，因为它和 `Optional` 一样是个 Monad。虽然 `Result` 只是一个非常简单的数据结构，它和同步异步一点关系都没有，它只跟错误处理有关。

## Error Handling

错误处理按执行顺序上可以分为同步（synchronization）和异步（concurrency）两种。

同步错误处理在 Cocoa 中有两种，一种是 `throw` + `try catch`，遇到异常的时候函数内部通过 `throw`/`raise`等关键词把异常信息抛出，调用方通过 `try catch` 进行捕获。另一种是古老的 C 中的 Error 指针，调用方通过把 Error 指针作为参数传到函数内部，当遇到错误时，给 Error 指针赋值，以达到把错误信息往外传递的目的。

异步错误处理在 Cocoa 中一般是通过 block, delegate 或者 notification 等方式进行传递，但大多数接口通常会把正常结果回调和异常结果回调合并一起进行回调，好处在于不需要两个 block 或者两个 delegate 函数或者 notification，缺点也就是上面的 `Result` 解决的问题。除此之外也有一些比较少见的方式，比如 [AVAssetExportSession](https://developer.apple.com/documentation/avfoundation/avassetexportsession)，它的 `completionHandler` 是 `@escaping () ->  Void`，并不携带任何正常和错误的信息，而是通过 `var status: AVAssetExportSession.Status` 和 `var error: Error?` 等属性来提供。

抛开遥遥无期的 async/await 不谈，对于异步错误处理来说，可能由于网络库之类的接触的太多，所以平时去设计 API 的时候都非常的顺手的就写出来了，要么 `completionHandler: (Value?, Error?) -> Void`，要么 `completionHandler: (Result<Value>) -> Void`。但是对于同步错误处理却不是这样。

首先在 Swift 中推荐的错误处理是 `throw` + `try catch`，所以 Error 指针是不需要再讨论的。但不知道是因为 `throw` + `try catch` 难用，还是因为懒，一般的项目中其实很少见到 `throw/throws/rethrows` 这样的关键词（ObjC 中很少见到 throw/raise 同理）。

注：异步中是无法直接使用 `throw` + `try catch`，下面的两种写法都是不合法的：

```swift
func remove(forKey key: StoreKey) throws {
    queue.async {
        let url = try disk.url(atPath: path(forKey: key), in: directory)
        try disk.remove(at: url)
    }
}

func remove(forKey key: StoreKey) throws {
    try queue.async {
        let url = try disk.url(atPath: path(forKey: key), in: directory)
        try disk.remove(at: url)
    }
}
```

### 万恶的 return

一个当前在做的 Alligator 项目中的代码片段：

```swift
/// 视频导出
///
/// - Parameter asset: 要导出的视频资源
/// - Parameter outputURL: 指定的导出地址
func export(asset: AVAsset, to outputURL: URL) {
    guard outputURL.isFileURL else {
        assertionFailure("output url must be file url.")
        return
    }
    ...
    guard let exportSession = AVAssetExportSession(asset: asset, presetName: AVAssetExportPresetHighestQuality) else {
        return
    }

    exportSession.outputURL = outputURL
    exportSession.outputFileType = .mp4
    ...
}
```

这个函数的大概功能是进行配置和导出视频到文件，看起来是不是很熟悉？有参赛合法性的判断，提供开发调试帮助的 assertion，`guard` 的使用也很合理。

再看一段：

```swift
// VideoProcessor.swift
func prepare() {
    let videoWriterInput = 
    let audioWriterInput = 
    guard let videoWriter = try? AVAssetWriter(outputURL: outputURL, fileType: .mp4) else {
        return
    }
    
    if videoWriter.canAdd(videoWriterInput) {
        videoWriter.add(videoWriterInput)
    } else {
        assertionFailure("can't add video writer input")
    }

    if videoWriter.canAdd(audioWriterInput) {
        videoWriter.add(audioWriterInput)
    } else {
        assertionFailure("can't add audio writer input")
    }
    ...
}
```

类似这样的代码在 project 中应该非常常见，但是却有着非常大的问题，那就是 **故意的忽略异常**。只处理了一切正常执行的分支，当遇到异常的时候，直接 return 或是加上 assertion 信息。这样写大多数情况下都没有什么问题，功能也正常，即使是遇到了异常情况，也不会引起 crash，但却有着很大的缺陷。（甚至有很多人连 assertion 都不用，替而代之的是 `print` :P

在这个例子中这些视频处理的逻辑实际上是相对比较独立的，功能也比较”单一”，这些异常一旦出现，后续的逻辑基本都是不可用，并且大多数的异常在实际应用中是需要被调用方知道并处理的，比如体现在 UI 上。

一个带异常处理（滑稽）的视频处理的示例代码片段：

```swift
// AVAsset+Processor.swift

extension Alligator where Base: AVAsset {
    /// Merge the given video asset and audio asset
    ///
    /// - Parameters:
    ///   - videoAsset: the given video asset
    ///   - audioAsset: the given audio asset
    /// - Returns: the merged asset
    /// - Throws: throws error when the given asset is invalid. e.g. video asset without video tracks.
    public static func merge(videoAsset: AVAsset, audioAsset: AVAsset) throws -> AVAsset {
        let mixComposition = AVMutableComposition()
        
        try mixComposition.agt.add(.video, from: videoAsset)

        let videoDuration = mixComposition.duration
        try mixComposition.agt.add(.audio, from: audioAsset, maxBounds: videoDuration)

        return mixComposition
    }

    /// Merge the given assets one by one
    ///
    /// - Parameters:
    ///   - segments: given assets, it can't be empty
    ///   - isMuted: if true, it will passthrough audio tracks
    /// - Returns: merged asset
    /// - Throws: throws error when segments is empry, or some segment is invalid.
    public static func merge(segments: [AVAsset], isMuted: Bool) throws -> AVAsset {
        guard !segments.isEmpty else {
            throw Error.segmentsEmpty
        }

        if segments.count > 1 {
            let mixComposition = AVMutableComposition()
            try mixComposition.agt.add(segments, isMuted: isMuted)
            return mixComposition
        } else {
            return segments[0]
        }
    }
}
```

### 无脑的 Optional

无脑的 Optional 指的是，对于一个有明确返回值类型 `T` 的函数，有可能出现某个入参不符合要求的情况，就把返回值改成 `T?`，用 `return nil` 来处理异常。如：

```swift
struct Formatter {
    /// Format a string, replace invalid symbol with empty character
    ///
    /// - Parameter string: string needs to be format
    /// - Returns: formatted string
    func format(_ string: String) -> String? {
        guard string.isEmpty else {
            return nil
        }
        return string.replacingOccurrences(of: "\n", with: "")
    }
}
```

应该大多数人都试过这样，并且甚至有人一直都是这样，不经思索。有人会觉得这样写并没有什么问题。那么再看：

```swift
struct Formatter {
    /// Format a string, replace invalid symbol with empty character. If it is empty or contains `@`, `#`, return nil.
    ///
    /// - Parameter string: string needs to be format
    /// - Returns: formatted string
    func format(_ string: String) -> String? {
        guard string.isEmpty else {
            throw nil
        }

        guard !string.contains("@") else {
            return nil
        }

        guard !string.contains("#") else {
            return nil
        }

        return string.replacingOccurrences(of: "\n", with: "")
    }
}
```

这样写有没有问题呢？或者说有没有更好的方案呢？

```swift
struct Formatter {
    enum Error: Swift.Error {
        case emptyString
        case containsHashtag
        case containsMention
    }

    /// Format a string, replace invalid symbol with empty character
    ///
    /// - Parameter string: string needs to be format
    /// - Returns: formatted string
    func format(_ string: String) throws -> String {
        guard string.isEmpty else {
            throw Error.emptyString
        }

        guard !string.contains("@") else {
            throw Error.containsMention
        }

        guard !string.contains("#") else {
            throw Error.containsHashtag
        }

        return string.replacingOccurrences(of: "\n", with: "")
    }
}
```

其实这个问题可以归为，**对于同步 API 的异常处理，什么时候应该使用 throw？，什么时候可以返回 nil？**

这是一个很大的话题，并且大多数情况下需要根据场景来选择。通过对比这两种设计，可以简单的理解为如果希望使用方以更加合适的方式来处理错误，错误信息分类清晰详细，那么应该使用 `throw`。是否需要隐藏异常，交给使用方来决定。如果错误比较单一明确，可以考虑使用 `Optional`。

### 混淆的人为错误和程序错误

简单来说对于人为错误，应该通过 `assertion`, `precondition`, `fatalError` 等来帮助在开发测试阶段发现问题。而对于程序错误，应该根据同步或者异步来区分处理，使得程序能继续正常的工作。

人为错误一般是指手误参数传错这种，如果没有手误（比如拼错单词、下标越界等），从逻辑上说不可能发生这种情况。

```swift
func tableView(_ tableView: UITableView, cellForRowAt indexPath: IndexPath) -> UITableViewCell {
    guard let cell = tableView.dequeueReusableCell(withIdentifier: "cell", for: indexPath) else {
        fataError("不应该啊兄弟")
    }
    return cell
}
```

程序错误更多是指所有参数都没有错误，但还是遇到异常了，并且不能忽略，比如磁盘满了导致无法写入文件。

> 最后值得一提的就是大多数人在写 ObjC 的时候都会选择性的忽略异常，经典的场景就是设计 API 的时候滥用 `id`，然后虽然在方法内部对参数类型进行了判断，但在出现参数类型不合法的时候，直接通过 `return` 来处理。

## 参考

[Swift Package Manager](https://github.com/apple/swift-package-manager/blob/master/Sources/Basic/Result.swift)
[Result&lt;T&gt; 还是 Result&lt;T, E: Error&gt;](https://onevcat.com/2018/10/swift-result-error/)


