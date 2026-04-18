---
categories: ["技术", "Python"]
title: Python的协程知识
date: 2020-09-01 10:20:47
tags: ["前后端", "杂七杂八"]
---

## 前言
最近心情一直不是太好，写出来的东西感觉也没有灵感，有些时候做了很多事，但是回想起来感觉自己还是什么都没有做。这是9月份第一次更新blog，也更新一篇相对高级一些的技术吧，有一阵没看Python了，想想还是不要落下了，剧透一下，下一篇文章还是针对Elasticsearch或者是前端框架React的，学习还是不能停下来，最近写歌也有问题，感觉自己什么也写不出来，仿佛失去了灵感，是生活还是时间消磨了我的灵气呢？我不愿意这么想，我会努力的，状态会调整过来的。

## 什么是协程？
首先上一个官方的解释

```
协程: 协程，又称微线程，纤程，英文名Coroutine。
协程的作用，是在执行函数A时，可以随时中断，去执行函数B，然后中断继续执行函数A（可以自由切换）。但这一过程并不是函数调用（没有调用语句），这一整个过程看似像多线程，然而协程只有一个线程执行.
```
你觉得抽象吗，那就让我给你一个完美的解释（狗头）

大白话解释，协程是个什么，其实就是告诉你，在线程的执行中，可以随时停止某个子程序，然后去执行别的子程序，在指定的时候，继续切回来干活，你可以把子程序认定为是Python中的函数，其实就是，几个兄弟干活，一个兄弟拉跨了，其他兄弟把他从工作岗位上扒拉下来，告诉他，小B崽子滚犊子，一边玩去，一会休息好了你再回来，还不耽误其他人干活，休息好了继续投身工作岗位。

一般协程在涉及到I/O操作的时候特别好用，你可以把协程理解为轻量级别的线程。

## 协程的好处是？
第一个就是解决I/O问题，什么是I/O问题呐，一般就是通过网络或者存储去访问或者写入数据，一般就是数据库取数据，或者是往数据库里面写数据等等，这都属于I/O操作。
协程说自己解决了I/O问题。其实就是协程由程序自己控制，减少线程切换的开销，不存在写变量的冲突，执行效率高于线程。

一般来说，由于GIL锁的限制，Python的线程相对拉跨，用了协程，就约等于起飞，至少在互联网，协程还是很重要的。

## Python协程的使用场景
一般都是高并发服务，用我自己的使用场景来说，举个例子

我现在有个服务，登陆的用户，需要定时更新自己的资料，当初的逻辑就是一个用户去开启一个线程访问，但是不停的开启，关闭线程开销太大了，如果登录用户过多，一次性开好几千个，岂不是很xx，这时候利用协程，一个线程开一大堆协程去处理这事儿，第一是减少了开销，第二是增加了效率。所以在频繁的I/O请求当中，协程是非常可取的。也是可靠的。

## Python协程的基础实现

这里分为Python2和Python3，这里的实现方式分很多种

### Python2的协程
Python2的协程支持不太好，但是兄弟们还是可以实现一下

Python2实现协程的方法就是 **yield + send** 和 **Gevent*

首先，Python怎么支持协程呐，是通过Generator实现的，也就是生成器，协程也是生成器的一种，只是遵循指定的规则，这边儿兄弟就不说生成器的逻辑了，这个后面再去研究，今儿只说协程。

在Python2中，指定一个生成器的方法是使用关键字**yield**，这里写一个简单的生产者消费者模型来说明协程的使用场景

首先这是一个普通的生产者消费者模型，看代码，这段代码来自于廖雪峰的网站
```python
def consumer():
    print("[CONSUMER] start")
    r = 'start'
    while True:
        n = yield r
        if not n:
            print("n is empty")
            continue
        print("[CONSUMER] Consumer is consuming %s" % n)
        r = "200 ok"


def producer(c):
    # 启动generator，send（None）是启动协程的必要环节
    start_value = c.send(None)
    print(start_value)
    n = 0
    #生产
    while n < 3:
        n += 1
        print("[PRODUCER] Producer is producing %d" % n)
        # 这里就是执行消费者的必要逻辑
        r = c.send(n)
        print('[PRODUCER] Consumer return: %s' % r)
    # 关闭generator
    c.close()


# 创建生成器
c = consumer()
# 传入generator
producer(c)
```

这里其实很好理解

1. 第一步，创建一个消费者生成器，生产者producer启动，**c.send(None)**的意思是启动/恢复生成器，这边启动一个生成器，开始生产。
2. 第二步，消费者是一个生成器对象，可以被生产者调用，代码在继续执行，执行到生产者的**c.send(n)**时，发送了一个n值给消费者，消费者获取到值，进行消费操作。
3. 当不再执行生产者，调用close，关闭操作。
   
这里有两个重要参数  

1. send(None) ： 启动

2. send(value) ： 传递参数

生产者生产消息之后，通过yield直接执行消费者，消费者执行完之后立刻切换生产者，同函数内操作，避免了线程锁，队列等待等，还是比较快的。

旧的生产者消费者模型，是通过lock来控制队列，在协程当中，producer和consumer相互合作，从头到尾没有用到lock，所以，这才是协程的最佳表现呀。

### Python3的协程

Python3引入了牛逼的async，这时候调用起来更加起飞，这个上一篇我写了个读源码的，其实简单一句话，async首先会整一个事件循环的loop，然后轮询任务，直到最后一个任务结束，这个我上一篇写过一个读源码的，这里也就不多废话了。

写点代码来表述一下Python3的协程怎么用，这里我直接参考了网上的一个兄弟，这里很感谢他，如果代码是你写的，请联系我，我加上你的署名，感恩！

```python
import time
import asyncio

async def taskIO_1():
    print('开始运行IO任务1...')
    await asyncio.sleep(2)  # 假设该任务耗时2s
    print('IO任务1已完成，耗时2s')
    return taskIO_1.__name__

async def taskIO_2():
    print('开始运行IO任务2...')
    await asyncio.sleep(3)  # 假设该任务耗时3s
    print('IO任务2已完成，耗时3s')
    return taskIO_2.__name__

async def main(): # 调用方
    tasks = [taskIO_1(), taskIO_2()]  # 把所有任务添加到task中
    done, pending = await asyncio.wait(tasks) # 子生成器
    for r in done: # done和pending都是一个任务，所以返回结果需要逐个调用result()
        print('协程无序返回值：'+ r.result())

if __name__ == '__main__':
    start = time.time()
    loop = asyncio.get_event_loop() # 创建一个事件循环对象loop
    try:
        loop.run_until_complete(main()) # 完成事件循环，直到最后一个任务结束
    finally:
        loop.close() # 结束事件循环
    print('所有IO任务总耗时%.5f秒' % float(time.time()-start))

```


## 结尾
基础的协程逻辑就是这些，最近很久没写blog了，下一篇应该是和Elasticsearch或者k8s有关，先这样吧。最近也太累了，想好好休息下，也要思考下换一份工作了。

