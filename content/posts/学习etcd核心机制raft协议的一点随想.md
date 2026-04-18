---
title: 学习etcd核心机制Raft协议的一点随想
date: 2021-04-04 14:50:25
tags: ["前后端"]
---

## 前言

最近开始学习k8s相关的东西，不可避免的和etcd搭上了交道，说来使用etcd的日子也不短了

中途开源过Python和Golang的etcd的api，代码在github上，公布一下

Golang的代码：

```
https://github.com/Alexanderklau/Go_poject/tree/master/Go-Etcd
```

我只能说我会用，但是，会用是远远不够的，所以，这几天看了一些书和博客，将etcd相关的一些重要的知识点进行梳理总结，并且整理输出

作为自己的笔记，希望可以帮到大家。

感谢 《etcd技术内幕》的作者！您的书给了我很大的启发，也推荐大家去看看！

## etcd到底是什么

是一个**分布式**的**KV存储**数据库，通过**raft算法**保持数据的一致性。

通常，我们会把etcd作为分布式系统的**数据共享**，或者是**服务发现**，**服务注册**等。

### 什么是服务发现？

当我们开发到后期，一个系统中包含的模块和服务越来越多，所以对需要对系统进行拆分，现在流行的微服务架构就差不多这样。

有时候多个服务互相依赖，假设，一个检索系统，tika和elasticsearch相互依赖，当tika挂掉之后，elasticsearch并不知道它挂了，所以，这就出现了负载均衡的问题。

当我们使用了服务发现后，tika和elasticsearch通过定期发送心跳，告知服务是否存活，当elasticsearch调用tika的时候，会调用服务发现组件，保证返回的服务地址可用，或者是良好的负载均衡。

### 什么是服务注册？

服务注册，其实和服务发现相辅相成，如果没有服务注册，那我们一般都是通过读取配置文件获取服务地址，如果有了服务注册，当启动服务，或者安装服务的时候，将信息注册到etcd当中，这时，服务发现就派上了用场了，请继续浏览上面的流程。

### 什么是数据共享？

这个很简单，在分布式系统中，我们经常需要同步配置文件，也有可能，分布式系统中的几个机器需要共用一个文件，例如rabbitmq的cookie。我们可以将这种数据存储在etcd当中，就避免了配置同步需要另写代码，或者是rpc，其他服务出现问题时同步失败的情况。

## etcd的基础数据模型

etcd有几个特殊功能，假设我们对一个键值进行修改

```json
{
    "work":"ak47"
}
```
如果现在我们把ak47换成m16，那么原有的ak47这个值不会消失/删除，而是会记录一个不同的版本号，把ak47和m16进行区分。这就是wathcer机制。

一个key会对应多个generation（翻译成代，世代的意思，可以记作周期），意思就是，一个key很可能有多个实例，也就是说，可以被修改多次，也可以对应多个值

当key被创建的时候，会同时创建一个generation实例，和这个key相互关联

当key被修改的时候，会记录当前版本到generation中

当key被删除时，会添加一个墓碑（tombstone）到 generation中，标识这个generation已经完蛋，并且创建一个新的generation。

所以，查询的步骤应该是

1. 寻找指定的key值
2. 获取全部generation版本号 
3. 根据查询的generation从存储中找到具体的Value值
4. 输出Value

所以，etcd不会进行覆盖的操作，而总是生成一个新的数据结构。

### 更详细的解释etcd的数据模型

首先，etcd将数据存放在一个持久化的B+树当中

etcd会维护一个字段序的B树索引，是为了加速针对key的范围扫描

在每个B树索引项中，都存储了一个key值，这也是为了快速定位指定的key或者进行范围扫描

所以回到上面讲的，etcd的每个key有多个版本，在每个revision的tree里，有多个对应的keys。


## etcd保证数据一致性的方法-Raft协议

etcd是一种分布式的数据库，分布式数据库里面，存在的一个很重要的问题就是，保证每个节点的数据一致，也就是**数据一致性**

一般的分布式数据库，etcd，elasticsearch之类的都会维护多个**副本**，这样我们上面的问题就变成了 **维护多节点副本的一致性**

那么什么又是**一致性**呢？

顾名思义，多个节点的数据一致，并且保持更新的状态，少数集群凉了也不影响整个集群的工作。

这就是**一致性**的妙处，Raft协议，就是实现一致性的重要算法。

### Raft协议的大白话解释

首先我看了一下Raft的一致性算法论文，论文的中文地址在

https://www.infoq.cn/article/raft-paper


我在这里大白话总结一下Raft协议的核心内容

首先，Raft会进行一个状态（角色）划分

假设现在我们有四个个节点node1, node2, node3，node4, 副本的协议是单副本

那么所有的节点就会处于三种角色状态

1. leader 主节点，通过follower选举出来的，接收所有的client请求
2. follower 跟随者，从主节点获取请求，进行相关操作
3. candidate 当leader出现故障时，选主过程打开，其他follower转为这个角色，直到选出新的leader

在这里可以看到了，leader是带头大哥，follower是小弟，任何时候都要听大哥指挥

大哥让你更新你就更新，让你删除你就删除。如果大哥被杀（leader挂掉），那么其他小弟（follower）就会竞争大哥的位置

你们看过《新世界》没，选帮派大哥，是不是有那味了？

竞选大哥（leader选举）的流程如下：

1. 其他小弟follower发现，大哥node1的心跳没了（leader节点不通，无返回，心跳机制接收不到leader的存活信息，leader timeout）
2. 开始触发选举 （election timeout）
3. 所有 follower 转变角色为 candidate
4. candidate（node2）开始投票，优先投票给自己（加勒比海盗选海盗王）
5. candidate (node2)给其他节点(node3, node4)发送选举请求（request vote）拉票
6. 其他的candidate(node3, node4)节点接收到请求后，如果他们还没有开始进行投票（上一步的投票给自己），就会投票给拉票节点(node2)
7. node2 获取了半数以上的票数，当选为新的大哥（leader）

但是，竞选大哥（leader）也可能出现一种情况，就是我支持大D，你支持阿乐，那谁当大哥啊？

这种问题也是有的，就是

当node2(大D) 得到了 node3 (根叔)的支持，拿到一票，但是node1（邓伯）投了 node4(阿乐)一票，这样就是2对2

当node2的选举请求（request vote）到达node3，但是node4 的选举请求到了node1的时候，会有一个election timeout的事件，这一步预示选举失败，要搞新和连胜了，重新选举一个大佬，所以上一轮选举失败。

node4(阿乐)的选举计时器（election timer）到期，直接timeout，阿乐会触发选举，继续发送选举请求（request vote）给根叔（node3）,邓伯（node4）,大D（node2），这时，邓伯和根叔的选举计时器（election timer也到期，接受到阿乐（node4）的信息，直接投票给阿乐（node4），大D（node4）这时候也到期了，这时node4（阿乐）已经获得了半数投票，加冕话事人，大D那票，也就无所谓了。

这就是Raft的大白话解读，我反正懂了，你们呢？

### 日志复制

首先书接上回，我们选出了大哥（leader）

大哥（leader）能干嘛

leader除了给小弟（follower）发号施令（心跳信息），还会接受到client的请求，比如更新，删除，等消息，发送到所有follower节点，进行相应的操作，当leader接受到半数以上的follower操作成功信息的时候，将成功的相应返回，对client进行应答。

回到本节，什么是日志复制？只是复制日志吗？

native了兄弟！在数据库里面，日志里面有数据也有操作，是很重要的东西，万一你把数据整丢了，通过日志也能恢复！

日志应该是挺重要的东西，etcd是怎么保护日志的那？

假设三个节点，node1（leader）, node2（follower）, node3（follower）

假设我们现在往etcd里面写了一条数据

```json
{
    "a":10
}
```

首先，这个写入的请求(set a=10)会被传入到node1（leader），然后被传输给node2，node3

node2，node3收到Append Entries的消息(set a=10)之后，会记录操作到本地的log当中，并且返回成功的信息

node1在收到半数以上的响应消息之后，才会认为集群已经成功记录了本次操作，node1会更新日志，提交对应日志的纪录状态，最后返回消息给client。


一个etcd集群当中，每个节点都需要维护一个log，除此之外，log维护还需要两个重要的数值

一个是commitIndex（当前日志索引值）

另一个是lastApplied （最后日志索引值）

说起来似乎有点抽象，其实就是，一个是现在日志的位置，另一个是最后一条日志的位置，做记录用的。

leader节点不仅要维护自己commitIndex，lastApplied，他还需要知道所有节点的commitIndex，lastApplied信息。

为啥呢，因为leader是老大啊，他需要知道每个follower节点的日志记录到哪了，从而保持大家的一致，也决定下次发送的消息里面包含什么信息。

所以leader还维护了两个数组，一个叫nextIndex，另一个叫matchIndex。

这个同样是为了更新用的

nextIndex记录的是发送给follower的下一条日志的索引值

matchIndex是记录了已经发送给follower的最大索引值

这都是由follower节点上报给leader的

太绕了，我举个例子吧。

还是用上面那个node1,node2,node3的环境。

假设，node3宕机过，现在数据不一致，那么leader节点的记录的三个节点的nextIndex和matchIndex的表示形式如下图，看图

![](https://blog-1256169066.cos.ap-chengdu.myqcloud.com/etcd/1.jpg)

黑色箭头代表的是nextIndex，红色箭头代表的是matchIndex，这边表示出来就是

```json
{
    "node1": {
        "nextIndex": 3,
        "matchIndex": 4
    }
}
```

所以leader里面应该是按照这个形式存储的nextIndex和matchIndex

我们发现node3和node1,node2有误差，是因为node3宕机过，现在在leader中，记录了node3的nextIndex和matchIndex

```json
{
    "node3": {
        "nextIndex": 0,
        "matchIndex": 1
    }
}
```
这时leader还能知道node3的日志具体位置，也知道该向node3发送哪些日志信息。

但是有一种特殊情况，leader挂掉之后，新leader接替上位，新leader会把所有的nextIndex和matchIndex都置为空。新leader会以自己本节点的日志为核心，将其他节点的nextIndex置换为本节点提交日志的最后一条，将matchIndex置为0值。

新的leader会持续向其他节点发送append Entriesx消息，node3并没有2，3这两条日志，node3将会返回给leader追加失败的相应，leader会修改node3的nextIndex,matchIndex，将他们前移一位，继续进行追加，直到返回成功为止。

可以看下这个图

![](https://blog-1256169066.cos.ap-chengdu.myqcloud.com/etcd/2.jpg)


### 日志的新旧对比

上一节提到了日志复制，但是我们如何去保持日志永远都是最新的呢？

如果需要比较节点之间的日志新旧，就需要找到最后一条日志的索引值和任期号，用来决定谁才是最新的。

如果任期号比较大，那么认为此日志比较新。

如果任期号相同，日志索引值比较大的比较新。

在选举过程中，candidate节点成为leader的过程中，向半数以上的节点都发送了日志信息，所以，leader节点必然和其他节点都有一个相同的日志信息，也就是交集。

### 日志压缩与快照

随着etcd服务集群部署的时间越长，数据量也就越来越大，占用的资源也就越多，我们前一阵子，etcd的内存占用率非常高，一看，原来是有人把操作日志全记录在etcd当中了。

日志不能无限量增长下去，所以需要压缩机制和清除机制来释放空间。

一般在etcd中，大家都用压缩这种机制，因为压缩非常简单，而且效率也非常高。

快照包含了节点当前的数据状态，也包含了最后一条日志记录的任期和索引号

也就是说，假设，我们的日志记录了1-100条日志的任期和索引号，现在我们生成快照文件，只需要记录最后第100条的任期和索引号就得了，1-99条日志记录全部丢弃。

一般恢复快照的时候，都是leader节点发送快照给follower，follower使用快照恢复数据，这里是通过GRPC发送的网络消息，这里比较复杂，未来我写一篇日志详细讲下。


## 后记

etcd的基础过了一遍了，其实我弄明白的差不多了。

学习嘛，见贤思齐焉，见不贤而内自省也。

希望能帮助大家。

```
描绘人生不用名贵的画笔

我吐字成金

勾勒速写生活的点滴

妈妈总是说你还是个孩子

没有办法处理好自己的事

可我已经开始长胡子

扛起不屈的意志
```
