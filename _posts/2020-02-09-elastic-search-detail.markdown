---
layout:     post
title:      "ElasticSearch 索引详解"
subtitle:   " \"教你如何使用elasticSearch进行索引\""
date:       2020-02-09 16:30:00
author:     "Weiwq"
header-img: "img/background/post-bg-js-module.jpg"
catalog: true
tags:
    - ElasticSearch
---

> 以下文档基于ElasticSearch 7.X版本，与老版本会有些出入

## 1、格式说明

elasticSearch的数据交互接口是基于http协议实现的，基本格式如下：

```java
http://localhost:9200/{index}/{type}/{id}  
```

- **index：索引名称，可以类比关系型数据库的表** 
- **type：类型名称，需要注意的是，在7.x之后，去掉了type属性，默认用“_doc”，8.x不再支持在请求中指定类型** 
- **id：即id，可以不指定，elasticSearch会自动生成**
- **文档：即对象的json序列化**
- ***元数据：即elasticSearch的数据格式，一般如下，"_source"对应的数据即为我们存储的文档***

```java
{
    "_index" :   "website",
    "_type" :    "_doc",
    "_id" :      "123",
    "_version" : 1, 
    "found" :    true,
    "_source" :  {
            "title": "My first blog entry",
            "text":  "Just trying this out...",
            "date":  "2014/01/01"
    }
}
```

请求格式说明（为了更好的理解代码，代码中"#" ，"//"表示注解）：

```java
POST http://localhost:9200/demo/_doc  # POST请求  请求路径
# 以下是请求体，请求类型为"application/json"
{"name": "小红","age": 25}
```

以上用okHttp表示为：

```java
private OkHttpClient okHttpClient = new OkHttpClient();
public String post() {
        String url = "http://localhost:9200/demo/_doc";
        String json = "{\"name\": \"小红\",\"age\": 25}";
        MediaType JSON = MediaType.parse("application/json");
        RequestBody requestBody = RequestBody.create(JSON, json);
        Request request = new Request.Builder().url(url).post(requestBody).build();
        try {
            Response response = okHttpClient.newCall(request).execute();
            // elasticSearch 执行成功会返回201
            if (response.code() == 200 || response.code() == 201) {
                return response.body().string();
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
        return "";
 }
```

## 2、基本数据操作

#### 插入数据（POST）

```java
http://localhost:9200/demo/_doc/1  # 指定id为1
http://localhost:9200/demo/_doc  # 自动生成id
{
  "name": "小红",
  "age": 25
}
```

#### 查询（GET）

```java
http://localhost:9200/demo/_doc/1?pretty # 返回元数据文档
http://localhost:9200/demo/_doc/1/_source # 返回文档
http://localhost:9200/demo/_doc/1?_source=name # 返回指定列,包含元数据
http://localhost:9200/demo/_doc/1/_source?_source=name # 返回文档指定列
http://localhost:9200/demo/_doc/_count # 查询该类型的个数
http://localhost:9200/demo/_doc/_search?size=1&from=1 # 分页查询，下标从0开始，size默认为10
http://localhost:9200/demo/_mapping # 查看“demo”索引的字段属性映射
```

#### 更新

```java
POST http://localhost:9200/demo/_doc/1 # 覆盖更新，指定需要更新的id
{
  "name": "小明",
  "age": 18
}

POST http://localhost:9200/demo/_doc/1/_update # 部分更新，指定需要更新的id
{
   "doc":{
       "name":"小红"
   }
}

# 脚本可以在 update API中用来改变 _source 的字段内容， 它在更新脚本中称为 ctx._source 。
POST http://localhost:9200/demo/_doc/1/_update # age列自增加，指定需要更新的id
{
 	"script":"ctx._source.age+=1"
}

```

添加新的**deleted**属性

```java
PUT http://localhost:9200/demo/_mapping?pretty

{
  "properties": {
      "deleted": {"type":"boolean"} # 添加新的deleted 属性，类型为Boolean
  }
}
```



#### 删除

```java
DELETE http://localhost:9200/demo/_doc/1 # 指定需要删除的id
DELETE http://localhost:9200/demo # 指定需要删除的索引
```

#### 批处理

[批处理基本操作和要求](https://www.elastic.co/guide/cn/elasticsearch/guide/current/bulk.html)

- **index和type可以在json中声明，或者直接在url中声明，如下所示（"_id"可以不指定，ES自动创建）**

```java
http://localhost:9200/demo/_doc/_bulk 
# 需要注意，每一行行尾，都必须包含"\n"，方便elasticSearch读取数据
{ "index": { "_id": "3"}}
{"name":"小明","age":29,"like":"篮球"}
{ "index": { "_id": "4" }}
{"name":"小红","age":30,"like":"排球"}

```

## 3、bool查询

如果需要执行类似sql的 AND 、OR、= 等操作，以上的 查询接口就不满足要求了，elasticSearch提供了bool索引，格式如下：

```java
POST  http://localhost:9200/demo/_doc/_search   # demo 为索引名，其他为默认接口
{
  "query": {
      "bool": {
        "must":     { "match": { "name": "小明" }}, # must 即必须包含的
        "must_not": { "match": { "name":  "小红" }}, # must_not 即必须不包含的
        "should":   { "match": { "like": "篮球" }}, # should 应该包含（可以不包含），会提高索引结果的评分
        "filter":   { "range": { "age" : { "gt" : 18 }} } # filter 过滤，作为一个索引值的范围
    }
  }
}
```

其中，range对应的含义如下:

- **gt: > 大于（greater than）**
- **lt: < 小于（less than）**
- **gte: >= 大于或等于（greater than or equal to）**
- **lte: <= 小于或等于（less than or equal to）**

如果must条件有多个怎么办？可以这样把“must”对应的值变为数组，同时“must_not”，“should”，“filter”也支持多条件查询

```java
POST  http://localhost:9200/demo/_doc/_search
{
  "query": { 
    "bool": { 
      "must": [
        { "match": { "name":   "小明"        }},
        { "match": { "age": 29}}
      ],
      "filter":   { "range": { "age" : { "gt" : 18 }} }
    }
  }
}
```

bool查询可以重复嵌套，例如，查询age是26，喜欢篮球的元数据

```java
POST  http://localhost:9200/demo/_doc/_search
{
  "query": {
       "bool":{ 
           "must":{"match": {"like": {"query": " 篮球"}}},
            "filter":{
                "bool":{
                    "must": { "match": { "age": 26 }}
                }
            }
       }
  }
}
```

如果只包含filter的bool查询，可以用"constant_score"，该查询为非评分方式，可以加快查询速度

```java
POST  http://localhost:9200/demo/_doc/_search
# "constant_score" 查询，用于取代只有 filter 语句的 bool 查询。如下是过滤"age"是"20"的元数据
{
  "query": {
    "constant_score":   {
        "filter": {
            "term": { "age": 20 }
        }
    }
  }
}

# 查找多个精确值，如查找age是20或者30的数据
{
  "query": {
    "constant_score":   {
        "filter": {
            "term": { "age": [20,30] }
        }
    }
  }
}
```

但是如果需要对text类型进行精确查找，例如查找只喜欢“篮球的”，这样就行不通了，详情可以见 [精确值查找](https://www.elastic.co/guide/cn/elasticsearch/guide/current/_finding_exact_values.html)

- **filters：过滤器(filters)速度很快，其不会计算结果的相关度，并且很容易被缓存，请尽可能使用filters**
- **term：如果作用于text，是包含而非equal的关系，如果需要用于查询text类型的字段，需要设置该字段为“not_analyzed” 详情参考[精确查询文本](https://www.elastic.co/guide/cn/elasticsearch/guide/current/_finding_exact_values.html#_term_查询文本)**
- **match_phrase：解决 analyzed字段无法精确搜索短语（例如"quick brown fox"）的问题 ，见[短语匹配](https://www.elastic.co/guide/cn/elasticsearch/guide/current/phrase-matching.html)**
- **slop：参数告诉 `match_phrase` 查询词条相隔多远时仍然能将文档视为匹配 。见[混合起来](https://www.elastic.co/guide/cn/elasticsearch/guide/current/slop.html)**

## 4、全文索引

看到这里，有同学会疑问，这不是将sql的功能照搬到elasticSearch上嘛~那我干嘛不用sql ？语法还简介些。别着急，接下来才是重点。

### 4.1 、基础的索引格式

如下是最简单的全文索引：

```java
GET http://localhost:9200/demo/_doc/_search?q=name:小红 # 直接通过"q"参数
POSt http://localhost:9200/demo/_doc/_search
# 其中 "and"表示分词全部都匹配，类似还有"or",表示有其中一个匹配就行
{
  "query": {
    "match": {
      "like": {   # 指定需要搜索的属性
        "query": "排球 篮球 ", # 需要搜索的词，分词结果与analyzer有关，后续会讲，这里先理解为“排球”，“篮球”
        "operator": "and"  # query中的词必须都包含
      }
    }
  }
}

# 如果需要至少匹配，可以传 "minimum_should_match": $percent%
{
  "query": {
    "match": {
      "like": {
        "query": "排球 篮球 羽毛球 乒乓球",
        "minimum_should_match": "75%"  // 表示至少匹配多词中3/4个
      }
    }
  }
}

```

也可以结合bool查询如下：content 字段必须包含 full 、 text 和 search 所有三个词。如果 content 字段也包含 Elasticsearch 或 Lucene ，文档会获得更高的评分 _score 。_

详情见  [查询语句提升权重](https://www.elastic.co/guide/cn/elasticsearch/guide/current/_boosting_query_clauses.html)

```java
POSt http://localhost:9200/demo/_doc/_search 
{
    "query": {
        "bool": {
            "must": {
                "match": {
                    "content": { 
                        "query":    "full text search",
                        "operator": "and"
                    }
                }
            },
            "should": [ 
                { "match": { "content": "Elasticsearch" }},
                { "match": { "content": "Lucene"        }}
            ]
        }
    }
}

```

### 4.2、analyzer

上面有提到分词（analyzer），这里就需要稍微提一下，因为全文索引的结果与分词器有很大关系，不然结果会与想象中不一样（笔者在这里踩了很多坑），问题来了，什么是分词？我们先看一个demo

```java
POST  http://localhost:9200/demo/_analyze
{
  "text":"full text search", # 需要分析的短语
   "analyzer": "standard"
}
```

"standard" 表示需要用哪种分词器去分析短语。返回结果如下，每个token就是分词后的结果

```java
{
  "tokens": [
    {
      "token": "full",
      "start_offset": 0,
      "end_offset": 4,
      "type": "<ALPHANUM>",
      "position": 0
    },
    {
      "token": "text",
      "start_offset": 5,
      "end_offset": 9,
      "type": "<ALPHANUM>",
      "position": 1
    },
    {
      "token": "search",
      "start_offset": 10,
      "end_offset": 16,
      "type": "<ALPHANUM>",
      "position": 2
    }
  ]
}
```

显然 "standard" 分词器将 "full text search" 分为三个单词，那么elasticSearch如何进行全文索引的呢？还是上面的短语，绘制成表格

| 查询╲存储 | full | text | search |
| --------- | :--- | ---- | ------ |
| full      | √    | ×    | ×      |
| text      | ×    | √    | ×      |

只要query 内容分词结果，在索引分词结果中，就算查找成功，因此分词器对于搜索结果至关重要。那如何配置analyzer呢？详情见 [配置analyzer](https://www.elastic.co/guide/en/elasticsearch/reference/7.x/configuring-analyzers.html)

如果是中文呢，standard又如何分词？试试不就知道了嘛~   这里剧透一下，中文会比较复杂一些，一个词会有多个字，并且是连接在一起，英文就不一样，都是用空格隔开，例如 "phone" 对应的中文是 ”电话“。如果用 "standard" 进行分词，会得到如下结果。

```java
{
  "tokens": [
    {
      "token": "电",
      "start_offset": 0,
      "end_offset": 1,
      "type": "<IDEOGRAPHIC>",
      "position": 0
    },
    {
      "token": "话",
      "start_offset": 1,
      "end_offset": 2,
      "type": "<IDEOGRAPHIC>",
      "position": 1
    }
  ]
}
```

它把每个字都分开了，如果我想搜索 "XX手机XX"  可能返回的结果就会包含 “XX手XXX机XXX”，就不是连接在一块了。这可愁死人啦!，没有关系，elasticSearch支持安装analyzer插件，这里找到了 “中文分词器”  [ik_smart](https://github.com/medcl/elasticsearch-analysis-ik/releases)

### 4.3、IK中文分词器

在elasticSearch安装目录下，通过如下安装，详情见[README](https://github.com/medcl/elasticsearch-analysis-ik)

```java
./bin/elasticsearch-plugin install https://github.com/medcl/elasticsearch-analysis-ik/releases/download/v6.3.0/elasticsearch-analysis-ik-6.3.0.zip
```

或者本地安装（window下）

```java
elasticsearch-7.5.1\bin\elasticsearch-plugin install file:///D:/elasticsearch/elasticsearch-7.5.1-windows-x86_64.zip
```

安装完成后，设置需要索引的字段属性（mapping）。如果之前有创建了索引，需要先将其删除，详情可以见[删除](#delete)

```java
PUT  http://localhost:9200/demo
{
  "mappings": {
     "dynamic":true, # 是否可以动态创建缺省的列
     "properties": { # 属性（固定格式）
      "like": { # 需要设置的字段
        "type":     "text", # 该字段的类型
        "analyzer": "ik_smart", # 指定创建索引的分词器
         "search_analyzer": "ik_smart" # 指定搜索的分词器
      }
    }
  }
}
```

我们可以通过如下查看索引字段的类型、分词器等映射，确定设置成功了

```java
GET http://localhost:9200/demo/_mapping # 查看“demo”索引的字段属性映射
```

再试试分词效果

```java
POST  http://localhost:9200/demo/_analyze
{
  "text":"电话",
   "analyzer": "ik_smart"
}

# 结果如下
{
  "tokens": [
    {
      "token": "电话",
      "start_offset": 0,
      "end_offset": 2,
      "type": "CN_WORD",
      "position": 0
    }
  ]
}
```

在搜索的时候可以指定分词器（如果之前已经指定了search_analyzer，就不需要再指定），这样就不会出现“XXX篮XXX球” 这样的结果了。

```java
POST  http://localhost:9200/demo/_doc/_search 
{
    "query": {
        "match": {
            "contentText":{
                "query":       "篮球 排球",
                "operator":    "and",
                "analyzer": "ik_smart"
            }
        }
    }
}
```

IK插件有两中分词器：ik_smart或者ik_max_word。

- **ik_max_word: 会将文本做最细粒度的拆分，比如会将“中华人民共和国国歌”拆分为“中华人民共和国,中华人民,中华,华人,人民共和国,人民,人,民,共和国,共和,和,国国,国歌”，会穷尽各种可能的组合，适合 Term Query；**

- **ik_smart: 会做最粗粒度的拆分，比如会将“中华人民共和国国歌”拆分为“中华人民共和国,国歌”，适合 Phrase 查询。**

## 5、排序 

在搜索结果中，我们也需要对其进行排序，这里我们对“age” 进行降序排列

```java
# 在"like"字段中搜索"篮球"，并以"age" 按照降序排列
GET http://localhost:9200/demo/_doc/_search?sort=date:desc&sort=age&q=like:篮球 
# 在"like"字段中搜索"篮球 排球"，并以"age" 按照降序排列 
POSt http://localhost:9200/demo/_doc/_search
{
  "query": {
      "match":{
          "like":{
              "query":"篮球 排球",
              "operator":"and"
          }
      }
  },
   "sort": { "age": { "order": "desc" }} # desc:降序排列;  asc:升序排列
}
```

如果有如下错误

```java
Fielddata is disabled on text fields by default. Set fielddata=true on [age] in order to load fielddata in memory by uninverting the inverted index. Note that this can however use significant memory. Alternatively use a keyword field instead.    
```

设置需要排序的列mapping

```java
http://localhost:9200/demo/_mapping  # 设置索引的mapping
{
  "properties": {
    "age": { 
      "type":     "text",
      "fielddata": true
    }
  }
}
```

更多详情见 [排序](https://www.elastic.co/guide/cn/elasticsearch/guide/current/_Sorting.html)

## 6、运维

- 查看表的健康状态：http://localhost:9200/_cat/indices?v

```java
health status index    uuid                   pri rep docs.count docs.deleted store.size pri.store.size
yellow open   v1       QsHSCQunT26UnUDDX_uqRw   1   1          1            0      6.4kb          6.4kb
yellow open   v2       wpryqVpBQe6eakz2uT79xA   1   1          1            0      6.9kb          6.9kb
```

## 7、后记

当然elasticSearch的功能远不止这些，由于篇幅关系，本文就不列举过多的接口，更多文档可以参考 [ElasticSearch文档](https://www.elastic.co/guide/en/elasticsearch/reference/7.x/elasticsearch-intro.html) 这里可以看到各个版本的文档信息。

最后，最近的新型冠状病毒尤为猖獗，疫情十分严重。这里祝愿各位读者：福如东海 、寿比南山，岁岁平安，升职加薪！

------ WeiWq 后记于 2020.02.09 广州。