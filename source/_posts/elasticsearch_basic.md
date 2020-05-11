---
title: ElasticSearch学习笔记
date: 2020-03-09 00:00:00
tags:
 - elasticsearch
 - kibana
categories:
 - 运营运维
---

# 运行 

> `ip`酌情修改

``` bash
$ docker pull kibana:6.5.4
$ docker pull elasticsearch:6.5.4
$ docker run -d -p 9200:9200 -p 9300:9300 -e "discovery.type=single-node" \
    --name elastic  elasticsearch:6.5.4
$ docker run -d -e "ELASTICSEARCH_URL=http://192.168.2.139:9200" \
    --name kibana   -p 5601:5601 kibana:6.5.4
```


## 7.3.0

> `认证`, [`filebeat lifecycle`](https://www.elastic.co/guide/en/beats/filebeat/current/ilm.html)
``` bash
$ docker pull docker.elastic.co/beats/filebeat:7.3.1

$ docker run -d -p 9200:9200 -p 9300:9300 -e "discovery.type=single-node" \
    -e "ELASTIC_USERNAME=elastic" \
    -e "ELASTIC_PASSWORD=kibana" \
    -e "xpack.security.enabled=true" \
    --name elastic  elasticsearch:7.3.0

$ docker run -d -e "ELASTICSEARCH_HOSTS=http://192.168.2.139:9200" \
    -e "ELASTICSEARCH_USERNAME=elastic" \
    -e "ELASTICSEARCH_PASSWORD=kibana" \
    -e "xpack.security.enabled=true" \
    --name kibana   -p 5601:5601 kibana:7.3.0


$ docker run -d --name=filebeat  \
	--volume="C:\docker\filebeat\filebeat.docker.yml:/usr/share/filebeat/filebeat.yml:ro" \
    --volume="D:\log:/var/lib/docker/containers:ro" \
    docker.elastic.co/beats/filebeat:7.3.0
```
### beat直接运行
<https://artifacts.elastic.co/downloads/beats/filebeat/filebeat-7.3.0-linux-x86_64.tar.gz>


``` bash
$ ./filebeat-7.3.0-linux-x86_64/filebeat -c a.yml  -e -d "*"
```
``` yml
filebeat.inputs:
- type: log
  paths:
    - ~/filebeat/x.log


output.elasticsearch:
  hosts: ["http://127.0.0.1:9200"]
  username: "filebeat"
  password: "filebeat"

setup.template.enabled: false
setup.ilm.rollover_alias: "filebeat-test"
setup.ilm.check_exists: true
setup.ilm.overwrite: false
setup.ilm.pattern: "{now/d}"
```

# [ECK](https://www.elastic.co/guide/en/cloud-on-k8s/current/k8s-quickstart.html)

## all in one
需要在集群中先安装 <https://www.elastic.co/guide/en/cloud-on-k8s/current/k8s-deploy-eck.html>

可以酌情修改

``` bash
$ kubectl apply -f https://download.elastic.co/downloads/eck/1.1.0/all-in-one.yaml
```

一些细节

+  比较费内存, 如果只是配置和验证, 可以适当调小
+ 1个master和1个data 跑不起来, 非得各三个, 待查

## 帐号
+ <https://www.elastic.co/guide/en/elasticsearch/reference/current/built-in-users.html>
+ <https://www.elastic.co/guide/en/elasticsearch/reference/current/users-command.html>


进入pod

```
# 仅供测试
$ bin/elasticsearch-users useradd king -p pwd -r superuser
```

port forward

``` bash
$ curl -k -u king -XGET "https://127.0.0.1:9200/_count?pretty  " -H 'Content-Type: application/json' -d'
{
  "query": {
    "match_all": {}
  }
}'
Enter host password for user 'king':
{
  "count" : 42,
  "_shards" : {
    "total" : 3,
    "successful" : 3,
    "skipped" : 0,
    "failed" : 0
  }
}
$ curl -k -u king:pwd -XPUT "https://127.0.0.1:9200/idx  " -H 'Content-Type: application/json'
{"acknowledged":true,"shards_acknowledged":true,"index":"idx"}
$ curl -k -u king:pwd -XGET "https://127.0.0.1:9200/_count?pretty  " -H 'Content-Type: application/json' -d'
{
  "query": {
    "match_all": {}
  }
}'
{
  "count" : 42,
  "_shards" : {
    "total" : 3,
    "successful" : 3,
    "skipped" : 0,
    "failed" : 0
  }
}
```




## Debug
+ curl
+ kibana -> dev tools
+ [ElasticSearch Head](https://chrome.google.com/webstore/detail/elasticsearch-head/ffmkiejjmecolpfloofpjologoblkegm)
+ <https://github.com/mobz/elasticsearch-head> 同上

``` bash
$ curl -XGET "http://192.168.2.139:9200/_count?pretty  " -H 'Content-Type: application/json' -d'
{
  "query": {
    "match_all": {}
  }
}'
```
``` bash
GET _count?pretty  
{
  "query": {
    "match_all": {}
  }
}
```

# 概念
+ 索引(index)
+ 类型(type)
+ 文档

``` bash
Relational DB -> Databases -> Tables -> Rows -> Columns
Elasticsearch -> Indices -> Types -> Documents -> Fields
```

> Elasticsearch集群可以包含多个索引(indices)（数据库），每一个索引可以包含多个类型(types)（表），每一个类型包含多个文档(documents)（行），然后每个文档包含多个字段(Fields)（列）。

## 基本操作
创建/查询
``` bash
PUT /megacorp/employee/1
{
  "first_name": "John",
  "last_name": "Smith",
  "age": 25,
  "about": "I love to go rock climbing",
  "interests": [
    "sports",
    "music"
  ]
}
```

| 名字 |说明 | note |
|-|-| - |
|megacorp| 索引名 | 文档存储的地方 |
|employee| 类型名 | 文档代表的对象的类|
|1| 这个员工的ID | 文档的唯一标识|

``` bash
GET /megacorp/employee/1
GET /megacorp/employee/_search?q=last_name:Smith
GET /megacorp/employee/_search
{
  "query": {
    "match": {
      "last_name": "Smith"
    }
  }
}
```

### 搜索类型
+ query string 将所有参数通过查询字符串定义 `上面第二个查询`
+ DSL 使用JSON完整的表示请求体(request body) `上面第三个查询`

### 查询与过滤
+ 结构化查询（Query DSL）
+ 结构化过滤（Filter DSL）

> 对比返回结果的差异

一条过滤语句会询问每个文档的字段值是否包含着特定值

``` json
{
  "_index" : "megacorp",
  "_type" : "employee",
  "_id" : "1",
  "_version" : 1,
  "found" : true,
  "_source" : {
    "first_name" : "John",
    "last_name" : "Smith",
    "age" : 25,
    "about" : "I love to go rock climbing",
    "interests" : [
      "sports",
      "music"
    ]
  }
}
```

一条查询语句会计算每个文档与查询语句的相关性，会给出一个相关性评分 _score ，并且按照相关性对匹配到的文档进行排序。

``` json
{
  "took" : 1,
  "timed_out" : false,
  "_shards" : {
    "total" : 5,
    "successful" : 5,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : 1,
    "max_score" : 0.2876821,
    "hits" : [
      {
        "_index" : "megacorp",
        "_type" : "employee",
        "_id" : "2",
        "_score" : 0.2876821,
        "_source" : {
          "first_name" : "Jane",
          "last_name" : "Smith",
          "age" : 32,
          "about" : "I like to collect rock albums",
          "interests" : [
            "music"
          ]
        }
      }
    ]
  }
}
```

### _version

每个文档都有一个 `_version`号码， 这个号码在文档被改变时加一。Elasticsearch使用这个 _version 保证所有修改都被正确排序。当一个旧版本出现在新版本之后，它会被简单的忽略。


# 查询
## Search([`_search`](https://www.elastic.co/guide/en/elasticsearch/reference/6.5/query-dsl-match-all-query.html))
+ 全局 `/_search`
+ 索引 `/gb/_search`, `/gb,us/_search`, `/g*,u*/_search`
+ 类型 `/gb/user/_search`, `/gb,us/user,tweet/_search`
+ `/_all/user,tweet/_search`




## 结果
``` json
{
	"hits": { # 实际结果
		"total": 14, # 匹配到的文档总数
		"hits": [{
			"_index": "us", # 索引
			"_type": "tweet", # 类型
			"_id": "7", # ID
			"_score": 1, # 排序的分数
			"_source": { # 实际的内容
				"date": "2014-09-17",
				"name": "John Smith",
				"tweet": "The Query DSL is really powerful and flexible",
				"user_id": 2
			}
		}],
		"max_score": 1 # 是所有文档匹配查询中 _score 的最大值
	},
	"took": 4, # 搜索请求花费的毫秒数。
	"_shards": {
		"failed": 0, # 失败的
		"successful": 10, # 成功的
		"total": 10 # 参与查询的分片数
	},
	"timed_out": false # 查询超时与否 `GET /_search?timeout=10ms`
}
```

+  _score 字段，这是相关性得分(`relevance score`)，它衡量了文档与查询的匹配程度

### 分页
```
GET /_search?size=5
GET /_search?size=5&from=5
GET /_search?size=5&from=10
```

### 检索一部分字段
```
GET /website/blog/123?_source=title,text
```
### 不要元数据
```
GET /website/blog/123/_source
```
### 是否存在
```
HEAD /megacorp/employee/1
```

### 批量查询
```
POST /_mget
{
  "docs": [
    {
      "_index": "megacorp",
      "_type": "employee",
      "_id": 2
    },
    {
      "_index": "megacorp",
      "_type": "employee",
      "_id": 1,
      "_source": "first_name"
    }
  ]
}
```
```
POST /megacorp/employee/_mget
{
  "ids": [
    "2",
    "1"
  ]
}
```
```
POST /megacorp/employee/_mget
{
  "docs": [
    {
      "_id": 2
    },
    {
      "_id": 1,
      "_source": "first_name"
    }
  ]
}
```
## 全文搜索
当不指定key时, elasticsearch会从_all的key中做全文搜索

Elasticsearch把所有字符串字段值连接起来放在一个大字符串中，它被索引为一个特殊的字段 `_all`


# 创建/更新/删除
```
PUT /website/blog/123
{
  "title": "My first blog entry",
  "text": "Just trying this out...",
  "date": "2014/01/01"
}
```

文档在Elasticsearch中是不可变的, 如果已经存在,那么会生成一份新的, 同时递增`_version`, 留意下面的`_version` 和 `result`

``` json
{
  "_index" : "megacorp",
  "_type" : "employee",
  "_id" : "1",
  "_version" : 2,
  "result" : "updated",
}
```

如果我们的数据没有自然ID，我们可以让Elasticsearch自动为我们生成(注意`POST`)

```
POST /website/blog/
{
  "title": "My second blog entry",
  "text": "Still trying this out...",
  "date": "2014/01/01"
}
```

## 只创建不更新

简单的方式是使用 POST 方法让Elasticsearch自动生成唯一 _id

其他方法

+ PUT /website/blog/123?op_type=create
+ PUT /website/blog/123/_create

成功响应状态码是 201 Created 。 失败将返回 409 Conflict 

## 删除
```
DELETE /website/blog/123
```

## 冲突
> elasticsearch 使用`_version`做乐观锁控制

index 、 get 、 delete 请求时，我们指出每个文档都有一个 _version 号码，这个号码在文档被改变时加一。Elasticsearch使用这个 _version 保证所有修改都被正确排序。当一个旧版本出现在新版本之后，它会被简单的忽略。

我们只希望文档的 _version 是 1 时更新才生效。
```
PUT /website/blog/1?version=1
{
  "title": "My first blog entry",
  "text": "Starting to get the hang of this..."
}
```

### 使用外部版本控制系统
可以在Elasticsearch的查询字符串后面添加`version_type=external` 来使用这些版本号。

```
PUT /website/blog/2?version=5&version_type=external
{
  "title": "My first external blog entry",
  "text": "Starting to get the hang of this..."
}
```

> 外部版本号与之前说的内部版本号在处理的时候有些不同。它不再检查 _version 是否与请求中指定的`一致`，而是检查是否`小于`指定的版本。如果请求成功，外部版本号就会被存储到 _version 中。

### 重试
乐观锁失败的情况下, 通常可以重试
```
POST /website/pageviews/1/_update?retry_on_conflict=5
{
  "script" : "ctx._source.views+=1",
  "upsert": {
    "views": 0
  }
}
```


## 局部更新
```
POST /website/blog/1/_update
{
  "doc" : {
    "tags" : [ "testing" ],
    "views": 0
  }
}
```

[`使用Groovy脚本?`](https://github.com/elastic/elasticsearch-groovy)

## 批量更新

```
POST /_bulk
{ "delete": { "_index": "website", "_type": "blog", "_id": "123" }}
{ "create": { "_index": "website", "_type": "blog", "_id": "123" }}
{ "title": "My first blog post" }
{ "index": { "_index": "website", "_type": "blog" }}
{ "title": "My second blog post" }
```
```
POST /website/_bulk
{ "index": { "_type": "log" }}
{ "event": "User logged in" }
```

> 每个子请求都被独立的执行，所以一个子请求的错误并不影响其它请求。如果任何一个请求失败，顶层的 error 标记将被设置为 true ，然后错误的细节将在相应的请求中被报告

|行为| 解释 |
|-|-|
|create| 当文档不存在时创建之 |
|index| 创建新文档或替换已有文档 |
|update| 局部更新文档 |
|delete| 删除一个文档|

### 为什么不用一个大json
便于服务器流式操作, Elasticsearch从网络缓冲区中一行一行的直接读取数据, 并分发到不同的node去处理, 而无需等待所有的内容并序列化,造成额外的内存开销,以及时间上的延误

# 映射(mapping)/分析(analysis)
+ 映射(mapping)机制用于进行字段类型确认，将每个字段匹配为一种确定的数据类型( string , number , booleans , date 等)。
+ 分析(analysis)机制用于进行全文文本(Full Text)的分词，以建立供搜索用的反向索引。

## Type
类型类似于rdm的表(table), 每个表有自己的定义(schema), 只不过Elasticsearch会对字段类型进行猜测，动态生成了字段和类型的映射关系, 如果没有明确定义。

> 最新的elasticsearch干掉了这个概念

比如2020-03-09 elasticsearch可能推导为日期格式, 那么本字段就不会匹配`2020`/`03`/`09`的任一个

当做全局搜索时, 从_all中匹配, 而后者会将所有字段转成字符串, 所以能匹配上

```
GET /_search?q=2020 # 12 个结果
GET /_search?q=2020-03-09 # 还是 12 个结果 !(只需要匹配2020就算, 只不过匹配度(score)略后)
GET /_search?q=date:2020-03-09 # 1 一个结果
GET /_search?q=date:2020 # 0 个结果 !
```

获取type的schema

```
GET /megacorp/_mapping/employee
```
``` json
{
  "megacorp" : {
    "mappings" : {
      "employee" : {
        "properties" : {
          "about" : {
            "type" : "text",
            "fields" : {
              "keyword" : {
                "type" : "keyword",
                "ignore_above" : 256
              }
            }
          },
          "age" : {
            "type" : "long"
          },
          "first_name" : {
            "type" : "text",
            "fields" : {
              "keyword" : {
                "type" : "keyword",
                "ignore_above" : 256
              }
            }
          },
          "interests" : {
            "type" : "text",
            "fields" : {
              "keyword" : {
                "type" : "keyword",
                "ignore_above" : 256
              }
            },
            "fielddata" : true
          }
        }
      }
    }
  }
}
```

|类型| 表示的数据类型|
|-|-|
|String| string|
|Whole number| byte , short , integer , long|
|Floating point| float , double|
|Boolean| boolean|
|Date| date|

### 自定义类型
自定义类型可以使你完成以下几点：

+ 区分全文（full text）字符串字段和准确字符串字段(就是分词与不分词，全文的一般要分词，准确的就不需要分词)
+ 使用特定语言的分析器(例如中文、英文、阿拉伯语，不同文字的断字、断词方式的差异)
+ 优化部分匹配字段
+ 指定自定义日期格式
+ ...

#### type
```
{
  "number_of_clicks": {
    "type": "integer"
    "index": "not_analyzed"
    "analyzer": "english"
  }
}
```
#### index

|值 | 解释|
|-|-|
|analyzed | 首先分析这个字符串，然后索引。换言之，以全文形式索引此字段。|
|not_analyzed | 索引这个字段，使之可以被搜索，但是索引内容和指定值一样。不分析此字段。|
|no  |不索引这个字段。这个字段不能为搜索到。|


### 索引
每一种核心数据类型(strings, numbers, booleans及dates)以不同的方式进行索引, 在Elasticsearch中他们是被区别对待的。

#### 确切值(Exact values) vs. 全文文本(Full text)

Elasticsearch中的数据可以大致分为两种类型：`确切值` 及 `全文文本`

确切值是确定的，正如它的名字一样。比如一个date或用户ID，也可以包含更多的字符串比如username或email地址。确切值 "Foo" 和 "foo" 就并不相同。确切值 2014 和 2014-09-15 也不相同。确切值是很容易查询的，因为结果是二进制的 -- 要么匹配，要么不匹配。

全文文本，从另一个角度来说是文本化的数据(常常以人类的语言书写)，比如英语单数/复数, 近义词 或者依赖上下文推导的(鼠标/老鼠), 而对于全文数据的查询来说, 存在一个匹配程度的问题

+ 一个针对 "UK" 的查询将返回涉及 "United Kingdom" 的文档
+ 一个针对 "jump" 的查询同时能够匹配 "jumped" ， "jumps" ， "jumping" 甚至 "leap"
+ "johnny walker" 也能匹配 "Johnnie Walker" ， "johnnie depp" 及 "Johnny Depp"
+ "fox news hunting" 能返回有关hunting on Fox News的故事，而 "fox hunting news" 也能返回关于fox hunting的新闻故事。


## 分析
为了方便在全文文本字段中进行这些类型的查询，Elasticsearch首先对文本分析(analyzes)

+ 使用`字符过滤器(character filter)`去掉无意义词(啊/哦/额/the 等)
+ `分词(analysis)`
+ 转换(复数->单数, 同义词 等等)

`分析器`就是处理这些步骤的功能逻辑

### 内建分析器
+ 标准分析器, 标准分析器是Elasticsearch默认使用的分析器。对于文本分析，它对于任何语言都是最佳选择, 它根据Unicode Consortium的定义的单词边界(word boundaries)来切分文本，然后去掉大部分标点符号。最后，把所有词转为小写。
+ 简单分析器, 简单分析器将非单个字母的文本切分，然后把每个词转为小写
+ 空格分析器, 空格分析器依据空格切分文本。它不转换小写。
+ 语言分析器
  - english 分析器 自带一套英语停用词库——像 and 或 the 这些与语义无关的通用词。
  - 中文1 <https://github.com/medcl/elasticsearch-analysis-pinyin>
  - 中文2 <https://github.com/medcl/elasticsearch-analysis-ik>

### 测试验证
```
GET /_analyze
{
  "analyzer" : "standard",
  "text" : "Text to analyze!"
}
```
``` json
{
  "tokens" : [
    {
      "token" : "text",
      "start_offset" : 0,
      "end_offset" : 4,
      "type" : "<ALPHANUM>",
      "position" : 0
    },
    {
      "token" : "to",
      "start_offset" : 5,
      "end_offset" : 7,
      "type" : "<ALPHANUM>",
      "position" : 1
    },
    {
      "token" : "analyze",
      "start_offset" : 8,
      "end_offset" : 15,
      "type" : "<ALPHANUM>",
      "position" : 2
    }
  ]
}
```

## 指定分析器
当Elasticsearch在你的文档中探测到一个新的字符串字段，它将自动设置它为全文 string 字段并用 standard 分析器分析。

创建一个新索引，指定 tweet 字段的分析器为 english 
```
PUT /gb
{
  "mappings": {
    "tweet": {
      "properties": {
        "tweet": {
          "type": "string",
          "analyzer": "english"
        },
        "date": {
          "type": "date"
        },
        "name": {
          "type": "string"
        },
        "user_id": {
          "type": "long"
        }
      }
    }
  }
}
```

定在 tweet 的映射中增加一个新的 `not_analyzed` 类型的文本字段，叫做 tag
```
PUT /gb/_mapping/tweet
{
  "properties": {
    "tag": {
      "type": "string",
      "index": "not_analyzed"
    }
  }
}
```


# 复杂搜索/聚合

+ <https://www.elastic.co/guide/en/elasticsearch/reference/6.8/query-dsl-bool-query.html>
+ <https://www.elastic.co/guide/en/elasticsearch/reference/6.5/agg-metadata.html>

# 集群和节点
+ 节点(node)是一个运行着的Elasticsearch实例
+ 集群(cluster)是一组具有相同 cluster.name 的节点集合，
+ 分片(shards), 个最小级别“工作单元(worker unit)”,它只是保存了索引中所有数据的一部分, 索引(index)——一个存储关联数据的地方。实际上，索引只是一个用来指向一个或多个分片(shards)的“逻辑命名空间(logical namespace)”

> 做为用户，我们能够与集群中的任何节点通信。每一个节点都知道文档存在于哪个节点上，它们可以转发请求到相应的节点上。我们访问的节点负责收集各节点返回的数据，最后一起返回给客户端。

## [Node](https://www.elastic.co/guide/en/elasticsearch/reference/current/modules-node.html)
+ [Master-eligible node](https://www.elastic.co/guide/en/elasticsearch/reference/current/modules-node.html#master-node) A node that has node.master set to true (default), which makes it eligible to be elected as the master node, which controls the cluster.
+ [Data node](https://www.elastic.co/guide/en/elasticsearch/reference/current/modules-node.html#data-node) A node that has node.data set to true (default). Data nodes hold data and perform data related operations such as CRUD, search, and aggregations.
+ [Ingest node](https://www.elastic.co/guide/en/elasticsearch/reference/current/ingest.html)
+ [Machine learning node](https://www.elastic.co/guide/en/elasticsearch/reference/current/modules-node.html#ml-node)

> 在可用性要求高的集群中, 搞一个Master Only的集群?

## 分片
分片就是一个`Lucene`实例，并且它本身就是一个完整的搜索引擎。我们的文档存储在分片中，并且在分片中被索引，但是我们的应用程序不会直接与它们通信，取而代之的是，直接与索引通信。

+ 主要分片(primary shard)
+ 复制分片(replica shard)，复制分片只是主分片的一个副本，它可以防止硬件故障导致的数据丢失，同时可以提供读请求，比如搜索或者从别的shard取回文档。

``` bash
PUT /blogs
{
  "settings" : {
    "number_of_shards" : 3, # 主要分片数
    "number_of_replicas" : 1 # 每个主要分片的副本数量
  }
}
```

> 文档存储在分片中，然后分片分配到你集群中的节点上。当你的集群扩容或缩小，Elasticsearch将会自动在你的节点间迁移分片，以使集群保持平衡。当索引创建完成的时候，主分片的数量就固定了，但是复制分片的数量可以随时调整。

### 集群状态

因为测试环境只有一个节点, 所以复制分片(replica shards)全部都不可用(unassigned 状态)

> 在同一个节点上保存相同的数据副本是没有必要的，如果这个节点故障了，那所有的数据副本也会丢失


``` bash
GET /_cluster/health
# resp
{
  "cluster_name" : "docker-cluster",
  "status" : "yellow", <----
  "timed_out" : false,
  "number_of_nodes" : 1,
  "number_of_data_nodes" : 1,
  ...
}
```
|颜色| 意义|
|-|-|
|green| 所有主要分片和复制分片都可用|
|yellow| 所有主要分片可用，但不是所有复制分片都可用|
|red| 不是所有的主要分片都可用|

接上,假如副本数量是1, 当新加一个node后, 那么elasticsearch会自动创建`复制分片`, 也可能转移部分`主分片`过去, 在当前节点分配复制分片, 此时所有的分片都已经可用, 那么健康状态就是`green`

此时的集群就相当于实现了`横向扩展`, (*请求任何一个节点都能读写文档*), `高可用`(*一个节点挂了, 还有另外一份, 并且支持从复制分片重新恢复主分片*)

### 横向扩展
``` bash
PUT /blogs/_settings
{
  "number_of_replicas" : 2
}
```
在保持两个node的情况下, 因为没有足够的node分配(复制)分片, 所以集群状态又将变黄, 新加node后, 又将变绿, 此时集群的负载能力将进一步增强

### 故障恢复
假如所有的节点都支持master选举和数据存储

现在master节点挂了, 那么集群将变红, 因为不是所有的主分片都可用(*如果master有主分片*), 接下来

+ 选择产生新的master节点
+ 把丢失的主分片对应的复制分片升级成主分片(选择策略? 如果有多个复制分片)

# 路由
`新的版本的实现有改动?`

当你索引一个文档，它被存储在单独一个主分片上。Elasticsearch通过根据文档的key, 并使用一个路由算法(hash)来判断文档在哪个node, 简单地
```
shard = hash(routing) % number_of_primary_shards
```

routing 值是一个任意字符串，它默认是 _id 但也可以自定义。这个 routing 字符串通过哈希函数生成一个数字，然后除以主切片的数量得到一个余数(remainder)，余数的范围永远是0到number_of_primary_shards - 1 ，这个数字就是特定文档所在的分片。这也解释了为什么主分片的数量只能在创建索引时定义且不能修改：如果主分片的数量在未来改变了，所有先前的路由值就失效了，文档也就永远找不到了。 

> 或者迁移数据?

所有的文档API（ get 、 index 、 delete 、 bulk 、 update 、 mget ）都接收一个 routing 参数，它用来自定义文档到分片的映射。自定义路由值可以确保所有相关文档——例如属于同一个人的文档——被保存在同一分片上。

## 写流转
我们将接收外部请求节点称之为请求节点(requesting node)

> 当我们发送请求，最好的做法是循环通过所有节点请求，这样可以平衡负载。

+ 客户端给 `请求节点`(node1) 发送新建、索引或删除请求。
+ 找到分片对应的node, 分发给对应node(node2)
+ node2完成`主要分片`的增删改
+ node2同步修改到所有的`复制分片`
+ 依次同步返回, 最后由`请求节点`返回给客户端

### 异步
当更新完主要分片后, 数据`基本`就安全了, 等待所有的复制分片同步完成, 会拖慢客户端请求时间

update API还接受 routing 、 replication 、 consistency 和 timout 参数。

`replication` 默认的值是 sync 。这将导致主分片得到复制分片的成功响应后才返回。

如果你设置 replication 为 async ，请求在主分片上被执行后就会返回给客户端。它依旧会转发请求给复制节点，但你将不知道复制节点成功与否。

默认主分片在尝试写入时需要规定数量(quorum)或过半的分片（可以是主节点或复制节点）可用。 `consistency` 允许的值为 one （只有一个主分片）， all （所有主分片和复制分片）或者默认的 quorum 或过半分片。

当分片副本不足时会怎样？Elasticsearch会等待更多的分片出现。默认等待一分钟。如果需要，你可以设置 `timeout` 参数让它终止的更早： 100 表示100毫秒， 30s 表示30秒。

## 读流转
> 文档能够从主分片或任意一个复制分片被检索。

+ 客户端给 `请求节点`(node1) 发送查询请求。
+ node1根据文档id找到对应的node(包含主要分片/复制分片)
+ 分发请求到对应节点, 对于读请求，为了平衡负载，请求节点会为每个请求选择不同的分片——它会循环所有分片副本。
+ 等待返回

> 可能的情况是，一个被索引的文档已经存在于主分片上却还没来得及同步到复制分片上。这时复制分片会报告文档未找到，



