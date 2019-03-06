## 停用词: 性能与精度  {#停用词性能与精度}

从早期的信息检索到如今， 我们已习惯于磁盘空间和内存被限制为很小一部分，所以 必须使你的索引尽可能小。 每个字节都意味着巨大的性能提升。 (查看 [*将单词还原为词根*](https://www.elastic.co/guide/cn/elasticsearch/guide/current/stemming.html) ) 词干提取的重要性不仅是因为它让搜索的内容更广泛、让检索的能力更深入，还因为它是压缩索引空间的工具。

一种最简单的减少索引大小的方法就是 _索引更少的词_。 有些词要比其他词更重要，只索引那些更重要的词来可以大大减少索引的空间。

那么哪些词条可以被过滤呢？ 我们可以简单分为两组:

- 低频词（Low-frequency terms）

  在文档集合中相对出现较少的词，因为它们稀少，所以它们的权重值更高。

- 高频词（High-frequency terms）

  在索引下的文档集合中出现较多的常用词，例如 `the`、`and`、和`is`。 这些词的权重小，对相关度评分影响不大。
>  ![提示](assets/tip.png)  当然，频率实际上是个可以衡量的标尺而不是非 *高* 即 *低* 的标签。我们可以在标尺的任何位置选取一个标准，低于这个标准的属于低频词，高于它的属于高频词。  

词项到底是低频或是高频取决于它们所处的文档。单词 `and` 如果在所有都是中文的文档里可能是个低频词。在关于数据库的文档集合里，单词 `database` 可能是一个高频词项，它对搜索这个特定集合毫无帮助。

每种语言都存在一些非常常见的单词，它们对搜索没有太大价值。在 Elasticsearch 中，英语默认的停用词为:

```
a, an, and, are, as, at, be, but, by, for, if, in, into, is, it,
no, not, of, on, or, such, that, the, their, then, there, these,
they, this, to, was, will, with
```

这些 *停用词* 通常在索引前就可以被过滤掉，同时对检索的负面影响不大。但是这样做真的是一个较好的解决方案？



### 停用词的优缺点

现在我们拥有更大的磁盘空间，更多内存，并且 还有更好的压缩算法。 将之前的 33 个常见词从索引中移除，每百万文档只能节省 4MB 空间。 所以使用停用词减少索引大小不再是一个有效的理由。 (不过这种说法还有一点需要注意，我们在 [停用词与短语查询](https://www.elastic.co/guide/cn/elasticsearch/guide/current/stopwords-phrases.html) 讨论。)

在此基础上，从索引里将这些词移除会使我们降低某种类型的搜索能力。将前面这些所列单词移除会让我们难以完成以下事情：

- 区分 *happy* 和 _not happy_。
- 搜索乐队名称 The The。
- 查找莎士比亚的名句 “To be, or not to be” （生存还是毁灭)。
- 使用挪威的国家代码: `no`。

移除停用词的最主要好处是性能，假设我们在个具有上百万文档的索引中搜索单词 `fox`。或许 `fox 只在其中 20 个文档中出现，也就是说 Elasticsearch 需要计算 20 个文档的相关度评分 `_score `从而排出前十。现在我们把搜索条件改为 `the OR fox`，几乎所有的文件都包含 `the` 这个词，也就是说 Elasticsearch 需要为所有一百万文档计算评分 `_score`。 由此可见第二个查询肯定没有第一个的结果好。

幸运的是，我们可以用来保持常用词搜索，同时还可以保持良好的性能。首先我们一块学习如何使用停用词。



### 使用停用词

移除停用词的工作是由 `stop` 停用词过滤器完成的，可以通过创建自定义的分析器来使用它（参见 使用停用词过滤器[`stop` 停用词过滤器](https://www.elastic.co/guide/en/elasticsearch/reference/5.6/analysis-stop-tokenfilter.html))。但是，也有一些自带的分析器预置使用停用词过滤器：

- [语言分析器](https://www.elastic.co/guide/en/elasticsearch/reference/5.6/analysis-lang-analyzer.html)

  每个语言分析器默认使用与该语言相适的停用词列表，例如：`english` 英语分析器使用 `_english_` 停用词列表。

- [`standard` 标准分析器](https://www.elastic.co/guide/en/elasticsearch/reference/5.6/analysis-standard-analyzer.html)

  默认使用空的停用词列表：`_none_` ，实际上是禁用了停用词。

- [`pattern` 模式分析器](https://www.elastic.co/guide/en/elasticsearch/reference/5.6/analysis-pattern-analyzer.html)

  默认使用空的停用词列表：为 `_none_` ，与 `standard` 分析器类似。

**停用词和标准分析器（Stopwords and the Standard Analyzer）**

为了让 标准分析器能与 自定义停用词表连用，我们要做的只需创建一个分析器的配置好的版本，然后将停用词列表传入：

```json
PUT /my_index
{
  "settings": {
    "analysis": {
      "analyzer": {
        "my_analyzer": {                           <1>
          "type": "standard",                      <2>
          "stopwords": [ "and", "the" ]            <3>
        }
      }
    }
  }
}
```
>  ![img](assets/1.png)  自定义的分析器名称为 `my_analyzer` 。  
>  
>  ![img](assets/2.png)  这个分析器是一个标准 `standard` 分析器，进行了一些自定义配置。   
>  
>  ![img](assets/3.png)  过滤掉的停用词包括 `and` 和 `the` 。  
>  ![提示](assets/tip.png)  任何语言分析器都可以使用相同的方式配置自定义停用词。  



**保持位置（Maintaining Positions）**

`analyzer` API 的输出结果很有趣:

```json
GET /my_index/_analyze?analyzer=my_analyzer
The quick and the dead
{
   "tokens": [
      {
         "token":        "quick",
         "start_offset": 4,
         "end_offset":   9,
         "type":         "<ALPHANUM>",
         "position":     1                     <1>
      },
      {
         "token":        "dead",
         "start_offset": 18,
         "end_offset":   22,
         "type":         "<ALPHANUM>",
         "position":     4
      }
   ]
}
```
>  ![img](assets/1.png)  `position` 标记每个词汇单元的位置。   

停用词如我们期望被过滤掉了，但有趣的是两个词项的位置 `position` 没有变化：`quick` 是原句子的第二个词，`dead` 是第五个。这对短语查询十分重要，因为如果每个词项的位置被调整了，一个短语查询 `quick dead` 会与以上示例中的文档错误匹配。



**指定停用词（Specifying Stopwords）**

停用词可以以内联的方式传入，就像我们在前面的 例子中那样，通过指定数组:

```json
"stopwords": [ "and", "the" ]
```

特定语言的默认停用词，可以通过使用 `_lang_` 符号来指定:

```json
"stopwords": "_english_"
```

TIP: Elasticsearch 中预定义的与语言相关的停用词列表可以在文档"languages"[`stop` 停用词过滤器](https://www.elastic.co/guide/en/elasticsearch/reference/5.6/analysis-stop-tokenfilter.html) 中找到。

停用词可以通过指定一个特殊列表 `_none_` 来禁用。例如，使用 `_english_` 分析器而不使用停用词，可以通过以下方式做到：

```json
PUT /my_index
{
  "settings": {
    "analysis": {
      "analyzer": {
        "my_english": {
          "type":      "english",    <1>
          "stopwords": "_none_"      <2>
        }
      }
    }
  }
}
```
>  ![img](assets/1.png)  `my_english` 分析器是基于 `english` 分析器。   
>  
>  ![img](assets/2.png)  但禁用了停用词。  

最后，停用词还可以使用一行一个单词的格式保存在文件中。此文件必须在集群的所有节点上，并且通过 `stopwords_path` 参数设置路径:

```json
PUT /my_index
{
  "settings": {
    "analysis": {
      "analyzer": {
        "my_english": {
          "type":           "english",
          "stopwords_path": "stopwords/english.txt"     <1>
        }
      }
    }
  }
}
```
>  ![img](assets/1.png)  停用词文件的路径，该路径相对于 Elasticsearch 的 `config` 目录。   



**使用停用词过滤器（Using the stop Token Filter）**

当你创建 `custom` 分析器时候，可以组合多个 [`stop` 停用词过滤器](https://www.elastic.co/guide/en/elasticsearch/reference/5.6/analysis-stop-tokenfilter.html) 分词器 。例如：我们想要创建一个西班牙语 的分析器:

- 自定义停用词列表
- `light_spanish` 词干提取器
- 在 `asciifolding` 词汇单元过滤器中除去附加符号

我们可以通过以下设置完成:

```json
PUT /my_index
{
  "settings": {
    "analysis": {
      "filter": {
        "spanish_stop": {
          "type":        "stop",
          "stopwords": [ "si", "esta", "el", "la" ]    <1>
        },
        "light_spanish": {                             <2>
          "type":     "stemmer",
          "language": "light_spanish"
        }
      },
      "analyzer": {
        "my_spanish": {
          "tokenizer": "spanish",
          "filter": [                                  <3>
            "lowercase",
            "asciifolding",
            "spanish_stop",
            "light_spanish"
          ]
        }
      }
    }
  }
}
```
>  ![img](assets/1.png)  停用词过滤器采用与 `standard` 分析器相同的参数 `stopwords` 和 `stopwords_path` 。   
>  
>  ![img](assets/2.png)  参见 算法提取器（Algorithmic Stemmers）。   
>  
>  ![img](assets/3.png)  过滤器的顺序非常重要，下面会进行解释。    

我们将 `spanish_stop` 过滤器放置在 `asciifolding` 过滤器之后.这意味着以下三个词组 `esta` 、`ésta` 、`++está++` ，先通过 `asciifolding` 过滤器过滤掉特殊字符变成了 `esta` ，随后使用停用词过滤器会将 `esta` 去除。 如果我们只想移除 `esta` 和 `ésta` ，但是 `++está++` 不想移除。必须将 `spanish_stop` 过滤器放置在 `asciifolding` 之前，并且需要在停用词中指定 `esta` 和 `ésta` 。



**更新停用词（Updating Stopwords）**

想要更新分析器的停用词列表有多种方式， 分析器在创建索引时，当集群节点重启时候，或者关闭的索引重新打开的时候。

如果你使用 `stopwords` 参数以内联方式指定停用词，那么你只能通过关闭索引，更新分析器的配置[update index settings API](https://www.elastic.co/guide/en/elasticsearch/reference/5.6/indices-update-settings.html#update-settings-analysis)，然后在重新打开索引才能更新停用词。

如果你使用 `stopwords_path` 参数指定停用词的文件路径 ，那么更新停用词就简单了。你只需更新文件(在每一个集群节点上)，然后通过两者之中的任何一个操作来强制重新创建分析器:

- 关闭和重新打开索引 (参考 [索引的开与关](https://www.elastic.co/guide/en/elasticsearch/reference/5.6/indices-open-close.html))，
- 一一重启集群下的每个节点。

当然，更新的停用词不会改变任何已经存在的索引。这些停用词的只适用于新的搜索或更新文档。如果要改变现有的文档，则需要重新索引数据。参加 [重新索引你的数据](https://www.elastic.co/guide/cn/elasticsearch/guide/current/reindex.html) 。



### 停用词与性能

保留停用词最大的缺点就影响搜索性能。使用 Elasticsearch 进行 全文搜索，它需要为所有匹配的文档计算相关度评分 `_score` 从而返回最相关的前 10 个文档。

通常大多数的单词在所有文档中出现的频率低于0.1％，但是有少数词（例如 `the` ）几乎存在于所有的文档中。假设有一个索引含有100万个文档，查询 `quick brown fox` 词组，能够匹配上的可能少于1000个文档。但是如果查询 `the quick brown fox` 词组，几乎需要对索引中的100万个文档进行评分和排序，只是为了返回前 10 名最相关的文档。

问题的关键是 `the quick brown fox` 词组实际是查询 `the` 或 `quick` 或 `brown` 或 `fox`— 任何文档即使它什么内容都没有而只包含 `the` 这个词也会被包括在结果集中。因此，我们需要找到一种降低待评分文档数量的方法。

**and 操作符 (and Operator)**

我们想要减少待评分文档的数量，最简单的方式就是在[`and` 操作符](https://www.elastic.co/guide/cn/elasticsearch/guide/current/match-multi-word.html#match-improving-precision) `match` 查询时使用 `and` 操作符， 这样可以让所有词都是必须的。

以下是 `match` 查询：

```json
{
    "match": {
        "text": {
            "query":    "the quick brown fox",
            "operator": "and"
             }
    }
}
```

上述查询被重写为 `bool` 查询如下：

```json
{
    "bool": {
        "must": [
            { "term": { "text": "the" }},
            { "term": { "text": "quick" }},
            { "term": { "text": "brown" }},
            { "term": { "text": "fox" }}
        ]
    }
}
```

`bool` 查询会智能的根据较优的顺序依次执行每个 `term` 查询：它会从最低频的词开始。因为所有词项都必须匹配，只要包含低频词的文档才有可能匹配。使用 `and` 操作符可以大大提升多词查询的速度。

**最少匹配数(minimum_should_match)**

在精度匹配[控制精度](https://www.elastic.co/guide/cn/elasticsearch/guide/current/match-multi-word.html#match-precision)的章节里面，我们讨论过使用 `minimum_should_match` 配置去掉结果中次相关的长尾。虽然它只对这个目的奏效，但是也为我们从侧面带来一个好处，它提供 `and` 操作符相似的性能。

```json
{
    "match": {
        "text": {
            "query": "the quick brown fox",
            "minimum_should_match": "75%"
        }
    }
}
```

在上面这个示例中，四分之三的词都必须匹配，这意味着我们只需考虑那些包含最低频或次低频词的文档。 相比默认使用 `or` 操作符的简单查询，这为我们带来了巨大的性能提升。不过我们有办法可以做得更好……



### 词项的分别管理

在查询字符串中的词项可以分为更重要（低频词）和次重要（高频词）这两类。 只与次重要词项匹配的文档很有可能不太相关。实际上，我们想要文档能尽可能多的匹配那些更重要的词项。

`match` 查询接受一个参数 `cutoff_frequency` ，从而可以让它将查询字符串里的词项分为低频和高频两组。 低频组（更重要的词项）组成 `bulk` 大量查询条件，而高频组（次重要的词项）只会用来评分，而不参与匹配过程。通过对这两组词的区分处理，我们可以在之前慢查询的基础上获得巨大的速度提升。

领域相关的停用词（Domain-Specific Stopwords）



`cutoff_frequency` 配置的好处是，你在 *特定领域* 使用停用词不受约束。 例如，关于电影网站使用的词 *movie* 、 *color* 、 *black* 和 *white* ，这些词我们往往认为几乎没有任何意义。使用 `stop`词汇单元过滤器，这些特定领域的词必须手动添加到停用词列表中。然而 `cutoff_frequency` 会查看索引里词项的具体频率，这些词会被自动归类为 *高频词汇* 。

以下面查询为例：

```json
{
  "match": {
    "text": {
      "query": "Quick and the dead",
      "cutoff_frequency": 0.01          <1>
    }
}
```
>  ![img](assets/1.png)   任何词项出现在文档中超过1%，被认为是高频词。`cutoff_frequency` 配置可以指定为一个分数（ `0.01` ）或者一个正整数（ `5` ）。   

此查询通过 `cutoff_frequency` 配置，将查询条件划分为低频组（ `quick` , `dead` ）和高频组（ `and` , `the`）。然后，此查询会被重写为以下的 `bool` 查询：

```json
{
  "bool": {
    "must": {                                       <1>
      "bool": {
        "should": [
          { "term": { "text": "quick" }},
          { "term": { "text": "dead"  }}
        ]
      }
    },
    "should": {                                     <2>
      "bool": {
        "should": [
          { "term": { "text": "and" }},
          { "term": { "text": "the" }}
        ]
      }
    }
  }
}
```
>  ![img](assets/1.png)   必须匹配至少一个低频／更重要的词项。   
>  
>  ![img](assets/2.png)  高频/次重要性词项是非必须的。   

`must` 意味着至少有一个低频词— `quick` 或者 `dead` —必须出现在被匹配文档中。所有其他的文档被排除在外。 `should` 语句查找高频词 `and` 和 `the` ，但也只是在 `must` 语句查询的结果集文档中查询。 `should` 语句的唯一的工作就是在对如 `Quick _and the_ dead` 和 `_The_ quick but　dead` 语句进行评分时，前者得分比后者高。这种方式可以大大减少需要进行评分计算的文档数量。
>  ![提示](assets/tip.png)  将操作符参数设置成 `and` 会要求所有低频词都必须匹配，同时对包含所有高频词的文档给予更高评分。但是，在匹配文档时，并不要求文档必须包含所有高频词。如果希望文档包含所有的低频和高频词，我们应该使用一个 `bool` 来替代。正如我们在[and 操作符 (and Operator)](https://www.elastic.co/guide/cn/elasticsearch/guide/current/stopwords-performance.html#stopwords-and)中看到的，它的查询效率已经很高了。  



**控制精度**

`minimum_should_match` 参数可以与 `cutoff_frequency` 组合使用，但是此参数仅适用与低频词。 如以下查询：

```json
{
  "match": {
    "text": {
      "query": "Quick and the dead",
      "cutoff_frequency": 0.01,
      "minimum_should_match": "75%"
    }
}
```

将被重写为如下所示:

```json
{
  "bool": {
    "must": {
      "bool": {
        "should": [
          { "term": { "text": "quick" }},
          { "term": { "text": "dead"  }}
        ],
        "minimum_should_match": 1            <1>
      }
    },
    "should": {                              <2>
      "bool": {
        "should": [
          { "term": { "text": "and" }},
          { "term": { "text": "the" }}
        ]
      }
    }
  }
}
```
>  ![img](assets/1.png)   因为只有两个词，原来的75%向下取整为 `1` ，意思是：必须匹配低频词的两者之一。   
>  
>  ![img](assets/2.png)  高频词仍可选的，并且仅用于评分使用。   



**高频词**

当使用 `or` 查询高频词条 ，如— `To be, or not to be` —进行查询时性能最差。只是为了返回最匹配的前十个结果就对只是包含这些词的所有文档进行评分是盲目的。我们真正的意图是查询整个词条出现的文档，所以在这种情况下，不存低频所言，这个查询需要重写为所有高频词条都必须：

```json
{
  "bool": {
    "must": [
      { "term": { "text": "to" }},
      { "term": { "text": "be" }},
      { "term": { "text": "or" }},
      { "term": { "text": "not" }},
      { "term": { "text": "to" }},
      { "term": { "text": "be" }}
    ]
  }
}
```



**对常用词使用更多控制（More Control with Common Terms）**

尽管高频/低频的功能在 `match` 查询中是有用的，有时我们还希望能对它 有更多的控制，想控制它对高频和低频词分组的行为。　`match` 查询针对 `common` 词项查询提供了一组功能。

例如，我们可以让所有低频词都必须匹配，而只对那些包括超过 75% 的高频词文档进行评分：

```json
{
  "common": {
    "text": {
      "query":                  "Quick and the dead",
      "cutoff_frequency":       0.01,
      "low_freq_operator":      "and",
      "minimum_should_match": {
        "high_freq":            "75%"
      }
    }
  }
}
```

更多配置项参见　[`common` terms query](https://www.elastic.co/guide/en/elasticsearch/reference/5.6/query-dsl-common-terms-query.html)。



### 停用词与短语查询

所有查询中 [短语匹配](https://www.elastic.co/guide/cn/elasticsearch/guide/current/phrase-matching.html) 大约占到5%，但是在慢查询里面它们又占大部分。 短语查询性能相对较差，特别是当短语中包括常用词的时候，如 `“To be, or not to be”` 短语全部由停用词组成，这是一种极端情况。原因在于几乎需要匹配全量的数据。

在 停用词的两面 [停用词的优缺点](https://www.elastic.co/guide/cn/elasticsearch/guide/current/pros-cons-stopwords.html),中，我们提到移除停用词只能节省倒排索引中的一小部分空间。这句话只部分正确，一个典型的索引会可能包含部分或所有以下数据：

- 词项字典（Terms dictionary）

  索引中所有文档内所有词项的有序列表，以及包含该词的文档数量。

- 倒排表（Postings list）

  包含每个词项的文档（ID）列表。

- 词频（Term frequency）

  每个词项在每个文档里出现的频率。

- 位置（Positions）

  每个词项在每个文档里出现的位置，供短语查询或近似查询使用。

- 偏移（Offsets）

  每个词项在每个文档里开始与结束字符的偏移，供词语高亮使用，默认是禁用的。

- 规范因子（Norms）

  用来对字段长度进行规范化处理的因子，给较短字段予以更多权重。

将停用词从索引中移除会节省 *词项字典* 和 *倒排表* 里的少量空间，但 *位置* 和 *偏移* 是另一码事。位置和偏移数据很容易变成索引大小的两倍、三倍、甚至四倍。

**位置信息**

`analyzed` 字符串字段的位置信息默认是开启的， 所以短语查询能随时使用到它。 词项出现的越频繁，用来存储它位置信息的空间就越多。在一个大的文档集合中，对于那些非常常见的词，它们的位置信息可能占用成百上千兆的空间。

运行一个针对高频词 `the` 的短语查询可能会导致从磁盘读取好几G的数据。这些数据会被存储到内核文件系统的缓存中，以提高后续访问的速度，这看似是件好事，但这可能会导致其他数据从缓存中被剔除，进一步使后续查询变慢。

这显然是我们需要解决的问题。

**索引选项**

我们首先应该问自己：是否真的需要使用短语查询 或 近似查询 ？

答案通常是：不需要。在很多应用场景下，比如说日志，我们需要知道一个词 *是否* 在文档中（这个信息由倒排表提供）而不是关心词的位置在哪里。或许我们要对一两个字段使用短语查询，但是我们完全可以在其他 `analyzed` 字符串字段上禁用位置信息。

`index_options` 参数 允许我们控制索引里为每个字段存储的信息。 可选值如下:

- `docs`

  只存储文档及其包含词项的信息。这对 `not_analyzed` 字符串字段是默认的。

- `freqs`

  存储 `docs` 信息，以及每个词在每个文档里出现的频次。词频是完成[TF/IDF](https://www.elastic.co/guide/cn/elasticsearch/guide/current/relevance-intro.html) 相关度计算的必要条件，但如果只想知道一个文档是否包含某个特定词项，则无需使用它。

- `positions`

  存储 `docs` 、 `freqs` 、 `analyzed` ，以及每个词项在每个文档里出现的位置。 这对 `analyzed` 字符串字段是默认的，但当不需使用短语或近似匹配时，可以将其禁用。

- `offsets`

  存储 `docs`,`freqs`,`positions`, 以及每个词在原始字符串中开始与结束字符的偏移信息( [`postings` highlighter](https://www.elastic.co/guide/en/elasticsearch/reference/5.6/search-request-highlighting.html#postings-highlighter))。这个信息被用以高亮搜索结果，但它默认是禁用的。

我们可以在索引创建的时候为字段设置 `index_options` 选项，或者在使用 `put-mapping` API新增字段映射的时候设置。我们无法修改已有字段的这个设置：

```json
PUT /my_index
{
  "mappings": {
    "my_type": {
      "properties": {
        "title": {                         <1>
          "type":          "string"
       },
        "content": {                       <2>
          "type":          "string",
          "index_options": "freqs"
      }
    }
  }
}
```
>  ![img](assets/1.png)  `title` 字段使用默认的 `positions` 设置，所以它适于短语或近似查询。   
>  
>  ![img](assets/2.png)  `content` 字段的位置设置是禁用的，所以它无法用于短语或近似查询。  

**停用词**

删除停用词是能显著降低位置信息所占空间的一种方式。 一个被删除停用词的索引仍然可以使用短语查询，因为剩下的词的原始位置仍然被保存着，这正如 [保持位置（Maintaining Positions）](https://www.elastic.co/guide/cn/elasticsearch/guide/current/using-stopwords.html#maintaining-positions) 中看到的那样。 尽管如此，将词项从索引中排除终究会降低搜索能力，这使我们难以区分 *Man in the moon* 与 *Man on the moon* 这两个短语。

幸运的是，鱼与熊掌是可以兼得的：请查看 [`common_grams` 过滤器](https://www.elastic.co/guide/cn/elasticsearch/guide/current/common-grams.html)。



### common_grams 过滤器  {#commongrams过滤器}

`common_grams` 过滤器是针对短语查询能更高效的使用停用词而设计的。 它与 shingles 过滤器 类似（参见 查找相关词（[寻找相关词](https://www.elastic.co/guide/cn/elasticsearch/guide/current/shingles.html))), 为每个相邻词对生成 ，用示例解释更为容易。

`common_grams` 过滤器根据 `query_mode` 设置的不同而生成不同输出结果：`false` （为索引使用） 或 `true`（为搜索使用），所以我们必须创建两个独立的分析器：

```json
PUT /my_index
{
  "settings": {
    "analysis": {
      "filter": {
        "index_filter": {                               <1>
          "type":         "common_grams",
          "common_words": "_english_"                   <2>
        },
        "search_filter": {                              <3>
          "type":         "common_grams",
          "common_words": "_english_",                  <4>
          "query_mode":   true
        }
      },
      "analyzer": {
        "index_grams": {                                <5>
          "tokenizer":  "standard",
          "filter":   [ "lowercase", "index_filter" ]
        },
        "search_grams": {                               <6>
          "tokenizer": "standard",
          "filter":  [ "lowercase", "search_filter" ]
        }
      }
    }
  }
}
```
>  ![img](assets/1.png)   ![img](assets/3.png)  首先我们基于 `common_grams` 过滤器创建两个过滤器： `index_filter` 在索引时使用（此时 `query_mode` 的默认设置是 `false` ）， `search_filter` 在查询时使用（此时 `query_mode` 的默认设置是 `true` ）。  
>  
>  ![img](assets/2.png)  ![img](assets/4.png)  `common_words` 参数可以接受与 `stopwords` 参数同样的选项（参见 指定停用词 [指定停用词（Specifying Stopwords）](https://www.elastic.co/guide/cn/elasticsearch/guide/current/using-stopwords.html#specifying-stopwords) ）。这个过滤器还可以接受参数 `common_words_path` ，使用存于文件里的常用词。  
>  
>  ![img](assets/5.png)  ![img](assets/6.png)  然后我们使用过滤器各创建一个索引时分析器和查询时分析器。  

有了自定义分析器，我们可以创建一个字段在索引时使用 `index_grams` 分析器：

```json
PUT /my_index/_mapping/my_type
{
  "properties": {
    "text": {
      "type":            "string",
      "analyzer":  "index_grams",     <1>
      "search_analyzer": "standard"   <2>
    }
  }
}
```
>  ![img](assets/1.png)  ![img](assets/2.png)  `text` 字段索引时使用 `index_grams` 分析器，但是在搜索时默认使用 `standard` 分析器，稍后我们会解释其原因。   



**索引时（At Index Time）**

如果我们对 短语 *The quick and brown fox* 进行拆分，它生成如下词项：

```text
Pos 1: the_quick
Pos 2: quick_and
Pos 3: and_brown
Pos 4: brown_fox
```

新的 `index_grams` 分析器生成以下词项：

```text
Pos 1: the, the_quick
Pos 2: quick, quick_and
Pos 3: and, and_brown
Pos 4: brown
Pos 5: fox
```

所有的词项都是以 `unigrams` 形式输出的（the、quick 等等），但是如果一个词本身是常用词或者跟随着常用词，那么它同时还会在 `unigram` 同样的位置以 `bigram` 形式输出：`the_quick` ， `quick_and` ， `and_brown` 。



**单字查询（Unigram Queries）**

因为索引包含 `unigrams` ，可以使用与其他字段相同的技术进行查询，例如：

```json
GET /my_index/_search
{
  "query": {
    "match": {
      "text": {
        "query": "the quick and brown fox",
        "cutoff_frequency": 0.01
      }
    }
  }
}
```

上面这个查询字符串是通过为文本字段配置的 `search_analyzer` 分析器 --本例中使用的是 `standard` 分析器-- 进行分析的， 它生成的词项为： `the` ， `quick` ， `and` ， `brown` ， `fox` 。

因为 `text` 字段的索引中包含与 `standard` 分析去生成的一样的 `unigrams` ，搜索对于任何普通字段都能正常工作。



**二元语法短语查询（Bigram Phrase Queries）**

但是，当我们进行短语查询时， 我们可以用专门的 `search_grams` 分析器让整个过程变得更高效：

```json
GET /my_index/_search
{
  "query": {
    "match_phrase": {
      "text": {
        "query":    "The quick and brown fox",
        "analyzer": "search_grams"                   <1>
      }
    }
  }
}
```
>  ![img](assets/1.png)  对于短语查询，我们重写了默认的 `search_analyzer` 分析器，而使用 `search_grams` 分析器。   

`search_grams` 分析器会生成以下词项：

```text
Pos 1: the_quick
Pos 2: quick_and
Pos 3: and_brown
Pos 4: brown
Pos 5: fox
```

分析器排除了所有常用词的 `unigrams`，只留下常用词的 `bigrams` 以及低频的 `unigrams`。如 `the_quick`这样的 `bigrams` 比单个词项 `the` 更为少见，这样有两个好处：

- `the_quick` 的位置信息要比 `the` 的小得多，所以它读取磁盘更快，对系统缓存的影响也更小。
- 词项 `the_quick` 没有 `the` 那么常见，所以它可以大量减少需要计算的文档。



**两词短语（Two-Word Phrases）**

我们的优化可以更进一步， 因为大多数的短语查询只由两个词组成，如果其中一个恰好又是常用词，例如：

```json
GET /my_index/_search
{
  "query": {
    "match_phrase": {
      "text": {
        "query":    "The quick",
        "analyzer": "search_grams"
      }
    }
  }
}
```

那么 `search_grams` 分析器会输出单个语汇单元：`the_quick` 。这将原来昂贵的查询（查询 `the` 和 `quick`）转换成了对单个词项的高效查找。  



### 停用词与相关性

在结束停用词 相关内容之前，最后一个话题是关于相关性的。在索引中保留停用词会降低相关度计算的准确性，特别是当我们的文档非常长时。

正如我们在 [词频饱和度](https://www.elastic.co/guide/cn/elasticsearch/guide/current/pluggable-similarites.html#bm25-saturation) 已经讨论过的， 原因在于 [词频饱和度](https://www.elastic.co/guide/cn/elasticsearch/guide/current/pluggable-similarites.html#bm25-saturation) 并没有强制对词频率的影响设置上限 。 基于逆文档频率的影响，非常常用的词可能只有很低的权重，但是在长文档中，单个文档出现的绝对数量很大的停用词会导致这些词被不自然的加权。

可以考虑对包含停用词的较长字段使用 [Okapi BM25](https://www.elastic.co/guide/cn/elasticsearch/guide/current/pluggable-similarites.html#bm25) 相似度算法，而不是默认的 Lucene 相似度。



## 同义词

词干提取是通过简化他们的词根形式来扩大搜索的范围，同义词 通过相关的观念和概念来扩大搜索范围。 也许没有文档匹配查询 “英国女王“ ，但是包含 “英国君主” 的文档可能会被认为是很好的匹配。

用户搜索 “美国” 并且期望找到包含 *美利坚合众国* 、 *美国* 、 *美洲* 、或者 *美国各州* 的文档。 然而，他们不希望搜索到关于 `国事` 或者 `政府机构` 的结果。

这个例子提供了宝贵的经验，它向我们阐述了，区分不同的概念对于人类是多么简单而对于纯粹的机器是多么棘手的事情。通常我们会对语言中的每一个词去尝试提供同义词以确保任何一个文档都是可发现的，以保证不管文档之间有多么微小的关联性都能够被检索出来。

这样做是不对的。就像我们更喜欢不用或少用词根而不是过分使用词根一样，同义词也应该只在必要的时候使用。 这是因为用户可以理解他们的搜索结果受限于他们的搜索词，如果搜索结果看上去几乎是随机时，他们就会变得无法理解（注：大规模使用同义词会导致查询结果趋向于让人觉得是随机的）。

同义词可以用来合并几乎相同含义的词，如 `跳` 、 `跳越` 或者 `单脚跳行` ，和 `小册子` 、 `传单` 或者 `资料手册`。 或者，它们可以用来让一个词变得更通用。例如， `鸟` 可以作为 `猫头鹰` 或 `鸽子` 的通用代名词，还有， `成人` 可以被用于 `男人` 或者 `女人` 。

同义词似乎是一个简单的概念，但是正确的使用它们却是非常困难的。在这一章，我们会介绍使用同义词的技巧和讨论它的局限性和陷阱。
>  ![提示](assets/tip.png)  同义词扩大了一个匹配文件的范围。正如 [词干提取](https://www.elastic.co/guide/cn/elasticsearch/guide/current/stemming.html) 或者 [部分匹配](https://www.elastic.co/guide/cn/elasticsearch/guide/current/partial-matching.html) ，同义词的字段不应该被单独使用，而应该与一个针对主字段的查询操作一起使用，这个主字段应该包含纯净格式的原始文本。 在使用同义词时，参阅 [多数字段](https://www.elastic.co/guide/cn/elasticsearch/guide/current/most-fields.html) 的解释来维护相关性。  



### 使用同义词

同义词可以取代现有的语汇单元或 通过使用 [`同义词` 语汇单元过滤器](https://www.elastic.co/guide/en/elasticsearch/reference/5.6/analysis-synonym-tokenfilter.html)，添加到语汇单元流中：

```json
PUT /my_index
{
  "settings": {
    "analysis": {
      "filter": {
        "my_synonym_filter": {
          "type": "synonym",           <1>
          "synonyms": [                <2>
            "british,english",
            "queen,monarch"
          ]
        }
      },
      "analyzer": {
        "my_synonyms": {
          "tokenizer": "standard",
          "filter": [
            "lowercase",
            "my_synonym_filter"        <3>
          ]
        }
      }
    }
  }
}
```
>  ![img](assets/1.png)  首先，我们定义了一个 `同义词` 类型的语汇单元过滤器。        
>  
>  ![img](assets/2.png)  我们在 [同义词格式](https://www.elastic.co/guide/cn/elasticsearch/guide/current/synonym-formats.html) 中讨论同义词格式。  
>  
>  ![img](assets/3.png)  然后我们创建了一个使用 `my_synonym_filter` 的自定义分析器。  
>  
>  ![提示](assets/tip.png)  同义词可以使用 `synonym` 参数来内嵌指定，或者必须 存在于集群每一个节点上的同义词文件中。 同义词文件路径由 `synonyms_path` 参数指定，应绝对或相对于 Elasticsearch `config`目录。参照 [更新停用词（Updating Stopwords）](https://www.elastic.co/guide/cn/elasticsearch/guide/current/using-stopwords.html#updating-stopwords) 的技巧，可以用来刷新的同义词列表。  

通过 `analyze` API 来测试我们的分析器，显示如下：

```json
GET /my_index/_analyze?analyzer=my_synonyms
Elizabeth is the English queen
Pos 1: (elizabeth)
Pos 2: (is)
Pos 3: (the)
Pos 4: (british,english)    <1>
Pos 5: (queen,monarch)      <2>
```
>  ![img](assets/1.png)  ![img](assets/2.png)  所有同义词与原始词项占有同一个位置。   

这样的一个文件将匹配任何以下的查询： `English queen` 、`British queen` 、 `English monarch` 或 `British monarch` 。 即使是一个短语查询也将会工作，因为每个词项的位置已被保存。
>  ![提示](assets/tip.png)  
在索引和搜索中使用相同的同义词语汇单元过滤器是多余的。 如果在索引的时候，我们用 `english` 和 `british` 这两个术语代替 `English` ， 然后在搜索的时候，我们只需要搜索这些词项中的一个。或者，如果在索引的时候我们不使用同义词，然后在搜索的时候，我们将需要把对 `English` 的查询转换为 `english` 或者 `british` 的查询。  

是否在搜索或索引的时候做同义词扩展可能是一个困难的选择。我们将探索更多的选择 [扩展或收缩](https://www.elastic.co/guide/cn/elasticsearch/guide/current/synonyms-expand-or-contract.html)。



### 同义词格式

同义词最简单的表达形式是 逗号分隔：

```
"jump,leap,hop"
```

如果遇到这些词项中的任何一项，则将其替换为所有列出的同义词。例如：

```text
原始词项:   取代:
────────────────────────────────
jump            → (jump,leap,hop)
leap            → (jump,leap,hop)
hop             → (jump,leap,hop)
```

或者, 使用 `=>` 语法，可以指定一个词项列表（在左边），和一个或多个替换（右边）的列表：

```
"u s a,united states,united states of america => usa"
"g b,gb,great britain => britain,england,scotland,wales"
原始词项:   取代:
────────────────────────────────
u s a           → (usa)
united states   → (usa)
great britain   → (britain,england,scotland,wales)
```

如果多个规则指定同一个同义词，它们将被合并在一起，且顺序无关，否则使用最长匹配。以下面的规则为例：

```
"united states            => usa",
"united states of america => usa"
```

如果这些规则相互冲突，Elasticsearch 会将 `United States of America` 转换为词项 `(usa),(of),(america)` 。否则，会使用最长的序列，即最终得到词项 `(usa)` 。



### 扩展或收缩

在 [同义词格式](https://www.elastic.co/guide/cn/elasticsearch/guide/current/synonym-formats.html) 中，我们看到了 可以通过 *简单扩展* 、 *简单收缩* 、或*类型扩展* 来指明同义词规则。 本章节我们将在这三者间做个权衡比较。
>  ![提示](assets/tip.png)  本节仅处理单词同义词。多词同义词又增添了一层复杂性，在 [多词同义词和短语查询](https://www.elastic.co/guide/cn/elasticsearch/guide/current/multi-word-synonyms.html) 中，我们将会讨论。  



**简单扩展**

通过 *简单扩展* ， 我们可以把同义词列表中的任意一个词扩展成同义词列表 *所有* 的词：

```
"jump,hop,leap"
```

扩展可以应用在索引阶段或查询阶段。两者都有优点 (⬆)︎ 和缺点 (⬇)︎。到底要在哪个阶段使用，则取决于性能与灵活性：

|                | 索引                                                         | 查询                                                     |
| -------------- | ------------------------------------------------------------ | -------------------------------------------------------- |
| **索引的大小** | ⬇︎ 大索引。因为所有的同义词都会被索引，所以索引的大小相对会变大一些。 | ⬆︎ 正常大小。                                             |
| **关联**       | ⬇︎ 所有同义词都有相同的 IDF（至于什么是 IDF ，参见 [什么是相关性?](https://www.elastic.co/guide/cn/elasticsearch/guide/current/relevance-intro.html)），这意味着通用的词和较常用的词都拥有着相同的权重。 | ⬆︎ 每个同义词 IDF 都和原来一样。                          |
| **性能**       | ⬆︎ 查询只需要找到查询字符串中指定单个词项。                   | ⬇︎ 对一个词项的查询重写来查找所有的同义词，从而降低性能。 |
| **灵活性**     | ⬇︎ 同义词规则不能改变现有的文件。对于有影响的新规则，现有的文件都要重建（注：重新索引一次文档）。 | ⬆︎ 同义词规则可以更新不需要索引文件。                     |



**简单收缩**

*简单收缩* ，把 左边的多个同义词映射到了右边的单个词：

```
"leap,hop => jump"
```

它必须同时应用于索引和查询阶段，以确保查询词项映射到索引中存在的同一个值。

相对于简单扩展方法，这种方法也有一些优点和一些缺点：

- 索引的大小

  ⬆︎ 索引大小是正常的，因为只有单一词项被索引。

- 关联

  ⬇︎ 所有词项的 IDF 是一样的，所以你不能区分比较常用的词、不常用的单词。

- 性能

  ⬆︎ 查询只需要在索引中找到单词的出现。

- 灵活性

  ⬆︎ 新同义词可以添加到规则的左侧并在查询阶段使用。例如，我们想添加 `bound` 到先前指定的同义词规则中。那么下面的规则将作用于包含 `bound` 的查询或包含 `bound` 的文档索引：`"leap,hop,bound => jump"`似乎对旧有的文档不起作用是么？其实我们可以把上面这个同义词规则改写下，以便对旧有文档同样起作用：`"leap,hop,bound => jump,bound"`当你重建索引文件，你可以恢复到上面的规则（注： `leap,hop,bound => jump` ）来获得查询单个词项的性能优势（注：因为上面那个规则相比这个而言，查询阶段就只要查询一个词了）。



**类型扩展**

类型扩展是完全不同于简单收缩 或扩张， 并不是平等看待所有的同义词，而是扩大了词的意义，使被拓展的词更为通用。以这些规则为例：

```
"cat    => cat,pet",
"kitten => kitten,cat,pet",
"dog    => dog,pet"
"puppy  => puppy,dog,pet"
```

通过在索引阶段使用类型扩展：

- 一个关于 `kitten` 的查询会发现关于 kittens 的文档。
- 查询一个 `cat` 会找到关于 kittens 和 cats 的文档。
- 一个 `pet` 的查询将发现有关的 kittens、cats、puppies、dogs 或者 pets 的文档。

或者在查询阶段使用类型扩展， `kitten` 的查询结果就会被拓展成涉及到 kittens、cats、dogs。

您也可以有两全其美的办法，通过在索引阶段应用类型扩展同义词规则，以确保类型在索引中存在。然后，在查询阶段， 你可以选择不采用同义词（使 `kitten` 查询只返回 kittens 的文件）或采用同义词， `kitten` 的查询操作就会返回包括 kittens、cats、pets（也包括 dogs 和 puppies）的相关结果。

前面的示例规则，对 `kitten` 的 IDF 将是正确的，而 `cat` 和 `pet` 的 IDF 将会被 Elasticsearch 降权。然而, 这是对你有利的，当一个针对 `kitten` 的查询被拓展成了针对 `kitten OR cat OR pet` 的查询， 那么 `kitten` 相关的文档就应该排在最上方，其次是 `cat` 的文件， `pet` 的文件将被排在最底部。



### 同义词和分析链

在 [同义词格式](https://www.elastic.co/guide/cn/elasticsearch/guide/current/synonym-formats.html) 一章中，我们使用 `u s a` 来举例阐述一些同义词相关的知识。那么为什么 我们使用的不是 `U.S.A.` 呢？原因是， 这个 `同义词` 的语汇单元过滤器只能接收到在它前面的语汇单元过滤器或者分词器的输出结果（这里看不到原始文本）。

假设我们有一个分析器，它由 `standard` 分词器、 `lowercase` 的语汇单元过滤器、 `synonym` 的语汇单元过滤器组成。文本 `U.S.A.` 的分析过程，看起来像这样的：

```text
original string（原始文本）                       → "U.S.A."
standard           tokenizer（分词器）            → (U),(S),(A)
lowercase          token filter（语汇单元过滤器）  → (u),(s),(a)
synonym            token filter（语汇单元过滤器）  → (usa)
```

如果我们有指定的同义词 `U.S.A.` ，它永远不会匹配任何东西。因为， `my_synonym_filter` 看到词项的时候，句号已经被移除了，并且字母已经被小写了。

这其实是一个非常需要注意的地方。如果我们想同时使用同义词特性与词根提取特性，那么 `jumps` 、 `jumped` 、 `jump` 、 `leaps` 、 `leaped` 和 `leap` 这些词是否都会被索引成一个 `jump` ？ 我们 可以把同义词过滤器放置在词根提取之前，然后把所有同义词以及词形变化都列举出来：

```
"jumps,jumped,leap,leaps,leaped => jump"
```

但更简洁的方式将同义词过滤器放置在词根过滤器之后，然后把词根形式的同义词列举出来：

```
"leap => jump"
```



**大小写敏感的同义词**

通常，我们把同义词过滤器放置在 `lowercase` 语汇单元过滤器之后，因此，所有的同义词 都是小写。 但有时会导致奇怪的合并。例如， `CAT` 扫描和一只 `cat` 有很大的不同，或者 `PET` （正电子发射断层扫描）和 `pet` 。 就此而言，姓 `Little` 也是不同于形容词 `little` 的 (尽管当一个句子以它开头时，首字母会被大写)。

如果根据使用情况来区分词义，则需要将同义词过滤器放置在 `lowercase` 筛选器之前。当然，这意味着同义词规则需要列出所有想匹配的变化（例如， `Little、LITTLE、little` ）。

相反,可以有两个同义词过滤器：一个匹配大小写敏感的同义词，一个匹配大小写不敏感的同义词。例如，大小写敏感的同义词规则可以是这个样子：

```
"CAT,CAT scan           => cat_scan"
"PET,PET scan           => pet_scan"
"Johnny Little,J Little => johnny_little"
"Johnny Small,J Small   => johnny_small"
```

大小不敏感的同义词规则可以是这个样子：

```
"cat                    => cat,pet"
"dog                    => dog,pet"
"cat scan,cat_scan scan => cat_scan"
"pet scan,pet_scan scan => pet_scan"
"little,small"
```

大小写敏感的同义词规则不仅会处理 `CAT scan` ，而且有时候也可能会匹配到 `CAT scan` 中的 `CAT` （注：从而导致 `CAT scan` 被转化成了同义词 `cat_scan scan` ）。出于这个原因，在大小写敏感的同义词列表中会有一个针对较坏替换情况的特异规则 `cat_scan scan` 。

提示： 可以看到它们可以多么轻易地变得复杂。同平时一样， `analyze` API 是帮手，用它来检查分析器是否正确配置。参阅 [测试分析器](https://www.elastic.co/guide/cn/elasticsearch/guide/current/analysis-intro.html#analyze-api)。



### 多词同义词和短语查询

至此，同义词看上去还挺简单的。然而不幸的是，复杂的部分才刚刚开始。 为了能使 [短语查询](https://www.elastic.co/guide/cn/elasticsearch/guide/current/phrase-matching.html) 正常工作， Elasticsearch 需要知道每个词在初始文本中的位置。多词同义词会严重破坏词的位置信息，尤其当新增的同义词标记长度各不相同的时候。

我们创建一个同义词语汇单元过滤器，然后使用下面这样的同义词规则：

```
"usa,united states,u s a,united states of america"
PUT /my_index
{
  "settings": {
    "analysis": {
      "filter": {
        "my_synonym_filter": {
          "type": "synonym",
          "synonyms": [
            "usa,united states,u s a,united states of america"
          ]
        }
      },
      "analyzer": {
        "my_synonyms": {
          "tokenizer": "standard",
          "filter": [
            "lowercase",
            "my_synonym_filter"
          ]
        }
      }
    }
  }
}

GET /my_index/_analyze?analyzer=my_synonyms&text=
The United States is wealthy
```

`解析器` 会输出下面这样的结果：

```text
Pos 1:  (the)
Pos 2:  (usa,united,u,united)
Pos 3:  (states,s,states)
Pos 4:  (is,a,of)
Pos 5:  (wealthy,america)
```

如果你用上面这个同义词语汇单元过滤器索引一个文档，然后执行一个短语查询，那你就会得到惊人的结果，下面这些短语都不会匹配成功：

- The usa is wealthy
- The united states of america is wealthy
- The U.S.A. is wealthy

但是这些短语会：

- United states is wealthy
- Usa states of wealthy
- The U.S. of wealthy
- U.S. is america

如果你是在查询阶段使同义词，那你就会看到更加诡异的匹配结果。看下这个 `validate-query` 查询：

```json
GET /my_index/_validate/query?explain
{
  "query": {
    "match_phrase": {
      "text": {
        "query": "usa is wealthy",
        "analyzer": "my_synonyms"
      }
    }
  }
}
```

查询关键字会被同义词语汇单元过滤器处理成类似这样的信息：

```
"(usa united u united) (is states s states) (wealthy a of) america"
```

这会匹配包含有 `u is of america` 的文档，但是匹配不出任何含有 `america` 的文档。
>  ![提示](assets/tip.png)  多词同义 对高亮匹配结果也会造成影响。一个针对 `USA` 的查询，返回的结果可能却高亮了： The *United States is wealthy* 。  



**使用简单收缩进行短语查询**

避免这种混乱的方法是使用 [简单收缩](https://www.elastic.co/guide/cn/elasticsearch/guide/current/synonyms-expand-or-contract.html#synonyms-contraction)， 用单个词项 表示所有的同义词， 然后在查询阶段，就只需要针对这单个词进行查询了：

```json
PUT /my_index
{
  "settings": {
    "analysis": {
      "filter": {
        "my_synonym_filter": {
          "type": "synonym",
          "synonyms": [
            "united states,u s a,united states of america=>usa"
          ]
        }
      },
      "analyzer": {
        "my_synonyms": {
          "tokenizer": "standard",
          "filter": [
            "lowercase",
            "my_synonym_filter"
          ]
        }
      }
    }
  }
}

GET /my_index/_analyze?analyzer=my_synonyms
The United States is wealthy
```

上面那个查询信息就会被处理成类似下面这样：

```text
Pos 1:  (the)
Pos 2:  (usa)
Pos 3:  (is)
Pos 5:  (wealthy)
```

现在我们再次执行我们之前做过的那个 `validate-query` 查询，就会输出一个简单又合理的结果：

```
"usa is wealthy"
```

这个方法的缺点是，因为把 `united states of america` 转换成了同义词 `usa`, 你就不能使用 `united states of america` 去搜索出 `united` 或者 `states` 。 你需要使用一个额外的字段并用另一个解析器链来达到这个目的。



**同义词与 query_string 查询**

本书很少谈论到 `query_string` 查询， 因为真心不推荐你用它。 在 [复杂查询](https://www.elastic.co/guide/cn/elasticsearch/guide/current/search-lite.html#query-string-query) 一节中有提到，由于 `query_string` 查询支持一个精简的 *查询语法* ，因此，可能这会导致它搜出一些出人意料的结果或者甚至是含有语法错误的结果。

这种查询方式存在不少问题，而其中之一便与多词同义有关。为了支持它的查询语法，你必须用指定的、该语法所能识别的操作符号来标示，比如 `AND` 、 `OR` 、 `+` 、 `-` 、 `field:` 等等。 (更多相关内容参阅 [`query_string` 语法](https://www.elastic.co/guide/en/elasticsearch/reference/5.6/query-dsl-query-string-query.html#query-string-syntax) 。)

而在这种语法的解析过程中，解析动作会把查询文本在空格符处作切分，然后分别把每个切分出来的词传递给相关性解析器。 这也即意味着你的同义词解析器永远都不可能收到类似 `United States` 这样的多个单词组成的同义词。由于不会把 `United States` 作为一个原子性的文本，所以同义词解析器的输入信息永远都是两个被切分开的词 `United` 和 `States` 。

所幸， `match` 查询相比而言就可靠得多了，因为它不支持上述语法，所以多个字组成的同义词不会被切分开，而是会完整地交给解析器处理。



### 符号同义词

最后一节内容我们来阐述下怎么对符号进行同义词处理，这和我们前面讲的同义词 处理不太一样。 *符号同义词* 是用别名来表示这个符号，以防止它在分词过程中被误认为是不重要的标点符号而被移除。

虽然绝大多数情况下，符号对于全文搜索而言都无关紧要，但是字符组合而成的表情，或许又会是很有意义的东西，甚至有时候会改变整个句子的含义，对比一下这两句话：

- 我很高兴能在星期天工作。
- 我很高兴能在星期天工作 :( （注：难过的表情）

`标准` （注：standard）分词器或许会简单地消除掉第二个句子里的字符表情，致使两个原本意思相去甚远的句子变得相同。

我们可以先使用 [`映射`](https://www.elastic.co/guide/en/elasticsearch/reference/5.6/analysis-mapping-charfilter.html)字符过滤器，在文本被递交给分词器处理之前， 把字符表情替换成 符号同义词 `emoticon_happy` 或者 `emoticon_sad` ：

```json
PUT /my_index
{
  "settings": {
    "analysis": {
      "char_filter": {
        "emoticons": {
          "type": "mapping",
          "mappings": [                      <1>
            ":)=>emoticon_happy",
            ":(=>emoticon_sad"
          ]
        }
      },
      "analyzer": {
        "my_emoticons": {
          "char_filter": "emoticons",
          "tokenizer":   "standard",
          "filter":    [ "lowercase" ]
          ]
        }
      }
    }
  }
}

GET /my_index/_analyze?analyzer=my_emoticons
I am :) not :(                                  <2>
```
>  ![img](assets/1.png)  `映射` 过滤器把字符从 `=>` 左边的格式转变成右边的样子。   
>  
>  ![img](assets/2.png)  输出： `i` 、 `am` 、 `emoticon_happy` 、 `not` 、 `emoticon_sad` 。  

很少有人会搜 `emoticon_happy` 这个词，但是确保类似字符表情的这类重要符号被存储到索引中是非常好的做法，在进行情感分析的时候会很有用。当然，我们也可以用真实的词汇来处理符号同义词，比如： `happy` 或者 `sad` 。

提示： `映射` 字符过滤器是个非常有用的过滤器，它可以用来对一些已有的字词进行替换操作， 你如果想要采用更灵活的正则表达式去替换字词的话，那你可以使用 [`pattern_replace` ](https://www.elastic.co/guide/en/elasticsearch/reference/5.6/analysis-pattern-replace-charfilter.html)字符过滤器。



## 拼写错误

我们期望在类似时间和价格的结构化数据上执行一个查询来返回精确匹配的文档。 然而，好的全文检索不应该是完全相同的限定逻辑。 相反，我们可以扩大范围以包括 *可能* 的匹配，而根据相关性得分将更好的匹配推到结果集的顶部。

事实上，只能完全匹配的全文搜索可能会困扰你的用户。 难道不希望在搜索 `quick brown fox` 时匹配一个包含 `fast brown foxes` 的文档， 搜索 `Johnny Walker` 同时匹配 `Johnnie Walker` ，搜索 `Arnold Shcwarzenneger` 同时匹配 `Arnold Schwarzenegger` ?

如果存在完全符合用户查询的文档，他们应该出现在结果集的顶部，而较弱的匹配可以被包含在列表的后面。 如果没有精确匹配的文档，至少我们可以显示有可能匹配用户要求的文档，它们甚至可能是用户最初想要的！

我们已经在 [*归一化词元*](https://www.elastic.co/guide/cn/elasticsearch/guide/current/token-normalization.html) 看过自由变音匹配， [*将单词还原为词根*](https://www.elastic.co/guide/cn/elasticsearch/guide/current/stemming.html) 中的词干， [*同义词*](https://www.elastic.co/guide/cn/elasticsearch/guide/current/synonyms.html) 中的同义词， 但所有这些方法假定单词拼写正确，或者每个单词拼写只有唯一的方法。

Fuzzy matching 允许查询时匹配错误拼写的单词，而语音语汇单元过滤器可以在索引时用来进行 *近似读音*匹配。



### 模糊性

*模糊匹配* 对待 “模糊” 相似的两个词似乎是同一个词。 首先，我们需要对我们所说的 *模糊性* 进行定义。

在1965年，Vladimir Levenshtein 开发出了 [Levenshtein distance](http://en.wikipedia.org/wiki/Levenshtein_distance)， 用来度量从一个单词转换到另一个单词需要多少次单字符编辑。他提出了三种类型的单字符编辑：

- 一个字符 *替换* 另一个字符： _f_ox → _b_ox
- *插入* 一个新的字符：sic → sic_k_
- *删除* 一个字符：b_l_ack → back

[Frederick Damerau](http://en.wikipedia.org/wiki/Frederick_J._Damerau) 后来在这些操作基础上做了一个扩展：

- 相邻两个字符的 *换位* ： _st_ar → _ts_ar

举个例子，将单词 `bieber` 转换成 `beaver` 需要下面几个步骤：

1. 把 `b` 替换成 `v` ：bie_b_er → bie_v_er
2. 把 `i` 替换成 `a` ：b_i_ever → b_a_ ever
3. 把 `e` 和 `a` 进行换位：b_ae_ver → b_ea_ver

这三个步骤表示 [Damerau-Levenshtein edit distance](https://en.wikipedia.org/wiki/Damerau%E2%80%93Levenshtein_distance) 编辑距离为 3 。

显然，从 `beaver` 转换成 `bieber` 是一个很长的过程—他们相距甚远而不能视为一个简单的拼写错误。 Damerau 发现 80% 的拼写错误编辑距离为 1 。换句话说， 80% 的拼写错误可以对原始字符串用 *单次编辑* 进行修正。

Elasticsearch 指定了 `fuzziness` 参数支持对最大编辑距离的配置，默认为 ２ 。

当然，单次编辑对字符串的影响取决于字符串的长度。对单词 `hat` 两次编辑能够产生 `mad` ， 所以对一个只有 3 个字符长度的字符串允许两次编辑显然太多了。 `fuzziness` 参数可以被设置为 `AUTO` ，这将导致以下的最大编辑距离：

- 字符串只有 1 到 2 个字符时是 `0`
- 字符串有 3 、 4 或者 5 个字符时是 `1`
- 字符串大于 5 个字符时是 `2`

当然，你可能会发现编辑距离 `2` 仍然是太多了，返回的结果似乎并不相关。 把最大 `fuzziness` 设置为 `1`，你可以得到更好的结果和更好的性能。



### 模糊查询

[`fuzzy` 查询](https://www.elastic.co/guide/en/elasticsearch/reference/5.6/query-dsl-fuzzy-query.html)是 `term` 查询的模糊等价。 也许你很少直接使用它，但是理解它是如何工作的，可以帮助你在更高级别的 `match` 查询中使用模糊性。

为了解它是如何运作的，我们首先索引一些文档：

```json
POST /my_index/my_type/_bulk
{ "index": { "_id": 1 }}
{ "text": "Surprise me!"}
{ "index": { "_id": 2 }}
{ "text": "That was surprising."}
{ "index": { "_id": 3 }}
{ "text": "I wasn't surprised."}
```

现在我们可以为词 `surprize` 运行一个 `fuzzy` 查询：

```json
GET /my_index/my_type/_search
{
  "query": {
    "fuzzy": {
      "text": "surprize"
    }
  }
}
```

`fuzzy` 查询是一个词项级别的查询，所以它不做任何分析。它通过某个词项以及指定的 `fuzziness` 查找到词典中所有的词项。 `fuzziness` 默认设置为 `AUTO` 。

在我们的例子中， `surprise` 比较 `surprise` 和 `surprised` 都在编辑距离 2 以内， 所以文档 1 和 3 匹配。通过以下查询，我们可以减少匹配度到仅匹配 `surprise` ：

```json
GET /my_index/my_type/_search
{
  "query": {
    "fuzzy": {
      "text": {
        "value": "surprize",
        "fuzziness": 1
      }
    }
  }
}
```



**提高性能**

`fuzzy` 查询的工作原理是给定原始词项及构造一个 *编辑自动机*— 像表示所有原始字符串指定编辑距离的字符串的一个大图表。

然后模糊查询使用这个自动机依次高效遍历词典中的所有词项以确定是否匹配。 一旦收集了词典中存在的所有匹配项，就可以计算匹配文档列表。

当然，根据存储在索引中的数据类型，一个编辑距离 2 的模糊查询能够匹配一个非常大数量的词项同时执行效率会非常糟糕。 下面两个参数可以用来限制对性能的影响：

- `prefix_length`

  不能被 “模糊化” 的初始字符数。 大部分的拼写错误发生在词的结尾，而不是词的开始。 例如通过将 `prefix_length` 设置为 `3` ，你可能够显著降低匹配的词项数量。

- `max_expansions`

  如果一个模糊查询扩展了三个或四个模糊选项， 这些新的模糊选项也许是有意义的。如 果它产生 1000 个模糊选项，那么就基本没有意义了。 设置 `max_expansions` 用来限制将产生的模糊选项的总数量。模糊查询将收集匹配词项直到达到 `max_expansions` 的限制。



### 模糊匹配查询

`match` 查询支持 开箱即用的模糊匹配：

```json
GET /my_index/my_type/_search
{
  "query": {
    "match": {
      "text": {
        "query":     "SURPRIZE ME!",
        "fuzziness": "AUTO",
        "operator":  "and"
      }
    }
  }
}
```

查询字符串首先进行分析，会产生词项 `[surprize, me]` ，并且每个词项根据指定的 `fuzziness` 进行模糊化。

同样， `multi_match` 查询也 支持 `fuzziness` ，但只有当执行查询时类型是 `best_fields` 或者 `most_fields` ：

```json
GET /my_index/my_type/_search
{
  "query": {
    "multi_match": {
      "fields":  [ "text", "title" ],
      "query":     "SURPRIZE ME!",
      "fuzziness": "AUTO"
    }
  }
}
```

`match` 和 `multi_match` 查询都支持 `prefix_length` 和 `max_expansions` 参数。
>  ![提示](assets/tip.png)  模糊性（Fuzziness）只能在 `match` and `multi_match` 查询中使用。不能使用在短语匹配、常用词项或 `cross_fields` 匹配。  



### 模糊性评分

用户喜欢模糊查询。他们认为这种查询会魔法般的找到正确拼写组合。 很遗憾，实际效果平平。

假设我们有1000个文档包含 ``Schwarzenegger`` ，只是一个文档的出现拼写错误 ``Schwarzeneger`` 。 根据 [term frequency/inverse document frequency](https://www.elastic.co/guide/cn/elasticsearch/guide/current/scoring-theory.html#tfidf) 理论，这个拼写错误文档比拼写正确的相关度更高，因为错误拼写出现在更少的文档中！

换句话说，如果我们对待模糊匹配 类似其他匹配方法，我们将偏爱错误的拼写超过了正确的拼写，这会让用户抓狂。
>  ![提示](assets/tip.png)  模糊匹配不应用于参与评分--只能在有拼写错误时扩大匹配项的范围。  

默认情况下， `match` 查询给定所有的模糊匹配的恒定评分为1。这可以满足在结果列表的末尾添加潜在的匹配记录，并且没有干扰非模糊查询的相关性评分。
>  ![提示](assets/tip.png)  在模糊查询最初出现时很少能单独使用。他们更好的作为一个 ``bigger`` 场景的部分功能特性，如 *search-as-you-type* [`完成` 建议](https://www.elastic.co/guide/en/elasticsearch/reference/5.6/search-suggesters-completion.html)或 *did-you-mean* [`短语` 建议](https://www.elastic.co/guide/en/elasticsearch/reference/5.6/search-suggesters-phrase.html)。



### 语音匹配

最后，在尝试任何其他匹配方法都无效后，我们可以求助于搜索发音相似的词， 即使他们的拼写不同。

有一些用于将词转换成语音标识的算法。 [Soundex](http://en.wikipedia.org/wiki/Soundex) 算法是这些算法的鼻祖， 而且大多数语音算法是 Soundex 的改进或者专业版本，例如 [Metaphone](http://en.wikipedia.org/wiki/Metaphone) 和 [Double Metaphone](http://en.wikipedia.org/wiki/Metaphone#Double_Metaphone) （扩展了除英语以外的其他语言的语音匹配）， [Caverphone](http://en.wikipedia.org/wiki/Caverphone) 算法匹配了新西兰的名称， [Beider-Morse](https://en.wikipedia.org/wiki/Daitch%E2%80%93Mokotoff_Soundex#Beider.E2.80.93Morse_Phonetic_Name_Matching_Algorithm) 算法吸收了 Soundex 算法为了更好的匹配德语和依地语名称， [Kölner Phonetik](http://de.wikipedia.org/wiki/K%C3%B6lner_Phonetik) 为了更好的处理德语词汇。

值得一提的是，语音算法是相当简陋的， 他们设计初衷针对的语言通常是英语或德语。这限制了他们的实用性。 不过，为了某些明确的目标，并与其他技术相结合，语音匹配能够作为一个有用的工具。

首先，你需要从 <https://www.elastic.co/guide/en/elasticsearch/plugins/current/analysis-phonetic.html>获取语音分析插件并在集群的每个节点安装， 然后重启每个节点。

然后，您可以创建一个使用语音语汇单元过滤器的自定义分析器，并尝试下面的方法：

```json
PUT /my_index
{
  "settings": {
    "analysis": {
      "filter": {
        "dbl_metaphone": {                       <1>
          "type":    "phonetic",
          "encoder": "double_metaphone"
        }
      },
      "analyzer": {
        "dbl_metaphone": {
          "tokenizer": "standard",
          "filter":    "dbl_metaphone"           <2>
        }
      }
    }
  }
}
```
>  ![img](assets/1.png)  首先，配置一个自定义 `phonetic` 语汇单元过滤器并使用 `double_metaphone` 编码器。   
>  
>  ![img](assets/2.png)  然后在自定义分析器中使用自定义语汇单元过滤器。  

现在我们可以通过 `analyze` API 来进行测试：

```json
GET /my_index/_analyze?analyzer=dbl_metaphone
Smith Smythe
```

每个 `Smith` 和 `Smythe` 在同一位置产生两个语汇单元： `SM0` 和 `XMT` 。 通过分析器播放 `John` ， `Jon` 和 `Johnnie` 将产生两个语汇单元 `JN` 和 `AN` ，而 `Jonathon` 产生语汇单元 `JN0N` 和 `ANTN` 。

语音分析器可以像任何其他分析器一样使用。 首先映射一个字段来使用它，然后索引一些数据：

```json
PUT /my_index/_mapping/my_type
{
  "properties": {
    "name": {
      "type": "string",
      "fields": {
        "phonetic": {                        <1>
          "type":     "string",
          "analyzer": "dbl_metaphone"
        }
      }
    }
  }
}

PUT /my_index/my_type/1
{
  "name": "John Smith"
}

PUT /my_index/my_type/2
{
  "name": "Jonnie Smythe"
}
```
>  ![img](assets/1.png)  `name.phonetic` 字段使用自定义 `dbl_metaphone` 分析器。  

可以使用 `match` 查询来进行搜索：

```json
GET /my_index/my_type/_search
{
  "query": {
    "match": {
      "name.phonetic": {
        "query": "Jahnnie Smeeth",
        "operator": "and"
      }
    }
  }
}
```

这个查询返回全部两个文档，演示了如何进行简陋的语音匹配。 用语音算法计算评分是没有价值的。 语音匹配的目的不是为了提高精度，而是要提高召回率--以扩展足够的范围来捕获可能匹配的文档。

通常更有意义的使用语音算法是在检索到结果后，由另一台计算机进行消费和后续处理，而不是由人类用户直接使用。

