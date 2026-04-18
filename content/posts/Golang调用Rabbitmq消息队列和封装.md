---
title: Golang调用Rabbitmq消息队列和封装
date: 2020-04-13 10:11:00
tags: ["前后端"]
---
## 前言
### 介绍Rabbimq
#### Rabbitmq消息队列是干嘛的？
简单的说，消息队列，引申一下就是传递消息用的队列，也可以称为传递消息的通信方法。用争抢订单的快车举个例子，假如，A用户发送了一个用车的消息，那么消息队列要做的就是把A用户用车的这个消息广而告之，发送到一个公用队列当中，司机只管取到消息，而不管是谁发布的，这就是一个简单的消息队列例子，Rabbitmq其实就是消息队列的一种，用的比较多的还可能有Redis，kafka，ActiceMq等等，这个后面的博文里面我会说，这次我们只说Rabbimq消息队列
#### Rabbitmq消息队列的好处是什么？为什么我们要用他？
这个网上有很多类似的玩意，我不说太多，就只说我在使用中感觉比较好的地方。
1. 分布式，多节点部署。一个集群，保证消息的持久化和高可用，某节点挂了，其他节点可以结力。

2. 路由Exchange，这个已经提供了内部的几种实现方法，可以指定路由，也就是指定传递的地址。
3. 多语言支持，我以前干活儿用Python，现在用Go和java，人家无缝对接，多牛逼！

4. Ack的消息确认机制，这样就保证了，任务下发时候的稳定性，ack消息确认可以手动，也可以自动，这样就保证了任务下发时候的可控和监控。

## 初步开始
### 简单的生产者和消费者的模型
讲那么多废话理论，还不如直接开始写代码更直观是吧，所以，奥莉给，干了兄弟们！我们实现一个简答的生产者，消费者模型。这个不用我多解释吧，基础的流程就是，我们定义一个生产者，生产信息到Rabbitmq中，然后再定义一个消费者，把数据从Rabbitmq中取出来，就这么简单，下面咱们就干了，先讲几个基础。
### Rabbitmq的基础知识
#### 发送 Publish
发送，你可以理解为上传，意思就是，上传一个消息到Rabbitmq当中。它这块的基础代码比较简单

```golang
package main

import (
  "log"
  "github.com/streadway/amqp"
)

func main() {
    //初始化一个Rabbimtq连接，后跟ip，user，password
    conn, err := amqp.Dial("amqp://guest:guest@localhost:5672/")
    if err != nil {
        return
    }
    defer conn.Close()
    //创建一个channel的套接字连接
    ch, _ := conn.Channel()
    //创建一个指定的队列
    q, _ := ch.QueueDeclare(
        "work", // 队列名
        false,   // durable
        false,   // 不使用删除？
        false,   // exclusive
        false,   // 不必等待
        nil,     // arguments
    )
    //定义上传的消息
    body := "work message"
    //调用Publish上传消息1到指定的work队列当中
    err = ch.Publish(
        "",     // exchange
        "work", // 队列名
        false,  // mandatory
        false,  // immediate
        amqp.Publishing {
                ContentType: "text/plain",
                //[]byte化body
                Body:        []byte(body),
        })
}
```

这样就完成了上传消息到work队列当中。
#### 接收 Consume
接收，顾名思义，就是接收到指定队列中的信息，信息存在队列当中，总要被拿出来用吧，放那里又不能下崽儿，所以，拿出来感觉用了才是最重要的。这块的基础代码如下

```golang
package main

import (
  "log"
  "github.com/streadway/amqp"
)

func main() {
    //初始化一个Rabbimtq连接，后跟ip，user，password
    conn, err := amqp.Dial("amqp://guest:guest@localhost:5672/")
    if err != nil {
        return
    }
    defer conn.Close()
    //创建一个channel的套接字连接
    ch, _ := conn.Channel()

    msgs, err := ch.Consume(
        "work" // 队列名
        "",     // consumer
        true,   // auto-ack
        false,  // exclusive
        false,  // no-local
        false,  // 不等待
        nil,    // args
    )
    //定义一个forever，让他驻留在后台，等待消息，来了就消费
    forever := make(chan bool)

    //执行一个go func 完成任务消费
    go func() {
        for d := range msgs {
            //打印body
            log.Printf("message %s", d.Body)
        }
    }()
    <-forever
}
```
### 生产者／消费者模型
上面简单说了一下rabbimq的发送和接收，这下咱们就要实现一个生产者消费者模型了，这个模型的主要逻辑，就是生产者发送任务到指定的队列，有一个，或者多个消费者，会在此留守，一有任务，就争抢并且消费。
#### 生产者逻辑 
其实生产者逻辑和上面的发送逻辑差不多，这里给出写法。

```golang
package main

import (
  "log"
  "github.com/streadway/amqp"
)

func main() {
    //初始化一个Rabbimtq连接，后跟ip，user，password
    conn, err := amqp.Dial("amqp://guest:guest@localhost:5672/")
    if err != nil {
        return
    }
    defer conn.Close()
    //创建一个channel的套接字连接
    ch, _ := conn.Channel()
    //创建一个指定的队列
    q, _ := ch.QueueDeclare(
        "work", // 队列名
        false,   // durable
        false,   // 不使用删除？
        false,   // exclusive
        false,   // 不必等待
        nil,     // arguments
    )
    //定义上传的消息
    body := "work message"
    //调用Publish上传消息1到指定的work队列当中
    err = ch.Publish(
        "",     // exchange
        "work", // 队列名
        false,  // mandatory
        false,  // immediate
        amqp.Publishing {
                ContentType: "text/plain",
                //[]byte化body
                Body:        []byte(body),
        })
}
```
#### 消费者逻辑
消费者逻辑这边，主要是加了一个qos控制和手动ack，代码如下

```golang
package main

import (
  "log"
  "github.com/streadway/amqp"
)

func main() {
    //初始化一个Rabbimtq连接，后跟ip，user，password
    conn, err := amqp.Dial("amqp://guest:guest@localhost:5672/")
    if err != nil {
        return
    }
    defer conn.Close()
    //创建一个channel的套接字连接
    ch, _ := conn.Channel()
    
    //创建一个qos控制
    err = ch.Qos(
            3,     // 同时最大消费数量（意思就是最多能消费几个任务）
            0,     // prefetch size
            false, // 全局设定？
        )
    if err != nil {
            return err
            }
    msgs, err := ch.Consume(
        "work" // 队列名
        "",     // consumer
        true,   // auto-ack
        false,  // exclusive
        false,  // no-local
        false,  // 不等待
        nil,    // args
    )
    //定义一个forever，让他驻留在后台，等待消息，来了就消费
    forever := make(chan bool)

    //执行一个go func 完成任务消费
    go func() {
        for d := range msgs {
            //打印body
            log.Printf("message %s", string(d.Body))
            //手动ack，不管是否发送完毕。
            d.Ack(false)
        }
    }()
    <-forever
}
```
## Golang封装Rabbitmq的基础接口
Rabbitmq会用了吧，上面那个估计比较简单，但是估摸着你们还想要别的功能，好，那我就惯大家一次，干了兄弟们，奥莉给！
### 初始化Rabbitmq连接
为了避免每次重复调用Rabbitmq连接，我这里提供一个简单写法。
```golang
package main

import (
"context"
"fmt"

"github.com/streadway/amqp"
)

//Rabbitmq 初始化rabbitmq连接
type Rabbitmq struct {
        conn *amqp.Connection
        err  error
}

func New(ip string) (*Rabbitmq, error) {
        amqps := fmt.Sprintf("amqp://guest:guest@%s:5672/", ip)
        conn, err := amqp.Dial(amqps)
        if err != nil {
            return nil, err
        }
        rabbitmq := &Rabbitmq{
            conn: conn,
        }
        return rabbitmq, nil
}
```

### 创建一个Queue队列

```golang
func (rabbitmq *Rabbitmq) CreateQueue(id string) error {
        ch, err := rabbitmq.conn.Channel()
        defer ch.Close()
        if err != nil {
            return err
        }
        _, err = ch.QueueDeclare(
            id,    // name
            true,  // durable
            false, // delete when unused
            false, // exclusive
            false, // no-wait
            nil,   // arguments
        )
        if err != nil {
            return err
        }
        return nil
}
```

### 上传消息到指定的queue中

```golang
func (rabbitmq *Rabbitmq) PublishQueue(id string, body string) error {
        ch, err := rabbitmq.conn.Channel()
        defer ch.Close()
        if err != nil {
            return err
        }
        err = ch.Publish(
            "",    // exchange
            id,    // routing key
            false, // mandatory
            false,
            amqp.Publishing{
                DeliveryMode: amqp.Persistent,
                ContentType:  "text/plain",
                Body:         []byte(body),
            })
        if err != nil {
            return err
        }
        return nil
}
```

### 从队列中取出消息并且消费

```golang
func (rabbitmq *Rabbitmq) PublishQueue(id string, body string) error {
        ch, err := rabbitmq.conn.Channel()
        defer ch.Close()
        if err != nil {
            return err
        }
        err = ch.Publish(
            "",    // exchange
            id,    // routing key
            false, // mandatory
            false,
            amqp.Publishing{
                DeliveryMode: amqp.Persistent,
                ContentType:  "text/plain",
                Body:         []byte(body),
            })
        if err != nil {
            return err
        }
        return nil
}
```

### 统计队列中预备消费的数据

```golang
func (rabbitmq *Rabbitmq) GetReadyCount(id string) (int, error) {
        count := 0
        ch, err := rabbitmq.conn.Channel()
        defer ch.Close()
        if err != nil {
            return count, err
        }
        state, err := ch.QueueInspect(id)
        if err != nil {
            return count, err
        }
        return state.Messages, nil
    }
```

### 统计消费者／正在消费的数据

```golang
func (rabbitmq *Rabbitmq) GetConsumCount(id string) (int, error) {
        count := 0
        ch, err := rabbitmq.conn.Channel()
        defer ch.Close()
        if err != nil {
            return count, err
        }
        state, err := ch.QueueInspect(id)
        if err != nil {
            return count, err
        }
        return state.Consumers, nil
}
```

### 清理队列

```golang
func (rabbitmq *Rabbitmq) ClearQueue(id string) (string, error) {
        ch, err := rabbitmq.conn.Channel()
        defer ch.Close()
        if err != nil {
            return "", err
        }
        _, err = ch.QueuePurge(id, false)
        if err != nil {
            return "", err
        }
        return "Clear queue success", nil
}
```

## 总结
简单讲了一下Rabbimtq是啥，怎么用，我是怎么用的。  
完整代码请访问我的Github： https://github.com/Alexanderklau/Go_poject/blob/master/Go-Rabbitmq/rabbitmq.go   
如果有不懂的欢迎留言！如果能帮大家的我一定会帮！也希望你们指出我的错误！一起进步！ 