---
categories: ["技术", "Golang"]
title: Golang 完成一个 Crontab定时器（1）
date: 2020-03-23 09:12:36
tags: ["前后端"]
---
## 前言
Linux的Crontab定时器似乎已经足够强大，但是我认为还是没有办法满足我们所有的需求，例如定时器某一瞬间需要动态添加／删除任务的功能，例如定时器只能在指定的节点上启动（主节点），其他节点不需要定时服务，这种情况Linux自带的Crontab就不能够满足我们的需求了，所以这次要徒手定义一个Crontab定时器，作为自己的备用。
## 需求分析
看我博客的基本也都知道，做任何事，都要进行一个需求分析。既然是一个定时器，那么应该支持的功能如下  
1. 定时启动任务（废话）
2. 支持基础的Crontab语法
3. 支持将时间转换为Crontab语法
4. 支持Crontab语法校验
5. 记录日志的功能
6. 支持crontab任务到秒级

综合上面的需求，不难看出，其实最重要的功能就是实现一个定时启动任务的玩意儿，在Go开发当中，也就是在某个时间点执行某个Go函数。说的简单点，就是，定时跑Go函数。也就是定时go func而已。
## robfig/cron 包的使用
### 安装robfig/cron包
我这里默认你们都有go 基础,先安装个包，这玩意儿是驱动Golang 驱动 Crontab的重要框架。   
```bash
go get github.com/robfig/cron
```
### robfig/cron的使用举例
启动一个定时任务其实很简单
```go
job := cron.New()
job.AddFunc("* * * * *", func() {fmt.Println("Start job....")})
job.Start()
```

一个定时任务就这样被写好了，其实仔细琢磨一下，添加任务的方式就是
```go
// 新任务
job := cron.New()
///任务添加
job.AddFunc("Cronta 语句", func() {执行函数（）})
//任务开始
job.Start()
```
这样就完成了一个定时任务的添加
## 支持crontab任务到秒级
估计你们看到这里一脸懵逼，为啥你个单独到秒级的也要整出来呢？其实我也表示，我也不想啊！奈何一点，robfig/cron这玩意有个很奇葩的一点，**它只支持的分钟级别的任务，不支持到秒级别！！！！**，**它只支持的分钟级别的任务，不支持到秒级别！！！！**，**它只支持的分钟级别的任务，不支持到秒级别！！！！**，重要的事儿说三遍！这里需要你自己定义秒级别的任务，在翻了一下它的test源码之后，拉到最底下，看到这么一行代码。
```go
// newWithSeconds returns a Cron with the seconds field enabled.
func newWithSeconds() *Cron {
	return New(WithParser(secondParser), WithChain())
}
```
这行代码啥意思呢，意思就是启用返回seconds字段的任务，说白了就是，你要加这个，才能开启秒级别的任务  
网上好多博客写的，都是你抄我，我抄你，抄来抄去，没一个代码能用的，大家如果发现抄网上那帮人的代码，跑不起来，那绝对就是这个原因！人家只支持到分钟，网上给出的例子全都是秒级别的，并且没打开秒级别任务定义，还能跑起来？我都怀疑你们怎么写的代码。  
**定义秒级别任务代码**  
这段代码主要的意思就是，开放到秒级别的任务支持，看到了second么，这段代码在源码包的test下有，你们可以自己去看看。
```go
func newWithSeconds() *cron.Cron {
	secondParser := cron.NewParser(cron.Second | cron.Minute |
		cron.Hour | cron.Dom | cron.Month | cron.DowOptional | cron.Descriptor)
	return cron.New(cron.WithParser(secondParser), cron.WithChain())
}
func main() {
    // 使用秒级别任务
    job := newWithSeconds()
    // 任务定义,3s输出一个j1 start job,不停循环
    job.AddFunc("0/3 * * * * ? ", func() {
        fmt.Println("j1 start job....", time.Now().Format("2020-03-20 15:04:05"))
    })
    // 任务定义，每分钟的第三秒执行任务
    job.AddFunc("3 * * * * ? ", func() {
        fmt.Println("j2 start job....", time.Now().Format("2020-03-20 15:04:05"))
    })
    //开始任务
    job.Start()
    select {}
}
```

输出的结果
```shell
j1 start job.... 2020-03-21 17:09:36
j1 start job.... 2020-03-21 17:09:39
j1 start job.... 2020-03-21 17:09:42
j1 start job.... 2020-03-21 17:09:45
```

## 总结
初步完成了crontab的基础功能，这篇文章默认，你们都比较懂crontab语法和go开发了，如果不懂就请期待下一篇，实现crontab语法自动生成和自动加载任务吧。