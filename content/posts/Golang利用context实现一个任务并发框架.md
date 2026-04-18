---
layout: new
title: Golang利用context实现一个任务并发框架
date: 2020-03-20 15:57:48
tags: ["前后端"]
---
# Golang利用context实现一个任务并发框架
## 消失这么久的原因
疫情太严重，哥们本来打算在新疆滑雪+吃烤肉度过一个美好的假期，结果没成想给困那里了，这不就尴尬了么，这不，博客没更新，现在我又回来了，哈哈哈哈！
## 我要实现个什么玩意儿
有一个需求，简单的说就是我要写一个任务管理框架，主要功能有任务开启，任务关闭，任务监控等等。说的抽象点，就是我要用Golang写一个任务管理的功能，任务很可能有多个，并且我想停任务就停任务，想开始任务我就开始任务！我不要你觉得，我要我觉得！
## 为什么我要用Golang
众所周知，golang这东西，有个黑科技，叫goroutine，这东西很牛逼，牛逼在哪儿呢？简单的说，快，小，短！协程切换快，占用资源少，并且，异步的，可以开多个，直接一个go 关键字就给人家打开了，多棒！  
## 实际需求分析
结合我们上面的任务需求，实现一个任务开始，任务停止的逻辑，这就说明，任务肯定不只有一个，并且任务都在后台，我们该怎么去监控，或者去管理这个go任务，或者go函数呢，golang提供了很多解决办法，例如WaitGroup，context等方法。  
思考一下，我们的这个需求，任务都是跑在后台的异步并发逻辑，这就说明不只一个任务会被启动和停止，这样对我们的任务管理是一个很大的挑战，因为任务都是在后台隐秘执行的，如果是一般逻辑，我们要停止任务，首先要找到任务的pid，然后kill任务进程，这是一个完整的结束任务的流程。  
回到我们这个需求，基本的流程就是：  

发送一个任务请求(开启任务/停止任务) -> 接收到任务请求 -> 执行任务请求   

思考一下，如果，我们开启任务之后，任务进入后台，那么，我们在停止任务的时候，怎么保证，能够找到这个任务，精准的打击（停止）它呢？看了题目你应该就知道了，用context就好了。下面我就来介绍一下它吧。

## 主角context介绍

网上很多博客介绍它，我粗粗看了一眼，非常抽象，很多人再一描述，就更麻烦更抽象了。我这里不说的太麻烦，简单描述一下，这个东西context，是干嘛呢，你们理解一下株连，连坐这两个词汇，这个东西相当于就是锁链，铁锁连舟，不进则退，说明白点，就是一个串连上下文的类似信号传递的玩意儿。每个调用链上的函数都要以它作为函数进行传递，举个例子,"株九族"这个词，是因为一个人犯罪，结果家人都因为他被砍了头，这个犯罪的人，就是父context，其他因为他被杀的人，就是子context，还可能有孙context，他被砍头了，其他人也得跟着一起死，用代码表示一波

```go
// 这是儿子函数
func gen(ctx context.Context) <-chan int {
	dst := make(chan int)
	n := 1
	go func() {
		for {
			select {
            // 接收到爹挂了的消息
			case <-ctx.Done():
                fmt.Println("儿子被砍头了。")
                // 退出任务
				return
			case dst <- n:
				n++
				time.Sleep(time.Second * 1)
			}
		}
	}()
	return dst
}
// 这是爹函数
fun test() {
    ctx, cancel := context.WithCancel(context.Background())
	// 让我造个儿子，给我儿子传个ctx
    intChan := gen(ctx)
    // 我被干了，cancel是结束
    defer Cancel()
}
```

这下说的明白了么？其实context还有很多别的，例如timeout之类的，但是那个是我后面准备写的，这一节就不写这些了。
## 实现我们的需求
我们的武器context已经准备好了，大概的使用逻辑我们也明白了，现在你们可以看到，我们只要拿到主函数（爹函数）的ctx和cancel，我们就可以控制子函数（儿子函数）的死活，我们在开发当中，任务的状态是不断在变化的，一个爹对应一个儿子，但是可能有多个任务，多个任务我们该怎么管理它？  
在一般的开发任务中，我们习惯将任务记录到数据库当中，然后在开发当中不停的遍历数据库，去判断任务的状态到底是开启还是停止，这里我们要考虑到，频繁遍历数据库，会不会带来大量的访问堆积？还是否有别的解决办法？  
### 我的解决方案
这次开发中，我选择定义一个全局变量的主map，并且定义一个任务的struct类型，代码如下

``` go
// 全局map
var jobmap = make(map[string]interface{})
//Job 任务
type Jobs struct {
	ID     string
	Status int
	Ctx    context.Context
	Cancel context.CancelFunc
}
```

然后我选择在任务开始的时候（创造儿子的时候），将信息填充，修改test代码如下

```go
func test() {
	var jobs Jobs
    ctx, cancel := context.WithCancel(context.Background())
    // 造一个儿子
    intChan := gen(ctx)
    // 任务开始了
    fmt.Println("start job")
    // 重要的东西传进去
	jobs.Status = 1
	jobs.Cancel = cancel
    jobs.Ctx = ctx
    // 定义一个任务id，这个可以用uuid，或者随便整个别的
	jobs.ID = "sdads"
    m1["sdads"] = jobs
    // 阻塞任务，假装任务执行很久
	for n := range intChan {
		fmt.Println(n)
		if n == 1000 {
			break
		}
	}
```

再然后，我选择写一个停止函数（砍头函数）

```go
func stopGetmi(id string) {
	//把任务停掉
	fmt.Println("stop jobs")
    jobss := m1[id]
    //interface 转 struct
    op, ok := jobss.(Jobs)
    // 调用砍头函数cancel
	defer op.Cancel()
}
```

进行测试

```go
func main() {
	go test()
	go stopGetmi("sdads")
	time.Sleep(time.Second * 200)
}
```

发现任务执行结果  
![](https://blog-1256169066.cos.ap-chengdu.myqcloud.com/images/3.png)  
这样你就完成了干掉老爹，也干掉儿子的素质操作。
## 总结
主要是context的基础和说明，其实context我还是推荐大家去看看原版，我这里写的太过于简单，不过这篇博客，也是我记录一下自己开发中遇到的难题，当时看网上没有类似的说明，于是写了这篇博客，希望大家多多包涵。  
祝大家都能躲过瘟疫，我们终究会在春花花开的地方相见。