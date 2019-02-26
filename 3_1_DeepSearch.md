## 深入搜索



在 [基础入门](https://www.elastic.co/guide/cn/elasticsearch/guide/current/getting-started.html) 中涵盖了基本工具并对它们有足够详细的描述，这让我们能够开始用 Elasticsearch 搜索数据。 用不了多长时间，就会发现我们想要的更多：希望查询匹配更灵活，排名结果更精确，不同问题域下搜索更具体。

想要进阶，只知道如何使用 `match` 查询是不够的，我们需要理解数据以及如何能够搜索到它们。本章会解释如何索引和查询我们的数据让我们能利用词的相似度（word proximity）、部分匹配（partial matching）、模糊匹配（fuzzy matching）以及语言感知（language awareness）这些优势。

理解每个查询如何贡献相关度评分 `_score` 有助于调试我们的查询：确保我们认为的最佳匹配文档出现在结果首页，以及削减结果中几乎不相关的 “长尾（long tail）”。

搜索不仅仅是全文搜索：我们很大一部分数据都是结构化的，如日期和数字。 我们会以说明结构化搜索与全文搜索最高效的结合方式开始本章的内容。



### 结构化搜索

*结构化搜索（Structured search）* 是指有关探询那些具有内在结构数据的过程。比如日期、时间和数字都是结构化的：它们有精确的格式，我们可以对这些格式进行逻辑操作。比较常见的操作包括比较数字或时间的范围，或判定两个值的大小。

文本也可以是结构化的。如彩色笔可以有离散的颜色集合： `红（red）` 、 `绿（green）` 、 `蓝（blue）` 。一个博客可能被标记了关键词 `分布式（distributed）` 和 `搜索（search）` 。电商网站上的商品都有 UPCs（通用产品码 Universal Product Codes）或其他的唯一标识，它们都需要遵从严格规定的、结构化的格式。

在结构化查询中，我们得到的结果 *总是* 非是即否，要么存于集合之中，要么存在集合之外。结构化查询不关心文件的相关度或评分；它简单的对文档包括或排除处理。

这在逻辑上是能说通的，因为一个数字不能比其他数字 *更* 适合存于某个相同范围。结果只能是：存于范围之中，抑或反之。同样，对于结构化文本来说，一个值要么相等，要么不等。没有 *更似* 这种概念。



#### 精确值查找

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



#### 组合过滤器

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



#### 查找多个精确值

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



#### 范围

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



#### 处理 Null 值  {#处理Null值}

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



#### 关于缓存

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

