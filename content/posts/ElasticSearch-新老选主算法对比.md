---
title: ElasticSearch-新老选主算法对比
date: 2021-06-16 15:17:06
categories: ["技术", "Elasticsearch"]
tags: ["前后端"]
---

## 前言

首先，ElasticSearch 7，也就是Es 7， 变动还是有点儿大，改了很多东西，例如取消了type，修改了选主算法之类的操作

正好几天在钻研一些选主算法一类的东西，看了ETCD，rabbitmq，kafka之类的一些选主算法，想起来似乎对于Es，我还没有细致研究

于是产生了写这篇文章的动力，这篇文章，也是一篇新老选主算法的对比文章

大概会描述一下Es的两种选主算法，然后分析一下新老算法的差异

好了，我们开始把。

老样子，一段freestyle


```

黑暗笼罩着我的眼

看不清到底是黎明还是黑夜

时间依旧不会停歇滴答滴答

本质是仅仅如此还是始终如一？

我问自己到底想要何物

夸父逐日

终于大泽之尾

停下来的风景似乎更美
```

## 老选主算法 Bully算法（在Es7之前的所有版本使用）

Bully算法，在ElasticSearch 7.0之前，都是Es的选主算法，在7.0之后被替换。bully，顾名思义，霸道选举，也叫霸道选举算法（自己编的）

### Bully算法的原理

首先，一句话概括：

**Bully算法的基本原理就是，根据节点的ID大小来判定谁是leader**

这个直接点明本质

#### Bully算法的消息类型

Bully算法在选举的时候会发送三种消息类型

1. 选举消息 （Election Message: Sent to announce election.）
2. 应答消息（Answer (Alive) Message: Responds to the Election message.）
3. 选举成功消息 （Coordinator (Victory) Message: Sent by winner of the election to announce victory.）


这三种消息类型组成了Bully的基础消息类型，这也是Bully算法选举必须要了解的东西。请注意。

#### Bully算法的选举流程

还是用最熟悉的《黑社会》电影举例

假设我们现在有node1（大D）, node2（阿乐）, node3（吉米）三个进程，现在开始选主。

1. node1的进程号比node2，node3都大，他会直接通知node2，node3,发送选举成功（Coordinator Message）消息，如果它的进程ID小于node2，node3，那么他将发送选举消息 （Election Message）
2. 如果发送选举消息没有回应，这时候有两个原因，一个是其他节点的进程ID大于它，另一个是其他节点还在向集群内节点广播选举消息，也就是互相发送选举消息，就类似，大D和阿乐都自认自己是话事人。
3. 如果node2的进程ID大于node1，那么node1就会收到应答消息（Answer (Alive) Message），表明，选举失败，你不是话事人，等待其他节点的选举成功（Coordinator Message）消息
4. 如果node2进程ID小于node1，node1会返回一个应答消息（Answer (Alive) Message），启动选举进程，向更高的进程发送选举消息 （Election Message）
5. 如果node1接受到了node2节点选举成功（Coordinator Message）消息，则说明，node1看node2是master节点。

为了方便阅读，这里有个图将bully算法选举流程列出如下

![](https://blog-1256169066.cos.ap-chengdu.myqcloud.com/%E6%96%B0%E6%97%A7%E7%AE%97%E6%B3%95/1.png)

分步解释

1. 节点1向节点，节点3发送选举，并且带上自己的序号**1**
2. 节点2，3接收到消息之后，进行序号比较，发觉自己的序号更大，向节点1返回应答消息Answer (Alive) Message，告知节点1被踢出选主序列（大概是这个意思）
3. 节点2向节点3发送选举请求，节点3找不到更高序号的节点发送选举请求了
4. 节点3向节点2返回应答消息，节点3收不到其他节点的应答消息了
5. 节点3被认为是leader，向其他节点发送Coordinator Message，选举成功的请求，将自己是master节点广播到节点1，节点2

### Bully算法在ElasticSearch中如何运用（如何选出es的Master）

上面我已经把bully算法如何选举详细的说了，聪明的你应该明白了吧？

要不明白可以给我发邮件或者留言

下面说一下Es里面，bully如何选举出master的

首先看一下基础步骤，这里很可能会贴出java源码，看不懂的我也米办法了，凑合看

#### 找到活动的active node

可以选举的节点，这个是在配置文件 elasticsearch.yml 里面的 **discovery.zen.ping.unicast.hosts** 定义的

首先Es的Java实例会去直接调用ping逻辑检查可用的节点

```java
  List<DiscoveryNode> activeMasters = new ArrayList<>();
    for (ZenPing.PingResponse pingResponse : pingResponses) {
        // We can't include the local node in pingMasters list, otherwise we may up electing ourselves without
        // any check / verifications from other nodes in ZenDiscover#innerJoinCluster()
        if (pingResponse.master() != null && !localNode.equals(pingResponse.master())) {
            activeMasters.add(pingResponse.master());
        }
    }
```

这里会直接去ping 那些节点，然后把除本节点以外的，能ping通的，可用的，正在活动的节点加入到**activeMasters**这一个链表里面

#### 找到可以成为master的node

```java
 // nodes discovered during pinging
    List<ElectMasterService.MasterCandidate> masterCandidates = new ArrayList<>();
    for (ZenPing.PingResponse pingResponse : pingResponses) {
        if (pingResponse.node().isMasterNode()) {
            masterCandidates.add(new ElectMasterService.MasterCandidate(pingResponse.node(), pingResponse.getClusterStateVersion()));
        }
    }
```
这里是将所有的可以选举的节点（包括本节点），加入到了一个新的masterCandidates链表里面

#### Bully算法选举

看下代码

如果刚才的那个活动选举的链表activeMasters为空，也就是说不存在活着的master节点，然后再去再去配置文件里面**elasticsearch.yml**，有个重要属性，discovery.zen.minimum_master_nodes，这代表着最小选举节点，当activeMasters为空，并且minimum_master_nodes > masterCandidates，满足匹配数量，直接走bully的选举，选举出最小ID的节点成为master节点。

```java
if (activeMasters.isEmpty()) {
        if (electMaster.hasEnoughCandidates(masterCandidates)) {
            final ElectMasterService.MasterCandidate winner = electMaster.electMaster(masterCandidates);
            logger.trace("candidate {} won election", winner);
            return winner.getNode();
        } else {
            // if we don't have enough master nodes, we bail, because there are not enough master to elect from
            logger.warn("not enough master nodes discovered during pinging (found [{}], but needed [{}]), pinging again",
                        masterCandidates, electMaster.minimumMasterNodes());
            return null;
        }
    } else {
        assert !activeMasters.contains(localNode) : "local node should never be elected as master when other nodes indicate an active master";
        // lets tie break between discovered nodes
        return electMaster.tieBreakActiveMasters(activeMasters);
    }
```

### Bully算法的优点

其实Bully算法，简单粗暴，好处就是**简单**

并且算法的难度低啊，怪不得Es最初几个版本要用它，简单就完事儿了


而且bully的选举速度很快，相互通知，比对大小，就得了，所以我想这就是es最初几版用它的原因吧。


### Bully算法的缺点

缺点的话。。。我感觉bully算法相比于其他主流算法，相对来说简单一些

但是简单很可能就是一种原罪

当有节点频繁加入或者退出的时候，主节点会频繁的进行切换，保存的节点信息的元数据也就会越来越大，越来越大

所以Bully算法的缺点，从我的角度上来看的话，也就是无法满足复杂场景下的选主需求，因为复杂场景下的选主，需要强力的选主算法支撑

因为主节点，能力越大，责任越大！必须保持稳定！包租婆，你说是吧！

![](https://blog-1256169066.cos.ap-chengdu.myqcloud.com/%E6%96%B0%E6%97%A7%E7%AE%97%E6%B3%95/2.png)


### 为什么要替换Bully算法

Es7 版本算是大改，不仅改了选主算法，连type都删了，这算是一个大变动，我记得很多人的Es都是5，或者6

我查询了一些资料，发觉在Es官方的文章上说明了为什么替换的原因

```

Elasticsearch 6.x 及之前的版本使用了一个叫作 Zen Discovery 的集群协调子系统。

这个子系统经过多年的发展，成功地为大大小小的集群提供支持。

然而，我们想做出一些改进，这需要对它的工作方式作出一些根本性的修改。

Zen Discovery 允许用户通过设置 

discovery.zen.minimum_master_nodes 来决定多少个符合主节点条件的节点可以形成仲裁。

在每个节点上一定要正确地配置这个参数，如果集群进行动态扩展，也需要正确地更新它，这一点非常重要。

系统不可能检测到用户是否错误配置了这个参数，而且在实践当中，在
添加或删除节点之后很容易忘记调整这个参数。

Zen Discovery 试图通过在每次主节点选举过程中等待几秒来防止出现这种错误配置。

这意味着，如果所选的主节点失败，在选择替代节点之前，集群至少在几秒钟内是不可用的。

如果集群无法选举出一个主节点，有时候很难知道是为什么。
```

这个翻译太蹩脚了，我来直接简单扼要的说明一下为什么要替换吧

1. 老版本里面有个discovery.zen.minimum_master_nodes，这个很重要，但是动态扩展的时候有些时候可能会忘记设置这个东西
2. 如果不设置这个东西，Zen Discovery会在每次选举过程中等待一阵，大概是几秒时间来防止这种错误配置，这就造成集群暂时不可用
3. 也有可能造成无法选主的问题，这个非常致命，所以我们要换这套算法。

## 新选主算法 类Raft算法（Es7之后更新）

接上节，那么，Es不用Bully之后，到底要用什么呢？

官方给的文章也说明了这个问题


```

我们重新设计并重建了 Elasticsearch 7.0 的集群协调子系统：

1. 移除 minimum_master_nodes 参数，让 Elasticsearch 自己选择可以形成仲裁的节点。

2. 典型的主节点选举现在只需要很短的时间就可以完成。

3. 集群的伸缩变得更安全、更容易，并且可能造成丢失数据的系统配置选项更少了。

4. 节点更清楚地记录它们的状态，有助于诊断为什么它们不能加入集群或为什么无法选举出主节点

```

这边说的很清楚，原有的Zen Discovery被替换了，不再使用

那么新版采用的算法是什么呢？

这个在官方文档里也有

```

我们经常被问到的一个问题是，为什么不简单地“插入”像 Raft 一样的标准分布式共识算法。

有很多公认的算法，每种算法都有不同的利弊权衡。

我们仔细评估了所有可以找到的文献，并从中汲取灵感。

在我们早期的概念验证中，有一个概念便使用了非常接近 Raft 的协议。
```

所以，我将Es7的选主算法命名为类Raft算法，也就是类似Raft的算法。但是不是完全是Raft算法，这个我会在后面详细说明。

### Raft算法的原理

首先Es的选主算法基础是来源于Raft，还是有必要分析下Raft算法的机制和原理

才有助于后期对Es 7 的核心源码进行解读分析

#### 什么是Raft算法？

这个又涉及到了一致性问题，这是分布式系统的一个老调常谈的部分

首先，Raft算法是用来解决分布式一致性问题，而编写出来的一种算法。

#### Raft算法是如何工作的？

在Raft算法中，一个节点分别可能有三种角色

1. Follower 跟随者
2. Candidate 候选人
3. Leader 领导者

当初始化的时候，所有的节点都是Follower跟随者

此时此刻，如果没有收到leader的消息，那么节点将会自动转换为Candidate候选人的身份

这时，成为候选人的节点，将向其他节点发送选举投票请求

这时，接收到信号的节点，将会回复这次投票请求

如果大多数节点都赞成选举这个节点为leader，那么这个节点将会成为集群leader节点

这就是领导者选举的步骤，通俗来讲，也就是选主步骤，这也是我今天要讲的核心。

这里我画了个图，将步骤完全简化，大家，凑合看下吧，聪明的你，一定一看就懂！

![](https://blog-1256169066.cos.ap-chengdu.myqcloud.com/%E6%96%B0%E6%97%A7%E7%AE%97%E6%B3%95/3.png)

开始选举的条件是：

1. 当follower节点接收不到leader节点的心跳
2. leader节点超时

但是要注意，当follower节点收不到leader节点心跳的时候，需要等待一点时间切换为Candidate候选人，才可以开始选举。

看下图吧，大概，也许差不离，就是这么选举。


### 类Raft算法在ElasticSearch中如何运用

大概知道了Raft算法的选举原理，那么在Es里面，如何使用新的算法去选举leader呢？

这里就需要追一下源码了，下面又到了追源码的路子了，大家要看不了的话，真没办法了！

首先，ElasticSearch的选举算法套了Raft的壳子，但是不是真的Raft逻辑


相比于Raft算法，Es的选主算法有如下不同

1. 初始为 Candidate状态
2. 允许多次投票，也就是每个有投票资格的节点可以投多票
3. 候选人可以有投票的机会
4. 可能会产生多个主节点

举例来说，如果node1，node2，node3进行选主

如果node1当选leader，但是node2发来了投票要求，那么node1无条件退出leader状态，node2选为主节点，但是node3也发来了投票要求，那么node2退出leader状态，node3当选主节点。

说明白了，就是**保证最后当选的leader为主leader**

那么，综上所述，选举的流程为

节点的初始状态，为Candidate，当加入集群的时候，如果我们的discovery发现集群已经存在 leader，那么新加入的节点自动转换为follower。

类似下图：

![](https://blog-1256169066.cos.ap-chengdu.myqcloud.com/%E6%96%B0%E6%97%A7%E7%AE%97%E6%B3%95/4.jpg)

假设Candidate开始，那么节点收到足够的投票数量，转换为leader，假设其他节点发送了拉票请求，此节点辞去leader，转换为Candidate，直到选出leader，转换为follower。

角色转变类似这张图：

![](https://blog-1256169066.cos.ap-chengdu.myqcloud.com/%E6%96%B0%E6%97%A7%E7%AE%97%E6%B3%95/5.jpg)


### 类Raft算法的优点

这个资料上面有，我就直接说了

1. 免去了 Zen Discovery 的 discovery.zen.minimum_master_nodes 配置，es 会自己选择可以形成仲裁的节点，用户只需配置一个初始 master 节点列表即可。也即集群扩容或缩容的过程中，不要再担心遗漏或配错 discovery.zen.minimum_master_nodes 配置
2. 新版 Leader 选举速度极大快于旧版。在旧版 Zen Discovery 中，每个节点都需要先通过 3 轮的 ZenPing 才能完成节点发现和 Leader 选举，而新版的算法通常只需要在 100ms 以内
3. 修复了 Zen Discovery 下的疑难问题，如重复的网络分区可能导致群集状态更新丢失问题

### 类Raft算法的缺点

1. 节点多的情况下，会造成重复选主多次，选主缓慢

暂时缺少资料支撑，只有些英文的。并且还TM不全。。回头补上。

### 其他Raft算法的实现（哪些知名项目也使用了Raft）

这个就有话说了

除开我研究过一阵的ETCD，ETCD是把Raft协议用到核心层的分布式数据库

这个我原来写过文章

[学习etcd核心机制Raft协议的一点随想](https://yemilice.com/2021/04/04/%E5%AD%A6%E4%B9%A0etcd%E6%A0%B8%E5%BF%83%E6%9C%BA%E5%88%B6raft%E5%8D%8F%E8%AE%AE%E7%9A%84%E4%B8%80%E7%82%B9%E9%9A%8F%E6%83%B3/)

可以自己去翻一下。

还有**kafka**

kafka原有的一致性算法是Zookeeper提供的ZAB协议，但是2.8被替换成了Raft，说明Zookeeper已经被抛弃了，Raft的一些机制的确是好于ZAB

当然也有RocketMQ，内部核心来源于dledger源码，也是基于Raft的。

## 总结

草草写完，过于浅显，入个门也够了，也作为我自己的笔记

希望有大佬看见，能给予指点，这次借鉴了**张超**老师的很多博客，包括他写的**Es源码解析**这本书，对我的帮助非常非常大！感谢您！张超老师！

6月第一篇blog，我还在不断努力，你们呢？

希望自己，努力的这段时间，会有好的结果，希望自己可以换个好点的工作。希望大家也是！看到博客的人，爱你们！

```

踌躇不前，

总是妄想能一步登天

登上山峰发觉

我依旧在山脚前沿

抬目望眼

所见皆是虚无贪念

停或走？

或是将生活翻面

直面内心懦弱的另一边。

```


参考文章
> [深入理解 Elasticsearch 7.x 新的集群协调层](https://www.easyice.cn/archives/332)
> [Elasticsearch 新版选主流程](https://helloyoubeautifulthing.net/blog/2019/08/20/new-cluster-coordination-in-elasticsearch/)