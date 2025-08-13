# HTTP

> **HTTP** 是一种用作获取诸如 HTML 文档这类资源的[协议](https://developer.mozilla.org/zh-CN/docs/Glossary/Protocol)。它是 Web 上进行任何数据交换的基础，同时，也是一种客户端—服务器（client-server）协议，也就是说，请求是由接受方——通常是 Web 浏览器——发起的。完整网页文档通常由文本、布局描述、图片、视频、脚本等资源构成。

![HTTP 作为应用层协议，处于 TCP（传输层）和 IP（网络层）之上，表示层之下。](https://mdn.github.io/shared-assets./images/diagrams/http/overview/http-layers.svg)



## HTTP 的基本性质

- **简约**：HTTP 报文能够被人读懂并理解，向开发者提供了更简单的测试方式，也对初学者降低了门槛。
- **可扩展**：只要服务器客户端之间对新标头的语义经过简单协商，新功能就可以被加入进来。
- **无状态**：在同一个连接中，两个执行成功的请求之间是没有关系的。可以借助 HTTP Cookie 就可使用有状态的会话。利用标头的扩展性，HTTP Cookie 被加进了协议工作流程，每个请求之间就能够创建会话，让每个请求都能共享相同的上下文信息或相同的状态。
- **连接**：HTTP 因此而依靠于 **TCP **的标准，即面向连接的。
  - 在客户端与服务器能够传递请求、响应之前，这两者间**必须建立 TCP 连接**，这个过程需要多次往返交互。HTTP/1.0 默认为每一对 HTTP 请求/响应都打开一个单独的 TCP 连接。当需要接连发起多个请求时，工作效率相比于它们之间共享同一个 TCP 连接要低。
  - 为了减轻这个缺陷，HTTP/1.1 引入了*流水线*（已被证明难以实现）和*持久化连接*：可以**通过 [`Connection`](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Reference/Headers/Connection) 标头来部分控制底层的 TCP 连接**。HTTP/2 则更进一步，通过在**一个连接中复合多个消息，让这个连接始终活跃并更加高效**。
  - 为了设计一种更匹配 HTTP 的传输协议，各种实验正在进行中。例如，Google 正在测试一种基于 UDP 构建，更可靠、高效的传输协议——[QUIC](https://zh.wikipedia.org/wiki/QUIC)

## HTTP流

进行信息交互时，过程表现为下面几步：

1. **打开一个 TCP 连接**：TCP 连接被用来发送一条或多条请求，以及接受响应消息。客户端可能打开一条新的连接，或重用一个已经存在的连接，或者也可能开几个新的与服务器的 TCP 连接。

2. **发送一个 HTTP 报文**：HTTP 报文（在 HTTP/2 之前）是人类可读的。在 HTTP/2 中，这些简单的消息被封装在了帧中，这使得报文不能被直接读取，但是原理仍是相同的。例如：

   ```http
   GET / HTTP/1.1
   Host: developer.mozilla.org
   Accept-Language: zh
   ```

3. **读取服务端返回的报文信息**：

   ```http
   HTTP/1.1 200 OK
   Date: Sat, 09 Oct 2010 14:28:02 GMT
   Server: Apache
   Last-Modified: Tue, 01 Dec 2009 20:18:22 GMT
   ETag: "51142bc1-7449-479b075b2891b"
   Accept-Ranges: bytes
   Content-Length: 29769
   Content-Type: text/html
   
   <!DOCTYPE html>…（此处是所请求网页的 29769 字节）
   ```

4. **关闭连接或者为后续请求重用连接**。

## HTTP报头

HTTP/1.1 以及更早的 HTTP 协议报文都是**语义可读的**。在 HTTP/2 中，这些报文被嵌入到了一个新的**二进制结构**，帧。帧允许实现很多优化，比如**报文标头的压缩以及多路复用**。即使只有原始 HTTP 报文的一部分以 HTTP/2 发送出来，每条报文的语义依旧不变，客户端会重组原始 HTTP/1.1 请求。因此用 HTTP/1.1 格式来理解 HTTP/2 报文仍旧有效。

### 请求

HTTP 请求的一个例子：

![带标头的 HTTP GET 请求概览](https://mdn.github.io/shared-assets./images/diagrams/http/overview/http-request.svg)

请求由以下元素组成：

- HTTP 方法，通常是由一个动词，像 [`GET`](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Reference/Methods/GET)、[`POST`](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Reference/Methods/POST) 等，或者一个名词，像 [`OPTIONS`](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Reference/Methods/OPTIONS)、[`HEAD`](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Reference/Methods/HEAD) 等，来定义客户端执行的动作。典型场景有：客户端意图获取某个资源（使用 `GET`）；发送 [HTML 表单](https://developer.mozilla.org/zh-CN/docs/Learn_web_development/Extensions/Forms)的参数值（使用 `POST`）；以及其他情况下需要的那些其他操作。
- 要获取的那个资源的路径——去除了当前上下文中显而易见的信息之后的 URL，比如说，它不包括[协议](https://developer.mozilla.org/zh-CN/docs/Glossary/Protocol)（`http://`）、[域名](https://developer.mozilla.org/zh-CN/docs/Glossary/Domain)（这里是 `developer.mozilla.org`），或是 TCP 的[端口](https://developer.mozilla.org/zh-CN/docs/Glossary/Port)（这里是 `80`）。
- HTTP 协议版本号。
- 为服务端表达其他信息的可选[标头](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Reference/Headers)（header）。
- 请求体（body），类似于响应中的请求体，一些像 `POST` 这样的方法，请求体内包含需要了发送的资源。
