# 全文搜索引擎简介


## 全文检索概念
#### 1. 什么是全文检索

对 非结构化数据 搜索/结构化数据就要全文检索，狭义的理解主要针对文本数据的搜索。
- 结构化数据：
业界指关系模型数据，即以关系数据库表形式管理的数据
- 半结构数据：
非关系模型的、有基本固定结构模式的数据，例如日志文件、XML文档、JSON文档、Email等。
-	非结构化数据：
没有固定模式的数据，如WORD、PDF、PPT、EXL，各种格式的图片、视频等。

非结构化数据是数据结构不规则或不完整，没有预定义的数据模型，不方便用数据库二维逻辑表来表现的数据。包括所有格式的办公文档、文本、图片、XML, HTML、各类报表、图像和音频/视频信息等等

#### 2. 全文检索的特点

- 相关度最高的排在最前面，官网中相关的网页排在最前面； java 
- 关键词的高亮。
- 只处理文本,不处理语义。 以单词方式进行搜索
	比如在输入框中输入“中国的首都在哪里”，搜索引擎不会以对话的形式告诉你“在北京”，而仅仅是列出包含了搜索关键字的网页。

## 全文搜索实现

### Lucene
#### 1. 概念
Lucene是apache下的一个开源的全文检索引擎工具包(一堆jar包)。它为软件开发人员提供一个简单易用的工具包（类库），以方便的在小型目标系统中实现全文检索的功能。虽然Lucene是当前业界公认的性能最好的全文搜索引擎，但是它只是一个工具包，你还需要对它进行整合封装进自己的项目中才可以使用，十分的复杂。这里对它进行一个简介。

#### 2. Lucene的工作流程

- 索引创建

将现实世界中所有的结构化和非结构化数据提取信息，创建索引的过程。那么索引里面究竟存的什么，以及如何创建索引呢？在这通过下面的例子来解答这个问题。
首先构造三个不同的句子，有长有短：

在①处分别为3个句子加上编号，然后进行分词，把被一个单词分解出来与编号对应放在②处；在搜索的过程总，对于搜索的过程中大写和小写指的都是同一个单词，在这就没有区分的必要，按规则统一变为小写放在③处；要加快搜索速度，就必须保证这些单词的排列时有一定规则，这里按照字母顺序排列后放在④处；最后再简化索引，合并相同的单词，就得到如下结果：

通常在数据库中我们都是根据文档找到内容，而这里是通过词，能够快速找到包含他的文档，这就是文档倒排链表。

以上就是lucene索引结构中最核心的部分。我们注意到关键字是按字符顺序排列的（lucene没有使用B树结构），因此lucene可以用二元搜索算法快速定位关键词。

- 索引搜索

就是得到用户的查询请求，搜索创建的索引，然后返回结果的过程。

比如我们要搜索java world两个关键词，符合java的有1,2两个文档，符合world的有1,3两个文档，在搜索引擎中直接这样排列两个词他们之间是OR的关系，出现其中一个都可以被找到，所以这里3个都会出来。全文检索中是有相关性排序的，那么结果在是怎么排列的呢？hello java world中包含两个关键字排在第一，另两个都包含一个关键字，得到结果，hello lucene world排在第二，java在最长的句子中占的权重最低排在结果集的第三。从这里可以看出相关度排序还是有一定规则的。

#### 3. Lucene的Hello-world基本使用

在 Lucene 中，索引库的创建和维护使用 IndexWriter，而索引库的搜索则使用 IndexSearcher。数据是以 document 的形式保存到 Lucene 中的，一个 document 数据可以理解为是一条数据，而 document 则是由 Field 组成，也就是字段。

- 创建索引

``` java
// 获取保存索引库的位置信息
Path path = Paths.get("/Users/chenyangm/Documents/JavaLearning/Day64_LuceneDemo/index");
// 创建索引库的目录
Directory directory = directory = FSDirectory.open(path);
String str1 = "hello world";
String str2 = "hello java";
String str3 = "Hello new Java";

// 定义分词器，这里使用最基本的分词器
IndexWriterConfig indexWriterConfig = new IndexWriterConfig(new SimpleAnalyzer());

// 创建 indexWriter 对象，写入和创建索引
IndexWriter indexWriter = new IndexWriter(directory, indexWriterConfig);

// 创建字段
TextField context1 = new TextField("context", str1, Field.Store.YES);
TextField context2 = new TextField("context", str2, Field.Store.YES);
TextField context3 = new TextField("context", str3, Field.Store.YES);

// 定义 document 对象， 一个 document 对象就相当于一条数据
Document doc1 = new Document();
Document doc2 = new Document();
Document doc3 = new Document();

doc1.add(context1);
doc2.add(context2);
doc3.add(context3);

// 写入数据，创建索引
indexWriter.addDocument(doc1);
indexWriter.addDocument(doc2);
indexWriter.addDocument(doc3);

// 提交数据
indexWriter.commit();

// 关闭
indexWriter.close();
```

- 搜索文档

在搜索文档的时候也会对查询短语进行分词，同时根据匹配的程度，进行排序和打分。

``` java
// 获取保存索引库的位置信息
Path path = Paths.get("/Users/chenyangm/Documents/JavaLearning/Day64_LuceneDemo/index");
// 创建索引库的目录
Directory directory = directory = FSDirectory.open(path);
// 创建默认解析器
QueryParser parser = new QueryParser("context", new SimpleAnalyzer());

// 查询对象
Query query = parser.parse("context:hello java");

// 通过 indexSearch 查询数据
// 创建 indexSearch 对象
IndexReader open = DirectoryReader.open(directory);
IndexSearcher indexSearcher = new IndexSearcher(open);

// 通过查询对象查询索引
TopDocs topDocs = indexSearcher.search(query, 100);

for (ScoreDoc scoreDoc : topDocs.scoreDocs) {
	float score = scoreDoc.score;
	int docIndex = scoreDoc.doc;
	Document doc = indexSearcher.doc(docIndex);
	System.out.println("索引为： " + docIndex + " ; 分数为：" + score + " ；内容为：" + doc.get("context"));
}
```

### ElasticSearch

#### 1. ElasticSearch 简介

虽然全文搜索领域，Lucene可以被认为是迄今为止最先进、性能最好的、功能最全的搜索引擎库。但是，Lucene只是一个库。想要使用它，你必须使用Java来作为开发语言并将其直接集成到你的应用中，更糟糕的是，Lucene的配置及使用非常复杂，你需要深入了解检索的相关知识来理解它是如何工作的。

实际项目中，我们建立一个网站或应用程序，并要添加搜索功能，令我们受打击的是：搜索工作是很难的。我们希望我们的搜索解决方案要快，我们希望有一个零配置和一个完全免费的搜索模式，我们希望能够简单地使用JSON/XML通过HTTP的索引数据，我们希望我们的搜索服务器始终可用，我们希望能够从一台开始并在需要扩容时方便地扩展到数百，我们要实时搜索，我们要简单的多租户，我们希望建立一个云的解决方案。

ES即为了解决原生Lucene使用的不足，优化Lucene的调用方式，并实现了高可用的分布式集群的搜索方案，其第一个版本于2010年2月出现在GitHub上并迅速成为最受欢迎的项目之一。
首先，ES的索引库管理支持依然是基于Apache Lucene(TM)的开源搜索引擎。ES也使用Java开发并使用Lucene作为其核心来实现所有索引和搜索的功能，但是它的目的是通过简单的 RESTful API来隐藏Lucene的复杂性，从而让全文搜索变得简单。

不过，ES的核心不在于Lucene，其特点更多的体现为：

- 分布式的实时文件存储，每个字段都被索引并可被搜索
- 分布式的实时分析搜索引擎
- 可以扩展到上百台服务器，处理PB级结构化或非结构化数据：KB-MB-GB-TB-PB
- 高度集成化的服务，你的应用可以通过简单的 RESTful API、各种语言的客户端甚至命令行与之交互。

#### 2. ElasticSearch使用

##### 1. 安装ElasticSearch

这里为了学习和测试，所以使用docker来进行安装，同时版本也是安装的最新版本 7.8.0 版本。

``` shell
docker run -d --name es -p 9200:9200 -p 9300:9300 -e ES_JAVA_OPTS="-Xms512m -Xmx512m" -e "discovery.type=single-node" --restart=always --name elasticsearch-dev  elasticsearch:7.8.0
```

同时也可以将 kibana 安装上，用来进行测试。

``` shell
docker run --restart=always --link elasticsearch-dev:elasticsearch -d -p 5601:5601 kibana:7.8.0
```

##### 2. ElasticSearch 简单使用

ElasticSearch使用 restful 风格的 API 来进行操作的，这里我们简单的使用一下

> Tips: 
> 需要注意的是 在 ElasticSearch 7.8 的版本中 ，_type 属性被移除了。根据官网的解释，在
> ES最初开发的时候，借用了关系型数据库的一些概念，比如说 index 类比为库， type 类比为表 
> document 则类比为 一行数据。但是在 Lucene 中，是没有 type 这个概念的，所以如果 不同 
> type 中 多个字段相同的话则会发生冲突，会在查询的时候都进行搜索。
> 
> 关于解决方案：
> 1. 使用多个index，将不同类型的数据放到不同的index中
> 2. 使用 mapping 自定义属性

###### 2.1 基本的增删改

``` json
# 创建一个index
PUT /employee

# 创建一个元素
PUT /employee/_doc/1
{
  "name": "老王",
  "age": 18
}

#批量创建元素
PUT /employee/_bulk?pretty&&refresh
{"index": {"_id": 2}}
{"name": "老孙2", "age": "19"}
{"index": {"_id": 3}}
{"name": "老孙3", "age": "20"}
{"index": {"_id": 4}}
{"name": "老王2", "age": "31"}

# 更新一个元素 这个会强制更新所有的内容，如果字段被删除了则更新后也会被删除
POST /employee/_doc/2
{
  "name": "老王1"
}

# 更新部分的文档 只会更新当前字段不会进行覆盖
POST /employee/_update/2
{
  "doc": {
    "age": 12,
    "count": 14
  }
}

# 更新部分的文档
POST /employee/_update/2
{
  "doc": {
    "count": 20
  }
}

# 使用脚本进行更新，可以进行简单的计算
POST /employee/_update/2
{
  "script": "ctx._source.count += 1"
}

# 批量获取
GET /employee/_mget
{
  "ids": ["1", "2", "3"]
}

#删除一个元素
DELETE /employee/_doc/1

#删除index
DELETE /employee/

# 获取一个元素
GET /employee/_doc/1

# 查找所有的元素
GET /employee/_search

# 以年龄进行倒序排列
GET /employee/_search
{
  "query": {"match_all": {}},
  "sort": {
    "age": "desc"
  }
}

# 批量插入大量数据
PUT /account/_bulk

# 使用uri进行简单的查询
GET /account/_search?q=account_number:1

# 使用_source 对返回的数据进行限制
GET /account/_search?q=account_number:1&&_source=account_number,firstname

```

###### 2.2 DSL查询相关

DSL 是 ES 的一种查询语法，可以对 ES 进行复杂查询。
在 ES 中，查询分为 DSL 查询和 DSL 过滤。

1. DSL查询和DSL过滤的区别

- DSL 查询
在查询上下文中，查询子句回答以下问题：“此文档与该查询子句的匹配程度如何？”。除了确定文档是否匹配之外，查询子句还计算_score元字段中的相关性得分。

只要将查询子句传递到查询参数（例如搜索API中的查询参数），查询上下文就有效。

- DSL 过滤
在过滤器上下文中，查询子句回答问题“此文档是否与此查询子句匹配？” 答案是简单的“是”或“否”，即不计算分数。 过滤器上下文主要用于过滤结构化数据，例如
>此时间戳记是否在2015年至2016年的范围内？
>状态字段设置为“已发布”吗？

常用过滤器将由Elasticsearch自动缓存，以提高性能。
只要将查询子句传递到过滤器参数（例如bool查询中的filter或must_not参数，constant_score查询中的filter参数或过滤器聚合），过滤器上下文即有效。

2. 关于 match 和 term的区别

- term 

先看看term的定义，term是代表完全匹配，也就是精确查询，搜索前不会再对搜索词进行分词拆解。可以先看几个例子:

如果执行下面的搜索
``` json
GET /account/_search
{
  "query": {
    "term": {
      "address": "kings"
      }
    }
}
```

我们可以查出来下面的数据

``` json
{
        "_index" : "account",
        "_type" : "_doc",
        "_id" : "20",
        "_score" : 5.990829,
        "_source" : {
          "age" : 36,
          "gender" : "M",
          "address" : "282 Kings Place",
          "employer" : "Scentric",
          "city" : "Ribera",
          "state" : "WA"
        }
}
```

但是如果我们搜索 `"address": "Kings"` 这个语句的话，我们搜索出来的就是一个空的文档。因为 address 这个字段在保存的时候已经进行了分词的操作，会将单词转为小写，而使用 `term` 进行查询的话，是不会对查询语句进行分词操作的，所以还是会去匹配 Kings 这个关键字，自然匹配不到。

而如果我们需要匹配多个词呢，使用 `term` 还能够帮我们匹配出来吗。答案是不能的。我们可以尝试一下

``` json
GET /account/_search
{
  "query": {
    "term": {
      "address": "kings place"
      }
    }
}
```
查出来的结果如下

``` json
{
  "took" : 1,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 0,
      "relation" : "eq"
    },
    "max_score" : null,
    "hits" : [ ]
  }
}
```

执行发现无数据，从概念上看，term属于精确匹配，只能查单个词。我想用term匹配多个词怎么做？可以使用terms来：

``` json
GET /account/_search
{
  "query": {
    "terms": {
      "address": ["kings", "place"]
      }
    }
}
```

发现全部查询出来，为什么。因为terms里的[ ] 多个是或者的关系，只要满足其中一个词就可以。想要通知满足两个词的话，就得使用bool的must来做。这里就不做演示了

- match

我们先来用用看，和上面一样我们来匹配 `kings place`。

``` json
GET /account/_search
{
  "query": {
    "match": {
      "address": "Kings Place"
    }
  }
}
```

你会发现和之前的 terms 一样，将所有的 包含了 kings 和 place 的所有文档都搜索出来了。因为match进行搜索的时候，会先进行分词拆分，拆完后，再来匹配，上面两个内容，他们address的词条为： Kings Place ，我们搜索的为Kings Place 我们进行分词处理得到为kings place ，并且属于或的关系，只要任何一个词条在里面就能匹配到。如果想 kings 和 place 同时匹配到的话，怎么做？使用 match_phrase。下面有做演示，这里就不详细描述了

3. DSL的基本使用演示。
``` json
### DSL 相关
# 以账号进行排序 默认情况下会返回前10条数据
GET /account/_search
{
  "query": {"match_all": {}},
  "sort": {
    "account_number": "asc"
  }
}

# 对单个字段进行查询
GET /account/_search
{
  "query": {
    "match": {
      "address": "mill lane"
    }
  }
}

# 会对短语进行匹配，而不是对关键字进行分词查询
GET /account/_search
{
  "query": {
    "match_phrase": {
      "address": "mill lane"
    }
  }
}

# 对查询结果进行分页
# from ： 从第几条开始，size：每页的大小
GET /account/_search
{
  "query": {
    "match": {
      "gender": "F"
    }
  },
  "from": 0,
  "size": 20
}

# 使用bool可以进行多条件判断。 must 与 should 或 must_not 非
# 年龄是40 且state不是ID的
GET /account/_search
{
  "query": {
    "bool": {
      "must": [
        {"match": {
          "age": "40"
        }}
      ],
      "must_not": [
        {"match": {
          "state": "ID"
        }}
      ]
    }
  }
}

# 进行模糊查询, 多条件查询
GET /account/_search
{
  "query": {
    "bool": {
      "must": [
        {"match": {"address": "244"}},
        {"match": {"employer": "Euron"}}
      ]
    }
  }
}

# 过滤器
GET /account/_search
{
  "query": {
    "bool": {
      "must": [
        {"match_all": {}}
      ],
      "filter": [
        {"range": {
          "balance": {
            "gte": 10000,
            "lte": 20000
          }
        }}
      ]
    }
  }
}
```

###### 2.3 Mapping

映射是定义文档及其包含的字段的存储和索引方式的过程。 例如，使用映射定义：

- 哪些字符串字段应视为全文字段。
- 哪些字段包含数字，日期或地理位置。
- 日期的[格式](https://www.elastic.co/guide/en/elasticsearch/reference/current/mapping-date-format.html)。
- 自定义规则来控制[动态添加字段的映射](https://www.elastic.co/guide/en/elasticsearch/reference/current/dynamic-mapping.html)。

映射定义具有：
- 元字段
元字段用于自定义如何处理文档的相关元数据。 元字段的示例包括文档的_index，_id和_source字段。
- 字段或属性
映射包含与文档相关的字段或属性的列表。

1. 字段的数据类型

每个字段的数据类型可以是：

- 一个简单的类型，例如text, keyword, date, long, double, boolean or ip

- 一种支持JSON的分层特性的类型，例如object or nested

- 或特殊的类型，例如 geo_point, geo_shape, or completion.

为不同的目的以不同的方式对同一字段建立索引通常很有用。 例如，可以将字符串字段索引为全文搜索的文本字段，以及作为排序或聚合的关键字字段。 另外，您可以使用标准分析器，英语分析器和法语分析器为字符串字段建立索引。

这是多字段的目的。 大多数数据类型通过fields参数支持多字段。

2. 创建一个index，同时制定 mapping 类型

``` json
PUT /testindex
{
  "mappings": {
    "properties": {
      "age":    { "type": "integer" }, 
      "email":  { "type": "keyword"  },
      "name":   { "type": "text"  }
    }
  }
}
```

3. 给已有的 index 添加 mapping

``` json
PUT /testindex/_mapping
{
  "properties": {
    "create_date": {"type": "date"}
  }
}
```

4. 可以给字段添加一个显示的 type 属性，来代替之前版本的 _type 属性

``` json
PUT twitter
{
  "mappings": {
    "_doc": {
      "properties": {
        "type": { "type": "keyword" }, 
        "name": { "type": "text" },
        "user_name": { "type": "keyword" },
        "email": { "type": "keyword" },
        "content": { "type": "text" },
        "tweeted_at": { "type": "date" }
      }
    }
  }
}
```

#### 3. 使用JAVA API 操作 ElasticSearch

ElasticSearch给我们提供了两种API，一种是 RESTFul 的API 一种是 TransportClient 的API。在 8.0 之后的版本中，TransportClient 的API 将会被移除，而只会保留 Restful 风格的 API了，需要注意。在这里我们使用 HighRestApi 进行演示。HighRestAPI 其实就是对 RESTAPI 的进一步封装

1. 引入对应版本的 jar 包，这里使用maven

``` xml
<dependency>
    <groupId>org.elasticsearch.client</groupId>
    <artifactId>elasticsearch-rest-high-level-client</artifactId>
    <version>7.8.0</version>
</dependency>
```

同时你还需要引入 log4j 的依赖

``` xml
<dependency>
    <groupId>org.apache.logging.log4j</groupId>
    <artifactId>log4j-core</artifactId>
    <version>2.11.1</version>
</dependency>
```

在 classpath 中添加对应的 properties 配置文件 `log4j2.properties`

``` properties
appender.console.type = Console
appender.console.name = console
appender.console.layout.type = PatternLayout
appender.console.layout.pattern = [%d{ISO8601}][%-5p][%-25c] %marker%m%n

rootLogger.level = info
rootLogger.appenderRef.console.ref = console
```

2. 基本的增删改查使用

- 创建客户端

`RestHighLevelClient` 的客户端实例需要依赖于  `REST low-level client builder` 来创建。

``` java
RestHighLevelClient client = new RestHighLevelClient(
        RestClient.builder(
            new HttpHost("localhost", 9200, "http"),
            new HttpHost("localhost", 9201, "http")));
```

- 关闭客户端

高级客户端将基于提供的构建器在内部创建用于执行请求的低级客户端。 该低级客户端维护一个连接池并启动一些线程，因此您应该在完全正确地使用它之后关闭高级客户端，这反过来又将关闭内部低级客户端以释放这些资源。

``` java
client.close();
```

- 创建索引和保存数据

使用 indexAPI 需要构造一个 IndexRequest 对象。

``` java
// 创建 map 对象保存文档数据
Map<String, Object> map = new HashMap<String, Object>();
map.put("id", i);
map.put("username", "rei");
map.put("sex", 1);
map.put("age", 15);

// 创建 index 的请求对象
IndexRequest request = new IndexRequest("javatest").id("1").source(map);
```

> 1. 使用有参构造，传入 索引库的名称，如果不存在会自动创建
> 2. 使用 id 来指定当前需要插入的文档的 id，如果不使用这个参数会创建默认 id
> 3. 使用 source 来传入需要添加的文档内容，支持方式，这里演示的是 map 类型

构造完 IndexRequest 后可以通过 客户端来发送请求了。

``` java
IndexResponse index = client.index(request, RequestOptions.DEFAULT);
```
这里会拿到一个 响应。可以通过 `getResult` 拿到执行的结果。

- 查询一篇文档

同上使用getAPi 也需要构造一个 GetRequest 对象。传入要查的索引库以及文档 id

``` java
// 创建获取请求
GetRequest getRequest = new GetRequest("javatest", "1");
```

执行请求

``` java
// 执行请求
GetResponse getResponse = client.get(getRequest, RequestOptions.DEFAULT);
```

通过 `getSource()` 可以获取一个map类型的文档结果，而使用 `getSourceAsString()` 可以得到一个 `json` 字符串

- 更新一篇文档

构造 `UpdateRequest` 对象，传入索引库和文档 id。默认情况下会使用 局部更新。也就是不回去删除没有的字段

``` java
Map<String, Object> map = new HashMap<String, Object>();
map.put("username", "rei");
map.put("age", 18);
map.put("address", "新东京");
UpdateRequest updateRequest = new UpdateRequest("javatest", "Fo-k1nIBqD72ELOU5jJC").doc(map);
```

执行请求

``` java
UpdateResponse update = client.update(updateRequest, RequestOptions.DEFAULT);
```

同 `IndexResponse`，使用 `getResult` 可以拿到对应的执行结果。

- 删除一篇文档

构造 `DeleteRequest` 对象，传入索引库和需要删除文档的 ID

``` java
DeleteRequest deleteRequest = new DeleteRequest("javatest", "1");
```

执行请求

``` java
DeleteResponse delete = client.delete(deleteRequest, RequestOptions.DEFAULT);
```
同 `IndexResponse`，使用 `getResult` 可以拿到对应的执行结果。

3. DSL查询基本使用

DSL查询是使用 ES 的一个重点，这里会拿一个例子做简单的演示。和基本的 CRUD 一样，要使用 DSL 查询需要构造一个 `SearchRequest` 来发送查询请求，同时还需要构造一些 `QueryBuilder` 对象来构建查询参数。下面是查询的需求：

查询 `username` 中含有 rei 的，年龄在 10-20 之间性别为 男 的数据。同时进行分页，分页大小为 20 页，查询 第1页。

需求分析：`username` 为匹配查询，可以使用 `match`。其他查询条件可以使用 `filter`。一同时有多个查询条件，需要使用 bool。

``` java
// 构造 SearchRequest 对象，传入需要查询的索引库，如果使用五参构造会默认查询所有的索引库
SearchRequest searchRequest = new SearchRequest("javatest");
// 创建 文档搜索建造类
SearchSourceBuilder searchSourceBuilder = new SearchSourceBuilder();
// 创建一个 bool搜索
BoolQueryBuilder boolQueryBuilder = QueryBuilders.boolQuery();
// 匹配 username 为 kei 的内容
boolQueryBuilder.must(QueryBuilders.matchQuery("username", "rei"));
// 创建一个过滤器
List<QueryBuilder> filter = boolQueryBuilder.filter();
// 匹配性别为男的
filter.add(QueryBuilders.termQuery("sex", 1));
// 年龄在10-20之间的
filter.add(QueryBuilders.rangeQuery("age").gte(10).lte(20));
// 传入查询条件
searchSourceBuilder.query(boolQueryBuilder);
// 分页
searchSourceBuilder.from(0).size(20);
searchRequest.source(searchSourceBuilder);
// 执行搜索
SearchResponse search = client.search(searchRequest, RequestOptions.DEFAULT);
// 获取搜索到的结果
SearchHits hits = search.getHits();
// 搜索到的总数
System.out.println("搜索到的结果总数为："+ hits.getTotalHits().value);
// 搜索到的文档内容
SearchHit[] hits1 = hits.getHits();
for (SearchHit documentFields : hits1) {
	System.out.println(documentFields.getSourceAsString());
}
```

> 1. 使用 searchSourceBuilder 可以使用更多的控制查询的选项。
> 2. 使用 QueryBuilders 工具类可以更加方便的构建各种类型的查询

关于 java API 的使用暂时介绍道这里，更加详细的内容可以查看 [官方文档](https://www.elastic.co/guide/en/elasticsearch/client/java-rest/current/java-rest-overview.html)

##### 4. 集成 SpringBoot 使用 Elasticsearch

环境版本：

|组件|版本|
|--|--|
|spring-boot|2.0.5-Release|
|spring-data|3.0.10.RELEASE|
|elasticsearch|5.5|

由于 spring-data 的版本和 elasticsearch 的版本是绑定的， 所以这里将 elasticsearch 换成了 5.5 可能有些地方会和 7.8 有些出入。

1. 导入 spring-data 的依赖

``` xml
<parent>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-parent</artifactId>
  <version>2.0.5.RELEASE</version>
</parent>
<!--集成elasticsearch-->
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-data-elasticsearch</artifactId>
</dependency>
<!--引入 test-->
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-test</artifactId>
</dependency>
```

2. 修改 application.yml 配置文件

``` yml
spring:
  data:
    elasticsearch:
      cluster-name: elasticsearch
      cluster-nodes: centos-linux:9300
```

>需要注意的是，如果你的 es 是通过 docker 或者 部署在远端上的话，直接访问可能会出现 `NoNodeAvailableException` 异常，也就是没有办法链接到你的那个集群上去。这里有几个地方可以排查下：
>1. 默认情况下的话，spring-data 会连接到集群名为`elasticsearch`的节点上，如果你在部署的时候修改了集群名，这里需要修改一下
>2. 如果是远端访问，有可能是因为  es 默认没有开启 `transportclient` 的访问权限，可以修改配置文件 `config/elasticsearch.yml` 将 `transport.host: 0.0.0.0` 这个配置放开即可。

2. 创建实体类

在 spring-data 中，是通过注解来实现 elaticsearch 的映射关系的。

``` java
// 当前实体类的索引库的名称和类型
@Document(indexName = "test", type = "test")
public class TestDoc {

    @Id //对应文档的id
    private Long id;

    @Field(type = FieldType.Keyword)    //指定为 不分词
    private String userName;

    private int age;

    @Field(type =FieldType.Text,analyzer = "ik_max_word",searchAnalyzer = "ik_max_word") //指定类型为text，需要分词，同时指定分词器
    private String intro;
}
```

3. 使用 ElasticsearchTemplate 创建索引库，添加映射

``` java
@Autowired
private ElasticsearchTemplate elasticsearchTemplate;

@Test
public void testElasticsearchIndex() throws Exception {
// 创建索引和map
  elasticsearchTemplate.createIndex(TestDoc.class);
  elasticsearchTemplate.putMapping(TestDoc.class);
}
```

4. 创建操作接口

spring-data 中提供了一系列的 Repository 接口来实现对数据的操作。Spring Data存储库的中心接口是Repository。 它需要实体类以及实体类的id类型作为类型参数来管理。 该接口主要用作标记接口，以捕获要使用的类型并帮助您发现扩展该接口的接口。 CrudRepository为正在管理的实体类提供了基本的CRUD功能。

PagingAndSortingRepository 接口提供了基本的 分页和排序通能，它同时还继承了 CrudRepository。

在这里，我们使用 ElasticsearchRepository 接口，它除了继承了 PagingAndSortingRepository 接口提供了基础的 crud 之外，还提供了一些方法可以实现 DSL 查询和过滤。不过 spring-data 提供的接口都加了 `NoRepositoryBean` 注解，不让我们直接注入，所以我们可以自己创建一些接口继承，并将其交给spring管理。

``` java
//操作接口
/**
*  由于 spring data 提供的api 无法被创建为bean交给spring管理，所以我们自己写一个接口，并将其交给spring管理
*/
@Repository
public interface TestElasticsearchRepository extends ElasticsearchRepository<TestDoc, Long> {
}
```

5. 实现基本的增删改查

- 新增数据

``` java
@Test
public void testSave() throws Exception {
  TestDoc testDoc = new TestDoc();
  testDoc.setAge(10);
  testDoc.setId(1L);
  testDoc.setIntro("测试1");
  testDoc.setUserName("测试1");

  testElasticsearchRepository.save(testDoc);
}
```

- 修改数据

修改数据其实就是将修改后的数据重新保存一遍就可以了。

- 删除数据

``` java
@Test
public void testDelete() throws Exception {
	testElasticsearchRepository.deleteById(1L);
}
```

- 查找数据

``` java
@Test
public void testGetById() throws Exception {
  Optional<TestDoc> byId = testElasticsearchRepository.findById(1L);
  TestDoc testDoc = byId.get();
  System.out.println(testDoc);
}
```

6. DSL的查询

DSL查询是我们使用 es 的重点，而在 spring-data 中使用它也和使用原生api差不多，通过 QueryBuilds 工具类构建一系列的查询参数，并传给查询接口。这里可以使用 `SearchQuery` 接口，它是对 QueryBuild 的进一步封装，提供了如排序 分页等等功能。

``` java
@Test
public void testQuery() throws Exception {
    // 查找需求： 找到 名字包含 java 年龄在10 到 40 之间的数据，通过年龄进行倒序排列，同时进行分页 20 条数据一列 查询第一列
    // 创建bool查询
    BoolQueryBuilder boolQueryBuilder = QueryBuilders.boolQuery();

		// 创建 匹配查询，并添加到 must条件中去
    MatchQueryBuilder matchQueryBuilder = QueryBuilders.matchQuery("userName", "php");

		// 设置 过滤条件
    // RangeQueryBuilder age = QueryBuilders.rangeQuery("age").from("10").to("40");

    boolQueryBuilder.must(matchQueryBuilder);
    // boolQueryBuilder.filter(age);
		// 通过 NativeSearchQueryBuilder建造者构造 SearchQuery对象，并指定分页和排序
    SearchQuery query = new NativeSearchQueryBuilder()
            .withQuery(boolQueryBuilder)
            .withSort(SortBuilders.fieldSort("age").order(SortOrder.DESC))
            .withPageable(PageRequest.of(0, 10)).build();

    Page<TestDoc> search = testElasticsearchRepository.search(query);

    System.out.println("总的条数: " + search.getTotalElements());
    System.out.println("总的页数: " + search.getTotalPages());

    List<TestDoc> content = search.getContent();
    content.forEach(System.out::println);
}
```
