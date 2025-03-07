# elasticsearch 
elasticsearch，基于lucene，隐藏复杂性，提供简单易用的restful api接口、java api接口（还有其他语言的api接口）。
Elasticsearch 是分布式的**文档**存储。它能存储和检索复杂的数据结构——以**实时**的方式。 换句话说，一旦一个文档被存储在 Elasticsearch 中，它就是可以被集群中的任意节点检索到。
1. 分布式的文档存储引擎
2. 分布式的搜索引擎和分析引擎
3. 分布式，支持PB级数据


入门说明
- [全文搜索引擎 Elasticsearch 入门教程](http://www.ruanyifeng.com/blog/2017/08/elasticsearch.html)
- [elasticsearch 基础 ](https://www.cnblogs.com/yufeng218/p/12128538.html)
- [Elasticsearch: 权威指南(基于2.0)](https://www.elastic.co/guide/cn/elasticsearch/guide/current/index.html)


## Elasticsearch 相关概念

### 文档
Elasticsearch 是分布式的 文档 存储。它能存储和检索复杂的数据结构—序列化成为JSON文档—以 实时 的方式。
> 一个 **对象** 是基于特定语言的内存的数据结构。为了通过网络发送或者存储它，我们需要将它表示成某种标准的格式。 JSON 是一种以人可读的文本表示对象的方法。

在 Elasticsearch 中，术语 _文档_ 有着特定的含义。它是指最顶层或者根对象, 这个根对象被序列化成 JSON 并存储到 Elasticsearch 中，指定了唯一ID。

文档元数据：一个文档不仅仅包含它的数据 ，也包含 _元数据_ —— 有关 *文档* 的信息。 三个必须的元数据元素如下：
- `_index` ：文档在哪存放。
- `_type` ：文档表示的对象类别。一个 _`type` 命名可以是大写或者小写，但是不能以下划线或者句号开头，不应该包含逗号， 并且长度限制为256个字符. 我们使用 blog 作为类型名举例。
- `_id`：文档唯一标识。ID 是一个字符串，当它和 `_index` 以及 `_type` 组合就可以唯一确定 Elasticsearch 中的一个文档。ID的生成可以自己指定或者让es自动生成
> 自动生成的 ID 是 URL-safe、 基于 Base64 编码且长度为20个字符的 GUID 字符串。 这些 GUID 字符串由可修改的 FlakeID 模式生成，这种模式允许多个节点并行生成唯一 ID ，且互相之间的冲突概率几乎为零。

```
{
   "_index":    "website",
   "_type":     "blog",
   "_id":       "AVFgSgVHUP18jI2wRx0w",
   "_version":  1,
   "created":   true
}
```

#### 更新文档
ElasticSearch的文档写入是以不可修改的形式写入的，一条文档记录一旦被写入其本身就不能被修改了。如果想要更新现有的文档，需要**重建索引**或者**进行替换**。

重新索引方式：
```
PUT /test/_doc/12374
{
  "id" : 12374,
  "created_time" : 1640763610000,
  "userId" : "00990100002000021122901003385",
  "deleted" : false
}

response：
{
  "_index" : "test",
  "_type" : "_doc",
  "_id" : "12374",
  "_version" : 4,
  "result" : "updated",
  "_shards" : {
    "total" : 2,
    "successful" : 1,
    "failed" : 0
  },
  "_seq_no" : 1667,
  "_primary_term" : 4
}

```

替换方式：
```
POST test/_doc/12374/_update
{
  "doc":{
    "user_id": "sdfasasdfasdfdaddf"
  }
}

```

在内部，Elasticsearch 已将旧文档标记为已删除，并增加一个全新的文档。 尽管你不能再对旧版本的文档进行访问，但它并不会立即消失。当继续索引更多的数据，Elasticsearch 会在**后台清理这些已删除文档**。
无论是重新索引还是局部更新的方式，实际上 Elasticsearch 按前述完全相同方式执行以下过程：
1. 从旧文档构建 JSON
2. 更改该 JSON
3. 删除旧文档
4. 索引一个新文档



#### 乐观并发控制
利用 `_version` 号来确保 应用中相互冲突的变更不会导致数据丢失。我们通过指定想要修改文档的 `version` 号来达到这个目的。 如果该版本不是当前版本号，我们的请求将会失败。


### 分片

#### 路由过程
当索引一个文档的时候，文档会被存储到一个主分片中。确定路由的分片由以下公式确定：
`shard = hash(routing) % number_of_primary_shards`
> `routing` 是一个可变值，默认是文档的 `_id` ，也可以设置成一个自定义的值。 routing 通过 hash 函数生成一个数字，然后这个数字再除以 `number_of_primary_shards` （主分片的数量）后得到 余数 。这个分布在 0 到 `number_of_primary_shards-1` 之间的余数，就是我们所寻求的文档所在分片的位置。\
> 这就解释了为什么我们要在创建索引的时候就确定好主分片的数量 并且永远不会改变这个数量：因为如果数量变化了，那么所有之前路由的值都会无效，文档也再也找不到了。

#### 主分片与副本分片交互

![avatar](https://gitee.com/rbmon/file-storage/raw/main/learning-note/other/es/cluster-1.png)
>  假设有一个集群由三个节点组成。 它包含一个叫 blogs 的索引，有两个主分片，每个主分片有两个副本分片。相同分片的副本不会放在同一节点

**协调节点**(coordinating node)：可以发送请求到集群中的任一节点。 每个节点都有能力处理任意请求。每个节点都知道集群中任一文档位置，所以可以直接将请求转发到需要的节点上。假如将所有的请求发送到 Node 1 ，我们将其称为**协调节点**。

##### 新建，索引和删除文档
![avatar](https://gitee.com/rbmon/file-storage/raw/main/learning-note/other/es/cluster-2.png)

以下是在主副分片和任何副本分片上面 成功新建，索引和删除文档所需要的步骤顺序：
1. 客户端向 `Node 1` 发送新建、索引或者删除请求。
2. 节点使用文档的 `_id` 确定文档属于分片 0 。请求会被转发到 `Node 3`，因为分片 0 的主分片目前被分配在 `Node 3` 上。
3. `Node 3` 在主分片上面执行请求。如果成功了，它将请求并行转发到 `Node 1` 和 `Node 2` 的副本分片上。**一旦所有的副本分片都报告成功**, `Node 3` 将向协调节点报告成功，协调节点向客户端报告成功。

**相关参数设置**：
> 有一些可选的请求参数允许影响整个写过程，可能以数据安全为代价提升性能。

`consistency`：在默认设置下，即使仅仅是在试图执行一个写操作之前，主分片都会要求 必须要有 _大多数(规定数量)_ 的分片副本处于活跃可用状态，才会去执行写操作。这是为了避免在发生网络分区故障（network partition）的时候进行 _写_ 操作，进而导致数据不一致。\
> 规定数量：`int( (primary + number_of_replicas) / 2 ) + 1` \
> `number_of_replicas`指的是在索引设置中的设定副本分片数，而不是指当前处理活动状态的副本分片数。
- 设置为`one`: 只要主分片状态 ok 就允许执行 _写_ 操作
- 设置为`all`: 必须要主分片和所有副本分片的状态没问题才允许执行 _写_ 操作
- 默认值为 quorum , 即大多数的分片副本状态没问题就允许执行 _写_ 操作。

`timeout`：如果没有足够的副本分片， Elasticsearch会等待，希望更多的分片出现。默认情况下，它最多等待1分钟。
>新索引默认有 1 个副本分片，这意味着为满足 **规定数量** 应该需要两个活动的分片副本。 但是，这些默认的设置会阻止我们在单一节点上做任何事情。为了避免这个问题，要求只有当 `number_of_replicas` 大于1的时候，规定数量才会执行。

##### 查询请求交互

基于ID的查询：

![avatar](https://gitee.com/rbmon/file-storage/raw/main/learning-note/other/es/cluster-3.png)
以下是从主分片或者副本分片检索文档的步骤顺序：
1. 客户端向 `Node 1` 发送获取请求。
2. 节点使用文档的 `_id` 来确定文档属于分片 0 。分片 0 的副本分片存在于所有的三个节点上。 在这种情况下，它将请求转发到 `Node 2` 。
3. `Node 2` 将文档返回给 `Node 1` ，然后将文档返回给客户端。
> 在处理读取请求时，协调结点在每次请求的时候都会**通过轮询所有的副本分片来达到负载均衡**。\
> **分区同步导致数据可能不一致**：在文档被检索时，已经被索引的文档可能已经存在于主分片上但是还没有复制到副本分片。 在这种情况下，副本分片可能会报告文档不存在，但是主分片可能成功返回文档。



##### 局部更新文档

![avatar](https://gitee.com/rbmon/file-storage/raw/main/learning-note/other/es/update-1.png)

部分更新一个文档的步骤：
1. 客户端向 `Node 1` 发送更新请求。
2. 它将请求转发到主分片所在的 `Node 3` 。
3. `Node 3 `从主分片检索文档，修改 `_source` 字段中的 `JSON` ，并且尝试重新索引主分片的文档。 如果文档已经被另一个进程修改，它会重试步骤 3 ，超过 `retry_on_conflict` 次后放弃。
4. 如果 `Node 3` 成功地更新文档，它将新版本的文档并行转发到 `Node 1` 和 `Node 2` 上的副本分片，重新建立索引。 一旦所有副本分片都返回成功， `Node 3` 向协调节点也返回成功，协调节点向客户端返回成功。
> 当主分片把更改转发到副本分片时， 它不会转发更新请求。 相反，它转发完整文档的新版本。请记住，这些更改将会异步转发到副本分片，并且不能保证它们以发送它们相同的顺序到达。 如果Elasticsearch仅转发更改请求，则可能以错误的顺序应用更改，导致得到损坏的文档。


#### 分页查询工作流程

##### 查询阶段
![avatar](https://gitee.com/rbmon/file-storage/raw/main/learning-note/other/es/search-1.png)
> 优先队列: 一个 优先队列 仅仅是一个存有 top-n 匹配文档的有序列表。优先队列的大小取决于分页参数 from 和 size 。例如，如下搜索请求将需要足够大的优先队列来放入100条文档。
>
> ```
>   GET /_search
>   {
>       "from": 90,
>       "size": 10
>   }
> ```

查询阶段包含以下三个步骤:
1. 客户端发送一个 `search` 请求到 Node 3 ， Node 3 会创建一个大小为 `from + size` 的空优先队列。
2. Node 3 将查询请求转发到索引的每个主分片或副本分片中。每个分片在本地执行查询并添加结果到大小为 `from + size` 的本地有序优先队列中。
3. 每个分片返回各自优先队列中所有文档的 ID 和排序值给协调节点，也就是 Node 3 ，它合并这些值到自己的优先队列中来产生一个全局排序后的结果列表。
> 当一个搜索请求被发送到某个节点时，这个节点就变成了协调节点。 这个节点的任务是广播查询请求到所有相关分片并将它们的响应整合成全局排序后的结果集合，这个结果集合会返回给客户端。\
> 分片返回一个轻量级的结果列表到协调节点，它仅包含文档 ID 集合以及任何排序需要用到的值，例如 `_score` 。


##### 取回阶段
![image](https://gitee.com/rbmon/file-storage/raw/main/learning-note/other/es/search-2.png)
分布式阶段由以下步骤构成：
1. 协调节点辨别出哪些文档需要被取回并向相关的分片提交多个 GET 请求。
2. 每个分片加载并 _丰富_ 文档，如果有需要的话，接着返回文档给协调节点。
3. 一旦所有的文档都被取回了，协调节点返回结果给客户端。
> 协调节点给持有相关文档的每个分片创建一个 multi-get request ，并发送请求给同样处理查询阶段的分片副本。


##### 搜索选项
**偏好 preference**：允许用来控制由哪些分片或节点来处理搜索请求。 它接受像` _primary, _primary_first, _local, _only_node:xyz, _prefer_node:xyz, _shards:2,3` 这样的值
> `Bouncing Results`：每次用户刷新页面，搜索结果表现是不同的顺序。主要指定的排序字段在不同分片上可能不一致，如`timestamp`。可以指定`preference`为用户会话Id解决该问题。

**超时问题**：查询花费的时间是**最慢分片的处理时间**加 **结果合并的时间**。参数 `timeout` 告诉分片允许处理数据的最大时间

**路由问题**：定制参数 `routing` ，它能够在索引时提供来确保相关的文档，比如属于某个用户的文档被存储在某个分片上。
> 这个技术在设计大规模搜索系统时就会派上用场


### 搜索
一个查询中常见的字段：
```
{
	"took": 2,
	"timed_out": false,
	"_shards": {
		"total": 2,
		"successful": 2,
		"skipped": 0,
		"failed": 0
	},
	"hits": {
		"total": 1088,
		"max_score": null,
		"hits": [{
			"_index": "test",
			"_type": "_doc",
			"_id": "12374",
			"_score": null,
			"_source": {
			    .....
			}
		}]
	}
}
```
- `his`: 在 hits 数组中每个结果包含文档的 `_index` 、 `_type` 、 `_id` ，加上 `_source` 字段。这意味着我们可以直接从返回的搜索结果中使用整个文档。
- `took` 值告诉我们执行整个搜索请求耗费了多少毫秒。
- `timed_out` 值告诉我们查询是否超时。默认情况下，搜索请求不会超时。该值可以手工指定。
- `_shards` 部分告诉我们在查询中参与分片的总数，以及这些分片成功了多少个失败了多少个。
> 分片失败是可能发生的。如果我们遭遇到一种灾难级别的故障，在这个故障中丢失了相同分片的原始数据和副本，那么对这个分片将没有可用副本来对搜索请求作出响应。假若这样，Elasticsearch 将报告这个分片是失败的，但是会继续返回剩余分片的结果。


**多索引多类型搜索**：
- `/gb,us/_search`：在 gb 和 us 索引中搜索所有的文档
- `/g*,u*/_search`：在任何以 g 或者 u 开头的索引中搜索所有的类型
- `/gb,us/user,tweet/_search`：在 gb 和 us 索引中搜索 user 和 tweet 类型

### 倒排索引
倒排索引：是一种可以根据属性的值来查找记录的索引。这种索引表中的每一项都包括一个属性值和具有该属性值的各条记录的地址。由于不是由记录来确定属性值，而是由属性值来确定记录的位置，故成为倒排索引。
> 经常被用来存储在全文搜索下某个单词在一个文档或者一组文档中的存储位置的映射。它是文档检索系统中最常用的数据结构。


#### 倒排索引创建
例如，假设我们有两个文档，每个文档的 content 域包含如下内容：
- position 1: The quick brown fox jumped over the lazy dog
- position 2: Quick brown foxes leap over lazy dogs in summer

**倒排索引创建过程**：
1. 首先把所有的原始数据进行编号，形成文档列表 DocId
2. 把文档数据进行分词 Term，得到很多的词条，以词条为索引。保存包含这些词条的文档的编号信息。


#### 搜索过程
当用户输入任意的词条时，首先对用户输入的数据进行分词，得到用户要搜索的所有词条，然后拿着这些词条去倒排索引列表中进行匹配。找到这些词条就能找到包含这些词条的所有文档的编号。

|**Term**    |  Doc_1  | Doc_2| Posting list|
| --- | --- | ---|  --- |
|Quick   |       |  X| 2|
|The     |   X   | | 1|
|brown   |   X   |  X| 1,2|
|dog     |   X   | | 1|
|dogs    |       |  X| 2|
|fox     |   X   | | 1|
|foxes   |       |  X| 2|
|in      |       |  X| 2|
|jumped  |   X   | | 1|
|lazy    |   X   |  X| 1,2|
|leap    |       |  X| 2|
|over    |   X   |  X| 1,2|
|quick   |   X   | | 1|
|summer  |       |  X| 2|
|the     |   X   |  | 1|

搜索 quick brown 

Term    |  Doc_1 |  Doc_2 |
 ---| ---| ---|
brown   |   X   |  X
quick   |   X   |
Total   |   2   |  1

两个文档都匹配，但是第一个文档比第二个匹配度更高。优先排在前面。

#### 倒排索引优化
倒排索引的在输入或者构建的分词中，经常有以下问题
1. 文本词条化：如带上撇号的格式——“Teacher’s office”，连字符格式——“English-speaking”,也需要进行对应的处理，把单词提取出来。
2. 停用词过滤：如英文：the, is, and，中文：的、是、个等等
3. 词条归一化：如英文：color、colour。
4. 词干提取、词形还原：如英文将“doing”、“done”、“did”转化成原型“do”


#### ES中分词

ES在处理输入的内容是通过分词器处理的。分析器处理讲过三个步骤：
1. 字符过滤器: 首先，字符串按顺序通过每个 字符过滤器 。他们的任务是在分词前整理字符串。一个字符过滤器可以用来去掉HTML，或者将 & 转化成 and。
2. 分词器:其次，字符串被 分词器 分为单个的词条。一个简单的分词器遇到空格和标点的时候，可能会将文本拆分成词条。
3. Token过滤器： 最后，词条按顺序通过每个 token 过滤器 。这个过程可能会改变词条（例如，小写化 Quick ），删除词条（例如， 像 a， and， the 等无用词），或者增加词条（例如，像 jump 和 leap 这种同义词）。


Elasticsearch的分词器的一般工作流程：
1. 切分关键词
2. 去除停用词
3. 对于英文单词，把所有字母转为小写（搜索时不区分大小写）

参考资料：
- [搜索引擎之倒排索引解读](https://zhuanlan.zhihu.com/p/28320841)

### 排序
为了按照**相关性**来排序，需要将相关性表示为一个数值。在 Elasticsearch 中，**相关性得分**由一个浮点数进行表示，并在搜索结果中通过 `_score` 参数返回， 默认排序是 `_score` 降序。


执行一条关于**日期排序**的查询，结果如下：
```
GET /_search
{
    "query" : {
        "bool" : {
            "filter" : {
                "term" : {
                    "user_id" : 1
                }
            }
        }
    }
}

"hits" : {
    "total" :           6,
    "max_score" :       null, 
    "hits" : [ {
        "_index" :      "us",
        "_type" :       "tweet",
        "_id" :         "14",
        "_score" :      null, 
        "_source" :     {
             "date":    "2014-09-24",
             ...
        },
        "sort" :        [ 1411516800000 ] 
    },
    ...
}
```
- `_score`: 不被计算, 因为它并没有用于排序。
- `sort`: 在每个结果中有一个新的名为 sort 的元素，它包含了我们用于排序的值。


#### 相关性
每个文档都有相关性评分，用一个正浮点数字段 `_score` 来表示 。 `_score` 的评分越高，相关性越高。

查询语句会为每个文档生成一个 `_score` 字段。\
评分的计算方式取决于查询类型，不同的查询语句用于不同的目的： 
- fuzzy 查询会计算与关键词的拼写相似程度
- terms 查询会计算 找到的内容与关键词组成部分匹配的百分比
> 通常我们说的 relevance 是我们用来计算全文本字段的值相对于全文本检索词相似程度的算法。

Elasticsearch 的相似度算法被定义为检索词频率/反向文档频率， TF/IDF ，包括以下内容：
- 检索词频率： 检索词在该字段出现的频率。出现频率越高，相关性也越高。 字段中出现过 5 次要比只出现过 1 次的相关性高。
- 反向文档频率：每个检索词在索引中出现的频率。频率越高，相关性越低。检索词出现在多数文档中会比出现在少数文档中的权重更低。
- 字段长度准则：字段的长度是多少。长度越长，相关性越低。 检索词出现在一个短的 title 要比同样的词出现在一个长的 content 字段权重更大。

**评分的标准可以通过`_explain`进行进一步了解**
```
GET /_search?explain 
{
   "query"   : { "match" : { "tweet" : "honeymoon" }}
}

结果：
"_explanation": { 
   "description": "weight(tweet:honeymoon in 0)
                  [PerFieldSimilarity], result of:",
   "value":       0.076713204,
   "details": [
      {
         "description": "fieldWeight in 0, product of:",
         "value":       0.076713204,
         "details": [
            {  
                // 检索词频率。`honeymoon` 在这个文档的 `tweet` 字段中的出现次数。
               "description": "tf(freq=1.0), with freq of:", 
               "value":       1,
               "details": [
                  {
                     "description": "termFreq=1.0",
                     "value":       1
                  }
               ]
            },
            { 
              // 反向文档频率。检索词 `honeymoon` 在索引上所有文档的 `tweet` 字段中出现的次数。
               "description": "idf(docFreq=1, maxDocs=1)",   
               "value":       0.30685282
            },
            { 
                // 字段长度准则。在这个文档中， `tweet` 字段内容的长度 -- 内容越长，值越小。
               "description": "fieldNorm(doc=0)",    
               "value":        0.25,
            }
         ]
      }
   ]
}
```


#### Doc Values 

在 Elasticsearch 中，`Doc Values` 就是一种列式存储结构，默认情况下每个字段的 `Doc Values` 都是激活的，`Doc Values` 是在索引时创建的，当字段索引时，Elasticsearch 为了能够快速检索，会把字段的值加入倒排索引中，同时它也会存储该字段的 `Doc Values`。

Elasticsearch 中的 Doc Values 常被应用到以下场景：
- 对一个字段进行排序
- 对一个字段进行聚合
- 某些过滤，比如地理位置过滤
- 某些与字段相关的脚本计算




### 深度分页
理解为什么深度分页是有问题的？\
> 我们可以假设在一个有 5 个主分片的索引中搜索。 当我们请求结果的第一页（结果从 1 到 10 ），每一个分片产生前 10 的结果，并且返回给 协调节点 ，协调节点对 50 个结果排序得到全部结果的前 10 个。\
> 现在假设我们请求第 1000 页—结果从 10001 到 10010 。所有都以相同的方式工作除了每个分片不得不产生前10010个结果以外。 然后协调节点对全部 50050 个结果排序最后丢弃掉这些结果中的 50040 个结果。\
> 可以看到，在分布式系统中，对结果排序的成本随分页的深度成指数上升。这就是 web 搜索引擎对任何查询都不要返回超过 1000 个结果的原因。


> 常规的查询中： 先查后取的过程支持用 `from、size` 参数分页，但是这是 有限制的 。 要记住需要传递信息给协调节点的每个分片必须先创建一个 `from + size` 长度的队列，协调节点需要根据 `number_of_shards * (from + size)` 排序文档，来找到被包含在 size 里的文档。\
> 是使用足够大的 from 值，排序过程可能会变得非常沉重，使用大量的CPU、内存和带宽。因为这个原因，我们强烈建议你不要使用深分页。\
> 实际上，“深分页” 很少符合人的行为。当2到3页过去以后，人会停止翻页，并且改变搜索标准。会不知疲倦地一页一页的获取网页直到你的服务崩溃的罪魁祸首一般是机器人或者web spider。

Es设置了 `max_result_window`(最大结果窗口)的参数，默认值是10000，它不仅限制了用户在一次查询中最多数据条数是1w条，并且限制了start+size 必须小于1w。

#### 游标查询Scroll
`scroll` 查询 可以用来对 Elasticsearch 有效地执行大批量的文档查询，而又不用付出深度分页那种代价。游标查询允许先做查询初始化，然后再批量地拉取结果。这有点儿像传统数据库中的`cursor`。
> 游标查询会取某个时间点的快照数据，并保存搜索的`search context`。查询初始化之后索引上的任何变化会被它忽略。 它通过保存旧的数据文件来实现这个特性，结果就像保留初始化时的索引 _视图_ 一样。\
> 在游标查询中，每个shard 它是通过lastEmittedDoc来确定游标位置的。\
> **Scroll不适合用于实时的搜索，因为scroll的搜索内容是基于快照的。**

**相关参数：**
- `search.max_open_scroll_context` ：用于防止开启过多的scroll request，默认500
- `scroll_id`: `_scroll_id`在每次查询的时候可能会发生变化，所以在下次查询的时候都要带上上次查询的`_socrll_id`。
- `scroll`：存活时间，快照的存活时间，用于告诉ES，快照及`search context`要保存多长时间。每次请求后，就会根据参数**刷新该时间重新计时**。
    > 何时释放快照及`search context`？ 超时、调用`clear scroll`


请求流程：
![image](https://gitee.com/rbmon/file-storage/raw/main/learning-note/other/es/scroll-1.png)

1. search阶段：第一次带查询参数的请求
```
// scroll=1m 保持游标查询窗口一分钟。
GET /old_index/_search?scroll=1m 
{
    "query": { "match_all": {}},
    "sort" : ["_doc"],  // 关键字 _doc 是最有效的排序顺序。
    "size":  1000
}

result:
{
	"_scroll_id": "DnF1ZXJ5VGhlbkZldGNoAgAAAAAAAaE_Fmc0Skc2OW9DUkZlQy1XX2w0eUZMbUEAAAAAAAGhPhZnNEpHNjlvQ1JGZUMtV19sNHlGTG1B",
	"took": 13,
	"timed_out": false,
	"_shards": {
		"total": 2,
		"successful": 2,
		"skipped": 0,
		"failed": 0
	},
	"hits": {
		"total": 1091,
		"max_score": null,
		"hits": [
		    { .... }
		]
	}
}
```

2. scroll阶段：第二次带scrollId的请求
> 第二阶段Scroll请求则大大简化，Search中的许多流程都不要再次进行，仅需要执行query、fetch、response三个阶段。而完整的search请求包含rewrite、can_match、dfs、query、fetch、dfs_query、expand、response等复杂的流程
```
GET /_search/scroll
{
    "scroll": "1m", 
    "scroll_id" : "DnF1ZXJ5VGhlbkZldGNoAgAAAAAAAaE_Fmc0Skc2OW9DUkZlQy1XX2w0eUZMbUEAAAAAAAGhPhZnNEpHNjlvQ1JGZUMtV19sNHlGTG1B"

}


{
  "_scroll_id" : "DnF1ZXJ5VGhlbkZldGNoAgAAAAAAAaE_Fmc0Skc2OW9DUkZlQy1XX2w0eUZMbUEAAAAAAAGhPhZnNEpHNjlvQ1JGZUMtV19sNHlGTG1B",
  "took" : 9,
  "timed_out" : false,
  "_shards" : {
    "total" : 2,
    "successful" : 2,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : 1091,
    "max_score" : null,
    "hits" : [
      {...}
      ]
	}
}
```

#### search after
Search接口另一种翻页方式是SearchAfter，时间复杂度O(n)，空间复杂度O(1)。SearchAfter是一种动态指针的技术，每次查询都会携带上一次的排序值，这样下次取结果只需要从上次的位点继续扫数据，前提条件也是该字段是数值类型且设置了docValue。
> 举个例子，假设"val_1"是数值类型的字段，然后使用Search接口查询时候添加Sort("val_1")，那么response中可以拿到最后一条数据的"val_1"的值，也就是response中sort字段的值，然后下次查询将该值放在query中的searchAfter参数中，下次查询就可以在上一次结果之后继续查询，如此反复，最后可以翻页很深，内存消耗相比size+from的方式降低了数倍。该方式效果类似于我们直接在bool查询中主动加一个rangeFilter，可以达到类似的效果。\
> 表面看这种方案能将查询速度降到O(1)的复杂度，实际上其内部还是会扫sort字段的docValue，翻页越深，则扫docValve越多，因此复杂度和翻页深度成正比，越往后查询越慢，但是相比size+from的方式，至少可以完成深度翻页的任务，不至于OOM，速度勉强可以接受。SearchAfter的翻页方式在性能上有了质的提升，但是其限制了用户只能一页一页往后翻，无法跳页，因此很多产品在功能设计时候是不允许跳页的，只能一页一页往后翻，也是有一定的技术原因的。



使用search after 要求每次查询都使用相同的query和sort参数。
> 如果在查询的过程中ES触发了refresh(refresh实现的是文档从内存移到文件系统缓存的过程)，查询的顺序改变导致分页的结果不准确，可以调用PIT(Point in time)API来避免这个场景。
> PIT会创建一个轻量级的视图，保证了查询的时候不会因为refresh导致分页数据不一致的情况。


```
POST twitter/_search 
{ 
    "size": 10, 
    "query": { 
        "match" : { 
            "title" : "es" 
        } 
    }, 
    "sort": [ 
        {"date": "asc"}, 
        {"_id": "desc"} 
    ] 
} 

response:
{
	"took": 29,
	"timed_out": false,
	"_shards": {
		"total": 1,
		"successful": 1,
		"skipped": 0,
		"failed": 0
	},
	"hits": {
		"total": {
			"value": 5,
			"relation": "eq"
		},
		"max_score": null,
		"hits": [{
			"_source": {},
			"sort": [
				124648691,
				"624812"
			]
		}]
	}
}
```

第二次查询：
```
GET twitter/_search 
{ 
    "size": 10, 
    "query": { 
        "match" : { 
            "title" : "es" 
        } 
    }, 
    "search_after": [124648691, "624812"], 
    "sort": [ 
        {"date": "asc"}, 
        {"_id": "desc"} 
    ] 
} 
```

#### 总结
- `from + size`:灵活性好，实现简单 深度分页问题 数据量比较小，能容忍深度分页问题
- `scroll`: 解决了深度分页问题 无法反应数据的实时性(快照版本)维护成本高，需要维护一个 scroll_id 海量数据的导出需要查询海量结果集的数据
- `search_after` : 性能最好不存在深度分页问题能够反映数据的实时变更 实现复杂，需要有一个全局唯一的字段连续分页的实现会比较复杂，因为每一次查询都需要上次查询的结果，它不适用于大幅度跳页查询 海量数据的分页
> 在7.*版本中，ES官方不再推荐使用Scroll方法来进行深分页，而是推荐使用带PIT的search_after来进行查询;


**search after 对比 scroll：**
优点：
1. 无状态查询，可以防止在查询过程中，数据的变更无法及时反映到查询中。
2. 不需要维护scroll_id，不需要维护快照，因此可以避免消耗大量的资源。

缺点：
1. 由于无状态查询，因此在查询期间的变更可能会导致跨页面的不一值，需要靠PIT保障。
2. 排序顺序可能会在执行期间发生变化，具体取决于索引的更新和删除。
3. 至少需要制定一个唯一的不重复字段来排序。
4. 它不适用于大幅度跳页查询，或者全量导出，对第N页的跳转查询相当于对es不断重复的执行N次search after，而全量导出则是在短时间内执行大量的重复查询。

SEARCH_AFTER不是自由跳转到任意页面的解决方案，而是并行滚动多个查询的解决方案。

#### 参考资料
- [Elasticsearch 5.x 源码分析（3）from size, scroll 和 search after](https://www.jianshu.com/p/91d03b16af77)
- [Elasticsearch之SearchScroll原理剖析和优化](https://developer.aliyun.com/article/771575)
- [ElasticSearch深度分页解决方案](https://developer.51cto.com/article/684507.html)

## docker 安装

### elasticsearch 安装
```

[root@VM-0-16-centos ~]# docker pull elasticsearch:7.13.1
[root@VM-0-16-centos ~]# docker images

[root@VM-0-16-centos ~]# docker pull kibana:7.13.1
[root@VM-0-16-centos ~]# docker images

// 创建自定义的网络(用于连接到连接到同一网络的其他服务(例如Kibana))
[root@VM-0-16-centos ~]# docker network create elknetwork
ea5897232c9daad0c00b4b47c240ff513177a42ae0b48b770068691a99949798


[root@VM-0-16-centos ~]# docker run -it --name elasticsearch -p 9200:9200 -p 9300:9300 -e "discovery.type=single-node" -e "ES_JAVA_OPTS=-Xms512m -Xmx512m" -d --net elknetwork elasticsearch:7.13.1
604ee6bb8b84fc5b3f6bf590fd57f04505fcdda8539d17493b87d6d4e8272b63

// 检查es是否启动
[root@VM-0-16-centos ~]# curl 127.0.0.1:9200
{
  "name" : "604ee6bb8b84",
  "cluster_name" : "docker-cluster",
  "cluster_uuid" : "tGGUtTAjSYiBu6hMSa_GzQ",
  "version" : {
    "number" : "7.13.1",
    "build_flavor" : "default",
    "build_type" : "docker",
    "build_hash" : "9a7758028e4ea59bcab41c12004603c5a7dd84a9",
    "build_date" : "2021-05-28T17:40:59.346932922Z",
    "build_snapshot" : false,
    "lucene_version" : "8.8.2",
    "minimum_wire_compatibility_version" : "6.8.0",
    "minimum_index_compatibility_version" : "6.0.0-beta1"
  },
  "tagline" : "You Know, for Search"
}

[root@VM-0-16-centos ~]# docker run -d --name kibana -p 5601:5601 --net elknetwork kibana:7.13.1
2dcdd5bc87bd2f6fcb2b020bd866174b760d7e43e1788c2ca7205848d7a46074

```

![avatar](https://gitee.com/rbmon/file-storage/raw/main/learning-note/other/es/ES.jpg)
  
参考资料： [Docker安装部署ELK教程](https://www.cnblogs.com/fbtop/p/11005469.html)
### ik分词器安装

#### 在线安装
```
// 进入容器
docker exec -it elasticsearch /bin/bash

// 在线下载并安装
./bin/elasticsearch-plugin install https://github.com/medcl/elasticsearch-analysis-ik/releases/download/v6.5.4/elasticsearch-analysis-ik-7.13.1.zip
```

#### 离线安装

```
wget https://github.com/medcl/elasticsearch-analysis-ik/releases/download/v7.13.1/elasticsearch-analysis-ik-7.13.1.zip

docker exec -it elasticsearch /bin/bash

// 创建目录
mkdir /usr/share/elasticsearch/plugins/ik

// 将文件压缩包移动到ik中
mv /usr/share/elasticsearch/plugins/elasticsearch-analysis-ik-7.13.1.zip /usr/share/elasticsearch/plugins/ik

// 进入目录
cd /usr/share/elasticsearch/plugins/ik

// 解压
unzip elasticsearch-analysis-ik-7.13.1.zip

// 删除压缩包
rm -rf elasticsearch-analysis-ik-7.13.1.zip

exit
// 重启进行
docker restart elasticsearch
```

#### 分词器测试
```
POST _analyze 
{
  "analyzer": "ik_smart",
  "text": "的说法是的发送到"
}


result:
{
  "tokens" : [
    {
      "token" : "的",
      "start_offset" : 0,
      "end_offset" : 1,
      "type" : "CN_CHAR",
      "position" : 0
    },
    {
      "token" : "说法",
      "start_offset" : 1,
      "end_offset" : 3,
      "type" : "CN_WORD",
      "position" : 1
    },
    {
      "token" : "是的",
      "start_offset" : 3,
      "end_offset" : 5,
      "type" : "CN_WORD",
      "position" : 2
    },
    {
      "token" : "发送到",
      "start_offset" : 5,
      "end_offset" : 8,
      "type" : "CN_WORD",
      "position" : 3
    }
  ]
}

```
参考资料：[docker 安装ElasticSearch的中文分词器IK](https://blog.csdn.net/weixin_34015566/article/details/93554240)
## linux http基本操作命令
### 基本操作
```
[root@VM-0-10-centos ~]# curl -X GET 'http://localhost:9200/_cat/indices?v'

[root@VM-0-10-centos ~]# curl -X PUT 'localhost:9200/accounts'

[root@VM-0-10-centos ~]# curl -X DELETE 'localhost:9200/account'

```

### 索引创建与新增元素
```
[root@VM-0-10-centos ~]# curl -H 'content-Type:application/json'  -X PUT 'localhost:9200/test' -d '

  {
    "settings":{
      "number_of_shards":3,
      "number_of_replicas":2
    },
    "mappings":{
      "properties":{
        "id":{"type":"long"},
        "name":{"type":"text"},
        "text":{"type":"text"}
      }
    }
   
  }';

[root@VM-0-10-centos ~]# curl -H 'content-Type:application/json' -X POST 'localhost:9200/test/_doc/1' -d '
{
  "name": "zhangsan",
  "desc": "databaseManager"
}' ;
{"_index":"test","_type":"_doc","_id":"1","_version":1,"result":"created","_shards":{"total":3,"successful":1,"failed":0},"_seq_no":0,"_primary_term":1}


[root@VM-0-10-centos ~]# curl -X GET 'http://localhost:9200/test/_doc/1'
{"_index":"test","_type":"_doc","_id":"1","_version":1,"_seq_no":0,"_primary_term":1,"found":true,"_source":
{
  "name": "zhangsan",
  "desc": "databaseManager"
}}

// 不带主键新增，默认es添加主键
[root@VM-0-10-centos ~]# cH 'content-Type:application/json' -X POST 'localhost:9200/test/_doc' -d '
{
  "name": "asibi",
  "desc": "soul"
}' ;
{"_index":"test","_type":"_doc","_id":"UPmoeXYBI1Dq1Op9wJWu","_version":1,"result":"created","_shards":{"total":3,"successful":1,"failed":0},"_seq_no":0,"_primary_term":1}

```


### 查询
```
[root@VM-0-10-centos ~]# curl -X GET 'http://localhost:9200/test/_search'
{"took":391,"timed_out":false,"_shards":{"total":3,"successful":3,"skipped":0,"failed":0},"hits":{"total":{"value":3,"relation":"eq"},"max_score":1.0,"hits":[{"_index":"test","_type":"_doc","_id":"UPmoeXYBI1Dq1Op9wJWu","_score":1.0,"_source":
{
  "name": "asibi",
  "desc": "soul"
}},{"_index":"test","_type":"_doc","_id":"2","_score":1.0,"_source":
{
  "name": "lisan",
  "desc": "zooManager"
}},{"_index":"test","_type":"_doc","_id":"1","_score":1.0,"_source":
{
  "name": "zhangsan",
  "desc": "databaseManager"
}}]}}


[root@VM-0-10-centos ~]# curl -H 'content-Type:application/json' -X GET 'http://localhost:9200/test/_search' -d  '
{
"query":{
    "match": {"name":"lisan"}
  } 
}';       
{"took":21,"timed_out":false,"_shards":{"total":3,"successful":3,"skipped":0,"failed":0},"hits":{"total":{"value":1,"relation":"eq"},"max_score":0.2876821,"hits":[{"_index":"test","_type":"_doc","_id":"2","_score":0.2876821,"_source":
{
  "name": "lisan",
  "desc": "zooManager"
}}]}}


[root@VM-0-10-centos ~]# curl -H 'content-Type:application/json' -X GET 'http://localhost:9200/test/_search' -d  '
{
"query":{
    "match": {"name":"lisan zhangsan"}
  }
}';
{"took":6,"timed_out":false,"_shards":{"total":3,"successful":3,"skipped":0,"failed":0},"hits":{"total":{"value":2,"relation":"eq"},"max_score":0.2876821,"hits":[{"_index":"test","_type":"_doc","_id":"2","_score":0.2876821,"_source":
{
  "name": "lisan",
  "desc": "zooManager"
}},{"_index":"test","_type":"_doc","_id":"1","_score":0.2876821,"_source":
{
  "name": "zhangsan",
  "desc": "databaseManager"
}}]}}
```

- [官网查询说明](https://www.elastic.co/guide/cn/elasticsearch/guide/current/query-dsl-intro.html)

## kibana 命令行操作

### 创建索引
```
PUT sw_test.trade_contract_v1
{
  "settings": {
    "index": {
      "number_of_shards": 2,
      "number_of_replicas": 1
    },
    "analysis": {
      "analyzer": {
        "default": {
          "type": "ik_max_word"
        },
        "default_search": {
          "type": "ik_smart"
        }
      }
    }
  },
  "aliases": {
    "sw_test.trade_contract": {}
  },
  "mappings": {
    "properties": {
      "created_time": {
        "type": "date"
      },
      "modified_time": {
        "type": "date"
      },
      "status": {
        "type": "keyword"
      },
      "contract_id": {
        "type": "keyword"
      },
      "invoice_no": {
        "type": "keyword"
      },
      "export_country": {
        "type": "keyword"
      },
      "exp_currency": {
        "type": "keyword"
      },
      "amount": {
        "type": "double"
      },
      "deleted": {
        "type": "boolean"
      },
      "extra": {
        "type": "text",
        "fields": {
          "keyword": {
            "type": "keyword"
          }
        }
      },
      "product_name": {
        "type": "text",
        "fields": {
          "keyword": {
            "type": "keyword"
          }
        }
      },
      "product_quantity": {
        "type": "long"
      },
      "product_quantity_unit": {
        "type": "keyword"
      },
      "note": {
        "type": "text",
        "fields": {
          "keyword": {
            "type": "keyword"
          }
        }
      }
    }
  }
}
```

### 中文分词

[参见](https://zhuanlan.zhihu.com/p/52543633)

analysis-ik分两种模式：ik_max_word和ik_smart模式

#### ik_max_word
会将文本做最细粒度的拆分，比如会将“中华人民共和国人民大会堂”拆分为“中华人民共和国、中华人民、中华、华人、人民共和国、人民、共和国、大会堂、大会、会堂等词语。

#### ik_smart
会做最粗粒度的拆分，比如会将“中华人民共和国人民大会堂”拆分为中华人民共和国、人民大会堂。

#### 最佳实践

两种分词器使用的最佳实践是：索引时用ik_max_word，在搜索时用ik_smart。
即：索引时最大化的将文章内容分词，搜索时更精确的搜索到想要的结果。

```json
{
  "mappings": {
    "_doc": {
      "properties": {
        "firm_name": {
          "type": "text",
          "analyzer": "ik_max_word", 
          "search_analyzer": "ik_smart" 
        }
      }
    }
  }
}
```

### 手动插入数据
```
POST sw_test.trade_contract/_doc
{
    "amount":3344.00,
    "contract_id":"01990100018000018070300001149",
    "created_time":"2021-01-01",
    "custom_id":"123123",
    "deleted": false,
    "exp_currency":"USD",
    "export_country":"澳大利亚",
    "export_country_code":"CN",
    "extra":"小小",
    "invoice_no":"qa_order_20210526194123288",
    "modified_time":"2021-01-01",
    "note":"1212",
    "product_name":"apple",
    "product_quantity":1200,
    "product_quantity_unit":"kg",
    "status":"Closed",
    "test_add": "时代峰峻卡上的福建省地方"
}
```

### 查询
```
POST sw_test.trade_contract/_search
{
   "query": { 
    "bool": { 
      "must": [
        { "match": { "exp_currency":   "USD" }},
        { "match": { "product_name": "apple" }}
      ],
      "filter": [ 
        { "term":  { "export_country": "澳大利亚" }},
        { "range": { "created_time": { "gte": "2015-01-01",
                "lte" : "2022-01-01" }}}
      ],
      "should": [{
					"match": {
						"test_add": "卡上的"
					}
				}
			]
    }
  }
}
```



#### 字段类型
[Field datatypes](https://www.elastic.co/guide/en/elasticsearch/reference/7.13/mapping-types.html)

#### filter and query
[Bool Query](https://www.elastic.co/guide/en/elasticsearch/reference/6.8/query-dsl-bool-query.html)

[Query and Filter context](https://www.elastic.co/guide/en/elasticsearch/reference/7.13/query-filter-context.html#query-filter-context)

[Different with Filter and Must Not](https://stackoverflow.com/questions/47226479/difference-between-must-not-and-filter-in-elasticsearch)

Basically, filter = must but without scoring.

**filter:**

    It is written in Filter context.
    It does not affect the score of the result.
    The matched query results will appear in the result.
    Exact match based, not partial match.

**must_not:**

    It is written again on the same filter context.
    Which means it will not affect the score of the result.
    The documents matched with this condition will NOT appear in the result.
    Exact match based.

| bool     |  similar |     context      | 影响评分 | 出现在结果集 | 精确匹配 |
| :------: | :------: |      :----:      | :---: | :-----: | :-----: |
| must     | AND      | query  context   | Y | Y | Y |
| filter   | AND      | filter context   | N | Y | Y |
| should   | OR       | query context    | Y | Y | N |
| must_not | AND NOT  | filter context   | N | N | Y  |



### 索引新增字段

```
POST sw_test.trade_contract_v1/_mapping
{
  "properties": {
     "test_add":{
        "type":"text"
     }
  }
}
```
### 更改字段类型为 multi_field
创建 mapping 时，可以为keyword指定ignore_above ，用来限定字符长度。\
超过 ignore_above 的字符会被存储，但不会被全文索引。

```
PUT /sw_test.trade_contract_v1/_mapping/
{
  "properties": {
     "test_add":{
        "type":"text",
        "fields": {
          "keyword": {
            "type": "keyword",
            "ignore_above": 256
          }
        }
     }
  }
}
```
### 其他
- object类型自动映射，无需手动新增
- int、long、date等类型自动映射，可以不手动新增
- string类型会自动映射成multi_field，并使用默认分词器，建议手动修改ES mapping


### 重建索引、修改Mapping的方式

[Index Aliases](https://www.elastic.co/guide/en/elasticsearch/reference/6.8/indices-aliases.html)
[Elasticsearch如何修改Mapping结构并实现业务零停机](https://juejin.im/post/5e2d32c95188254d9032a7dd)


#### 步骤1: 建立新索引
```
PUT sw_test.trade_contract_v2
```

#### 步骤2: 复制数据
```
POST _reindex
{
    "source": {
        "index": "sw_test.trade_contract_v1"
    },
    "dest": {
        "index": "sw_test.trade_contract_v2"
    }
}
```
#### 步骤3: 修改别名关联
```
POST /_aliases
{
    "actions": [
        { "remove": { "index": " sw_test.trade_contract_v1", "alias": " sw_test.trade_contract" }},
        { "add":    { "index": " sw_test.trade_contract_v2", "alias": " sw_test.trade_contract" }}
    ]
}

```
#### 步骤4: 删除旧索引
```
DELETE  sw_test.trade_contract_v1
```


## shard & replica

```
PUT sw_test.trade_contract_v1
{
  "settings": {
    "index": {
      "number_of_shards": 2,
      "number_of_replicas": 1
    },
    ....
}
```
参考文章：
1. [shards-and-replicas-in-elasticsearch](https://stackoverflow.com/questions/15694724/shards-and-replicas-in-elasticsearch)
2. [es-glossary](https://www.elastic.co/guide/en/elasticsearch/reference/current/glossary.html)


### primary shard 主分片

When you create an index (an index is automatically created when you index the first document as well) you can define how many shards it will be composed of.\
If you don't specify a number it will have the default number of shards: 5 primaries. What does it mean?

It means that elasticsearch will create 5 primary shards that will contain your data:

```
 ____    ____    ____    ____    ____
| 1  |  | 2  |  | 3  |  | 4  |  | 5  |
|____|  |____|  |____|  |____|  |____|
```

Every time you index a document, elasticsearch will decide which primary shard is supposed to hold that document and will index it there.\
`Primary shards are not a copy of the data, they are the data!` Having multiple shards does help taking advantage of parallel processing on a single machine,\
but the whole point is that if we start another elasticsearch instance on the same cluster, `the shards will be distributed in an even way over the cluster`.

Node 1 will then hold for example only three shards:

```
 ____    ____    ____ 
| 1  |  | 2  |  | 3  |
|____|  |____|  |____|
```

Since the remaining two shards have been moved to the newly started node:

```
 ____    ____
| 4  |  | 5  |
|____|  |____|
```

Why does this happen? Because elasticsearch is a distributed search engine and this way you can make use of multiple
nodes/machines to manage big amounts of data.

Every elasticsearch index is composed of at least one primary shard since that's where the data is stored.
Every shard comes at a cost, though, therefore if you have a single node and no foreseeable growth,
just stick with a single primary shard.


### replica shard 副本分片

Another type of shard is a replica. The default is 1, meaning that every `primary shard will be copied to another shard that will contain the same data`.\
Replicas are used to increase search performance and for fail-over.\
`A replica shard is never going to be allocated on the same node where the related primary is`\
(it would pretty much be like putting a backup on the same disk as the original data).

Back to our example, with 1 replica we'll have the whole index on each node,
since 2 replica shards will be allocated on the first node, and they will contain exactly the same data as the primary shards on the second node:

`Node1`
```
 ____    ____    ____    ____    ____
| 1  |  | 2  |  | 3  |  | 4R |  | 5R |
|____|  |____|  |____|  |____|  |____|
```

Same for the second node, which will contain a copy of the primary shards on the first node:


`Node2`
```
 ____    ____    ____    ____    ____
| 1R |  | 2R |  | 3R |  | 4  |  | 5  |
|____|  |____|  |____|  |____|  |____|
```

With a setup like this, if a node `goes down`, you still have the whole index.
The replica shards will automatically become primaries, and the cluster will work properly despite the node failure, as follows:

```
 ____    ____    ____    ____    ____
| 1  |  | 2  |  | 3  |  | 4  |  | 5  |
|____|  |____|  |____|  |____|  |____|
```



## spring 集成
- [spring data与elasticsearch版本对应](https://docs.spring.io/spring-data/elasticsearch/docs/4.1.9/reference/html/#preface.requirements)
- [spring data 官网文档](https://docs.spring.io/spring-data/elasticsearch/docs/4.1.9/reference/html/#elasticsearch.clients)

## 面试题

### ES 的分布式架构原理