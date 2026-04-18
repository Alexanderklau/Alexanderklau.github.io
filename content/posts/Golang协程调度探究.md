---
categories: ["技术", "Golang"]
title: Golang协程调度探究
date: 2021-11-08 09:43:19
tags: ["前后端"]
---

## 前言

这几天攻关了一下go协程的调度，然后写了这篇文章

这篇文章的工作量真挺大的，我看的也很累，找资料也很累

这篇文章，需要感谢《Go语言底层原理剖析》这本书的作者，这本书给了我很大启发

并且这本书写的非常好，这本书类似我这篇文章的大纲，对我非常有帮助！

希望大家支持这本书的作者！

我的Go系列已经完成如下部分

分别为

1. **[Golang协程基础 （已经完成！）](https://yemilice.com/2021/10/27/golang%E5%8D%8F%E7%A8%8B%E5%9F%BA%E7%A1%80%E6%8E%A2%E7%A9%B6/)**
2. **Golang协程调度 （已经完成！）**
3. Golang协程控制
4. Golang协程通信
5. Golang垃圾回收机制

所以我会持续更新，大家请期待吧，爱你们！

```
总是笃定成功必有收获

每次都是竹篮打水得过且过

总在幻想能写出什么旷世杰作

咽下苦果，传道而授业解惑

```

## 协程的几种状态

### Gidle
表示协程刚开始创建的状态

### Gdead
当新的协程初始化后,会转为Gdead状态,被销毁的时候也是这个状态

### Grunnable
表示协程在运行队列中,正在等待运行

### Grunning
表示协程正在运行,已经分配好了线程 

### Gwaiting
表示协程被所动,不能执行代码,一般垃圾回收,channel通信的时候会出现这种状态

### Gsyscall
表示协程正在执行系统调用

### Gpreempted
表示协程被强制抢占的状态

### Gcopystack
表示发现需要扩容/缩小协程栈空间,转移到新栈的状态

## 协程的状态转移

首先看个图

![](https://blog-1256169066.cos.ap-chengdu.myqcloud.com/%E5%8D%8F%E7%A8%8B2/8.png)

解释下这个图,首先,协程被创建,状态为Gidle

然后协程的状态转为Gdead,再转为Grunnable,等待运行

然后执行协程,状态为Grunning,此时可能出现多个状态

第一个,协程被抢占,转为状态Gpreempted

第二个,协程正在调用,转为状态Gsyscall

第三个,协程正常执行结束,转为状态Gdead

这就是基础的协程状态转移过程,这也是协程的生命周期


## g0协程是什么

在Golang当中，协程分为两种

一种是主协程main

另一种是子协程

主协程只能有一个，但是翻阅一下Golang的源码，每个线程里面都有一个g0协程

首先看下m（线程）的源码

```go
type m struct {
    g0      *g     // 带有调度栈的goroutine
    gsignal       *g         // 处理信号的goroutine
    tls           [6]uintptr // thread-local storage
    mstartfn      func()
    curg          *g       // 当前运行的goroutine
    caughtsig     guintptr
    p             puintptr // 关联p和执行的go代码
    nextp         puintptr
    id            int32
    mallocing     int32 // 状态
    spinning      bool // m是否out of work
    blocked       bool // m是否被阻塞
    inwb          bool // m是否在执行写屏蔽
    .........
}
```

可以看到g0在工作线程m当中，也就是g0协程运行在操作系统栈上

g0，类似于一个特殊角色，每一个m都会有一个g0如影随形，这是每个m被创建开始的第一个协程

它的主要作用就是执行协程调度部分的代码，也就是控制协程切换的

下面我就讲一下g0是怎么搞协程调度和切换的

## g0和协程切换

首先先复习一下GMP原理

这个上一篇博客我有讲过,翻一下就得了

[Golang协程基础探究](https://yemilice.com/2021/10/27/golang%E5%8D%8F%E7%A8%8B%E5%9F%BA%E7%A1%80%E6%8E%A2%E7%A9%B6/)


我们可以知道，P是产生M的，而M产生G，g0是创建的第一个协程goroutine

看个图，最初始的创建状态应该如下所示

![](https://blog-1256169066.cos.ap-chengdu.myqcloud.com/%E5%8D%8F%E7%A8%8B2/4.png)

然后我们假设现在开始执行一个协程g1，如下图所示

![](https://blog-1256169066.cos.ap-chengdu.myqcloud.com/%E5%8D%8F%E7%A8%8B2/1.png)

现在g1出现了问题（超时，停止等）


g2通过g0进行调度，替换g1为最新的执行goroutine

![](https://blog-1256169066.cos.ap-chengdu.myqcloud.com/%E5%8D%8F%E7%A8%8B2/2.png)

g2成为正在执行的goroutine

![](https://blog-1256169066.cos.ap-chengdu.myqcloud.com/%E5%8D%8F%E7%A8%8B2/3.png)

所以它和协程的关系大概就类似这样，如图所示

![](https://blog-1256169066.cos.ap-chengdu.myqcloud.com/%E5%8D%8F%E7%A8%8B2/5.png)

协程的切换过程就是 **g1 -> g0 -> g2**

协程的切换，被称为**协程的上下文切换**

协程g1执行切换的时候，需要保存当前的**执行现场**，这是保证协程切换回来的时候能够正常执行

**执行现场**的源码存储在gobuf的结构体当中，这里面分别保存了rsp，rbp， rip等重要信息，它们是CPU重要的寄存器值。

执行现场gobuf的源码如下

```go
type gobuf struct {
    // 保存CPU的rsp的寄存器值
	sp   uintptr
    // 保存CPU的rip寄存器值
	pc   uintptr
    // 记录gobuf 属于哪个 协程goroutine
	g    guintptr
	ctxt unsafe.Pointer
    // 保存调用的返回值
	ret  uintptr
	lr   uintptr
    // 保存rbp寄存器的值
	bp   uintptr // for framepointer-enabled architectures
}
```

解释下几个重要名词

rsp：始终指向函数调用栈栈顶

rip：执行程序要执行的下一条指令的地址

rbp：存储函数栈帧的起始位置

这几个东西相当于犯罪现场的笔录

当你重返犯罪现场的时候，开启笔录，你依旧可以开始调查，不会停止。

## g0的总结

所以，调度协程g0和普通的协程g完全不同

g0作为调度协程，执行的函数和流程是固定的，并且为了避免栈溢出

g0栈是会重复使用的

g0不仅负责协程调度，还负责

1. 垃圾回收
2. 动态栈增长

这就是调度协程g0的主要功能


## 工作线程的绑定

工作线程是什么鬼

通俗来说，GMP里面，M就是工作线程

一般Go的调度器，使用**线程本地存储**将操作系统线程和代表线程的M结构体绑定在一起，然后才能继续其他的操作

所以我们现在就来研究下**线程本地存储**这个东西

首先看下M结构体的源码

```go
type m struct {
    ......
    tls           [tlsSlots]uintptr // thread-local storage (for x86 extern register)
    ......
}
```

我这里截取了关键代码，tls指向的是**本地存储的线程的地址**

艹，太绕了，我们来追一下下**本地存储的线程的地址**是个什么鸟玩意儿

首先，线程本地存储是一种计算机方法，使用线程本地的静态和全局内存，线程本地存储的变量值，仅仅只对当前线程可见

所以这种线程存储变量是**私有的**，操作系统使用的是FS/GS存储线程本地变量

现在，替换一下我们前面的理解

结构体M存储的是线程本地存储中线程的地址，所以在线程内部，我们可以获取当前线程的协程g, 逻辑处理器p等信息。

结构体M存储在Fs寄存器当中，FS寄存器里面又存储了本地的线程变量，因为这个关系，从而实现工作线程和结构体M之间的绑定

太TM饶了这个

## 调度循环

那啥，这里不是重复一遍，上面讲的是协程切换

这里是协程调度循环，是一个循环的流程，全称叫做调度循环

我简单说一下吧

就是从协程调度g0开始，找到要运行的协程g1进行切换

然后切换回g0，通过不通的调度策略，获取其他的g，进行切换，直到没有g为止

这里和协程切换不一样的一点是，协程切换关心的是协程状态，调度循环关注的是调度流程

我拿蒋校长微操举个例子

蒋校长告诉你我们要打徐州会战

协程切换管的是切换的具体状态，

协程切换就是开打，部队冲，打输了，退下来，优势在我这种状态

![](https://blog-1256169066.cos.ap-chengdu.myqcloud.com/%E5%8D%8F%E7%A8%8B2/7.png)

协程调度循环管的是切换的具体流程，

也就是怎么打，换谁上，先打哪里，先调谁，后用谁，机枪右移50米，空投手令之类的。

这就是协程调度循环

### 细化协程调度循环

调度循环的流程，可以参看下图，我将它们分为两个阶段进行解说

看图，这里是个具体流程

![](https://blog-1256169066.cos.ap-chengdu.myqcloud.com/%E5%8D%8F%E7%A8%8B2/5.png)

开始分析

首先，协程g0切换到用户协程g，经历了schedule函数到execute函数再到gogo函数的过程

其中用到的函数功能如下
1. schedule函数 调度策略合集，处理调度
2. execute函数 状态转移，g和m之间的绑定
3. gogo函数 操作系统函数，完成栈切换和CPU寄存器的恢复等

这里细化一下流程图

接下来进入第二阶段

切换到协程g之后，g开始执行，当g出现了主动让渡，抢占，退出之后，将会执行到协程g0开始第二轮调度

首先，从协程g切换到协程g0的时候，mcall用来保存当前协程的执行现场，这个我们上两节说过，gobuf存储了执行现场。然后进行切换，切换到g0之后，会根据切换的原因进行不同的函数选择

具体的状态有如下几种

1. 主动让渡 这时候需要调用Gosched函数
2. 协程退出 这时候需要调用Goexit函数

执行完毕之后，将会切换回g0函数，执行第一步阶段，开始下一个循环，形成闭环，无限重回


## 调度策略

上面我讲了调度循环，里面讲到一个函数**schedule**

这个函数是总管协程的调度策略的，相当于国防部

这个代码的放置位置是 "runtime/proc.go"

这里简单给大家看几个重要的函数

```go
func schedule() {
    // 获取到当前的goroutine
	_g_ := getg()
        ...


	var gp *g
	var inheritTime bool

	// 检查goroutine是否需要唤醒
	tryWakeP := false
	if trace.enabled || trace.shutdown {
		gp = traceReader()
		if gp != nil {
			casgstatus(gp, _Gwaiting, _Grunnable)
			traceGoUnpark(gp, 0)
			tryWakeP = true
		}
	}
	if gp == nil && gcBlackenEnabled != 0 {
		gp = gcController.findRunnableGCWorker(_g_.m.p.ptr())
		if gp != nil {
			tryWakeP = true
		}
	}
	if gp == nil {
        // 检查全局队列，每隔一段时间
		// Check the global runnable queue once in a while to ensure fairness.
		// Otherwise two goroutines can completely occupy the local runqueue
		// by constantly respawning each other.
		if _g_.m.p.ptr().schedtick%61 == 0 && sched.runqsize > 0 {
			lock(&sched.lock)
			gp = globrunqget(_g_.m.p.ptr(), 1)
			unlock(&sched.lock)
		}
	}
	if gp == nil {
		gp, inheritTime = runqget(_g_.m.p.ptr())
		// We can see gp != nil here even if the M is spinning,
		// if checkTimers added a local goroutine via goready.
	}
	if gp == nil {
		gp, inheritTime = findrunnable() // blocks until work is available
	}
}
```

在这个函数里面，首先会检测程序是否处于垃圾回收阶段，然后再检测是否需要标记协程

接下来是几个重点概念

1. Go会使用队列，将等待执行的协程存放在其中
2. Go的协程队列分为**局部运行队列**和**全局运行队列**
3. Go的运行队列是一个**先入先出**的队列

来看一下Go调度器P的源码


局部队列源码
```go
type p struct {
	id          int32
	status      uint32 // one of pidle/prunning/...
        .....

	// Queue of runnable goroutines. Accessed without lock.
        // 可运行的协程队列
	runqhead uint32
	runqtail uint32
        // 256个局部运行队列
	runq     [256]guintptr
        // 指向下一个要执行的协程
	runnext guintptr
        .....
}
```

全局队列源码

```go

// 有一个头，看起来类似next的链表
type gQueue struct {
	head guintptr
	tail guintptr
}

// 这里是全局列表的代码
type schedt struct {
	// accessed atomically. keep at top to ensure alignment on 32-bit systems.
	goidgen   uint64
	// Global runnable queue.
        // 全局队列源码
	runq     gQueue
	runqsize int32
        ....
}

```

现在我们有了全局队列和局部队列

### 调度的优先级

上面我们引入了两个很重要的概念，**全局队列** 和 **局部队列**

那么我们原有的Golang的GMP模型就可以进行一下修改

看个图

![](https://blog-1256169066.cos.ap-chengdu.myqcloud.com/%E5%8D%8F%E7%A8%8B2/9.png)

首先，M生成P，P会去**局部队列**里获取G，然后**全局队列**在此处Stand By。

所以可以总结出一般的调用逻辑

根据上述的那张图，一般的思路是先去查找每个P的局部队列获取G，当局部队列为空的时候，再从全局队列中获取G，直到都没有G为止

但是这里涉及一个很大的问题，**就是很可能如果循环执行局部队列的G，就可能导致全局队列的G永远不会被运行获取。**

差了一些资料，Go里解决这个问题的策略是：

```shell
当P中执行了61次调度之后，就必须从全局队列获取一个G，到当前P中执行
```

这里看下源码


源码位置在：src/runtime/proc.go ，依旧还是那个调度函数schedule
```go
	if gp == nil {
		// Check the global runnable queue once in a while to ensure fairness.
		// Otherwise two goroutines can completely occupy the local runqueue
		// by constantly respawning each other.
		// 如果可以被61整除并且全局队列的数量大于0
		if _g_.m.p.ptr().schedtick%61 == 0 && sched.runqsize > 0 {
			lock(&sched.lock)
			// 获取一个g
			gp = globrunqget(_g_.m.p.ptr(), 1)
			unlock(&sched.lock)
		}
	}
```

我们分析了队列调度的先后，那么就可以规划出来队列调度的优先级顺序了

梳理了一下，有个图，大家伙看看

![](https://blog-1256169066.cos.ap-chengdu.myqcloud.com/%E5%8D%8F%E7%A8%8B2/10.png)

首先，P执行调度的时候，首先会通过runnext获取下一个可执行的G，如果获取不到

那么将尝试从局部队列寻找G，如果局部队列没有G了,就会从全局队列寻找G

如果全局队列也没有G，那么将会从其他的P中窃取G进行运行

如果都找不到G，那么P将会解除和M的绑定，M也进入休眠状态中

### 获取局部队列

上一节我们梳理了什么是局部队列，什么是全局队列，以及他们的优先度，这段我就来说下他们是怎么获取的

先看一下源码

```go
func runqget(_p_ *p) (gp *g, inheritTime bool) {
	// 获取下一个可用的G
	next := _p_.runnext
	// 如果下一个next地址不等于0并且cas成功，直接返回一个G
	// 如果下一个next地址不等于0并且cas失败，那么它只能被另一个P抢夺
	// 其他P可以设置为0，当前只有P可以设置非0
	// 只有CAS失败，则不需要重试
	if next != 0 && _p_.runnext.cas(next, 0) {
		return next.ptr(), true
	}

	for {
		// 获取头部
		h := atomic.LoadAcq(&_p_.runqhead) // 加载，获取，和其他P同步
		// 获取尾部
		t := _p_.runqtail
		// 如果头尾相等，那么没有协程可以运行
		if t == h {
			return nil, false
		}
		// 访问加锁
		gp := _p_.runq[h%uint32(len(_p_.runq))].ptr()
		if atomic.CasRel(&_p_.runqhead, h, h+1) { // 推送执行了多少G
			return gp, false
		}
	}
}
```

首先这里先检查runnext是否为空，如果不为空就直接拿出G

如果为空就从局部队列进行查找，从队列头部获取到一个G然后返回

循环获取头部尾部，头尾相等，证明没有协程可用，返回nil和false

返回有个访问加锁，这里是避免其他P窃取任务时和当前P进行同时访问

这里大概就是获取局部队列的一些相关知识了，这里看的很累，比较复杂

### 获取全局队列

前面说过，当P执行了61次调度的时候，就会从全局队列里面拿取G优先执行

全局队列，其实就是全局链表，每个P都可以从里面拿东西，有点类似消费者生产者的模型

梳理一下队列转移的方法

```
首先，我们要先统计P的数量，然后根据P的数量平均分配全局队列的G，

但是要拿取的数量不能超过局部队列的一半，当前是 256 / 2 = 128，

也就是说，不能一次性拿走超过128个G放到自己的局部队列里

获取到G之后，再通过循环将全局队列的G放入平均分配的P的局部队列当中
```

下面看一些源码

```go
// Try get a batch of G's from the global runnable queue.
// sched.lock must be held.
func globrunqget(_p_ *p, max int32) *g {
	assertLockHeld(&sched.lock)

	if sched.runqsize == 0 {
		return nil
	}

	// 平均分配
	// n就是每个p获取的g的数量
	n := sched.runqsize/gomaxprocs + 1
	if n > sched.runqsize {
		n = sched.runqsize
	}
	if max > 0 && n > max {
		n = max
	}
	if n > int32(len(_p_.runq))/2 {
		n = int32(len(_p_.runq)) / 2
	}

	// 全局队列 - n
	sched.runqsize -= n

	// gp = 队伍的最后一个
	gp := sched.runq.pop()
	// 递减
	n--
	// 如果 n > 0
	for ; n > 0; n-- {
		gp1 := sched.runq.pop()
		// 上传
		runqput(_p_, gp1, false)
	}
	return gp
}
```





### 协程窃取

如果一个P，获取不到它自己的局部队列的G，并且全局队列的G他也拿不到，那么他就可以去其他的P里面拿G过来跑

这是Go很巧妙的一个设计逻辑，避免的持续性等待，空放P在那什么也不做

首先这里分为两个部分讲解

第一个是从其他P获取到G

第二个是窃取G到自己的P

那么，开始吧

#### 从其他P获取G

当自己的局部队列和全局队列都拿不到G的时候，则会开启这个分支剧情

首先需要获取到所有的P，然后通过P获取到他们的局部队列

这里最重要的几个特性就是

1. 随机性
2. 公平性

首先P存储在全局的allp[] 里面，这是一个全局变量

获取P的源码如下

```go
//从任何P中进行窃取的函数
func stealWork(now int64) (gp *g, inheritTime bool, rnow, pollUntil int64, newWork bool) {
	//获取到G
	pp := getg().m.p.ptr()

	ranTimer := false

	//重复4次操作
	const stealTries = 4
	for i := 0; i < stealTries; i++ {
		stealTimersOrRunNextG := i == stealTries-1

		// 二层循环，找到可窃取的P马上返回
		for enum := stealOrder.start(fastrand()); !enum.done(); enum.next() {
			if sched.gcwaiting != 0 {
				// GC work may be available.
				return nil, false, now, pollUntil, true
			}
			p2 := allp[enum.position()]
			if pp == p2 {
				continue
			}

			// 从P2里面进行窃取
			if stealTimersOrRunNextG && timerpMask.read(enum.position()) {
				tnow, w, ran := checkTimers(p2, now)
				now = tnow
				if w != 0 && (pollUntil == 0 || w < pollUntil) {
					pollUntil = w
				}
				if ran {
					// 检查现在是否要运行的本地G
					if gp, inheritTime := runqget(pp); gp != nil {
						return gp, inheritTime, now, pollUntil, ranTimer
					}
					ranTimer = true
				}
			}

			//如果P2空闲，不进行窃取
			if !idlepMask.read(enum.position()) {
				if gp := runqsteal(pp, p2, stealTimersOrRunNextG); gp != nil {
					return gp, false, now, pollUntil, ranTimer
				}
			}
		}
	}

	// 没有发现的协程G，持续等待或者推出
	return nil, false, now, pollUntil, ranTimer
}
```

这里的代码比较难整，简单说一下意思

第一层For循环，意思就是重复执行4次循环逻辑，至于循环什么呢。。。

```go
	const stealTries = 4
	for i := 0; i < stealTries; i++ {
		stealTimersOrRunNextG := i == stealTries-1

		// 二层循环，找到可窃取的P马上返回
		for enum := stealOrder.start(fastrand()); !enum.done(); enum.next() {
			if sched.gcwaiting != 0 {
				// GC work may be available
	.....
```

这里有一个二层循环

二层循环里面用了一个随机算法，这里我直接讲一下这个随机算法吧

我们用一个例子来说明， 假设一共有8个P， 

第1步: fastrand 函数 选择一个随机数并对8取模， 算法选择了一个0 - 8之间的随机数，假设为6

第2步，找到一个比8小且与8互质的 数。 比8小且与8互质的数有4个： 

coprimes=[1，3，5，7]， 

代码中取coprimes[6%4]= 5， 这4个数中任取一个都有相同的数学特性。

大概就是这样，这部分我直接参考了 **Go语言底层剖析**这本书，我属实没看明白这段。

#### 窃取G到现在的P

这部分的源码如下

```go
func runqgrab(_p_ *p, batch *[256]guintptr, batchHead uint32, stealRunNextG bool) uint32 {
	for {
		h := atomic.LoadAcq(&_p_.runqhead) // load-acquire, synchronize with other consumers
		t := atomic.LoadAcq(&_p_.runqtail) // load-acquire, synchronize with the producer
		n := t - h
		// 偷取一半的G个数
		n = n - n/2
		.....
		// 放入自己的队列
		for i := uint32(0); i < n; i++ {
			g := _p_.runq[(h+i)%uint32(len(_p_.runq))]
			batch[(batchHead+i)%uint32(len(batch))] = g
		}
		if atomic.CasRel(&_p_.runqhead, h, h+n) { // cas-release, commits consume
			return n
		}
	}
}
```

这里窃取了其他指定P一般的G的个数到自己队列当中，这段核心代码大概就是这个意思



上一节讲了调度策略，讲的是大战略方针，调度时机就是讲的什么时候发生调度

还是拿蒋公举个例子

蒋公定了徐州战役，现在要打，订好了策略，那什么时候打？

调度时机就是说明了时间，也就是什么时候发生调度，生效调度策略之类的。

一般来说，调度时机分为几种

1. 主动调度
2. 被动调度
3. 抢占调度

下面就分别来讲一下这三种调度时机

### 主动调度

协程可以主动选择过渡自己的执行权利，也就是让出自己的执行权利给其他的G协程。

在大多数情况下，我们（开发人员）是不会去执行这种调度的，因为Go会默认的主动检查调用函数，判断G是否被抢占

主动调度的核心代码如下

```go
func goschedImpl(gp *g) {
	// 判断g的状态
	status := readgstatus(gp)
	// 如果g不是_Grunning状态
	if status&^_Gscan != _Grunning {
		// 获取g的详细信息
		dumpgstatus(gp)
		// 输出提示
		throw("bad g status")
	}
	// 取消g和m之间的绑定关系
	casgstatus(gp, _Grunning, _Grunnable)
	dropg()
	// 加锁g
	lock(&sched.lock)
	// g放入全局运行队列
	globrunqput(gp)
	// 关闭锁
	unlock(&sched.lock)
	// 新一轮调度
	schedule()
}
```

这里的大致意思就是，首先检查g的状态，然后从当前协程切换到g0

首先取消g和m的绑定关系，然后把g放入全局队列，然后开始新一轮循环（调用schedule函数）

### 被动调度

这里就是可以被我们（开发人员）所控制的调度了

一般我们在开发里面会遇到例如应用休眠，timeout，堵塞等情况

所以应用内部的协程这时候就会被动的将自己的执行权限交出去

因为不同的原因，调度器可能执行的操作也不同

被动调度是协程内部发起的操作

它的源码如下

```go
func gopark(unlockf func(*g, unsafe.Pointer) bool, lock unsafe.Pointer, reason waitReason, traceEv byte, traceskip int) {
	if reason != waitReasonSleep {
		checkTimeouts() // timeouts may expire while two goroutines keep the scheduler busy
	}
	mp := acquirem()
	gp := mp.curg
	status := readgstatus(gp)
	if status != _Grunning && status != _Gscanrunning {
		throw("gopark: bad g status")
	}
	mp.waitlock = lock
	mp.waitunlockf = unlockf
	gp.waitreason = reason
	mp.waittraceev = traceEv
	mp.waittraceskip = traceskip
	releasem(mp)
	//执行被动调度
	mcall(park_m)
}
```

这个函数会去执行park_m这个函数

```go
func park_m(gp *g) {
	_g_ := getg()

	if trace.enabled {
		traceGoPark(_g_.m.waittraceev, _g_.m.waittraceskip)
	}

	// 解除G和M的关系
	casgstatus(gp, _Grunning, _Gwaiting)
	dropg()

	// 判定执行被动调度的原因
	if fn := _g_.m.waitunlockf; fn != nil {
		ok := fn(gp, _g_.m.waitlock)
		// 执行waitlock
		_g_.m.waitunlockf = nil
		_g_.m.waitlock = nil
		if !ok {
			if trace.enabled {
				traceGoUnpark(gp, 2)
			}
			casgstatus(gp, _Gwaiting, _Grunnable)
			execute(gp, true) // Schedule it back, never returns.
		}
	}
	// 重新调度
	schedule()
}
```

首先，还是老样子，切换协程到g0，然后解除G和M的关系，根据执行被动调度的不同原因，执行不同的wautunlockf函数

但是要注意一点，被动调度的函数不会放入全局队列

协程的状态会从_Gwaiting转换为_Grunnable

然后放入当前P的局部队列当中，开始执行。

### 抢占调度

抢占调度的源码如下

```go
func retake(now int64) uint32 {
	n := 0
	lock(&allpLock)
	// 重新获取allp
	for i := 0; i < len(allp); i++ {
		_p_ := allp[i]
		if _p_ == nil {
			// This can happen if procresize has grown
			// allp but not yet created new Ps.
			continue
		}
		pd := &_p_.sysmontick
		s := _p_.status
		sysretake := false
		if s == _Prunning || s == _Psyscall {
			// Preempt G if it's running for too long.
			t := int64(_p_.schedtick)
			if int64(pd.schedtick) != t {
				pd.schedtick = uint32(t)
				pd.schedwhen = now
			} else if pd.schedwhen+forcePreemptNS <= now {
				preemptone(_p_)
				// 系统调用 没有连接到P的M
				sysretake = true
			}
		}
		if s == _Psyscall {
			// 如果存在1个sysmon，从系统调用中获取P
			t := int64(_p_.syscalltick)
			if !sysretake && int64(pd.syscalltick) != t {
				pd.syscalltick = uint32(t)
				pd.syscallwhen = now
				continue
			}
			// 防止进入深度睡眠无法huan'x
			if runqempty(_p_) && atomic.Load(&sched.nmspinning)+atomic.Load(&sched.npidle) > 0 && pd.syscallwhen+10*1000*1000 > now {
				continue
			}
			unlock(&allpLock)
			// 较少空闲锁定M的数量
			// 增加nmidle报告死锁
			incidlelocked(-1)
			if atomic.Cas(&_p_.status, s, _Pidle) {
				if trace.enabled {
					traceGoSysBlock(_p_)
					traceProcStop(_p_)
				}
				n++
				_p_.syscalltick++
				handoffp(_p_)
			}
			incidlelocked(1)
			lock(&allpLock)
		}
	}
	unlock(&allpLock)
	return uint32(n)
}
```

这段代码头晕了，大致意思就是如果协程运行时间过长，或者处于系统调度阶段

则会抢占当前G的执行

## 总结

这篇文章其实挺费力的，我本来就不是什么勤奋的人

但是写下来没少花功夫，最近感觉自己在逐渐懈怠

这样不太行，不太可，还是要继续hold on

chill man
