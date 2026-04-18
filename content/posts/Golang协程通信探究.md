---
title: Golang协程通信探究
date: 2021-12-17 16:40:00
tags: ["前后端"]
---

## 前言

这几天攻关了一下协程通信，也顺利完成了

我觉得每次学习都是一个新的经验

狗公司今年没有年终

我肯定是要走的，没必要继续停留

1. **[Golang协程基础 （已经完成！）](https://yemilice.com/2021/10/27/golang%E5%8D%8F%E7%A8%8B%E5%9F%BA%E7%A1%80%E6%8E%A2%E7%A9%B6/)**
2. **[Golang协程调度 （已经完成！）](https://yemilice.com/2021/11/08/golang%E5%8D%8F%E7%A8%8B%E8%B0%83%E5%BA%A6%E6%8E%A2%E7%A9%B6/)**
3. Golang协程通信 (已经完成)
4. Golang协程控制
5. Golang垃圾回收机制

## CSP思想

要了解协程通信，首先就要了解CSP思想

CSP全称是 Communicating Sequential Processes，意思是通信顺序过程

这个的核心就是并发过程中进行交互，需要通过**通道**传递信息

在CSP的设计思想当中，通过指定的通道发送信息或者接收信息完成通信

CSP思想是Go并发的设计思想，所以Go语言里面定义了**通道**这种重要机制，这也是我们必须面对和掌握的

我们也可以这么理解：**Go的通道是实现Go协程通信的重要媒介**

## Go语言的通道是什么

直接了当的说明吧

我们会在Go代码里面看到很多类似下面这种代码

```golang
var work chan T
chan <- float
<-chan string
```

但凡带有chan的，全都是通道的东西，也就是说，当你们看到chan的时候

就要明白，通道它来了。

## Go预言通道的使用方法

我在这里默认大家都懂Go，都写过Go，知道Go的基本语法。所以基础的语法，我这就不讲了。。大家不懂得回去再学一阵

### 声明通道

声明一个名叫work的chan，通道里存储的数据类型为int

```go
var work chan int
```

声明一个chan 存储int，不带箭头表示可读可写
```go
chan int
```

声明一个chan，只能写入int，不能读
```go
chan <- int
```

声明一个chan，只能读int，不能写
```go
<- chan int
```



### 初始化通道

我们声明完通道之后，不能马上使用，如果你用了，往里面写东西，就类似

```go
package main

import "fmt"

func main() {
	var message chan string

	go func() {
		message <- "work"
	}()

	msg := <-message

	fmt.Println(msg)

}
```

你就会得到一个错误返回

```go
fatal error: all goroutines are asleep - deadlock!

goroutine 1 [chan send (nil chan)]:
main.main()
	.....
exit status 2
```

这他喵就是Go的一个固定逻辑，当你不初始化的时候，你是没发往里写东西的

初始化的操作就是

```go
message := make(chan string)
```

修改下上面的代码

```go
......
message := make(chan string)

go func() {
    message <- "work"
}()

msg := <-message

fmt.Println(msg)
```

现在的返回完全正常了

### 通道写入数据

前面我已经举了个简单的代码例子了

其实再讲明白点，箭头代表你要干嘛

```
通道变量 <- 值
```


把信息写入到channel里面
```go
messages <- "ping"
```

这就是写入，简单吧

### 通道读取数据

读取，一般的用法，是把取出来的值赋值出去

类似

```go
msg := <-message
```

读取分为两种方式

#### 阻塞式接收

这种就类似上面那种形式

```go
msg := <-message
```

这里的message如果为空，是不会退出的，会持续性阻塞在这里

直到获取到message之后，才会完全退出

#### 非阻塞接收

```go
msg, ok := <-message
```
这里的ok是一个bool类型，如果获取到bool为false，则通道完全关闭

### 关闭通道

很简单，直接close通道

```go
close(message)
```

如果你往一个close的通道里面持续写入

```go
func main() {
	message := make(chan string)

	go func() {
		message <- "work"
		close(message)
	}()
	msg, ok := <-message
	message <- "sda"
	fmt.Println(msg, ok)

}
```

这里会报错

```shell
panic: send on closed channel

goroutine 1 [running]:
main.main()
	....
exit status 2
```


你不可以向一个已经关闭的通道写数据，但是！但是！但是！

你可以向一个已经关闭的通道读数据！

```go
func main() {
	message := make(chan int)

	go func() {
		message <- 1
		close(message)
	}()

	msg := <-message

	fmt.Println(msg)

	time.Sleep(time.Second * 2)

	msg2 := <-message

	fmt.Println(msg2)

}
```

这里返回

```go
1 true
0 false
```

看到没，即使已经关闭，我们都可以读到一个0值

**如果我们不去判断通道是否关闭，而是只获取值，那么函数将永远不会结束**

所以这里就需要我们上面的判断了

改写一下代码

```go
msg2, ok := <-message
if !ok {
	fmt.Println("stop")
}
```

这样写不够优雅，我们后面更新一个优雅的写法

### 通道只读/只写

这个前面咱们讲过，可以根据箭头指定写入还是读取

那么建立的时候，咱们也可以通过箭头给它规定死到底是读还是写

#### 只写通道
```go
var c = make(chan <- int)
```

这句的意思就是，这个通道只能写入不能读取整数

#### 只读通道

```go
var c = make( <- chan int)
```

#### 普通通道（可读可写）

```go
var c = make(chan int)
```



### 通道缓冲（限制通道大小）

这里的通道缓冲，可以理解为控制消息数量，你将它理解为队列机制

假设我们的机器处理能力有限，需要限制接收的消息

限制为最大接收3个消息

```go
work := make(chan int， 3)
```

当超过这个消息的时候，则会发生阻塞

所以我们可以利用chan来控制消息数量

或者限制协程执行数量

或者实现一个简单的协程池


## 根据用例深入了解通道

上面我们讲了什么是通道，或者通道是怎么使用的，下面我们找一段代码来逐渐分析下

通道一般在代码里面是怎么使用的

写一段简单的生产者-消费者模型代码

```go
package main

import "fmt"

func work() {
	// 创建一个无缓存的chan 队列，往里面写int数据
	g := make(chan int)
	// 创建一个quit，传输状态
	quit := make(chan bool)
	// 创建一个全局判断状态，传输状态
	quitwork := make(chan bool)

	// 启动一个线程开始监听
	go func() {
		// 走一个无限循环
		for {
			// select循环机制
			select {
			// 如果我们可以持续获取到g
			case v := <-g:
				// 打印一发
				fmt.Println(v)
			// 如果我们检测到quit == true，则退出
			case <-quit:
				fmt.Println("读取协程 end")
				// 将主状态定为true
				quitwork <- true
				return
			}
		}
	}()

	// 启动一个写入协程，开始写数据
	go func() {
		// 往g里面塞数据逻辑的协程
		for i := 0; i < 10; i++ {
			g <- i
		}
		// 给quit赋个值为true
		fmt.Println("写入协程，读取协程退出")
		quit <- true
	}()
	<-quitwork
	fmt.Println("work end")
}

func main() {
	work()
}

```

上面我完成了一个协程写入，一个协程监听的逻辑

所以说，如果，我们要进行chan的开发

我们需要两个协程

一个往里写

一个往外读

往里写我已经讲明白了，现在我讲一下往外读的逻辑

各位发现了没，协程监听逻辑里面有一段代码

```go
// 启动一个线程开始监听
go func() {
	// 走一个无限循环
	for {
		// select循环机制
		select {
		// 如果我们可以持续获取到g
		case v := <-g:
			// 打印一发
			fmt.Println(v)
		// 如果我们检测到quit == true，则退出
		case <-quit:
			fmt.Println("读取协程 end")
			// 将主状态定为true
			quitwork <- true
			return
		}
	}
}()
```

这段代码就是**往外读**的核心机制

我把它改写一些，让它不要看着那么复杂

```go
func ReadChan(g chan int, quit chan bool, quitwork chan bool) {
	for {
		// select循环机制
		select {
		// 如果我们可以持续获取到g
		case v := <-g:
			// 打印一发
			fmt.Println(v)
		// 如果我们检测到quit == true，则退出
		case <-quit:
			fmt.Println("读取协程 end")
			// 将主状态定为true
			quitwork <- true
			return
		}
	}
}
```

好了，核心就是在那个**for-select**部分，下面，我就来讲一下这里吧。

## select

首先，当我们需要和多个通道进行通信，或者需要对通道进行判断的时候

我们一定会用到select，select使用的核心就是一个通道读写阻塞的时候，不会影响其他通道进行work

select的使用方法有些像switch

```go
select {
	case <-ch1:
	 ...
	case <-ch2:
	 ...
	default:
	 ...
}
```

### select的随机性

首先，select，意思是选择，如果我们有多个管道待执行，同时准备好操作

那么select将会随机选择管道执行，举个例子

```go
c := make(chan int, 1)
c <-1
select {
	case <-c:
		fmt.Println("work 1")
	case <-c:
		fmt.Println("work 2")
}
```

这里的返回，有时候输出 work 1，有时候输出work 2，这就是它的随机机制。

### select的阻塞

继续上面的代码，我把它改写一下

```go
c := make(chan int, 1)
// c <-1
select {
	case <-c:
		fmt.Println("work 1")
	case <-c:
		fmt.Println("work 2")
}
```

如果select处没有任何通道能够符合要求，则会持续性阻塞下去，直到传入一个新的值为止

那么如何处理这种问题呢？

加一个default就行，这个意思就是，当都不满足要求，执行这个分支

```go
c := make(chan int, 1)
// c <-1
select {
	case <-c:
		fmt.Println("work 1")
	case <-c:
		fmt.Println("work 2")
	default：
		fmt.Println("work 3")
}
```

### select的控制

我们可以利用select控制管道，也可以自己指定规则进行退出，或者是下一步操作

例如，假设我们要实现一个，如果300s没有消息传入，那么我们就退出这个select

报告超时

超时机制实现

```go
c := make(chan int, 1)
select {
	case <-c:
		fmt.Println("work 1")
	case <-c:
		fmt.Println("work 2")
	case <-time.After(5 * time.Second):
		fmt.Println("timeout.....")
}
```



### for-select循环

这也就是我上面写的那个逻辑

一般情况下，我们不希望它马上退出，而是希望它不断循环操作

```go
for {
	// select循环机制
	select {
	// 如果我们可以持续获取到g
	case v := <-g:
		// 打印一发
		fmt.Println(v)
	// 如果我们检测到quit == true，则退出
	case <-quit:
		fmt.Println("读取协程 end")
		// 将主状态定为true
		quitwork <- true
		return
	}
}
```

这种一般适用于生产者消费者模型，协程池等场景

还有一种**定时发送**的场景

```go
// 每隔2s发送一个信息，写入通道数据
tick := time.Tick(time.Second * 2)
for {
	// select循环机制
	select {
	// 如果我们可以持续获取到g
	case v := <-g:
		// 打印一发
		fmt.Println(v)
	// 往里写东西
	case <-tick:
		fmt.Println("tick")
	// 如果我们检测到quit == true，则退出
	case <-quit:
		fmt.Println("读取协程 end")
		// 将主状态定为true
		quitwork <- true
		return
	}
}
```

## Go通道的原理

又到了喜闻乐见看源码的时间了，我每次看源码看完脑瓜仁都会痛。。。

那就开始呗

chan的源码，放置在/go/src/runtime/chan.go 下

### chan的结构体

```go
type hchan struct {
	qcount   uint           // 通道中的消息个数
	dataqsiz uint           // 通道中的数据大小
	buf      unsafe.Pointer // 存放实际数据的指针
	elemsize uint16         // 通道类型大小
	closed   uint32         // 通道是否关闭
	elemtype *_type 		// 通道类型
	sendx    uint   		// 发送者的序号（ID）
	recvx    uint   		// 接受者的序号（ID）
	recvq    waitq  		// 读取的阻塞队列
	sendq    waitq  		// 写入的阻塞队列
	lock mutex				// 并发的保护锁
}

type waitq struct {
	first *sudog   //代表等待列表中的一个g（开始）
	last  *sudog   //代表等待列表中的一个g（结束）
}
```

*sudog这个函数指的是某个g，这个g用于发送和接收

这部分代码在/go/src/runtime/runtime2.go里

我将chan结构体都做了解释翻译，看注释就行了

其实，将chan理解为一个环形队列

数组，序号recvx和revcq组成了一个环形队列

recvx代表元素在通道里面的位置（读取位置）

sendx代表写入通道时元素所在的位置（写入位置）

recvx到sendex的距离就是通道里面的消息个数总数（读取位置+写入位置 = 消息总数）

举个例子

一个chan一共有4个缓存

[0, 0, 0, 0]

现在写入数据

[1, 2, 3, 4]

那么sendx = 1，recvx = 3

所以count = sendx + recvx

这块非常复杂，我看了源码和书都不太能整明白，只有后面继续学习了。

### 初始化chan的源码

```go
func makechan(t *chantype, size int) *hchan {

	// 检查 
	...

	// 计算需要分配多少大小，根据你传入的size来决定
	mem, overflow := math.MulUintptr(elem.size, uintptr(size))
	if overflow || mem > maxAlloc-hchanSize || size < 0 {
		panic(plainError("makechan: size out of range"))
	}

	var c *hchan
	switch {
	// 当分配的大小为0
	case mem == 0:
		//在内存中进行gc
		// mallocgc是分配内存的函数
		c = (*hchan)(mallocgc(hchanSize, nil, true))
		// Race detector uses this location for synchronization.
		c.buf = c.raceaddr()
	// 如果元素不包含指针
	case elem.ptrdata == 0:
		// hcahan 元素 和 size元素全相加
		c = (*hchan)(mallocgc(hchanSize+mem, nil, true))
		c.buf = add(unsafe.Pointer(c), hchanSize)
	default:
		// 默认情况,包含指针,单独分配内存空间
		c = new(hchan)
		c.buf = mallocgc(mem, elem, true)
	}
	...
}
```

这里最核心的部分就是分配内存的地方

首先,如果我们指定分配长度为10，那么就会计算10长度需要多少mem，分配指定大内存大小，单独分配内存空间才可以进行gc回收


### 写入chan的源码

```go

func chansend(c *hchan, ep unsafe.Pointer, block bool, callerpc uintptr) bool {

	// 如果通道关闭
	if c.closed != 0 {
			unlock(&c.lock)
			panic(plainError("send on closed channel"))
		}

	// 如果有正在等待的读取协程
	if sg := c.recvq.dequeue(); sg != nil {
		// 发送协程
		send(c, sg, ep, func() { unlock(&c.lock) }, 3)
		return true
	}

	// 如果队列的总数小于通道的数量（缓冲区有可用空间）
	if c.qcount < c.dataqsiz {
		qp := chanbuf(c, c.sendx)
		if raceenabled {
			racenotify(c, c.sendx, nil)
		}
		// 分配内存，加入队列
		typedmemmove(c.elemtype, qp, ep)
		// 修改链表的next + 1
		c.sendx++
		// 如果增加之后就缓冲区满了
		if c.sendx == c.dataqsiz {
			// 将sendx置换为0
			c.sendx = 0
		}
		// 将总量 + 1
		c.qcount++
		unlock(&c.lock)
		return true
	}

	// 阻塞通道操作
	gp := getg()
	mysg := acquireSudog()
	mysg.releasetime = 0
	if t0 != 0 {
		mysg.releasetime = -1
	}
	// No stack splits between assigning elem and enqueuing mysg
	// on gp.waiting where copystack can find it.
	mysg.elem = ep
	mysg.waitlink = nil
	mysg.g = gp
	mysg.isSelect = false
	mysg.c = c
	gp.waiting = mysg
	gp.param = nil
	// 发送消息到queue里面
	c.sendq.enqueue(mysg)
}
```

写入协程的时候，有三种不同状态，将进行下面的分析

#### 直接写入协程

如果我们不分配协程的缓存直接写入，会走到这个分支

首先，hchan的recvq维护了一个协程链表

recvq指的是等待的协程链表，每个协程就是一个*sudog，这个前面讲过，就是一个协程的表示方法

首先，recvq会获取协程的元素指针，默认获取链表中的**第一个**协程

会直接将sudog这个玩意儿复制给对应协程，然后进行协程唤醒操作（执行协程）

#### 缓冲区写入协程

我们在meke协程的时候，如果指定了缓冲区，那么就会走到这个分支

首先判断现有队列里面元素的总量：c.qcount

然后获取到缓冲区的总量： c.dataqsiz

如果小于总量，则认为缓冲区还有空间

这时候执行分配内存的操作（向缓冲区写入数据）

然后调整sendx,让它+1(理解链表)

#### 阻塞协程（缓冲区无空余，阻塞操作）

这个操作就是咱们提到的阻塞

如果缓冲区满了,或者协程通道没准备好,就会造成阻塞

这里将sudog进行了赋值修正,然后调用**c.sendq.enqueue(mysg)**,这里是将sudog放入到链表的末尾,然后协程就会进入休眠状态 **gp.waiting**



### 读取chan的源码

```go
func chanrecv(c *hchan, ep unsafe.Pointer, block bool) (selected, received bool) {

	// 如果为空
	if c == nil {
		if !block {
			return
		}
		gopark(nil, nil, waitReasonChanReceiveNilChan, traceEvGoStop, 2)
		throw("unreachable")
	}

	// 如果获取到了等待的协程（可以读取）
	if sg := c.sendq.dequeue(); sg != nil {
		recv(c, sg, ep, func() { unlock(&c.lock) }, 3)
		return true, true
	}

	// 如果缓冲区里面有元素
	if c.qcount > 0 {
		// Receive directly from queue
		qp := chanbuf(c, c.recvx)
		if raceenabled {
			racenotify(c, c.recvx, nil)
		}
		if ep != nil {
			typedmemmove(c.elemtype, ep, qp)
		}
		typedmemclr(c.elemtype, qp)
		c.recvx++
		if c.recvx == c.dataqsiz {
			c.recvx = 0
		}
		c.qcount--
		unlock(&c.lock)
		return true, true
	}

	gp := getg()
	mysg := acquireSudog()
	mysg.releasetime = 0
	if t0 != 0 {
		mysg.releasetime = -1
	}
	mysg.elem = ep
	mysg.waitlink = nil
	gp.waiting = mysg
	mysg.g = gp
	mysg.isSelect = false
	mysg.c = c
	gp.param = nil
	c.recvq.enqueue(mysg)
}
```

其实读取的代码和和写入的有点像，直接分析

### 读取正在等待的协程

还是默认操作，从协程链表rescv里面获取第一个协程

然后复制，最后唤醒阻塞的写入协程

### 读取缓冲区里的协程

缓冲区里面有数据，直接读取，然后写入当前的读取协程中

### 阻塞协程（无法读取，缓冲区为空）

缓冲区没数据，把sudog放入链表末尾，然后休眠协程，等待写入并且重新执行

## Go通道的几个使用场景

因为篇幅，下面讲一下几个重要的应用场景，我这边会给出大致代码，实现的话我会附上我的GITHUB链接，如果需要的话直接去GITHUB里DOWN下来就行了

### 控制协程数

channel可以控制协程的数量，换句话说，我们可以通过控制channel缓存，来控制有几个协程执行

所以我们可以根据这个特性弄个协程池出来

首先，协程池已经有相关的第三方开源实现

例如[ants]("https://github.com/panjf2000/ants")就非常好用

未来我会详细的读一下ants，然后出个blog给大家看

我这里只是举个简单的例子，也就是通过chan去控制同时执行的协程

假设我们现在写个代码，去并发执行一些逻辑


```go

//Read 假装在执行一个逻辑
func Read(i int) {
	fmt.Printf("go func: %d\n", i)
	time.Sleep(time.Second)
}

func main() {
	userCount := math.MaxInt64
	for i := 0; i < userCount; i++ {
		go func(i int) {
			go Read(i)
		}(i)
	}
}
```

这时候，你不要去跑这段代码，因为你一定会卡死。因为你开了太多的协程。。。


然后我们拿chan控制一下这个孙子

说下我的思想

1. 首先make chan 造一个有缓存的通道
2. 传输通道， wg.add 一个协程
3. 通过通道控制并发

改写一下代码，如下


```go
var wg = sync.WaitGroup{}


func Read(ch chan bool, i int) {
	defer wg.Done()

	ch <- true
	fmt.Printf("go func: %d, time: %d\n", i, time.Now().Unix())
	time.Sleep(time.Second)
	<-ch
}

func main() {
	userCount := 10
	ch := make(chan bool, 2)
	for i := 0; i < userCount; i++ {
		wg.Add(1)
		go Read(ch, i)
	}

	wg.Wait()
}

```

这就是大概的协程控制逻辑



                          
### 超时操作

这个前面我记得我讲过

固定的time.After这个逻辑就行

这部分代码套用了

https://eddycjy.gitbook.io/golang/di-1-ke-za-tan/control-goroutine#chang-shi-chan-+-sync

```go
func doWithTimeOut(timeout time.Duration) (int, error) {
    select {
    case ret := <-do():
        return ret, nil
    case <-time.After(timeout):
        return 0, errors.New("timeout")
    }
}

func do() <-chan int {
    outCh := make(chan int)
    go func() {
        go work()
    }()
    return outCh
}
```


## 结尾

其实这篇文章写挺久了。，。。

但是我还是没把它好好写完

我太懒了，不行，我要加倍努力，加油加油！！！！

