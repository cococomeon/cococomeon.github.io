---
layout: post
title: Elasticsearch(1)
subtitle:
categories: Elasticsearch
tags: [es]
banner: "/assets/images/banners/home.jpeg"

---

## 一. 基础概念

### 1.1 文档

1. 面向文档， 文档是最小的可搜索单位

- 日志文件中的日志项
- 一本对象的具体信息， 定时任务，电影，唱片
- 一首歌，一个pdf中的具体内容

2.  文档在es中会被序列化成JSON格式， 保存在elasticsearch中

- json对象由字段组成
- 每个字段都有对应的字段类型（字符串/数值/布尔/日期/二进制/范围类型）

3. 每一个文档都有自己的的UniqueID

- 可以自己制定
- 通过es自动生成

4. **一个json文档包含一系列的字段， 相当于数据库中的一条记录**

json文档不需要预定义格式

- 字段类型可以指定或者让es自动推算 
- 支持数组和嵌套

5. 文档中的元数据

- _index 文档所属的的索引名

- _type 文档类型

- _id 文档的唯一ID

- _source 文档的原始数据

- _all 整合所有的字段内容到这里， 现在已经废除

- version 文档的版本信息，修改一次这里会加一

- _score 文档的相关信打分

### 1.2 索引

*在es的文档语义中索引既可以做动词也可以做名词*

名词定义：索引是文档的容器，是一类文档的集合

索引包含3个概念：shard，mapping和setting

- shard     index表示逻辑空间上的一类文档集合，shard主要是体现在物理空间上的文档存储集合
- mapping   定义文档的字段类型
- setting  定义不同的数据分布

动词定义：保存一个文档到elasticsearch中的过程动作也叫索引。indexing

index的type

在7.0之前 一个index可以设置多个type，7.0开始一个index只能设置一个type ：_doc



传统数据库与elasticsearch的类比

| DBRM   | elasticsearch |
| ------ | ------------- |
| table  | index(type)   |
| row    | Document      |
| Column | field         |
| Schema | mapping       |
| sql    | Dsl           |

两者区别与使用场景：

Elasticsearch: 有全文搜索要求的，需要根据分数关联进行搜索查询的时候

Dbrm: 对事务要求比较高



### 1.3 节点

一个节点是一个elasticsearch实例，本质一个java进程，每个节点有自己的名字， 可以通过启动脚本-Enode.name指定,也可以在配置文件中进行指定



#### 1.3.1 Master-eligible node(可选举节点)

可参与选主流程的主节点，每个节点启动默认都是，可通过配置文件中的node.master=false进行指定

#### 1.3.2 Master node(主节点)

解释选举后成为主节点的节点。每个es节点上面都保存有整个集群的信息， 但是只有主节点可以修改集群的状态信息

集群的状态信息包括以下内容：

- 集群中各个节点的信息
- 集群中所有的索引和mapping还有setting的信息
- 分片的路由信

#### 1.3.3 Data node(数据节点)

可以保存数据的节点叫做数据节点，负责保存分片数据在数据拓展上起到至关重要的作用

#### 1.3.4 Coordinating Node(协调节点)

负责分发请求到每个合适的节点上面，并且最终把结果进行汇聚到一起，所有节点默认都具有协调节点的功能

#### 1.3.5 Hot & Warm node()

hot节点就是硬件资源比较好的节点，负责更多的数据处理任务

warm就是机器配置相对较低的节点， 这些节点就会负责存储一些相对比较旧一点的数据

#### 1.3.6 Mechine Learning Node

机器学习节点，负责跑机器学习的JOB，用来检测异常

#### 1.3.7 Tribe Node

连接到不同的集群的节点， 在5.3开始淘汰， 使用elasticsearch中的一个Cross Cluster Search的功能进行替代

### 1.4 分片

Primary shard(主分片)，主要用来解决数据的水平拓展性问题，通过分片将数据打散分布到集群中所有的不同的集群中。

- 主分片数在创建一个集群的时候就应该指定，且后面不可以继续进行修改。处分reindex
- 一个主分片就是一个运行的lucene实例

在创建一个索引的时候主分片已经制定好， 比如是3，同时集群中现有的数据节点也是3，那些这三个分片数据都会存在这三个节点上面， 及时后面添加节点进来也写分片数据也还是会保存在原来的这三个节点上面， 不能起到水平拓展的效果。

Replica shard(复制分片)，主要用来解决数据的高可用性问题， 这个就是主分片的副本



### 1.5 集群

elasticsearch是一个分布式的搜索系统。

### 1.6 倒排索引

正排索引就是文档id到内容，倒排索引就是文档内容到id的过程



## 二. mapping

### 2.1 什么是mapping

mapping类似数据库中的schema定义，作用：

- 定义字段的名称
- 定义字段的数据类型
- 字段的[倒排序索引](https://www.elastic.co/guide/cn/elasticsearch/guide/current/inverted-index.html)的相关配置
- mapping会把json文档映射成lucene所需要的扁平化格式文档

一个mapping属于一个索引的type，每个文档都有自己的一个type

### 2.2 字段类型

1. 简单类型
   - text/keyword
   - date
   - Integer/floating
   - Boolean
   - Ipv4/ipv6
2. 复杂类型
   - 对象类型、嵌套类型
3. 特殊类型
   - geo_point&geo_shape 



### 2.3 Dynamic Mapping

作用：写入文档的时候，如果索引不存在，会自动创建索引和

优点：无序手动定义mapping，es会自动推算出字段类型

缺点：有的复杂类型不一定推算正确，当字段的类型设置如果不对的时候会导致一些功能无法正常使用， 例如一些range查询

#### 2.3.1 原理

es动态映射类型的自动识别机制的原理是根据json字符串语法进行识别定义的，根据输入的字符串如果是一个json字符串就可以自动识别类型

| json类型 | elasticsearch类型                                            |
| -------- | ------------------------------------------------------------ |
| 字符串   | 1、匹配日期格式 设置为Date  2、数字配置为float或者long，需要打开配置支持 3、设置为text并且增加keyword子字段 |
| 布尔值   | boolean                                                      |
| 浮点数   | float                                                        |
| 整数     | long                                                         |
| 对象     | Object                                                       |
| 数组     | 第一个非空数值类型决定                                       |
| 空值     | 忽略                                                         |



### 2.4 Mapping字段的修改

- 新增字段

  - Dynamic设置为true的时候， mapping也会被更新，新增字段可以被索引
  - Dynamic设置为false的时候，mapping不会被更新，新增字段不可以被索引
  - Dynamic设置为stric的时候，新增字段会报错

- 已有字段

  - 一旦写入，字段类型便不可以再修改，如果需要修改则需要reindex，重建索引、

  

### 2.5 显式mapping 定义

#### 2.5.1 定义方式

PUT INDEX{

​	"mappings":{

​		//

​	}

}

#### 2.5.2 自定义建议

- 通过[官方文档]()纯手写定义
- 通过写入一些样本数据生成动态的mapping定义，然后再根据api或者这个动态的mapping定义自己的映射
- 删除索引

### 2.6 常用mapping属性

- Index 定义字段是否被索引

  ```java
  put index{
    "mapping":{
      "properties":{
        "属性名":{
          "type":"",
          "index":false,
          "index_options":offset,
          "null_value":"NULL",
          "copy_to":"new field name"
        }
      }
    }
  }
  ```

  

- Index_options 

  es中的索引是通过建立倒排索引实现的，倒排索引记录的级别又分为四种，可以通过Index option控制这个级别， 每个级别倒排索引记录的内容都是不一样的

  - docs : 记录doc id
  - freqs: 记录doc id 和 term frequenices
  - positions：记录doc id、term frequency、term position
  - offset：doc id、term frequency、term position、character offects

  越往下记录的内容就越多，占用的存储空间就越大，默认text格式的字段为positions，其他为docs

- null_value

  当插入索引的时候，字段中存在null的时候，需要对null值进行搜索的时候，可以用定义的那个NULL字符串进行搜索，但是实际的_source中的字段值还是一个真正的null

- copy_to

  当插入索引的时候对索引中的一些字段拷贝到一个别名字段中，满足通过这个逻辑别名字段搜索这些关联字段，copy_to的字段不会出现在_source中

- 数组类型

  Elasticsearch 中专门提供数组类型， 任何字段都可以包含相同的类型的数据的多个值组成数组

  PUT /index/_doc/docid

  {

  ​	"name":"",

  ​	"array_addr":["guangzhou","shenzhen"] //这里mapping看这个索引的定义时，array_addr这个字段的类型依然会是text

  }

- 多字段类型

  

  Exact Value VS Full Text

  Exact value表示确切的一个结构化的值，不需要做分词处理，通常用keyword进行标识某个字段为确切值

  Full Text 表示一个非结构化的文本数据，会，通常使用 text并且没有keyword字段进行标识，如日志或文本数据

  两者的区别，精确值在做倒排索引的时候不需要做特殊的分词处理

  

  

## 三. Analysis&Analyzer(分词与分词器)

**Analysis**文本分析， 就是将文本转换为一系列单词的过程， 也叫分词

**Analyzer**就是做分词这个动作的一个工具， 也叫分词器/分析器，可用elasticsearch内置，也可以自己进行定制



### 3.1 Analyzer组成

分词器就是专门做分词的组件， 分词器主要有三个部分组成

- Character Fliters （针对原始文本处理，看名字便可得知）
- Tokenizer (按照规则切分单词)
- Token Fliter （对单词进行加工， 小写，删除，增加同义词等）

elasticsearch自带的分词器：

- Stander analyzer 默认分词器，按词切分， 小写处理
- Simple analyzer



### 3.2 自定义分词器

当elasticsearch自带的分词器无法满足时， 可以自定义分词器，通过组合不同的组件实现

#### 3.2.1 Character Fliter

在Tokenizer之前，对文本做一些特殊处理，例如增加删除以及替换字符，可以配置多个Character Fliter会影响Tokenizer的positions和offset的信息。下面是一些自带的Character Fliter 

- HTML Strip - 出去html标签
- Mapping - 字符串替换
- Pattern replace - 正则匹配替换

#### 3.2.2 Tokenizer

根据一定的规则，将原始的文本切分为词（term or token)

elasticsearch中自带的一些tokenizer：

- whitespace 对空格进行分词
- standard 以连续的字母的方式进行切分，会把非数字和字母的东西都过滤掉
- uax_url_email 对url和email进行分词
- pattern 正则表达式分词
- keyword 不做任何处理，把一个字符串作为一个term输出
- pattern hierarchy 对文件路径进行分词处理

除此之外可以通过java开发一些插件，实现自己的Tokenizer

#### 3.2.3 Token Fliter

将Tokenizer输出的单次，进行二次加工，自带的Token Fliters

- Lowercase 转小写
- stop 辅助词过滤， 例如a、 the、 an等
- synonym - 添加近义词

```java
POST _analyzer{
  "tokenizer":"standard",
  "char_fliter":[{
    "type":"mapping",
    "mapping":["- => _", "(: => haha"]
  }],
  "text":"123-abc"
}
```

## 四. 搜索

elasticsearch搜索主要可以分为两类搜索， URI Search 和 Request Body Search两种

- uri search 主要表示使用http get的方式进行搜索，通过uri参数进行传搜索条件
- request body search 是指通过elasticsearch提供的，基于json的更完备的搜索语言

指定查询索引:

| 语法                   | 范围                           |
| ---------------------- | ------------------------------ |
| /_search               | 集群上面的所有的索引           |
| /index/_search         | 索引index上面的所有东西        |
| /index1,index2/_search | index1和index2上面的所有的索引 |
| /index*/_search        | index开头的所有的索引          |



### 4.1 uri search

使用q进行指定查询字符串，同时使用kv键值对进行查询

curl -xget "地址/索引名称/_search?q=service:custume"

这里就表示查询指定索引中字段service等于custume这个值的的数据

Get /cmc/_search?q=serviceName&df=service&sorlt=year:desc&from=0&size=10&timeout=1s

{

​		"profile":true

}

- q 指定查询语句，使用query string syntax
- df 搜索字段，不指定的时候会对所有的字段进行查询
- sort排序，from和size用于分页
- profile可以查看查询是如何被执行的

#### 4.1.1 query string syntax（查询表达式）

1. 指定字段查询vs泛查询

   q=title:111 vs q=aa

2. Term & phrase

   - Beatuiful mind 等价于beautiful or mind  (termquery)
   - "Beautiful mind" 等价于beautiful mind，绝对一致查询(phrase query)

3. 分组查询vs phrase查询

   - titile:(beatuiful mind) 分组查询表示 title:beautiful | title:mind
   - title="beautiful mind" phrase 强一致

4. 布尔操作

   AND / OR / NOT 或者 &&  /  ||  /  !

   - 必须大写
   - Titile:(beautiful NOT mind)

5. 分组

   \+ 表示must  \- 表示must not

   title:(+title  -notsearch)

6. 范围查询

   区间：[] 闭区间 {}开区间

   - Year:{2019 TO 2018]
   - Year:[* TO 2018]
   - Year:>2010
   - Year:(>2010&&<=2018)

7. 通配符查询

   通配符查询效率低，占用内存大，不建议使用

   ? 代表1个字符，*代表0或者多个字符

8. 正则表达式查询

9. 模糊匹配与近似查询

### 4.2 request body search

```java
curl -xget "地址/索引名称/_search" -H "Content-Type:application/json" -d {
  "_source":["","","字段名过滤"],
  "sort":[{"order_date":"desc"}],
  "from":10,
  "size":"20,
    "query":{
      "match_all":{}
    }
}
```

- source字段过滤（支持通配符）

- 排序（最好选择日期或者数字字段）

- 分页

- 脚本字段（根据原有字段生成一个新的字段）

  

### 4.3 查询表达式

- match

  ```java
  GET /index/_doc/_search{
    "query":{
      "match":"Last Christmas",  //这里的词法分析默认是OR的关系
      "operator":"AND" //可以在这里添加逻辑操作字符干预词法分析为and
    }
  }
  ```

  

- 短语搜索 match phrase

  ```java
  GET /index/_doc/_search{
    "query":{
      "match_phrase":{
        "字段名":{ //在特定字段名上搜索， 全局搜索则不要这个
          "query":"this is a query string",
          "slop":1 //这个属性表示在中间可插入指定数量的短语，比如这里1表示，this is a awsome query string也可以匹配到，中间只插入了一个短语
        }
      }
    }
  }
  ```

  

- query string （与uri search中的query string一样）

  ```java
  GET /index/_search{
    "query":{
      "query_string":{
        "default_field":"name", //同uri search 中的df，默认搜索字段
        "field":["name", "addr"], //指定搜索字段
        "query":" LI AND MING AND LIN"
      }
    }
  }
  ```

  

- simple query string 

  - 类似上面query string，会自动忽略错误的语法，同时只支持部分语法
  - 不支持AND OR NOT, 这些关键字只会当做字符串处理
  - term（词语）之间的的关系默认为OR,支持操作符operator
  - 支持部分逻辑 +代表AND |代表OR -代表NOT

  

## 五. Index templates 索引模板

### 5.1 什么是index templates?

- 帮助你设定Mapping和Setting，并按照一定的规则自动匹配到新创建的索引上
- 模板仅在一个索引被创建时，才会起作用，修改模板不会影响已创建的索引
- 可以设置多个索引模板，这些设置会被merge在一起
- 可以指定order的数值，控制merging的过程

## 5.2 index templates工作方式

当一个索引被创建时

- 应用elasticsearch默认的mapping和settting
- 应用order数值低的index templates设定
- 应用order数值高的index templates中的设定，之前的设定会被覆盖
- 应用创建索引时指定的mapping和settting，覆盖之前的那些设定

```java
PUT _template/模板名称{
  "index_patterns":["*"],
  "order":0,
  "version":1,
  "setting":{
    "number_of_shards":1,
    "number_of_replicas":1
  },
  "mapping":{
    "date_detection":false,
    "numberic_detection":true
  }
}
```



### 5.3 Dynamic Template 动态模板

前面的index template是应用在所有的index上面的，dynamic template是应用在一个具体的index上面的。

可以根据elasticsearch自动识别数据类型的特性，**结合字段名称，动态设置数据类型**的一个东西。

例如：

- 所有的字段类型都设置成keyword类型， 或者关闭keyword
- is开头的字段都设置成boolean
- long_开头的都设置成long类型

### dynamic 实践

```java
put index{
  "mapping":{
    "dynamic_templates":[{//定义在一个具体的index的mapping定义里面\
        "full_name":{//template的名字
          "path_match":"name.*",//匹配字段的规则
          "path_unmatch":"*.middle",
          "mapping":{//为匹配到的字段使用特定的mapping
            "type":"text",
            "copy_to":"full_name"
          }
        }
      }
    ]
  }
}
```



## 六. Aggregation 聚合

聚合应用实例：旅游网站中的那些搜索选项的勾选后的聚合

### 6.1 集合分类

- Bucket Aggregation - 一些满足特定条件的文档的集合
- Metric Aggregation - 一些数学运算，对文档字段进行统计分析
- Pipeline Aggregation - 对其他的聚合结果进行二次聚合
- Matrix Aggregation - 支持对多个字段的操作并提供一个结果矩阵

### 6.2 bucket & metric

bucket - 一组满足条件的文档，类似sql 中的 group

metric - 一系列的数学统计方法，类似sql中的count

#### 6.2.1 metric

- 基于数据集计算结果， 支持在字段上进行计算， 也支持在脚本（painless script) 产生的结果进行计算
- 大多都metric计算都只输出一个值
  - min/max/sum/avg/cardinality
- 部分metric支持输出多个值
  - stat 统计数，会把最高，最低，平均值返回
  - percentiles 根据百分位数输出不同的数值
  - percentiles_ranks 

#### 6.2.2 bucket

```java
GET INDEX/_search
{
	"size":0,
	"aggs":{//聚合查询
		"seletcName":{//此次查询自定义的名字
      "terms":{
        "fields":"统计字段名"//会根据这个统计字段进行统计
      }
		},
    "average_price":{
      "avg":{
        "field":"统计的字段名"
      }
		},
    "max_price":{
      "max":{
        "field":"统计的字段名"
      }
    }
	}
}
```

#### 6.2.3嵌套

在已经聚合出来的结构在再进行聚合查询

```java
GET INDEX/_search
{
	"size":0,
	"aggs":{//聚合查询
		"seletcName":{//此次查询自定义的名字
      "terms":{
        "fields":"统计字段名"//会根据这个统计字段进行统计
      },
      "aggs":{//嵌套查询
        "nestAgreeselectName":{//嵌套名称
          "term":{//对聚合出来的文档进行二次bucket聚合
            "field":"字段名"
          }
        }
      }
		}
	}
}
```

