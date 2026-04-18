---
title: 读Python的协程库asyncio源码随想
date: 2020-08-27 14:39:33
tags: ["前后端", "黑科技", "杂七杂八", "其他技术"]
---
## 前言
其实想了一下，Python有一阵没好好看了，这样不好。刚好前阵子看了一下Golang的协程，我寻思看了那么多协程逻辑，也该看看Python的。

一看Python，都更新到3.8.3了，真厉害啊！我们现在线上的项目还是2.7，迁移的代价太大了，Python现在更新了asyncio库，这个库可以傻瓜式完成协程，异步等等操作，看起来是相当牛逼，话不多说，去看看它的源码，弄明白它的运行逻辑，顺便学习一下Python官方的代码写作手段，可别到时候，代码没写好，整出一大堆格式问题呀。

**这是我第一次写读源码类的文章，如果写的不好，请多多包涵呀！**

## Python中的协程是什么
前阵子我说了一下Golang的协程，现在又跳到Python的协程来了，巧了嘿，golang的协程，其实就是用一个关键字go去完成的

类似

```golang
go example()
```

而Python的协程是怎么表现的呢，其实就是通过一个关键字async定义，就像这样

```python
async def async_work():
    return 1
```

理论上来说，Python的协程调用类似

```python
>>> import asyncio

>>> async def main():
...     print('hello')
...     await asyncio.sleep(1)
...     print('world')

>>> asyncio.run(main())
hello
world
```

它并不像go一样，直接调用关键字就可以执行

简单地调用一个协程并不会将其加入执行日程，需要用到**run()**这个函数

所以还是有不一样的。。。这属于只是有Golang那味儿，但是核心不是人家西方那一套。

其实Python由于自身GIL线程全局锁，在处理一些需要高并发处理的操作时候就显得有些力不从心了，一般这种时候，在我用Python2开发的时候，我会用多进程+多线程混用操作，务必榨干Python2的所有性能，但是Python3之后引入了类似golang的关键字async，可以直接调用协程，这样不就有golang那味儿了嘛，所以我看了一下这个库的文档，也基本弄明白这东西怎么使了。

## 协程的创建（asyncio.create_task）

create_task函数用来并发运行作为 asyncio任务的多个协程，举个例子

将 coro 协程 打包为一个 Task 排入日程准备执行。返回 Task 对象

该任务会在 get_running_loop() 返回的循环中执行，如果当前线程没有在运行的循环则会引发 RuntimeError。
```python
async def coro():
    worksomething...

task = asyncio.create_task(coro())

```
基础使用看完了，然后呢。。。

先去源码库里面看一下create_task的源码(Python的版本是3.8.3)

```python
def create_task(self, coro, *, name=None):
    """Schedule a coroutine object.
    Return a task object.
    """
    self._check_closed()
    if self._task_factory is None:
        task = tasks.Task(coro, loop=self, name=name)
        if task._source_traceback:
            del task._source_traceback[-1]
    else:
        task = self._task_factory(self, coro)
        tasks._set_task_name(task, name)

    return task
```
这里主要接受到一个协程作为参数，并且还有name

```python
task = tasks.Task(coro, loop=self, name=name)
```

这里会新建一个task实例并且返回。

## 协程的休眠（asyncio.sleep）

有些时候需要让协程去等待，或者阻塞指定的时间，就需要调用sleep函数，sleep一般会把任务挂起，然后不影响其他任务运行。

以下协程示例运行 5 秒，每秒显示一次当前日期
```python
import asyncio
import datetime

async def display_date():
    loop = asyncio.get_running_loop()
    end_time = loop.time() + 5.0
    while True:
        print(datetime.datetime.now())
        if (loop.time() + 1.0) >= end_time:
            break
        await asyncio.sleep(1)

asyncio.run(display_date())
```

看下人家的源码
```python

@types.coroutine
def __sleep0():
    """Skip one event loop run cycle.
    This is a private helper for 'asyncio.sleep()', used
    when the 'delay' is set to 0.  It uses a bare 'yield'
    expression (which Task.__step knows how to handle)
    instead of creating a Future object.
    """
    yield


async def sleep(delay, result=None, *, loop=None):
    """Coroutine that completes after a given time (in seconds)."""
    if delay <= 0:
        await __sleep0()
        return result

    if loop is None:
        loop = events.get_running_loop()
    else:
        warnings.warn("The loop argument is deprecated since Python 3.8, "
                      "and scheduled for removal in Python 3.10.",
                      DeprecationWarning, stacklevel=2)

    future = loop.create_future()
    h = loop.call_later(delay,
                        futures._set_result_unless_cancelled,
                        future, result)
    try:
        return await future
    finally:
        h.cancel()
```

这里的写法让我一瞬间想到golang，粗略说一下把，这里主要是传入一个协程等待时间delay，通过调取get_running_loop()获取事件循环，等待yield，然后暂停执行协程达到的协程阻塞效果。

这里引出来一个重要概念，get_running_loop()，这个东西是干嘛的？

## get_running_loop函数是干嘛的？

首先去定位一下人家的源码

```python
class _RunningLoop(threading.local):
    loop_pid = (None, None)


_running_loop = _RunningLoop()


def get_running_loop():
    """Return the running event loop.  Raise a RuntimeError if there is none.
    This function is thread-specific.
    """
    # NOTE: this function is implemented in C (see _asynciomodule.c)
    loop = _get_running_loop()
    if loop is None:
        raise RuntimeError('no running event loop')
    return loop


def _get_running_loop():
    """Return the running event loop or None.
    输出一个event loop，主要是一个循环事件的TLS
    This is a low-level function intended to be used by event loops.
    This function is thread-specific.
    """
    # NOTE: this function is implemented in C (see _asynciomodule.c)
    running_loop, pid = _running_loop.loop_pid
    if running_loop is not None and pid == os.getpid():
        return running_loop
```

可以看出来一点，get_running_loop（获取事件循环）的主要功能就是返回当前 OS 线程中正在运行的事件循环，这样可能说起来抽象一点，大白话的意思就是事件循环是asyncio的核心，异步任务的运行、任务完成之后的回调、网络IO操作、子进程的运行，都是通过事件循环完成的。所以这个相当于是发动机了。

## 协程的结果（asyncio.Future）

这里换种理解方法

有些时候我们想得到协程的返回值，但是在Python的协程里面任务有些时候会执行，但有些时候不会，所以Future相当于一个最终结果，无论协程是否执行。

Future是一个await（可等待）对象，await我下面会说，意思就是，它其实和Task一样，是一个可以被等待的。

其实理解一下，就是Future是协程的封装函数。

这部分源码太多了，我直接拿官方例子来说明

```python
async def set_after(fut, delay, value):
    # 设置一个等待时间 depay是秒数
    await asyncio.sleep(delay)

    # 设置一个future的结果
    fut.set_result(value)


async def main():
    # 来一个loop循环
    loop = asyncio.get_running_loop()

    # 来一个新的future
    fut = loop.create_future()

    # Run "set_after()" coroutine in a parallel Task.
    # 创建一个task，去设置future的结果
    # We are using the low-level "loop.create_task()" API here because
    # 开始
    # we already have a reference to the event loop at hand.
    # Otherwise we could have just used "asyncio.create_task()".
    loop.create_task(
        set_after(fut, 1, '... world'))

    print('hello ...')

    # Wait until *fut* has a result (1 second) and print it.
    print(await fut)

asyncio.run(main())
```

上面那个例子就是创建一个Future对象，创建和调度一个异步任务去设置Future结果，最后再去等待结果。这就是基础的用法。

## 协程的等待（await）

await其实就是声明协程挂起，其实就是在执行的时候挂起某个协程函数，等待挂起条件结束后，再回来执行。

它其实是一个关键字，就像async一样

这里举个例子

```python
>>> import asyncio
 
>>> async def main():
...     print('hello')
        # 等待1s
...     await asyncio.sleep(1)
...     print('world')
 
>>> asyncio.run(main())
hello
world
```


## 结尾
这块真的好久没看了，我也是第一次写源码类文章，我估摸着写的的确不太好，不过一回生，二回熟，下次我会写的更好的，哈哈哈

今天终于周五啦,好开心啊，下一步应该是继续刷算法看书了，大家有什么不懂的可以给我发邮件的。


