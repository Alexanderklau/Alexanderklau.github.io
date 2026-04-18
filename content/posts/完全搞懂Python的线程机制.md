---
title: 完全搞懂Python的线程机制
date: 2020-12-01 13:28:46
tags: ["前后端"]
---

## 前言

总觉得用了Python那么久，感觉隔一层面纱似的，只是会用而已。

现在Python更新了3.9，越来越厉害了，多了很多新的语法特性。

想想自己，也不能总是写一些Golang的东西，Python也要持续跟上，所以打算开一个Python三巨头的坑，把Python的线程，进程，协程都讲明白，这样第一个是让我加深理解，第二个是让我彻底搞懂这三者的机制，我认为这样我才可以走的更远。

在开发的过程当中，我们总是需要去提高效率，一般在Python里面，我们经常会听到所谓多线程，多进程，加上现在的协程的诸多方法去解决性能问题，那么，到底这些术语到底是个什么意思，我们什么时候用什么方法，才是最合适的呢？这才是促使我写这篇文章的原因吧。

因为这些东西，应该是Python最核心的了，搞懂这个，Python也就没什么特别难的问题了。所以，我们开始吧

```
从来不畏惧到底明天会是什么模样

活在今天告诉你生命不只一种颜色

咬牙挺住却坚信黑暗之后即是黎明

解剖生活才能看到未来的光
```

## Python的多线程是什么

首先，线程是什么，说的简单一点，线程就是进程的儿子，一个进程可以搞出多个线程，但是一个线程只能有一个爹（进程），不允许随便乱认爹（进程）。

这就好比你看电影，电影是一个进程，里面的画面是一个线程，声音是一个线程，字幕又是一个线程，这就是线程的一种表现。

如果是系统执行进程，首先需要划分独立的进程空间/内存，但是线程就不会这么麻烦。

线程是独立执行的，并且也是并发的，占用资源也小，一个线程可以去创建/杀死另一个线程，相当于“子子孙孙无穷匮也”，并且线程之间还支持通信，可以共享其中的数据/内存等等。

上面这些都是线程的优点，但是！但是！但是！**Python的线程和这个不一样**！

Python的线程其实是个假的，为啥呢，因为Python有**GIL全局锁**。

不知道你们看过《无限恐怖》这本小说没有，里面有个东西叫做基因锁，因为有基因锁的存在，导致人类的各项机能被限制了，同理，GIL全局锁也是一样，它的逻辑就是，只要你用了带GIL锁的解释器，任何时候，你都只能，也只可以在同一时间执行一个线程。

带GIL锁的解释器有哪些呢？当然是我现在说的CPython啦！

CPython用C实现的，用的人也最多，所以为什么市面上的教材都说Python的线程不好使，是因为大家全都是用的CPython。

有点儿绕吧，其实意思就是，Python的多线程实际上是个伪多线程，并不是真正的多线程，实际上，无论如何，它只能保证一个线程执行。

我来描述一下Python多线程执行的逻辑

```
线程1获取到GIL锁
执行业务代码
线程1释放GIL锁

线程2获取到GIL锁
执行业务代码
线程2释放GIL锁
......
```

这个是不是贼TM绕，我写个代码给大家伙儿瞅瞅到底这个是怎么表现出来的

```python
import threading
import time

exitFlag = 0

# 继承的方法写一个多线程逻辑
class myThread (threading.Thread):

    def __init__(self, threadID, name, counter):
        threading.Thread.__init__(self)
        self.threadID = threadID
        self.name = name
        self.counter = counter
        
    def run(self):
        print ("开始线程：" + self.name)
        print_time(self.name, self.counter, 5)
        print ("退出线程：" + self.name)

def print_time(threadName, delay, counter):
    while counter:
        if exitFlag:
            threadName.exit()
        # 这里模拟一个阻塞
        time.sleep(delay)
        print ("%s: %s" % (threadName, time.ctime(time.time())))
        counter -= 1

# 创建新线程
thread1 = myThread(1, "Thread-1", 1)
thread2 = myThread(2, "Thread-2", 2)

# 开启新线程
thread1.start()
thread2.start()

# join等待
thread1.join()
thread2.join()
print ("退出主线程")
```

这里用了一个继承的方法，写了一个多线程逻辑

这里输出

```
开始线程：Thread-1
开始线程：Thread-2
Thread-1: Tue Dec  1 09:58:21 2020
Thread-2: Tue Dec  1 09:58:22 2020
Thread-1: Tue Dec  1 09:58:22 2020
Thread-1: Tue Dec  1 09:58:23 2020
Thread-2: Tue Dec  1 09:58:24 2020
Thread-1: Tue Dec  1 09:58:24 2020
Thread-1: Tue Dec  1 09:58:25 2020
退出线程：Thread-1
Thread-2: Tue Dec  1 09:58:26 2020
Thread-2: Tue Dec  1 09:58:28 2020
Thread-2: Tue Dec  1 09:58:30 2020
退出线程：Thread-2
退出主线程
```

发现了没，这个，有个时间差，这两根本不是同时去运行的，这就是GIL的石锤

再看个图，这就是GIL的运行流程

![](https://blog-1256169066.cos.ap-chengdu.myqcloud.com/Python%E7%BA%BF%E7%A8%8B/2.jpg)

整明白了嘛？

但是你会说，哎呀，我没感觉到缓慢阿，害挺快的，是，这就是我下面要聊的，它到底用了什么技术，才能让我们感觉到挺快的。

## Python的多线程在代码部分是怎么实现的？

又涉及到看源码的东西了,这里首先追踪一把，threading的代码放置在

```
https://github.com/python/cpython/blob/3.9/Lib/threading.py
```

先从入口的Start追踪

```python
def start(self):
    """Start the thread's activity.

    It must be called at most once per thread object. It arranges for the
    object's run() method to be invoked in a separate thread of control.

    This method will raise a RuntimeError if called more than once on the
    same thread object.

    """
    # 线程是否初始化（这里就是刚那个init函数
    if not self._initialized:
        raise RuntimeError("thread.__init__() not called")

    # 设置线程的开始状态（把线程状态置为start
    if self._started.is_set():
        raise RuntimeError("threads can only be started once")

    # 加锁，调用GIL
    with _active_limbo_lock:
        _limbo[self] = self

    # 去启动新的线程
    try:
        _start_new_thread(self._bootstrap, ())
    # 如果启动出了问题
    except Exception:
        # 把锁给del了
        with _active_limbo_lock:
            del _limbo[self]
        raise
    # 释放锁，阻塞。直到被唤醒/超时
    self._started.wait()
```

其实threading的代码写的很好，已经大概把流程和步骤都说明白了，涉及到lock的部分是C写的, 首先追踪溯源，看一下启动线程的C代码

```c
void PyEval_InitThreads(void)
{
    if (gil_created())
        return;
    // 创建GIL锁
    create_gil();
    // 申请GIL锁
    take_gil(PyThreadState_GET());
    // 主线程
    main_thread = PyThread_get_thread_ident();
    // 如果没有等待锁
    if (!pending_lock)
        // 创建等待锁
        pending_lock = PyThread_allocate_lock();
}
```

可以看到
核心的GIL代码如下

```c
static void take_gil(PyThreadState *tstate)
{
    MUTEX_LOCK(gil_mutex);    // 加锁

    if (!_Py_atomic_load_relaxed(&gil_locked))
        // 获取 GIL，若已释放，直接获取，跳转
        goto _ready;

    while (_Py_atomic_load_relaxed(&gil_locked)) {
        // GIL 未释放
        int timed_out = 0;
        unsigned long saved_switchnum;
        // 记录切换次数
        saved_switchnum = gil_switch_number;
        // 利用 pthread_cond_tim 阻塞等待超时
        COND_TIMED_WAIT(gil_cond, gil_mutex, INTERVAL, timed_out);
        /* 等待超时，仍未释放, 发送释放请求信号 */
        if (timed_out &&
            _Py_atomic_load_relaxed(&gil_locked) &&
            gil_switch_number == saved_switchnum) {
            // 设置 gil_drop_request=1，eval_breaker=1
            // 释放信号
            SET_GIL_DROP_REQUEST();
        }
    }
_ready:
    /* We now hold the GIL */
    _Py_atomic_store_relaxed(&gil_locked, 1); // 设置 GIL 占用
    _Py_ANNOTATE_RWLOCK_ACQUIRED(&gil_locked, /*is_write=*/1);

    if (tstate != (PyThreadState*)_Py_atomic_load_relaxed(&gil_last_holder)) {
        _Py_atomic_store_relaxed(&gil_last_holder, (uintptr_t)tstate);
        ++gil_switch_number;
    }

    if (_Py_atomic_load_relaxed(&gil_drop_request)) {
        // 重置 gil_drop_request=0
        RESET_GIL_DROP_REQUEST();
    }
    if (tstate->async_exc != NULL) {
        _PyEval_SignalAsyncExc();
    }

    MUTEX_UNLOCK(gil_mutex);    // 解锁
}
```

这里实现了互斥锁gil_mutex，如果GIL被占用，那么将持续等待，超时后将修改重置变量。

这点代码给我看的累死了。

回到本段开始的问题，为什么GIL有时候感觉不到慢？

首先，GIL类似一个信号锁，意思就像是尚方宝剑，持有它的线程告诉其他线程，都不许动阿，我现在正用着呢，你们要么等着我主动释放，要么等我超时了自己释放，然后继续竞争切换持有GIL的线程。

核心的意思就是，多线程的好处在于，阻塞并不会影响其他的线程，因为阻塞的线程持有的GIL马上就被释放了，其他线程可以接力马上干活儿，不会出现阻塞的情况。

这里可以看一下切换线程的代码

```c
PyObject *
_PyEval_EvalFrameDefault(PyFrameObject *f, int throwflag){
    ...
    for (;;) {
        if (_Py_atomic_load_relaxed(&eval_breaker)) {
            if (_Py_OPCODE(*next_instr) == SETUP_FINALLY ||
                    _Py_OPCODE(*next_instr) == YIELD_FROM) {
                goto fast_next_opcode;
            }
            if (_Py_atomic_load_relaxed(&pendingcalls_to_do)) {
                if (Py_MakePendingCalls() < 0)
                    goto error;
            }

            // 如果检测到gil_drop_request，释放GIL
            if (_Py_atomic_load_relaxed(&gil_drop_request)) {
                /* Give another thread a chance */
                if (PyThreadState_Swap(NULL) != tstate)
                    Py_FatalError("ceval: tstate mix-up");
                drop_gil(tstate);

                // 继续去调用GIL抢占
                take_gil(tstate);

                // 检查是否退出线程
                if (_Py_Finalizing && _Py_Finalizing != tstate) {
                    drop_gil(tstate);
                    PyThread_exit_thread();
                }

                if (PyThreadState_Swap(tstate) != NULL)
                    Py_FatalError("ceval: orphan tstate");
            }
        }
        ...
    }
}
```

当检测到 eval_breaker、gil_drop_request 时，会被动的释放 GIL，跟其他线程一起再次竞争 GIL，所以它几乎没有阻塞，虽然这狗GIL把你给锁住了，但是也保证了效率，当某个线程执行时间过长，可以迅速切换下一个线程调用。

类似这样

![](https://blog-1256169066.cos.ap-chengdu.myqcloud.com/Python%E7%BA%BF%E7%A8%8B/1.jpg)

## 如何绕过GIL锁？

如何绕过GIL锁，这个也是老生常谈的问题了。

在我这里的话，如果是我，涉及到需要绕过的地方

1. 我会直接写C代码，用CPython去调用
2. 我会用JPython
3. 我会用Golang

大概就这么几个方法，没别的方法了，再也不用看了

## Python多线程适用于哪些场景？

前面说了，Python多线程并不是真正意义的多线程，其实是个假的，但是还是有很多老哥不遗余力的推荐它，到底为啥，上一节也说得很清楚了，GIL虽然恶心，但是人家还是支持自动切换比较慢的阻塞线程的，不会影响其他线程运行，所以，你品一下，在咱们的开发岁月中，多线程，到底适用于哪些场景？

首先，发生阻塞的是什么情况，网络的连接时间过长，或者是处理多个业务，读取多个文件，访问多个网页等等。

网上很多文章都说： **I/O 密集场景，多线程最合适**

你抄我，我抄你，抄到最后就这么一句话，讲真，我看了那么多文章，真的没几个说清楚的，举个例子的都没。

那行嘛，那我来举例子吧。

首先，I/O密集型，指的是涉及到网络、磁盘IO的任务，现在互联网的web大部分都是IO密集的场景

举几个常用的自用例子吧

1. 网络爬虫，这个都写烂了
2. web应用，例如多用户访问，多用户登录，多用户下载
3. 数据库写入，Mysql/Es等等
4. RPC框架

大概就这些常用的，Python多线程，大概就这些比较合适。

## Python多线程的几个基础用例

说了那么多，最后还是加几个基础的用例，拿去举一反三，你们都是聪明人

### 基础的多线程框架

这个基础的多线程框架包含的功能

1. 启动/停止线程
2. 展示线程的状态

```python
import threading
import time

exitFlag = 0

# 利用继承的方法
class myThread (threading.Thread):

    def __init__(self, threadID, name, counter):
        threading.Thread.__init__(self)
        self.threadID = threadID
        self.name = name
        self.counter = counter

    def run(self):
        print ("开始线程：" + self.name)
        print_time(self.name, self.counter, 5)
        print ("退出线程：" + self.name)

def print_time(threadName, delay, counter):
    while counter:
        if exitFlag:
            threadName.exit()
        time.sleep(delay)
        print ("%s: %s" % (threadName, time.ctime(time.time())))
        counter -= 1

# 创建新线程
thread1 = myThread(1, "Thread-1", 1)
thread2 = myThread(2, "Thread-2", 2)

# 开启新线程
thread1.start()
thread2.start()

thread1.join()
thread2.join()
print ("退出主线程")
```


### 多线程的线程通信框架（利用队列）

有些时候我们需要在线程之间交换信息和数据，所以多线程之间需要互相通信

多线程的线程通信框架包含的功能

1. 利用队列共享数据（queue）
2. 创建一个生产者消费者模型，启动生产者，消费者线程

这种框架同样适用于爬虫

```python
import threading
import time
from queue import Queue


queue = Queue(20)


# 生产者
def Producer():
    i = 0
    print("开始线程")
    while True:
        i = i + 1
        print("生产数据", i, "现有数据", queue.qsize())
        time.sleep(1)
        queue.put(i)

# 消费者
def Consumer():
    while True:
        i = queue.get()
        time.sleep(0.5)
        print("消费数据", i)


if __name__ == "__main__":
    Th1 = threading.Thread(target=Producer, )

    Th2 = threading.Thread(target=Consumer, )

    Th2.start()
    Th1.start()

    Th1.join()
    Th2.join()
```
大概就是这样，这个害挺简单的。

### 线程锁框架

当需要修改共享数据的时候，多个线程会造成冲突，所以对执行的部分进行加锁很有必要，这里采用互斥锁逻辑

线程池加锁框架大概实现的功能

1. 在程序中加锁，避免竞争冲突

这部分代码我懒得写了，直接用了
https://www.cnblogs.com/tashanzhishi/p/10775641.html

这位老兄的代码写的很好，看一遍就懂了，感恩!

```python
import threading
import time

count = 0


# 做一个累加的函数
def add_num():
    global count
    if lock.acquire():  # 获得锁，并返回True
        tmp = count
        time.sleep(0.001)
        count = tmp + 1
        lock.release()  # 执行完释放锁


def run(add_fun):
    global count
    thread_list = []

    # 累加100次
    for i in range(100):
        t = threading.Thread(target=add_fun)
        t.start()
        thread_list.append(t)  

    # 等待线程执行完
    for j in thread_list:
        j.join()

    print(count)


if __name__ == '__main__':
    # 添加全局锁
    lock = threading.Lock()
    # 执行函数
    run(add_num)
```

### 线程池框架

线程池相当于冰箱里的啤酒

你只要想喝打开冰箱拿就行

你不喝人家也不会跑

就在冰箱里，不吵不闹

等待你的下一次临幸

```python
from socket import AF_INET, SOCK_STREAM, socket
from concurrent.futures import ThreadPoolExecutor


# 假装跑一个服务器client
def echo_client(sock, client_addr):
    '''
    Handle a client connection
    '''
    print('Got connection from', client_addr)
    while True:
        msg = sock.recv(65536)
        if not msg:
            break
        sock.sendall(msg)
    print('Client closed connection')
    sock.close()


# 假装跑一个server
def echo_server(addr):
    # 定义一个线程池
    pool = ThreadPoolExecutor(128)
    sock = socket(AF_INET, SOCK_STREAM)
    sock.bind(addr)
    sock.listen(5)
    while True:
        client_sock, client_addr = sock.accept()
        # 从池子里拿线程消费
        pool.submit(echo_client, client_sock, client_addr)

echo_server(('',15000))
```

## 结尾

线程这块，基本核心的东西都弄完了，其实梳理完，觉得还好，最起码我弄明白人家是干嘛的了，以后碰到任何线程的问题，我也不害怕了，这是12月第一篇blog，我要坚持三四个月，直到黎明的到来。

end。
