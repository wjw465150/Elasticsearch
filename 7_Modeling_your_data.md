# 数据建模



Elasticsearch 是如此与众不同，特别是如果你来自 SQL 的世界。 Elasticsearch 有非常多的优点：高性能、可扩展、近实时搜索，并支持大数据量的数据分析。一切都很容易！ 只需下载并开始使用它。

但它不是魔法。为了充分利用 Elasticsearch，你需要了解它的工作机制，以及如何让它如你所需的进行工作。

和专用的关系型数据存储有所不同，Elasticsearch 并没有对处理实体之间的关系给出直接的方法。 一个关系数据库的黄金法则是 --规范化你的数据（范式）-- 但这不适用于 Elasticsearch。 在 [*关联关系处理*](https://www.elastic.co/guide/cn/elasticsearch/guide/current/relations.html) 、 [*嵌套对象*](https://www.elastic.co/guide/cn/elasticsearch/guide/current/nested-objects.html) 和 [*父-子关系文档*](https://www.elastic.co/guide/cn/elasticsearch/guide/current/parent-child.html) 我们讨论了这些提供的方法的优点和缺点。

然后在 [*扩容设计*](https://www.elastic.co/guide/cn/elasticsearch/guide/current/scale.html) 我们谈论 Elasticsearch 提供的快速、灵活的扩容能力。 当然扩容并没有一个放之四海而皆准的方案。你需要考虑这些通过系统产生的数据流的具体特点， 据此设计你的模型。例如日志事件或者社交网络流这些时间序列数据类型，和静态文档集合在处理模型上有着很大的不同。

最后，我们聊一下 Elasticsearch 里面不能伸缩的一件事。



## 关联关系处理

现实世界有很多重要的关联关系：博客帖子有一些评论，银行账户有多次交易记录，客户有多个银行账户，订单有多个订单明细，文件目录有多个文件和子目录。

关系型数据库被明确设计--毫不意外--用来进行关联关系管理 ：

- 每个实体（或 *行* ，在关系世界中）可以被 *主键* 唯一标识。
- 实体 *规范化* （范式）。唯一实体的数据只存储一次，而相关实体只存储它的主键。只能在一个具体位置修改这个实体的数据。
- 实体可以进行关联查询，可以跨实体搜索。
- 单个实体的变化是 *原子的* ， *一致的* ， *隔离的* ， 和 *持久的* 。 （可以在 [*ACID Transactions*](http://en.wikipedia.org/wiki/ACID_transactions) 中查看更多细节。）
- 大多数关系数据库支持跨多个实体的 ACID 事务。

但是关系型数据库有其局限性，包括对全文检索有限的支持能力。 实体关联查询时间消耗是很昂贵的，关联的越多，消耗就越昂贵。特别是跨服务器进行实体关联时成本极其昂贵，基本不可用。 但单个的服务器上又存在数据量的限制。

Elasticsearch ，和大多数 NoSQL 数据库类似，是扁平化的。索引是独立文档的集合体。 文档是否匹配搜索请求取决于它是否包含所有的所需信息。

Elasticsearch 中单个文档的数据变更是 [ACIDic](http://en.wikipedia.org/wiki/ACID_transactions) 的， 而涉及多个文档的事务则不是。当一个事务部分失败时，无法回滚索引数据到前一个状态。

扁平化有以下优势：

- 索引过程是快速和无锁的。
- 搜索过程是快速和无锁的。
- 因为每个文档相互都是独立的，大规模数据可以在多个节点上进行分布。

但关联关系仍然非常重要。某些时候，我们需要缩小扁平化和现实世界关系模型的差异。 以下四种常用的方法，用来在 Elasticsearch 中进行关系型数据的管理：

- [Application-side joins](https://www.elastic.co/guide/cn/elasticsearch/guide/current/application-joins.html)
- [Data denormalization](https://www.elastic.co/guide/cn/elasticsearch/guide/current/denormalization.html)
- [Nested objects](https://www.elastic.co/guide/cn/elasticsearch/guide/current/nested-objects.html)
- [Parent/child relationships](https://www.elastic.co/guide/cn/elasticsearch/guide/current/parent-child.html)

通常都需要结合其中的某几个方法来得到最终的解决方案。



### 应用层联接

我们通过在我们的应用程序中实现联接可以（部分）模拟关系 数据库。 例如，比方说我们正在对用户和他们的博客文章进行索引。在关系世界中，我们会这样来操作：

```json
PUT /my_index/user/1                        <1>
{
  "name":     "John Smith",
  "email":    "john@smith.com",
  "dob":      "1970/10/24"
}

PUT /my_index/blogpost/2                    <2>
{
  "title":    "Relationships",
  "body":     "It's complicated...",
  "user":     1                             <3>
}
```
>  ![img](assets/1.png)  ![img](assets/2.png)  每个文档的 `index`, `type`, 和 `id` 一起构造成主键。         
>  
>  ![img](assets/3.png)  `blogpost` 通过用户的 `id` 链接到用户。`index` 和 `type` 并不需要因为在我们的应用程序中已经硬编码。  

通过用户的 ID `1` 可以很容易的找到博客帖子。

```json
GET /my_index/blogpost/_search
{
  "query": {
    "filtered": {
      "filter": {
        "term": { "user": 1 }
      }
    }
  }
}
```

为了找到用户叫做 John 的博客帖子，我们需要运行两次查询： 第一次会查找所有叫做 John 的用户从而获取他们的 ID 集合，接着第二次会将这些 ID 集合放到类似于前面一个例子的查询：

```json
GET /my_index/user/_search
{
  "query": {
    "match": {
      "name": "John"
    }
  }
}

GET /my_index/blogpost/_search
{
  "query": {
    "filtered": {
      "filter": {
        "terms": { "user": [1] }         <1>
      }
    }
  }
}
```
>  ![img](assets/1.png)  执行第一个查询得到的结果将填充到 `terms` 过滤器中。   

应用层联接的主要优点是可以对数据进行标准化处理。只能在 `user` 文档中修改用户的名称。缺点是，为了在搜索时联接文档，必须运行额外的查询。

在这个例子中，只有一个用户匹配我们的第一个查询，但在现实世界中，我们可以很轻易的遇到数以百万计的叫做 John 的用户。 包含所有这些用户的 IDs 会产生一个非常大的查询，这是一个数百万词项的查找。

这种方法适用于第一个实体（例如，在这个例子中 `user` ）只有少量的文档记录的情况，并且最好它们很少改变。这将允许应用程序对结果进行缓存，并避免经常运行第一次查询。



### 非规范化你的数据

使用 Elasticsearch 得到最好的搜索性能的方法是有目的的通过在索引时进行非规范化 [denormalizing](http://en.wikipedia.org/wiki/Denormalization)。对每个文档保持一定数量的冗余副本可以在需要访问时避免进行关联。

如果我们希望能够通过某个用户姓名找到他写的博客文章，可以在博客文档中包含这个用户的姓名：

```json
PUT /my_index/user/1
{
  "name":     "John Smith",
  "email":    "john@smith.com",
  "dob":      "1970/10/24"
}

PUT /my_index/blogpost/2
{
  "title":    "Relationships",
  "body":     "It's complicated...",
  "user":     {
    "id":       1,
    "name":     "John Smith"                <1>
  }
}
```
>  ![img](assets/1.png)  这部分用户的字段数据已被冗余到 `blogpost` 文档中。   

现在，我们通过单次查询就能够通过 `relationships` 找到用户 `John` 的博客文章。

```json
GET /my_index/blogpost/_search
{
  "query": {
    "bool": {
      "must": [
        { "match": { "title":     "relationships" }},
        { "match": { "user.name": "John"          }}
      ]
    }
  }
}
```

数据非规范化的优点是速度快。因为每个文档都包含了所需的所有信息，当这些信息需要在查询进行匹配时，并不需要进行昂贵的联接操作。



### 字段折叠

一个普遍的需求是需要通过特定字段进行分组。 例如我们需要按照用户名称 *分组* 返回最相关的博客文章。按照用户名分组意味着进行 `terms` 聚合。 为能够按照用户 *整体* 名称进行分组，名称字段应保持 `not_analyzed` 的形式， 具体说明参考 [聚合与分析](https://www.elastic.co/guide/cn/elasticsearch/guide/current/aggregations-and-analysis.html)：

```json
PUT /my_index/_mapping/blogpost
{
  "properties": {
    "user": {
      "properties": {
        "name": {                            <1>
          "type": "string",
          "fields": {
            "raw": {                         <2>
              "type":  "string",
              "index": "not_analyzed"
            }
          }
        }
      }
    }
  }
}
```
>  ![img](assets/1.png)   `user.name` 字段将用来进行全文检索。  
>  
>  ![img](assets/2.png)  `user.name.raw` 字段将用来通过 `terms` 聚合进行分组。  

然后添加一些数据:

```json
PUT /my_index/user/1
{
  "name": "John Smith",
  "email": "john@smith.com",
  "dob": "1970/10/24"
}

PUT /my_index/blogpost/2
{
  "title": "Relationships",
  "body": "It's complicated...",
  "user": {
    "id": 1,
    "name": "John Smith"
  }
}

PUT /my_index/user/3
{
  "name": "Alice John",
  "email": "alice@john.com",
  "dob": "1979/01/04"
}

PUT /my_index/blogpost/4
{
  "title": "Relationships are cool",
  "body": "It's not complicated at all...",
  "user": {
    "id": 3,
    "name": "Alice John"
  }
}
```

现在我们来查询标题包含 `relationships` 并且作者名包含 `John` 的博客，查询结果再按作者名分组，感谢 [`top_hits` aggregation](https://www.elastic.co/guide/en/elasticsearch/reference/5.6/search-aggregations-metrics-top-hits-aggregation.html) 提供了按照用户进行分组的功能：

```json
GET /my_index/blogpost/_search
{
  "size" : 0,                                                  <1>
  "query": {                                                   <2>
    "bool": {
      "must": [
        { "match": { "title":     "relationships" }},
        { "match": { "user.name": "John"          }}
      ]
    }
  },
  "aggs": {
    "users": {
      "terms": {
        "field":   "user.name.raw",                            <3>
        "order": { "top_score": "desc" }                       <4>
      },
      "aggs": {
        "top_score": { "max":      { "script":  "_score"           }},   <5>
        "blogposts": { "top_hits": { "_source": "title", "size": 5 }}    <6>
      }
    }
  }
}
```
>  ![img](assets/1.png)  我们感兴趣的博客文章是通过 `blogposts` 聚合返回的，所以我们可以通过将 `size` 设置成 0 来禁止 `hits` 常规搜索。   
>  
>  ![img](assets/2.png)  `query` 返回通过 `relationships` 查找名称为 `John` 的用户的博客文章。   
>  
>![img](assets/3.png)  `terms` 聚合为每一个 `user.name.raw` 创建一个桶。      
>
>  ![img](assets/4.png)![img](assets/5.png)   `top_score` 聚合对通过 `users` 聚合得到的每一个桶按照文档评分对词项进行排序。   
>  
>  ![img](assets/6.png)  `top_hits` 聚合仅为每个用户返回五个最相关的博客文章的 `title` 字段。   

这里显示简短响应结果：

```json
...
"hits": {
  "total":     2,
  "max_score": 0,
  "hits":      []                                       <1>
},
"aggregations": {
  "users": {
     "buckets": [
        { 
           "key":       "John Smith",                   <2>
           "doc_count": 1,
           "blogposts": {
              "hits": {                                 <3>
                 "total":     1,
                 "max_score": 0.35258877,
                 "hits": [
                    {
                       "_index": "my_index",
                       "_type":  "blogpost",
                       "_id":    "2",
                       "_score": 0.35258877,
                       "_source": {
                          "title": "Relationships"
                       }
                    }
                 ]
              }
           },
           "top_score": {                               <4>
              "value": 0.3525887727737427
           }
        },
...
```
>  ![img](assets/1.png)  因为我们设置 `size` 为 0 ，所以 `hits` 数组是空的。  
>  
>  ![img](assets/2.png)  在顶层查询结果中出现的每一个用户都会有一个对应的桶。  
>  
>  ![img](assets/3.png)   在每个用户桶下面都会有一个 `blogposts.hits` 数组包含针对这个用户的顶层查询结果。  
>
>  ![img](assets/4.png)  用户桶按照每个用户最相关的博客文章进行排序。  

使用 `top_hits` 聚合等效执行一个查询返回这些用户的名字和他们最相关的博客文章，然后为每一个用户执行相同的查询，以获得最好的博客。但前者的效率要好很多。

每一个桶返回的顶层查询命中结果是基于最初主查询进行的一个轻量 *迷你查询* 结果集。这个迷你查询提供了一些你期望的常用特性，例如高亮显示以及分页功能。



### 非规范化和并发

当然，数据非规范化也有弊端。 第一个缺点是索引会更大因为每个博客文章文档的 `_source` 将会更大，并且这里有很多的索引字段。这通常不是一个大问题。数据写到磁盘将会被高度压缩，而且磁盘已经很廉价了。Elasticsearch 可以愉快地应付这些额外的数据。

更重要的问题是，如果用户改变了他的名字，他所有的博客文章也需要更新了。幸运的是，用户不经常更改名称。即使他们做了， 用户也不可能写超过几千篇博客文章，所以更新博客文章通过 [`scroll`](https://www.elastic.co/guide/cn/elasticsearch/guide/current/scroll.html) 和 [`bulk`](https://www.elastic.co/guide/cn/elasticsearch/guide/current/bulk.html)APIs 大概耗费不到一秒。

然而，让我们考虑一个更复杂的场景，其中的变化很常见，影响深远，而且非常重要，并发。

在这个例子中，我们将在 Elasticsearch 模拟一个文件系统的目录树，非常类似 Linux 文件系统：根目录是 `/` ，每个目录可以包含文件和子目录。

我们希望能够搜索到一个特定目录下的文件，等效于：

```
grep "some text" /clinton/projects/elasticsearch/*
```

这就要求我们索引文件所在目录的路径：

```json
PUT /fs/file/1
{
  "name":     "README.txt",                                       <1>
  "path":     "/clinton/projects/elasticsearch", 
  "contents": "Starting a new Elasticsearch project is easy..."   <2>
}
```
>  ![img](assets/1.png)   文件名    
>  
>  ![img](assets/2.png)文件所在目录的全路径  

>  ![注意](assets/note.png)  事实上，我们也应当索引 `directory` 文档，如此我们可以在目录内列出所有的文件和子目录，但为了简洁，我们将忽略这个需求。  

我们也希望能够搜索到一个特定目录下的目录树包含的的任何文件，相当于此：

```
grep -r "some text" /clinton
```

为了支持这一点，我们需要对路径层次结构进行索引：

- `/clinton`
- `/clinton/projects`
- `/clinton/projects/elasticsearch`

这种层次结构能够通过 `path` 字段使用 [`path_hierarchy` tokenizer](https://www.elastic.co/guide/en/elasticsearch/reference/5.6/analysis-pathhierarchy-tokenizer.html) 自动生成：

```json
PUT /fs
{
  "settings": {
    "analysis": {
      "analyzer": {
        "paths": {                          <1>
          "tokenizer": "path_hierarchy"
        }
      }
    }
  }
}
```
>  ![img](assets/1.png)  自定义的 `paths` 分析器在默认设置中使用 [`path_hierarchy` tokenizer](https://www.elastic.co/guide/en/elasticsearch/reference/5.6/analysis-pathhierarchy-tokenizer.html)。   

`file` 类型的映射看起来如下所示：

```json
PUT /fs/_mapping/file
{
  "properties": {
    "name": {                         <1>
      "type":  "string",
      "index": "not_analyzed"
    },
    "path": {                         <2>
      "type":  "string",
      "index": "not_analyzed",
      "fields": {
        "tree": {                     <3>
          "type":     "string",
          "analyzer": "paths"
        }
      }
    }
  }
}
```
>  ![img](assets/1.png)  `name` 字段将包含确切名称。    
>  
>  ![img](assets/2.png) ![img](assets/3.png)  `path` 字段将包含确切的目录名称，而 `path.tree` 字段将包含路径层次结构。   

一旦索引建立并且文件已被编入索引，我们可以执行一个搜索，在 `/clinton/projects/elasticsearch` 目录中包含 `elasticsearch` 的文件，如下所示：

```json
GET /fs/file/_search
{
  "query": {
    "filtered": {
      "query": {
        "match": {
          "contents": "elasticsearch"
        }
      },
      "filter": {
        "term": {                                       <1>
          "path": "/clinton/projects/elasticsearch"
        }
      }
    }
  }
}
```
>  ![img](assets/1.png)  仅在该目录中查找文件。   

所有在 `/clinton` 下面的任何子目录存放的文件将在 `path.tree` 字段中包含 `/clinton` 词项。所以我们能够搜索 `/clinton` 的任何子目录中的所有文件，如下所示：

```json
GET /fs/file/_search
{
  "query": {
    "filtered": {
      "query": {
        "match": {
          "contents": "elasticsearch"
        }
      },
      "filter": {
        "term": {                               <1>
          "path.tree": "/clinton"
        }
      }
    }
  }
}
```
>  ![img](assets/1.png)  在这个目录或其下任何子目录中查找文件。  

**重命名文件和目录**

到目前为止一切顺利。 重命名一个文件很容易--所需要的只是一个简单的 `update` 或 `index` 请求。 你甚至可以使用 [optimistic concurrency control](https://www.elastic.co/guide/cn/elasticsearch/guide/current/optimistic-concurrency-control.html) 确保你的变化不会与其他用户的变化发生冲突：

```json
PUT /fs/file/1?version=2                                          <1>
{
  "name":     "README.asciidoc",
  "path":     "/clinton/projects/elasticsearch",
  "contents": "Starting a new Elasticsearch project is easy..."
}
```
>  ![img](assets/1.png) `version` 编号确保该更改仅应用于该索引中具有此相同的版本号的文档。     

我们甚至可以重命名一个目录，但这意味着更新所有存在于该目录下路径层次结构中的所有文件。 这可能快速或缓慢，取决于有多少文件需要更新。我们所需要做的就是使用 [`scroll`](https://www.elastic.co/guide/cn/elasticsearch/guide/current/scroll.html) 来检索所有的文件， 以及 [`bulk` API](https://www.elastic.co/guide/cn/elasticsearch/guide/current/bulk.html) 来更新它们。这个过程不是原子的，但是所有的文件将会迅速转移到他们的新存放位置。



### 解决并发问题

当我们允许多个人 *同时* 重命名文件或目录时，问题就来了。 设想一下，你正在对一个包含了成百上千文件的目录 `/clinton` 进行重命名操作。 同时，另一个用户对这个目录下的单个文件 `/clinton/projects/elasticsearch/README.txt` 进行重命名操作。 这个用户的修改操作，尽管在你的操作后开始，但可能会更快的完成。

以下有两种情况可能出现：

- 你决定使用 `version` （版本）号，在这种情况下，当与 `README.txt` 文件重命名的版本号产生冲突时，你的批量重命名操作将会失败。
- 你没有使用版本控制，你的变更将覆盖其他用户的变更。

问题的原因是 Elasticsearch 不支持 [ACID 事务](http://en.wikipedia.org/wiki/ACID_transactions)。 对单个文件的变更是 ACIDic 的，但包含多个文档的变更不支持。

如果你的主要数据存储是关系数据库，并且 Elasticsearch 仅仅作为一个搜索引擎 或一种提升性能的方法，可以首先在数据库中执行变更动作，然后在完成后将这些变更复制到 Elasticsearch。 通过这种方式，你将受益于数据库 ACID 事务支持，并且在 Elasticsearch 中以正确的顺序产生变更。 并发在关系数据库中得到了处理。

如果你不使用关系型存储，这些并发问题就需要在 Elasticsearch 的事务水准进行处理。 以下是三个切实可行的使用 Elasticsearch 的解决方案，它们都涉及某种形式的锁：

- 全局锁
- 文档锁
- 树锁

>  ![提示](assets/tip.png)当使用一个外部系统替代 Elasticsearch 时，本节中所描述的解决方案可以通过相同的原则来实现。  



**全局锁**

通过在任何时间只允许一个进程来进行变更动作，我们可以完全避免并发问题。 大多数的变更只涉及少量文件，会很快完成。一个顶级目录的重命名操作会对其他变更造成较长时间的阻塞，但可能很少这样做。

因为在 Elasticsearch 文档级别的变更支持 ACIDic，我们可以使用一个文档是否存在的状态作为一个全局锁。 为了请求得到锁，我们尝试 `create` 全局锁文档：

```json
PUT /fs/lock/global/_create
{}
```

如果这个 `create` 请求因冲突异常而失败，说明另一个进程已被授予全局锁，我们将不得不稍后再试。 如果请求成功了，我们自豪的成为全局锁的主人，然后可以继续完成我们的变更。一旦完成，我们就必须通过删除全局锁文档来释放锁：

```json
DELETE /fs/lock/global
```

根据变更的频繁程度以及时间消耗，一个全局锁能对系统造成大幅度的性能限制。 我们可以通过让我们的锁更细粒度的方式来增加并行度。



**文档锁**

我们可以使用前面描述相同的方法技术来锁定个体文档，而不是锁定整个文件系统。 我们可以使用 [scrolled search](https://www.elastic.co/guide/cn/elasticsearch/guide/current/scroll.html) 检索所有的文档，这些文档会被变更影响因此每一个文档都创建了一个锁文件：

```json
PUT /fs/lock/_bulk
{ "create": { "_id": 1}}     <1>
{ "process_id": 123    }     <2>
{ "create": { "_id": 2}}
{ "process_id": 123    }
```
>  ![img](assets/1.png)  `lock` 文档的 ID 将与应被锁定的文件的 ID 相同。  
>  
>  ![img](assets/2.png)   `process_id` 代表要执行变更进程的唯一 ID。    

如果一些文件已被锁定，部分的 `bulk` 请求将失败，我们将不得不再次尝试。

当然，如果我们试图再次锁定 *所有* 的文件， 我们前面使用的 `create` 语句将会失败，因为所有文件都已被我们锁定！ 我们需要一个 `update` 请求带 `upsert` 参数以及下面这个 `script` ，而不是一个简单的 `create`语句：

```groovy
if ( ctx._source.process_id != process_id ) {    <1>
  assert false;                                  <2>
}
ctx.op = 'noop';                                 <3> 
```
>  ![img](assets/1.png)  `process_id` 是传递到脚本的一个参数。  
>  
>  ![img](assets/2.png)  `assert false` 将引发异常，导致更新失败。  
>  
>  ![img](assets/3.png)   将 `op` 从 `update` 更新到 `noop` 防止更新请求作出任何改变，但仍返回成功。  

完整的 `update` 请求如下所示：

```json
POST /fs/lock/1/_update
{
  "upsert": { "process_id": 123 },
  "script": "if ( ctx._source.process_id != process_id )
  { assert false }; ctx.op = 'noop';"
  "params": {
    "process_id": 123
  }
}
```

如果文档并不存在， `upsert` 文档将会被插入--和前面 `create` 请求相同。 但是，如果该文件 *确实* 存在，该脚本会查看存储在文档上的 `process_id` 。 如果 `process_id` 匹配，更新不会执行（ `noop` ）但脚本会返回成功。 如果两者并不匹配， `assert false` 抛出一个异常，你也知道了获取锁的尝试已经失败。

一旦所有锁已成功创建，你就可以继续进行你的变更。

之后，你必须释放所有的锁，通过检索所有的锁文档并进行批量删除，可以完成锁的释放：

```json
POST /fs/_refresh                     <1>
 
GET /fs/lock/_search?scroll=1m        <2> 
{
    "sort" : ["_doc"],
    "query": {
        "match" : {
            "process_id" : 123
        }
    }
}

PUT /fs/lock/_bulk
{ "delete": { "_id": 1}}
{ "delete": { "_id": 2}}
```
>  ![img](assets/1.png)  `refresh` 调用确保所有 `lock` 文档对搜索请求可见。  
>  
>  ![img](assets/2.png)  当你需要在单次搜索请求返回大量的检索结果集时，你可以使用 [`scroll`](https://www.elastic.co/guide/cn/elasticsearch/guide/current/scroll.html) 查询。  

文档级锁可以实现细粒度的访问控制，但是为数百万文档创建锁文件开销也很大。 在某些情况下，你可以用少得多的工作量实现细粒度的锁定，如以下目录树场景中所示。



**树锁**

在前面的例子中，我们可以锁定的目录树的一部分，而不是锁定每一个涉及的文档。 我们将需要独占访问我们要重命名的文件或目录，它可以通过 *独占锁* 文档来实现：

```json
{ "lock_type": "exclusive" }
```

同时我们需要共享锁定所有的父目录，通过 *共享锁* 文档：

```json
{
  "lock_type":  "shared",
  "lock_count": 1                <1>
}
```
>  ![img](assets/1.png)  `lock_count` 记录持有共享锁进程的数量。   

对 `/clinton/projects/elasticsearch/README.txt` 进行重命名的进程需要在这个文件上有 *独占锁* ， 以及在 `/clinton` 、 `/clinton/projects` 和 `/clinton/projects/elasticsearch` 目录有 *共享锁* 。

一个简单的 `create` 请求将满足独占锁的要求，但共享锁需要脚本的更新来实现一些额外的逻辑：

```groovy
if (ctx._source.lock_type == 'exclusive') {
  assert false;                                  <1>
}
ctx._source.lock_count++                         <2>
```
>  ![img](assets/1.png)  如果 `lock_type` 是 `exclusive` （独占）的，`assert` 语句将抛出一个异常，导致更新请求失败。   
>  
>  ![img](assets/2.png)   否则，我们对 `lock_count` 进行增量处理。  

这个脚本处理了 `lock` 文档已经存在的情况，但我们还需要一个用来处理的文档还不存在情况的 `upsert` 文档。 完整的更新请求如下：

```json
POST /fs/lock/%2Fclinton/_update                            <1>
{
  "upsert": {                                               <2>
    "lock_type":  "shared",
    "lock_count": 1
  },
  "script": "if (ctx._source.lock_type == 'exclusive')
  { assert false }; ctx._source.lock_count++"
}
```
>  ![img](assets/1.png)  文档的 ID 是 `/clinton` ，经过URL编码后成为 `%2fclinton` 。   
>  
>  ![img](assets/2.png)  `upsert` 文档如果不存在，则会被插入。   

一旦我们成功地在所有的父目录中获得一个共享锁，我们尝试在文件本身 `create` 一个独占锁：

```json
PUT /fs/lock/%2Fclinton%2fprojects%2felasticsearch%2fREADME.txt/_create
{ "lock_type": "exclusive" }
```

现在，如果有其他人想要重新命名 `/clinton` 目录，他们将不得不在这条路径上获得一个独占锁：

```json
PUT /fs/lock/%2Fclinton/_create
{ "lock_type": "exclusive" }
```

这个请求将失败，因为一个具有相同 ID 的 `lock` 文档已经存在。 另一个用户将不得不等待我们的操作完成以及释放我们的锁。独占锁只能这样被删除：

```json
DELETE /fs/lock/%2Fclinton%2fprojects%2felasticsearch%2fREADME.txt
```

共享锁需要另一个脚本对 `lock_count` 递减，如果计数下降到零，删除 `lock` 文档：

```groovy
if (--ctx._source.lock_count == 0) {
  ctx.op = 'delete'                           <1>
}
```
>  ![img](assets/1.png)  一旦 `lock_count` 达到0， `ctx.op` 会从 `update` 被修改成 `delete` 。   

此更新请求将为每级父目录由下至上的执行，从最长路径到最短路径：

```json
POST /fs/lock/%2Fclinton%2fprojects%2felasticsearch/_update
{
  "script": "if (--ctx._source.lock_count == 0) { ctx.op = 'delete' } "
}
```

树锁用最小的代价提供了细粒度的并发控制。当然，它不适用于所有的情况--数据模型必须有类似于目录树的顺序访问路径才能使用。
>  ![注意](assets/note.png)  这三个方案--全局、文档或树锁--都没有处理锁最棘手的问题：如果持有锁的进程死了怎么办？  
>  
>  一个进程的意外死亡给我们留下了2个问题：
>  
>  - 我们如何知道我们可以释放的死亡进程中所持有的锁？
>  - 我们如何清理死去的进程没有完成的变更？
>  
>  这些主题超出了本书的范围，但是如果你决定使用锁，你需要给对他们进行一些思考。

当非规范化成为很多项目的一个很好的选择，采用锁方案的需求会带来复杂的实现逻辑。 作为替代方案，Elasticsearch 提供两个模型帮助我们处理相关联的实体： *嵌套的对象* 和 *父子关系* 。



## 嵌套对象

由于在 Elasticsearch 中单个文档的增删改都是原子性操作,那么将相关实体数据都存储在同一文档中也就理所当然。 比如说,我们可以将订单及其明细数据存储在一个文档中。又比如,我们可以将一篇博客文章的评论以一个 `comments` 数组的形式和博客文章放在一起：

```json
PUT /my_index/blogpost/1
{
  "title": "Nest eggs",
  "body":  "Making your money work...",
  "tags":  [ "cash", "shares" ],
  "comments": [                                  <1>
    {
      "name":    "John Smith",
      "comment": "Great article",
      "age":     28,
      "stars":   4,
      "date":    "2014-09-01"
    },
    {
      "name":    "Alice White",
      "comment": "More like this please",
      "age":     31,
      "stars":   5,
      "date":    "2014-10-22"
    }
  ]
}
```
>  ![img](assets/1.png)  如果我们依赖[字段自动映射](https://www.elastic.co/guide/cn/elasticsearch/guide/current/dynamic-mapping.html),那么 `comments` 字段会自动映射为 `object` 类型。   

由于所有的信息都在一个文档中,当我们查询时就没有必要去联合文章和评论文档,查询效率就很高。

但是当我们使用如下查询时,上面的文档也会被当做是符合条件的结果：

```json
GET /_search
{
  "query": {
    "bool": {
      "must": [
        { "match": { "name": "Alice" }},
        { "match": { "age":  28      }}          <1>
      ]
    }
  }
}
```
>  ![img](assets/1.png)  Alice实际是31岁,不是28!   

正如我们在 [对象数组](https://www.elastic.co/guide/cn/elasticsearch/guide/current/complex-core-fields.html#object-arrays) 中讨论的一样,出现上面这种问题的原因是 JSON 格式的文档被处理成如下的扁平式键值对的结构。

```json
{
  "title":            [ eggs, nest ],
  "body":             [ making, money, work, your ],
  "tags":             [ cash, shares ],
  "comments.name":    [ alice, john, smith, white ],
  "comments.comment": [ article, great, like, more, please, this ],
  "comments.age":     [ 28, 31 ],
  "comments.stars":   [ 4, 5 ],
  "comments.date":    [ 2014-09-01, 2014-10-22 ]
}
```

`Alice` 和 31 、 `John` 和 `2014-09-01` 之间的相关性信息不再存在。虽然 `object` 类型 (参见 [内部对象](https://www.elastic.co/guide/cn/elasticsearch/guide/current/complex-core-fields.html#inner-objects)) 在存储 *单一对象* 时非常有用,但对于对象数组的搜索而言,毫无用处。

*嵌套对象* 就是来解决这个问题的。将 `comments` 字段类型设置为 `nested` 而不是 `object` 后,每一个嵌套对象都会被索引为一个 *隐藏的独立文档* ,举例如下:

```json
{   <1>
  "comments.name":    [ john, smith ],
  "comments.comment": [ article, great ],
  "comments.age":     [ 28 ],
  "comments.stars":   [ 4 ],
  "comments.date":    [ 2014-09-01 ]
} 
{   <2>
  "comments.name":    [ alice, white ],
  "comments.comment": [ like, more, please, this ],
  "comments.age":     [ 31 ],
  "comments.stars":   [ 5 ],
  "comments.date":    [ 2014-10-22 ]
}
{   <3>
  "title":            [ eggs, nest ],
  "body":             [ making, money, work, your ],
  "tags":             [ cash, shares ]
}
```
>  ![img](assets/1.png)   第一个 `嵌套文档`    
>  
>  ![img](assets/2.png)  第二个 `嵌套文档`   
>  
>  ![img](assets/3.png)  *根文档* 或者也可称为父文档   

在独立索引每一个嵌套对象后,对象中每个字段的相关性得以保留。我们查询时,也仅仅返回那些真正符合条件的文档。

不仅如此,由于嵌套文档直接存储在文档内部,查询时嵌套文档和根文档联合成本很低,速度和单独存储几乎一样。

嵌套文档是隐藏存储的,我们不能直接获取。如果要增删改一个嵌套对象,我们必须把整个文档重新索引才可以。值得注意的是,查询的时候返回的是整个文档,而不是嵌套文档本身。











