---
title: 完全理解Golang并发模式(1)
date: 2020-11-03 18:22:08
tags: ["前后端"]
---

## 前言
其实我写Golang有段时间了，平常写一些业务代码，也涉及不到什么高端东西，Golang的核心其实就是在于**goroutine**这套东西，在处理高并发任务的时候非常强势，甩Python一个车位，所以，写下这篇博客，作为笔记，记录Golang核心的并发机制，并且给出一些自己写的代码作为示例，本次博客大部分代码为自己手写，参考了《Golang并发之道》，《Go语言圣经》这两本书，这两本书是我的进阶之路上的灯塔，感谢这两本书的译者，也希望大家去看看这两本书。

## Golang在并发处理上有什么优势

一个是方便，一个是随用随写。

Golang的特性goroutine是Go的基础语法，一般调用的方法十分简单

```golang
go func()
```

这时的goroutine是一个并发的函数，说起来非常抽象，举个例子，你现在想要执行一个打招呼的逻辑

```golang
func Work(){
    fmt.Println("im working!")
}

func main() {
    go Work()
}
```

一个基础打招呼的逻辑就形成了。同理，你可以干多个事儿，类似

```golang
func Work(){
    fmt.Println("im working!")
}

func Exit(){
    fmt.Println("im go home!")
}

func main() {
    go Work()
    go Exit()
}
```
这些都是goroutine的好处，就是随便写随便用，混用混写。

## Golang并发模型的标准

Go遵循一种 fork-join的并发模型标准/规则

fork的意思是程序中的任意一点，goroutine和主程序一起运行，join的意思他们最终合并在一起，用中国话来说叫殊途同归，拿上面那个打招呼的例子详细说一下。

```golang
...
func main() {
    go Work()
    go Exit()
    fmt.Println("END")
}
```
如果你自己跑一下这个程序，就有可能发现，END的输出，是不会等待你的work和exit完成的，很大概率几乎这两个goroutine都没执行，说白了就是，主程跑起来，跑完了，你子程，我才不管你跑了没呢！像不像个渣男，只顾着自己爽，出来了就完了。

画个图解释一下

![](https://blog-1256169066.cos.ap-chengdu.myqcloud.com/golang%E5%B9%B6%E5%8F%91%E6%A8%A1%E5%BC%8F/%E5%B9%B6%E5%8F%911test.jpg)

这里其实是没有join的，这么想一下吧，大家都坐过高铁，高铁上人来人往，把高铁想成主函数main，我们每个人就是goroutine，我们要去的终点是不一样的，如果有人在中途下车上厕所，没上来，不通知高铁一声，高铁就跑了，就不会管你是不是到终点，所以这个join，就类似通知，或者是链接主函数的一个点。现在改写一下这个程序。

利用WaitGroup来改写一下，对主程进行一个等待和通知的操作

```golang
var wg sync.WaitGroup
Work := func() {
    defer wg.Done()
    fmt.Println("im working!")
}
Exit := func() {
    defer wg.Done()
    fmt.Println("im go home!")
}
wg.Add(2)
go Work()
go Exit()
wg.Wait()
fmt.Println("END")
```

这里写的稍微粗糙了一点，其实很简单，利用了waitgroup建立一个连接点，这里其实就会等待work和exit完全执行完毕再进行END的输出。可以总结成一个图

![](https://blog-1256169066.cos.ap-chengdu.myqcloud.com/golang%E5%B9%B6%E5%8F%91%E6%A8%A1%E5%BC%8F/%E5%B9%B6%E5%8F%912.jpg)

总结一下，Go的并发模型，就是**无论你goroutine怎么疯，怎么耗时，怎么难搞，都要通过join逻辑通知主程去做一个等待/通知，从而实现并发。**

这个东西应该才是Golang并发模式的核心吧，以后编写的代码，也都是遵循这种框架。

## Golang并发的几个基础用例解析随想

想用Golang实现并发模型，一般有几个固定的组件，例如**waitgroup**, **context**, **channel**等，大家都会根据自己项目的需求选择不一样的并发组件进行开发，我记得我以前针对这三种都写过对应的博客，分别是

1. [context随想](https://yemilice.com/2020/06/01/golang%E4%B8%ADcontext%E4%BD%BF%E7%94%A8%E7%9A%84%E4%B8%80%E7%82%B9%E9%9A%8F%E6%83%B3/)
2. [Watigroup随想](https://yemilice.com/2020/05/26/golang%E4%B8%ADwaitgroup%E4%BD%BF%E7%94%A8%E7%9A%84%E4%B8%80%E7%82%B9%E9%9A%8F%E6%83%B3/)
3. [Channel随想](https://yemilice.com/2020/05/25/golang%E4%B8%ADchannel%E4%BD%BF%E7%94%A8%E7%9A%84%E4%B8%80%E7%82%B9%E9%9A%8F%E6%83%B3/)

这三篇Blog都已经大概说了最常见的几种并发模型，我这几天查了一点资料，看了一下Golang并发的几个用例，发觉遗漏了一些重要部分，这里还是继续更新一下吧。

我会将基础的使用场景和用例代码贴出，并且加上我自己的解析，如果有错误，请给予我指点，感谢！

### 如何处理并发循环问题？

有时候我们可能会面临这种需求，一次性访问多个网页，或者一次循环切片中多个数据，类似这样

```golang

func do_something(url string) {
	fmt.Println(url)
	time.Sleep(time.Second * 3)
}

func main() {
	url_lists := []string{"baid.com", "wangyi.com"}
	for _, url := range url_lists {
		do_something(url)
	}

	fmt.Println("END")
}

```

这时的循环模式是单个循环，一循到底部，这里就是一个个循环下去，效率非常低那种。

**我们现在把它改成并发循环**

#### 改写我们的并发循环代码，实现基础并发循环

其实很简单，改动一个地方就行

```golang

func do_something(url string) {
	fmt.Println(url)
	time.Sleep(time.Second * 3)
}


func main() {
	url_lists := []string{"baid.com", "wangyi.com"}
	for _, url := range url_lists {
		go do_something(url)
	}

	fmt.Println("END")
}
```
在这里，你会发现压根没打印就直接输出了END，看到了吧，问题来了，你没有遵循fork-join的并发逻辑。

你是不是想加个Sleep去等待它完成，其实你想的没错，但是Sleep不是一个join，它只是一个计时器而已。

赶紧，上一个WaitGroup，你也可以用chan，这都是一样的。

waitgroup写法
```golang

var wg sync.WaitGroup

func do_something(url string) {
	defer wg.Done()
	fmt.Println(url)
	time.Sleep(time.Second * 3)
}

func main() {
	url_lists := []string{"baid.com", "wangyi.com"}
	for _, url := range url_lists {
		wg.Add(1)
		go do_something(url)
	}
	wg.Wait()
	fmt.Println("END")
}
```

chan写法

```golang
func do_somethings(url string) {
	fmt.Println(url)
	time.Sleep(time.Second * 2)
}

func main() {
	ch := make(chan struct{})
	ips := []string{"string1", "string2"}
	for _, ip := range ips {
		go func(ip string) {
			do_somethings(ip)
			ch <- struct{}{}
		}(ip)
	}

	for range ips {
		<-ch
	}
}
```

#### 并发循环中Wg的控制

上面的代码里，有个函数叫做

```golang
wg.Add(1)
```
这兄弟是干嘛的呢，它其实就是增加一个计数器，你可以认为他就是增加一个子线程

```golang
wg.Done()
```
这个兄弟是减去一个计数器，意思就是销毁，干掉

```
wg.Wait()
```
这个兄弟是阻塞主程的，直到计数器为0的时候，再继续往下。

#### 并发循环中输出错误

有些时候在并发循环中遇到了错误，不能够忽略，要求即刻输出，改写一下我们刚才的代码。

chan写法
```golang
func do_somethings(url string) (err error) {
	fmt.Println(url)
	if url == "string2" {
		return fmt.Errorf("bad............")
	}
	time.Sleep(time.Second * 2)
	return nil
}

func main() {
	ips := []string{"string1", "string2"}
	errors := make(chan error)
	for _, ip := range ips {
		go func(ip string) {
			err := do_somethings(ip)
			errors <- err
		}(ip)
	}
	for range ips {
		if err := <-errors; err != nil {
			fmt.Println(err)
			return
		}
	}
}
```

其实chan就是在各个goroutine中做传递的管道，相当于感情的通讯员~


#### 控制并发goroutine的数量

举个简单的例子，如果我们的机器太破，咱不可能开无限个goroutine去做事儿吧，有些时候我们需要对并发的数量进行控制，比如说给定一个最大并发量为6，一次性只能并发6个，也就是一次性只能有6个消费者同时进行消费，那么咱们该怎么实现这个操作呢？

一般人都会说，咱们来个池子，随用随取。但是Golang这么搞就没必要了，显得很没有Go style。

首先要明确我们的需求，我们的最终目标是，限制并发数，避免过度并发。

说一下我的解决方案

首先定义一个有长度限定的channel

```golang
var jobs = make(chan string, 6)
```
这里将会存储6个work

再写一个消费逻辑

```golang
for url := range jobs {
	do_somethingss(url)
}
```
这时就大概完成了基础的消费逻辑，那么，我们该如何往这个空channel里传数据呢？

```golang
for _, url := range worklist {
		jobs <- url
		fmt.Println("add", url)
	}
```

现在我们有了消费者，也有了生产者，那么我们把代码封装一下

详细代码如下

```golang
func do_somethingss(url string) (err error) {
	fmt.Println("hello:" + url)
	if url == "string2" {
		time.Sleep(time.Second * 10)
		fmt.Println("all die")
		return nil
	}
	time.Sleep(time.Second * 2)
	return nil
}

func main() {
	worklist := []string{"string1", "string2", "string3", "string4", "string5", "string6", "string7"}

	wg := sync.WaitGroup{}
	var jobs = make(chan string, 6)
	//最大6个消费者
	for i := 0; i < 6; i++ {
		go func() {
			// 消费数据
			for url := range jobs {
				do_somethingss(url)
				//消费完通知，移除一个已经消费的值
				wg.Done()
			}
		}()
	}

	// 将worklist里的数据上传到jobs当中
	for _, url := range worklist {
		jobs <- url
		wg.Add(1)
		fmt.Println("add", url)
	}
	//增加等待，避免退出
	wg.Wait()
}
```

其实很简单，我们捋一下逻辑

1. 创建一个有长度限制的空channel
2. 空channel是一个消费者队列
3. 创建一个生产者逻辑，生产者就是往channel传递数据
4. 遍历channel进行消费
5. 增加wait等待，避免没消费完自动退出




## 结尾
这一节给出了几个基础并发模型，讲了一下Golang并发的哲学思维，这里其实是做了一个笔记，可以持续性发散思维解决问题。下一节我会实现一个分布式并发框架，都会用到这里面所有的东西。over。

并发循环这套东西，说多不多，说少不少，重头戏Context我放到下一节去写了，因为我认为那个还是需要花点功夫的，最近是年底了，学习还是不能停下来，向前吧老铁们！