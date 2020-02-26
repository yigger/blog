---
title: 关于 HTTP
date: 2020-02-19
categories: 网络
---
[top]

### HTTP 的前世今生
http 早在 1990 年问世，它是稍早于 1.0 的版本，这个版本称作 http 0.9 。直到1996 年 5 月， HTTP 正式作为标准进行公布，这就是早期的 HTTP 1.0，随后的一年，1997 年 1月，又接着公布了 HTTP/1.1。在接下来的日子里几乎没有更新，而我们现在听到的 HTTP 2.0 是怎么回事呢？HTTP 2.0 在2013年8月进行首次合作共事性测试。在开放互联网上HTTP 2.0将只用于https:// 网址，而 http:// 网址将继续使用HTTP/1，目的是在开放互联网上增加使用加密技术，以提供强有力的保护去遏制主动攻击。

### http 与 tcp 的关系
每当我们谈及 http ，我们总是会联系到 tcp/ip。实际上，我们通常使用的网络是在 TCP/IP 协议族的基础上运作的，而 HTTP 则是这 TCP/IP 协议族中的其中一员，其中我们耳熟能详的 ICMP、DNS、FTP、TCP、UDP 等等也是在TCP/IP 协议族里面。

TCP/IP 协议族一般分为 4 层，应用层，传输层，网络层，数据链路层。在利用 TCP/IP 协议族进行通信时，会通过分层顺序与对方进行通信，发送端从上往下走，接收端从下往上走。如图所示

![/images/http/http_0.png](通信过程)

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

### HTTP 状态码
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
- 301 

#### 4XX 客户端错误

#### 5XX 服务端错误


### HTTP 报文
http 分为请求报文和响应报文，它们都差不多，主要区别于各自的报文首部
请求报文的组成部分：报文首部，报文主体，首部和主体之间使用回车符和换行符进行分隔。
![/images/http/http_1.png](请求报文)

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

### HTTP 部分首部字段与用处

### HTTP1.0 / 1.1 的区别


### HTTPS

### HTTP 2.0


### 附录
部分图片来源于以下地址
https://blog.csdn.net/ChenglinBen/article/details/90812365

