---
title: Golang语言的一些基础(针对面向基础的笔/面试)
date: 2020-08-25 09:43:13
tags: ["前后端"]
---
## 前言
昨天让无糖信息的面试官嘲讽了之后，我火速写了一篇博客嘲讽回去一波，但是嘲讽归嘲讽，该做的事儿还是要做的，所以昨天晚上花了一些时间总结了一些Golang的基础，作为查漏补缺和自己学习。

还有就是，无糖信息这种公司对待面试者的态度，也决定这个公司的基本格局，反正我是不会去了，如果看到这个blog的人，你们去面无糖信息的Golang还是要用点心，得有个大心脏，因为那个面试官根本就不会care你的感受。

嘿嘿，不过那个面试官，我不知道你的名字，我也不会care你是否能看到，我只希望你以后懂得尊重这个词语的意义。

面试官先生，你应该不会看到我的博客，那我用一些歌词回击你吧，不过看你那个样子，应该不懂音乐（狗头）。

```
自由的灵魂，

从来都不需要拟行程，

我们的目标在云层。
```

## Golang的理论基础

### Golang的保留字段有哪些？

```golang
break default func interface select
case defer go map struct
chan else goto package switch
const fallthrough if range type
continue for import return var
```

### Golang声明变量的方法？

```golang
var a int = 1     //第一种: var variable_name variable_name
value := 1        //第二种: value_name := 1
var b, c, d = 1, 2, 3     //第三种: 合并声明
var(                          //第四种: 合并声明
      value1 int      = 1
      value2 string = "hello world"
)
```

### Golang声明常量的方法？

```golang
const var a int = 1
const var (
        b int = 1
        c string = "hello world"
)
```

### Golang的init函数是什么？
程序运行前的注册功能，理解为Python的init也可以

init函数的特性
```
1. init函数先于main函数自动执行，不能被其他函数调用；
2. init函数没有输入参数、返回值；
3. 每个包可以有多个init函数；
4. 包的每个源文件也可以有多个init函数，这点比较特殊；
5. 同一个包的init执行顺序，golang没有明确定义，编程时要注意程序不要依赖这个执行顺序。
6. 不同包的init函数按照包导入的依赖关系决定执行顺序。
```

### defer是什么？
延迟函数的意思

举个例子

下面的函数返回什么值？
```golang
func test1() (result int) {
	defer func() {
		result++
	}()
	return 0
}
```
此处应该返回一个1

### defer后进先出是怎么表现的？

举个例子

```golang
func main() {
	for i := 0; i < 5; i++ {
		defer fmt.Printf("%d ", i)
	}
}
```
此处会返回，4，3，2，1，0

### Golang的协程是什么？
这个问题很宽泛，无糖信息的那个面试官问了我一个很玄幻的问题：“你协程用的多吗？”

我当时很想问，什么叫我协程用的多，Golang有协程，Python也有，用的多不多你看我项目不就完了，分布式，大规模集群的，有几个没用过协程的，这问题就很可笑。

回到主题，协程是什么，按照golang的语法，创建一个协程只需要

```golang
go example()
```

就可以了。

协程（coroutine）是Go语言中的轻量级线程实现，由Go运行时（runtime）管理。 Golang 的协程本质上其实就是对 IO 事件的封装，并且通过语言级的支持让异步的代码看上去像同步执行的一样。

协程的控制，一般是channel，waitgroup，context之类的，这个我写过文章详细说了，这里就不再描述，如果你们想看，翻一下我以前的blog就可以了。

### struct是怎么用的？或者struct是干嘛的？
这兄弟名字叫结构体，写过Python的话，你理解为一个固定了格式的dict就可以了

使用的方法
```golang
import "fmt"
 
type Example struct {
	Name string
	Age  int
}

 
func main() {
    // 声明
	e := Example{}
	fmt.Println(t)
	t.Name = "back"
	t.Age = 10
	fmt.Println(t)
}
 
返回的值:

{ 0}
{back 10}
```

### interface是什么？
有些东西名义上是接口，其实啥都能干

interface是一种值，它可以像是值一样传递。并且在它的底层，它其实是一个值和类型的元组，interface是一种万能数据类型，它可以接收任何类型的值。

举个例子

```golang
var a1 interface{} = 1
var a2 interface{} = "abc"
list := make([]interface{}, )
list = append(list, a1)
list = append(list, a2)
fmt.Println(list)
```

判断interface的值的类型

```golang
switch v := i.(type) {
case int:
    fmt.Println("int")
case string:
    fmt.Println("string")
}
```

interface是一个nil，你理解为Python里面的nil就可以了。

nil也可以调用interfece

### panic是干嘛的？

类似 try-catch-finally 中的 finally，接收一个interface{}类型的值（也就是任何值了）作为参数，Golang没有try catch，所以panic会直接挂掉程序，如果panic中有defer，那么将先会执行defer，然后，再次抛出panic错误，打印堆栈。

### 多个defer的执行顺序

```golang
func c() (i int) {
    defer func() { i++ }()
    return 1
}
```

这时候应该返回 2

### 一个Go project是如何管理的？
很简单，go mod，

创建一个新的工程

```golang
// 初始化go mod
go mod init
// 下载依赖包/源码包
go mod vendor
```

vendor是一个源码包的集合。这个过于基础了，实在没想到这种题都能考。

### slice切片的基础操作

把切片理解为Python中的list，这样是不是就简单多了

声明空切片

```golang
var sliceTmp []int  
```

初始化一个切片

```golang
sliceTmp2 := []string{"a","b","c"}  
```

修改一个切片的值

```golang
sliceTmp2[0] = "b" 
```

追加切片值

```golang
sliceTmp2 = append(sliceTmp2,"d")
```

截取切片段
```golang
var sliceTmp5 = sliceTmp4[1:4]
```

遍历切片

```golang
for index, value :=range s1{
  doSomething
}
```

### make是干嘛的？
简单地说，make就是告诉计算机，我要申请内存了，你给我腾地儿。

因为我们对于引用类型的变量，不光要声明它，还要为它分配内容空间

make用于内存分配，但只用于通道chan、映射map以及切片slice的内存创建。

举个例子

```golang
// 创建一个指定长度的切片
mySlice1 := make([]int, 5)

//创建一个初始元素长度为5的数组切片，元素初始值为0，并预留10个元素的存储空间： 
mySlice2 := make([]int, 5, 10) 

//创建了一个键类型为string、值类型为PersonInfo
myMap = make(map[string] PersonInfo) 

//也可以选择是否在创建时指定该map的初始存储能力，创建了一个初始存储能力为100的map.
myMap = make(map[string] PersonInfo, 100) 

//创建并初始化map的代码.
myMap = map[string] PersonInfo{ 
  "1234": PersonInfo{"1", "Jack", "Room 101,..."}, 
} 

//创建有缓存通道
ch := make(chan int, 10)
//创建无缓存通道
ch := make(chan int)

```

### Golang中Json忽略字段
有些时候有些字段我们要忽略掉，这里要在定义struct的时候做文章，这边采用了定义nil的手段

```golang
type astruct struct {
    A   string
    B   string
    C   interface{}
}

func main() {
    var a = astruct{
        A: "hello",
        b: "hello",
        C: nil
    }
}

定义nil直接忽略值就可。
```

### map的一般用法

想成Python中的dict，不同的是这东西需要你提前定义

map的常见操作有：声明、赋值、添加、删除、查询、遍历、清空等

```golang
varstuMapmap[int]string  //声明 var map名称 map[键类型]值类型
mapScore:=make(map[string]float32) //或者这样声明
​
stuMap=map[int]string{1001:"Tom",1002:"Tim"} //赋值
stuMap[1003] ="Tem"       //添加
delete(stuMap, 1003)        //删除
​
data, flag:=stuMap[1003] //查询该数据是否存在，不存在时flag为false;存在时  
                                   //data存储数据，flag为true
​
forkey, data:=rangestuMap{ //遍历键和值
   fmt.Println(key, data)
   delete(stuMap, key)        //循环删除清空
}
stuMap=make(map[int]string)  //或者重新make新的空间以清空stuMap，推荐方法
```
map的常见方法有：键值存在性、排序、嵌套
```golang

data, flag:=stuMap[1003] //判断存在性，查询该数据是否存在，不存在时flag为false;存在时  
                                   //data存储数据，flag为true
​
import"sort"                 //利用sort包完成排序功能
varsortSlice[]int            //定义sortSlice切片
   forkey, _:=rangestuMap{
   sortSlice=append(sortSlice, key) //合成切片
} 
sort.Ints(sortSlice)
varstuMap2map[int](map[int]string) //嵌套，即值类型可以嵌套其他类型
```

### Golang中的goroutine并发执行有什么规律？
首先，并发，不是并行

如果设置了
```golang
runtime.GOMAXPROCS(n) 
```
这个东西相当于告诉程序，此刻你只能有n个协程去并行，那个n相当于就是一个控制因素

```golang
import (
	"fmt"
	"time"
)
 
func ready(w string, sec int64) {
	time.Sleep(time.Duration(sec * 1e9))
	fmt.Println(w, "is ready!")
}
func main() {
	go ready("Tee", 2)
	go ready("Coffee", 1)
	fmt.Println("I'm waiting")
	time.Sleep(5 * 1e9)
}
 
结果：
I'm waiting
Coffee is ready!
Tee is ready!
```

## 结尾
细细一看写了不少了，如果能帮到谁，那我真的非常开心，希望大家的技术都能越来越好。

也希望有些面试官真的，认清自己的能力，你的确有地方可以，但是，下场比划比划，你也有不是个数的地方，所以，对技术抱有敬畏之心，对未知抱有好奇之心，对他人抱有尊重之心，才是能走的更远的本质，这个我相信您一定能懂，不过我也不希望你懂，因为就您这态度，我希望你继续保持眼高于顶的态度。因为我希望你35岁被裁员的时候也会记得你有一天这么对待过别人。

嘿嘿，最后用歌词结束这两天不开心的心情吧。

```golang
让时间去给看法

错的会被正义斩杀

你知道天堂很美

孩子们千万别喊害怕

你想要的东西

必须用全力去争取

金钱和权力

才不是你想要的生活真谛

check~
```