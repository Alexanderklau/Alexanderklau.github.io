---
categories: ["技术", "Elasticsearch"]
title: ElasticSearch高级用法(Golang实现)
date: 2020-12-02 14:38:21
tags: ["前后端"]
---


## 前言

前阵子开发了一个检索服务器，用了一些Es的高阶功能，网上随便看了一圈，基本没有人公开这些功能的中文版，我寻思咱们不能只满足了自己，不满足其他老哥吧，所以，我将开发中用到的Es高阶功能总结输出.

此次开发

我用的是

Golang 1.13.6 

Es的版本是6.3.2，

Package: "github.com/olivere/elastic"

请注意。

## 定义Mapping模板
首先，先展示一下我定义的mapping模板，这个模板是我们创建index之前定义的，这个十分重要！十分重要！十分重要！切记切记！

首先我直接展示一下我的mapping定义，接下来我会一点点的说明每个参数的含义

```json
var Contentmapping = `{
	"settings": {
	  "number_of_shards": 5,
	  "number_of_replicas": 1,
	  "codec": "best_compression",
	  "max_result_window": "100000000"
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
			"format": "yyyy-MM-dd HH:mm:ss",
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
  }`
```
mapping主字段，我定义了如下几个字段

| 字段          | 字段说明     | 字段mapping类型 | 字段举例                                     |
| ------------- | ------------ | --------------- | -------------------------------------------- |
| name          | 姓名         | keyword         | YK_example                                   |
| id            | id           | keyword         | 12345678                                     |
| user          | 用户         | keyword         | YK                                           |
| content       | 全文检索字段 | text            | 中国123......                                |
| size          | 大小         | long            | 14500                                        |
| data          | 嵌套数据     | nested          | [{key:123,value:456}, {key:789,value:101112} |
| last_mod_time | 最后修改时间 | date            | 2020-12-02 12:00:00                          |

这样是不是就说的很清楚了，下面我细致说一下mapping里面这种的设计方法吧

1. 首先是**keyword**类型，如果你的字段，有全检索的需求，也就是完全匹配的需求，你需要使用这个类型，但是keyword能完全检索的长度有限，也就是说，他只能完全匹配指定长度的数据，我查了一下，大概是2766个UTF-8字节数，所以超过这个数的将不会被检索到
2. 我这里面出现了**ignore_above**这个东西，这个东西是：最大可被检索字段 意思就是，当超过我定义的ignore_above的字符数的时候，多出来的将不会被检索到，这里是根据业务场景自己划分。
3. **long**类型，一般用来存储数字，这种情况，大部分都是存储文件大小之类的，一开始一定要用long定义，因为int类型大小超过100000会直接存不进去的。
4. **text**类型，这里一般都是存储那种特别大的文字数据的，比如你存了一个PDF进来，或者存了一本字典进来，就需要用text存储，text理论上支持存储无限大的文字数据。这里不想keyword，它不会被支持全文检索和精准匹配。需要定义分词器/在检索语句上下功夫
5. **nested**类型，这里一般是复杂嵌套，类似列表中包含字典的操作，[{key:123,value:456}，类似这样
6. **date**类型，这里一般是时间格式，你要自己格式化，"format": "yyyy-MM-dd HH:mm:ss",这种比对的就是：2020-12-02 12:00:00，如果你是2020-12-02 12:00:00:22，是肯定会导入失败的

大概就是这些，包含的部分也就足够你一般使用了，还有一些mapping的高级用法，后面我会单独写博客分析，本篇内容不是这个，此处进行跳过了。

## 定义分词器

上面说了，分词器是做全文检索的时候必须要用的，分词器的功能就是**能够让你对text字段进行检索**，text字段是不可被全文检索的，因为它的大小不定，但是，用了分词器能让你进行全文检索，就像这样

![](https://blog-1256169066.cos.ap-chengdu.myqcloud.com/es%E9%AB%98%E7%BA%A7%E7%94%A8%E6%B3%95/1.jpg)

所以，定义分词的的参数是

```json
"content": {
			"analyzer": "ik_smart",
			"term_vector": "with_positions_offsets",
			"type": "text"
		  }
```
加一个analyzer参数，指定某个字段使用ik分词器，这里可以使用

```
"analyzer": "ik_smart",
```

也可以使用

```
"analyzer": "ik_max_word",
```
这里根据你的业务自己选择

## 定义最大展示条数

一般ElasticSearch为了保证一次不拿取过多数据，会进行一个限制，限制最大不能读取10000条数据，但是我们有些时候因为一些特殊原因需要拿取超过10000条数据（翻页

这时候有两种方法

第一种直接通过Es的接口

```json
PUT
	_all/_settings
	{
	"index.max_result_window":200000
	}
```

这种好处就是随时随地可以改

第二种方法就是，定义mapping的时候，直接定义

```json
"settings": {
	  "max_result_window": "100000000"
	},
```

我个人还是比较喜欢第二种啦，因为事先定义好总比发现了问题再去调接口好一百倍。

## 根据模板创建index

mapping现在有了，咱们现在要做的就是，利用mapping创建一个index，涉及一些连接Es之类的，我在这就不说了，我以前写过一个Golang调用Es的接口，大家要不自己去瞅瞅？

```
https://yemilice.com/2020/05/14/golang%E5%B0%81%E8%A3%85elasticsearch%E5%B8%B8%E7%94%A8%E5%8A%9F%E8%83%BD/
```

直接调用一把代码

```golang
func (Es *Elastic) CreateIndex(index, mapping string) bool {
	// 判断索引是否存在
	exists, err := Es.Client.IndexExists(index).Do(context.Background())
	if err != nil {
		fmt.Printf("<CreateIndex> some error occurred when check exists, index: %s, err:%s", index, err.Error())
		return false
	}
	if exists {
		fmt.Printf("<CreateIndex> index:{%s} is already exists", index)
		return true
	}
	// 创建index
	createIndex, err := Es.Client.CreateIndex(index).Body(mapping).Do(context.Background())
	if err != nil {
		fmt.Printf("<CreateIndex> some error occurred when create. index: %s, err:%s", index, err.Error())
		return false
	}
	if !createIndex.Acknowledged {
		// Not acknowledged
		fmt.Printf("<CreateIndex> Not acknowledged, index: %s", index)
		return false
	}
	return true
}

func main() {
	// 初始化连接Es
	es, err := InitES()
	if err != nil {
		return
	}
	// 创建ES
	es.CreateIndex("text", Contentmapping)
}

```

然后我们去查看我们的Es

![](https://blog-1256169066.cos.ap-chengdu.myqcloud.com/es%E9%AB%98%E7%BA%A7%E7%94%A8%E6%B3%95/2.jpg)

![](https://blog-1256169066.cos.ap-chengdu.myqcloud.com/es%E9%AB%98%E7%BA%A7%E7%94%A8%E6%B3%95/3.jpg)

就可以看到详细的信息了。大概就完成了

## 根据模板上传数据

Golang有个很奇葩的地方，就是，你必须要按照规则传递数据，也就是事先定义好的结构体，如果你不按照这个规矩传，那你肯定是传不进去的，我这边模拟一个简单的数据传递逻辑吧，大家都是聪明孩子，肯定一点就通

首先，观察一下咱们的mapping，定义一个结构体，用来传递数据/输出数据

```golang
type ContentEsInfo struct {
	Name        string     `json:"name"`
	ID          string     `json:"id"`
	Size        uint64     `json:"size"`
	LastModTime string     `json:"last_mod_time"`
	User        string     `json:"user"`
	Data        []DataType `json:"data"`
	Content     string     `json:"content"`
}

type DataType struct {
	Key   string `json:"key"`
	Value string `json:"value" `
}

//Put 传入index名， typ，还有组合成的结构体
func (Es *Elastic) Put(index string, typ string, bodyJSON interface{}) (bool, error) {
	_, err := Es.Client.Index().
		Index(index).
		Type(typ).
		BodyJson(bodyJSON).
		Do(context.Background())
	if err != nil {
		// Handle error
		fmt.Printf("<Put> some error occurred when put.  err:%s", err.Error())
		return false, err
	}
	return true, nil
}

func main() {
	z := ContentEsInfo{Name: "yk", ID: "ak47", Size: 10423, Data: []DataType{{Key: "1", Value: "2"}}, User: "yk123", LastModTime: "2020-01-01 12:00:00", Content: "dssads"}
	es.Put("content_test", "doc", z)
}

```
传完了咱们看一下Es，里面已经有数据了。

如图：

![](https://blog-1256169066.cos.ap-chengdu.myqcloud.com/es%E9%AB%98%E7%BA%A7%E7%94%A8%E6%B3%95/4.jpg)

![](https://blog-1256169066.cos.ap-chengdu.myqcloud.com/es%E9%AB%98%E7%BA%A7%E7%94%A8%E6%B3%95/5.jpg)

## 输出字段

既然已经有字段了，我们现在要输出字段，看下检索结果，同理，你还是需要**结构体**，我告诉你，你就逃不开和结构体的孽缘，哈哈哈哈！

```golang
type ContentEsInfo struct {
	Name        string     `json:"name"`
	ID          string     `json:"id"`
	Size        uint64     `json:"size"`
	LastModTime string     `json:"last_mod_time"`
	User        string     `json:"user"`
	Data        []DataType `json:"data"`
	Content     string     `json:"content"`
}
//GetMsg 获取Msg
func (Es *Elastic) GetMsg(indexname, typ string) {
	var contentinfo ContentEsInfo
	res, _ := Es.Client.Search(indexname).Type(typ).Do(context.Background())
	//从搜索结果中取数据的方法
	for _, item := range res.Each(reflect.TypeOf(contentinfo)) {
		if t, ok := item.(ContentEsInfo); ok {
			fmt.Println(t)
		}
	}
}

func main() {
	es, err := InitES()
	if err != nil {
		return
	}
	es.GetMsg("content_test", "doc")
}
```

这边输出了：

```
{yk ak47 10423 2020-01-01 12:00:00 yk123 [{1 2}] dssads}
```

## 输出指定字段

有些时候你不想显示太多字段？没问题，可以让Es返回的时候指定只显示某些字段。有些时候如果某个字段特别大，我们可以直接屏蔽它，让它不包装返回。

假设我们要让**content**这个字段不返回

```golang
//ShieldAnotherfield 屏蔽指定字段
func (Es *Elastic) ShieldAnotherfield(indexname, typ string) {
	var contentinfo ContentEsInfo
	//指定返回的字段
	fsc := elastic.NewFetchSourceContext(true).Include("name", "type", "user", "size", "last_mod_time", "data")
	res, _ := Es.Client.Search(indexname).Type(typ).FetchSourceContext(fsc).Do(context.Background())
	//从搜索结果中取数据的方法
	for _, item := range res.Each(reflect.TypeOf(contentinfo)) {
		if t, ok := item.(ContentEsInfo); ok {
			fmt.Println(t)
		}
	}
}

func main() {
	es, err := InitES()
	if err != nil {
		return
	}
	es.ShieldAnotherfield("content_test", "doc")
}
```

可以查看一下返回

```golang
{yk  10423 2020-01-01 12:00:00 yk123 [{1 2}] }
```

和上面对比一下，是不是少了dsds那个字段，那个字段就是content，如果当content特别大的时候，它就相当有作用。

## 翻页的实现和优化

翻页，这是老生常谈的问题了，我的blog里面写过Es的翻页优化方法，其实很多人无脑推scroll动态翻页，这是不可取的，有些时候，你要根据自己的需求来定义翻页的逻辑，不能说别人用scroll，你就scroll，from+size也能满足一些不一样的需求。

1. 当你的翻页需要支持跳页，指定页数翻页，最前/最后翻页，随机跳页的时候，我建议你用from+size
2. 当你的翻页是动态的，例如下拉加载，例如往下滑持续加载，动态加载的时候，你要用scroll深度翻页，因为这个才是对你机器负载最低的一种翻页模式。

好了，我们来实现翻页吧。

我这里因为要支持随机跳页，所以我用了from+size的逻辑

```golang
//FromSize 翻页方法
func (Es *Elastic) FromSize(indexname, typ string, size, from int) {
	var contentinfo ContentEsInfo
	res, _ := Es.Client.Search(indexname).Type(typ).Size(size).From(from).Do(context.Background())
	//从搜索结果中取数据的方法
	for _, item := range res.Each(reflect.TypeOf(contentinfo)) {
		if t, ok := item.(ContentEsInfo); ok {
			fmt.Println(t)
		}
	}
}

func main() {
	es, err := InitES()
	if err != nil {
		return
	}
	es.FromSize("content_test", "doc", 10, 0)
}
```
这里输出

```golang
{yk ak47 10423 2020-01-01 12:00:00 yk123 [{1 2}] dssads}
```
**这里的size，你可以理解为每页展示条数，**

**而from，你可以理解为页数，**

要注意，这是从0开始1计算的，类似列表，第一个下标为0，如果你将size改为1，那么将搜索不到数据，理由是：第11条数据不存在，因为我们的数据库只有1条数据（暂时


## 高亮检索关键字

ElasticSearch支持高亮返回，回到刚刚咱们讨论的话题

类似百度文库那样的搜索逻辑，如果检索到之后返回，高亮我们检索的值，这种一般怎么处理呢。

这种其实Es也是支持的

我们现在来一发检索

```golang
//HighlightMsg 高亮方法
func (Es *Elastic) HighlightMsg(indexname, typ string, size, from int, keyword string) {
	// var contentinfo ContentEsInfo

	boolQ := elastic.NewBoolQuery()
	boolZ := elastic.NewBoolQuery()

	// 定义highlight
	highlight := elastic.NewHighlight()
	// 指定需要高亮的字段
	highlight = highlight.Fields(elastic.NewHighlighterField("content"))
	// 指定高亮的返回逻辑 <span style='color: red;'>...msg...</span>
	highlight = highlight.PreTags("<span style='color: red;'>").PostTags("</span>")

	escontent := elastic.NewMatchQuery("content", keyword)
	boolZ.Filter(boolQ.Should(escontent))

	res, _ := Es.Client.Search(indexname).Type(typ).Highlight(highlight).Query(boolZ).Do(context.Background())
	
	// 高亮的输出和doc的输出不一样，这里要注意，我只输出了匹配到高亮的第一个词
	for _, highliter := range res.Hits.Hits {
		fmt.Println(highliter.Highlight["content"][0])
	}
}

func main() {
	es, err := InitES()
	if err != nil {
		return
	}
	// 我们检索一下带有名气的doc，然后高亮输出
	es.HighlightMsg("content_test", "doc", 10, 0, "名气")
}

```

看一下返回

```golang
那就当因为刘先
<span style='color: red;'>名气</span>太大，
曹操不得不展现出极度的宽容吧，
但是神奇的是后面的“周不疑之死”。
```

看到了没，“名气”这个词语被高亮输出了。

## 全文检索-精准度调整

因为分词器的缘故，我们在检索词汇的时候，经常会搜索到一些不相干的词，例如

我们搜索"今天是美好的一天"，我们想要的自然是匹配到 “今天是美好的一天” 的所有doc，但是分词器不会这么想，分词器会将这句话进行分词

切分为：[今天，美好，一天，美好的，美好的一天]

这样Es在检索的时候，就会把上面分词了的数据也检索到，意思就是，包含有“今天”，“美好”，“一天”。。。。之类的数据都可以被检索出来，这绝对不是我们想要的

但是我们可以设定短句搜索，并且调整它的精准度

```golang
//Precisesearch 精准检索
func (Es *Elastic) Precisesearch(indexname, typ string, size, from int, keyword string) {
	// var contentinfo ContentEsInfo

	boolQ := elastic.NewBoolQuery()
	boolZ := elastic.NewBoolQuery()

	// 定义highlight
	highlight := elastic.NewHighlight()
	// 指定需要高亮的字段
	highlight = highlight.Fields(elastic.NewHighlighterField("content"))
	// 指定高亮的返回逻辑 <span style='color: red;'>...msg...</span>
	highlight = highlight.PreTags("<span style='color: red;'>").PostTags("</span>")

	// 短句匹配
	escontent := elastic.NewMatchPhrasePrefixQuery("content", keyword).MaxExpansions(10)

	boolZ.Filter(boolQ.Should(escontent))

	res, _ := Es.Client.Search(indexname).Type(typ).Highlight(highlight).Query(boolZ).Do(context.Background())
	for _, highliter := range res.Hits.Hits {
		fmt.Println(highliter.Highlight["content"][0])
	}
}


func main() {
	es, err := InitES()
	if err != nil {
		return
	}
	// 搜个冷门词，ik里绝壁没有的
	es.HighlightMsg("content_test", "doc", 10, 0, "渔阳三檛")
}
```

得到返回值

```golang
想想祢衡的类似表演（<span style='color: red;'>渔</span>
<span style='color: red;'>阳</span>
<span style='color: red;'>三</span>
<span style='color: red;'>檛</span>），
曹操都只能表示惹不起，恭送出许。
那就当因为刘先名气太大，曹操不得不展现出极度的宽容吧，但是神奇的是后面的“周不疑之死”。
```
这就匹配到了，并且也不会出现乱匹配的问题了。

## 总结

大概也就这么多了，其实很多Es的检索逻辑在我以前的blog里面都写过了

再放送一遍旧文章地址：
```
https://yemilice.com/2020/05/14/golang%E5%B0%81%E8%A3%85elasticsearch%E5%B8%B8%E7%94%A8%E5%8A%9F%E8%83%BD/
```

今天所有写过的代码都在：
```
https://github.com/Alexanderklau/Go_poject/blob/master/Go-Elasticdb/ElasticSearch_adv_use/Es_adv.go
```


大家可以自己下下来自己改着玩玩

这次用Es开发了一个文件检索服务器，作词作曲又是我自己，自认为在Es这个部分，我应该算是接触不少了吧，最近我在看Es源码，准备弄明白Es到底为什么检索那么快，下一篇不是Python三巨头就是Mysql vs Es，大家期待吧！

年底了，希望大家都保重身体啊。

```
翻看着年初自己许下的承诺

到了如今却只有沉默

告诉镜子里的他你已经不小了

你该学会衡量什么是你想要的

你可以无畏自由的向天空宣泄

大喊着我要走自己的路

check~
```