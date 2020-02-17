---
title: JWT的基本概念
date: 2019-08-13 00:12:03
categories: Web编程
---

由于 http 是无状态的，所以如果要服务端要标示用户的话只能是通过 session 和 cookie 组合的方式来进行交互。

然而，这种方式由于高度依赖于服务端，在单机模式下，固然没有问题，但是，比方说要增加机器的时候，同一个用户如何在两台机器上都识别正确，这也是一个需要考虑的问题。

虽然，业界已经有很多方案可以解决这种问题，比方说，可以把 session_key 记录到数据库，或者记录到 redis 等等。今天，我们来介绍一下另一种校验方案 JWT

### 什么是 JWT？
jwt 是 json web token 的缩写，是目前较流行的跨域认证解决方案。 JWT 一共由 3 个部分组成，分别是 **header**，**payload**，**Signature**

#### header(头部)
header 通常由两部分的 json 数据格式组成，`alg` 和 `typ`， alg 表示签名的算法(algorithm)，而 typ 表示这个 token 的类型，通常来说，都是 JWT。如下所示
```
{
    "alg": "HS256",
    "typ": "JWT"
}
```
接着，通过 Base64URL 算法转换成字符串，转换后，该字符串就是 jwt 三段中的第一段


#### payload(载荷)
payload 也是一个 json 对象，用于存放所需传输的数据，比方说可以存放 user 的相关信息。JWT 还定义了 7 个官方字段：

- iss (issuer)：签发人
- exp (expiration time)：过期时间
- sub (subject)：主题
- aud (audience)：受众
- nbf (Not Before)：生效时间
- iat (Issued At)：签发时间
- jti (JWT ID)：编号
```
{
    "iss": "yigger",
    "iat": 1562767301,
    "exp": 1562770901,
    "aud": "yigger.cn",
    "sub": "yigger@example.com"
}
```
当然，你不一定全部都要用上，具体还是得看你业务需要，但是，**JWT 默认是不加密的，请不要把敏感信息放入到此字段!!!**

同样的，跟第一步一样，通过 Base64URL 把 json 对象转换为字符串生成第二段内容

#### Signature(签名)
签名需要以上两步所生成的内容，然后再根据密钥（该密钥只有你自己知道），和指定的加密算法，就可以生成了第三段的内容了

```
HMACSHA256(
  base64UrlEncode(header) + "." +
  base64UrlEncode(payload),
  secret
)
```

至此，已经生成了三段内容了，然后 `token = 第一段字符串 + "." + 第二段字符串 + "." + 第三段字符串`

以下就是一个 token 的示例格式
```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwiaWF0IjoxNTE2MjM5MDIyfQ.SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c
```

### 实际使用
1. 通过登录，输入账号密码，服务端校验成功后，返回 token
2. 客户端可存储在本地，然后下次请求那些需要认证的接口的时候，把它带过去。需要注意的是，之前用 session 方式的话，是通过 cookie 带到后端的，但是 jwt 这种方式，是通过附加在 header 参数的 authorization 字段，值的前面加Bearer关键字和空格。如：`authorization: Bearer token`
3. 服务端校验客户端传送过来的 token ...

### 总结
jwt 使得服务端变得无状态，无论 token 发送到哪台服务器，只要有相同的逻辑代码，都能校验通过，换句话说，服务端变成无状态了。但是弊端也是存在的：
1. 因为 token 一旦生成了，可用性就会延续到过期时间为止，你无法中断一个正在使用的 token，除非更改后端的加密算法，这也意外着之前签发的所有 token 全部失效。
2. session 机制具有续签机制，但是要实现 token 的续签，可能会相对麻烦些，具体方案也有，已经有人总结出来了，具体可查看参考链接的第二个链接



### 参考
https://jwt.io/introduction/

http://blog.didispace.com/learn-how-to-use-jwt-xjf/
