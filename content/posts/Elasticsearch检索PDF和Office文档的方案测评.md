---
title: Elasticsearch检索PDF和Office文档的方案测评
date: 2020-07-29 14:02:32
tags: ["其他技术","前后端"]
---
## 前言
这段时间主要攻关了一下Elasticsearch的一些特性，发觉Elasticsearch还是个挺牛逼的玩意儿，我以前经常用它存日志，还没想着拿来存别的东西，一般不都SQL嘛，但是我看了一下现在流行的检索方案，基本都是elasticsearch做检索引擎，说明这个东西经历了时间的考研，用了的人都说好。

我以前拿Elasticsearch来存储日志，搜东西相当方便，定义好检索语句直接走你，但是我看百度文库搜索，还可以做到docx，pdf关键字检索并且高亮，这个就很厉害了，这两天紧急攻关了一下，把自己踩的坑和收获记录下来，方便未来自己复习/查看

## 需求分析
老样子，写任何blog，我都要做需求分析，这样才能更好的进行开发和写作。

### 我们要干嘛
我们要实现一个elasticsearch检索pdf和office家族文件的功能，并且要对pdf/office文件进行一个全文检索，也就是说，搜索“Yemilice”，不仅仅是标题含有“Yemilice”，内容含有“Yemilice”的文件也要检索出来，并且给人家高亮。

### 我们的工作环境

**底层环境**

Centos7服务器

**Elasticsearch的版本**

6.3.2

## 现有的检索解决方案

经过我的发力工作（Google/baidu）,现在市面上流行这么几种方案

1. Elasticsearch 官方插件 **ingest-attachment**
2. 第三方开源服务 **fscrawler**
3. 大数据平台 **Ambari**

接下来麻烦事儿就来了，这三个方案，到底选哪个？才能更那啥呢，我倒不如把他们一一实现一遍，然后给大家伙儿展示一拨，然后最后再选一个合适的不就得了嘛！

没事儿，苦了我一个，幸福大家伙儿呗！

## 1. ingest-attachment插件
这玩意儿是什么呢，说白了就是elasticsearch官方给大家贡献的一个插件，支持你把docx，pdf之类的东西导入到es当中。当然要你自己手动去实现接口。

### 1.1 ingest-attachment插件的安装
首先，我认为你现在在CN境内，你就**不要**看网上那些老哥教的直接执行

```bash
./elasticsearch-plugin install ingest-attachment
```

这样你会等到地老天荒也下载不完，现在你需要的就是下个离线包，然后再去安装

下载离线包的网址

```
https://artifacts.elastic.co/downloads/elasticsearch-plugins/ingest-attachment/ingest-attachment-6.3.2.zip.
```
注意一点啊，一定要注意！你的ingest-attachment版本必须和你的elasticsearch一致，你要强行不听我的，到时候费心思整完，弄不好，那你就只能来颗华子再来一次了。

安装离线包
```bash
./elasticsearch-plugin install ingest-attachment-6.3.2.zip
```
没报错就说明你安上了，不放心的话

```bash
./elasticsearch-plugin list
```
看一下，有这个插件了说明你已经有了这个插件

### 1.2 ingest-attachment插件的基础使用
看了下官网，其实他的逻辑很简单，整个管道，你把你的pdf/doc什么的给转成base64，然后通过管道把base64传进去，es会把所有的东西给你安排好。

我现在简单的整了个pdf，名字就叫1.pdf，现在开始吧。

**抽取管道的函数编写，这里是告诉es，创建一个抽取管道pipeline**
```bash 
curl -X PUT "localhost:9200/_ingest/pipeline/attachment" -d '{
 "description" : "Extract attachment information",
 "processors":[
 {
    "attachment":{
        "field":"data",
        "indexed_chars" : -1,
        "ignore_missing":true
     }
 },
 {
     "remove":{"field":"data"}
 }]}'

```

**创建一个index（表），名字叫pdf，设定1个分片（单节点，当然你自己可以改）**
```bash 
curl --location --request PUT '127.0.0.1:9200/pdf' \
--header 'Content-Type: application/json' \
-d '{
    "settings": {
        "index": {
            "number_of_shards": 1,
            "number_of_replicas": 0
        }
    }
}'
```

**将pdf转为base64编码，然后上传到elasticsearch当中**

这里网上那个方法是直接perl调base64，我这里一直报错，如果和我一样的老哥，用我下面给得另一种方法

方法1

```bash
curl --location --request PUT '10.0.7.234:9200/pdf/pdf/1?pipeline=attachment' \
--header 'Content-Type: application/json' \
-d '{  "data":" '`base64 -w 0 /root/pdf/pdf.pdf | perl -pe's/\n/\\n/g'`'"
}'
```

方法2

直接导入base64码

```bash
curl --location --request PUT '10.0.7.234:9200/pdf/pdf/1?pipeline=attachment' \
--header 'Content-Type: application/json' \
-d '{  "data":"JVBERi0xLjcNCiW1tbW1DQoxIDAgb2JqDQo8PC9UeXBlL0NhdGFsb2cvUGFnZXMgMiAwIFIvTGFuZyh6aC1DTikgL1N0cnVjdFRyZWVSb290IDE1IDAgUi9NYXJrSW5mbzw8L01hcmtlZCB0cnVlPj4vTWV0YWRhdGEgMzMgMCBSL1ZpZXdlclByZWZlcmVuY2VzIDM0IDAgUj4+DQplbmRvYmoNC......（这里我不展示完了）"
}'
```

现在去查看index（表）

![](https://blog-1256169066.cos.ap-chengdu.myqcloud.com/Elasticsearch%E6%A3%80%E7%B4%A2PDF%E5%92%8COffice%E6%96%87%E6%A1%A3%E7%9A%84%E6%96%B9%E6%A1%88%E6%B5%8B%E8%AF%84/1.jpg)

显示了作者，之类的杂七杂八信息，而content就是详细的全文，可以直接搜索，因为有我的大名我就马赛克了。。

这就导入了，搜索也是按一般的搜索方法

```bash
GET pdf/_search
{

  "query": {
    "match": {
      "attachment.content": "pdf"
    }
  }
}
```

### 1.3 ingest-attachment插件的性能评测
**测试导入PDF（10M大小，无图片）**

base64命令在perl中执行了大概2s左右，

并且CPU/内存 在转换->存储这个阶段，占用了

![](https://blog-1256169066.cos.ap-chengdu.myqcloud.com/Elasticsearch%E6%A3%80%E7%B4%A2PDF%E5%92%8COffice%E6%96%87%E6%A1%A3%E7%9A%84%E6%96%B9%E6%A1%88%E6%B5%8B%E8%AF%84/2.jpg)

可见在抽取文本 ——> 传输存储入es的时候，插件可能会占用少部分CPU

**测试导入PDF（40M大小，图片/文字混用）**

base64命令在perl中执行了大概10s左右，并且还返回了错误（perl长度过大的错误）

并且CPU/内存 在转换->存储这个阶段，占用了

![](https://blog-1256169066.cos.ap-chengdu.myqcloud.com/Elasticsearch%E6%A3%80%E7%B4%A2PDF%E5%92%8COffice%E6%96%87%E6%A1%A3%E7%9A%84%E6%96%B9%E6%A1%88%E6%B5%8B%E8%AF%84/3.jpg)

在图片/文字混用并且pdf文件比较大的情况下，内存占用较大，并且返回较慢，这里需要注意，并且解析的base64会直接塞到系统内存当中，如果做多文件抽取，可能会有CPU/内存占用较大的情况出现。

### 1.4 ingest-attachment插件的优缺点
优点： 

1. 安装方便，只需要下载zip安装包，调用elasticsearch本身自带的安装模块即可
2. 使用方便，编辑一条管道抽取命令，可以直接利用linux的base64命令转码存入elasticsearch中
3. 支持的格式比较多，支持ppt，pdf，doc，docx，xls等office常用的办公软件格式导入到elasticsearch当中。
4. 导入后的文档可以直接输入中文进行检索

缺点：

1. 大文件支持不够，如果超过100M的pdf，base64转码将比较缓慢，并且，存入到elasticsearch中的数据也十分庞大
2. 必须要将文件下载到本地，然后必须进行base64转码才可以存到elasticsearch中，等于说，必须要实体文件，因为需要base64转码，如果文件在对象存储/云上，那就不可以这么操作了
3. 对于pdf，doc中含有图片的情况，没有办法将图片中文文字识别出来，而且如果出现图片较多的情况，转码较慢，含有特殊字符的base64码导入也会无法识别。

### 1.5 ingest-attachment插件的使用场景
1. 存储的文件不那么大的情况
2. 存储的文件大部分都是纯文字的情况
3. 存储的文件全文搜索精准度要求不那么高


## 2. fscrawler服务
一个类似filebeat的监控服务，监控某个文件夹下面的文档，定时去遍历一次，如果发现了新添加的文档则会直接写入到elasticsearch当中。

### 2.1 fscrawler安装
首先非常重要的一点，确定你的elasticseach版本，

如果你的版本 <= 6.3.2，那你需要下载2.5 版本的fscrawler，这个非常重要，否则是无法启动任务的！！！！

```
https://repo1.maven.org/maven2/fr/pilato/elasticsearch/crawler/fscrawler/2.5/fscrawler-2.5.zip
```

如果你的版本 > 6.3.2, 你就可以随意选择2.6，2.7的fscrawler了，区别还是有一些的，后面我会说。

下载下来，解压，这个你肯定会

```
unzip fscrawler-2.5.zip
```

尝试跑一下

```
./fscrawler
```

输出

```bash
03:53:39,425 INFO  [f.p.e.c.f.c.FsCrawler] No job specified. Here is the list of existing jobs:
03:53:39,433 INFO  [f.p.e.c.f.c.FsCrawler] [1] - work01
03:53:39,433 INFO  [f.p.e.c.f.c.FsCrawler] Choose your job [1-1]...
```
说明能用

### 2.2 fscrawler的使用方法

启动服务

![](https://blog-1256169066.cos.ap-chengdu.myqcloud.com/Elasticsearch%E6%A3%80%E7%B4%A2PDF%E5%92%8COffice%E6%96%87%E6%A1%A3%E7%9A%84%E6%96%B9%E6%A1%88%E6%B5%8B%E8%AF%84/4.png)


查看一下文件结构

![](https://blog-1256169066.cos.ap-chengdu.myqcloud.com/Elasticsearch%E6%A3%80%E7%B4%A2PDF%E5%92%8COffice%E6%96%87%E6%A1%A3%E7%9A%84%E6%96%B9%E6%A1%88%E6%B5%8B%E8%AF%84/5.png)

编写指定的json去进行服务监控/设置

![](https://blog-1256169066.cos.ap-chengdu.myqcloud.com/Elasticsearch%E6%A3%80%E7%B4%A2PDF%E5%92%8COffice%E6%96%87%E6%A1%A3%E7%9A%84%E6%96%B9%E6%A1%88%E6%B5%8B%E8%AF%84/6.png)

现在导入一个1.pdf的文件

![](https://blog-1256169066.cos.ap-chengdu.myqcloud.com/Elasticsearch%E6%A3%80%E7%B4%A2PDF%E5%92%8COffice%E6%96%87%E6%A1%A3%E7%9A%84%E6%96%B9%E6%A1%88%E6%B5%8B%E8%AF%84/7.png)


监控到fscrawler发生了变化

![](https://blog-1256169066.cos.ap-chengdu.myqcloud.com/Elasticsearch%E6%A3%80%E7%B4%A2PDF%E5%92%8COffice%E6%96%87%E6%A1%A3%E7%9A%84%E6%96%B9%E6%A1%88%E6%B5%8B%E8%AF%84/9.png)

去查看elasticsearch，已经有了这个1.pdf的文件

![](https://blog-1256169066.cos.ap-chengdu.myqcloud.com/Elasticsearch%E6%A3%80%E7%B4%A2PDF%E5%92%8COffice%E6%96%87%E6%A1%A3%E7%9A%84%E6%96%B9%E6%A1%88%E6%B5%8B%E8%AF%84/8.png)

### 2.3 fscrawler的配置文件解析（彩蛋）
一般来说，咱们fscrawler的配置文件都是默认在/root/.fscrawler/_settings.json下的，但是，如果你的es版本高于6.3.2，你下载的fs版本不是2.5，那么你的settings文件将会是yaml格式的，不知道是不是在向filebeat看齐。。。

```json
{
    // 工程名
  "name" : "work01",  
  "fs" : {
    //   监控的文件夹
    "url" : "/tmp/es",
    // 多长时间去扫描一次
    "update_rate" : "30s",
    // 添加例外
    "excludes" : [ "*/~*" ],
    "json_support" : false,
    // 将文件名作为ID
    "filename_as_id" : false,
    // 文件大小添加
    "add_filesize" : true,
    // 同步删除
    "remove_deleted" : true,
    "add_as_inner_object" : false,
    "store_source" : false,
    "index_content" : true,
    "attributes_support" : false,
    "raw_metadata" : true,
    "xml_support" : false,
    "index_folders" : true,
    "lang_detect" : false,
    "continue_on_error" : false,
    // ocr开启
    "pdf_ocr" : true,
    "ocr" : {
        // ocr英文
      "language" : "eng"
    }
  },
  "elasticsearch" : {
    "nodes" : [ {
        // esip
      "host" : "127.0.0.1",
      "port" : 9200,
      "scheme" : "HTTP"
    } ],
    "bulk_size" : 100,
    "flush_interval" : "5s",
    "byte_size" : "10mb"
  },
  "rest" : {
    "scheme" : "HTTP",
    "host" : "127.0.0.1",
    "port" : 8080,
    "endpoint" : "fscrawler"
  }
}

```

### 2.4 fscrawler的性能评测
首先走一波导入文件测试

**导入1M纯文字的pdf**

瞬间就完成了，速度极快

elasticsearch监控信息如下

![](https://blog-1256169066.cos.ap-chengdu.myqcloud.com/Elasticsearch%E6%A3%80%E7%B4%A2PDF%E5%92%8COffice%E6%96%87%E6%A1%A3%E7%9A%84%E6%96%B9%E6%A1%88%E6%B5%8B%E8%AF%84/10.jpg)

Fscrawler占用CPU/内存如下

![](https://blog-1256169066.cos.ap-chengdu.myqcloud.com/Elasticsearch%E6%A3%80%E7%B4%A2PDF%E5%92%8COffice%E6%96%87%E6%A1%A3%E7%9A%84%E6%96%B9%E6%A1%88%E6%B5%8B%E8%AF%84/11.jpg)

**导入10M纯文字的pdf**

瞬间就完成了，速度极快

elasticsearch监控信息如下

![](https://blog-1256169066.cos.ap-chengdu.myqcloud.com/Elasticsearch%E6%A3%80%E7%B4%A2PDF%E5%92%8COffice%E6%96%87%E6%A1%A3%E7%9A%84%E6%96%B9%E6%A1%88%E6%B5%8B%E8%AF%84/10.jpg)

Fscrawler占用CPU/内存如下

![](https://blog-1256169066.cos.ap-chengdu.myqcloud.com/Elasticsearch%E6%A3%80%E7%B4%A2PDF%E5%92%8COffice%E6%96%87%E6%A1%A3%E7%9A%84%E6%96%B9%E6%A1%88%E6%B5%8B%E8%AF%84/11.jpg)

**导入40M文字图片都有的pdf**

速度很快，但是CPU占比一下就上去了，es还是没有什么变化

elasticsearch监控信息如下

![](https://blog-1256169066.cos.ap-chengdu.myqcloud.com/Elasticsearch%E6%A3%80%E7%B4%A2PDF%E5%92%8COffice%E6%96%87%E6%A1%A3%E7%9A%84%E6%96%B9%E6%A1%88%E6%B5%8B%E8%AF%84/13.jpg)

Fscrawler占用CPU/内存如下

![](https://blog-1256169066.cos.ap-chengdu.myqcloud.com/Elasticsearch%E6%A3%80%E7%B4%A2PDF%E5%92%8COffice%E6%96%87%E6%A1%A3%E7%9A%84%E6%96%B9%E6%A1%88%E6%B5%8B%E8%AF%84/12.jpg)

### 2.5 fscrawler的优缺点

fscrawler我其实是比较推荐的

**优点**
1. 支持的格式很多，并且自带OCR，如果内存/CPU富裕的情况可以直接上OCR，识别率虽然不是特别高但也够用了。
2. 导入不用自己动手，设置好配置之后，fscrawler会帮你创建好index，然后自动导入文件
3. 支持文件过滤，如果监控的文件夹里面闲杂文件比较多可以写正则过滤掉
4. CPU/内存占比比较低，我是用的自己的虚拟机，导入40Mpdf完全无问题，CPU占比上去了一瞬间就降下来了。

**缺点**
1. 结合我们的使用场景，我们的文件在远端，需要下载下来，下载到指定文件夹，但是下载到指定文件夹后，fscrawler不会马上导入，是去定时扫描，下载到本地的话可能会把系统盘内存撑满，加快扫描时间会让CPU占比上升
2. 安装部署比较麻烦，需要自己写service，自己写维护
3. 配置文件需要自己定义，容错有问题，如果es down机，原有的文件发送一遍失败之后，不会继续发送

## 3. Ambari服务
这个有个很奇葩的点，是要Es去依赖它，而不是它依赖Es，这东西本身是做大数据的！我感觉咱都有Es了还要啥自行车！

这个是做大数据用的，安装一个大概1个多G，但是功能比较全面，支持各种大文件导入，还支持图片文字的读取，自动分词之类的，其实还是很好用。

### 3.1 使用一波Ambari

Ambari的安装就不在这里介绍了。。这个安装比较复杂，但是网上大部分都是docker集成的，我们现在的环境没有docker，所以我只有手动安装了，在我的机器已经安上了，现在需要安装Ambari的Elasticsearch插件

```bash
wget https://community.hortonworks.com/storage/attachments/87416-elasticsearch-mpack-2600-9.tar.gz
```

安装mpack

```bash
ambari-server install-mpack --mpack=/path/to/87416-elasticsearch-mpack-2600-9.tar.gz --verbose
```

重启ambari-server

```bash
ambari-server restart
```

在ambari-server 上部署elasticsearch

```bash
curl --user admin:admin -i -H 'X-Requested-By: ambari' -X GET "http://ambari.server:8080/api/v1/clusters/$cluster_name/configurations?type=cluster-env"
```

启动elasticsearch

```bash
systemctl restart elasticsearch
```

现在可以在Ambari的管理页面看到es服务器了

![](https://blog-1256169066.cos.ap-chengdu.myqcloud.com/Elasticsearch%E6%A3%80%E7%B4%A2PDF%E5%92%8COffice%E6%96%87%E6%A1%A3%E7%9A%84%E6%96%B9%E6%A1%88%E6%B5%8B%E8%AF%84/14.jpg)

设置ambar的配置文件

```yml
my-files:
    depends_on: 
      serviceapi: 
        condition: service_healthy 
    image: ambar/ambar-local-crawler
    restart: always
    networks:
      - internal_network
    expose:
      - "8082"
    environment:      
      - name=my-files
      - ignoreFolders=**/ForSharing/**
      - ignoreExtensions=.{exe,dll,rar}
      - ignoreFileNames=*backup*
      - maxFileSize=15mb
    volumes:
      - /media/Docs:/usr/data
```

现在尝试导入一个PDF

```bash
curl -X POST \
  http://ambar/api/files/Books/1984-george_orwell.pdf \
  -H 'content-type: multipart/form-data; boundary=----WebKitFormBoundary7MA4YWxkTrZu0gW' \
  -F 1984-george_orwell.rtf=@1984-george_orwell.pdf
```

这下就完成了导入

### 3.2 高级特性

**监控S3对象存储**

设定ak，sk
```bash
echo ACCESS_KEY_ID:SECRET_ACCESS_KEY >  ~/.passwd-s3fs
chmod 600  ~/.passwd-s3fs
```

挂载s3 bucket

```bash
mkdir /mnt/s3-bucket
```

添加挂载到fstab中

```
mybucket /mnt/s3-bucket fuse.s3fs _netdev,allow_other 0 0    
```

修改配置文件

```yml
my-files:
    depends_on: 
      serviceapi: 
        condition: service_healthy 
    image: ambar/ambar-local-crawler
    restart: always
    networks:
      - internal_network
    expose:
      - "8082"
    environment:      
      - name=my-files
      - ignoreFolders=**/ForSharing/**
      - ignoreExtensions=.{exe,dll,rar}
      - ignoreFileNames=*backup*
      - maxFileSize=15mb
    volumes:
      - /media/Docs:/mnt/s3-bucket
```

**OCR文件识别**

ambari自带强势的ocr识别，可以识别多种文字/字母/格式的文件，但是需要富裕的CPU和内存

### 3.3 可能会出现的问题
 
**优点**
1. 它可以很好地处理大文件（> 100 MB）
2. 它从PDF中提取内容（即使格式不佳并带有嵌入式图像），并对图像进行OCR
3. 它为用户提供了简单易用的REST API和WEB UI
4. 部署非常容易（感谢Docker）
5. 它是根据Fair Source 1 v0.9许可开源的
6. 开箱即用地为用户提供解析和即时搜索体验。

**缺点**
1. 部署可能会相当麻烦，没有docker部署，单独部署的话配置比较复杂
2. OCR会占用大量的CPU/内存
3. 如果不是大数据，有些大材小用了

## 4. 总结
我都分析成这样了，该用哪个，心里是不是有点数了，其实很简单

当你的pdf/doc文件不大的情况下，上ingest-attachment插件

当你的文件经常发生变动，上fscrawler

当你是大数据老哥，上Ambari

今天的文章就到这，憋了个大的，如果帮到你，我很开心。

end

2020-07-29