---
title: 开发中常用的Golang高级用法
date: 2020-05-22 14:37:09
categories: ["技术", "Golang"]
tags: ["前后端"]
---
# 开发中常用的Golang高级用法

## 前言

忙碌了两个月，这次开发终于要结束了，今天下午公司在重组集群机器，也没办法干活儿了，就写一些东西，相当于，留住一些东西，来纪念这辛苦的两个月吧。做一个纪念，也是为了方便以后自己去查看。在这次开发中，学习了不少Golang的高级特性，并且付诸于实现，也踩了不少坑，留下这篇文字，也是方便其他人能够查看，或者借鉴，如果帮到你，那么我也会很开心你。

## 开发常遇到的问题

### Golang判断一个元素在不在切片/列表当中
在Python中，我们可以直接用 in 的方式去判断，例如

```python
if "i" in lists
```

但是在Golang中，没有这种语法糖或者是关键字可以帮助我们处理这种问题，所以还是只能靠循环去处理这种问题，为此我封装了一个Golang函数，函数如下  

```golang
//FindType 循环对比，匹配到返回true，不匹配返回false
func FindType(a string, typelist []string) bool {
	for _, b := range typelist {
		if b == a {
			return true
		}
	}
	return false
}
```

### Golang获取文件的详细信息

Golang获取文件信息的方法相对来说容易一些，都已经有了对应的package，我这里只是把怎么用展示出来  

```golang
//timespecToTime 转换
func timespecToTime(ts syscall.Timespec) time.Time {
	return time.Unix(int64(ts.Sec), int64(ts.Nsec))
}

//GetFileinfo 获取文件信息
func GetFileinfo(path string) {
	fileInfo, err := os.Stat(path)
	if err != nil {
		return 0
    }
    //文件大小
    filesize := fileInfo.Size()
    //文件创建时间
    stat_ts := fileInfo.Sys().(*syscall.Stat_t)
    Ctime := timespecToTime(stat_ts.Ctim).Format("2006/01/02")
    //文件修改时间
    Mtime := timespecToTime(stat_ts.Mtim).Format("2006/01/02")
    //文件访问时间
    Attim := timespecToTime(stat_ts.Atim).Format("2006/01/02")
    //获取文件所有者
    stat_ts := fileInfo.Sys().(*syscall.Stat_t)
	uid := strconv.Itoa(int(stat_ts.Uid))
    usrs, err := user.LookupId(string(uid))
    username := usrs.Username
    //获取文件名
    filename := fileInfo.Name()
}
```

### Golang比较两个list/切片的不同之处（差集）

在开发中，我需要比较两个list，然后取出他们之中不同的部分，这里Golang也没有合适的法子，一般的方法就是转map进行处理

```golang
//difference 进行比对，输出不同的[]string
func difference(slice1, slice2 []string) []string {
	m := make(map[string]int)
	nn := make([]string, 0)
	inter := intersect(slice1, slice2)
	for _, v := range inter {
		m[v]++
	}

	for _, value := range slice1 {
		times, _ := m[value]
		if times == 0 {
			nn = append(nn, value)
		}
	}
	return nn
}

//intersect 把两个对比的列表进行map化处理
func intersect(slice1, slice2 []string) []string {
	m := make(map[string]int)
	nn := make([]string, 0)
	for _, v := range slice1 {
		m[v]++
	}

	for _, v := range slice2 {
		times, _ := m[v]
		if times == 1 {
			nn = append(nn, v)
		}
	}
	return nn
}

func main() {
    infinode := []string{"10.0.9.1","10.0.9.2"}
    syncnodes := []string{"10.0.9.1","10.0.9.2", "10.0.9.3"}
    ips := difference(infinode, syncnodes)
}
```

### Golang格式化时间

golang 一般都可以通过Format的方法进行时间格式化，一般你可以直接调format，例如

```golang
fmt.Println(time.Now().Format("2006-01-02 15:04:05"))
```

有些时候是他娘的时间戳  

```golang
fmt.Println(time.Unix(1389028339, 0).Format("2006-01-02 15:04:05"))
```

有些时候会让你搞成别的样子，比如"2006/01/02 15:04:05"

```golang
fmt.Println(time.Now().Format("2006/01/02 15:04:05"))
```

有些时候会让你比较个时间大小，例如

```golang
//比对时间，看看它在不在开始时间/结束时间的范围内
starttime := "2020-05-12"
endtime := "2020-05-20"
ctime := "2020-05-15"
ft, err := time.Parse("2006-01-02", ctime)
st, err := time.Parse("2006-01-02", starttime)
et, err := time.Parse("2006-01-02", endtime)
if ft.After(et) && ft.Before(st) {
    fmt.Println("在里面儿！")
} else {
     fmt.Println("不在里面儿！")
}
```

### Golang并发循环的使用

我们处理循环的时候，在Python中，我举个例子  

```python
lists = [1, 2, 3, 4]
for i in lists:
    print i
```

这里打印的话是 

```python
1
2
3
4
```

Golang有个黑科技是并发循环，简单说，就是能一下给你把这四个都给你打印出来，不是一条条循环，也不用等上一个循环的结果返回，也可以选择略过错误。我在这里简单实现一个

#### 直接并发循环（无需考虑错误）

这里不考虑error的情况，具体代码如下

```golang
ips := []string{"10.0.9.1","10.0.9.2"}
ch := make(chan struct{})
//并发循环ip
for _, ip := range ips {
    go func(ip string) {
        //模拟一个展示ip
        fmt.Println(ip)
        ch <- struct{}{}
    }(ip)
}
for range ips {
    <-ch
}
```

#### 输出错误的并发循环(考虑错误)

```golang
ips := []string{"10.0.9.1","10.0.9.2"}
//定义一个error
errors := make(chan error)
for _, ip := range ips {
    go func(ip string) {
        _, err := test(ip)
        errors <- err
    }(ip)
}
for range ips {
    if err := <-errors; err != nil {
        return err
    }
}
```

### Golang做一个不退出的无限循环

有些时候我们希望一个服务/脚本/函数不断运行，或者每隔一段时间来一发（运行一次），这时候我们就需要定义无限循环

```golang
func main() {
    for {
        //每隔1s打印一次start work
        time.Sleep(time.Second * 1)
		fmt.Println("Start work......")
	}
}
```

### Golang实现一个遇到错误的重试机制

当我们最开发的时候，有些时候遇到错误，需要进行重试，这时候我们就需要进行错误捕获和函数重载

```golang
//Retry 重试逻辑
//传入函数，如果捕获到错误，则重载函数，重新来一次执行，直到没错误为止
func Retry(fn func() error) error {
	if err := fn(); err != nil {
		if s, ok := err.(stop); ok {
			return s.error
		}
		return Retry(fn)
	}
	return nil
}
//TestWork 测试函数，无意义
func TestWork() error {
    //测试函数，这里是伪代码。
    _, err := GetIpNode()
    if err != nil {
        return err
    }
    return nil
}
func main() {
    go Retry(TestWork)
}

```

### Golang执行Linux命令

有些时候我们需要执行linux相关命令，Golang封装了相应的包，但是少部分时候我们需要控制一些很久不返回结果的命令，所以我加了一个超时时间，代码如下


```golang
var (
    //这里我写死了，你可以自己定义，更灵活一些
	Timeout = 60 * time.Second
)
//Command 执行shell
func Command(arg string) ([]byte, error) {
	ctxt, cancel := context.WithTimeout(context.Background(), Timeout)
	defer cancel()

	cmd := exec.CommandContext(ctxt, "/bin/bash", "-c", arg)

	var buf bytes.Buffer
	cmd.Stdout = &buf
	cmd.Stderr = &buf

	if err := cmd.Start(); err != nil {
		return buf.Bytes(), err
	}

	if err := cmd.Wait(); err != nil {
		return buf.Bytes(), err
	}

	return buf.Bytes(), nil
}

func main() {
    _, err := Command("ls")
    if err != nil {
        fmt.Println(err)
    }
}
```

### Golang 简单的日志逻辑

这边贡献一个简单的日志实现代码，往文件里面写，可以自己定义格式，代码如下

```golang
import (
	"fmt"
	"os"

	"github.com/op/go-logging"
)

func Monlog() (logs *logging.Logger) {
    var log = logging.MustGetLogger("monlog")
    //定义格式 时间 go文件 行数 等级 错误信息
	var format = logging.MustStringFormatter(
		`%{time:2006-01-02T15:04:05} %{shortfile} %{shortfunc} %{level} %{message}`)
    //文件的写入位置 追加模式/没有自动创建
	logFile, err := os.OpenFile("/var/log/monlog.log", os.O_WRONLY|os.O_APPEND|os.O_CREATE, 0666)
	if err != nil {
		panic(fmt.Sprintf("Open faile error!"))
    }
    //头相关, 具体意思就是支持追加，然后格式化，且按照那个格式往里面写
	backend1 := logging.NewLogBackend(logFile, "", 0)
	backend2 := logging.NewLogBackend(os.Stderr, "", 0)

	backend2Formatter := logging.NewBackendFormatter(backend2, format)
	backend1Formatter := logging.NewBackendFormatter(backend1, format)
	backend1Leveled := logging.AddModuleLevel(backend1Formatter)
	backend1Leveled.SetLevel(logging.INFO, "")
	logging.SetBackend(backend1Leveled, backend2Formatter)
	return log
}

func main() {
    log := Monlog()
    log.Info("info")
 	log.Notice("notice")
 	log.Warning("warning")
}
```

### Golang 使用 Aws-sdk 获取到指定bucket中全部的item

我前阵子写过一篇文章，是Golang 调用 s3对象存储的，使用指定的api可以获取其中的item，但是人家限定100条，估摸着怕撑爆内存，但是如果我们想要获取到所有的item，这个该怎么做呢？其实循环获取就好了，但是我还是徒手实现一下吧，大家抄抄得了。

```golang
func GetRgwItem3(buckets string) ([]*s3.Object, error) {
    var items []*s3.Object
    sess, err := session.NewSession(&aws.Config{
		Credentials:      credentials.NewStaticCredentials(ak, sk, ""),
		Endpoint:         aws.String(endpoint + ":7480"),
		Region:           aws.String("us-east-1"),
		DisableSSL:       aws.Bool(true),
		S3ForcePathStyle: aws.Bool(false), //virtual-host style方式，不要修改
	})
	if err != nil {
		return nil, err
	}
	svc := s3.New(sess)
	err := svc.ListObjectsPages(&s3.ListObjectsInput{
		Bucket: &buckets,
	}, func(p *s3.ListObjectsOutput, last bool) (shouldContinue bool) {
		for _, obj := range p.Contents {
			items = append(items, obj)
		}
		return false
	})
	if err != nil {
		return items, err
	}
	return items, nil
}
```

我这里没毛病啊，万一你有问题你就咨询我

## 后记
暂时先更这么多，这也不少了，大部分代码我都给出了具体实现，要是还不会就留言给我或者发邮件。   

其实我特想对那些写几个爬虫在那里爬我博客的人说，你偷博客没所谓，复制也没所谓，不加名字说转载就很说不过去了，还TM标
榜你是原创，脸呢？cnblog我会逐渐弃掉，因为我有我自己的博客了，我也在逐渐搞迁移，把我原来写的博客都迁移到我自己的博客上去，未来我的cnblog博客将逐渐少更或者停更，就算更我也会用英语/日语双更，我让你TM抄我东西不说，fuck off bitch。