---
layout: new
title: ETCD分布式锁实现选主机制(Golang)
date: 2019-12-13 15:41:04
tags: ["前后端"]
---
# ETCD分布式锁实现选主机制(Golang)
## 为什么要写这篇文章
做架构的时候，涉及到系统的一个功能，有一个服务必须在指定的节点执行，并且需要有个节点来做任务分发，想了半天，那就搞个主节点做这事呗，所以就有了这篇文章的诞生，我把踩的坑和收获记录下来，方便未来查看和各位兄弟们参考。
## 选主机制是什么
举个例子，分布式系统内，好几台机器，总得分个三六九等，发号施令的时候总得有个带头大哥站出来，告诉其他小弟我们今天要干嘛干嘛之类的，这个大哥就是master节点，master节点一般都是做信息处理分发，或者重要服务运行之类的。所以，选主机制就是，选一个master出来，这个master可用，并且可以顺利发消息给其他小弟，其他小弟也认为你是master，就可以了。
## ETCD的分布式锁是什么
首先认为一点，它是**唯一的**，**全局的**，一个key值  
为什么一定要强调这个唯一和全局呢，因为分布式锁就是指定只能让一个客户端访问这个key值，其他的没法访问，这样才能保证它的唯一性。  
再一个，认为分布式锁是一个**限时的**，**会过期的**的key值  
你创建了一个key，要保证访问它的客户端时刻online，类似一个“心跳”的机制，如果持有锁的客户端崩溃了，那么key值在过期后会被删除，其他的客户端也可以继续抢key，继续接力，实现高可用。
## 选主机制怎么设计
其实主要的逻辑前面都说清楚了，我在这里叙述下我该怎么做。  
我们假设有三个节点，node1,node2,node3  
1. 三个节点都去创建一个全局的唯一key /dev/lock
2. 谁先创建成功谁就是master主节点  
3. 其他节点持续待命继续获取，主节点继续续租key值（key值会过期）
4. 持有key的节点down机，key值过期被删，其他节点创key成功，继续接力。
## ETCD分布式锁简单实现
看一下ETCD的golang代码，还是给出了如何去实现一个分布式锁，这个比较简单，我先写一个简单的Demo说下几个接口的功能  
1. 创建锁
```golang
kv = clientv3.NewKV(client)
txn = kv.Txn(context.TODO())
txn.If(clientv3.Compare(clientv3.CreateRevision("/dev/lock"),"=",0)).Then(
clientv3.OpPut("/dev/lock","占用",clientv3.WithLease(leaseId))).Else(clientv3.OpGet("/dev/lock"))
txnResponse,err = txn.Commit()
if err !=nil{
    fmt.Println(err)
    return
    }
```
2. 判断是否抢到锁
```golang
if txnResponse.Succeeded {
    fmt.Println("抢到锁了")
    } else {
        fmt.Println("没抢到锁",txnResponse.Responses[0].GetResponseRange().Kvs[0].Value)
        }
```
3. 续租逻辑
```golang
for {
    select {
        case leaseKeepAliveResponse = <-leaseKeepAliveChan:
        if leaseKeepAliveResponse != nil{
            fmt.Println("续租成功,leaseID :",leaseKeepAliveResponse.ID)
            }else {
                fmt.Println("续租失败")
                }
            }
            time.Sleep(time.Second*1)
        }
```
## 我的实现逻辑
首先我的逻辑就是，大家一起抢，谁抢到谁就一直续，要是不续了就另外的老哥上，能者居之嘛！我上一下我的实现代码
```golang
package main

import (
	"fmt"
	"context"
	"time"
	//"reflect"

	"go.etcd.io/etcd/clientv3"
)

var (
	lease                  clientv3.Lease
	ctx                    context.Context
	cancelFunc             context.CancelFunc
	leaseId                clientv3.LeaseID
	leaseGrantResponse     *clientv3.LeaseGrantResponse
	leaseKeepAliveChan     <-chan *clientv3.LeaseKeepAliveResponse
	leaseKeepAliveResponse *clientv3.LeaseKeepAliveResponse
	txn                    clientv3.Txn
	txnResponse            *clientv3.TxnResponse
	kv                     clientv3.KV
)

type ETCD struct {
	client                 *clientv3.Client
	cfg                    clientv3.Config
	err                    error
}

// 创建ETCD连接服务
func New(endpoints ...string) (*ETCD, error) {
	cfg := clientv3.Config{
		Endpoints: endpoints,
		DialTimeout: time.Second * 5,
	}

	client, err := clientv3.New(cfg)
	if err != nil {
		fmt.Println("连接ETCD失败")
		return nil, err
	}

	etcd := &ETCD{
		cfg: cfg,
		client: client,
	}

	fmt.Println("连接ETCD成功")
	return etcd, nil
}

// 抢锁逻辑
func (etcd *ETCD) Newleases_lock(ip string) (error) {
	lease := clientv3.NewLease(etcd.client)
	leaseGrantResponse, err := lease.Grant(context.TODO(), 5)
	if err != nil {
		fmt.Println(err)
		return err
	}
	leaseId := leaseGrantResponse.ID
	ctx, cancelFunc := context.WithCancel(context.TODO())
	defer cancelFunc()
	defer lease.Revoke(context.TODO(), leaseId)
	leaseKeepAliveChan, err := lease.KeepAlive(ctx, leaseId)
	if err != nil {
		fmt.Println(err)
		return err
	}

    // 初始化锁
	kv := clientv3.NewKV(etcd.client)
	txn := kv.Txn(context.TODO())
	txn.If(clientv3.Compare(clientv3.CreateRevision("/dev/lock"), "=", 0)).Then(
		clientv3.OpPut("/dev/lock", ip, clientv3.WithLease(leaseId))).Else(
		clientv3.OpGet("/dev/lock"))
	txnResponse, err := txn.Commit()
	if err != nil {
		fmt.Println(err)
		return err
    }
    // 判断是否抢锁成功
	if txnResponse.Succeeded {
		fmt.Println("抢到锁了")
        fmt.Println("选定主节点", ip)
        // 续租节点
        for {
            select {
            case leaseKeepAliveResponse = <-leaseKeepAliveChan:
                if leaseKeepAliveResponse != nil {
                    fmt.Println("续租成功,leaseID :", leaseKeepAliveResponse.ID)
                } else {
                    fmt.Println("续租失败")
                }
            }
        }
	} else {
        // 继续回头去抢，不停请求
		fmt.Println("没抢到锁", txnResponse.Responses[0].GetResponseRange().Kvs[0].Value)
		fmt.Println("继续抢")
		time.Sleep(time.Second * 1)
	}
	return nil
}

func main(){
    // 连接ETCD
	etcd, err := New("xxxxxxxx:2379")
	if err != nil {
		fmt.Println(err)
    }
    // 设定无限循环
	for {
		etcd.Newleases_lock("node1")
	}
}
```
## 总结
相关代码写入到github当中，其中的地址是  
https://github.com/Alexanderklau/Go_poject/tree/master/Go-Etcd/lock_work  
实现这个功能废了不少功夫，好久没写go了，自己太菜了，如果有老哥发现问题请联系我，好改正。