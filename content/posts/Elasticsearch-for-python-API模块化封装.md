---
title: Elasticsearch for python API模块化封装
date: 2019-10-25 09:57:50
tags: ["前后端"]
---
# Elasticsearch for python API模块化封装
## 模块的具体功能
1. 检测Elasticsearch节点是否畅通
2. 查询Elasticsearch节点健康状态
3. 查询包含的关键字的日志（展示前10条）
4. 查询指定的索引下的数据，并且分页
5. 输出所有日志(输出全部)
6. 输出去重后的日志(分页，带关键字）
7. 删除指定索引的值
8. 往索引中添加数据
9. 获取指定index、type、id对应的数据
10. 更新指定index、type、id所对应的数据
11. 批量插入数据


## 使用方法
一般作为独立的包进行导入，并且对其进行了大数据预览的优化和处理  
作为一个独立Python模块进行导入，并且调取接口使用。  
调用方法

```python
import elasticdb.es_sysdb as es
esdb = es.Es()
```

## 使用举例  
打印出索引（表）内的所有数据：  
需要index名，也就是指定索引名，在这里，假设我要查所有的monlog数据，那么查询语句如:

```python
a = esdb.search_all(client=esdb.conn, index=monlog, type="doc")
for i in a:
    c.append(i["_source"]["message"])
```

## 接口详情

接口参数说明

参数|必选|类型|说明
:-------   |:-------|:-----|-----                               |
index |ture    |str|索引名 ，可认为是数据库
type|true    |str|索引类型，可认为是表名
keywords|ture    |str|关键字 
page|ture    |str|页数，分页逻辑
size|ture    |str|每页展示条数，分页逻辑使用

### 查询包含的关键字的日志（展示前10条）

```python
a = esdb.search_searchdoc(index=monlog, type="doc", keywords="cpu")
for i in a:
    print i["_source"]["message"]
```
### 查询指定的索引下的数据，并且分页  
示例：查询index为”oplog-2018-08,oplog-2018-12”，并且每页展示（size）5条，输出第二页（page）

```python
for i in esdb.serch_by_index(index="oplog-2018-08,oplog-2018-12", page=2, size=5)["hits"]["hits"]:
     print(i["_source"]["message"])
```
### 输出所有日志(输出全部)

```python
for i in esdb.search_all(client=esdb.conn, index="monlog-*", type="doc"):
     print i
```

### 输出去重后的日志(分页，带关键字）  
示例：关键字为空，搜索monlog的所有数据，展示第一页，并且每页展示10条

```python
for i in esdb.serch_es_count(keywords = "", index="monlog-*", type="doc",page=1, size=10):
     print i
```

### 删除指定索引的值  
示例：删除monlog的所有值

```python
esdb.delete_all_index(index="monlog-*", type="doc")
```

### 查询集群健康状态

```python
esdb.check_health()
```

### 往索引中添加数据

```python
body = {"name": 'lucy2', 'sex': 'female', 'age': 10}
print esdb.insertDocument(index='demo', type='test', body=body)
```

### 获取指定index、type、id对应的数据

```python
print esdb.getDocById(index='demo', type='test', id='6gsqT2ABSm0tVgi2UWls')
```

### 更新指定index、type、id所对应的数据

```python
body = {"doc": {"name": 'jackaaa'}}#修改部分字段
print esdb.updateDocById('demo', 'test', 'z', body)
```

### 批量插入数据

```python

_index = 'demo'
_type = 'test_df'
import pandas as pd
frame = pd.DataFrame({'name': ['tomaaa', 'tombbb', 'tomccc'],
                        'sex': ['male', 'famale', 'famale'],
                        'age': [3, 6, 9],
                        'address': [u'合肥', u'芜湖', u'安徽']})

print esAction.insertDataFrame(_index, _type, frame)

```

## 代码示例
```python
from elasticsearch import Elasticsearch
from elasticsearch import helpers


class Es:
    def __init__(self):
        self.hosts = "127.0.0.1"
        self.conn = Elasticsearch(hosts=self.hosts, port=9200)


    def check(self):
        '''
        输出当前系统的ES信息
        '''
        return self.conn.info()

    def ping(self):
        return self.conn.ping()

    def check_health(self):
        '''
        检查集群的健康状态
        :return:
        '''
        status = self.conn.transport.perform_request('GET', '/_cluster/health', params=None)["status"]
        return statuu

    def get_index(self):

        return self.conn.indices.get_alias("*")

    def search_specify(self, index=None, type=None, keywords=None, page=None, size=None):
        # 查询包含的关键字的日志
        query = {
            'query': {
                'match': {
                    'message': keywords
                }
            },
            'from':page * size,
            'size':size
        }
        message = self.searchDoc(index, type, query)
        return message
```

完整的代码地址：https://github.com/Alexanderklau/elasticdb