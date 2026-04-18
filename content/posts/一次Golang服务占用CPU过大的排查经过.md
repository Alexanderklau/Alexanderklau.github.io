---
title: 一次Golang服务占用CPU过大的排查经过
date: 2020-08-17 09:51:47
tags: ["前后端"]
---
## 前言
前阵子写了个ETCD选主的代码，持续后台执行，相安无事一阵我就干别的事儿去了，我寻思小爷虽然代码写的一般，但是不至于出错啊。

但是我想的实在是太单纯了，应了那句话，我还是too young啊，高估了自己的姿势水平，一下搞出来一个大新闻。周六大半夜告警在那里 biubiubiu 的往我邮箱里塞，我当时正在COD战场上挥汗如雨，手机的震动就像电动马达一样给我腿都快整麻了，一看邮箱，好家伙，CPU占用百分之200多！Cgroup都没给它限制住。

所以我寻思，咱就好好分析下到底怎么回事儿吧。

## 工具准备
一般验证一个server详细的CPU和内存占用，Python我会选择PDB，C++我会选择GDB，但是GOlang这个就相对来说比较陌生，经过我查询资料（谷歌）过后得知，Golang有神奇pprof，还可以分析整个函数占用，这就很牛逼了，果断学习了一波之后上手。

## pprof的简单使用

首先pprof分为两个大包

``` golang
net/http/pprof 

runtime/pprof
```

如果你的go程序是用http包启动的web服务器，你想查看自己的web服务器的状态。这个时候就可以选择net/http/pprof。你只需要引入包_"net/http/pprof"。

例如

```golang
import _ "net/http/pprof"

go func() {
    http.ListenAndServe("0.0.0.0:8080", nil)
}()
```

我现在要分析我的go server为什么占用了过大的CPU，所以，我需要在我的Go server的main中添加几行导入pprof的代码，一些业务代码我这里会直接隐去，请见谅

```golang
import ( 
    _ "net/http/pprof"
    "github.com/gin-gonic/gin"
    ······
)

func main() {
    // 这里使用了gin框架，模拟我的业务环境
    gin.SetMode(gin.ReleaseMode)
    router := gin.Default()
    //ETCD选主的逻辑，这里很可能是持续占用CPU的罪魁祸首
    go infimanage.Retry(infiapi.InitSyncStart)
    //调用pprof的逻辑，指定6161端口
    go func() {
		http.ListenAndServe(":6161", nil)
    }()
    //这里是原本的服务接口,这里只是举个例子，模拟一波
    ....
    router.POST("/synctask/add", addwork, AddSyncWorks)
    router.GET("/synctask/del", DeleteSyncWorks)
    s := &http.Server{
		Addr:           ":8788",
		Handler:        router,
		ReadTimeout:    10 * time.Second,
		WriteTimeout:   10 * time.Second,
	}
	_ = s.ListenAndServe()
}
```
现在走一波

```golang
go run main.go
```

现在server就启起来了，然后咱们要来一波分析
首先指定监控main server 30s
```go
go tool pprof http://127.0.0.1:6161/debug/pprof/profile -seconds 30
```

会冒出下面一大堆东西，就相当于进入了pprof，类似pdb，gdb的调试shell下
```go
Fetching profile over HTTP from http://localhost:6161/debug/pprof/profile?seconds=30
Saved profile in /Users/eddycjy/pprof/pprof.samples.cpu.007.pb.gz
Type: cpu
Duration: 1mins, Total samples = 26.55s (44.15%)
Entering interactive mode (type "help" for commands, "o" for options)
(pprof) 
```
获取CPU占比前10的函数

```go
(pprof) top10
Showing nodes accounting for 25.92s, 97.63% of 26.55s total
Dropped 85 nodes (cum <= 0.13s)
Showing top 10 nodes out of 21
      flat  flat%   sum%        cum   cum%
    23.28s 87.68% 87.68%     23.29s 87.72%  syscall.Syscall
     0.77s  2.90% 90.58%      21.77s  80.90%  runtime.selectgo
     0.58s  2.18% 92.77%      0.58s  2.18%  runtime.mcall
     0.53s  2.00% 94.76%      1.42s  5.35%  runtime.timerproc
     0.36s  1.36% 96.12%      30.39s  98.47%  go.etcd.io/etcd/clientv3/...
     0.35s  1.32% 97.44%      0.45s  1.69%  runtime.greyobject
     0.02s 0.075% 97.51%     24.96s 94.01%  main.main.func1
     0.01s 0.038% 97.55%     23.91s 90.06%  os.(*File).Write
     0.01s 0.038% 97.59%      0.19s  0.72%  runtime.mallocgc
     0.01s 0.038% 97.63%     23.30s 87.76%  syscall.Write
```

这一波就能看出来了，到底谁才是占用CPU过大的罪魁祸首，看到了么，有个select和etcd的keeplive，但是这样似乎还不那么直观，这时候你需要一个叫做火焰图的东西

## pprof输出火焰图

### 安装输出火焰图的server

安装配置FlameGraph


```bash
git clone https://github.com/brendangregg/FlameGraph.git
```

配置FlameGraph
```
PATH=$PATH:/rootg/go/src/github.com/brendangregg/FlameGraph
```

配置go-torch，这是生成火焰图的必备工具
```
go get -v github.com/uber/go-torch
```

### 生成火焰图

确保都安装完成了之后

直接执行命令

此时你要确保第一步那些监控代码已经写入
```go
go-torch -u http://127.0.0.1:6161/debug/pprof/ -p > cpu-local.svg
```

这里会生成一个svg，这个东西可以用浏览器打开，打开是这样的

![](https://blog-1256169066.cos.ap-chengdu.myqcloud.com/Golang%E7%A8%8B%E5%BA%8FCPU%E5%8D%A0%E7%94%A8%E5%BC%82%E5%B8%B8%E5%88%86%E6%9E%90/1.png)

看到没，这一下就暴露出来了，谁占比多，你还可以点进去看详细，比如

![](https://blog-1256169066.cos.ap-chengdu.myqcloud.com/Golang%E7%A8%8B%E5%BA%8FCPU%E5%8D%A0%E7%94%A8%E5%BC%82%E5%B8%B8%E5%88%86%E6%9E%90/2.png)

这下可以定位了，就是选主逻辑当中的select或者是for循环出了问题

## 解决问题
我们回头去反查ETCD的选主逻辑代码

```golang
//ElectMasterNode 争抢主节点服务
func ElectMasterNode() error {
	ips, err := GetNodeIp()
	if err != nil {
		return err
	}
	etcd, err := infidb.New()
	if err != nil {
		return err
	}
	for {
		_ = etcd.Newleaseslock(ips)
	}
}

//Masterwork 选主尝试
func Masterwork() {
    select {
        ElectMasterNode()
    }
}
```
这里的问题就是，在select中重复调用了for循环，应该是出现了for死循环，导致了CPU持续被抢占，这边修改一下代码逻辑

```golang
//ElectMasterNode 争抢主节点服务
func ElectMasterNode() error {
	ips, err := GetNodeIp()
	if err != nil {
		return err
	}
	etcd, err := infidb.New()
	if err != nil {
		return err
	}
	_ = etcd.Newleaseslock(ips)
}

//Masterwork 选主尝试
func Masterwork() {
    select {
        ElectMasterNode()
    }
}
```

这样问题应该就完全解决了。