---
categories: ["技术", "Golang"]
title: Golang 完成一个 Crontab定时器（2）
date: 2020-03-23 09:14:40
tags: ["前后端"]
---
## 前言
上篇文章，大概讲了一下robfig/cron 包的使用，怎么开始一个定时任务，那个东西比较简单，也就是调用函数而已，人家都给你把包都封装好了。鉴于上一章我没提到cron相关，这一章专门我写个cron相关，讲讲怎么cron语法，然后再实现一个自动生成cron语句的逻辑。
## 需求分析
1. cron的基础科普
2. 根据时间自动生成可用的cron语句

## Cron表达式的基础
Go的Cron和linux的Cron的区别就是，linux只到分钟，但是Go的Cron可以通过我上一节描述的代码设置精确到秒。所以一般的Cron表达式就是 

```shell
* * * * * * command
```

可以看出来，这是一个时间集合，但是其中每个 * 代表什么含义呢？下面给出Golang的cron设置表  

| 字段 | 需要的值 | 字符表示 |
| ---- | -------- | -------- |
| 秒   | 0-59     | * / , -  |
| 分   | 0-59     | * / , -  |
| 时   | 0-23     | * / , -  |
| 日   | 1-31     | * / , -  |
| 月   | 1-12     | * / , -  |
| 星期 | 0-6      | * / , -  |
  
下面举几个cron的具体例子
**每秒执行一次任务**

```shell
* * * * * * Command
```

**每分钟执行一次任务**

```shell
* *／1 * * * * Command
```

**每天12点执行一次任务**

```shell
* 0 12 * * * Command
```

**每个月1号12点执行一次任务**

```shell
* 0 12 1 * * Command
```

**2月14号12点执行一次任务（执行一次）**

```shell
0 0 12 14 2 * Command
```

**每周二12点执行一次任务**

```shell
* 0 12 * * 1 Command
```
## Golang 实现一个Cron表达式自动生成器
Cron这个东西，其实没那么难，但是你每次让我们徒手撸，还是会有点烦，特别是现在网上基本没有在线自动生成Cron语法的网站了，所以我们还是站撸一个Cron自动生成器，首先咱们要明确一个重要东西,**任务可能是循环的，也可能是只执行一次的**，看到了么，这下我们就要针对不同的任务类型，输出不同的任务表达式。
### 规定输入的时间格式
首先输入的时间有多种多样，我们没办法控制输入的时间表达，所以我在这里先行规定，我的代码也是按照这个规定来的，前提在此。  
#### 循环执行的任务
对于循环执行的任务，可能有每月，每周，每日，每时等等，所以我在这里举例   

**每月3号12点执行**
```
m,03,12:00
```

**每周三的12点执行**
```
w,3,12:00
```

**每天的12点执行**
```
d,12:00
```
观察一下，聪明的你应该知道我要做什么，拿每天循环执行来举例
```go
timelists := strings.Split(times, ",")
hours := strings.Split(timelists[1], ":")[0]
minutes := strings.Split(timelists[1], ":")[1]
crontab := fmt.Sprintf("* %s %s * * *", minutes, hours)
fmt.Println(crontab)
```

结合其他部分
```go
timelists := strings.Split(times, ",")
// 在这里判断类型，天，月，周
if timelists[0] == "d" {
    hours := strings.Split(timelists[1], ":")[0]
    minutes := strings.Split(timelists[1], ":")[1]
    crontab := fmt.Sprintf("* %s %s * * *", minutes, hours)
    return crontab
} else if timelists[0] == "w" {
    days := strings.Split(timelists[1], ",")[0]
    hours := strings.Split(strings.Split(timelists[2], ",")[0], ":")[0]
    minutes := strings.Split(strings.Split(timelists[2], ",")[0], ":")[1]
    crontab := fmt.Sprintf("* %s %s * * %s", minutes, hours, days)
    return crontab
} else if timelists[0] == "m" {
    days := strings.Split(timelists[1], ",")[0]
    hours := strings.Split(strings.Split(timelists[2], ",")[0], ":")[0]
    minutes := strings.Split(strings.Split(timelists[2], ",")[0], ":")[1]
    crontab := fmt.Sprintf("* %s %s %s * *", minutes, hours, days)
    return crontab
} else {
    crontab := "* * * * * *"
    return crontab
}
```

执行一发看看,生成个每月的cron表达式   
```shell
* 0 12 03 * * Command
```

诶，怎么多了个03。。。看起来咱们需要格式化一下，把它转换一下成可用的。

```go
func FkZero(times string) (fmttime string) {
    // 如果第一个值不为0，直接认为是正常的
	if string(times[0]) != "0" {
        return times
    // 判断00的情况
	} else if strings.Split(times, "0")[1] == "" {
		fkzero := "0"
        return fkzero
    // 清理03，为 3
	} else {
		fkzero := strings.Split(times, "0")[1]
		return fkzero
	}
}
```
#### 执行一次的任务
执行一次的任务表达方式  **2020-03-20 12:00**   
处理代码如下
```go
timelists := strings.Split(times, " ")[0]
month := strings.Split(timelists, "-")[1]
day := strings.Split(timelists, "-")[2]
timework := strings.Split(times, " ")[1]
hours := strings.Split(timework, ":")[0]
minutes := strings.Split(timework, ":")[1]
crontab := fmt.Sprintf("0 %s %s %s %s *", minutes, hours, day, month)
return crontab
```
## 总结
其实这个代码主要就是一个strings的split切分，但是涉及到了crontab语言的输出，其实没那么难，也就是麻烦，我把它传到github上了，有需要可以自己get下来。  
https://github.com/Alexanderklau/Go_poject/tree/master/Go-Script/crontab