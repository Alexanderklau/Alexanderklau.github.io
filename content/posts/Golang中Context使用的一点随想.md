---
categories: ["技术", "Golang"]
title: Golang中Context使用的一点随想
date: 2020-06-01 10:18:36
tags: ["前后端"]
---

# Golang中Context使用的一点随想

## 前言 
这一篇是三巨头最后一篇了，前两篇介绍了channel，waitgroup，今天这篇来介绍一下context，相比于其他两种，我倒是更推荐context（上下文）这种控制goroutine的方法，为什呢，下面我就来详细的说一说吧。

## 什么是Context（逻辑上下文）
Context是一种链式的调用逻辑，一般是用来控制goroutine，例如说，控制goroutine的启动，停止，暂停，取消等等。 我这里举一个简单的例子来说明Context怎么用

```golang

//WorkDir 定义一个遍历目录逻辑，假设这个目录很大，但是咱们需要随时把它结束掉
func WorkDir(ctx context.Context, name string) {
    //假装这是一个大列表，很大很大那种
    filePathList := []string{........}
    for _, file := range(filePathList) {
        //假装这里处理一下目录, 打印一下目录
        fmt.Println(file)
        select {
        //检测到ctx发生了变化
        case <-ctx.Done():
            fmt.Println(name,"任务退出，骨灰都给你扬了")
            return
        //否则什么都不做
        default:
        }
    }
}

func main() {
    //定义一个waitcancel
    ctx, cancel := context.WithCancel(context.Background())
    //将ctx作为参数传入函数当中
    go WorkDir(ctx, "三秒之内撒了你")
    //让程序跑个几秒
    time.Sleep(3 * time.Second)
    fmt.Println("该杀任务了")
    //杀任务操作
    cancel()
    //没有监控输出，就表示停止
    time.Sleep(2 * time.Second)
    fmt.Println("任务让我杀了")
}
```

怎么样，是不是很简单，简单来说，ctx就是传递的信号，你给每一层传递的ctx，不管传的再多，都只有一个爹  

```golang
ctx, cancel := context.WithCancel(context.Background())
```

cancel是来通知goroutine结束的，当爹ctx执行了cancel之后，就表示，我死了，你们得和我一起被株连，大家一起完蛋。

**链式**的意思就是，爹ctx是母体，它可以不断的继续下发产生多个子ctx，当cancel通知母体死亡的时候，子ctx也要跟着一起完蛋，所以链式就是，一荣俱荣，一关俱关的，cancel就是丧钟，调用就会释放所有的被感染（ctx下发）的goroutine。

## Context的几个重要方法

### WithCancel方法

WithCancel方法的接口

```golang
func WithCancel(parent Context) (ctx Context, cancel CancelFunc)
```
这里会返回一个ctx和cancel，ctx是一个可以被拷贝的context，这个context可以作为父节点继续向下传递，cancel则是通知ctx退出的信号，调用CancelFunc的时候，关闭c.done()，直接退出goroutine。 举个简单的例子，算了我偷个懒，用上面的例子，ctrl c ctrl v一波

```golang
//WorkDir 定义一个遍历目录逻辑，假设这个目录很大，但是咱们需要随时把它结束掉
func WorkDir(ctx context.Context, name string) {
    //假装这是一个大列表，很大很大那种
    filePathList := []string{........}
    for _, file := range(filePathList) {
        //假装这里处理一下目录, 打印一下目录
        fmt.Println(file)
        select {
        //检测到ctx发生了变化
        case <-ctx.Done():
            fmt.Println(name,"任务退出，骨灰都给你扬了")
            return
        //否则什么都不做
        default:
        }
    }
}

func main() {
    //定义一个waitcancel
    ctx, cancel := context.WithCancel(context.Background())
    //将ctx作为参数传入函数当中
    go WorkDir(ctx, "三秒之内撒了你")
    //让程序跑个几秒
    time.Sleep(3 * time.Second)
    fmt.Println("该杀任务了")
    //杀任务操作
    cancel()
    //没有监控输出，就表示停止
    time.Sleep(2 * time.Second)
    fmt.Println("任务让我杀了")
}
```

### WithDeadline方法

WithDeadline方法的接口

```golang
func WithDeadline(parent Context, deadline time.Time) (Context, CancelFunc)
```
这下机制出现了变化，cancel换成了deadline，参数也换了，换成了time，聪明的你应该想到了什么吧，这个函数其实是设置一个具体的死亡时间（deadline），当到达了指定的deadline时间之后，将所有的goroutine进行退出。咱们简单点儿，告诉你，就是到点儿退出。这里咱们照旧，举个例子，用例子来个详细说明。

```golang
func WorkDir(ctx context.Context, name string) {
    //假装这是一个大列表，很大很大那种
    filePathList := []string{........}
    for _, file := range(filePathList) {
        //假装这里处理一下目录, 打印一下目录
        fmt.Println(file)
        select {
        //检测到ctx发生了变化
        case <-ctx.Done():
            fmt.Println(name,"任务超时了，骨灰都给你扬了")
            return
        case <-time.After(1 * time.Second):
            fmt.Println("我还在......")
        //否则什么都不做
        default:
        }
    }
}

func main() {
    //定义一个WithDeadline，这里定义了一个具体的时间点
	ctx, cancel := context.WithDeadline(context.Background(), time.Now().Add(10 *  time.Second))
    //将ctx作为参数传入函数当中
    go WorkDir(ctx, "从现在开始计时，10s之后撒了你")
    //让程序跑个几秒
    time.Sleep(3 * time.Second)
    fmt.Println("该杀任务了")
    //杀任务操作
    cancel()
    //没有监控输出，就表示停止
    time.Sleep(2 * time.Second)
    fmt.Println("任务让我杀了")
}
```

千万注意，后面还有个方法叫WithTimeout，它们两个说一样也不一样，WithTimeout是设置一个超时时间，WithDeadline是设定一个具体时间点，比如我上面那个，10s之内撒了goroutine，这个较为抽象，后面WithTimeout讲解的时候我会多费点功夫的。

### WithTimeout方法

WithTimeout方法的接口

```golang
func WithTimeout(parent Context, timeout time.Duration) (Context, CancelFunc)
```
对比一下上面那个WithDeadline，看到没，都需要传入一个时间变量，但是还是那句话，上面的WithDeadline需要的是具体时间，下面的WithTimeout就简单多了，就需要一个时间就行了，说简单点，上面的WithDeadline，要的是一个具体时间，例如 “2020-01-01 12:00:00”，
```golang
time.Now().Add(5*time.Second)
```
但是下面的WithTimeout，就很简单，就要一个超时时间就好了，例如5s  

```golang
timeout := 3 * time.Second
```
类似crontab的用法。好了，继续举个例子

```golang
func WorkDir(ctx context.Context, name string) {
    //假装这是一个大列表，很大很大那种
    filePathList := []string{........}
    for _, file := range(filePathList) {
        //假装这里处理一下目录, 打印一下目录
        fmt.Println(file)
        select {
        //检测到ctx发生了变化
        case <-ctx.Done():
            fmt.Println(name,"任务超时了，骨灰都给你扬了")
            return
        case <-time.After(1 * time.Second):
            fmt.Println("我还在......")
        //否则什么都不做
        default:
        }
    }
}

func main() {
    //定义一个WithDeadline，这里定义了一个具体的时间点
	ctx, cancel:= context.WithTimeout(context.Background(), 5 * time.Second)
    //将ctx作为参数传入函数当中
    go WorkDir(ctx, "不管怎么样，没返回，5s之后撒了你")
    //让程序跑个几秒
    time.Sleep(20 * time.Second)
    fmt.Println("该杀任务了")
    //杀任务操作
    cancel()
    //没有监控输出，就表示停止
    time.Sleep(2 * time.Second)
    fmt.Println("任务让我杀了")
}
```
其实这两个用法都差不多，就是一个要求指定时间，一个随机时间，都是可以自己定义的.

### WithValue方法

WithValue方法的接口

```golang
func WithValue(parent Context, key interface{}, val interface{}) (Context)
```

这个看一下人家的传入，一个context，一个key, 一个接口interface，这明显就是一个key value类型的参数，这个方法的主要功能就是传递消息用，传递的元数据一般都是在子ctx中要使用到的，这个函数其实我用的不多，我参考了一下其他老哥的写法，总结了一下，具体的例子如下

```golang
//定义一个key
var key string = "keywork"

func main() {
	ctx, cancel := context.WithCancel(context.Background())
	//附加值
	valueCtx := context.WithValue(ctx, key, "【监控1】")
	go watch(valueCtx)
	time.Sleep(10 * time.Second)
	fmt.Println("可以了，通知监控停止")
	cancel()
	//为了检测监控过是否停止，如果没有监控输出，就表示停止了
	time.Sleep(5 * time.Second)
}

func watch(ctx context.Context) {
	for {
		select {
		case <-ctx.Done():
		//取出值
			fmt.Println(ctx.Value(key), "监控退出，停止了...")
			return
		default:
		//取出值
			fmt.Println(ctx.Value(key), "goroutine监控中...")
			time.Sleep(2 * time.Second)
		}
	}
}
```

## Context的使用注意事项

1. Context 始终要以参数的形式进行传递，整个结构体是不太行的，当然，利用map存储进行设计管理还是可以的。
2. Context 做参数你得永远把它放第一位，无论什么情况都切记
3. Context 可以在多个goroutine中进行传递，无需担心，它是线程安全的。

## 结尾
两天终于把这个系列整完了，其实相当于自己复习了一下，也借鉴了一些别人的代码，value那个转手是在太多，我找不着原作者了，要是谁看到说是你写的，马上邮件我啊，我加上你的名字。

下一步我要开个大坑，我要用Golang 整个大活儿，爬虫之类的我都不想写了，我现在打算用Golang复写我的一些安全工具，或者是写一个Elasticsearch相关的东西，期待吧！