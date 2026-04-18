---
title: ElasticSearch定义自己的分词器词典（检索生僻词）
date: 2020-12-08 15:20:23
tags: ["前后端"]
---

## 前言

ElasticSearch当中，有许多的分词器供我们调用，中文用的最多的就是IK分词器，一些基本的词汇都包含了。

但是，一些基本的生僻词就很难检索了，比如一些特定专业词汇，或者一些流行词汇之类的，所以这篇文章，我会讲一下自定义分词器词典的设置和扩展，让我们能够检索的词汇变的更多。如果能帮到你，我也会很开心！


## 分词器怎么分词的

首先要明白分词器分词的原理，这个我的blog以前说过，中文分词遵循的是最大匹配算法，Maximum

https://yemilice.com/2020/08/21/%E4%B8%AD%E6%96%87%E5%88%86%E8%AF%8D%E7%9A%84%E7%AE%97%E6%B3%95%E5%88%86%E6%9E%90/


可以参看一下这篇博客，说的很清楚了。


我们来举个简单的分词例子

看下这个分词怎么分的：首先

```bash
GET http://10.0.9.28:9200/_analyze

{
    "analyzer": "ik_max_word",
    "text": "年轻人不讲武德"
}
```

分词的结果是

```bash
{
    "tokens": [
        {
            "token": "年轻人",
            "start_offset": 0,
            "end_offset": 3,
            "type": "CN_WORD",
            "position": 0
        },
        {
            "token": "年轻",
            "start_offset": 0,
            "end_offset": 2,
            "type": "CN_WORD",
            "position": 1
        },
        {
            "token": "人",
            "start_offset": 2,
            "end_offset": 3,
            "type": "CN_CHAR",
            "position": 2
        },
        {
            "token": "不讲",
            "start_offset": 3,
            "end_offset": 5,
            "type": "CN_WORD",
            "position": 3
        },
        {
            "token": "讲武",
            "start_offset": 4,
            "end_offset": 6,
            "type": "CN_WORD",
            "position": 4
        },
        {
            "token": "武德",
            "start_offset": 5,
            "end_offset": 7,
            "type": "CN_WORD",
            "position": 5
        }
    ]
}
```
这就是最大粒度的分词结果

举个例子，

你搜 “武德”，这句话就能搜出来

你搜 “年轻人”， 这句话也能搜出来

但是你要搜 “人不”，你铁定毛都搜不出来

为啥，因为这个没被分词儿，你肯定是什么都搜不到的。

我估摸着，你没弄明白我说的啥意思？你那么聪明，智慧，美丽，大方，你肯定能懂吧。

咱们拿生僻词举个例子

假设，我现在搜 “意带利黑手哥”

让我们和他比划比划

让你比划比划，让你知道什么叫黑手！

![](https://blog-1256169066.cos.ap-chengdu.myqcloud.com/es_fenci/4.jpg)


```bash
GET http://10.0.9.28:9200/_analyze

{
    "analyzer": "ik_max_word",
    "text": "意带利黑手哥"
}
```

分词出的结果就是

```bash
{
    "tokens": [
        {
            "token": "意",
            "start_offset": 0,
            "end_offset": 1,
            "type": "CN_CHAR",
            "position": 0
        },
        {
            "token": "带",
            "start_offset": 1,
            "end_offset": 2,
            "type": "CN_CHAR",
            "position": 1
        },
        {
            "token": "利",
            "start_offset": 2,
            "end_offset": 3,
            "type": "CN_CHAR",
            "position": 2
        },
        {
            "token": "黑手",
            "start_offset": 3,
            "end_offset": 5,
            "type": "CN_WORD",
            "position": 3
        },
        {
            "token": "哥",
            "start_offset": 5,
            "end_offset": 6,
            "type": "CN_CHAR",
            "position": 4
        }
    ]
}
```

所以你搜 “意带利”,或者 “意带利黑手哥” 是肯定没这几个词儿的。

那就不能和他比划了。

但是我们就是要这个词可以被检索

那么我们现在该怎么做呢？

## 分词器词汇的扩展

百度，谷歌每天都会更新热点词汇，像 “意带利黑手”，“三日杀神”这样的人物，早就存在词库里面了。

![](https://blog-1256169066.cos.ap-chengdu.myqcloud.com/es_fenci/5.jpg)

当然，我ElasticSearch作为数一数二的检索引擎，能没这功能吗，其实也是有的

一般我们如果要定义自己的词典，首先就要去修改ik的配置

ik的配置一般放置在

```bash
/etc/elasticsearch/analysis-ik
```

这下面有个文件叫做 **IKAnalyzer.cfg.xml**

打开它

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE properties SYSTEM "http://java.sun.com/dtd/properties.dtd">
<properties>
        <comment>IK Analyzer 扩展配置</comment>
        <!--用户可以在这里配置自己的扩展字典 -->
        <entry key="ext_dict"></entry>
         <!--用户可以在这里配置自己的扩展停止词字典-->
        <entry key="ext_stopwords"></entry>
        <!--用户可以在这里配置远程扩展字典 -->
        <!-- <entry key="remote_ext_dict">words_location</entry> -->
        <!--用户可以在这里配置远程扩展停止词字典-->
        <!-- <entry key="remote_ext_stopwords">words_location</entry> -->
</properties>
```

这注释，很清楚啊，我想。。大家不需要我再解释参数了吧。

如何定义扩展字典？

首先看一下目录下面的几个文件

- IKAnalyzer.cfg.xml：用来配置自定义词库
- main.dic：ik 原生内置的中文词库，总共有 27 万多条
- quantifier.dic：放了一些单位相关的词
- suffix.dic：放了一些后缀
- surname.dic：中国的姓氏
- stopword.dic：英文停用词

这里可以看到，我们有两个方法去增加扩展词

一个是直接修改main.dic，进行词汇追加，另外一个就是重新写入一个dic文件，直接在IKAnalyzer.cfg.xml里面进行修改。这两种方法都介绍一下吧。

### 追加词汇

直接在main.dic里面追加词汇 “意带利黑手”

![](https://blog-1256169066.cos.ap-chengdu.myqcloud.com/es_fenci/1.jpg)

然后重启ElasticSearch

再进行一次检索

```bash
GET http://10.0.9.28:9200/_analyze

{
    "analyzer": "ik_max_word",
    "text": "意带利黑手"
}
```
返回如下
```bash
{
    "tokens": [
        {
            "token": "意带利黑手",
            "start_offset": 0,
            "end_offset": 5,
            "type": "CN_WORD",
            "position": 0
        },
        {
            "token": "黑手",
            "start_offset": 3,
            "end_offset": 5,
            "type": "CN_WORD",
            "position": 1
        }
    ]
}
```
这下就能搜到了。

### 新建字典

首先，我们在/etc/elasticsearch/analysis-ik里面建个字典，名字叫 new.dic

![](https://blog-1256169066.cos.ap-chengdu.myqcloud.com/es_fenci/2.jpg)

在里面写入我们要检索的词

![](https://blog-1256169066.cos.ap-chengdu.myqcloud.com/es_fenci/3.jpg)

然后修改配置

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE properties SYSTEM "http://java.sun.com/dtd/properties.dtd">
<properties>
        <comment>IK Analyzer 扩展配置</comment>
        <!--用户可以在这里配置自己的扩展字典 -->
        <entry key="ext_dict">./new.dic</entry>
         <!--用户可以在这里配置自己的扩展停止词字典-->
        <entry key="ext_stopwords"></entry>
        <!--用户可以在这里配置远程扩展字典 -->
        <!-- <entry key="remote_ext_dict">words_location</entry> -->
        <!--用户可以在这里配置远程扩展停止词字典-->
        <!-- <entry key="remote_ext_stopwords">words_location</entry> -->
</properties>
```

重启elasticsearch

然后继续检索

```bash
GET http://10.0.9.28:9200/_analyze

{
    "analyzer": "ik_max_word",
    "text": "意带利黑手"
}
```
返回如下
```bash
{
    "tokens": [
        {
            "token": "意带利黑手",
            "start_offset": 0,
            "end_offset": 5,
            "type": "CN_WORD",
            "position": 0
        },
        {
            "token": "黑手",
            "start_offset": 3,
            "end_offset": 5,
            "type": "CN_WORD",
            "position": 1
        }
    ]
}
```
这就能和黑手哥比划比划了。

扩展字典的逻辑，大概就是这样。

## 结尾
这两天在做智能检索，自己的ElasticSearch还是要多补补啊，最近成都疫情又变严重了，sad，要老老实实在家待一阵好好学习了。

希望能够帮到大家，希望大家多提意见，多和我比划比划。

![](https://blog-1256169066.cos.ap-chengdu.myqcloud.com/es_fenci/6.jpg)

年底了，大家要快乐呀！
