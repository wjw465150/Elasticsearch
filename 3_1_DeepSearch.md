# 深入搜索



在 [基础入门](https://www.elastic.co/guide/cn/elasticsearch/guide/current/getting-started.html) 中涵盖了基本工具并对它们有足够详细的描述，这让我们能够开始用 Elasticsearch 搜索数据。 用不了多长时间，就会发现我们想要的更多：希望查询匹配更灵活，排名结果更精确，不同问题域下搜索更具体。

想要进阶，只知道如何使用 `match` 查询是不够的，我们需要理解数据以及如何能够搜索到它们。本章会解释如何索引和查询我们的数据让我们能利用词的相似度（word proximity）、部分匹配（partial matching）、模糊匹配（fuzzy matching）以及语言感知（language awareness）这些优势。

理解每个查询如何贡献相关度评分 `_score` 有助于调试我们的查询：确保我们认为的最佳匹配文档出现在结果首页，以及削减结果中几乎不相关的 “长尾（long tail）”。

搜索不仅仅是全文搜索：我们很大一部分数据都是结构化的，如日期和数字。 我们会以说明结构化搜索与全文搜索最高效的结合方式开始本章的内容。



## 结构化搜索

*结构化搜索（Structured search）* 是指有关探询那些具有内在结构数据的过程。比如日期、时间和数字都是结构化的：它们有精确的格式，我们可以对这些格式进行逻辑操作。比较常见的操作包括比较数字或时间的范围，或判定两个值的大小。

文本也可以是结构化的。如彩色笔可以有离散的颜色集合： `红（red）` 、 `绿（green）` 、 `蓝（blue）` 。一个博客可能被标记了关键词 `分布式（distributed）` 和 `搜索（search）` 。电商网站上的商品都有 UPCs（通用产品码 Universal Product Codes）或其他的唯一标识，它们都需要遵从严格规定的、结构化的格式。

在结构化查询中，我们得到的结果 *总是* 非是即否，要么存于集合之中，要么存在集合之外。结构化查询不关心文件的相关度或评分；它简单的对文档包括或排除处理。

这在逻辑上是能说通的，因为一个数字不能比其他数字 *更* 适合存于某个相同范围。结果只能是：存于范围之中，抑或反之。同样，对于结构化文本来说，一个值要么相等，要么不等。没有 *更似* 这种概念。



### 精确值查找

当进行精确值查找时， 我们会使用过滤器（filters）。过滤器很重要，因为它们执行速度非常快，不会计算相关度（直接跳过了整个评分阶段）而且很容易被缓存。我们会在本章后面的 [过滤器缓存](https://www.elastic.co/guide/cn/elasticsearch/guide/current/filter-caching.html) 中讨论过滤器的性能优势，不过现在只要记住：请尽可能多的使用过滤式查询。

**term 查询数字**

我们首先来看最为常用的 `term` 查询， 可以用它处理数字（numbers）、布尔值（Booleans）、日期（dates）以及文本（text）。

```
让我们以下面的例子开始介绍，创建并索引一些表示产品的文档，文档里有字段 `price` 和 `productID` （ `价格` 和 `产品ID` ）：
```



```json
POST /my_store/products/_bulk
{ "index": { "_id": 1 }}
{ "price" : 10, "productID" : "XHDK-A-1293-#fJ3" }
{ "index": { "_id": 2 }}
{ "price" : 20, "productID" : "KDKE-B-9947-#kL5" }
{ "index": { "_id": 3 }}
{ "price" : 30, "productID" : "JODL-X-1937-#pV7" }
{ "index": { "_id": 4 }}
{ "price" : 30, "productID" : "QQPX-R-3956-#aD8" }
```

我们想要做的是查找具有某个价格的所有产品，有关系数据库背景的人肯定熟悉 SQL，如果我们将其用 SQL 形式表达，会是下面这样：

```sql
SELECT document
FROM   products
WHERE  price = 20
```

在 Elasticsearch 的查询表达式（query DSL）中，我们可以使用 `term` 查询达到相同的目的。 `term` 查询会查找我们指定的精确值。作为其本身， `term` 查询是简单的。它接受一个字段名以及我们希望查找的数值：

```js
{
    "term" : {
        "price" : 20
    }
}
```

通常当查找一个精确值的时候，我们不希望对查询进行评分计算。只希望对文档进行包括或排除的计算，所以我们会使用 `constant_score` 查询以非评分模式来执行 `term` 查询并以一作为统一评分。

最终组合的结果是一个 `constant_score` 查询，它包含一个 `term` 查询：

```js
GET /my_store/products/_search
{
    "query" : {
        "constant_score" : { <1>
            "filter" : {
                "term" : {   <2>
                    "price" : 20
                }
            }
        }
    }
}
```
>  ![img](assets/1.png)  我们用 `constant_score` 将 `term` 查询转化成为过滤器   
>
>  ![img](assets/2.png)  我们之前看到过的 `term` 查询   

执行后，这个查询所搜索到的结果与我们期望的一致：只有文档 2 命中并作为结果返回（因为只有 `2` 的价格是 `20` ）:

```json
"hits" : [
    {
        "_index" : "my_store",
        "_type" :  "products",
        "_id" :    "2",
        "_score" : 1.0,       <1>
        "_source" : {
          "price" :     20,
          "productID" : "KDKE-B-9947-#kL5"
        }
    }
]
```
>  ![img](assets/1.png)  查询置于 `filter` 语句内不进行评分或相关度的计算，所以所有的结果都会返回一个默认评分 `1` 。 

**term 查询文本**

如本部分开始处提到过的一样 ，使用 `term` 查询匹配字符串和匹配数字一样容易。如果我们想要查询某个具体 UPC ID 的产品，使用 SQL 表达式会是如下这样：

```sql
SELECT product
FROM   products
WHERE  productID = "XHDK-A-1293-#fJ3"
```

转换成查询表达式（query DSL），同样使用 `term` 查询，形式如下：

```js
GET /my_store/products/_search
{
    "query" : {
        "constant_score" : {
            "filter" : {
                "term" : {
                    "productID" : "XHDK-A-1293-#fJ3"
                }
            }
        }
    }
}
```



但这里有个小问题：我们无法获得期望的结果。为什么呢？问题不在 `term` 查询，而在于索引数据的方式。如果我们使用 `analyze` API ([分析 API](https://www.elastic.co/guide/cn/elasticsearch/guide/current/analysis-intro.html#analyze-api))，我们可以看到这里的 UPC 码被拆分成多个更小的 token ：

```js
GET /my_store/_analyze
{
  "field": "productID",
  "text": "XHDK-A-1293-#fJ3"
}
```



```js
{
  "tokens" : [ {
    "token" :        "xhdk",
    "start_offset" : 0,
    "end_offset" :   4,
    "type" :         "<ALPHANUM>",
    "position" :     1
  }, {
    "token" :        "a",
    "start_offset" : 5,
    "end_offset" :   6,
    "type" :         "<ALPHANUM>",
    "position" :     2
  }, {
    "token" :        "1293",
    "start_offset" : 7,
    "end_offset" :   11,
    "type" :         "<NUM>",
    "position" :     3
  }, {
    "token" :        "fj3",
    "start_offset" : 13,
    "end_offset" :   16,
    "type" :         "<ALPHANUM>",
    "position" :     4
  } ]
}
```

这里有几点需要注意：

- Elasticsearch 用 4 个不同的 token 而不是单个 token 来表示这个 UPC 。
- 所有字母都是小写的。
- 丢失了连字符和哈希符（ `#` ）。

所以当我们用 `term` 查询查找精确值 `XHDK-A-1293-#fJ3` 的时候，找不到任何文档，因为它并不在我们的倒排索引中，正如前面呈现出的分析结果，索引里有四个 token 。

显然这种对 ID 码或其他任何精确值的处理方式并不是我们想要的。

为了避免这种问题，我们需要告诉 Elasticsearch 该字段具有精确值，要将其设置成 `not_analyzed` 无需分析的。 我们可以在 [自定义字段映射](https://www.elastic.co/guide/cn/elasticsearch/guide/current/mapping-intro.html#custom-field-mappings) 中查看它的用法。为了修正搜索结果，我们需要首先删除旧索引（因为它的映射不再正确）然后创建一个能正确映射的新索引：

```js
DELETE /my_store   <1>

PUT /my_store      <2>
{
    "mappings" : {
        "products" : {
            "properties" : {
                "productID" : {
                    "type" : "string",
                    "index" : "not_analyzed"    <3>
                }
            }
        }
    }

}
```
>  ![img](assets/1.png)  删除索引是必须的，因为我们不能更新已存在的映射。   
>  
>  ![img](assets/2.png)  在索引被删除后，我们可以创建新的索引并为其指定自定义映射。   
>  
>  ![img](assets/3.png)  这里我们告诉 Elasticsearch ，我们不想对 `productID` 做任何分析。    

现在我们可以为文档重建索引：

```js
POST /my_store/products/_bulk
{ "index": { "_id": 1 }}
{ "price" : 10, "productID" : "XHDK-A-1293-#fJ3" }
{ "index": { "_id": 2 }}
{ "price" : 20, "productID" : "KDKE-B-9947-#kL5" }
{ "index": { "_id": 3 }}
{ "price" : 30, "productID" : "JODL-X-1937-#pV7" }
{ "index": { "_id": 4 }}
{ "price" : 30, "productID" : "QQPX-R-3956-#aD8" }
```

此时， `term` 查询就能搜索到我们想要的结果，让我们再次搜索新索引过的数据（注意，查询和过滤并没有发生任何改变，改变的是数据映射的方式）：

```js
GET /my_store/products/_search
{
    "query" : {
        "constant_score" : {
            "filter" : {
                "term" : {
                    "productID" : "XHDK-A-1293-#fJ3"
                }
            }
        }
    }
}
```

因为 `productID` 字段是未分析过的， `term` 查询不会对其做任何分析，查询会进行精确查找并返回文档 1 。成功！

**内部过滤器的操作**

在内部，Elasticsearch 会在运行非评分查询的时执行多个操作：

1. *查找匹配文档*.

   `term` 查询在倒排索引中查找 `XHDK-A-1293-#fJ3` 然后获取包含该 term 的所有文档。本例中，只有文档 1 满足我们要求。

2. *创建 bitset*.

   过滤器会创建一个 *bitset* （一个包含 0 和 1 的数组），它描述了哪个文档会包含该 term 。匹配文档的标志位是 1 。本例中，bitset 的值为 `[1,0,0,0]` 。在内部，它表示成一个 ["roaring bitmap"](https://www.elastic.co/blog/frame-of-reference-and-roaring-bitmaps)，可以同时对稀疏或密集的集合进行高效编码。

3. *迭代 bitset(s)*

   一旦为每个查询生成了 bitsets ，Elasticsearch 就会循环迭代 bitsets 从而找到满足所有过滤条件的匹配文档的集合。执行顺序是启发式的，但一般来说先迭代稀疏的 bitset （因为它可以排除掉大量的文档）。

4. *增量使用计数*.

   Elasticsearch 能够缓存非评分查询从而获取更快的访问，但是它也会不太聪明地缓存一些使用极少的东西。非评分计算因为倒排索引已经足够快了，所以我们只想缓存那些我们 *知道* 在将来会被再次使用的查询，以避免资源的浪费。

   为了实现以上设想，Elasticsearch 会为每个索引跟踪保留查询使用的历史状态。如果查询在最近的 256 次查询中会被用到，那么它就会被缓存到内存中。当 bitset 被缓存后，缓存会在那些低于 10,000 个文档（或少于 3% 的总索引数）的段（segment）中被忽略。这些小的段即将会消失，所以为它们分配缓存是一种浪费。

实际情况并非如此（执行有它的复杂性，这取决于查询计划是如何重新规划的，有些启发式的算法是基于查询代价的），理论上非评分查询 *先于* 评分查询执行。非评分查询任务旨在降低那些将对评分查询计算带来更高成本的文档数量，从而达到快速搜索的目的。

从概念上记住非评分计算是首先执行的，这将有助于写出高效又快速的搜索请求。



### 组合过滤器

前面的两个例子都是单个过滤器（filter）的使用方式。 在实际应用中，我们很有可能会过滤多个值或字段。比方说，怎样用 Elasticsearch 来表达下面的 SQL ？

```sql
SELECT product
FROM   products
WHERE  (price = 20 OR productID = "XHDK-A-1293-#fJ3")
  AND  (price != 30)
```

这种情况下，我们需要 `bool` （布尔）过滤器。 这是个 *复合过滤器（compound filter）* ，它可以接受多个其他过滤器作为参数，并将这些过滤器结合成各式各样的布尔（逻辑）组合。

**布尔过滤器**

一个 `bool` 过滤器由三部分组成：

```js
{
   "bool" : {
      "must" :     [],
      "should" :   [],
      "must_not" : [],
   }
}
```

- `must`

  所有的语句都 *必须（must）* 匹配，与 `AND` 等价。

- `must_not`

  所有的语句都 *不能（must not）* 匹配，与 `NOT` 等价。

- `should`

  至少有一个语句要匹配，与 `OR` 等价。

就这么简单！ 当我们需要多个过滤器时，只须将它们置入 `bool` 过滤器的不同部分即可。
>  ![注意](assets/note.png)  一个 `bool` 过滤器的每个部分都是可选的（例如，我们可以只有一个 `must` 语句），而且每个部分内部可以只有一个或一组过滤器。

用 Elasticsearch 来表示本部分开始处的 SQL 例子，将两个 `term` 过滤器置入 `bool` 过滤器的 `should` 语句内，再增加一个语句处理 `NOT` 非的条件：

```js
GET /my_store/products/_search
{
   "query" : {
      "filtered" : {   <1>
         "filter" : {
            "bool" : {
              "should" : [
                 { "term" : {"price" : 20}},   <2>
                 { "term" : {"productID" : "XHDK-A-1293-#fJ3"}}    <3>
              ],
              "must_not" : {
                 "term" : {"price" : 30}  <4>
              }
           }
         }
      }
   }
}
```
>  ![img](assets/1.png)  注意，我们仍然需要一个 `filtered` 查询将所有的东西包起来。    
>
>  ![img](assets/2.png) ![img](assets/3.png)  在 `should` 语句块里面的两个 `term` 过滤器与 `bool` 过滤器是父子关系，两个 `term` 条件需要匹配其一。   
>
>  ![img](assets/4.png)  如果一个产品的价格是 `30` ，那么它会自动被排除，因为它处于 `must_not` 语句里面。  

我们搜索的结果返回了 2 个命中结果，两个文档分别匹配了 `bool` 过滤器其中的一个条件：

```json
"hits" : [
    {
        "_id" :     "1",
        "_score" :  1.0,
        "_source" : {
          "price" :     10,
          "productID" : "XHDK-A-1293-#fJ3" <1>
        }
    },
    {
        "_id" :     "2",
        "_score" :  1.0,
        "_source" : {
          "price" :     20, <2>
          "productID" : "KDKE-B-9947-#kL5"
        }
    }
]
```
>  ![img](assets/1.png) 与 `term` 过滤器中 `productID = "XHDK-A-1293-#fJ3"` 条件匹配   
>
>  ![img](assets/2.png)  与 `term` 过滤器中 `price = 20` 条件匹配  



**嵌套布尔过滤器**

尽管 `bool` 是一个复合的过滤器，可以接受多个子过滤器，需要注意的是 `bool` 过滤器本身仍然还只是一个过滤器。 这意味着我们可以将一个 `bool` 过滤器置于其他 `bool` 过滤器内部，这为我们提供了对任意复杂布尔逻辑进行处理的能力。

对于以下这个 SQL 语句：

```sql
SELECT document
FROM   products
WHERE  productID      = "KDKE-B-9947-#kL5"
  OR (     productID = "JODL-X-1937-#pV7"
       AND price     = 30 )
```

我们将其转换成一组嵌套的 `bool` 过滤器：

```js
GET /my_store/products/_search
{
   "query" : {
      "filtered" : {
         "filter" : {
            "bool" : {
              "should" : [
                { "term" : {"productID" : "KDKE-B-9947-#kL5"}}, <1>
                { "bool" : { <2>
                  "must" : [
                    { "term" : {"productID" : "JODL-X-1937-#pV7"}}, 
                    { "term" : {"price" : 30}} <4>
                  ]
                }}
              ]
           }
         }
      }
   }
}
```
>  ![img](assets/1.png) ![img](assets/2.png)  因为 `term` 和 `bool` 过滤器是兄弟关系，他们都处于外层的布尔逻辑 `should` 的内部，返回的命中文档至少须匹配其中一个过滤器的条件。  
>  
>  ![img](assets/3.png) ![img](assets/4.png)  这两个 `term` 语句作为兄弟关系，同时处于 `must` 语句之中，所以返回的命中文档要必须都能同时匹配这两个条件。  

得到的结果有两个文档，它们各匹配 `should` 语句中的一个条件：

```json
"hits" : [
    {
        "_id" :     "2",
        "_score" :  1.0,
        "_source" : {
          "price" :     20,
          "productID" : "KDKE-B-9947-#kL5" <1>
        }
    },
    {
        "_id" :     "3",
        "_score" :  1.0,
        "_source" : {
          "price" :      30, <2>
          "productID" : "JODL-X-1937-#pV7" <3>
        }
    }
]
```
>  ![img](assets/1.png)  这个 `productID` 与外层的 `bool` 过滤器 `should` 里的唯一一个 `term` 匹配。  
>  
>  ![img](assets/2.png) ![img](assets/3.png)  这两个字段与嵌套的 `bool` 过滤器 `must` 里的两个 `term` 匹配。  

这只是个简单的例子，但足以展示布尔过滤器可以用来作为构造复杂逻辑条件的基本构建模块。



### 查找多个精确值

`term` 查询对于查找单个值非常有用，但通常我们可能想搜索多个值。 如果我们想要查找价格字段值为 $20 或 $30 的文档该如何处理呢？

不需要使用多个 `term` 查询，我们只要用单个 `terms` 查询（注意末尾的 *s* ）， `terms` 查询好比是 `term` 查询的复数形式（以英语名词的单复数做比）。

它几乎与 `term` 的使用方式一模一样，与指定单个价格不同，我们只要将 `term` 字段的值改为数组即可：

```js
{
    "terms" : {
        "price" : [20, 30]
    }
}
```

与 `term` 查询一样，也需要将其置入 `filter` 语句的常量评分查询中使用：

```js
GET /my_store/products/_search
{
    "query" : {
        "constant_score" : {
            "filter" : {
                "terms" : { <1>
                    "price" : [20, 30]
                }
            }
        }
    }
}
```
>  ![img](assets/1.png)  这个 `terms` 查询被置于 `constant_score` 查询中     

运行结果返回第二、第三和第四个文档：

```json
"hits" : [
    {
        "_id" :    "2",
        "_score" : 1.0,
        "_source" : {
          "price" :     20,
          "productID" : "KDKE-B-9947-#kL5"
        }
    },
    {
        "_id" :    "3",
        "_score" : 1.0,
        "_source" : {
          "price" :     30,
          "productID" : "JODL-X-1937-#pV7"
        }
    },
    {
        "_id":     "4",
        "_score":  1.0,
        "_source": {
           "price":     30,
           "productID": "QQPX-R-3956-#aD8"
        }
     }
]
```

**包含，而不是相等**

一定要了解 `term` 和 `terms` 是 *包含（contains）* 操作，而非 *等值（equals）* （判断）。 如何理解这句话呢？

如果我们有一个 term（词项）过滤器 `{ "term" : { "tags" : "search" } }` ，它会与以下两个文档 *同时*匹配：

```js
{ "tags" : ["search"] }
{ "tags" : ["search", "open_source"] } <1>
```
>  ![img](assets/1.png)  尽管第二个文档包含除 `search` 以外的其他词，它还是被匹配并作为结果返回。  

回忆一下 `term` 查询是如何工作的？ Elasticsearch 会在倒排索引中查找包括某 term 的所有文档，然后构造一个 bitset 。在我们的例子中，倒排索引表如下：

| Token         | DocIDs  |
| ------------- | ------- |
| `open_source` | `2`     |
| `search`      | `1`,`2` |

当 `term` 查询匹配标记 `search` 时，它直接在倒排索引中找到记录并获取相关的文档 ID，如倒排索引所示，这里文档 1 和文档 2 均包含该标记，所以两个文档会同时作为结果返回。
>  ![注意](assets/note.png)  由于倒排索引表自身的特性，整个字段是否相等会难以计算，如果确定某个特定文档是否 *只（only）* 包含我们想要查找的词呢？首先我们需要在倒排索引中找到相关的记录并获取文档 ID，然后再扫描 *倒排索引中的每行记录* ，查看它们是否包含其他的 terms 。
>
>  ​                    可以想象，这样不仅低效，而且代价高昂。正因如此， `term` 和 `terms` 是 *必须包含（must contain）* 操作，而不是 *必须精确相等（must equal exactly）* 。



**精确相等**

如果一定期望得到我们前面说的那种行为（即整个字段完全相等），最好的方式是增加并索引另一个字段， 这个字段用以存储该字段包含词项的数量，同样以上面提到的两个文档为例，现在我们包括了一个维护标签数的新字段：

```js
{ "tags" : ["search"], "tag_count" : 1 }
{ "tags" : ["search", "open_source"], "tag_count" : 2 }
```
一旦增加这个用来索引项 term 数目信息的字段，我们就可以构造一个 `constant_score` 查询，来确保结果中的文档所包含的词项数量与要求是一致的：

```js
GET /my_index/my_type/_search
{
    "query": {
        "constant_score" : {
            "filter" : {
                 "bool" : {
                    "must" : [
                        { "term" : { "tags" : "search" } }, <1>
                        { "term" : { "tag_count" : 1 } }    <2>
                    ]
                }
            }
        }
    }
}
```
>  ![img](assets/1.png) 查找所有包含 term `search` 的文档。  

>  ![img](assets/2.png) 确保文档只有一个标签。  

这个查询现在只会匹配具有单个标签 `search` 的文档，而不是任意一个包含 `search` 的文档。



### 范围

本章到目前为止，对于数字，只介绍如何处理精确值查询。 实际上，对数字范围进行过滤有时会更有用。例如，我们可能想要查找所有价格大于 $20 且小于 $40 美元的产品。

在 SQL 中，范围查询可以表示为：

```sql
SELECT document
FROM   products
WHERE  price BETWEEN 20 AND 40
```

Elasticsearch 有 `range` 查询， 不出所料地，可以用它来查找处于某个范围内的文档：

```js
"range" : {
    "price" : {
        "gte" : 20,
        "lte" : 40
    }
}
```

`range` 查询可同时提供包含（inclusive）和不包含（exclusive）这两种范围表达式，可供组合的选项如下：

- `gt`: `>` 大于（greater than）
- `lt`: `<` 小于（less than）
- `gte`: `>=` 大于或等于（greater than or equal to）
- `lte`: `<=` 小于或等于（less than or equal to）

**下面是一个范围查询的例子：.** 

```js
GET /my_store/products/_search
{
    "query" : {
        "constant_score" : {
            "filter" : {
                "range" : {
                    "price" : {
                        "gte" : 20,
                        "lt"  : 40
                    }
                }
            }
        }
    }
}
```

如果想要范围无界（比方说 >20 ），只须省略其中一边的限制：

```js
"range" : {
    "price" : {
        "gt" : 20
    }
}
```

**日期范围**

`range` 查询同样可以应用在日期字段上：

```js
"range" : {
    "timestamp" : {
        "gt" : "2014-01-01 00:00:00",
        "lt" : "2014-01-07 00:00:00"
    }
}
```

当使用它处理日期字段时， `range` 查询支持对 *日期计算（date math）* 进行操作，比方说，如果我们想查找时间戳在过去一小时内的所有文档：

```js
"range" : {
    "timestamp" : {
        "gt" : "now-1h"
    }
}
```

这个过滤器会一直查找时间戳在过去一个小时内的所有文档，让过滤器作为一个时间 *滑动窗口（sliding window）* 来过滤文档。

日期计算还可以被应用到某个具体的时间，并非只能是一个像 now 这样的占位符。只要在某个日期后加上一个双管符号 (`||`) 并紧跟一个日期数学表达式就能做到：

```js
"range" : {
    "timestamp" : {
        "gt" : "2014-01-01 00:00:00",
        "lt" : "2014-01-01 00:00:00||+1M" <1>
    }
}
```
>  ![img](assets/1.png) 早于 2014 年 1 月 1 日加 1 月（2014 年 2 月 1 日 零时）  

日期计算是 *日历相关（calendar aware）* 的，所以它不仅知道每月的具体天数，还知道某年的总天数（闰年）等信息。更详细的内容可以参考： [时间格式参考文档](https://www.elastic.co/guide/en/elasticsearch/reference/5.6/mapping-date-format.html) 。

**字符串范围**

`range` 查询同样可以处理字符串字段， 字符串范围可采用 *字典顺序（lexicographically）* 或字母顺序（alphabetically）。例如，下面这些字符串是采用字典序（lexicographically）排序的：

- 5, 50, 6, B, C, a, ab, abb, abc, b
>  ![注意](assets/note.png)  在倒排索引中的词项就是采取字典顺序（lexicographically）排列的，这也是字符串范围可以使用这个顺序来确定的原因。

如果我们想查找从 `a` 到 `b` （不包含）的字符串，同样可以使用 `range` 查询语法：

```js
"range" : {
    "title" : {
        "gte" : "a",
        "lt" :  "b"
    }
}
```
---

>  **注意基数**
>  
>  数字和日期字段的索引方式使高效地范围计算成为可能。 但字符串却并非如此，要想对其使用范围过滤，Elasticsearch 实际上是在为范围内的每个词项都执行 `term` 过滤器，这会比日期或数字的范围过滤慢许多。
>
>  字符串范围在过滤 *低基数（low cardinality）* 字段（即只有少量唯一词项）时可以正常工作，但是唯一词项越多，字符串范围的计算会越慢。

---


<a name="处理Null值"></a>
### 处理 Null 值

回想在之前例子中，有的文档有名为 `tags` （标签）的字段，它是个多值字段， 一个文档可能有一个或多个标签，也可能根本就没有标签。如果一个字段没有值，那么如何将它存入倒排索引中的呢？

这是个有欺骗性的问题，因为答案是：什么都不存。让我们看看之前内容里提到过的倒排索引：

| Token         | DocIDs  |
| ------------- | ------- |
| `open_source` | `2`     |
| `search`      | `1`,`2` |

如何将某个不存在的字段存储在这个数据结构中呢？无法做到！简单的说，一个倒排索引只是一个 token 列表和与之相关的文档信息，如果字段不存在，那么它也不会持有任何 token，也就无法在倒排索引结构中表现。

最终，这也就意味着 ，`null`, `[]` （空数组）和 `[null]` 所有这些都是等价的，它们无法存于倒排索引中。

显然，世界并不简单，数据往往会有缺失字段，或有显式的空值或空数组。为了应对这些状况，Elasticsearch 提供了一些工具来处理空或缺失值。

**存在查询**

第一件武器就是 `exists` 存在查询。 这个查询会返回那些在指定字段有任何值的文档，让我们索引一些示例文档并用标签的例子来说明：

```js
POST /my_index/posts/_bulk
{ "index": { "_id": "1"              }}
{ "tags" : ["search"]                }  
{ "index": { "_id": "2"              }}
{ "tags" : ["search", "open_source"] }  
{ "index": { "_id": "3"              }}
{ "other_field" : "some data"        }  
{ "index": { "_id": "4"              }}
{ "tags" : null                      }  
{ "index": { "_id": "5"              }}
{ "tags" : ["search", null]          }  
```
>  ![img](assets/1.png)  `tags` 字段有 1 个值。  

>  ![img](assets/2.png)  `tags` 字段有 2 个值。  

>  ![img](assets/3.png)  `tags` 字段缺失。   

>  ![img](assets/4.png)  `tags` 字段被置为 `null` 。  

>  ![img](assets/5.png)  `tags` 字段有 1 个值和 1 个 `null` 。  

以上文档集合中 `tags` 字段对应的倒排索引如下：

| Token         | DocIDs      |
| ------------- | ----------- |
| `open_source` | `2`         |
| `search`      | `1`,`2`,`5` |

我们的目标是找到那些被设置过标签字段的文档，并不关心标签的具体内容。只要它存在于文档中即可，用 SQL 的话就是用 `IS NOT NULL` 非空进行查询：

```sql
SELECT tags
FROM   posts
WHERE  tags IS NOT NULL
```

在 Elasticsearch 中，使用 `exists` 查询的方式如下：

```js
GET /my_index/posts/_search
{
    "query" : {
        "constant_score" : {
            "filter" : {
                "exists" : { "field" : "tags" }
            }
        }
    }
}
```

这个查询返回 3 个文档：

```json
"hits" : [
    {
      "_id" :     "1",
      "_score" :  1.0,
      "_source" : { "tags" : ["search"] }
    },
    {
      "_id" :     "5",
      "_score" :  1.0,
      "_source" : { "tags" : ["search", null] } 
    },
    {
      "_id" :     "2",
      "_score" :  1.0,
      "_source" : { "tags" : ["search", "open source"] }
    }
]
```
>  ![img](assets/1.png)  尽管文档 5 有 `null` 值，但它仍会被命中返回。字段之所以存在，是因为标签有实际值（ `search` ）可以被索引，所以 `null` 对过滤不会产生任何影响。  

显而易见，只要 `tags` 字段存在项（term）的文档都会命中并作为结果返回，只有 3 和 4 两个文档被排除。

**缺失查询**

这个 `missing` 查询本质上与 `exists` 恰好相反： 它返回某个特定 _无_ 值字段的文档，与以下 SQL 表达的意思类似：

```sql
SELECT tags
FROM   posts
WHERE  tags IS NULL
```

我们将前面例子中 `exists` 查询换成 `missing` 查询：

```js
GET /my_index/posts/_search
{
    "query" : {
        "constant_score" : {
            "filter": {
                "missing" : { "field" : "tags" }
            }
        }
    }
}
```

按照期望的那样，我们得到 3 和 4 两个文档（这两个文档的 `tags` 字段没有实际值）：

```json
"hits" : [
    {
      "_id" :     "3",
      "_score" :  1.0,
      "_source" : { "other_field" : "some data" }
    },
    {
      "_id" :     "4",
      "_score" :  1.0,
      "_source" : { "tags" : null }
    }
]
```

---

>  **当 null 的意思是 null**
>  
>  有时候我们需要区分一个字段是没有值，还是它已被显式的设置成了 `null` 。在之前例子中，我们看到的默认的行为是无法做到这点的；数据被丢失了。不过幸运的是，我们可以选择将显式的 `null` 值替换成我们指定 *占位符（placeholder）* 。
>  
>  在为字符串（string）、数字（numeric）、布尔值（Boolean）或日期（date）字段指定映射时，同样可以为之设置 `null_value` 空值，用以处理显式 `null` 值的情况。不过即使如此，还是会将一个没有值的字段从倒排索引中排除。
>  
>  当选择合适的 `null_value` 空值的时候，需要保证以下几点：
>  
>  - 它会匹配字段的类型，我们不能为一个 `date` 日期字段设置字符串类型的 `null_value` 。
>  - 它必须与普通值不一样，这可以避免把实际值当成 `null` 空的情况。

---

**对象上的存在与缺失**

不仅可以过滤核心类型， `exists` and `missing` 查询 还可以处理一个对象的内部字段。以下面文档为例：

```js
{
   "name" : {
      "first" : "John",
      "last" :  "Smith"
   }
}
```

我们不仅可以检查 `name.first` 和 `name.last` 的存在性，也可以检查 `name` ，不过在 [映射](https://www.elastic.co/guide/cn/elasticsearch/guide/current/mapping.html) 中，如上对象的内部是个扁平的字段与值（field-value）的简单键值结构，类似下面这样：

```js
{
   "name.first" : "John",
   "name.last"  : "Smith"
}
```

那么我们如何用 `exists` 或 `missing` 查询 `name` 字段呢？ `name` 字段并不真实存在于倒排索引中。

原因是当我们执行下面这个过滤的时候：

```js
{
    "exists" : { "field" : "name" }
}
```

实际执行的是：

```js
{
    "bool": {
        "should": [
            { "exists": { "field": "name.first" }},
            { "exists": { "field": "name.last" }}
        ]
    }
}
```

这也就意味着，如果 `first` 和 `last` 都是空，那么 `name` 这个命名空间才会被认为不存在。



### 关于缓存

在本章前面（[过滤器的内部操作](https://www.elastic.co/guide/cn/elasticsearch/guide/current/_finding_exact_values.html#_internal_filter_operation)）中，我们已经简单介绍了过滤器是如何计算的。 其核心实际是采用一个 bitset 记录与过滤器匹配的文档。Elasticsearch 积极地把这些 bitset 缓存起来以备随后使用。一旦缓存成功，bitset 可以复用 *任何* 已使用过的相同过滤器，而无需再次计算整个过滤器。

这些 bitsets 缓存是“智能”的：它们以增量方式更新。当我们索引新文档时，只需将那些新文档加入已有 bitset，而不是对整个缓存一遍又一遍的重复计算。和系统其他部分一样，过滤器是实时的，我们无需担心缓存过期问题。

**独立的过滤器缓存**

属于一个查询组件的 bitsets 是独立于它所属搜索请求其他部分的。这就意味着，一旦被缓存，一个查询可以被用作多个搜索请求。bitsets 并不依赖于它所存在的查询上下文。这样使得缓存可以加速查询中经常使用的部分，从而降低较少、易变的部分所带来的消耗。

同样，如果单个请求重用相同的非评分查询，它缓存的 bitset 可以被单个搜索里的所有实例所重用。

让我们看看下面例子中的查询，它查找满足以下任意一个条件的电子邮件：

- 在收件箱中，且没有被读过的
- *不在* 收件箱中，但被标注重要的

```js
GET /inbox/emails/_search
{
  "query": {
      "constant_score": {
          "filter": {
              "bool": {
                 "should": [
                    { "bool": {
                          "must": [
                             { "term": { "folder": "inbox" }}, <1>
                             { "term": { "read": false }}
                          ]
                    }},
                    { "bool": {
                          "must_not": {
                             "term": { "folder": "inbox" } <2>
                          },
                          "must": {
                             "term": { "important": true }
                          }
                    }}
                 ]
              }
            }
        }
    }
}
```
>  ![img](assets/1.png) ![img](assets/2.png)  两个过滤器是相同的，所以会使用同一 bitset 。    

尽管其中一个收件箱的条件是 `must` 语句，另一个是 `must_not` 语句，但他们两者是完全相同的。这意味着在第一个语句执行后， bitset 就会被计算然后缓存起来供另一个使用。当再次执行这个查询时，收件箱的这个过滤器已经被缓存了，所以两个语句都会使用已缓存的 bitset 。

这点与查询表达式（query DSL）的可组合性结合得很好。它易被移动到表达式的任何地方，或者在同一查询中的多个位置复用。这不仅能方便开发者，而且对提升性能有直接的益处。

**自动缓存行为**

在 Elasticsearch 的较早版本中，默认的行为是缓存一切可以缓存的对象。这也通常意味着系统缓存 bitsets 太富侵略性，从而因为清理缓存带来性能压力。不仅如此，尽管很多过滤器都很容易被评价，但本质上是慢于缓存的（以及从缓存中复用）。缓存这些过滤器的意义不大，因为可以简单地再次执行过滤器。

检查一个倒排是非常快的，然后绝大多数查询组件却很少使用它。例如 `term` 过滤字段 `"user_id"` ：如果有上百万的用户，每个具体的用户 ID 出现的概率都很小。那么为这个过滤器缓存 bitsets 就不是很合算，因为缓存的结果很可能在重用之前就被剔除了。

这种缓存的扰动对性能有着严重的影响。更严重的是，它让开发者难以区分有良好表现的缓存以及无用缓存。

为了解决问题，Elasticsearch 会基于使用频次自动缓存查询。如果一个非评分查询在最近的 256 次查询中被使用过（次数取决于查询类型），那么这个查询就会作为缓存的候选。但是，并不是所有的片段都能保证缓存 bitset 。只有那些文档数量超过 10,000 （或超过总文档数量的 3% )才会缓存 bitset 。因为小的片段可以很快的进行搜索和合并，这里缓存的意义不大。

一旦缓存了，非评分计算的 bitset 会一直驻留在缓存中直到它被剔除。剔除规则是基于 LRU 的：一旦缓存满了，最近最少使用的过滤器会被剔除。



## 全文搜索

我们已经介绍了搜索结构化数据的简单应用示例，现在来探寻 *全文搜索（full-text search）* ：怎样在全文字段中搜索到最相关的文档。

全文搜索两个最重要的方面是：

- 相关性（Relevance）

  它是评价查询与其结果间的相关程度，并根据这种相关程度对结果排名的一种能力，这种计算方式可以是 TF/IDF 方法（参见 [相关性的介绍](https://www.elastic.co/guide/cn/elasticsearch/guide/current/relevance-intro.html)）、地理位置邻近、模糊相似，或其他的某些算法。

- 分析（Analysis）

  它是将文本块转换为有区别的、规范化的 token 的一个过程，（参见 [分析的介绍](https://www.elastic.co/guide/cn/elasticsearch/guide/current/analysis-intro.html)） 目的是为了（a）创建倒排索引以及（b）查询倒排索引。

一旦谈论相关性或分析这两个方面的问题时，我们所处的语境是关于查询的而不是过滤。



### 基于词项与基于全文

所有查询会或多或少的执行相关度计算，但不是所有查询都有分析阶段。 和一些特殊的完全不会对文本进行操作的查询（如 `bool` 或 `function_score` ）不同，文本查询可以划分成两大家族：

- 基于词项的查询

  如 `term` 或 `fuzzy` 这样的底层查询不需要分析阶段，它们对单个词项进行操作。用 `term` 查询词项 `Foo` 只要在倒排索引中查找 *准确词项* ，并且用 TF/IDF 算法为每个包含该词项的文档计算相关度评分 `_score` 。记住 `term` 查询只对倒排索引的词项精确匹配，这点很重要，它不会对词的多样性进行处理（如， `foo`或 `FOO` ）。这里，无须考虑词项是如何存入索引的。如果是将 `["Foo","Bar"]` 索引存入一个不分析的（ `not_analyzed` ）包含精确值的字段，或者将 `Foo Bar` 索引到一个带有 `whitespace` 空格分析器的字段，两者的结果都会是在倒排索引中有 `Foo` 和 `Bar` 这两个词。

- 基于全文的查询

  像 `match` 或 `query_string` 这样的查询是高层查询，它们了解字段映射的信息：如果查询 `日期（date）` 或 `整数（integer）` 字段，它们会将查询字符串分别作为日期或整数对待。如果查询一个（ `not_analyzed` ）未分析的精确值字符串字段， 它们会将整个查询字符串作为单个词项对待。但如果要查询一个（ `analyzed` ）已分析的全文字段， 它们会先将查询字符串传递到一个合适的分析器，然后生成一个供查询的词项列表。一旦组成了词项列表，这个查询会对每个词项逐一执行底层的查询，再将结果合并，然后为每个文档生成一个最终的相关度评分。我们将会在随后章节中详细讨论这个过程。

我们很少直接使用基于词项的搜索，通常情况下都是对全文进行查询，而非单个词项，这只需要简单的执行一个高层全文查询（进而在高层查询内部会以基于词项的底层查询完成搜索）。
>  ![注意](assets/note.png)  当我们想要查询一个具有精确值的 `not_analyzed` 未分析字段之前， 需要考虑，是否真的采用评分查询，或者非评分查询会更好。
>  
>  单词项查询通常可以用是、非这种二元问题表示，所以更适合用过滤， 而且这样做可以有效利用[缓存](https://www.elastic.co/guide/cn/elasticsearch/guide/current/filter-caching.html)：
>  
>  ```js
>  GET /_search
>  {
>      "query": {
>          "constant_score": {
>              "filter": {
>                  "term": { "gender": "female" }
>              }
>          }
>      }
>  }
>  ```



### 匹配查询

匹配查询 `match` 是个 *核心* 查询。无论需要查询什么字段， `match` 查询都应该会是首选的查询方式。 它是一个高级 *全文查询* ，这表示它既能处理全文字段，又能处理精确字段。

这就是说， `match` 查询主要的应用场景就是进行全文搜索，我们以下面一个简单例子来说明全文搜索是如何工作的：

**索引一些数据**

首先，我们使用 [`bulk` API](https://www.elastic.co/guide/cn/elasticsearch/guide/current/bulk.html) 创建一些新的文档和索引：

```js
DELETE /my_index   <1>

PUT /my_index
{ "settings": { "number_of_shards": 1 }}   <2>

POST /my_index/my_type/_bulk
{ "index": { "_id": 1 }}
{ "title": "The quick brown fox" }
{ "index": { "_id": 2 }}
{ "title": "The quick brown fox jumps over the lazy dog" }
{ "index": { "_id": 3 }}
{ "title": "The quick brown fox jumps over the quick dog" }
{ "index": { "_id": 4 }}
{ "title": "Brown fox brown dog" }
```
>  ![img](assets/1.png)  删除已有的索引。  
>  
>  ![img](assets/2.png)  稍后，我们会在 [被破坏的相关性！](https://www.elastic.co/guide/cn/elasticsearch/guide/current/relevance-is-broken.html) 中解释只为这个索引分配一个主分片的原因。   



**单个词查询**

我们用第一个示例来解释使用 `match` 查询搜索全文字段中的单个词：

```js
GET /my_index/my_type/_search
{
    "query": {
        "match": {
            "title": "QUICK!"
        }
    }
}
```

Elasticsearch 执行上面这个 `match` 查询的步骤是：

1. *检查字段类型* 。

   标题 `title` 字段是一个 `string` 类型（ `analyzed` ）已分析的全文字段，这意味着查询字符串本身也应该被分析。

2. *分析查询字符串* 。

   将查询的字符串 `QUICK!` 传入标准分析器中，输出的结果是单个项 `quick` 。因为只有一个单词项，所以 `match` 查询执行的是单个底层 `term` 查询。

3. *查找匹配文档* 。

   用 `term` 查询在倒排索引中查找 `quick` 然后获取一组包含该项的文档，本例的结果是文档：1、2 和 3 。

4. *为每个文档评分* 。

   用 `term` 查询计算每个文档相关度评分 `_score` ，这是种将 词频（term frequency，即词 `quick` 在相关文档的 `title` 字段中出现的频率）和反向文档频率（inverse document frequency，即词 `quick` 在所有文档的 `title` 字段中出现的频率），以及字段的长度（即字段越短相关度越高）相结合的计算方式。参见 [相关性的介绍](https://www.elastic.co/guide/cn/elasticsearch/guide/current/relevance-intro.html) 。

这个过程给我们以下（经缩减）结果：

```js
"hits": [
 {
    "_id":      "1",
    "_score":   0.5,         <1>
    "_source": {
       "title": "The quick brown fox"
    }
 },
 {
    "_id":      "3",
    "_score":   0.44194174,   <2>
    "_source": {
       "title": "The quick brown fox jumps over the quick dog"
    }
 },
 {
    "_id":      "2",
    "_score":   0.3125,       <3>
    "_source": {
       "title": "The quick brown fox jumps over the lazy dog"
    }
 }
]
```
>  ![img](assets/1.png)   文档 1 最相关，因为它的 `title` 字段更短，即 `quick` 占据内容的一大部分。  
>  
>  ![img](assets/2.png)  ![img](assets/3.png)  文档 3 比 文档 2 更具相关性，因为在文档 3 中 `quick` 出现了两次。  



### 多词查询

如果我们一次只能搜索一个词，那么全文搜索就会不太灵活，幸运的是 `match` 查询让多词查询变得简单：

```js
GET /my_index/my_type/_search
{
    "query": {
        "match": {
            "title": "BROWN DOG!"
        }
    }
}
```

上面这个查询返回所有四个文档：

```js
{
  "hits": [
     {
        "_id":      "4",
        "_score":   0.73185337,              <1>
        "_source": {
           "title": "Brown fox brown dog"
        }
     },
     {
        "_id":      "2", 
        "_score":   0.47486103,              <2>
        "_source": {
           "title": "The quick brown fox jumps over the lazy dog"
        }
     },
     {
        "_id":      "3",
        "_score":   0.47486103,              <3>
        "_source": {
           "title": "The quick brown fox jumps over the quick dog"
        }
     },
     {
        "_id":      "1",
        "_score":   0.11914785,              <4>
        "_source": {
           "title": "The quick brown fox"
        }
     }
  ]
}
```
>  ![img](assets/1.png)  文档 4 最相关，因为它包含词 `"brown"` 两次以及 `"dog"` 一次。   
>  
>  ![img](assets/2.png) ![img](assets/3.png)  文档 2、3 同时包含 `brown` 和 `dog` 各一次，而且它们 `title` 字段的长度相同，所以具有相同的评分。 
>  
>  ![img](assets/4.png)  文档 1 也能匹配，尽管它只有 `brown` 没有 `dog` 。            |

因为 `match` 查询必须查找两个词（ `["brown","dog"]` ），它在内部实际上先执行两次 `term` 查询，然后将两次查询的结果合并作为最终结果输出。为了做到这点，它将两个 `term` 查询包入一个 `bool` 查询中，详细信息见 [布尔查询](https://www.elastic.co/guide/cn/elasticsearch/guide/current/bool-query.html)。

以上示例告诉我们一个重要信息：即任何文档只要 `title` 字段里包含 *指定词项中的至少一个词* 就能匹配，被匹配的词项越多，文档就越相关。

**提高精度**

用 *任意* 查询词项匹配文档可能会导致结果中出现不相关的长尾。 这是种散弹式搜索。可能我们只想搜索包含 *所有* 词项的文档，也就是说，不去匹配 `brown OR dog` ，而通过匹配 `brown AND dog` 找到所有文档。

`match` 查询还可以接受 `operator` 操作符作为输入参数，默认情况下该操作符是 `or` 。我们可以将它修改成 `and` 让所有指定词项都必须匹配：

```js
GET /my_index/my_type/_search
{
    "query": {
        "match": {
            "title": {      <1>
                "query":    "BROWN DOG!",
                "operator": "and"
            }
        }
    }
}
```
>  ![img](assets/1.png)  `match` 查询的结构需要做稍许调整才能使用 `operator` 操作符参数。 

这个查询可以把文档 1 排除在外，因为它只包含两个词项中的一个。

**控制精度**

在 *所有* 与 *任意* 间二选一有点过于非黑即白。 如果用户给定 5 个查询词项，想查找只包含其中 4 个的文档，该如何处理？将 `operator` 操作符参数设置成 `and` 只会将此文档排除。

有时候这正是我们期望的，但在全文搜索的大多数应用场景下，我们既想包含那些可能相关的文档，同时又排除那些不太相关的。换句话说，我们想要处于中间某种结果。

`match` 查询支持 `minimum_should_match` 最小匹配参数， 这让我们可以指定必须匹配的词项数用来表示一个文档是否相关。我们可以将其设置为某个具体数字，更常用的做法是将其设置为一个百分数，因为我们无法控制用户搜索时输入的单词数量：

```js
GET /my_index/my_type/_search
{
  "query": {
    "match": {
      "title": {
        "query":                "quick brown dog",
        "minimum_should_match": "75%"
      }
    }
  }
}
```

当给定百分比的时候， `minimum_should_match` 会做合适的事情：在之前三词项的示例中， `75%` 会自动被截断成 `66.6%` ，即三个里面两个词。无论这个值设置成什么，至少包含一个词项的文档才会被认为是匹配的。
>  ![注意](assets/note.png)  参数 `minimum_should_match` 的设置非常灵活，可以根据用户输入词项的数目应用不同的规则。完整的信息参考文档<https://www.elastic.co/guide/en/elasticsearch/reference/5.6/query-dsl-minimum-should-match.html#query-dsl-minimum-should-match>  

为了完全理解 `match` 是如何处理多词查询的，我们就需要查看如何使用 `bool` 查询将多个查询条件组合在一起。



### 组合查询

在 [组合过滤器](https://www.elastic.co/guide/cn/elasticsearch/guide/current/combining-filters.html) 中，我们讨论过如何使用 `bool` 过滤器通过 `and` 、 `or` 和 `not` 逻辑组合将多个过滤器进行组合。在查询中， `bool` 查询有类似的功能，只有一个重要的区别。

过滤器做二元判断：文档是否应该出现在结果中？但查询更精妙，它除了决定一个文档是否应该被包括在结果中，还会计算文档的 *相关程度* 。

与过滤器一样， `bool` 查询也可以接受 `must` 、 `must_not` 和 `should` 参数下的多个查询语句。比如：

```js
GET /my_index/my_type/_search
{
  "query": {
    "bool": {
      "must":     { "match": { "title": "quick" }},
      "must_not": { "match": { "title": "lazy"  }},
      "should": [
                  { "match": { "title": "brown" }},
                  { "match": { "title": "dog"   }}
      ]
    }
  }
}
```

以上的查询结果返回 `title` 字段包含词项 `quick` 但不包含 `lazy` 的任意文档。目前为止，这与 `bool` 过滤器的工作方式非常相似。

区别就在于两个 `should` 语句，也就是说：一个文档不必包含 `brown` 或 `dog` 这两个词项，但如果一旦包含，我们就认为它们 *更相关* ：

```js
{
  "hits": [
     {
        "_id":      "3",
        "_score":   0.70134366,  <1>
        "_source": {
           "title": "The quick brown fox jumps over the quick dog"
        }
     },
     {
        "_id":      "1",
        "_score":   0.3312608,
        "_source": {
           "title": "The quick brown fox"
        }
     }
  ]
}
```
>  ![img](assets/1.png)  文档 3 会比文档 1 有更高评分是因为它同时包含 `brown` 和 `dog` 。 

**评分计算**

`bool` 查询会为每个文档计算相关度评分 `_score` ， 再将所有匹配的 `must` 和 `should` 语句的分数 `_score`求和，最后除以 `must` 和 `should` 语句的总数。

`must_not` 语句不会影响评分； 它的作用只是将不相关的文档排除。

**控制精度**

所有 `must` 语句必须匹配，所有 `must_not` 语句都必须不匹配，但有多少 `should` 语句应该匹配呢？ 默认情况下，没有 `should` 语句是必须匹配的，只有一个例外：那就是当没有 `must` 语句的时候，至少有一个 `should` 语句必须匹配。

就像我们能控制 [`match` 查询的精度](https://www.elastic.co/guide/cn/elasticsearch/guide/current/match-multi-word.html#match-precision) 一样，我们可以通过 `minimum_should_match` 参数控制需要匹配的 `should` 语句的数量， 它既可以是一个绝对的数字，又可以是个百分比：

```js
GET /my_index/my_type/_search
{
  "query": {
    "bool": {
      "should": [
        { "match": { "title": "brown" }},
        { "match": { "title": "fox"   }},
        { "match": { "title": "dog"   }}
      ],
      "minimum_should_match": 2   <1>
    }
  }
}
```
>  ![img](assets/1.png)]  这也可以用百分比表示。 

这个查询结果会将所有满足以下条件的文档返回： `title` 字段包含 `"brown" AND "fox"` 、 `"brown" AND "dog"` 或 `"fox" AND "dog"` 。如果有文档包含所有三个条件，它会比只包含两个的文档更相关。



### 如何使用布尔匹配

目前为止，可能已经意识到[多词 `match` 查询](https://www.elastic.co/guide/cn/elasticsearch/guide/current/match-multi-word.html)只是简单地将生成的 `term` 查询包裹 在一个 `bool` 查询中。如果使用默认的 `or` 操作符，每个 `term` 查询都被当作 `should` 语句，这样就要求必须至少匹配一条语句。以下两个查询是等价的：

```js
{
    "match": { "title": "brown fox"}
}
{
  "bool": {
    "should": [
      { "term": { "title": "brown" }},
      { "term": { "title": "fox"   }}
    ]
  }
}
```

如果使用 `and` 操作符，所有的 `term` 查询都被当作 `must` 语句，所以 *所有（all）* 语句都必须匹配。以下两个查询是等价的：

```js
{
    "match": {
        "title": {
            "query":    "brown fox",
            "operator": "and"
        }
    }
}
{
  "bool": {
    "must": [
      { "term": { "title": "brown" }},
      { "term": { "title": "fox"   }}
    ]
  }
}
```

如果指定参数 `minimum_should_match` ，它可以通过 `bool` 查询直接传递，使以下两个查询等价：

```js
{
    "match": {
        "title": {
            "query":                "quick brown fox",
            "minimum_should_match": "75%"
        }
    }
}
{
  "bool": {
    "should": [
      { "term": { "title": "brown" }},
      { "term": { "title": "fox"   }},
      { "term": { "title": "quick" }}
    ],
    "minimum_should_match": 2   <1>
  }
}
```
>  ![img](assets/1.png)  因为只有三条语句，`match` 查询的参数 `minimum_should_match` 值 75% 会被截断成 `2` 。即三条 `should` 语句中至少有两条必须匹配。  

当然，我们通常将这些查询用 `match` 查询来表示，但是如果了解 `match` 内部的工作原理，我们就能根据自己的需要来控制查询过程。有些时候单个 `match` 查询无法满足需求，比如为某些查询条件分配更高的权重。我们会在下一小节中看到这个例子。



### 查询语句提升权重

当然 `bool` 查询不仅限于组合简单的单个词 `match` 查询， 它可以组合任意其他的查询，以及其他 `bool` 查询。 普遍的用法是通过汇总多个独立查询的分数，从而达到为每个文档微调其相关度评分 `_score` 的目的。

假设想要查询关于 “full-text search（全文搜索）” 的文档， 但我们希望为提及 “Elasticsearch” 或 “Lucene” 的文档给予更高的 *权重* ，这里 *更高权重* 是指如果文档中出现 “Elasticsearch” 或 “Lucene” ，它们会比没有的出现这些词的文档获得更高的相关度评分 `_score` ，也就是说，它们会出现在结果集的更上面。

一个简单的 `bool` *查询* 允许我们写出如下这种非常复杂的逻辑：

```js
GET /_search
{
    "query": {
        "bool": {
            "must": {
                "match": {
                    "content": {  <1>
                        "query":    "full text search",
                        "operator": "and"
                    }
                }
            },
            "should": [   <2>
                { "match": { "content": "Elasticsearch" }},
                { "match": { "content": "Lucene"        }}
            ]
        }
    }
}
```
>  ![img](assets/1.png)  `content` 字段必须包含 `full` 、 `text` 和 `search` 所有三个词。 
>  
>  ![img](assets/2.png)  如果 `content` 字段也包含 `Elasticsearch` 或 `Lucene` ，文档会获得更高的评分 `_score` 。 

`should` 语句匹配得越多表示文档的相关度越高。目前为止还挺好。

但是如果我们想让包含 `Lucene` 的有更高的权重，并且包含 `Elasticsearch` 的语句比 `Lucene` 的权重更高，该如何处理?

我们可以通过指定 `boost` 来控制任何查询语句的相对的权重， `boost` 的默认值为 `1` ，大于 `1` 会提升一个语句的相对权重。所以下面重写之前的查询：

```js
GET /_search
{
    "query": {
        "bool": {
            "must": {
                "match": {  <1>
                    "content": {
                        "query":    "full text search",
                        "operator": "and"
                    }
                }
            },
            "should": [
                { "match": {
                    "content": {
                        "query": "Elasticsearch",
                        "boost": 3   <2>
                    }
                }},
                { "match": {
                    "content": {
                        "query": "Lucene",
                        "boost": 2   <3>
                    }
                }}
            ]
        }
    }
}
```
>  ![img](assets/1.png)  这些语句使用默认的 `boost` 值 `1` 。   
>
>  ![img](assets/2.png)  这条语句更为重要，因为它有最高的 `boost` 值。  
>  
>  ![img](assets/3.png) 这条语句比使用默认值的更重要，但它的重要性不及 `Elasticsearch` 语句。 
>  
>  ![注意](assets/note.png)  `boost` 参数被用来提升一个语句的相对权重（ `boost` 值大于 `1` ）或降低相对权重（ `boost`值处于 `0` 到 `1` 之间），但是这种提升或降低并不是线性的，换句话说，如果一个 `boost` 值为 `2` ，并不能获得两倍的评分 `_score` 。
>
> 相反，新的评分 `_score` 会在应用权重提升之后被 *归一化* ，每种类型的查询都有自己的归一算法，细节超出了本书的范围，所以不作介绍。简单的说，更高的 `boost` 值为我们带来更高的评分 `_score` 。
>
> 如果不基于 TF/IDF 要实现自己的评分模型，我们就需要对权重提升的过程能有更多控制，可以使用 [`function_score` 查询](https://www.elastic.co/guide/cn/elasticsearch/guide/current/function-score-query.html)操纵一个文档的权重提升方式而跳过归一化这一步骤。  

更多的组合查询方式会在下章[多字段搜索](https://www.elastic.co/guide/cn/elasticsearch/guide/current/multi-field-search.html)中介绍，但在此之前，让我们先看另外一个重要的查询特性：文本分析（text analysis）。



### 控制分析

查询只能查找倒排索引表中真实存在的项， 所以保证文档在索引时与查询字符串在搜索时应用相同的分析过程非常重要，这样查询的项才能够匹配倒排索引中的项。

尽管是在说 *文档* ，不过分析器可以由每个字段决定。 每个字段都可以有不同的分析器，既可以通过配置为字段指定分析器，也可以使用更高层的类型（type）、索引（index）或节点（node）的默认配置。在索引时，一个字段值是根据配置或默认分析器分析的。

例如为 `my_index` 新增一个字段：

```js
PUT /my_index/_mapping/my_type
{
    "my_type": {
        "properties": {
            "english_title": {
                "type":     "string",
                "analyzer": "english"
            }
        }
    }
}
```

现在我们就可以通过使用 `analyze` API 来分析单词 `Foxes` ，进而比较 `english_title` 字段和 `title` 字段在索引时的分析结果：

```js
GET /my_index/_analyze
{
  "field": "my_type.title",   <1>
  "text": "Foxes"
}

GET /my_index/_analyze
{
  "field": "my_type.english_title",   <2>
  "text": "Foxes"
}
```
>  ![img](assets/1.png)  字段 `title` ，使用默认的 `standard` 标准分析器，返回词项 `foxes` 。 
>  
>  ![img](assets/2.png)  字段 `english_title` ，使用 `english` 英语分析器，返回词项 `fox` 。 

这意味着，如果使用底层 `term` 查询精确项 `fox` 时， `english_title` 字段会匹配但 `title` 字段不会。

如同 `match` 查询这样的高层查询知道字段映射的关系，能为每个被查询的字段应用正确的分析器。 可以使用 `validate-query` API 查看这个行为：

```js
GET /my_index/my_type/_validate/query?explain
{
    "query": {
        "bool": {
            "should": [
                { "match": { "title":         "Foxes"}},
                { "match": { "english_title": "Foxes"}}
            ]
        }
    }
}
```

返回语句的 `explanation` 结果：

```
(title:foxes english_title:fox)
```

`match` 查询为每个字段使用合适的分析器，以保证它在寻找每个项时都为该字段使用正确的格式。

**默认分析器**

虽然我们可以在字段层级指定分析器， 但是如果该层级没有指定任何的分析器，那么我们如何能确定这个字段使用的是哪个分析器呢？

分析器可以从三个层面进行定义：按字段（per-field）、按索引（per-index）或全局缺省（global default）。Elasticsearch 会按照以下顺序依次处理，直到它找到能够使用的分析器。索引时的顺序如下：

- 字段映射里定义的 `analyzer` ，否则
- 索引设置中名为 `default` 的分析器，默认为
- `standard` 标准分析器

在搜索时，顺序有些许不同：

- 查询自己定义的 `analyzer` ，否则
- 字段映射里定义的 `analyzer` ，否则
- 索引设置中名为 `default` 的分析器，默认为
- `standard` 标准分析器

有时，在索引时和搜索时使用不同的分析器是合理的。 我们可能要想为同义词建索引（例如，所有 `quick`出现的地方，同时也为 `fast` 、 `rapid` 和 `speedy` 创建索引）。但在搜索时，我们不需要搜索所有的同义词，取而代之的是寻找用户输入的单词是否是 `quick` 、 `fast` 、 `rapid` 或 `speedy` 。

为了区分，Elasticsearch 也支持一个可选的 `search_analyzer` 映射，它仅会应用于搜索时（ `analyzer` 还用于索引时）。还有一个等价的 `default_search` 映射，用以指定索引层的默认配置。

如果考虑到这些额外参数，一个搜索时的 *完整* 顺序会是下面这样：

- 查询自己定义的 `analyzer` ，否则
- 字段映射里定义的 `search_analyzer` ，否则
- 字段映射里定义的 `analyzer` ，否则
- 索引设置中名为 `default_search` 的分析器，默认为
- 索引设置中名为 `default` 的分析器，默认为
- `standard` 标准分析器

**分析器配置实践**

就可以配置分析器地方的数量而言是十分惊人的， 但是实际非常简单。

**保持简单**

多数情况下，会提前知道文档会包括哪些字段。最简单的途径就是在创建索引或者增加类型映射时，为每个全文字段设置分析器。这种方式尽管有点麻烦，但是它让我们可以清楚的看到每个字段每个分析器是如何设置的。

通常，多数字符串字段都是 `not_analyzed` 精确值字段，比如标签（tag）或枚举（enum），而且更多的全文字段会使用默认的 `standard` 分析器或 `english` 或其他某种语言的分析器。这样只需要为少数一两个字段指定自定义分析：或许标题 `title` 字段需要以支持 *输入即查找（find-as-you-type）* 的方式进行索引。

可以在索引级别设置中，为绝大部分的字段设置你想指定的 `default` 默认分析器。然后在字段级别设置中，对某一两个字段配置需要指定的分析器。
>  ![注意](assets/note.png)  对于和时间相关的日志数据，通常的做法是每天自行创建索引，由于这种方式不是从头创建的索引，仍然可以用 [索引模板（Index Template）](https://www.elastic.co/guide/en/elasticsearch/reference/5.6/indices-templates.html) 为新建的索引指定配置和映射。 


<a name="被破坏的相关度"></a>
### 被破坏的相关度！

在讨论更复杂的 [多字段搜索](https://www.elastic.co/guide/cn/elasticsearch/guide/current/multi-field-search.html) 之前，让我们先快速解释一下为什么只在主分片上 [创建测试索引](https://www.elastic.co/guide/cn/elasticsearch/guide/current/match-query.html#match-test-data) 。

用户会时不时的抱怨无法按相关度排序并提供简短的重现步骤： 用户索引了一些文档，运行一个简单的查询，然后发现明显低相关度的结果出现在高相关度结果之上。

为了理解为什么会这样，可以设想，我们在两个主分片上创建了索引和总共 10 个文档，其中 6 个文档有单词 `foo` 。可能是分片 1 有其中 3 个 `foo` 文档，而分片 2 有其中另外 3 个文档，换句话说，所有文档是均匀分布存储的。

在 [什么是相关度？](https://www.elastic.co/guide/cn/elasticsearch/guide/current/relevance-intro.html)中，我们描述了 Elasticsearch 默认使用的相似度算法，这个算法叫做 *词频/逆向文档频率* 或 TF/IDF 。词频是计算某个词在当前被查询文档里某个字段中出现的频率，出现的频率越高，文档越相关。 *逆向文档频率* 将 *某个词在索引内所有文档出现的百分数* 考虑在内，出现的频率越高，它的权重就越低。

但是由于性能原因， Elasticsearch 不会计算索引内所有文档的 IDF 。 相反，每个分片会根据 *该分片* 内的所有文档计算一个本地 IDF 。

因为文档是均匀分布存储的，两个分片的 IDF 是相同的。相反，设想如果有 5 个 `foo` 文档存于分片 1 ，而第 6 个文档存于分片 2 ，在这种场景下， `foo` 在一个分片里非常普通（所以不那么重要），但是在另一个分片里非常出现很少（所以会显得更重要）。这些 IDF 之间的差异会导致不正确的结果。

在实际应用中，这并不是一个问题，本地和全局的 IDF 的差异会随着索引里文档数的增多渐渐消失，在真实世界的数据量下，局部的 IDF 会被迅速均化，所以上述问题并不是相关度被破坏所导致的，而是由于数据太少。

为了测试，我们可以通过两种方式解决这个问题。第一种是只在主分片上创建索引，正如 [`match` 查询](https://www.elastic.co/guide/cn/elasticsearch/guide/current/match-query.html) 里介绍的那样，如果只有一个分片，那么本地的 IDF *就是* 全局的 IDF。

第二个方式就是在搜索请求后添加 `?search_type=dfs_query_then_fetch` ， `dfs` 是指 *分布式频率搜索（Distributed Frequency Search）* ， 它告诉 Elasticsearch ，先分别获得每个分片本地的 IDF ，然后根据结果再计算整个索引的全局 IDF 。
>  ![提示](assets/tip.png)  不要在生产环境上使用 `dfs_query_then_fetch` 。完全没有必要。只要有足够的数据就能保证词频是均匀分布的。没有理由给每个查询额外加上 DFS 这步。 



## 多字段搜索

查询很少是简单一句话的 `match` 匹配查询。通常我们需要用相同或不同的字符串查询一个或多个字段，也就是说，需要对多个查询语句以及它们相关度评分进行合理的合并。

有时候或许我们正查找作者 Leo Tolstoy 写的一本名为 _War and Peace_（战争与和平）的书。或许我们正用 “minimum should match” （最少应该匹配）的方式在文档中对标题或页面内容进行搜索，或许我们正在搜索所有名字为 John Smith 的用户。

在本章，我们会介绍构造多语句搜索的工具及在特定场景下应该采用的解决方案。



### 多字符串查询

最简单的多字段查询可以将搜索项映射到具体的字段。 如果我们知道 *War and Peace* 是标题，Leo Tolstoy 是作者，很容易就能把两个条件用 `match` 语句表示， 并将它们用 [`bool` 查询](https://www.elastic.co/guide/cn/elasticsearch/guide/current/bool-query.html) 组合起来：

```js
GET /_search
{
  "query": {
    "bool": {
      "should": [
        { "match": { "title":  "War and Peace" }},
        { "match": { "author": "Leo Tolstoy"   }}
      ]
    }
  }
}
```

`bool` 查询采取 *more-matches-is-better* 匹配越多越好的方式，所以每条 `match` 语句的评分结果会被加在一起，从而为每个文档提供最终的分数 `_score` 。能与两条语句同时匹配的文档比只与一条语句匹配的文档得分要高。

当然，并不是只能使用 `match` 语句：可以用 `bool` 查询来包裹组合任意其他类型的查询， 甚至包括其他的 `bool` 查询。我们可以在上面的示例中添加一条语句来指定译者版本的偏好：

```js
GET /_search
{
  "query": {
    "bool": {
      "should": [
        { "match": { "title":  "War and Peace" }},
        { "match": { "author": "Leo Tolstoy"   }},
        { "bool":  {
          "should": [
            { "match": { "translator": "Constance Garnett" }},
            { "match": { "translator": "Louise Maude"      }}
          ]
        }}
      ]
    }
  }
}
```

为什么将译者条件语句放入另一个独立的 `bool` 查询中呢？所有的四个 `match` 查询都是 `should` 语句，所以为什么不将 translator 语句与其他如 title 、 author 这样的语句放在同一层呢？

答案在于评分的计算方式。 `bool` 查询运行每个 `match` 查询，再把评分加在一起，然后将结果与所有匹配的语句数量相乘，最后除以所有的语句数量。处于同一层的每条语句具有相同的权重。在前面这个例子中，包含 translator 语句的 `bool` 查询，只占总评分的三分之一。如果将 translator 语句与 title 和 author 两条语句放入同一层，那么 title 和 author 语句只贡献四分之一评分。

**语句的优先级**

前例中每条语句贡献三分之一评分的这种方式可能并不是我们想要的， 我们可能对 title 和 author 两条语句更感兴趣，这样就需要调整查询，使 title 和 author 语句相对来说更重要。

在武器库中，最容易使用的就是 `boost` 参数。为了提升 `title` 和 `author` 字段的权重， 为它们分配的 `boost` 值大于 `1` ：

```js
GET /_search
{
  "query": {
    "bool": {
      "should": [ 
        { "match": {                                <1>
            "title":  {
              "query": "War and Peace",
              "boost": 2
        }}},
        { "match": {                                <2>
            "author":  {
              "query": "Leo Tolstoy",
              "boost": 2
        }}},
        { "bool":  {                                <3>
            "should": [
              { "match": { "translator": "Constance Garnett" }},
              { "match": { "translator": "Louise Maude"      }}
            ]
        }}
      ]
    }
  }
}
```
>  ![img](assets/1.png) ![img](assets/2.png)  `title` 和 `author` 语句的 `boost` 值为 `2` 。 
>  
>  ![img](assets/3.png)  嵌套 `bool` 语句默认的 `boost` 值为 `1` 。     

要获取 `boost` 参数 “最佳” 值，较为简单的方式就是不断试错：设定 `boost` 值，运行测试查询，如此反复。 `boost` 值比较合理的区间处于 `1` 到 `10` 之间，当然也有可能是 `15` 。如果为 `boost` 指定比这更高的值，将不会对最终的评分结果产生更大影响，因为评分是被 [归一化的（normalized）](https://www.elastic.co/guide/cn/elasticsearch/guide/current/_boosting_query_clauses.html#boost-normalization) 。



### 单字符串查询

`bool` 查询是多语句查询的主干。 它的适用场景很多，特别是当需要将不同查询字符串映射到不同字段的时候。

问题在于，目前有些用户期望将所有的搜索项堆积到单个字段中，并期望应用程序能为他们提供正确的结果。有意思的是多字段搜索的表单通常被称为 *高级查询 （Advanced Search）* —— 只是因为它对用户而言是高级的，而多字段搜索的实现却非常简单。

对于多词（multiword）、多字段（multifield）查询来说，不存在简单的 *万能* 方案。为了获得最好结果，需要 *了解我们的数据* ，并了解如何使用合适的工具。

**了解我们的数据**

当用户输入了单个字符串查询的时候，通常会遇到以下三种情形：

- 最佳字段

  当搜索词语具体概念的时候，比如 “brown fox” ，词组比各自独立的单词更有意义。像 `title` 和 `body` 这样的字段，尽管它们之间是相关的，但同时又彼此相互竞争。文档在 *相同字段* 中包含的词越多越好，评分也来自于 *最匹配字段* 。

- 多数字段

  为了对相关度进行微调，常用的一个技术就是将相同的数据索引到不同的字段，它们各自具有独立的分析链。主字段可能包括它们的词源、同义词以及 *变音词* 或口音词，被用来匹配尽可能多的文档。相同的文本被索引到其他字段，以提供更精确的匹配。一个字段可以包括未经词干提取过的原词，另一个字段包括其他词源、口音，还有一个字段可以提供 [词语相似性](https://www.elastic.co/guide/cn/elasticsearch/guide/current/proximity-matching.html) 信息的瓦片词（shingles）。其他字段是作为匹配每个文档时提高相关度评分的 *信号* ， *匹配字段越多* 则越好。

- 混合字段

  对于某些实体，我们需要在多个字段中确定其信息，单个字段都只能作为整体的一部分：Person： `first_name` 和 `last_name` （人：名和姓）Book： `title` 、 `author` 和 `description` （书：标题、作者、描述）Address： `street` 、 `city` 、 `country` 和 `postcode` （地址：街道、市、国家和邮政编码）在这种情况下，我们希望在 *任何* 这些列出的字段中找到尽可能多的词，这有如在一个大字段中进行搜索，这个大字段包括了所有列出的字段。

上述所有都是多词、多字段查询，但每个具体查询都要求使用不同策略。本章后面的部分，我们会依次介绍每个策略。



### 最佳字段

假设有个网站允许用户搜索博客的内容， 以下面两篇博客内容文档为例：

```js
PUT /my_index/my_type/1
{
    "title": "Quick brown rabbits",
    "body":  "Brown rabbits are commonly seen."
}

PUT /my_index/my_type/2
{
    "title": "Keeping pets healthy",
    "body":  "My quick brown fox eats rabbits on a regular basis."
}
```

用户输入词组 “Brown fox” 然后点击搜索按钮。事先，我们并不知道用户的搜索项是会在 `title` 还是在 `body` 字段中被找到，但是，用户很有可能是想搜索相关的词组。用肉眼判断，文档 2 的匹配度更高，因为它同时包括要查找的两个词：

现在运行以下 `bool` 查询：

```js
{
    "query": {
        "bool": {
            "should": [
                { "match": { "title": "Brown fox" }},
                { "match": { "body":  "Brown fox" }}
            ]
        }
    }
}
```

但是我们发现查询的结果是文档 1 的评分更高：

```js
{
  "hits": [
     {
        "_id":      "1",
        "_score":   0.14809652,
        "_source": {
           "title": "Quick brown rabbits",
           "body":  "Brown rabbits are commonly seen."
        }
     },
     {
        "_id":      "2",
        "_score":   0.09256032,
        "_source": {
           "title": "Keeping pets healthy",
           "body":  "My quick brown fox eats rabbits on a regular basis."
        }
     }
  ]
}
```

为了理解导致这样的原因， 需要回想一下 `bool` 是如何计算评分的：

1. 它会执行 `should` 语句中的两个查询。
2. 加和两个查询的评分。
3. 乘以匹配语句的总数。
4. 除以所有语句总数（这里为：2）。

文档 1 的两个字段都包含 `brown` 这个词，所以两个 `match` 语句都能成功匹配并且有一个评分。文档 2 的 `body` 字段同时包含 `brown` 和 `fox` 这两个词，但 `title` 字段没有包含任何词。这样， `body` 查询结果中的高分，加上 `title` 查询中的 0 分，然后乘以二分之一，就得到比文档 1 更低的整体评分。

在本例中， `title` 和 `body` 字段是相互竞争的关系，所以就需要找到单个 *最佳匹配* 的字段。

如果不是简单将每个字段的评分结果加在一起，而是将 *最佳匹配* 字段的评分作为查询的整体评分，结果会怎样？这样返回的结果可能是： *同时* 包含 `brown` 和 `fox` 的单个字段比反复出现相同词语的多个不同字段有更高的相关度。

**dis_max 查询**

不使用 `bool` 查询，可以使用 `dis_max` 即分离 *最大化查询（Disjunction Max Query）* 。分离（Disjunction）的意思是 *或（or）* ，这与可以把结合（conjunction）理解成 *与（and）* 相对应。分离最大化查询（Disjunction Max Query）指的是： *将任何与任一查询匹配的文档作为结果返回，但只将最佳匹配的评分作为查询的评分结果返回* ：

```js
{
    "query": {
        "dis_max": {
            "queries": [
                { "match": { "title": "Brown fox" }},
                { "match": { "body":  "Brown fox" }}
            ]
        }
    }
}
```

得到我们想要的结果为：

```js
{
  "hits": [
     {
        "_id":      "2",
        "_score":   0.21509302,
        "_source": {
           "title": "Keeping pets healthy",
           "body":  "My quick brown fox eats rabbits on a regular basis."
        }
     },
     {
        "_id":      "1",
        "_score":   0.12713557,
        "_source": {
           "title": "Quick brown rabbits",
           "body":  "Brown rabbits are commonly seen."
        }
     }
  ]
}
```



### 最佳字段查询调优

当用户搜索 “quick pets” 时会发生什么呢？ 在前面的例子中，两个文档都包含词 `quick` ，但是只有文档 2 包含词 `pets` ，两个文档中都不具有同时包含 *两个词* 的 *相同字段* 。

如下，一个简单的 `dis_max` 查询会采用单个最佳匹配字段， 而忽略其他的匹配：

```js
{
    "query": {
        "dis_max": {
            "queries": [
                { "match": { "title": "Quick pets" }},
                { "match": { "body":  "Quick pets" }}
            ]
        }
    }
}
```



```js
{
  "hits": [
     {
        "_id": "1",
        "_score": 0.12713557,                         <1>
        "_source": {
           "title": "Quick brown rabbits",
           "body": "Brown rabbits are commonly seen."
        }
     },
     {
        "_id": "2",
        "_score": 0.12713557,                         <2>
        "_source": {
           "title": "Keeping pets healthy",
           "body": "My quick brown fox eats rabbits on a regular basis."
        }
     }
   ]
}
```
>  ![img](assets/1.png)  ![img](assets/2.png)  注意两个评分是完全相同的。

我们可能期望同时匹配 `title` 和 `body` 字段的文档比只与一个字段匹配的文档的相关度更高，但事实并非如此，因为 `dis_max` 查询只会简单地使用 *单个* 最佳匹配语句的评分 `_score` 作为整体评分。

**tie_breaker 参数**

可以通过指定 `tie_breaker` 这个参数将其他匹配语句的评分也考虑其中：

```js
{
    "query": {
        "dis_max": {
            "queries": [
                { "match": { "title": "Quick pets" }},
                { "match": { "body":  "Quick pets" }}
            ],
            "tie_breaker": 0.3
        }
    }
}
```

结果如下：

```js
{
  "hits": [
     {
        "_id": "2",
        "_score": 0.14757764,                        <1>
        "_source": {
           "title": "Keeping pets healthy",
           "body": "My quick brown fox eats rabbits on a regular basis."
        }
     },
     {
        "_id": "1",
        "_score": 0.124275915,                       <2>
        "_source": {
           "title": "Quick brown rabbits",
           "body": "Brown rabbits are commonly seen."
        }
     }
   ]
}
```
>  ![img](assets/1.png)  ![img](assets/2.png)  文档 2 的相关度比文档 1 略高  



`tie_breaker` 参数提供了一种 `dis_max` 和 `bool` 之间的折中选择，它的评分方式如下：

1. 获得最佳匹配语句的评分 `_score` 。
2. 将其他匹配语句的评分结果与 `tie_breaker` 相乘。
3. 对以上评分求和并规范化。

有了 `tie_breaker` ，会考虑所有匹配语句，但最佳匹配语句依然占最终结果里的很大一部分。
>  ![注意](assets/note.png)  `tie_breaker` 可以是 `0` 到 `1` 之间的浮点数，其中 `0` 代表使用 `dis_max` 最佳匹配语句的普通逻辑， `1` 表示所有匹配语句同等重要。最佳的精确值需要根据数据与查询调试得出，但是合理值应该与零接近（处于 `0.1 - 0.4` 之间），这样就不会颠覆 `dis_max` 最佳匹配性质的根本。


<a name="multi_match查询"></a>
### multi_match 查询

`multi_match` 查询为能在多个字段上反复执行相同查询提供了一种便捷方式。

>  ![注意](assets/note.png)  `multi_match` 多匹配查询的类型有多种，其中的三种恰巧与 [了解我们的数据](https://www.elastic.co/guide/cn/elasticsearch/guide/current/_single_query_string.html#know-your-data) 中介绍的三个场景对应，即： `best_fields` 、 `most_fields` 和 `cross_fields` （最佳字段、多数字段、跨字段）。



默认情况下，查询的类型是 `best_fields` ， 这表示它会为每个字段生成一个 `match` 查询，然后将它们组合到 `dis_max` 查询的内部，如下：

```js
{
  "dis_max": {
    "queries":  [
      {
        "match": {
          "title": {
            "query": "Quick brown fox",
            "minimum_should_match": "30%"
          }
        }
      },
      {
        "match": {
          "body": {
            "query": "Quick brown fox",
            "minimum_should_match": "30%"
          }
        }
      },
    ],
    "tie_breaker": 0.3
  }
}
```

上面这个查询用 `multi_match` 重写成更简洁的形式：

```js
{
    "multi_match": {
        "query":                "Quick brown fox",
        "type":                 "best_fields",              <1>
        "fields":               [ "title", "body" ],
        "tie_breaker":          0.3,
        "minimum_should_match": "30%"                       <2>
    }
}
```
>  ![img](assets/1.png)  `best_fields` 类型是默认值，可以不指定。   
>  
>  ![img](assets/2.png)  如 `minimum_should_match` 或 `operator` 这样的参数会被传递到生成的 `match` 查询中。 

**查询字段名称的模糊匹配**

字段名称可以用模糊匹配的方式给出：任何与模糊模式正则匹配的字段都会被包括在搜索条件中， 例如可以使用以下方式同时匹配 `book_title` 、 `chapter_title` 和 `section_title` （书名、章名、节名）这三个字段：

```js
{
    "multi_match": {
        "query":  "Quick brown fox",
        "fields": "*_title"
    }
}
```

**提升单个字段的权重**

可以使用 `^` 字符语法为单个字段提升权重，在字段名称的末尾添加 `^boost` ， 其中 `boost` 是一个浮点数：

```js
{
    "multi_match": {
        "query":  "Quick brown fox",
        "fields": [ "*_title", "chapter_title^2" ]     <1>
    }
}
```
>  ![img](assets/1.png)   `chapter_title` 这个字段的 `boost` 值为 `2` ，而其他两个字段 `book_title` 和 `section_title` 字段的默认 boost 值为 `1` 。 



### 多数字段

全文搜索被称作是 *召回率（Recall）* 与 *精确率（Precision）* 的战场： *召回率* ——返回所有的相关文档；*精确率* ——不返回无关文档。目的是在结果的第一页中为用户呈现最为相关的文档。

为了提高召回率的效果，我们扩大搜索范围 ——不仅返回与用户搜索词精确匹配的文档，还会返回我们认为与查询相关的所有文档。如果一个用户搜索 “quick brown box” ，一个包含词语 `fast foxes` 的文档被认为是非常合理的返回结果。

如果包含词语 `fast foxes` 的文档是能找到的唯一相关文档，那么它会出现在结果列表的最上面，但是，如果有 100 个文档都出现了词语 `quick brown fox` ，那么这个包含词语 `fast foxes` 的文档当然会被认为是次相关的，它可能处于返回结果列表更下面的某个地方。当包含了很多潜在匹配之后，我们需要将最匹配的几个置于结果列表的顶部。

提高全文相关性精度的常用方式是为同一文本建立多种方式的索引， 每种方式都提供了一个不同的相关度信号 *signal* 。主字段会以尽可能多的形式的去匹配尽可能多的文档。举个例子，我们可以进行以下操作：

- 使用词干提取来索引 `jumps` 、 `jumping` 和 `jumped` 样的词，将 `jump` 作为它们的词根形式。这样即使用户搜索 `jumped` ，也还是能找到包含 `jumping` 的匹配的文档。
- 将同义词包括其中，如 `jump` 、 `leap` 和 `hop` 。
- 移除变音或口音词：如 `ésta` 、 `está` 和 `esta` 都会以无变音形式 `esta` 来索引。

尽管如此，如果我们有两个文档，其中一个包含词 `jumped` ，另一个包含词 `jumping` ，用户很可能期望前者能排的更高，因为它正好与输入的搜索条件一致。

为了达到目的，我们可以将相同的文本索引到其他字段从而提供更为精确的匹配。一个字段可能是为词干未提取过的版本，另一个字段可能是变音过的原始词，第三个可能使用 *shingles* 提供 [词语相似性](https://www.elastic.co/guide/cn/elasticsearch/guide/current/proximity-matching.html) 信息。这些附加的字段可以看成提高每个文档的相关度评分的信号 *signals* ，能匹配字段的越多越好。

一个文档如果与广度匹配的主字段相匹配，那么它会出现在结果列表中。如果文档同时又与 *signal* 信号字段匹配，那么它会获得额外加分，系统会提升它在结果列表中的位置。

我们会在本书稍后对同义词、词相似性、部分匹配以及其他潜在的信号进行讨论，但这里只使用词干已提取（stemmed）和未提取（unstemmed）的字段作为简单例子来说明这种技术。

**多字段映射**

首先要做的事情就是对我们的字段索引两次： 一次使用词干模式以及一次非词干模式。为了做到这点，采用 *multifields* 来实现，已经在 [multifields](https://www.elastic.co/guide/cn/elasticsearch/guide/current/multi-fields.html) 有所介绍：

```js
DELETE /my_index

PUT /my_index
{
    "settings": { "number_of_shards": 1 },           <1>
    "mappings": {
        "my_type": {
            "properties": {
                "title": {                           <2>
                    "type":     "string",
                    "analyzer": "english",
                    "fields": {
                        "std":   {                   <3>
                            "type":     "string",
                            "analyzer": "standard"
                        }
                    }
                }
            }
        }
    }
}
```
>  ![img](assets/1.png)  参考 [被破坏的相关度](https://www.elastic.co/guide/cn/elasticsearch/guide/current/relevance-is-broken.html).   
>  
>  ![img](assets/2.png)  `title` 字段使用 `english` 英语分析器来提取词干。      
>
>  ![img](assets/3.png)  `title.std` 字段使用 `standard` 标准分析器，所以没有词干提取。   

接着索引一些文档：

```js
PUT /my_index/my_type/1
{ "title": "My rabbit jumps" }

PUT /my_index/my_type/2
{ "title": "Jumping jack rabbits" }
```

这里用一个简单 `match` 查询 `title` 标题字段是否包含 `jumping rabbits` （跳跃的兔子）：

```js
GET /my_index/_search
{
   "query": {
        "match": {
            "title": "jumping rabbits"
        }
    }
}
```

因为有了 `english` 分析器，这个查询是在查找以 `jump` 和 `rabbit` 这两个被提取词的文档。两个文档的 `title` 字段都同时包括这两个词，所以两个文档得到的评分也相同：

```js
{
  "hits": [
     {
        "_id": "1",
        "_score": 0.42039964,
        "_source": {
           "title": "My rabbit jumps"
        }
     },
     {
        "_id": "2",
        "_score": 0.42039964,
        "_source": {
           "title": "Jumping jack rabbits"
        }
     }
  ]
}
```

如果只是查询 `title.std` 字段，那么只有文档 2 是匹配的。尽管如此，如果同时查询两个字段，然后使用 `bool` 查询将评分结果 *合并* ，那么两个文档都是匹配的（ `title` 字段的作用），而且文档 2 的相关度评分更高（ `title.std` 字段的作用）：

```js
GET /my_index/_search
{
   "query": {
        "multi_match": {
            "query":  "jumping rabbits",
            "type":   "most_fields",               <1>
            "fields": [ "title", "title.std" ]
        }
    }
}
```
>  ![img](assets/1.png)  我们希望将所有匹配字段的评分合并起来，所以使用 `most_fields` 类型。这让 `multi_match` 查询用 `bool` 查询将两个字段语句包在里面，而不是使用 `dis_max` 查询。 

```js
{
  "hits": [
     {
        "_id": "2",
        "_score": 0.8226396,                   <1>
        "_source": {
           "title": "Jumping jack rabbits"
        }
     },
     {
        "_id": "1",
        "_score": 0.10741998,                  <2>
        "_source": {
           "title": "My rabbit jumps"
        }
     }
  ]
}
```
>  ![img](assets/1.png)  ![img](assets/2.png)  文档 2 现在的评分要比文档 1 高。 

用广度匹配字段 `title` 包括尽可能多的文档——以提升召回率——同时又使用字段 `title.std` 作为 *信号*将相关度更高的文档置于结果顶部。

每个字段对于最终评分的贡献可以通过自定义值 `boost` 来控制。比如，使 `title` 字段更为重要，这样同时也降低了其他信号字段的作用：

```js
GET /my_index/_search
{
   "query": {
        "multi_match": {
            "query":       "jumping rabbits",
            "type":        "most_fields",
            "fields":      [ "title^10", "title.std" ]     <1>
        }
    }
}
```
>  ![img](assets/1.png)  `title` 字段的 `boost` 的值为 `10` 使它比 `title.std` 更重要。 



### 跨字段实体搜索

现在讨论一种普遍的搜索模式：跨字段实体搜索（cross-fields entity search）。 在如 `person` 、 `product`或 `address` （人、产品或地址）这样的实体中，需要使用多个字段来唯一标识它的信息。 `person` 实体可能是这样索引的：

```js
{
    "firstname":  "Peter",
    "lastname":   "Smith"
}
```

或地址：

```js
{
    "street":   "5 Poland Street",
    "city":     "London",
    "country":  "United Kingdom",
    "postcode": "W1V 3DG"
}
```

这与之前描述的 [多字符串查询](https://www.elastic.co/guide/cn/elasticsearch/guide/current/multi-query-strings.html) 很像，但这存在着巨大的区别。在 [多字符串查询](https://www.elastic.co/guide/cn/elasticsearch/guide/current/multi-query-strings.html) 中，我们为每个字段使用不同的字符串，在本例中，我们想使用 *单个* 字符串在多个字段中进行搜索。

我们的用户可能想搜索 “Peter Smith” 这个人，或 “Poland Street W1V” 这个地址，这些词出现在不同的字段中，所以如果使用 `dis_max` 或 `best_fields` 查询去查找 *单个* 最佳匹配字段显然是个错误的方式。

**简单的方式**

依次查询每个字段并将每个字段的匹配评分结果相加，听起来真像是 `bool` 查询：

```js
{
  "query": {
    "bool": {
      "should": [
        { "match": { "street":    "Poland Street W1V" }},
        { "match": { "city":      "Poland Street W1V" }},
        { "match": { "country":   "Poland Street W1V" }},
        { "match": { "postcode":  "Poland Street W1V" }}
      ]
    }
  }
}
```

为每个字段重复查询字符串会使查询瞬间变得冗长，可以采用 `multi_match` 查询， 将 `type` 设置成 `most_fields` 然后告诉 Elasticsearch 合并所有匹配字段的评分：

```js
{
  "query": {
    "multi_match": {
      "query":       "Poland Street W1V",
      "type":        "most_fields",
      "fields":      [ "street", "city", "country", "postcode" ]
    }
  }
}
```

**most_fields 方式的问题**

用 `most_fields` 这种方式搜索也存在某些问题，这些问题并不会马上显现：

- 它是为多数字段匹配 *任意* 词设计的，而不是在 *所有字段* 中找到最匹配的。
- 它不能使用 `operator` 或 `minimum_should_match` 参数来降低次相关结果造成的长尾效应。
- 词频对于每个字段是不一样的，而且它们之间的相互影响会导致不好的排序结果。



### 字段中心式查询

以上三个源于 `most_fields` 的问题都因为它是 *字段中心式（field-centric）* 而不是 *词中心式（term-centric）* 的：当真正感兴趣的是匹配词的时候，它为我们查找的是最匹配的 *字段* 。
>  ![注意](assets/note.png)  `best_fields` 类型也是字段中心式的， 它也存在类似的问题。  

首先查看这些问题存在的原因，再想如何解决它们。



**问题 1 ：在多个字段中匹配相同的词**

回想一下 `most_fields` 查询是如何执行的：Elasticsearch 为每个字段生成独立的 `match` 查询，再用 `bool`查询将他们包起来。

可以通过 `validate-query` API 查看：

```js
GET /_validate/query?explain
{
  "query": {
    "multi_match": {
      "query":   "Poland Street W1V",
      "type":    "most_fields",
      "fields":  [ "street", "city", "country", "postcode" ]
    }
  }
}
```

生成 `explanation` 解释：

```
(street:poland   street:street   street:w1v)
(city:poland     city:street     city:w1v)
(country:poland  country:street  country:w1v)
(postcode:poland postcode:street postcode:w1v)
```

可以发现， *两个* 字段都与 `poland` 匹配的文档要比一个字段同时匹配 `poland` 与 `street` 文档的评分高。



**问题 2 ：剪掉长尾**

在 [匹配精度](https://www.elastic.co/guide/cn/elasticsearch/guide/current/match-multi-word.html#match-precision) 中，我们讨论过使用 `and` 操作符或设置 `minimum_should_match` 参数来消除结果中几乎不相关的长尾，或许可以尝试以下方式：

```js
{
    "query": {
        "multi_match": {
            "query":       "Poland Street W1V",
            "type":        "most_fields",
            "operator":    "and",                                        <1>
            "fields":      [ "street", "city", "country", "postcode" ]
        }
    }
}
```
>  ![img](assets/1.png)  所有词必须呈现。 

但是对于 `best_fields` 或 `most_fields` 这些参数会在 `match` 查询生成时被传入，这个查询的 `explanation` 解释如下：

```
(+street:poland   +street:street   +street:w1v)
(+city:poland     +city:street     +city:w1v)
(+country:poland  +country:street  +country:w1v)
(+postcode:poland +postcode:street +postcode:w1v)
```

换句话说，使用 `and` 操作符要求所有词都必须存在于 *相同字段* ，这显然是不对的！可能就不存在能与这个查询匹配的文档。



**问题 3 ：词频**

在 [什么是相关](https://www.elastic.co/guide/cn/elasticsearch/guide/current/relevance-intro.html) 中，我们解释过每个词默认使用 TF/IDF 相似度算法计算相关度评分：

- 词频

  一个词在单个文档的某个字段中出现的频率越高，这个文档的相关度就越高。

- 逆向文档频率

  一个词在所有文档某个字段索引中出现的频率越高，这个词的相关度就越低。

当搜索多个字段时，TF/IDF 会带来某些令人意外的结果。

想想用字段 `first_name` 和 `last_name` 查询 “Peter Smith” 的例子， Peter 是个平常的名 Smith 也是平常的姓，这两者都具有较低的 IDF 值。但当索引中有另外一个人的名字是 “Smith Williams” 时， Smith 作为名来说很不平常，以致它有一个较高的 IDF 值！

下面这个简单的查询可能会在结果中将 “Smith Williams” 置于 “Peter Smith” 之上，尽管事实上是第二个人比第一个人更为匹配。

```js
{
    "query": {
        "multi_match": {
            "query":       "Peter Smith",
            "type":        "most_fields",
            "fields":      [ "*_name" ]
        }
    }
}
```

这里的问题是 `smith` 在名字段中具有高 IDF ，它会削弱 “Peter” 作为名和 “Smith” 作为姓时低 IDF 的所起作用。



**解决方案**

存在这些问题仅仅是因为我们在处理着多个字段，如果将所有这些字段组合成单个字段，问题就会消失。可以为 `person` 文档添加 `full_name` 字段来解决这个问题：

```js
{
    "first_name":  "Peter",
    "last_name":   "Smith",
    "full_name":   "Peter Smith"
}
```

当查询 `full_name` 字段时：

- 具有更多匹配词的文档会比只有一个重复匹配词的文档更重要。
- `minimum_should_match` 和 `operator` 参数会像期望那样工作。
- 姓和名的逆向文档频率被合并，所以 Smith 到底是作为姓还是作为名出现，都会变得无关紧要。

这么做当然是可行的，但我们并不太喜欢存储冗余数据。取而代之的是 Elasticsearch 可以提供两个解决方案——一个在索引时，而另一个是在搜索时——随后会讨论它们。


<a name="自定义all字段"></a>
### 自定义 _all 字段

在 [all-field](https://www.elastic.co/guide/cn/elasticsearch/guide/current/root-object.html#all-field) 字段中，我们解释过 `_all` 字段的索引方式是将所有其他字段的值作为一个大字符串索引的。然而这么做并不十分灵活，为了灵活我们可以给人名添加一个自定义 `_all` 字段，再为地址添加另一个 `_all` 字段。

Elasticsearch 在字段映射中为我们提供 `copy_to` 参数来实现这个功能：

```js
PUT /my_index
{
    "mappings": {
        "person": {
            "properties": {
                "first_name": {
                    "type":     "string",
                    "copy_to":  "full_name"       <1>
                },
                "last_name": {
                    "type":     "string",
                    "copy_to":  "full_name"       <2>
                },
                "full_name": {
                    "type":     "string"
                }
            }
        }
    }
}
```
>  ![img](assets/1.png)  ![img](assets/2.png)  `first_name` 和 `last_name` 字段中的值会被复制到 `full_name` 字段。 

有了这个映射，我们可以用 `first_name` 来查询名，用 `last_name` 来查询姓，或者直接使用 `full_name` 查询整个姓名。

`first_name` 和 `last_name` 的映射并不影响 `full_name` 如何被索引， `full_name` 将两个字段的内容复制到本地，然后根据 `full_name` 的映射自行索引。

----

>  ![警告](assets/warning.png)  `copy_to` 设置对[multi-field](https://www.elastic.co/guide/cn/elasticsearch/guide/current/multi-fields.html)无效。如果尝试这样配置映射，Elasticsearch 会抛异常。 

为什么呢？多字段只是以不同方式简单索引“主”字段；它们没有自己的数据源。也就是说没有可供 `copy_to` 到另一字段的数据源。

只要对“主”字段 `copy_to` 就能轻而易举的达到相同的效果：

```js
PUT /my_index
{
    "mappings": {
        "person": {
            "properties": {
                "first_name": {
                    "type":     "string",
                    "copy_to":  "full_name",   <1>
                    "fields": {
                        "raw": {
                            "type": "string",
                            "index": "not_analyzed"
                        }
                    }
                },
                "full_name": {
                    "type":     "string"
                }
            }
        }
    }
}
```
>  ![img](assets/1.png)  `copy_to` 是针对“主”字段，而不是多字段的   

----


<a name="crossfields跨字段查询"></a>
### cross-fields 跨字段查询

自定义 `_all` 的方式是一个好的解决方案，只需在索引文档前为其设置好映射。 不过， Elasticsearch 还在搜索时提供了相应的解决方案：使用 `cross_fields` 类型进行 `multi_match` 查询。 `cross_fields` 使用词中心式（term-centric）的查询方式，这与 `best_fields` 和 `most_fields` 使用字段中心式（field-centric）的查询方式非常不同，它将所有字段当成一个大字段，并在 *每个字段* 中查找 *每个词* 。

为了说明字段中心式（field-centric）与词中心式（term-centric）这两种查询方式的不同， 先看看以下字段中心式的 `most_fields` 查询的 `explanation` 解释：

```js
GET /_validate/query?explain
{
    "query": {
        "multi_match": {
            "query":       "peter smith",
            "type":        "most_fields",
            "operator":    "and",                          <1>
            "fields":      [ "first_name", "last_name" ]
        }
    }
}
```
>  ![img](assets/1.png)  所有词都是必须的。

对于匹配的文档， `peter` 和 `smith` 都必须同时出现在相同字段中，要么是 `first_name` 字段，要么 `last_name` 字段：

```
(+first_name:peter +first_name:smith)
(+last_name:peter  +last_name:smith)
```

*词中心式* 会使用以下逻辑：

```
+(first_name:peter last_name:peter)
+(first_name:smith last_name:smith)
```

换句话说，词 `peter` 和 `smith` 都必须出现，但是可以出现在任意字段中。

`cross_fields` 类型首先分析查询字符串并生成一个词列表，然后它从所有字段中依次搜索每个词。这种不同的搜索方式很自然的解决了 [字段中心式](https://www.elastic.co/guide/cn/elasticsearch/guide/current/field-centric.html) 查询三个问题中的二个。剩下的问题是逆向文档频率不同。

幸运的是 `cross_fields` 类型也能解决这个问题，通过 `validate-query` 可以看到：

```js
GET /_validate/query?explain
{
    "query": {
        "multi_match": {
            "query":       "peter smith",
            "type":        "cross_fields",                 <1>
            "operator":    "and",
            "fields":      [ "first_name", "last_name" ]
        }
    }
}
```
>  ![img](assets/1.png)  用 `cross_fields` 词中心式匹配。  

它通过 *混合* 不同字段逆向索引文档频率的方式解决了词频的问题：

```
+blended("peter", fields: [first_name, last_name])
+blended("smith", fields: [first_name, last_name])
```

换句话说，它会同时在 `first_name` 和 `last_name` 两个字段中查找 `smith` 的 IDF ，然后用两者的最小值作为两个字段的 IDF 。结果实际上就是 `smith` 会被认为既是个平常的姓，也是平常的名。

------
>  
>  ![注意](assets/note.png)  为了让 `cross_fields` 查询以最优方式工作，所有的字段都须使用相同的分析器， 具有相同分析器的字段会被分组在一起作为混合字段使用。
>  
>  如果包括了不同分析链的字段，它们会以 `best_fields` 的相同方式被加入到查询结果中。例如：我们将 `title` 字段加到之前的查询中（假设它们使用的是不同的分析器）， explanation 的解释结果如下：
>  
>  ```
>  (+title:peter +title:smith)
>  (
>    +blended("peter", fields: [first_name, last_name])
>    +blended("smith", fields: [first_name, last_name])
>  )
>  ```
>  
>  当在使用 `minimum_should_match` 和 `operator` 参数时，这点尤为重要。
>  
------



**按字段提高权重**

采用 `cross_fields` 查询与 [自定义 `_all` 字段](https://www.elastic.co/guide/cn/elasticsearch/guide/current/custom-all.html) 相比，其中一个优势就是它可以在搜索时为单个字段提升权重。

这对像 `first_name` 和 `last_name` 具有相同值的字段并不是必须的，但如果要用 `title` 和 `description`字段搜索图书，可能希望为 `title` 分配更多的权重，这同样可以使用前面介绍过的 `^` 符号语法来实现：

```js
GET /books/_search
{
    "query": {
        "multi_match": {
            "query":       "peter smith",
            "type":        "cross_fields",
            "fields":      [ "title^2", "description" ]     <1>
        }
    }
}
```
>  ![img](assets/1.png)  `title` 字段的权重提升值为 `2` ， `description` 字段的权重提升值默认为 `1` 。   

自定义单字段查询是否能够优于多字段查询，取决于在多字段查询与单字段自定义 `_all` 之间代价的权衡，即哪种解决方案会带来更大的性能优化就选择哪一种。


<a name="ExactValue精确值字段"></a>
### Exact-Value 精确值字段

在结束多字段查询这个话题之前，我们最后要讨论的是精确值 `not_analyzed` 未分析字段。 将 `not_analyzed` 字段与 `multi_match` 中 `analyzed` 字段混在一起没有多大用处。

原因可以通过查看查询的 explanation 解释得到，设想将 `title` 字段设置成 `not_analyzed` ：

```js
GET /_validate/query?explain
{
    "query": {
        "multi_match": {
            "query":       "peter smith",
            "type":        "cross_fields",
            "fields":      [ "title", "first_name", "last_name" ]
        }
    }
}
```

因为 `title` 字段是未分析过的，Elasticsearch 会将 “peter smith” 这个完整的字符串作为查询条件来搜索！

```
title:peter smith
(
    blended("peter", fields: [first_name, last_name])
    blended("smith", fields: [first_name, last_name])
)
```

显然这个项不在 `title` 的倒排索引中，所以需要在 `multi_match` 查询中避免使用 `not_analyzed` 字段。





