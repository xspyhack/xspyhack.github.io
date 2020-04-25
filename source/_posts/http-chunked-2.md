---
title: HTTP Streaming/Chunked 2
tags:
    - swift
    - chunked
    - streaming
    - http
categories: networking
---
## TL;DR

一个非常悲伤的消息是，`Azure` 和 `Cocoa` 的配合还是出了问题，原因出在 `chunk` 的实现上。这个锅应该由 `Cocoa` 来背，`HTTP/1.1` 给出的 `chunk` 的格式中，每一个 `chunk` 的结尾应该是 `CRLF`，而 `Cocoa` 的实现（可能是为了实现上的方便）把这个 `CRLF` 放在了下一个 `chunk` 的开头。对于连续的 `chunk` 来说看起好像没有什么区别，但是在我们的场景中，使用第一个 `chunk` 来建立连接，建立完成之后才会发信令的 `chunk`，也就是第一个 `chunk` 和后续的 `chunk` 不会连续。然而 `Azure` 在收到第一个 `chunk` 后，发现结尾还没有收到 `CRLF`（虽然此时已经收到了正确长度的数据），然后进入继续等待状态，不会把数据包转发给后台数据服务器，这样后台数据服务器就没法和 `downlink` 匹配并且建立连接。

## Chunked Transfer Encoding

首先根据 `HTTP/1.1` 的 [RFC](https://www.w3.org/Protocols/rfc2616/rfc2616-sec3.html) 对于 `Chunked Transfer Encoding` 规定。

```yaml
Chunked-Body   = *chunk
                last-chunk
                trailer
                CRLF
chunk          = chunk-size [ chunk-extension ] CRLF
                chunk-data CRLF
chunk-size     = 1*HEX
last-chunk     = 1*("0") [ chunk-extension ] CRLF
chunk-extension= *( ";" chunk-ext-name [ "=" chunk-ext-val ] )
chunk-ext-name = token
chunk-ext-val  = token | quoted-string
chunk-data     = chunk-size(OCTET)
trailer        = *(entity-header CRLF)
```

重点部分为 `chunk` 的格式，可以看到每一个 `chunk` 应该由一个 16 进制的 `chunk-size` 开始，然后接着一个 `CRLF`，然后是 `chunk-data`，末尾是一个 `CRLF`。

```yaml
chunk          = chunk-size [ chunk-extension ] CRLF
                chunk-data CRLF
```

再看 `Cocoa` 的实现 [CFHTTPFilter.c](https://opensource.apple.com/source/CFNetwork/CFNetwork-128.2/HTTP/CFHTTPFilter.c.auto.html)，虽然它在源码里面也贴了上边 `RFC` 中的结构，可是它的实现中并不那么回事。看第一行注释就暴露了它的每个 `chunk header` 中除了 `first chunk` 之外，其他的都会出现两个 `CRLF`，其中一个叫做 `leading CRLF`，也就是开头说的，它是在下一个 `chunk` 的头部，插入一个 `CRLF`。

```objc
// CFIndex <= uint64 so no more than 16 characters to encode + 2 for CRLF + 2 for leading CRLF
#define MAX_CHUNK_HEADER_SIZE (20)
static void sendChunkHeader(CFWriteStreamRef stream, CFIndex chunkLength, Boolean firstChunk, CFStreamError *error) {
    // hex representation of chunkLength, followed by CRLF
    UInt8 writeBuffer[MAX_CHUNK_HEADER_SIZE];
    UInt8 *writeBase;
    CFIndex bytesWritten;
    error->error = 0;
    writeBuffer[MAX_CHUNK_HEADER_SIZE - 1] = '\n';
    writeBuffer[MAX_CHUNK_HEADER_SIZE - 2] = '\r';
    writeBase = &(writeBuffer[MAX_CHUNK_HEADER_SIZE-3]); 
    while (chunkLength > 0) {
        int nextDigit = chunkLength & 0xF;
        *writeBase = nextDigit < 10 ? '0' + nextDigit : 'A' + nextDigit - 10;
        writeBase --;
        chunkLength = chunkLength >> 4;
    }
    if (firstChunk) {
        writeBase ++;
    } else {
        *writeBase = '\n';
        writeBase --;
        *writeBase = '\r';
    }
    while (writeBase < writeBuffer + MAX_CHUNK_HEADER_SIZE) {
        bytesWritten = CFWriteStreamWrite(stream, writeBase, writeBuffer + MAX_CHUNK_HEADER_SIZE - writeBase);
        if (bytesWritten < 0) {
            *error = CFWriteStreamGetError(stream);
            break;
        } else if (bytesWritten == 0) {
            // Premature EOF; can we come up with a better error code?
            error->domain = kCFStreamErrorDomainHTTP;
            error->error = kCFStreamErrorHTTPParseFailure;
            break;
        } else {
            writeBase += bytesWritten;
        }
    }
}
```

合理的样子是

```yaml
7\r\n
Chunk_1\r\n
7\r\n
Chunk_2\r\n
```

而 `Cocoa` 是这样子的

```yaml
7\r\n
Chunk_1
\r\n
7\r\n
Chunk_2
```

## 解决方案

既然 `Cocoa` 的实现不符合 `Azure` 的要求，那么有没有办法来 **绕过** 一下呢？既能让 `Azure` 转发第一个 `chunk`，也能让后端数据服务器正确识别。

### 自己添加一个 CRLF

既然 `Azure` 会等待 `CRLF`，那么在第一个 `chunk` 结尾多加一个 `CRLF` 行不行？肯定不行，因为 `CRLF` 是不算在 `chunk-size` 里面的。并且会导致下一个 `chunk` 到来的时候，出现连续的 `CRLFCRLF`，虽然按要求最后一个 `chunk` 是 `0CRLFCRLF`，但有些服务器会兼容只有 `CRLFCRLF` 的情况。

### 多发一个 chunk

不能多加一个 `CRLF`，那么我们在建立 `name_channel` 的时候，多发一个 `chunk`，那么 `Azure` 收到的数据将会是这样的。

```yaml
C\r\n
name_channel

\r\n
2\r\n
OK
```

上面的格式是为了方便区分两个 `chunk`，其实真正的数据是没有换行的，因为已经把 `\r\n` 写出来了。换一个写法。

```yaml
C\r\nname_channel\r\n2\r\nOK
```

那么 `Azure` 收到这样的数据包之后，就会认为第一个 `chunk` （`C\r\nname_channel\r\n`）已经完整收到了，但第二个 `chunk` 还需要等待最后的 `CRLF`。这时 `Azure` 就会将 `name_channel` 这个 `chunk` 转发给后台服务器，然后就能正常的建立起连接了。当然后天服务器需要处理多发的那个包。

后续信令的 `chunk` 还需不需要每次都多发一个 `chunk`？在我们的场景中不需要也没有什么影响，因为后续的信令，是源源不断的发送出去的，所以不会出现等待下一个包的情况。即使是用户没有操作了，那么也会有心跳包，和 `ack` 的包，唯一影响的是后台服务器收到最后一个 `ack` 包的时间间隔会长一点。

### libcurl

这个方案很简单，因为 `libcurl` 的实现，是跟我们预想中的那样的，每个 `chunk` 结尾都是一个 `CRLF`。

## Conclusion

A 和 B 在一起没问题，B 和 C 在一起没问题，C 和 A 在一起也没问题，但当 A 和 C 在一起的时候，出现了问题，并不能说明这是 A 的问题或者说是 C 的问题。

在测试环境中，上面多发一个 `chunk` 的方式是没有任何问题的。但是当部署到线上环境时，问题依然存在，后台服务器依然收不到请求包。除了一个是 `http` 一个是 `https` 之外，各种配置完全一样，`Azure CDN` 没法调试也没有日志可以打，这个问题暂时没有办法解决。

绝望之余，我看到了一种叫做 [HTTP request smuggling - Wikipedia](https://en.wikipedia.org/wiki/HTTP_request_smuggling) 的攻击手段，能给绕过 `Azure` 不能转发的问题。