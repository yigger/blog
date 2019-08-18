---
title: 初识 channel
date: 2019-08-17
tags: golang
---

在了解了 goroutine 之后，我们知道，协程都是独立运行的，如果我们完全不关心协程的执行结果，那大可不需要 channel。然而在实际场景中，你肯定需要从协程中获得返回值，或者把某个协程的结果输入到另一个协程中去，这就涉及到协程之间是怎么进行数据通信

其实 channel 的概念跟 C 的 pipe(管道) 很类似，我们可以先了解一下 C 语言是怎么进行通信的

### C pipe
C pipe 是 Unix 下的一种 IPC（进程间通信） 方式，它是半双工的，也就是说，数据只能从一个地方流动，如果需要双方建立通信，则需要两个管道。

C pipe 的函数定义：
```
头文件：unistd.h
int pipe(filedes[2]);
filedes[2]：输出参数，用于接收pipe返回的两个文件描述符；filedes[0]读管道、filedes[1]写管道
返回值：成功返回0，失败返回-1，并设置errno
```

OK，大致了解了管道是怎么回事，那么尝试下用 C 基于 pipe 实现进程间通信呗
```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <errno.h>
#include <unistd.h>
int main(int argc, char *argv[]){
    if(argc < 2){
        fprintf(stderr, "usage: %s parent_sendmsg child_sendmsg\n", argv[0]);
        exit(EXIT_FAILURE);
    }

    // 定义 pipe 数组，作为参数传入到 pipe 函数
    // 以上就已经创建了一个管道了，分别有读端和写端
    int pipes[2];
    if(pipe(pipes) < 0){
        perror("pipe");
        exit(EXIT_FAILURE);
    }

    pid_t pid = fork();
    if (pid < 0) {
        exit(EXIT_FAILURE);
    } else if(pid > 0) {
        // 先掐掉管道读取的那一端
        close(pipes[0]);

        char buf[BUFSIZ + 1];
        // 获取终端输入的值
        strcpy(buf, argv[1]);
        // 把这个值塞入到管道里面去
        write(pipes[1], buf, strlen(buf));
    } else if (pid == 0) {
        char buf[BUFSIZ + 1];
        // 掐掉管道写入的那一端
        close(pipes[1]);
        // 子进程从管道里面取数据
        int nbuf = read(pipes[0], buf, BUFSIZ);
        buf[nbuf] = 0;
        printf("子进程 (%d) 成功接收到父进程发来的贺电： %s\n", getpid(), buf);
    }

    return 0;
}
```

编译运行后
```
$ gcc pipe.c
$ ./a.out haha
$ 子进程 (68567) 成功接收到父进程发来的贺电： haha
```

大致的流程如下：
![image](http://pvzwttsw1.bkt.clouddn.com/WX20190815-182850@2x.png)

其实也可以不关闭管道，让两端都可以通信，但是一般不推荐使用匿名管道 `pipe` 来实现单一进程下既可读又可写，而是使用命名管道 `mkfifo`。当然了，这里只是做简单的了解，毕竟目前我还是想去了解 go 的 channel 

### 通道阻塞
pipe 是进程内通信的一种方式，而 channel 也是 go 协程间进行通信的一种数据结构。
默认情况下，通过 `c = make(chan int)` 创建的 channel 都是无缓冲的，如果把 channel 类比成队列，这种方式下创建的队列的长度为 1，又因为 channel 的写入和获取都是同步的，
所以每个定义的 channel 都需要读端和写端

如以下代码是不被允许的，因为 dataChannel 一直在等待 channel 写入数据（因为只有"别人"往里面写入数据了，它才能真正执行到）
```golang
func main() {
	dataChannel := make(chan int)
	<- dataChannel
}
```

那是不是改成这样就可以了呢？
```golang
func main() {
	dataChannel := make(chan int)
	dataChannel <- 1
	<- dataChannel
}
```

同样的，这也会报 runtime 的错误，虽然 dataChannel 已经写入了，但是代码一直都还是在 `dataChannel <- 1` 这个地方阻塞着，等待别人从通道中 “读走” 数据

### 解法一
解决方案可以把 `dataChannel <- 1` 扔到协程，或者把 `<- dataChannel` 扔到协程，都行
```golang
dataChannel := make(chan int)
go func(){
	fmt.Println(<- dataChannel)
}()
dataChannel <- 4

或者

dataChannel := make(chan int)
go func(){
    dataChannel <- 4	
}()
fmt.Println(<- dataChannel)
```

因为把阻塞的操作都扔到 go 协程，主协程虽然是阻塞的，但是程序知道某一刻一定会存在有人把我这个值给“读走/写入”，所以可以这么改

但是，如果互换位置的就不行了，比如这样
```golang
dataChannel := make(chan int)
dataChannel <- 4
go func(){
	fmt.Println(<- dataChannel)
}()
```
因为程序在 `dataChannel <- 4` 这一步就已经阻塞了，换句话说，也就是永远都不会执行下面的 `go func`

### 解法二
```golang
dataChannel := make(chan int, 2)
dataChannel <- 4
go func(){
	fmt.Println(<- dataChannel)
}()
```

这段代码跟上一段的唯一区别就在 `dataChannel` 初始化的时候，加上了 2 的参数，代表该队列的长度是 2 个，也就是说可以容纳两个元素，超出，就会继续堵塞，这种就是**缓冲通道**
在有空余位置的情况下，不会阻塞当前主协程

### 使用通道实现超时
go 超时的实现也是相当有趣，因为 go 不像其它动态语言，只需要简简单单一句 timeout(1000) 就可以实现超时，go 的超时也是通过 channel 的 select 来实现的

第一种：
```go
timeout := make(chan bool)
exChan := make(chan int)
go func() {
	time.Sleep(second)
	timeout <- true
}()
go func(){
	time.Sleep(2 * second)
	exChan <- 23
}()
select {
case <- exChan:
	fmt.Println("输出 exChan")
case <- timeout:
	fmt.Println("超时了")
}
```

第二种：
```
exChan := make(chan int)
go func(){
	time.Sleep(2 * second)
	exChan <- 23
}()
select {
case <- exChan:
	fmt.Println("输出 exChan")
case <- time.After(second):
	fmt.Println("超时了")
}
```



### 死锁
[可参考这篇文章](https://www.cnblogs.com/bigdataZJ/p/go-channel-deadlock.html)
