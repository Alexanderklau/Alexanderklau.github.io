---
title: 使用Golang的gRPC框架的一点随想
date: 2020-07-07 13:32:21
tags: ["前后端","其他技术"]
---
## 前言（写文章的原因）
最近开发项目太重了，我有些感觉自己状态不太对，想要去换一个地方生活，但是说到要做的事就一定要做，学习是不能停下来的。

最近开发使用了一下gRPC，不同于以前我自己写的Python 的 RPC，gRPC相对来说用起来更简单，写起来更容易，可能一开始有点不好使，其实熟悉了也还是不错的，那么，这篇文章主要就写一下，gRPC的基础使用和在我项目中的实际应用吧。

## RPC框架是什么
简单说下什么是RPC，RPC是什么，全称是Remote Procedure Call Protocol，中文名称是远程调用协议，这个就说的很明白了

RPC干嘛的，远程调用的呗！

远程调什么？参数呗！函数呗！发消息呗！

大白话来说，像调用本地服务一样调用远程服务，就是你现在要完成一个动作，你需要一个老哥（机器）和你联动一起完成，所以你就要通过一个渠道告诉他，说，老铁，我们要一起做这个东西，知道吗。

这个渠道，就叫RPC，一般我们在网络服务，分布式服务，微服务中都需要这个东西。

从技术的角度上来说，RPC更类似于一种client，server的通信过程，client发送消息给server，server接收到消息，然后返回消息给client。

![](https://blog-1256169066.cos.ap-chengdu.myqcloud.com/%E4%BD%BF%E7%94%A8Golang%E7%9A%84gRPC%E6%A1%86%E6%9E%B6%E7%9A%84%E4%B8%80%E7%82%B9%E9%9A%8F%E6%83%B3/v2-21431b51eaf16789cb8be2db812cfe15_r.jpg)

## gRPC是什么
我是在写Golang网络服务的时候发现了这个框架gRPC，一看就是谷歌亲儿子框架，粗看了一下，感觉其实还好，所以我仔细看了一下这个框架

1. 首先这货需要一个ProtoBuf序列化工具，是给客户端提供接口的，ProtoBuf这个东西需要另外安装，没他就不能解析接口，这个的好处是，你的开发语言不受这个影响，相当于就是翻译器，能把多种语言编写的接口api翻译出来，给gRPC去用
2. 第二就是需要gRPC的本体，这个如果你有Golang环境直接go get 即可
3. 除此之外，粗看一下gRPC，其实也需要client和server端，也是互相交互的。

### gRPC主要的好处
这货的好处

1. 其实说白了，就是性能高，从Golang的开发来说，就表明了gRPC走的也是高性能的逻辑。
2. protobuf这东西，也是gRPC钦定的接口编写工具，这东西可以严格的约束住接口，告诉他，你可以传什么，不能传什么，加强了gRPC的安全机制
3. protobuf可以压缩编码为二进制，二进制你们懂吧，这个东西就更快了，耍起来那就是大杀器，首先相比大刀，他就是手枪，不仅小，威力还大，还可以达到kill 人的目的，因为压缩成二进制，传递的数据量也小了，通过基础的http2还可以实现异步请求，这就很nice了。
   
## 来玩玩gRPC吧

### 安装protobuf

首先你要先去protobuf的大本营 **https://github.com/protocolbuffers/protobuf/releases**，先找对应的版本下载

我这里是Centos7 64位， 所以我选了[protoc-3.12.3-linux-x86_64.zip](https://github-production-release-asset-2e65be.s3.amazonaws.com/23357588/f7d68880-a4fb-11ea-9ab1-f9b12ce00cda?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Credential=AKIAIWNJYAX4CSVEH53A%2F20200707%2Fus-east-1%2Fs3%2Faws4_request&X-Amz-Date=20200707T065558Z&X-Amz-Expires=300&X-Amz-Signature=4f4d0b1cc59536cb85626624f3c1380e23d9efab2073e23c564715a351c3908b&X-Amz-SignedHeaders=host&actor_id=14084047&repo_id=23357588&response-content-disposition=attachment%3B%20filename%3Dprotoc-3.12.3-linux-x86_64.zip&response-content-type=application%2Foctet-stream) 下载，你们要去找对应的版本下载。

然后你需要解压 
```bash
unzip protoc-3.12.3-linux-x86_64.zip
```

CD到目录, 进到目录的bin下，给protoc赋上可执行的权限
```bash
cd protoc-3.12.3-linux-x86_64/bin
chmod 777 protoc
```

这下protoc可用了，你应该把它移动到bin下，方便未来调用
```bash
cp protoc /usr/bin
```
### 定义一个protobuf接口
重头戏来了，现在我们从零开始开发了，现在我们先创建一个项目文件夹hellogRPC,这下面现在什么也没有。

看图

![](https://blog-1256169066.cos.ap-chengdu.myqcloud.com/%E4%BD%BF%E7%94%A8Golang%E7%9A%84gRPC%E6%A1%86%E6%9E%B6%E7%9A%84%E4%B8%80%E7%82%B9%E9%9A%8F%E6%83%B3/1594106006(1).jpg)

现在我们创一个名字叫manage的文件夹，在下面创建一个名为manage.proto的接口定义文件，类似这样

看图

![](https://blog-1256169066.cos.ap-chengdu.myqcloud.com/%E4%BD%BF%E7%94%A8Golang%E7%9A%84gRPC%E6%A1%86%E6%9E%B6%E7%9A%84%E4%B8%80%E7%82%B9%E9%9A%8F%E6%83%B3/1594106339(1).png)


现在开始编写proto文件

```protobuf
syntax = "proto3";

option objc_class_prefix = "HLW";

package manage;

//定义服务
service GreeterServer {
    // 定义一个具体的接口，定义输出和输出的样式
    rpc SayHello (HelloRequest) returns (HelloReply) {}
}

// 定义输入 string
message HelloRequest {
    string name = 1;
}

// 定义输出 string
message HelloReply {
    string message = 1;
}

```
这里具体的意思就是，我，下一道圣旨，以后，都按着这个给我输入输出，现在我只开了一个sayhello的接口，别的进不来，就这样。

### 编译protobuf文件
这一步是要把protobuf生成为二进制文件，就像我们前面说的，这也是人家gRPC的一个有点，就是有点儿操蛋，因为麻烦。。。。

```golang
go get -u github.com/golang/protobuf/protoc-gen-go
```

下下来之后，你还得把他移动一下下，到你的go path下面，我这里是/root/go
```bash
cd /root/go/bin
cp -r  protoc-gen-go /usr/bin
```
再编译一下，走一波编译

```bash
protoc --go_out=plugins=grpc:. *.proto
```

这下你可以看见，多出一个同名得pb.go文件，类似这样

![](https://blog-1256169066.cos.ap-chengdu.myqcloud.com/%E4%BD%BF%E7%94%A8Golang%E7%9A%84gRPC%E6%A1%86%E6%9E%B6%E7%9A%84%E4%B8%80%E7%82%B9%E9%9A%8F%E6%83%B3/PX%5B2DARE%5B%60FGPROJD2%401RF4.png)


这就编译成功了。下一步继续干活儿

### 编写client服务
client相当于发号施令的大哥，主要就是去找server，告诉server，你们要干嘛，要干嘛这种。

咱们现在创建一个client.go文件

现在开写

```golang
package main

import (
	"log"

	pb "/你的目录/hellogRPC/manage"

	"golang.org/x/net/context"
	"google.golang.org/grpc"
)

//HelloWork 定义一个hello world 方法
func HelloWork(ip string, name string) {
	conn, err := grpc.Dial(ip, grpc.WithInsecure())
	if err != nil {
		log.Fatalf("did not connect: %v", err)
	}
	defer conn.Close()
	c := pb.NewGreeterClient(conn)
	r, err := c.SayHello(context.Background(), &pb.HelloRequest{Name: name})
	if err != nil {
		log.Fatalf("could not greet: %v", err)
	}
	log.Printf("Greeting: %s", r.Message)
}

func main() {
    //传输要发送的ip，传输name
	HelloWork("127.0.0.1:4321", "work")
}
```

### 编写server服务
server服务就类似于等待发号施令的小弟，他们会常驻在后台等待，并且会开启一个专用端口去接受client发来的信息，类似这样

```golang
package main

import (
	"log"
	"net"

	pb "/你的目录/hellogRPC/manage"

	"golang.org/x/net/context"
	"google.golang.org/grpc"
)

const (
    //定义一个端口
	port = ":4321"
)

// 定义server
type server struct{}

//定义SayHello 记住，这里的SayHello要和你proto里面的接口名一致
func (s *server) SayHello(ctx context.Context, in *pb.HelloRequest) (*pb.HelloReply, error) {
	return &pb.HelloReply{Message: "Hello " + in.Name}, nil
}

func main() {
	lis, err := net.Listen("tcp", port)
	if err != nil {
		log.Fatalf("failed to listen: %v", err)
	}
	s := grpc.NewServer()
	pb.RegisterGreeterServer(s, &server{})
	s.Serve(lis)
}
```

### 来跑一下

先启动 server.go

```golang
go run server.go
```

再启动 client.go

```golang
go run clinet.go
```

发现输出了

```bash
2020/07/07 16:01:10 Greeting: Hello work
```

这时我们的目录结构如下，基础的gRPC服务也就此完成。

![](https://blog-1256169066.cos.ap-chengdu.myqcloud.com/%E4%BD%BF%E7%94%A8Golang%E7%9A%84gRPC%E6%A1%86%E6%9E%B6%E7%9A%84%E4%B8%80%E7%82%B9%E9%9A%8F%E6%83%B3/1594108929(1).jpg)


## 总结

其实这个还是比较简单的，但是具体到后面使用，你可能会遇到，gRPC和ETCD冲突，编译的时候出现bug，编译的时候编译器版本对不上等等等等，我这次踩了不少坑，不过还好过来了，项目快要结束了，终于要结束了，我想我需要休息一下，然后去找一个适合我的job。

预告一下，未来我将进行kail linux 和渗透测试的文章更新，基础的技术我应该不会再更了，毕竟我15年入行，16年正式上班，是该有一些蜕变了，未来我将持续输出高质量的文章，请多多期待吧。