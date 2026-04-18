---
title: 写一个高可用Rpc服务（基础知识篇）
date: 2021-08-30 18:30:31
tags: ["前后端"]
---

## 前言

最近实在是太忙啦，每天都加班，这是开博这么久首次，一个月都没UPDATE，my bad，这个月保证日更三篇，补上补上。

久违的freestyle时间，（虽然我现在玩爵士了）

```
每天reload存档

似乎总会是一样

加班就像是吃饭一样平常

内心是否总想这样

看不见未来的远方

迷雾它层层阻挡

思绪飞扬总在寂寞的晚上

让我来静静的品尝
```

## 为什么要写这篇文章

首先是因为我自己，最近非常的忙，但是我又在想，我他喵的忙了个什么

汇报一下最近（这一个月）的战果

1. 从无到有编写完成了一个RPC服务，并且开源给全公司的项目使用
2. 完成了一个CEPH多站点数据同步的开发（我在想这玩意到底有什么用）
3. 完成了一个ElasticSearch项目的开发，大概就是优化优化再优化

其实收获最大的还是开发一个**RPC Server**，当然对于大佬们可能啥也不算

我其实不建议自己造轮子，因为现有已经有很多的Rpc框架可用，而且都非常稳定

但是此次开发，涉及到一些特殊的功能，所以不得不自己造轮子

但是对我很重要，我将这次开发的过程记录下来

并且，将此次RPC代码抹去项目核心，开源至GITHUB当中，希望大家好好利用

爱你们！

![](https://blog-1256169066.cos.ap-chengdu.myqcloud.com/rpc-python/5.jpg)

## RPC服务的基础

首先，需要知道什么是RPC，或者RPC到底是干嘛的

这里我将会简单的说明一下，RPC服务的核心

### 什么是RPC

首先，RPC的全名，叫做Remote Procedure Call

大白话来说，就是远程过程调用

更大白话来说，就是A，B两个服务器，我希望在A上控制B进行某些操作，比如 rm -rf（狗头），我们需要一个服务进行交互，这个服务就叫做RPC

所以RPC其实是一种网络通信的框架，基于HTTP，或者TCP，或者什么有的没的，这不重要

RPC不是一种协议，它没那么高深，它是一种工具。

### 常见的RPC有什么？

这个说起来就多了

Golang 有 **GRPC**，这个我写过文章，可能有个两三篇了吧

[使用Golang的gRPC框架的一点随想](https://yemilice.com/2020/07/07/%E4%BD%BF%E7%94%A8golang%E7%9A%84grpc%E6%A1%86%E6%9E%B6%E7%9A%84%E4%B8%80%E7%82%B9%E9%9A%8F%E6%83%B3/)

[学习etcd的消息协议gRPC一点随想](https://yemilice.com/2021/04/06/etcd%E7%9A%84%E6%B6%88%E6%81%AF%E5%8D%8F%E8%AE%AE-grpc%E5%AD%A6%E4%B9%A0%E9%9A%8F%E6%83%B3/)

自己个儿去翻一下，哈哈

Java系的，有福报厂的**Dubbo**

这位更是重量级

![](https://blog-1256169066.cos.ap-chengdu.myqcloud.com/rpc-python/1.png)

还有我曾经用过的**Thrift**（难用的1B）

## 一个完整的RPC框架应该包含什么？

### RPC的核心功能

1. client端 （调用方）
2. server端 （服务提供方）
3. Network Service （底层协议，HTTP/TCP）

这大概就是一个RPC服务的核心功能，其实简单的理解为

```

发送方（client） ---> 中间协议 ---> 接收方（server）

```

![](https://blog-1256169066.cos.ap-chengdu.myqcloud.com/rpc-python/1.jpg)

这样就可以表示它们的关系了。一个RPC框架，也需要包含这些功能，无论是简单的，还是复杂的，都要有这些功能。

### RPC的功能详解

研究了Grpc，还有其他的一些RPC服务，除了上述三个必须功能以外，还有几个**加强**功能，方便RPC服务续航前进的。

下面将这些功能解说一下，帮助大家打好基础，深入学习

#### 服务端

服务端，也就是Server端，这个东西，一般是以一个服务的形式，常驻在后台，等待客户端的调用

这个东西相当于大门，他规定了一些很重要的东西

1. 指定开放的端口
2. 维护一个call id，这个东西映射了一个call id map，call id map是对应的函数指针,这个我后面会说
3. 等待客户端请求
4. 调用本地函数
5. 返回执行结果

server端是常驻在后台的服务，也就是广义上俗称的RPC服务，没了server端，就相当于老虎没牙。

#### 客户端

客户端，也就是client端，一般就是发起调用的那一方，和server端是好基友，缺了谁都不行的那种

这个玩意相当于钥匙，告诉你我要调用，我要开门，我要进去那种

1. 将调用的函数/命令 映射为call id
2. 将调用/发送的数据流 传输给 server 端
3. 指定server的端口，密码，鉴权之类的东西
4. 等待执行结果
5. 获取执行结果

client端通常在我们的业务代码中调用，一般就是你写在业务代码里面的部分，client端相当于是钥匙，打开server端大门，执行我们要的操作。

#### 序列化

首先，rpc服务是去其他节点执行函数/操作，所以肯定会带有参数

我们有时候要传输参数，所以就要有个固定格式，然后也要有解析格式的步骤

但是A -> B 传输的过程中，传输的参数不在一个内存里，这样就没法进行通讯调用了

从 A 传输参数到 B，需要 A 把参数转换成字节流，传给B

然后B将字节流转换成自身能读取的格式

转换成字节流的步骤就叫**序列化**

字节流转换成自身能读取的格式被称为**反序列化**

例如Grpc服务里面有protobuf，这是很典型的序列化操作，这里兄弟们可以自己去看下

我也写过。

[学习etcd的消息协议gRPC一点随想](https://yemilice.com/2021/04/06/etcd%E7%9A%84%E6%B6%88%E6%81%AF%E5%8D%8F%E8%AE%AE-grpc%E5%AD%A6%E4%B9%A0%E9%9A%8F%E6%83%B3/)

![](https://blog-1256169066.cos.ap-chengdu.myqcloud.com/rpc-python/5.jpg)


#### 网络传输

顾名思义，网络传输，就是通过网络进行消息传递，那么这里肯定有一个网络传输逻辑，术语称为**网络传输层**

网络传输层的主要工作就是

```
把经过序列化/反序列化的字节流传递给服务端/客户端
```

这里就涉及到网络协议了，一般有什么TCP，UDP，HTTP之类的，基本都是网络的玩意儿

Grpc用了HTTP2协议

一般的RPC都是TCP连接，本次开发的RPC服务我也会用**TCP**协议，请注意


## 一个完整的RPC服务的主要流程

上一节给铁子们讲了一下RPC主要的功能，这一节就来说说，RPC的主要功能如何联系起来

顺便画几个流程图，更加方便大家学习浏览

这章挺重要的，因为这样类似于我们设计的RPC服务的基础架构

请大家，务必，认真细致的阅读此章

因为免得后面我写代码你啥也看不懂！

认真阅读！

认真阅读！

认真阅读！

重要的事儿说三遍。

你要不好好读，到时候对不上了，指定没有你好果汁

![](https://blog-1256169066.cos.ap-chengdu.myqcloud.com/rpc-python/4.jpg)

### RPC服务的流程

上一节我们详细讲述了RPC服务的四个重要功能

1. 服务端
2. 客户端
3. 序列化/反序列化
4. 网络传输

那么他们之间的联系是什么样的呢？

首先，我们还是假定我们有两个节点，一个client节点，一个server节点

假设我们现在要在远端执行一个 "Print" 函数

主要目的就是：Client节点请求访问Server节点上的Print函数，并且获取返回的结果

基本的流程如下

1. client节点，发起调用Print的请求
2. client节点，找到Print 函数的 call id
3. client节点，序列化调用的参数/call id
4. client节点，根据指定的ip，协议，端口发送字节流到server节点
5. server节点，接收到client的字节流
6. server节点，反序列化字节流，拿到id 或者参数
7. server节点，根据id 去 call map里面寻找函数
8. server节点，调用print函数
9. server节点，获取到print函数返回
10. server节点，序列化返回结果
11. serve节点，传输返回结果的字节流
12. client节点，接收返回结果字节流
13. client节点，反序列化字节流
14. client节点，输出返回结果

这个大概就是RPC调用函数的具体流程，细化一下，画个图

流程如下

![](https://blog-1256169066.cos.ap-chengdu.myqcloud.com/rpc-python/3.jpg)

这就是基础的RPC流程



### 写一个基础的RPC服务

上面，我们已经过了一遍RPC服务的流程

下面我们就来写代码，实现一个简单的RPC流程吧

首先给出我的开发环境

1. Python环境：3.7
2. 使用包：无第三方包

实现功能

```
1. client传递一个Message给server端
2. server端返回传递的Message
```

Server端代码

```python

# coding: utf-8

__author__ = 'Yemilice_lau'


from xmlrpc.server import SimpleXMLRPCServer

class SimpleRpcServer:
    # 创建一个指定的rpc 函数 list
    _rpc_methods_ = ['PrintWork']
    def __init__(self, address):
        # 倒入自有的rpc服务包
        self._serv = SimpleXMLRPCServer(address, allow_none=True)
        for name in self._rpc_methods_:
            self._serv.register_function(getattr(self, name))

    # 主函数
    def PrintWork(self, name):
        print(name)

    # 持续运行的server
    def serve_forever(self):
        self._serv.serve_forever()

# Example
if __name__ == '__main__':
    print("服务开始.....")
    # 指定一个端口 7900
    kvserv = SimpleRpcServer(('127.0.0.1', 7900))
    kvserv.serve_forever()
```

client端代码

```python
# coding: utf-8

__author__ = 'Yemilice_lau'

from xmlrpc.client import ServerProxy

s = ServerProxy('http://localhost:7900', allow_none=True)

# 发送一个消息给server端
s.PrintWork("Good job")
```

这下可以调用一下server玩玩

```shell
python rpcserver.py
```

输出

```
服务开始.....
```

现在执行client

```shell
python rpcclient.py
```

这时server端返回了

```python
127.0.0.1 - - [30/Aug/2021 17:13:49] "POST /RPC2 HTTP/1.1" 200 -
Good job
```

这样就完成了一个简单的RPC服务流程

## 一个优秀的RPC服务应该有哪些特点？

上面我们简单的实现了一个RPC服务，但是那只是非常非常简单的RPC服务

相当于你想整个高达，但是最后整出来个卡布达，还是人工智障形态的

这部分关系到我们后面章节的开发，还是认真看一下

既然要做，就要做到最好，所以我自己思考了一下

那么到底什么样的RPC服务才能被称为优秀的RPC服务呢？

根据我这一阵的开发来看，一个优秀的RPC服务，肯定是有如下优点的

### 速度

速度肯定是很重要的

以前我的博客聊过，影响RPC速度有这么几个重要原因

1. 序列化/反序列化部分影响
2. 网络环境影响
3. server代码影响

首先，序列化，反序列化受传输字节流大小影响，转换比较大的字节流是比较花费时间的

所以Grpc改用了protobuf作为序列化/反序列化的核心，抛弃掉传统的json传输转换，Grpc为什么很快，很大一部分原因来源于protobuf的高效率，所以序列化/反序列化对速度的影响非常重要

网络环境，这个属实是基础逻辑了，你网慢，自然而然返回，发送的就慢，现在基础的有Tcp传输，http传输，需要针对综合性的问题，进行协议选择

server代码，这部分可谈的也很多，例如队列机制，多进程机制，令牌桶一类，都可进行调优。

### 可用性

拿我此次开发来说，我面临的是一个集群（机器数量10+），并且需要支持

1. Rpc直接支持Python库调用
2. Rpc直接支持Shell命令调用
3. Rpc直接支持集群级别的消息发送和返回

这里就直接面对自己造轮子的境地，而且未来还不确定是不是会有跨语言的需求，例如某天突然要支持Golang，Java

这里需要考虑是否跨平台, restful json? http XML?

也需要考虑是否有异步操作，异步通信？同步通信？

所以Rpc的可用性，需要在此处就进行详细的考虑，免得后续开发造成很大的麻烦。

### 负载均衡

避免单个服务器接收过多的Rpc请求

### 隐私性

涉及到鉴权部分了，这里相对来说比较麻烦

常用的鉴权逻辑有

1. Public Key鉴权
2. 证书鉴权
3. 网关鉴权

需要根据自己的实际情况进行分析，这边扯这个淡时间就太长了

我这次开发鉴权用的是证书，哥几个心里有数就行。

### 容错性

容错性，这个说起来还是很简单，但是你细分析下就觉得这破玩意是真的挺多

1. 出现错误之后的处理
2. 是否有异常的处理机制
3. 远端和本地数据不一致的问题（一致性问题）
4. 异常事务的回滚机制

首先，一致性问题就够TM头大了，这个原来我写过Raft协议，专门写过这个，这里不再细说

还有异常情况，这里也需要进行细分

1. 网络？
2. 执行返回结果？
3. 其他错误？

发生了异常处理之后，还有

1. 是否重试？
2. 是否回滚？
3. 是否抛出？

这个真的麻烦，这也是我们要考虑的点

## 暂时结尾

ok！兄弟们，全体目光像我看齐，看我看我，我宣布个事儿

![](https://blog-1256169066.cos.ap-chengdu.myqcloud.com/rpc-python/6.jpg)

咳咳，错了

基础知识篇这就结束了，咱们马上要进入高阶开发篇了！

这次开发花费时间太长了，我也太累了，最近都没好好学习！

这样可不行！

努把力，继续刷算法，努力！哥们！努力!

预告一下下一篇：

高阶开发篇！

下一篇的主要内容有

1. 实现Rpc队列
2. 实现Rpc直接调用Python库
3. 实现Rpc直接调用shell命令
4. 实现Rpc传输json
5. 实现协程Rpc

爱你们！