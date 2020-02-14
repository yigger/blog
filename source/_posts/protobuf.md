---
title: protocol buffer
date: 2020-02-11 21:50:39
tags: protobuf
---

### protocol buffers 简介
protocol buffers 是由 google 开发的一款**数据交换格式**，可用于进行通信协议，数据存储。说人话就是类似于 XML、JSON 的一种用于序列化数据的东西，那么为什么不直接用 XML、JSON，而要重新开发 protocol buffers 呢？那还用说，当然是因为它又小又快。

**简单概括：占容量小、速度快、平台无关、语言无关**


### protobuf 的简单使用
protobuf 的使用方法很简单，它比较类似于定义一个结构体，但是只有属性，没有方法。

另外，protobuf 目前有两个版本，一个是 proto2，另一个是 proto3，虽然 proto3 看上去比 proto2 新，但是在一些处理上其实还不如 proto2，比如说默认值和未定义的字段的处理就不如 proto2。但是 proto3 确实也修复了不少的问题和新增了 feature，所以一般情况下都会选用 proto3。

附Proto3 区别于 Proto2 的使用情况
- 在第一行非空非注释行，必须写：syntax = "proto3";
- 字段规则移除 「required」，并把 「optional」改为 「singular」
- 「repeated」字段默认使用 paced 编码
- 移除 default 选项
- 枚举类型的第一个字段必须要为 0
- 移除对扩展的支持，新增 Any 类型， Any 类型是用来替代 proto2 中的扩展的
- 增加了 JSON 映射特性

值得注意的是，如果要使用 proto3，那么在定义的时候第一行必须填写 `syntax = "proto3"`，**否则，将按照 proto2 的语法进行处理。**

```
// filename: login.proto
syntax = "proto3";
package pb;
message LoginReq {
  string username = 1;
  string password = 2;
}

message LoginResp {
	string code = 1;
	string msg = 2;
}
```

对于 JSON 数据，我们可以通过语言层面去直接解析，但是 protobuf 不行，在使用之前，我们需要把文件使用工具编译一下


编译器可以在 https://github.com/protocolbuffers/protobuf 官方的 release 找到并进行安装，安装之后，在当前文件目录下执行 `protoc --go_out=. login.proto` 后，会生成 login.pb.go 文件，该文件生成以后，我们就可以通过引用来直接使用了。

```
func main(){
  req := &pb.LoginReq{
    username: "test",
    password: "123"
  }
  
  resp = &pb.LoginResp{
    ...
  }
}
```

### 使用场景
作为 RPC 的数据交换相对来说用得比较多，grpc 默认也是与 protocol buffer 搭配一起使用

### 编码原理
ProtocolBuffers 使用 Varint 进行编码。

Varint 是一种紧凑的表示数字的方法。它用一个或多个字节来表示一个数字，值越小的数字使用越少的字节数。这能减少用来表示数字的字节数。

Varint 中的每个字节（最后一个字节除外）都设置了最高有效位（msb），这一位表示还会有更多字节出现。每个字节的低 7 位用于以 7 位组的形式存储数字的二进制补码表示，最低有效组首位。

### 优缺点
#### 优点
首先我们来了解一下 XML 的封解包过程。XML 需要从文件中读取出字符串，再转换为 XML 文档对象结构模型。之后，再从 XML 文档对象结构模型中读取指定节点的字符串，最后再将这个字符串转换成指定类型的变量。这个过程非常复杂，其中将 XML 文件转换为文档对象结构模型的过程通常需要完成词法文法分析等大量消耗 CPU 的复杂计算。

反观 Protobuf，它只需要简单地将一个二进制序列，按照指定的格式读取到 C++ 对应的结构类型中就可以了。

所以结论当然是：容量小，速度快，跨平台

#### 缺点
通用性来说的话，如果面向开放的 api 的话，还是 JSON 比较通用，为什么呢？一方面是因为大家都熟悉这套玩法了，另一方面就是省事，不用每次都编译 proto 文件

### 参考链接
https://developers.google.com/protocol-buffers/docs/proto3
https://halfrost.com/protobuf_encode/#proto3message
https://www.ibm.com/developerworks/cn/linux/l-cn-gpb/index.html
https://www.cnblogs.com/makor/p/protobuf-and-grpc.html