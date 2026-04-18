---
categories: ["技术", "Golang"]
title: Golang中Channel使用的一点随想
date: 2020-05-25 18:25:38
tags: ["前后端"]
---
# Golang中Channel使用的一点随想

## 前言（为什么要写这篇文章）

在Golang中，搞同步/并发控制的方法有很多，有channel(管道)，WaitGroup(等待线程结束)，context(上下文管理)，我一直想深入研究一下它们，因为这次开发我遇到了很多比较棘手的问题，我认为万变不离其宗，所以我看了一下他们的源码，然后简单的写了几个Demo，结合了我自己的开发经验，写成此文，做记录的同时，希望可以帮到其他兄弟，未来我还会出context随想，waitgroup随想，一点一点来吧。

## 什么是channel

首先你要了解两个东西，一个是goroutine，一个是CSP(Communicating Sequential Processes)

goroutine：Go协程，比线程小，内存占用小，Go的主打

CSP:一种模型，并发模型，说白了，就是依赖channel，认为信息的载体更加重要。这里相对来说比较复杂，请大家参考 http://www.usingcsp.com/cspbook.pdf 去学习一个。

那么channel到底是什么呢，其实就是一种在goroutine之间通用的通信方式，这么理解吧，军队每天都看到人守夜，每天都有不同的口号，比如哨兵a，哨兵b进行交接的时候，就要对暗号，我们理解一下，哨兵a,b是两个goroutine，那么口令就是channel，通过channel，我们可以控制哨兵下岗，上岗，巡逻，那么换算到goroutine中，我们就可以控制goroutine启动，停止，这就是channel的牛逼之处。

## channel长什么样？（我们怎么定义channel）
channel的定义方法非常简单，利用chan 关键字就可以

```golang
test := make(chan bool)
```

这里的make，没有定义缓存值，channel可以定义一个缓存值，例如

```golang
make(chan int, 100)
```

这里代表channel 的缓存大小为100，如果不设置缓存值，那么channel没有缓存，在没有通讯的情况下，将被阻塞。

是的，这个就定义好了，现在你就有一个名叫test的bool类型的channel，你可能会问，卧槽，这东西有什么用？行，我马上就举个例子告诉你这玩意儿怎么使。我知道你们大部分都不爱看理论，只爱看解决问题的模型，没问题，我惯你！


## 来个channel的并发例子
你细分析分析，一般你什么时候会用到goroutine间通信？那不就是一个不够使，多开几个，增加咱们program的效率嘛。是，你可以用 go example() 这种类型，开它十几二十个，但是你怎么知道他们结束了呢？估摸着你写sleep啊，就像  

```golang
listip := []string{"10.0.9.11","10.0.9.22","10.0.9.33"}
for _, ip := range(listip) {
    //假设我们执行一个ping ip 的逻辑
    go PingIPWork(ip)
}
time.sleep(time.Second * 5)
```

这样其实没啥毛病，因为你不加那个sleep，人家估摸着就会一闪而过，你什么消息都收不到，我们这里有三个Ip，相当于你 go PingIPWork(ip)这里执行，外部的主程会直接退出去，人家才不管你搞完没呢，你又没和人家说是吧，所以channel就是干这个的，就是告诉主程，里面还有人呢嘿，别关门！

```golang
listip := []string{"10.0.9.11","10.0.9.22","10.0.9.33"}
ch := make(chan struct{})
for _, ip := range listip {
    go func(ip string) {
        //假设执行一个ping ip的逻辑
        go PingIPWork(ip)
        ch <- struct{}{}
    }(ip)
}
for range ips {
    <-ch
}
```

这里去并发去执行PingIPWork，并且主程会等待所有goroutine完成之后才会彻底退出。发现了没，channel和goroutine一般都是放在一起的。


## 在for...i range中使用channel
有时候我们需要阻塞range，或者是控制循环不退出，这时候就可以用到channel

```golang
listip := ["10.0.9.11","10.0.9.22","10.0.9.33"]

//创建channel
forever := make(chan bool)

go func() {
  for _, d := range msgs {
    log.Printf("Received a message: %s", d)
  }
}()

//阻塞，让它不退出
<-forever
```

## 在select中使用channel
类似switch的骚操作，一般来说，你给人家整死循环了，你不得负责给人家搞退出，或者说满足必要条件退出
```golang
// 创建 quit channel
quit := make(chan string)
// 启动生产者 goroutine
c := boring("Joe", quit)
// 从生产者 channel 读取结果
for i := rand.Intn(10); i >= 0; i-- { fmt.Println(<-c) }
// 通过 quit channel 通知生产者停止生产
quit <- "Bye!"
fmt.Printf("Joe says: %q\n", <-quit)

select {
case c <- fmt.Sprintf("%s: %d", msg, i):
    fmt.Println("work....")
case <-quit:
    quit <- "See you!"
    return
}
```

有些时候我们不想等那么久，所以我们需要（timeout）机制

```golang
c1 := make(chan string, 1)
go func() {
    time.Sleep(time.Second * 2)
    c1 <- "result 1"
}()
select {
case res := <-c1:
    fmt.Println(res)
case <-time.After(time.Second * 1):
    fmt.Println("timeout 1")
}
```

## 关闭一个channel
不用的东西要打包放好，否则你开一大堆channel在那里，搞一大堆panic: send on closed channel很吼么？ 随意关闭channel的姿势你也要学到，其实关闭的逻辑也很简单，关键字close  

```golang
c := make(chan int, 10)
c <- 1
close(c)
```

这就关掉了，但是这样关不严谨，你还是能拿到c已经发出去的数据，而且还能不断的读到0值，所以一定要换个方法

```golang
c := make(chan int, 10)
close(c)
i, ok := <-c
fmt.Printf("%d, %t", i, ok) 
```
这样就能判断，当close时，读到的值是零值还是正常值，也就避免了上面出现的那种情况，一直读，一直有。

## 后记
我感觉我学的还是不够深，我希望我的技术能更进一步，所以我还是要不断学习，我可以的，我会做到的。这里面其实都是些皮毛。。我感觉，我希望未来一定要多学多读多看。哈哈，这篇文章写完了我也要去吃饭了，今天我吃仔肺粉 + 锅盔，我去吃饭啦！有问题给我留言或者邮件吧！

写于2020-05-25 19:36分