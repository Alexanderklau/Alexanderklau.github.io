---
categories: ["技术", "Golang"]
title: Golang中WaitGroup使用的一点随想
date: 2020-05-26 13:29:11
tags: ["前后端"]
---

# Golang中WaitGroup使用的一点随想

## 前言（为什么又要写一篇随想文）
上次我写了一个channel的文章，我寻思，这Golang控制三大巨头，channel，waitgroup，context，我得尽快都安排上，最近工作太忙，压力过大，但是Update Blog还是不能够停下来，所以继续补上，学习还是不能停，那么来吧。

## WaitGroup的简单用法(等待组)
你品一下人家这名字，等待组。等待什么，等待goroutine完成啊。有些时候，我们启动多个goroutine去执行任务，我举个例子  

```golang
listip := []string{"10.0.9.11","10.0.9.22","10.0.9.33"}

for _, ip := range(listip) {
    //假设我们执行一个ping ip 的逻辑
    go PingIPWork(ip)
}

```

我这里执行了一个多ip去ping的逻辑，一般这种时候，你要是执行一波，人家肯定毛都不会返回给你，为什么？因为人家主线程直接就退出了，还是那句话，你又没告诉人家主线程要等这ip全部都ping 完，所以你必须要加个等待，等着Goroutine完成，这里我再举一个网上的例子  

```golang
package main

import (
    "fmt"
)

func main() {
    go func() {
        fmt.Println("Goroutine 1")
    }()

    go func() {
        fmt.Println("Goroutine 2")
    }()

    //来个睡眠，等Goroutine结束
    time.Sleep(time.Second * 1)
}
```

看到了么，加了一个sleep，用sleep去等着Goroutine跑完，上面我举的那个例子也可以这么来

```golang
listip := []string{"10.0.9.11","10.0.9.22","10.0.9.33"}

for _, ip := range(listip) {
    //假设我们执行一个ping ip 的逻辑
    go PingIPWork(ip)
}

time.Sleep(time.Second * 1)
```

加个sleep可以等待完成，但是万一啊，Goroutine有的跑的快，有的慢，你那sleep就一秒，要是有的Goroutine没跑完不就白瞎了吗，所以咱们需要一个机制，这个机制可以帮助咱们去管理Goroutine，让我们知道Goroutine这东西什么时候停，什么时候完成。所以，WaitGroup这个东西，就可以帮助我们解决这个问题，还是老样子，我举一个简单的例子来说明我的想法。

```golang
package main

import (
    "fmt"
	"sync"
)


func PingIPWork(ip string) {
	fmt.Println(ip)
}


func main() {

    //定义一个等待阿祖
	var wg sync.WaitGroup


	wg.Add(3) // 因为有3个Ip，咱们定义三个动作，所以来三个计数

	listip := []string{"10.0.9.11","10.0.9.22","10.0.9.33"}

	for _, ip := range(listip) {
		//假设我们执行一个ping ip 的逻辑
		go func(ip string) {
            //执行一个work
            PingIPWork(ip)
            //操作完成之后，done一个计数，也就是3-1
			wg.Done()
		}(ip)
 }
    //等待
	wg.Wait() // 等待，直到计数为0
}
```

这里我举了一个简单的例子，其实wg的用法较为简单，在这个例子里面我们用到了

```golang
wg.wait
等待Goroutine结束之后退出主进程

wg.Add
添加Goroutine，其实你可以把它想成，可添加的最大Goroutine数

wg.Done
想象成销毁参数，当Goroutine结束之后调用，意思就是，你没了，我减1
```

## WaitGroup的其他注意事项

### 将Wg作为参数进行传递的时候，需要使用指针
有些时候，咱们不想写的这么麻烦，就寻思怎么才能简单一点，或者可变性稍微强一点，有些时候我们要把wg最为参数，在函数内部调用，我们该怎么写呢？

```golang
package main

import (
	"fmt"
	"sync"
)


func PingIPWork(ip string, wg *sync.WaitGroup) {
	fmt.Println(ip)
	wg.Done()
}


func main() {

	var wg sync.WaitGroup


	wg.Add(3) // 因为有两个动作，所以增加2个计数

	listip := []string{"10.0.9.11","10.0.9.22","10.0.9.33"}

	for _, ip := range(listip) {
		//假设我们执行一个ping ip 的逻辑
		go PingIPWork(ip, &wg)
		}

	wg.Wait() // 等待，直到计数为0
}
```

看到了么，如果你把Wg作为参数进行传递，你得要用指针的形式传值，否则就会死锁！！！！！！！！


### Wg.Add的数值不能为负

wg.Add()的数值必须为正数，如果为负数，将会抛出异常。

```golang
panic: sync: negative WaitGroup counter

goroutine 1 [running]:
sync.(*WaitGroup).Add(0xc042008230, 0xffffffffffffff9c)
    D:/Go/src/sync/waitgroup.go:75 +0x1d0
main.main()
    D:/code/go/src/test-src/2-Package/sync/waitgroup/main.go:10 +0x54
```