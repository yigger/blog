---
title: HTTP 1.0/1.1/2.0
date: 2020-02-19
categories: 网络
---
### HTTP 的前世今生
http 早在 1990 年问世，它是稍早于 1.0 的版本，这个版本称作 http 0.9 。直到1996 年 5 月， HTTP 正式作为标准进行公布，这就是早期的 HTTP 1.0，随后的一年，1997 年 1月，又接着公布了 HTTP/1.1。在接下来的日子里几乎没有更新，而我们现在听到的 HTTP 2.0 是怎么回事呢？HTTP 2.0 在2013年8月进行首次合作共事性测试。在开放互联网上HTTP 2.0将只用于https:// 网址，而 http:// 网址将继续使用HTTP/1，目的是在开放互联网上增加使用加密技术，以提供强有力的保护去遏制主动攻击。

### http 与 tcp 的关系
每当我们谈及 http ，我们总是会联系到 tcp/ip。实际上，我们通常使用的网络是在 TCP/IP 协议族的基础上运作的，而 HTTP 则是这 TCP/IP 协议族中的其中一员，其中我们耳熟能详的 ICMP、DNS、FTP、TCP、UDP 等等也是在TCP/IP 协议族里面。

TCP/IP 协议族一般分为 4 层，应用层，传输层，网络层，数据链路层。在利用 TCP/IP 协议族进行通信时，会通过分层顺序与对方进行通信，发送端从上往下走，接收端从下往上走。如图所示

![通信过程](/images/http/http_0.png)

### HTTP 方法
- GET：获取资源
- POST：传输实体的主体
- PUT：传输文件
- DELETE：删除文件
- HEAD：获得报文首部
- OPTIONS：询问支持的方法
- TRACE：追踪路径
主要从事 Web 方面的开发，RESTful 肯定绕不开，而使用RESTFul，GET、POST、PUT、DELETE 一定是十分常用的，就不多说了，剩下的其实不是很常用。

HEAD方法，在服务器响应时不会返回消息体。一个HEAD请求的响应中，HTTP头中包含的元信息应该和一个GET请求的响应消息相同。这种方法可以用来获取请求中隐含的元信息，而不用传输实体本身。也经常用来测试超链接的有效性、可用性和最近的修改。

值得一提的是，在前端跨域的解决过程中，当采用 CORS 方法来解决跨域问题的话，非简单请求的CORS请求，会在正式通信之前，增加一次HTTP查询请求，称为"预检"请求（preflight），"预检"请求用的请求方法是OPTIONS，表示这个请求是用来询问的。头信息里面，关键字段是Origin，表示请求来自哪个源。

### HTTP 响应状态码
#### 1XX
- 100 Continue
客户端继续其请求

- 101 Switching Protocols
切换协议。服务器根据客户端的请求切换协议，比如切换为 websocket 的连接
#### 2XX 成功
- 200 OK
请求成功

- 204 No Content
无内容返回

- 206 Partial Content
部分内容。服务器成功处理了部分 GET 请求

#### 3XX 重定向
- 301 Move Permannently
永久性重定向。该状态码表示请求的资源已被分配了新的URI，以后应使用资源现在所指的URI。
（比如：客户端接收到请求后，把书签的引用进行变更，换为新的地址）

- 302 Found
表示临时重定向 Moved Temporarily。由于这样的重定向是临时的，客户端应继续向原有地址发送以后的请求，只有在 Cache-Control 或 Expires 中进行了指定的情况下，这个响应才是可缓存的。

- 303 See Other
作用和 302 基本一致，表示新的资源在其它URI，**需要用 GET 方法请求它**。 302 也能实现，但是当你希望客户端明确的使用 GET 请求进行访问另一个 URI 时应当使用 302 比较符合规范。

- 304 Not Modify
当客户端的请求头含有 IF-Modified-Since、If-None-Match、If-Range、If-Unmodified-Since 任一首部时，服务端可以返回 304，表示客户端当前请求的资源没有变化，可以使用客户端本地的缓存，所以此时服务端不会返回任何主体数据。

- 307 Temporary Redirect
临时重定向，该状态码和 302 有着同样的含义，只是因为在日常使用中，302 标准是禁止 Post 请求转换为 Get 请求，但是实际使用时大家都不遵守这个标准。所以，就有了 307，它硬性的规定 POST请求不能变为 Get 请求，但是每种浏览器都有可能出现不同的情况

简单的总结
- 301：永久重定向，客户端以后不应该再次请求此地址
- 302：临时重定向，客户端可以（也应该）再次请求此地址
P.S. 由于 302 语义不明确，所以不推荐使用 302 重定向，应该使用 303 或者 307 代替
- 303：临时重定向，并使用 GET 方法请求新的 URL
- 307：临时重定向，并使用原方法请求新的 URL


#### 4XX 客户端错误
- 400 Bad Request
请求格式错误，比如说后端校验请求的时候发现请求参数不对的时候可以返回此状态码

- 401 Unauthorized
未授权。常见的如未登录的用户访问资源

- 403 Forbidden
无权限访问此资源

- 404 Not Found
资源找不到

- 409 Conflict
表示请求的资源与资源的当前状态发生冲突

- 410 Gone
表示服务器上的某个资源被永久性的删除。

#### 5XX 服务端错误
- 500
- 503
服务端错误/服务端超载

### HTTP 报文
http 分为请求报文和响应报文，它们都差不多，主要区别于各自的报文首部
请求报文的组成部分：报文首部，报文主体，首部和主体之间使用回车符和换行符进行分隔。
![请求报文](/images/http/http_1.png)

例子：
- 请求报文
报文首部分为：请求行、请求首部字段
```
// 这是请求行
GET /index.html HTTP/1.1
// 以下是请求首部字段
Host: www.baidu.com
Connection: close
User-agent: Mozilla/5.0
```
GET 是没有请求主体的，POST 才有，平时我们提交的表单数据就是通过请求主体进行传输的

- 响应报文
报文首部分为：状态行、响应首部字段
```
HTTP/1.1 200 OK
Content-Length: 662
Content-Type: text/plain; charset=UTF-8
Date: Wed, 19 Feb 2020 14:49:41 GMT
```
响应主体就是服务端传输回来的文本数据

### HTTP1.0 vs 1.1 vs 2.0

|            | 1.0                                                 | 1.1      | 2.0                                                          |
| ---------- | --------------------------------------------------- | -------- | ------------------------------------------------------------ |
| 长连接     | 需要使用`keep-alive` 参数来告知服务端建立一个长连接 | 默认支持 | 默认支持                                                     |
| HOST域     | X                                                   | ✔        | ✔                                                            |
| 多路复用   | X                                                   | -        | ✔                                                            |
| 数据压缩   | X                                                   | X        | 使用`HAPCK`算法对header数据进行压缩，使数据体积变小，传输更快 |
| 服务器推送 | X                                                   | X        | ✔                                                            |

#### 长连接

http1.1默认保持长连接，数据传输完成保持tcp连接不断开,继续用这个通道传输数据


#### 管道化

基于长连接的基础，我们先看没有管道化请求响应：

tcp没有断开，用的同一个通道

```
请求1 > 响应1 --> 请求2 > 响应2 --> 请求3 > 响应3
```

管道化的请求响应：

```
请求1 --> 请求2 --> 请求3 > 响应1 --> 响应2 --> 响应3
```

即使服务器先准备好响应2,也是按照请求顺序先返回响应1

虽然管道化，可以一次发送多个请求，但是响应仍是顺序返回，仍然无法解决队头阻塞的问题



题外话：为什么管线化必须按照顺序进行返回呢？
由于 HTTP/1.1 是个文本协议，同时返回的内容也并不能区分对应于哪个发送的请求，所以顺序必须维持一致。比如你向服务器发送了两个请求 GET /query?q=A 和 GET /query?q=B，服务器返回了两个结果，浏览器是没有办法根据响应结果来判断响应对应于哪一个请求的。




#### 缓存处理

当浏览器请求资源时，先看是否有缓存的资源，如果有缓存，直接取，不会再发请求，如果没有缓存，则发送请求

通过设置字段cache-control来控制



### http2.0 特性

#### 二进制分帧
HTTP/1.1的头信息是文本（ASCII编码），数据体可以是文本，也可以是二进制；HTTP/2 头信息和数据体都是二进制，统称为“帧”：头信息帧和数据帧

#### 多路复用(双工通信)
http/2之前，每次请求都需要客户端主动发送请求信息，才能得到响应信息，虽然 http1.1 有管道化的特性，可以同时发送多个请求给服务端，但是由于阻塞性，所以即使是多个请求同时发送，但响应也是按顺序进行返回。

http/2 引入了多路复用，此时可以解决这种囧局，在一次连接中，客户端和服务端都可同时进行发送和接收操作，而不需要按照顺序一一对应，这样避免了队头阻塞的问题。

题外话：为什么 HTTP1.1 不能实现多路复用？
HTTP/1.1 不是二进制传输，而是通过文本进行传输。由于没有流的概念，在使用多路复用传递数据时，接收端在接收到响应后，并不能区分多个响应分别对应的请求，所以无法将多个响应的结果重新进行组装，也就实现不了多路复用。

#### header压缩
HTTP 协议不带有状态，每次请求都必须附上所有信息。所以，请求的很多字段都是重复的，，一模一样的内容，每次请求都必须附带，这会浪费很多带宽，也影响速度。HTTP/2 对这一点做了优化，引入了头信息压缩机制（header compression）。一方面，头信息压缩后再发送（SPDY 使用的是通用的DEFLATE 算法，而 HTTP/2 则使用了专门为首部压缩而设计的 HPACK 算法）。另一方面，客户端和服务器同时维护一张头信息表，所有字段都会存入这个表，生成一个索引号，以后就不发送同样字段了，只发送索引号，这样就提高速度了。

#### 服务器推送
HTTP/2 允许服务器未经请求，主动向客户端发送资源，这叫做服务器推送（server push）。常见场景是客户端请求一个网页，这个网页里面包含很多静态资源。正常情况下，客户端必须收到网页后，解析HTML源码，发现有静态资源，再发出静态资源请求。其实，服务器可以预期到客户端请求网页后，很可能会再请求静态资源，所以就主动把这些静态资源随着网页一起发给客户端了。


### HTTP/2 缺点
HTTP 是基于 TCP 实现的，但是 TCP 有个缺陷，**丢包重传**。HTTP/2出现丢包时，整个 TCP 都要开始等待重传，那么就会阻塞该TCP连接中的所有请求。


### 参考地址
https://blog.csdn.net/ChenglinBen/article/details/90812365
https://zhuanlan.zhihu.com/p/61423830
https://mp.weixin.qq.com/s/sakIv-NidqkO1tviBHxtWQ
