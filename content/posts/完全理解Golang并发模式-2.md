---
title: 完全理解Golang并发模式(2)
date: 2020-11-10 10:46:02
tags: ["前后端"]
---

## 前言

这几天的攻关都是在Golang并发上面，也算是有了一些收获，把一些基础的并发场景和并发模式都过了一遍，觉得自己又提升了一层楼，其实我发觉，在我的开发生涯中，总是纠结实现，却不是去看最底层的东西，任何东西，都是由简到难，或者就是从底层到核心，无论是Python还是Golang，或者是我正在狂刷算法的Java，我都是去纠结实现，但是不会钻研核心，所以我写代码都是面向搜素引擎编程。所以，也是时候该改变一些东西。

```
决意想要改变

心情就像离弦的箭

收不回的思绪每天在反复缠绵
```

## 并发的超时处理

如果我们在开发一个Web框架，势必在运行一个goroutine的时候，会出现运行时间过长的问题，一般这种情况，如果不进行超时控制，到最后的结果就是阻塞在那里，迟迟不返回，这样肯定是不行的，所以我们要对并发进行一个超时限制，说人话就是，**加一个timeout之后自动退出或者报错**。

首先这里有个新东西，或者是新包，叫做"Context"，这个我的博客里面的久文章也写过了，在这不多废话。你不会还想让我介绍一下它吧！

### Context的几个调用方法
寻思了一下，如果不介绍，很可能你们不知道我在说什么，还是讲几个基础用法吧，其实这个还是比较简单的，Golang的语法比起Rust简单多了。

Context的接口如下

```golang
type Context interface {

    Deadline() (deadline time.Time, ok bool)

    Done() <-chan struct{}

    Err() error

    Value(key interface{}) interface{}
}
```

其中有四个方法

1. Deadline返回绑定当前context的任务被取消的截止时间；如果没有设定期限，将返回ok == false。
2. Done 当绑定当前context的任务被取消时，将返回一个关闭的channel；如果当前context不会被取消，将返回nil。
3. Err 如果Done返回的channel没有关闭，将返回nil;如果Done返回的channel已经关闭，将返回非空的值表示任务结束的原因。如果是context被取消，Err将返回Canceled；如果是context超时，Err将返回DeadlineExceeded。
4. Value 返回context存储的键值对中当前key对应的值，如果没有对应的key,则返回nil。

### 简单的超时取消模型

咱们现在来实现一个超时的例子，其实这个直接调用context的timeout逻辑就可以了，有一张图可以明显说清楚超时处理的逻辑

![](https://blog-1256169066.cos.ap-chengdu.myqcloud.com/golang%E5%B9%B6%E5%8F%912/1.png)

```golang
ctx, cancel := context.WithTimeout(context.TODO(), time.Second*2)
```

看一下context.WithTimeout的源码

```golang
func WithTimeout(parent Context, timeout time.Duration) (Context, CancelFunc) {
    return WithDeadline(parent, time.Now().Add(timeout))
}
```

这个家伙传入了一个时间，并且调用了withDeadline函数，看下它的源码

```golang
func WithDeadline(parent Context, d time.Time) (Context, CancelFunc) {
    if cur, ok := parent.Deadline(); ok && cur.Before(d) {
        // The current deadline is already sooner than the new one.
        return WithCancel(parent)
    }
    c := &timerCtx{
        cancelCtx: newCancelCtx(parent),
        deadline:  d,
    }
    
// 监听parent的取消，或者向parent注册自身
    propagateCancel(parent, c)
    dur := time.Until(d)
    if dur <= 0 {
       // 已经过期
        c.cancel(true, DeadlineExceeded) // deadline has already passed
        return c, func() { c.cancel(false, Canceled) }
    }
    c.mu.Lock()
    defer c.mu.Unlock()
    if c.err == nil {
        c.timer = time.AfterFunc(dur, func() {
            c.cancel(true, DeadlineExceeded)
        })
    }
    return c, func() { c.cancel(true, Canceled) }
}
```

这里输出了两个值，一个是ctx，一个是cancel

你可以这么理解，ctx就是信号，cancel是阀门，ctx的信号结束时，关闭阀门（关闭goroutine）


**写个简单的超时退出模型**

```golang

// 定义一个超时时间/开始时间/信号
var (
	Timeout   = 1 * time.Second
	starttime = time.Now()
	signal    = make(chan bool)
)

// 模拟一个执行时间很长的任务
func do_something_work_1(ctx context.Context, url string) string {
    // 执行任务
	go func() {
        // 假装阻塞在这里
		time.Sleep(time.Second * 10)
        fmt.Println(url)
        // 如果执行完，singal为true
		signal <- true
	}()
	select {
        // 如果接受到关闭信号（超时）
    case <-ctx.Done():
		fmt.Println("tomeout..........")
        fmt.Printf("%.2fs - workers(%s) killed\n", time.Since(starttime).Seconds(), url)
        // 杀掉
		return "tomeout........"
    case <-signal:
    // 如果正常关闭，没超时
        fmt.Println("success")
        // 正常输出，正常杀掉
		return "success"
	}
	return "success"
}

func main() {
    // 定义一个Timeout，超时时间为1s
	ctxt, cancel := context.WithTimeout(context.Background(), Timeout)
	defer cancel()

    // 开始执行goroutine
	go do_something_work_1(ctxt, "hallo")

    //假装在做别的任务 
	time.Sleep(time.Second * 5)

    // 结束主程
	fmt.Println("END")
}

```

这里的输出结果是

```shell
tomeout..........
1.00s - workers(hallo) killed
END
```

解析一下这个框架吧，这个框架实现了一个简单的超时功能.

首先，我定义了一个假设执行时间很长的task，利用context，我给这个task定义了一个超时时间，并且通过select去监控它，当我监控到ctx有输出的时候，那么我就认为task超时，则直接输出超时并且杀死task，如果task直接run到底，那么会有一个信号signal被赋值，select接受到signal被赋值之后，将会认为任务执行成功，输出成功并且杀死task，大概的逻辑就是这样。

## 并发的取消和退出

并发的退出，其实又回到了上面的一个例子，刚才我演示了一个叫做**cancel()**的东西，这个东西就相当于是杀死goroutine的侩子手。来瞅瞅这个东西到底是啥

首先又回到刚那个逻辑

```golang
ctx, cancel := context.WithTimeout(context.TODO(), time.Second*2)
```
它输出了一个ctx，一个cancel，我们看下cancel到底是个什么东西

在源码里面，它是个

```golang
type CancelFunc func()
```

是不是很蒙蔽，这具体是个啥呢，官方文档里面说了

```
A CancelFunc tells an operation to abandon its work. A CancelFunc does not wait for the work to stop. A CancelFunc may be called by multiple goroutines simultaneously. After the first call, subsequent calls to a CancelFunc do nothing.

大意就是这个参数是放弃操作用的，并不等待工作停止，多个任务可调用多个cancel，第一个任务调用cancel之后，其他调用的操作不起作用。
```
还是蒙蔽阿，瞅一眼其他调用代码，继续追踪，最后在withdeadline找到了

```golang
func (c *cancelCtx) cancel(removeFromParent bool, err error) {
	if err == nil {
		panic("context: internal error: missing cancel error")
	}
	c.mu.Lock()
	if c.err != nil {
		c.mu.Unlock()
		return // already canceled
	}
	c.err = err
	if c.done == nil {
		c.done = closedchan
	} else {
		close(c.done)
	}
	for child := range c.children {
		// NOTE: acquiring the child's lock while holding parent's lock.
		child.cancel(false, err)
	}
	c.children = nil
	c.mu.Unlock()

	if removeFromParent {
		removeChild(c.Context, c)
	}
}
```
这部分就相对来说清楚一些了，这里主要是用了一个互斥锁问mutex.lock，看起来是访问共享内存，保护和协调内存访问的，首先先对ctx进行加锁，然后去遍历子goroutine，执行close，没有child之后再进行锁释放，这里的代码真的值得一看。

**并发的退出（context代码）**

```golang
// 定义一个int值，这个值没啥意义，单纯就是做了一个传输，看看就得
var c = make(chan int)

func do_something_work(ctx context.Context) string {
	i := 0

	for {
		select {
        // 接收到退出信号
		case <-ctx.Done():
			fmt.Println("Task Exit")
            return "Task Exit"
        // 做点事儿
		case c <- i * i:
			i++
		}
	}

	return "Task Success"
}

func main() {
	ctx, cancel := context.WithCancel(context.Background())

    // 开始干活儿
	go do_something_work(ctx)

	for i := 0; i < 5; i++ {
		fmt.Println("Next square is", <-c)
	}

    // 杀掉任务
    cancel()

    // 假装还在干别的活儿
    time.Sleep(time.Second * 3)
    
	fmt.Println("END")
}
```

再来一个其他模式的代码，还有一种是通过chan管道传递，其实实现的逻辑也差不多，这里也贴出来，然后给予讲解

**chan类型退出框架**
```golang
func do_somethings(done chan bool, url string) {
	go func() {
		for {
			select {
			// 接受到信号关闭
			case <-done:
				fmt.Println("退出")
				return
			default:
				fmt.Println(url + " " + "执行中")
				time.Sleep(1 * time.Second)
			}
		}
	}()
}

func main() {
	done := make(chan bool)
	// 假装干个活儿
	do_somethings(done, "hello")
	// 假装正在干别的活儿
	time.Sleep(3 * time.Second)
	// 关闭
	close(done)
}

```

这里首先定义了一个chan，通过close chan来进行协程的退出，还是老样子，每次一定要有一个select去监控chan，当执行close时，chan会有输出，当获取到chan输出时，则结束杀死任务。


## goroutine的心跳模式

有些时候，我们需要一个后台常驻任务，去做一些有的没的，但是有些时候不确定goroutine是否还活着，所以你需要每隔一段时间通知一下，报告情况，虽然存在静默状态，但是会隔固定的时间进行一次通知，这里对并发代码很有用，避免了异常挂住或者zombie的问题。

一般我写心跳代码都是后台常驻的服务，比如监控服务，比如时不时跳出来的告警服务，比如前阵子我写了个网络服务，一直就后台挂着等着别人来访问，有些时候不确定啥时候来人访问，就写了个心跳代码，所以我感觉并发里面，心跳还是很有必要的，特别是分布式的系统。

首先看图，心跳在Golang里怎么表现的

![](https://blog-1256169066.cos.ap-chengdu.myqcloud.com/golang%E5%B9%B6%E5%8F%912/2.jpg)

分析下心跳的实现方法

1. 我们建立一个空的channel，设定时间去关闭它
2. 按照时间间隔定时向空的channel发送一个值，类似定时通知机制，并且每次都能读到channel的内容

```golang
func dowork(done <-chan interface{}, pulseInterval time.Duration) (<-chan interface{}, <-chan time.Time) {

	// 设定心跳通道
	heartbeat := make(chan interface{}) //1
	results := make(chan time.Time)

	go func() {
		defer close(heartbeat)
		defer close(results)

		// 心跳间隔
		pulse := time.Tick(pulseInterval) //2

		// 工作间隔
		workGen := time.Tick(2 * pulseInterval) //3

		// 发送心跳
		sendPulse := func() {
			select {
			case heartbeat <- struct{}{}:
			default: //4
			}
		}

		// 发送结果
		sendResult := func(r time.Time) {
			for {
				select {
				case <-done:
					return
				case <-pulse: //5
					sendPulse()
				case results <- r:
					return
				}
			}
		}

		// 循环，如果关闭就renturn，获取pulse就续租
		for {
			select {
			case <-done:
				return
			case <-pulse: //5
				sendPulse()
			case r := <-workGen:
				sendResult(r)
			}
		}
	}()
	return heartbeat, results
}

func main() {
	done := make(chan interface{})
	time.AfterFunc(10*time.Second, func() { close(done) })

	const timeout = 2 * time.Second
	heartbeat, results := dowork(done, timeout/2)
	for {
		select {
		case _, ok := <-heartbeat:
			if ok == false {
				return
			}
			fmt.Println("pulse")
		case r, ok := <-results:
			if ok == false {
				return
			}
			fmt.Printf("results %v\n", r)
		// 没有收到心跳
		case <-time.After(timeout):
			fmt.Println("worker goroutine is not healthy!")
			return
		}
	}
}
```


## 峰值限制（防止过大的并发请求数）

又是抽象的一B的东西，说人话就是，把系统的稳定和平衡性控制在可控范围内。举个简单的例子，如果你是一个动漫网站站长，你每天只允许100个人访问你的网站看动漫，那这个100就是你的限制，就是访问限制，如果第101个人想要看动漫，就只能阻塞在外面继续等待，等100个人中的其中几个人退出才可以继续进去。

这里一般的处理方法是令牌算法

令牌算法的解释如下
```

假设要使用资源，你必须拥有资源的访问令牌。没有令牌，请求会被拒绝。想象这些令牌存储在等待被检索以供使用的桶中。该桶的深度为d，表示它一次可以容纳d个访问令牌。

现在，每次你需要访问资源时，你都会进入存储桶并删除令牌。如果你的存储桶包含五个令牌，那么您可以访问五次资源。在第六次访问时，没有访问令牌可用，那么必须将请求加入队列，直到令牌变为可用，或拒绝请求。

```

写个代码你看看

```golang
    bar24x7 := make(Bar, 10) // 此酒吧只能同时招待10个顾客
    for customerId := 0; ; customerId++ {
        time.Sleep(time.Second)
        consumer := Consumer{customerId}
        select {
        case bar24x7 <- consumer: // 试图进入此酒吧
            go bar24x7.ServeConsumer(consumer)
        default:
            log.Print("顾客#", customerId, "不愿等待而离去")
        }
    }
```

这个其实相对来说可能比较简单，定义一个队列就可以了，但是这不是最好的办法

## 速率限制（设定某一时刻的并发请求数）

和上面那个兄弟的机制没啥两样，其实一个是瞬时请求，一个是总请求，说人话就是，上面那个管全局，下面这个管某一时刻，比如在选课的时候，瞬间就有一大堆人过来抢课，这不把服务器撑爆了。所以，速率限制，常用来限制吞吐和确保在一段时间内的资源使用不会超标。

写个代码你看看

```golang
type Request interface{}

func handle(r Request) { fmt.Println(r.(int)) }

const RateLimitPeriod = time.Minute
const RateLimit = 10 // 任何一分钟内最多处理10个请求
func handleRequests(requests <-chan Request) {
	quotas := make(chan time.Time, RateLimit)
	go func() {
		tick := time.NewTicker(RateLimitPeriod / RateLimit)
		defer tick.Stop()
		for t := range tick.C {
			select {
			case quotas <- t:
			default:
			}
		}
	}()
	for r := range requests {
		<-quotas
		go handle(r)
	}
}
func main() {
	requests := make(chan Request)
	go handleRequests(requests)
	// time.Sleep(time.Minute)
	for i := 0; ; i++ {
		requests <- i
	}
}
```

## 并发最快回应机制

有时候，一份数据可能同时从多个数据源获取。这些数据源将返回相同的数据。因为各种因素，这些数据源的回应速度参差不一，甚至某个特定数据源的多次回应速度之间也可能相差很大。同时从多个数据源获取一份相同的数据可以有效保障低延迟。我们只需采用最快的回应并舍弃其它较慢回应.

写个代码你看看

```golang
package main
import (
    "fmt"
    "time"
    "math/rand"
)
func source(c chan<- int32) {
    ra, rb := rand.Int31(), rand.Intn(3) + 1
    // 睡眠1秒/2秒/3秒
    time.Sleep(time.Duration(rb) * time.Second)
    c <- ra
}
func main() {
    rand.Seed(time.Now().UnixNano())
    startTime := time.Now()
    c := make(chan int32, 5) // 必须用一个缓冲通道
    for i := 0; i < cap(c); i++ {
        go source(c)
    }
    rnd := <- c // 只有第一个回应被使用了
    fmt.Println(time.Since(startTime))
    fmt.Println(rnd)
}
```

差不多就是这样，这里我偷懒了一下，借用了他人的代码，他的环境是，这篇文章给了我很大的启发，感恩！

[通道用例大全](https://www.bookstack.cn/read/golang101/6bbafd2e243d4d07.md)

## 结尾

细细看了一下，写了不少了，但是其中有一两个东西我也没整的太明白，比如goroutine的心跳之类的，这个东西实在是用的不多，所以我感觉我还是，嗯，需要持续性进步才能解决问题，我写的不那么好，如果有错，请老铁们指出来，感恩！

```
予你开心叫我宝宝

予你异见上我镣铐

小人以为我拿它没办法

光脚不惧穿鞋我又有何怕

check~
```