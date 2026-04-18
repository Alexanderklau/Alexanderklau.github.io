---
title: Golang封装Elasticsearch常用功能
date: 2020-05-14 09:45:49
tags: ["前后端"]
---

## 前言（为什么要写这篇文章）

首先看过我博客的都应该知道，我去年发了一篇Python封装Elasticsearch的文章。但那是去年了，今年我将我的检索服务后端用Golang全部重写了一波，相当于用Go重构了以前的Python代码，不过我个人感觉Golang的效率还是高于Python的，而且我还加了一些异常判断和处理，这次的代码只会比以前更好更牛逼，为了纪念这一个多月的重构历程，我将关键功能记录下来，方便自己复习和各位兄弟姐妹查看。

## 使用的Go包

我的Elasticsearch版本是6.3.2，6系列了，现在（2020-05-13）最新版本应该是7，不过新版本和旧版本应该就是少了Type，我的6版本代码，请各位自己自行斟酌使用。  
安装指定的Go包 olivere/elastic，现在有官方驱动的包了，但是我这篇文章用的包是 olivere/elastic，所以一切的代码都是以olivere为主。

```bash
go get github.com/olivere/elastic
```

## 基础的使用（Simple的使用）

接下来我举几个例子来说下这个golang 如何驱动 elasticsearch的

### 连接Elasticserach

```golang
package elasticdb

import (
	"context"
	"fmt"

	"github.com/olivere/elastic"
)

func main() {
    //连接127.0.0.1
    client, err := elastic.NewClient(elastic.SetURL("http://127.0.0.1:9200"))
    if err != nil {
        fmt.Println(err)
        return
    }
    //检查健康的状况，ping指定ip，不通报错
    _, _, err = client.Ping(ip).Do(context.Background())
    if err != nil {
        fmt.Println(err)
        return
    }
}

```

### 创建一个index索引

这里我默认你们都有一定的Es基础，其实你把index想成Mysql里面的表就可以了。

```golang
//indexname 你可以想成表名
func CreateIndex(indexname string) {
    client, err := elastic.NewClient(elastic.SetURL("http://127.0.0.1:9200"))
    if err != nil {
        fmt.Println(err)
        return
    }
    client.CreateIndex(indexname).Do(context.Background())
}
```

### 删除一个index索引

想成删除一张表

```golang
//indexname 你可以想成表名
func CreateIndex(indexname string) {
    client, err := elastic.NewClient(elastic.SetURL("http://127.0.0.1:9200"))
    if err != nil {
        fmt.Println(err)
        return
    }
    client.DeleteIndex(indexname).Do(context.Background())
}
```

### 往指定的index当中导一条数据

想成往一张表里面导入一条数据，在Golang中，我们可以导入json的字符串，我们也可以导入golang的struct类型，例如  

```golang
//结构体
type Task struct {
    Taskid  string  `json:"taskid"`
    Taskname  string  `json:"taskname"`
}
//字符串
jsonmsg := `{"taskid":"123456", "taskname":"lwb"}`
```

```golang
//导入数据，你需要index名，index的type，导入的数据
func PutData(index string, typ string, bodyJSON interface{}) bool {
    client, _ := elastic.NewClient(elastic.SetURL("http://127.0.0.1:9200"))
	_, err := client.Index().
		Index(index).
		Type(typ).
		BodyJson(bodyJSON).
		Do(context.Background())
	if err != nil {
		//验证是否导入成功
		fmt.Sprintf("<Put> some error occurred when put.  err:%s", err.Error())
		return false
	}
	return true
}
func main() {
    //json字符串导入
    jsonmsg = `{"taskid":"123456", "taskname":"hahah"}`
    status := PutData("test", "doc", jsonmsg)
    //struct结构体导入
    task := Task{Taskid: "123", Taskname: "hahah"}
    status := PutData("test", "doc", task)
}
```

### 删除一条数据

删除数据需要ID，这个ID是个啥玩意儿呢。。。就是咱们不是刚导了一条数据进去么，你可以设置这数据的唯一ID，也可以让Elasticsearch帮你自动生成一个，一般没事儿干谁自己设置啊，还容易重复，一重复就报错。。我在这里把这个删除的方法教给大家，记住这个ID一定是唯一的

```golang
func DeleteData(index, typ, id string) {
    client, _ := elastic.NewClient(elastic.SetURL("http://127.0.0.1:9200"))
    _, err := client.Delete().Index(index).Type(typ).Id(id).Do(context.Background())
    if err != nil {
        fmt.Println(err)
        return
    }
}
```

## 高阶使用（条件查询／封装等）

简单讲了一下增删改，现在我们来讲一下高阶用法，高级增删改查吧，其实官方的文档讲的还算是比较清楚，不过我等大中华程序狗的姿势水平。。。至少我是图样图森破，看英文也算是废了九牛二虎之力，算是捋出来了一些高阶用法，顺手自己造了个轮子，现在我就来挑几点来说下吧。

### 自动选择可用的Es节点

配合olivere的ping机制，可以做一个自动检测Es ip 是否可用的逻辑，这样可以增加我们put，updatge时候的稳定性

#### 自动检测节点Elasticseach的IP是否可用

olivere的Elasticserach sdk 限定了只能用一个ip，类似“http://127.0.0.1:9200”这样，我对原本的逻辑进行了一点改造，改成支持一个ip list，依次检测Es ip 是否可用

```golang
package elasticdb

import (
	"context"
	"fmt"

	"github.com/olivere/elastic"
)

//Elastic es的连接，
type Elastic struct {
	Client *elastic.Client
	host   string
}

//Connect 基础的连接代码
func Connect(ip string) (*Elastic, error) {
    //引入IP
	client, err := elastic.NewClient(elastic.SetURL(ip))
	if err != nil {
		return nil, err
    }
    //Ping 的方式检查是否可用
	_, _, err = client.Ping(ip).Do(context.Background())
	if err != nil {
		return nil, err
    }
    //输出一个struct类型，可以被继承
	es := &Elastic{
		Client: client,
		host:   ip,
	}
	return es, nil
}

//InitES 初始化Es连接
func InitES() (*Elastic, error) {
    //host是一个列表
    host := []string{"http://10.0.6.245:9200","http://10.0.6.246:9200","http://10.0.6.247:9200"}
    //统计host的数量
    Eslistsnum := len(host)
    //如果为零就不继续接下来的逻辑
	if Eslistsnum == 0 {
		return nil, fmt.Errorf("Cluster Not Es Node")
	}
    //创建新的连接
	for i, ip := range host {
		//判断是不是最后一个节点ip
		if (Eslistsnum - 1) != i {
			es, err := Connect(ip)
			//如果连接出错，则跳过
			if err != nil {
				fmt.Println(err)
				continue
			}
            return es, nil
            //如果是最后一个节点
		} else {
            es, err := Connect(ip)
            //输出错误
			if err != nil {
				return nil, err
			}
			return es, nil
		}
	}
	return nil, nil
}

```

后续我们可以采用继承的方法调用Es的client的连接，这个在后面我就不详细说了，聪明的你，看代码一定能整明白，再整不明白，你就直接上Git拷贝我的代码就得了。

### 条件查询

以前我写过一个Python的Elasticsearch Sdk，那里面的查询基本都用了query，简单来说，就是你，给Es的api发一个query，es给你返回一个查询结果。这里我会举几个常用的条件查询例子，然后用golang封装一波。  
这里我先定义一下数据结构，假设我们的Elasticsearch中，有一个叫做Task的index(索引)，其中存储着很多task的运行日志，它们的数据格式如下:  
```json
{
    "taskid": "081c255b-936c-11ea-8001-000000000000",
    "starttime": "2020/05/13 18:38:21",
    "endtime": "2020/05/13 18:38:47",
    "name": "cifs01",
    "status": 1,
    "count": 365
}
```
我们现在要做的就是围绕task这个index和其中的数据做条件查找的例子，我说的很明白了吧？开工了！

### 查询时间范围/年龄大小的条件查询方法
在业务需求中，我们经常会检索各种各样的数据，其中，范围查找应该是用的比较多的，所以我把它放到了最前面。

```golang
type Task struct {
	TaskID    string `json:"taskid"`
	StartTime string `json:"starttime"`
	EndTime   string `json:"endtime"`
	Name      string `json:"name"`
	Status    int    `json:"status"`
	Count     int    `json:"count"`
}
//查找时间范围大于2020/05/13 18:38:21，并且小于2020/05/14 18:38:21的数据
func (Es *Elastic) FindTime() {
    var typ Task
    boolQ := elastic.NewBoolQuery()
    //生成查询语句，筛选starttime字段，查找大于2020/05/13 18:38:21，并且小于2020/05/14 18:38:21的数据
    boolQ.Filter(elastic.NewRangeQuery("starttime").Gte("2020/05/13 18:38:21"), elastic.NewRangeQuery("starttime").Lte("2020/05/14 18:38:21"))
    res, _ := Es.Client.Search("task").Type("doc").Query(boolQ).Do(context.Background())
        //从搜索结果中取数据的方法
        for _, item := range res.Each(reflect.TypeOf(typ)) {
            if t, ok := item.(Task); ok {
                fmt.Println(t)
            }
        }
}
```
### 查询包含关键字的查询方法

我们经常遇到那种，搜那么一两个字，让你展示所有包含这一两个字的结果，就像百度，你搜个"开发"，就能搜出来例如"软件开发","硬件开发"等。接下来咱们也实现一个这个功能

```golang
//查找包含"cifs"的所有数据
func (Es *Elastic) FindKeyword() {
    //因为不确定cifs如何出现，可能是cifs01，也可能是01cifs，所以采用这种方法
    keyword := "cifs"
    keys := fmt.Sprintf("name:*%s*", keyword)
    boolQ.Filter(elastic.NewQueryStringQuery(keys))
     res, _ := Es.Client.Search("task").Type("doc").Query(boolQ).Do(context.Background())
        //从搜索结果中取数据的方法
        for _, item := range res.Each(reflect.TypeOf(typ)) {
            if t, ok := item.(Task); ok {
                fmt.Println(t)
            }
        }
}
```

### 多条件查询

如果说我们现在不仅仅需要找到符合时间的，也需要找到符合关键字的查询，那么就需要在查询条件上做文章。

```golang
func (Es *Elastic) FindAll() {
    //因为不确定cifs如何出现，可能是cifs01，也可能是01cifs，所以采用这种方法
    keyword := "cifs"
    keys := fmt.Sprintf("name:*%s*", keyword)
    boolQ.Filter(elastic.NewRangeQuery("starttime").Gte("2020/05/13 18:38:21"), elastic.NewRangeQuery("starttime").Lte("2020/05/14 18:38:21"), elastic.NewQueryStringQuery(keys))
	res, err := Es.Client.Search("task").Type("doc").Query(boolQ).Do(context.Background())
        //从搜索结果中取数据的方法
        for _, item := range res.Each(reflect.TypeOf(typ)) {
            if t, ok := item.(Task); ok {
                fmt.Println(t)
            }
        }
}
```

### 统计数量/多条件统计数量

有些时候我们需要去统计符合查询条件的结果数量，做统计用，这里也有直接可用的Sdk

```golang
func (Es *Elastic) GetTaskLogCount() (int, error) {
	boolQ := elastic.NewBoolQuery()
	boolQ.Filter(elastic.NewRangeQuery("starttime").Gte("2020/05/13 18:38:21"), elastic.NewRangeQuery("starttime").Lte("2020/05/14 18:38:21"))
	//统计count
	count, err := Es.Client.Count("task").Type("doc").Query(boolQ).Do(context.Background())
	if err != nil {
		return 0, nil
	}
	return int(count), nil
}
```

## 总结
我这边完成了几个查询/导入的基础功能，当然，我的代码大部分都放置在了github当中  
放置在: https://github.com/Alexanderklau/Go_poject/tree/master/Go-Elasticdb/Elasticsearch_sdk  
最近项目比较忙，我打算月中写一篇我开发的时候使用的一些Go特性，或者高级用法。如果喜欢的话麻烦Star我！最近压力颇大，想要换一个地方生活，所以也要准备离开了。如果大家有什么问题，可以直接给我提问，我看到了就会帮助大家的。