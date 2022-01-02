---
title: HTTP Streaming/Chunked
date: 2020-04-15 20:00:00
tags:
    - swift
    - chunked
    - streaming
    - http
categories: networking
---
## TL;DR

使用 `Cocoa Foundation` 框架来实现 `HTTP Streaming/Chunked` 的时候，底层会将设置的 `Transfer-Encoding: chunked` 的值改成大写的 `Chunked` 发出去，如果后端不支持，那么必须通过比如 `hook` 的方式不让底层对这个值进行修改并发送出去。其次就是如果服务器已经返回了数据但是框架没有给任何回调，可能是因为服务器返回的 `Content-Type` 为 `text/html`，导致框架等待 `first 512 bytes` 数据，改用其他的类型即可。


## 背景

提供全球服务的能音视频通话的 IM 产品，大多数都是以 **自定义信令** 为基础来提升网络性通信能以及防止被封禁。

常见的几种通信方式有：

* IP/Port (TCP)
* TLS (TLS Resumption or TLS Verify)
* GCM (FCM)
* WebSocket
* HTTP Persistent Connection

前面几种都是其实都是通过 `IP/Port` 的方式来建立连接，那么一旦 IP 被封禁了，那么就只能通过其他手段下发 IP 地址然后不断的更换 IP 来绕过。 `GCM`([FCM](https://firebase.google.com/docs/cloud-messaging/)) 是谷歌提供的信令通道。`WebSocket` 需要服务器的支持，`Nginx` 和 `Amazon Cloudfront` 是支持的，但是 `Azure` 和 `Azure Domain Fronting` 不支持。所以最近为了进一步完善网络通信的建设，决定增加 HTTP 长链接的通道作为候补。


## HTTP Persistent Connection

HTTP Persistent Connection 也叫 HTTP Connection Reuse 或者其他的 Keep Alive 等。像 IM 类型应用，如果每一次通信都使用普通的 HTTP(s) 的话，那么由于每次建立 TCP 都需要三次握手，外加上 SSL/TLS 握手，可能需要 10+ 个 RTT，这对于 IM 来说是难以接受的。优化的方向也很明显，就是尽量避免 HTTPS 建立和关闭所带来的性能消耗，最好就是整个生命周期内，只建立一次关闭一次。

HTTP 1.1 中默认使用 `Keep-Alive` ，来达到连接复用，但仅仅是这个对于 IM 应用来说，还是太捉襟见肘了。然后对比 `Long Poll` 的方式，显然 `Streaming` 的方式性能会更加好。

> HTTP 2.0 也由于服务器不支持的原因，暂时不可行。


## Streaming/Chunked

HTTP 1.1 中支持了 chunked message （见 [Chunked transfer encoding - Wikipedia](https://en.wikipedia.org/wiki/Chunked_transfer_encoding)）。利用这个特性，我们可以通过 chunked 的方式，来避免重复建立连接。

网络请求包含了上行和下行，所以我们调研了两种方案：

1. 利用一条 URL 同时上下行 chunked 数据，发现 `cloudfront` 是支持的，但 `Azure` 不支持。Azure cdn 在收到第一个下行包时，会在上行包中插入 chunked 结束符，并不再转发上行包，但下行包仍然会被转发到客户端。❌
2. 利用两条 URL 来实现，一条s用于发送信令，一条用于读取信令。✅


### How to

既然是 HTTP 协议，那么首先想到的就是使用 `Foundation` 中的 `URLSession` 来实现。为了方便我们先把上行的 URL 叫做 `uplink` ，下行专用的 URL  叫做 `downlink`。

首先按照 `HTTP 1.1` 的标准，将 `uplink` 的请求头 `Transfer-Encoding` 设为 `chunked`。为了能给异步持续的将信令数据发送到服务器，需要利用 `Bound Pair Stream`，将 `InputStream` 连上 `httpBody`，然后持有 `OutputStream`，每次有数据需要发送的时候，就将数据从 `OutputStream` 写入，然后数据就会流到 `InputStream` 去，`URLSession` 就会不断的读取数据并发送到服务器。

至于 `downlink`，只需要服务器在 response 的时候，将 `Transfer-Encoding` 设为 `chunked` 即可，这样我们就可以通过 `delegate` 方法 `func urlSession(_:dataTask:didReceive:)` 源源不断的读取下行数据。

除此之外，后端还需要对 `uplink` 和 `downlink` 进行配对，进行密钥交换等步骤。成功之后才能够开始发送和接收普通的信令消息。举例比如我们配对的步骤叫做 `name channel`，首先是 `uplink` 发送第一个 `name channel` 的包，将加密方式、密钥和 `Connection ID` 等发送给服务器，然后进入等待状态。然后 `downlink` 发起请求，将 `name channel` 的数据发送给服务器，服务器通过对比 `Connection ID` 配对成功后，通过 `downlink` 返回第一个包，通知客户端已经连接建立完成，后续可以开始发送普通信令。


### But

但往往理想很丰满，现实却很骨感。


#### 问题 1

按照前面的流程建立连接，`uplink` 在发送第一个 chunk 数据的时候，服务端马上就返回了 `Server 500` 的错误。这个错误和 `Transfer-Encoding` 有关，通过抓包能看到请求的 `header` 和 `body` 都很普通。之前调研时使用 `python` 写的测试代码，已经证明了这个方案是没问题的。但到了 iOS 这一端，服务器却发生了错误。

通过连调，发现 `uplink` 走到了普通 `Post` 的流程，而不是应该的 `Stream` 流程。如果将 `Transfer-Encoding` 去掉，却不会 `Server 500`，连接正常结束了。得出结论是当存在 `Tranfer-Encoding` 为 `Chunked` 时，后端走到了 `Post` 流程，然后用 `Post` 流程的方式去获取 `body` 的时候，数据异常导致了 `Server 500`，简单来说也就是 `Transfer-Encoding` 和 `body` 不匹配。

根据抓包的数据对比，iOS 端的包和正常的 python 的包，**唯一** 的区别是 `Chunked` 和 `chunked`，一个大写一个小写。并且 [Wireshark](https://www.wireshark.org) 中也会提示 `[Expert Info (Warning/Undecoded): Unknown transfer coding name in Transfer-Encoding header]`。而我们在设置 `Transfer-Encoding` 的时候，其实也是使用小写的 `chunked`，但经过 `URLSession` 后会变成了大写。HTTP RFC 规定 header field 大小写不敏感，但没有对 value 进行规定。对比用的 python 的框架，则会将 `Transfer-Encoding` 处理成小写。这个问题和迷幻，但我们可以得到结论就是后端所使用的服务，并不支持大写的 `Chunked`。

解决办法就是通过 Hook 的方式，禁止底层将 `header` 中的 `Transfer-Encoding` 进行更改。

> 实际操作中其实还有更多问题，比如 `URLSessionDataTask` 进行处理 `orignalRequest` 的时候，会把 `currentRequest` 的 `Transfer-Encoding` 的值设为 `nil`。


#### 问题 2

在 `downlink` 发送完 `name channel` 之后，一直等待不到返回，直到超时后才返回第一个包，`HTTP/1.1 200 OK`，并马上结束连接。由于 `downlink` 其实就是一个普通的 `POST` 请求，需要的参数也比较少，`name channel` 的数据也很容易确定是否正确，并且返回的第一个包 `header` 和 `body` 除了 `Transfer-Encoding` 为 `Identity` 外并无异常。

首先还是进行抓包，发现其实在请求刚发出去不久，服务器就马上返回了，并且 `header` 和 `body` 所有的数据都很正常，包括 `Transfer-Encoding` 为 `chunked`。问题来了，为什么服务器返回给 `URLSession` 了，`URLSession` 却没有通过 `delegate` 回调给我们呢，它连 `func urlSession(_:dataTask:didReceive, completionHandler:`，明明 `header` 和第一个 `chunk` 的 `body` 都已经返回了。

这个问题比问题 1 还迷，虽然它把 `Transfer-Encoding` 从 `chunked` 改为了 `Identity` 还能接受，因为我们还可以通过查看 `header` 中没有 `Content-Length` 来判断，它其实就是 `chunked` 的方式。

这个问题的解决过程比较曲折，首先我发现不等待 `downlink` 返回 `name channel` 成功，继续通过 `uplink` 发送信令时，`downlink` 就能够返回 `name channel` 成功的 `chunk` 以及后续信令的 `chunk`，`downlink` 不会超时断开。但是我们的整个流程是必须要等 `name channel` 成功之后才能发生消息，中间牵扯着很多 `seq` 和 `ack` 的问题。不能发送普通的信令，那么就尝试多发送一次 `name channel` 的时候，发现也还是不行。

发送普通信令可以，发送 `name channel` 却不行。后来尝试了其他信令，看看是否是偶然问题，最终发现了主要的问题：** `URLSession` 会等待第一个 512 bytes 数据，只有当 buffer 大于 512 bytes 后才会返回 response 和 data**。

通过以 `first 512 bytes` 为关键词搜索时，在 [HTML 5 rules for determining content types](https://dev.w3.org/html5/cts/html5-type-sniffing.html) 发现 H5 对 `first 512 bytes` 有一定的要求。猜想原因和 `Content-Type` 为 `text/html` 有关，后来验证了这个猜想。

反过来一想，也很难说 `URLSession` 的实现是否有问题，但没有任何文档说明确认给人带来不少困扰。对比起来 `Java` 和 `Python` 都没有遇到这个问题。至于为什么服务器返回的是 `text/html`，而不是做客户端常用的 `application/json` 之类的，大概是因为后端用的服务框架是一个标准对 `web` 框架吧。

到这里解决方式就很简单了，** 把该死的 `Content-Type` 改为符合场景的 `application/octet-stream` **。


#### 问题 3

其实是最初遇到的问题，就是 `Proxy` 的影响。首先 `Proxy` 看到的数据，是否可信的问题，其次就是 `Proxy` 对 `header` 带来的影响，比如 `End-to-end headers` 和 `Hop-by-hop headers`。([HTTP headers - HTTP | MDN](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers)）

开发过程中，早期使用 `Charles` 和 `ProxyMan` 来抓 `HTTPS` 的包，没有遇到问题 1 `Server 500` 的问题，某一次重启电脑后，没有打开 `Proxy`，然后才发现会稳定出现 `Server 500`。

还有值得一提的就是代码服务器的问题，在开发过程中也遇到过直连和通过代理服务器后出现不一致的问题，虽然后端说代理服务器不会做任何处理，只是将数据进行透传。


#### 其他的方案

在解决上面的几个问题时，由于文档和相关的讨论都比较少，并且没有看到有人遇到这些问题，在确定问题出在 `URLSession` 及其底层实现上的情况下，在没头绪的时候还尝试过 `libcurl` 的方案。

由于 apple 为了建设 Swift 的生态，提供了一个开源版的 [Foundation](https://github.com/apple/swift-corelibs-foundation)，其中就包含有 [URLSession](https://github.com/apple/swift-corelibs-foundation/tree/master/Sources/FoundationNetworking/URLSession) 的实现。在使用过程中发现它在处理 `body` 的时候，是有问题的： `当 Body 为 Stream 的情况下，根据 libcurl 的 read function 来读取 body 数据，每次读取一个指定大小的 chunk，当读取的时候，发现 inputStream 已经没有数据了，就认为是读取完所有数据了。` 这显然与我们的要求，以及 iOS 上的 `URLSession` 的行为不一致，`input stream` 在读的时候没读到数据，有可能 `output stream` 还在写或者即将写，并不代表整个流已经完成。最简单来说就是为了性能考虑，`Output stream` 并不是用循环的方式进行写数据的，它有一个有限大小的 `buffer`，必须要等到有足够空间的时候才能成功写入数据。所以一般的方式也是 [官方推荐](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/Streams/Streams.html#//apple_ref/doc/uid/10000188-SW1) 的方式，都是使用 `RunLoop` 。

所以我们需要将这部分的实现改一下，改为 `RunLoop` 的方式，当 `read function` 要读取数据时，发现 `input stream` 还没有数据，那么就返回 `retryLater`，告诉 `libcurl` 的 `read socket` 暂停发送数据，先等待。当 `RunLoop` 告诉我们有数据了 `hasBytesAvailable` 的时候，再让 `read socket` 进行 `resume`。可以参考 `File` 类型的 `Body` 的实现。


### Conclusion

如果想用 `Cocoa Foundation` 实现 `Streaming/Chunked`，可能并不会那么顺利。有些问题在其他平台没有暴露出来，也不代表着一定就是 `Cocoa Foundation` 的锅，要善于用可靠的方式从一个更 low level 的角度找到不同平台不同实现直接的差异。当没有文档资料对问题有帮助的时候，「思考 -> 猜测 -> 验证」，可能是最佳的解决问题的方式。