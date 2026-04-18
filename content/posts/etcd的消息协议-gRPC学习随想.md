---
title: 学习etcd的消息协议gRPC一点随想
date: 2021-04-06 10:39:55
tags: ["前后端"]
---

## 前言

首先，要在这里说明一个情况，我的etcd的学习进入了第二个阶段

再一个，未来我将不会更新一些基础代码用法，未来的博客文章将会持续性输出阅读源码和算法之类的文章，因为我的学习也进入了另一个阶段。

大家共勉吧，希望大家都能找到合适的，适合自己的工作，也希望大家都可以快快乐乐的工作和生活。

爱你们！

我以前也写过一篇gRPC鸡翅使用的的相关文章

如果大家不了解gRPC，请先看看怎么用

[使用golang的grpc框架的一点随想](https://yemilice.com/2020/07/07/%E4%BD%BF%E7%94%A8golang%E7%9A%84grpc%E6%A1%86%E6%9E%B6%E7%9A%84%E4%B8%80%E7%82%B9%E9%9A%8F%E6%83%B3/)


## etcd和grpc的关系

首先，为什么我要把etcd和grpc放到一起，或者说，他们到底有什么PY交易，导致etcd一定要用grpc呢。

这里就要将一些基础理论了

首先简单说明他们的关系

etcd v3 使用了gRPC作为了它的消息协议。

etcd 项目包括基于 gRPC 的 Go client 和 命令行工具 etcdctl，通过 gRPC 和 etcd 集群通讯。

对于不支持 gRPC 支持的语言，etcd 提供 JSON 的 grpc-gateway。这个网关提供 RESTful 代理，翻译 HTTP/JSON 请求为 gRPC 消息。

这里的资料来源于[etcd官方文档中文版](https://doczhcn.gitbook.io/etcd/)

这里用大白话来说，就是

etcd v3 是通过grpc通信的，并且etcd惯用的管理命令 etcdctl，也是通过grpc进行etcd集群的管理和消息分发的。

## 为什么etcd v3要使用gRPC

这里就要了解一下etcd 协议的变迁了

首先，etcd v2 使用的是传统的 http+JSON 和server端进行交互，http+JSON的组合必须为每个请求建立一个连接，相当于**一对一**。

etcd v3 就开始完全采用了gRPC进行通信底层的协议消息。

gRPC是通过了protocol buffer进行定义管理，gRPC在处理网络连接的优势非常明显，因为它使用单一连接的HTTP2， 实现多路复用的RPC，相当于**一对多**

为什么原有的http+JSON被替换了呢？

分开剖析一下。

### JSON 和 protobuf的对比

首先，RPC这个东西，主要就是把消息（内存对象）转换成信息流，发给server端，然后server端再给它转换成需要的数据类型。

etcd v2 是通过JSON作为消息传递的数据格式

etcd v3 是通过protobuf作为消息传递的数据格式

不同的地方出现了，首先，protobuf 替代了 JSON，那，为什么JSON这种老牌数据格式被替换了呢？

这里我查阅了一些资料，[这篇文章](https://zhuanlan.zhihu.com/p/331593548)写的真的很好，一看就大概明白了，我在这里感谢这个作者！

这个作者的文章地址在：https://zhuanlan.zhihu.com/p/331593548

很感谢您！您是我的指路明灯！

首先先说一下结论：**protobuf的效率高于JSON**

现在分析一下为什么会这样。

看下这段JSON

```json
{
    "work":123,
    "work2":"456",
    "work3":{
        "work31":789
        }
}

{
    "work":789,
    "work2":"456",
    "work3":{
        "work31":123
        }
}
```

可以看到，JSON中包含很多类型的值，但是有些值是**非字符串**的，丽日，work的值是123，这个值的类型是int，内存表示只占两个字节，转成 JSON 却要五个字节。bool 字段则占了四或五个字节，所以，JSON其实有很多不必要的内存占用的。

还有一个问题**重复传输字段**，可以看见，同样的key，work，只是因为值不同，就要传输两次work这个key值，所以造成了不必要的冗余，所以这就是JSON不足的地方。

那么protobuf是从哪里解决这个问题的呢？

首先，protobuf首先需要定义好要传输的字段类型和字段名，例如

```protobuf
message TestWork {
    string message = 1;
    int32  code = 2;
}
```

定义好之后，直接编译成二进制文件 .proto

这部分不明白的可以看一下我以前的blog: [使用golang的grpc框架的一点随想](https://yemilice.com/2020/07/07/%E4%BD%BF%E7%94%A8golang%E7%9A%84grpc%E6%A1%86%E6%9E%B6%E7%9A%84%E4%B8%80%E7%82%B9%E9%9A%8F%E6%83%B3/)

编译之后，protobuf对数字之类的编码，使用了VarInts

Varints是将一个整数序列化为一个或多个Bytes的方法,越小的整数，使用的Bytes越小。所以解决了JSON的第一个问题，非字符串的资源占用**效率问题**。

并且看上面，protobuf直接定义好了要传输的字段名，给每个字段指定了一个整数编号。就像上面。这里传输的时候，可以直接传递编号，不用带上字段传输，这样增加了效率，避免了第二个问题：**冗余问题**

所以，这就是protobuf替代JSON的必要条件。

### HTTP API 和 gRPC的对比

从上面可以知道，etcd v2 是直接用了http api，etcd v3兼容了两种模式，一种是 http api，一种是gRPC，那么，这两种服务，有什么不同的地方吗？

这边简单列一个表格，对他们进行比较

| 功能 |  gRPC   | HTTP API  |
| ---  |  ----   | ----      |
| 协定 | .proto(必须用)  | OpenAPI（可以不用） |
|  协议 | HTTP/2  | HTTP |
|  Payload | Protobuf(二进制，不可外部读取)  | JSON（可外部读取） |
|  规定性 | 非常严格  | 宽松 |
|  流式处理 | 客户端，服务器，双向  | 客户端，服务器 |
|  浏览器支持 | 不支持  | 支持 |
|  安全性 | TLS  | TLS |
|  客户端代码生成 | 可生成  | OpanAPI |

gRPC 替代 HTTP API的一些原因，相信大家在上面那个表里也能看出来。

我在这里总结一下gRPC的**优点**

1. 性能： protobuf序列化字段，负载小
2. 协议：转为HTTP/2 设计，比普通的HTTP紧凑高效，单个TCP可复用多个HTTP/2 调用
3. 代码生成：.proto文件自动生成，并且端到端生成消息和客户端代码
4. 严格规范：避免多平台的情况下出现分歧，各个平台实现一致。
5. 流式处理：支持一元，服务到客户端，客户到服务端，双向流式传输
6. 超时处理支持：支持rpc内部的timeout，并且可以取消timeout的服务

现在，为什么要用gRPC替代HTTP API，我相信，你心里，也应该有数了。

## etcd的gRPC源码简单解读

读源码真是个很头大的工作，反正我是觉得自己很菜，读起来很累，不过查询了一些资料，自己也读了一点，也算是明白了那么点点门道，哈哈。

### server端

首先，阅读gRPC源码，还是要先找到proto

etcd的gRPC的proto，放置的位置在：/etcdserver/etcdserverpb/rpc.proto

首先先看一下这个文件定义了哪些服务

```protobuf
service KV

service Watch

service Lease

service Cluster

service Maintenance

service Auth
``` 

定义了6个服务，服务里面有多个RPC方法，我们选一个最常用的KV来进行简单的分析吧。

首先看KV这个服务，代码如下

```protobuf
service KV {
  // Range gets the keys in the range from the key-value store.
  rpc Range(RangeRequest) returns (RangeResponse) {
      option (google.api.http) = {
        post: "/v3beta/kv/range"
        body: "*"
    };
  }

  // Put puts the given key into the key-value store.
  // A put request increments the revision of the key-value store
  // and generates one event in the event history.
  rpc Put(PutRequest) returns (PutResponse) {
      option (google.api.http) = {
        post: "/v3beta/kv/put"
        body: "*"
    };
  }

  // DeleteRange deletes the given range from the key-value store.
  // A delete request increments the revision of the key-value store
  // and generates a delete event in the event history for every deleted key.
  rpc DeleteRange(DeleteRangeRequest) returns (DeleteRangeResponse) {
      option (google.api.http) = {
        post: "/v3beta/kv/deleterange"
        body: "*"
    };
  }

  // Txn processes multiple requests in a single transaction.
  // A txn request increments the revision of the key-value store
  // and generates events with the same revision for every completed request.
  // It is not allowed to modify the same key several times within one txn.
  rpc Txn(TxnRequest) returns (TxnResponse) {
      option (google.api.http) = {
        post: "/v3beta/kv/txn"
        body: "*"
    };
  }

  // Compact compacts the event history in the etcd key-value store. The key-value
  // store should be periodically compacted or the event history will continue to grow
  // indefinitely.
  rpc Compact(CompactionRequest) returns (CompactionResponse) {
      option (google.api.http) = {
        post: "/v3beta/kv/compaction"
        body: "*"
    };
  }
}
```

然后我们需要找到对应的服务端的go文件，文件名叫 v3_server.go

先看下Put方法
```golang
func (s *EtcdServer) Put(ctx context.Context, r *pb.PutRequest) (*pb.PutResponse, error) {
	resp, err := s.raftRequest(ctx, pb.InternalRaftRequest{Put: r})
	if err != nil {
		return nil, err
	}
	return resp.(*pb.PutResponse), nil
}
```

看到了吗，有个**raftRequest**函数，追踪一下

```golang
func (s *EtcdServer) raftRequest(ctx context.Context, r pb.InternalRaftRequest) (proto.Message, error) {
	for {
		resp, err := s.raftRequestOnce(ctx, r)
		if err != auth.ErrAuthOldRevision {
			return resp, err
		}
	}
}
```

这部分代码调用**raftRequestOnce**，大概的意思就是如果出现错误，就进行重试。

```golang
func (s *EtcdServer) raftRequestOnce(ctx context.Context, r pb.InternalRaftRequest) (proto.Message, error) {
	result, err := s.processInternalRaftRequestOnce(ctx, r)
	if err != nil {
		return nil, err
	}
	if result.err != nil {
		return nil, result.err
	}
	return result.resp, nil
}
```

回到PUT部分的代码，大致意思就是，上传信息，如果错误，重试。

再看下Range方法

```golang
func (s *EtcdServer) Range(ctx context.Context, r *pb.RangeRequest) (*pb.RangeResponse, error) {
    // 判断请求是否是可以 read
	if !r.Serializable {
		err := s.linearizableReadNotify(ctx)
		if err != nil {
			return nil, err
		}
	}
	var resp *pb.RangeResponse
	var err error
    // 检查权限，看看权限是否可用
	chk := func(ai *auth.AuthInfo) error {
		return s.authStore.IsRangePermitted(ai, r.Key, r.RangeEnd)
	}
    // 查询kv时候的回调函数
	get := func() { resp, err = s.applyV3Base.Range(nil, r) }
	if serr := s.doSerialize(ctx, chk, get); serr != nil {
		return nil, serr
	}
	return resp, err
}
```

调用了一个doSerialize函数

看下它

```golang
func (s *EtcdServer) doSerialize(ctx context.Context, chk func(*auth.AuthInfo) error, get func()) error {
	for {
        // 获取权限相关信息
		ai, err := s.AuthInfoFromCtx(ctx)
		if err != nil {
			return err
		}
		if ai == nil {
			// chk expects non-nil AuthInfo; use empty credentials
			ai = &auth.AuthInfo{}
		}
        // 回调执行chk函数，校验权限
		if err = chk(ai); err != nil {
			if err == auth.ErrAuthOldRevision {
				continue
			}
			return err
		}
		// fetch response for serialized request
        // 回调get函数，通过authStore读取kv
		get()
		//  empty credentials or current auth info means no need to retry
        // 读完，权限没有更改，结束，否则，重试
		if ai.Revision == 0 || ai.Revision == s.authStore.Revision() {
			return nil
		}
		// avoid TOCTOU error, retry of the request is required.
	}
}
```


### cilent端

server看完了，该看下cilent端的部分代码了

client端的代码 放置在：/clientv3/client.go

下面，将针对几个重要函数进行源码解析

如果我们要启动一个etcd 的 client连接，我们应该

```golang
	client, err := clientv3.New(cfg)
	if err != nil {
		fmt.Println("连接ETCD失败")
		return nil, err
	}
```
追踪到核心代码 newClient

这里为了避免文章太长，将一些不必要的操作打了省略号，请注意！！

```golang
func newClient(cfg *Config) (*Client, error) {
    
    ......
	// use a temporary skeleton client to bootstrap first connection
    ......

	ctx, cancel := context.WithCancel(baseCtx)
    // 这里检测配置信息，并且创建一个client实例
	client := &Client{
		conn:     nil,
		dialerrc: make(chan error, 1),
		cfg:      *cfg,
		creds:    creds,
		ctx:      ctx,
		cancel:   cancel,
		mu:       new(sync.Mutex),
		callOpts: defaultCallOpts,
	}
    // 记录账户和密码
	if cfg.Username != "" && cfg.Password != "" {
		client.Username = cfg.Username
		client.Password = cfg.Password
	}
	......

    // 初始化balancer实例
	client.balancer = newHealthBalancer(cfg.Endpoints, cfg.DialTimeout, func(ep string) (bool, error) {
		return grpcHealthCheck(client, ep)
	})

	// use Endpoints[0] so that for https:// without any tls config given, then
	// grpc will assume the certificate server name is the endpoint host.
    // 建立一个网络连接
	conn, err := client.dial(cfg.Endpoints[0], grpc.WithBalancer(client.balancer))
	if err != nil {
		client.cancel()
		client.balancer.Close()
		return nil, err
	}
	client.conn = conn

    ......

    // 初始化多个客户端，前面的介绍过，有6个
	client.Cluster = NewCluster(client)
	client.KV = NewKV(client)
	client.Lease = NewLease(client)
	client.Watcher = NewWatcher(client)
	client.Auth = NewAuth(client)
	client.Maintenance = NewMaintenance(client)

	if cfg.RejectOldCluster {
		if err := client.checkVersion(); err != nil {
			client.Close()
			return nil, err
		}
	}

    // 启动一个goroutine，同步集群中的URL
	go client.autoSync()
	return client, nil
}
```

最后一步执行了一个goroutine，执行了一个autoSync 方法

这个方法的代码如下

```golang
func (c *Client) autoSync() {
    ......

	for {
		select {
		case <-c.ctx.Done():
			return
		case <-time.After(c.cfg.AutoSyncInterval):
			ctx, cancel := context.WithTimeout(c.ctx, 5*time.Second)
			err := c.Sync(ctx)
			cancel()
			if err != nil && err != c.ctx.Err() {
				logger.Println("Auto sync endpoints failed:", err)
			}
		}
	}
}
```

这里循环执行了一个Sync方法，方法代码如下

```golang
func (c *Client) Sync(ctx context.Context) error {
	mresp, err := c.MemberList(ctx)
	if err != nil {
		return err
	}
	var eps []string
	for _, m := range mresp.Members {
		eps = append(eps, m.ClientURLs...)
	}
	c.SetEndpoints(eps...)
	return nil
}
```

这里的的操作步骤，是请求当前的节点列表，然后更新本地的缓存。

下面我们举一个简单的put例子，看一下put的代码怎么写的

首先，写一个put代码

```golang
 etcd.client.Put(context.Background(), name, value)
```

追踪代码 clientv3/kv.go

```golang
type KV interface {
	// Put puts a key-value pair into etcd.
	// Note that key,value can be plain bytes array and string is
	// an immutable representation of that bytes array.
	// To get a string of bytes, do string([]byte{0x10, 0x20}).
	Put(ctx context.Context, key, val string, opts ...OpOption) (*PutResponse, error)
    ....
}
```

持续追踪

```golang
func (kv *kv) Put(ctx context.Context, key, val string, opts ...OpOption) (*PutResponse, error) {
	r, err := kv.Do(ctx, OpPut(key, val, opts...))
	return r.put, toErr(ctx, err)
}
```

调用了kv.Do部分

看下kv.Do的代码

```golang
func (kv *kv) Do(ctx context.Context, op Op) (OpResponse, error) {
	var err error
	switch op.t {
    // 查询操作
	case tRange:
        .......
    // 上传操作
	case tPut:
		var resp *pb.PutResponse
		r := &pb.PutRequest{Key: op.key, Value: op.val, Lease: int64(op.leaseID), PrevKv: op.prevKV, IgnoreValue: op.ignoreValue, IgnoreLease: op.ignoreLease}
		resp, err = kv.remote.Put(ctx, r, kv.callOpts...)
		if err == nil {
			return OpResponse{put: (*PutResponse)(resp)}, nil
		}
    // 删除操作
	case tDeleteRange:
        .......
	case tTxn:
        .......
	default:
		panic("Unknown op")
	}
	return OpResponse{}, toErr(ctx, err)
}
```

看到了吗，put调用了 KVclient.Put的方法，这个方法在刚刚上面那个位置

/etcdserver/etcdserverpb/rpc.pb.go里面

```golang
type KVClient interface {
	Put(ctx context.Context, in *PutRequest, opts ...grpc.CallOption) (*PutResponse, error)
}
```

client v3的服务流程，就这样走完了。


## 后记

etcd grpc这部分就讲完了

其实grpc还有很多可以讲的东西，不过这篇blog不是这么玩的。

下一篇博客将会详细分析gRPC，或者是ElasticSearch的一些原理或者源码解读，或者是算法，请大家期待吧。


```
突然想写一首diss 歌曲

内心依旧激昂翻滚从未平息

想到那些不尊重人的faker coder 面试官

竖起中指对你们亲切表达

从来不care他人的看法

评判我的资格你还没有拿下

回去继续敲你那没用的代码

甩你开源5个身位

冒牌faker程序员还有资格坐在高位？

fuck off 垃圾傀儡。
```