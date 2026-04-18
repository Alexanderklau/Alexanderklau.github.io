---
title: ElasticSearch重大漏洞-log4j的处理方案-12月20日更新！！！！！
date: 2021-12-14 15:45:12
tags: ["前后端"]
---

## 前言

最近暴露出一个炸弹级别的漏洞 log4j

Log4j的GitHub公开披露了一个影响 Apache Log4j 2 实用程序多个版本的高严重性漏洞 (CVE-2021-44228)。

该漏洞影响了 Apache Log4j 2 的 2.0 到 2.14.1 版本。

2.0 <= Apache log4j2 <= 2.14.1

具体详情可以参看 https://logging.apache.org/log4j/2.x/

首先，受到影响的Es家族系列有

- Elasticsearch 
- Logstash 
- Elastic Cloud 
- APM Java Agent 
- Elastic Cloud Enterprise 
- Elastic Cloud on Kubernetes 
- Swiftype

很不幸，鄙人维护的多个Es项目都在此内，需要立马处理并且解决问题

## 处理方案

### 处理方案（1）

1. 检查所有使用了 Log4j 组件的系统，并且立即使用log4j-2.17进行替换
2. 重启 Elasticsearch 节点

检查所有使用了 Log4j 组件的系统，并且立即使用log4j-2.17进行替换

原来是2.15版本。但是2.15版本被暴露出有新的bug。。。

这个修复起来属实没完了，所以我这里是最新的2.17版本！

官方给出了具体链接：https://logging.apache.org/log4j/2.x/changes-report.html#a2.17.0

但是我寻思估计大家都比较懒。。。

所以我写了个脚本去处理这个问题

我把脚本和包都放置到了github里，直接调用bash就行了，应该是最简单的办法

具体使用方法在下面

### 处理方案（2）

如果不方便替换jar，则必须修改系统变量

1. 修改 JVM 参数 -Dlog4j2.formatMsgNoLookups=true
2. 修改配置 log4j2.formatMsgNoLookups=True
3. 将系统环境变量 FORMAT_MESSAGES_PATTERN_DISABLE_LOOKUPS 设置为 true
4. 重启 Elasticsearch 节点


### 处理方案（3）

立马将线上机器下线，并且进行网关配置

例如nginx其他API访问阻隔请求，这里我不说了，因为我这里没涉及到。


## 脚本处理方案

首先把我的脚本下下来

地址：https://github.com/Alexanderklau/Amusing_python/tree/master/Network_security/log4j-fix

或者直接拖那个rar下来，都是一样的

https://github.com/Alexanderklau/Amusing_python/blob/master/Network_security/log4j-fix/log4j-fix.rar

解压完之后

首先进行检查

![](https://blog-1256169066.cos.ap-chengdu.myqcloud.com/Elasticsearch%E5%A4%A7%E6%95%B0%E6%8D%AE%E9%87%8F%E4%B8%8B%E7%9A%84%E4%BC%98%E5%8C%96%E6%96%B9%E6%B3%95/new1.jpg)

此处显示无风险，因为我已经修复过了，如果你的jar版本低于15，应该会显示有风险

再运行脚本进行替换

![](https://blog-1256169066.cos.ap-chengdu.myqcloud.com/Elasticsearch%E5%A4%A7%E6%95%B0%E6%8D%AE%E9%87%8F%E4%B8%8B%E7%9A%84%E4%BC%98%E5%8C%96%E6%96%B9%E6%B3%95/new2.jpg)

替换完毕

手动重启Es服务

```shell
systemctl restart elasticsearch
```

结束

