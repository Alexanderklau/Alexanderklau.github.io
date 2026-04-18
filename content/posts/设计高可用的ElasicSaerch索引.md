---
categories: ["技术", "Elasticsearch"]
title: 设计高可用的ElasicSearch索引
date: 2021-09-09 15:36:23
tags: ["前后端"]
---

## 前言

最近又是学习的爆发期，我戒除了上班划水和看知乎故事会，也戒除了游戏和小说

也戒除了频繁参加外面的无用活动，也逐渐修复了寂寞侵蚀内心的恐惧

将心思逐渐稳定下来，所以，我本月开始到年底，将会爆发式更新blog和歌曲

请大家和我一起学习吧！对了，如果有机会，请不要忘记帮我看看是否有合适的工作

我最近在换工作，走过路过，也别忘记我找份工作。

开始！

```
从不去关心他人的定位

对我喜欢讨厌或者是敬畏

始终在不停的韬光养晦

为我喜欢的事鞠躬尽瘁

妈妈问我每天是否疲惫

生活似好似坏前路隐晦

希望的濒危物种不令人敬佩

只想和路过的你说声幸会
```


## 当我们在说Es索引设计时，我们在说什么？

在我的开发生涯中，我使用过Mysql，Etcd，Mongodb之类的数据库

当我拿到一个数据库要进行开发任务的时候，我的第一件事往往就是查看数据结构，或者是查看表结构

如果是Mysql一类的数据库，数据表中，表的字段类型是否合适，表设计是否合理，涉及到联合查询的机制是否完全？这往往就是数据库表设计的核心。

嵌套到es当中，es的索引，你也可以理解为表

当我们要设计索引的时候，我们就是设计表

当我们说要设计高可用的的索引时，往往就是指索引

1. 可支撑大数据量
2. 性能影响小
3. 结合业务场景，全维度考虑增，删，改，查


## ElasticSearch索引的基础知识

索引，这个东西在数据库里面就是帮助加快检索速度的

它是一种数据结构，这个我原来提过一点点

具体的地址，请参看

[ElasticSearch检索的核心-倒排索引解读](https://yemilice.com/2021/05/14/elasticsearch%E6%A3%80%E7%B4%A2%E7%9A%84%E6%A0%B8%E5%BF%83-%E5%80%92%E6%8E%92%E7%B4%A2%E5%BC%95%E8%A7%A3%E8%AF%BB/)

今天我们的主题并不是原理，而是如何设计索引

如何设计索引才能让我们的ES更加健壮，避免后期存入数据过多后还要更改结构

这个才是这篇文章的核心。

所以基础知识这里，我将不会表述太多，请大家见谅。

默认你已经会用Es，就这样，Over。

## Mapping的设计

### Mapping是什么?

Mapping在Es里面，相当于表结构，我举个例子

```json
"mappings": {
	"doc": {
		"properties": {
			"name": {
				"ignore_above": 256,
				"type": "keyword"
			},
			"id": {
				"type": "keyword"
			},
			"size": {
				"type": "long"
			},
			"last_mod_time": {
				"format": "yyyy-MM-dd HH:mm:ss",
				"type": "date"
			}
		}
	}
}
```

1. 我们定义的mapping里面有若干个字段，name，id...
2. 类型不同，有long，keyword，date

Mapping在索引设计中是非常重要部分，就和表设计一样的，你需要定义字段

一个好的字段能让我们的检索速度飞快，一个差的字段会让我们浪费很多平白无故的存储空间

首先Mapping的基础知识可以参看这篇文章

[一文搞懂 Elasticsearch 之 Mapping](https://www.cnblogs.com/wupeixuan/p/12514843.html#4667500)

这篇文章的大佬把Mapping说的很清楚了

Mapping 分为两种，静态Mapping和动态Mapping

下面我分别说下静态和动态的几个优缺点

### 静态Mapping

事先定义好字段，也就是定义好基础的Mapping格式，就像章节最开始我举的例子。

静态Mapping，就相当于我们创建Mysql表，事先建表，规定好所有的字段和类型

静态Mapping的优点如下

1. 可选可控
2. 节约存储空间

静态Mapping的缺点如下

1. 设计要求高，后期无法进行更改
2. 将数据格式全部规定死，不够动态

### 动态Mapping

我们不创表，直接传入数据，Es会根据你传入的数据自动匹配合适的类型，也就是自动识别数据

说白了，就是你，传个json过来，Es根据你的json自动匹配类型，然后创建Mapping

动态Mapping这东西，其实我是不推荐用的，因为相当于，你把数据类型的判断逻辑交给了Es，这是很不可取的，因为Es，很傻，说下原因

1. 当你不想检索一些字段的时候，你可以通过设置mapping事先指定，但是动态的不行
2. 动态Mapping可控性没那么高，你往里导数据，要保证数据的一致性



### 高可用Mapping设计的流程

查了一些资料，其实大家可以看一下铭毅天下这个老哥的公众号

他的公众号给了我很大启发！

[铭毅天下-死磕Elasticsearch方法论](https://github.com/mingyitianxia/deep_elasticsearch/wiki/%E3%80%8A%E6%AD%BB%E7%A3%95Elasticsearch%E6%96%B9%E6%B3%95%E8%AE%BA%E3%80%8B%E2%80%94%E2%80%94%E9%93%AD%E6%AF%85%E5%A4%A9%E4%B8%8B%E5%87%BA%E5%93%81)


首先，设计Mapping，需要预先考虑如下几点

1. 是否需要进行全文检索
2. 是否需要排序
3. 数据类型的多重选择

首先，先将Mapping里的几个参数及其基础设置梳理

| 参数           | 具体说明                   | 传入参数   |
| -------------- | -------------------------- | ---------- |
| enabled        | 设置是否需要检索           | True/False |
| index          | 设置是否构建倒排索引       | True/False |
| index_option   | 设置存储倒排索引的哪些信息 | True/False |
| doc_values     | 是否开启聚合分析           | True/False |
| dynamic        | 动态更新mapping            | True/False |
| data_detection | 自动识别日期类型           | True/False |


进行Mapping设计时，首先要考虑我们到底有多少字段，这些字段的属性是什么

这里非常重要，因为，当你创建完Index的时候，Es是不许更新表结构，也就是Mapping的，

如果要选择更新，则需要重建Index，当数据量很大的时候，这种方案肯定是不可以的。

所以，我这里定义的流程如下

#### 第一步：确定存储的数据类型

先细分析一波，先根据我们的需求，确定建立好我们的数据类型

一般的数据类型如下

![](https://blog-1256169066.cos.ap-chengdu.myqcloud.com/index/1.png)

根据我们的需求去梳理数据类型

这里提几点选择策略

1. 如果你有很多文字，不需要分词，选text，需要分词，选keyword
2. 如果你未来不确定你到底有多少数据，将类型设置为long是比较合适的
3. keyword检索比较短的字符会更快，如果字符很大，那么会增加存储和检索成本


#### 第二步：确定哪些字段需要检索

如果我们存储的一个json十分巨大，例如content是一篇文章，或者我们的Mapping字段非常的多，那么如果全部默认检索，那我们会搜出一大堆无用的东西

这样既增加了存储成本，又减弱了检索效率，所以下一步，我们就要根据需求，确定哪些字段需要检索

一些固定参数，或者不想给用户展示的参数也就没有检索价值

例如ID一类的参数

这里如果我们不想ID被检索，那么需要设置

```json
 "id":{
        "type": "text",
        "index": false
      }
```

#### 第三步：检索的方式

这里一般指的是分词的设置，这里需要指定分词器

分词器我以前的blog里面也讲过

[定义自己的分词器](https://yemilice.com/2020/12/08/elasticsearch%E5%AE%9A%E4%B9%89%E8%87%AA%E5%B7%B1%E7%9A%84%E5%88%86%E8%AF%8D%E5%99%A8%E8%AF%8D%E5%85%B8%EF%BC%88%E6%A3%80%E7%B4%A2%E7%94%9F%E5%83%BB%E8%AF%8D%EF%BC%89/)

这里相当于是，给指定的字段，添加分词

假如我们现在有一个字段，名字叫name，我们需要指定中文分词器IK

```json
 "name":{
        "type": "text",
		"analyzer": "ik_smart
      }
```

#### 第四步：指定mapping的特殊设置

这里的设置，不针对某个字段，而是针对整个mapping的setting设置

这里有如下几个常用设置

```json
{
	"settings": {
	// 分片数量
	"number_of_shards": 5,
	// 副本
	"number_of_replicas": 1,
	// 最好压缩
	"codec": "best_compression",
	// 最大展示条数
	"max_result_window": "100000000",
	// 刷新时间
	"refresh_interval":"30s"
}
```

一般都是这么几个设置

甭管那么多，直接上，不要怂

这张图是铭毅天下那里找来的！爱你！

这个流程，非常清楚了

![](https://blog-1256169066.cos.ap-chengdu.myqcloud.com/index/3.jpg)


### Mapping的模版

这里给出我自己开发时候的一个模版，这个模版扛住了千万数据量，直接上，不要怕

大家拿过来改改字段就行

```json
{
	"settings": {
	  "number_of_shards": 5,
	  "number_of_replicas": 1,
	  "codec": "best_compression",
	  "max_result_window": "100000000",
	  "refresh_interval":"30s"
	},
	"mappings": {
	  "doc": {
		"properties": {
		  "name": {
			"type": "keyword"
		  },
		  "id": {
			"type": "keyword"
		  },
		  "data": {
			"type": "nested",
			"properties": {
			  "value": {
				"type": "text",
				"fields": {
				  "keyword": {
					"ignore_above": 256,
					"type": "keyword"
				  }
				}
			  },
			  "key": {
				"type": "text",
				"fields": {
				  "keyword": {
					"ignore_above": 256,
					"type": "keyword"
				  }
				}
			  }
			}
		  },
		  "size": {
			"type": "long"
		  },
		  "last_mod_time": {
			"format": "yyyy-MM-dd HH:mm:ss||yyyy-MM-dd || epoch_millis",
			"type": "date"
		  },
		  "user": {
			"type": "keyword"
		  },
		  "content": {
			"analyzer": "ik_smart",
			"term_vector": "with_positions_offsets",
			"type": "text"
		  }
		}
	  }
	}
}
```

## 大数据下索引的切分

当我通过filebeat或者logstash倒入数据到Es中的时候

传输到Es中，创建的index是可以自己控制的

一般的划分方法是：

1. 年
2. 月
3. 日
4. 自定义

举例来说

假如我们的log索引，名字叫"log-2021-09"

那么每天的数据量都有千万，那么这个索引就会越来越大，越来越大

你做检索的时候，就会在一个非常大的索引中进行

这样检索的时候，卡，慢就出现了

万一这索引坏了，那。。。丢数据吧

所以，如何进行切分，或者优化索引，势在必行

下面我这里将给出一些我自己实战的手法，给大家一些建议。

### 动态创建索引

首先，上一节我们讲了Mapping模版，这里我就不再多说

我们需要结合模版去动态创建索引

方法如下

#### rollver滚动创建索引

现在开始设定 log-2021-09 这个索引

这步的意思是创一个索引，名字叫做log-2021-09-00001，别名为logs_write

```json
curl -XPUT 'localhost:9200/log-2021-09-00001 ?pretty' -d'
{
  "aliases": {
    "logs_write": {}
  }
}'
```

现在开始设置动态索引创建

这步的意思是，设定logs_write，当天数大于7天，或者文档数量大于100000，或者大小大于5gb，创建一个新的log-2021-09-00002 索引

```json
POST /logs_write/_rollover 
{
  "conditions": {
    "max_age":   "7d",
    "max_docs":  100000,
    "max_size":  "5gb"
  }
}
```


### 定时清理过期数据

#### Curator索引管理工具
这个工具，可以进行索引管理

安装的方法，网上找RPM包直接安就得了

https://www.elastic.co/guide/en/elasticsearch/client/curator/current/yum-repository.html


它的功能有

1. 关闭索引
2. 创建快照
3. 创建索引
4. 打开索引
.....

这里，我们可以通过Curator来进行索引创建，定时配置删除等等

例如我们要动态删除7天前的索引

```yml
# Remember, leave a key empty if there is no value.  None will be a string,
# not a Python "NoneType"
#
# Also remember that all examples have 'disable_action' set to True.  If you
# want to use this action as a template, be sure to set this to False after
# copying it.
actions:
  1:
    action: delete_indices　　　　　　　　# 这里执行操作类型为删除索引
    description: >-
      Delete metric indices older than 3 days (based on index name), for zou_data-2018-05-01
      prefixed indices. Ignore the error if the filter does not result in an
      actionable list of indices (ignore_empty_list) and exit cleanly.
    options:
      ignore_empty_list: True
    filters:
    - filtertype: pattern
      kind: prefix
      value: logs-　　　　　　　　# 这里是指匹配前缀为 “order_” 的索引，还可以支持正则匹配等，详见官方文档
    - filtertype: age　　　　　　　　　　　 # 这里匹配时间
      source: name　　　　　　　　　　　　　 # 这里根据索引name来匹配，还可以根据字段等，详见官方文档
      direction: older
      timestring: '%Y-%m-%d'　　　　　　　 # 用于匹配和提取索引或快照名称中的时间戳
      unit: days　　　　　　　　　　　　　　　# 这里定义的是days，还有weeks,months等，总时间为unit * unit_count
      unit_count: 7
```

以上命令删除了7天前，以log-*开头的索引

可以看下这篇文章

[ElasticSearch——Curator索引管理](https://www.cnblogs.com/caoweixiong/p/11995525.html)



## 结尾

这些大概就是我实战遇到的一些问题，结合网上的一些资料

我输出了这篇blog

最近我开始疯狂学习，希望未来会有突破吧

狗公司不发年终，淦，还是赶紧找机会跑路，才是最重要的。

![](https://blog-1256169066.cos.ap-chengdu.myqcloud.com/index/2.jpg)

大家加油吧！写算法Ing！

加油！
