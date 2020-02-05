---
title: 并发和并行的个人理解
date: 2019-08-13 01:28:03
tags: golang
---

并发和并行这两个概念基本上在学习多线程编程的时候总会碰到，虽然只有一字之差，但其实是两个不一样的概念，从英文单词的角度来看就能发现他的不一样之处，并发的英文单词是 **Concurrency** 而并行的英文单词是 **Parallelism**

我在理解这两种概念之前，看了很多篇文章，确实很多文章写得很优秀，比如知乎上的一些非常形象的比喻，足以让你一看就明白。但是，当我看完这些概念之后出去撒泡尿回来准备总结一下的时候，我脑海中只有什么“和尚挑水”、“美女”等故事关键词，要我重新组织语言我确实不知道该说些什么

直到我看了一篇写得非常好的文章，以下内容，是我自己结合文章的小小总结，如果想查看原文，请文章底部链接：

在开始之前，先深刻理解这句话

**并发指的是程序的结构，并行指的是运行时的状态**

### 并行（Parallelism）
顾名思义，并行是指程序同时执行，判断程序是否并行，只要看**同一时刻**存在两个执行流即可，真的不需要过度去解读这个词语。`并行 => 同时执行`

### 并发（Concurrency）
并发指的是程序的“结构”，当我们说这个程序是并发的，实际上，这句话应当表述成“这个程序采用了支持并发的设计”。好，既然并发指的是人为设计的结构，那么怎样的程序结构才叫做支持并发的设计？

**正确的并发设计的标准是：使多个操作可以在重叠的时间段内进行**

举个例子：程序需要执行两个任务，获取一段数据，把这段数据先写入文件（注：这个数据非常大，可能需要耗时 10s），然后再写入数据库(5s)

先来看一段不支持并发的设计：
```
write_to_file(data)
write_to_db(data)
// 这段代码一共耗时 10s + 5s = 15s 
```

接下来再看支持并发的设计（使用go协程）：
```
go write_to_file(data)
go write_to_db(data)
// 这段代码一共耗时 10s
```

为什么第二段代码耗时 10s？因为他们是并发执行的，也就是说在 `write_to_file`写入的 10s 这个时间段内，`write_to_db`操作也在进行，所以符合上面所说的**使多个操作可以在重叠的时间段内进行**


所以总结如下：并发并不要求必须并行，比如单核cpu上的多任务系统，并发的要求是任务能切分成独立执行的片段。而并行关注的是同时执行，必须是多（核）cpu，要能并行的程序必须是支持并发的。

参考文章：

[还在疑惑并发和并行？](https://laike9m.com/blog/huan-zai-yi-huo-bing-fa-he-bing-xing,61/)
