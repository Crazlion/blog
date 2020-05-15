+++
title='Go 中的并发'
tags=["go"]
categories=["go"]
date="2020-05-13T12:00:27+08:00"
toc=true
draft=true
+++


# Go 中的并发 #

学习一下go的并发策略

## 线程分类 ##

一般我们都知道线程会分为两类：  

1. 内核线程  
2. 用户线程

### 用户线程 ###

* 用户线程在用户空间中实现，内核并没有直接对用户线程进程调度，内核的调度对象和传统进程一样，还是进程（用户进程）本身，内核并不能看到用户线程，内核并不知道用户线程的存在。

* 不需要内核支持而在用户程序中实现的线程，其不依赖于操作系统核心，应用进程利用线程库提供创建、同步、调度和管理线程的函数来控制用户线程。

* 内核资源的分配仍然是按照进程（用户进程）进行分配的；各个用户线程只能在进程内进行资源竞争。

* 用户级线程内核的切换由用户态程序自己控制内核切换（通过系统调用来获得内核提供的服务）,不需要内核干涉，少了进出内核态的消耗，但不能很好的利用多核Cpu。目前Linux pthread大体是这么做的。

* 每个用户线程并不具有自身的线程上下文。因此，就线程的同时执行而言，任意给定时刻每个进程只能够有一个线程在运行，而且只有一个处理器内核会被分配给该进程。

### 内核线程 ###

* 内核线程又称为守护进程，内核线程的调度由***内核负责***，一个内核线程处于阻塞状态时不影响其他的内核线程，因为其是调度的基本单位。这与用户线程是不一样的；

* 这些线程可以在全系统内进行资源的竞争；

* 内核空间内为每一个内核支持线程设置了一个线程控制块（TCB），内核根据该控制块，感知线程的存在，并进行控制。在一定程度上类似于进程，只是创建、调度的开销要比进程小。有的统计是1：10。

* 内核线程切换由内核控制，当线程进行切换的时候，由用户态转化为内核态。切换完毕要从内核态返回用户态，即存在用户态和内核态之间的转换，比如多核cpu，还有win线程的实现。

ps(自己的一些思考):其实可以看出来，内核线程由内核来实现创建、销毁，是内核的调度策略  
用户线程的话更像是搭载在内核线程之上的，是用户自己定义调度策略，这样就可以只用一个内核线程完成多个用户线程的任务，go的调度策略就是这样的

## go的线程 ##

操作系统会在物理处理器上调度线程来运行，而Go语言的运行时会在逻辑处理器上调度goroutine来运行。  
每个逻辑处理器都分别绑定到单个操作系统线程。  

在1.5 版本如图所示，可以看到操作系统线程、逻辑处理器和本地运行队列之间的关系。如果创建一个goroutine 并准备运行，这个goroutine 就会被放到调度器的全局运行队列中。
![图6-2](/img/20200513122602.png)  
之后，调度器就将这些队列中的goroutine 分配给一个逻辑处理器，并放到这个逻辑处理器对应的本地运行队列上，Go语言的运行时默认会为每个可用的物理处理器分配一个逻辑处理器。在1.5 版本之前的版本中，默认给
整个应用程序只分配一个逻辑处理器。

在图6-2 中，可以看到操作系统线程、逻辑处理器和本地运行队列之间的关系。如果创建一个goroutine 并准备运行，这个goroutine 就会被放到调度器的全局运行队列中。之后，调度器就将这些队列中的goroutine 分配给一个逻辑处理器，并放到这个逻辑处理器对应的本地运行队列中。本地运行队列中的goroutine 会一直等待直到自己被分配的逻辑处理器执行。

有时，正在运行的goroutine 需要执行一个阻塞的系统调用，如打开一个文件。当这类调用发生时，线程和goroutine 会从逻辑处理器上分离，该线程会继续阻塞，等待系统调用的返回。与此同时，这个逻辑处理器就失去了用来运行的线程。所以，调度器会***创建一个新线程***，并将其绑定到该逻辑处理器上。之后，调度器会从本地运行队列里选择另一个goroutine 来运行。一旦被阻塞的系统调用执行完成并返回，对应的goroutine 会放回到本地运行队列，而***之前的线程会保存好（复用）***，以便之后可以继续使用。

基于调度器的内部算法，一个正运行的goroutine 在工作结束前，可以***被停止并重新调度***。调度器这样做的目的是***防止某个goroutine长时间占用逻辑处理器***。当goroutine 占用时间过长时，调度器会停止当前正运行的goroutine，并给其他可运行的goroutine 运行的机会。可以通过runtime.GOMAXPROCS(1)来验证这个结论

### 验证问题 ###

#### 验证1 ####

```go
package main

import (
    "fmt"
    "runtime"
    "sync"
)

// main 是所有Go 程序的入口
func main() {
    // 分配一个逻辑处理器给调度器使用
    runtime.GOMAXPROCS(1)
    // wg 用来等待程序完成
    // 计数加 2，表示要等待两个goroutine
    var wg sync.WaitGroup
    wg.Add(2)
    fmt.Println("Start Goroutines")
    // 声明一个匿名函数，并创建一个goroutine
    go func() {
        // 在函数退出时调用Done 来通知main 函数工作已经完成
        defer wg.Done()
        // 显示字母表3 次
        for count := 0; count < 3; count++ {
            for char := 'a'; char < 'a'+26; char++ {
                fmt.Printf("%c ", char)
            }
        }
    }()
    // 声明一个匿名函数，并创建一个goroutine
    go func() {
        // 在函数退出时调用Done 来通知main 函数工作已经完成
        defer wg.Done()

        // 显示字母表3 次
        for count := 0; count < 3; count++ {
            for char := 'A'; char < 'A'+26; char++ {
                fmt.Printf("%c ", char)
            }
        }
    }()
    // 等待 goroutine 结束
    fmt.Println("Waiting To Finish")
    wg.Wait()
    fmt.Println("\nTerminating Program")
}
```

结果为：  
Start Goroutines  
Waiting To Finish  
A B C D E F G H I J K L M N O P Q R S T U V W X Y Z A B C D E F G H I J K L M N
O P Q R S T U V W X Y Z A B C D E F G H I J K L M N O P Q R S T U V W X Y Z a b
c d e f g h i j k l m n o p q r s t u v w x y z a b c d e f g h i j k l m n o p
q r s t u v w x y z a b c d e f g h i j k l m n o p q r s t u v w x y z  
Terminating Program  
当只有一个核工作的时候只能执行顺序执行（需要函数的运行时间不超过一定的阀值），明明是小写打印在前但是最终的结果是大写打印在前呢

从逻辑处理器的角度展示了这一场景。在第1 步，调度器开始运行goroutine A，而goroutine B 在运行队列里等待调度。之后，在第2 步，调度器交换了goroutine A 和goroutine B。由于goroutine A 并没有完成工作，因此被放回到运行队列。之后，在第3 步，goroutine B 完成了它的工作并被系统销毁。这也让goroutine A 继续之前的工作。
![图](/img/20200515080446.png)

上面的程序主要验证的问题：goroutine执行的方式是通过队列

#### 验证2 ####

```go
// 这个示例程序展示goroutine 调度器是如何在单个线程上
// 切分时间片的
package main

import (
    "fmt"
    "runtime"
    "sync"
)

// wg 用来等待程序完成
var wg sync.WaitGroup

// main 是所有Go 程序的入口
func main() {
    // 分配一个逻辑处理器给调度器使用
    runtime.GOMAXPROCS(1)
    // 计数加 2，表示要等待两个goroutine
    wg.Add(2)
    // 创建两个goroutine
    fmt.Println("Create Goroutines")
    go printPrime("A")
    go printPrime("B")
    // 等待 goroutine 结束
    fmt.Println("Waiting To Finish")
    wg.Wait()
    fmt.Println("Terminating Program")
}

// printPrime 显示 5000 以内的素数值
func printPrime(prefix string) {
    // 在函数退出时调用Done 来通知main 函数工作已经完成
    defer wg.Done()
next:
    for outer := 2; outer < 10000; outer++ {
        for inner := 2; inner < outer; inner++ {
            if outer%inner == 0 {
                continue next
            }
        }
        fmt.Printf("%s:%d\n", prefix, outer)
    }
    fmt.Println("Completed", prefix)
}

```

部分结果：
A:5147  
A:5153  
B:3769  
B:3779  

B:7841  
B:7853  
B:7867  
A:9791  
A:9803  

goroutine B 先显示素数。一旦goroutine B 打印时间点阀值，调度器就会将正运行的goroutine切换为goroutine A。之后goroutine A 在线程上执行了一段时间，再次切换为goroutine B。这次goroutine B 完成了所有的工作。一旦goroutine B 返回，就会看到线程再次切换到goroutine A 并完成所有的工作。

上面的程序主要验证的问题：调度器会停止一个占用时间过长的goroutine，将资源让给别的goroutine

## 最后总结 ##

go中的协程其实就是用户线程,由go语言的开发者实现  
调用的方式是通过队列，同时会有时间阀值来控制，保证大家都能得到运行，而不会因为一个goroutine占用时间太多，导致其他goroutine饿死的情况
