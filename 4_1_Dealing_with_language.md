# 处理人类语言

------

“我认识这句话里的所有单词，但并不能理解全句。” 
                                                                                  -- Matt Groening     

------

全文搜索是一场 *查准率* 与 *查全率* 之间的较量—查准率即尽量返回较少的无关文档，而查全率则尽量返回较多的相关文档。 尽管能够精准匹配用户查询的单词，但这仍然不够，我们会错过很多被用户认为是相关的文档。 因此，我们需要把网撒得更广一些，去搜索那些和原文不是完全匹配但却相关的单词。

难道你不期待在搜索“quick brown fox“时匹配到包含“fast brown foxed“的文档，或是搜索“Johnny Walker“时匹配到“Johnnie Walker“， 又或是搜索“Arnolt Schwarzenneger“时匹配到“Arnold Schwarzenegger“吗？

如果文档 *确实* 包含用户查询的内容，那么这些文档应当出现在返回结果的最前面，而匹配程度较低的文档将会排在靠后的位置。 如果没有任何完全匹配的文档，我们至少可以给用户展示一些潜在的匹配结果；它们甚至可能就是用户最初想要的结果。

以下列出了一些可优化的地方：

- 清除类似 `´` ， `^` ， `¨` 的变音符号，这样在搜索 `rôle` 的时候也会匹配 `role` ，反之亦然。请见 [*归一化词元*](https://www.elastic.co/guide/cn/elasticsearch/guide/current/token-normalization.html)。
- 通过提取单词的词干，清除单数和复数之间的差异—`fox` 与 `foxes`—以及时态上的差异—`jumping`、 `jumped` 与 `jumps` 。请见 [*将单词还原为词根*](https://www.elastic.co/guide/cn/elasticsearch/guide/current/stemming.html)。
- 清除常用词或者 *停用词* ，如 `the` ， `and` ， 和 `or` ，从而提升搜索性能。请见 [*停用词: 性能与精度*](https://www.elastic.co/guide/cn/elasticsearch/guide/current/stopwords.html)。
- 包含同义词，这样在搜索 `quick` 时也可以匹配 `fast` ，或者在搜索 `UK` 时匹配 `United Kingdom` 。 请见 [*同义词*](https://www.elastic.co/guide/cn/elasticsearch/guide/current/synonyms.html)。
- 检查拼写错误和替代拼写方式，或者 *同音异型词* —发音一致的不同单词，例如 `their` 与 `there` ， `meat` 、 `meet` 与 `mete` 。 请见 [*拼写错误*](https://www.elastic.co/guide/cn/elasticsearch/guide/current/fuzzy-matching.html)。

在我们可以操控单个单词之前，需要先将文本切分成单词， 这也意味着我们需要知道 *单词* 是由什么组成的。我们将在 [*词汇识别*](https://www.elastic.co/guide/cn/elasticsearch/guide/current/identifying-words.html) 章节阐释这个问题。

在这之前，让我们看看如何更快更简单地开始。



## 开始处理各种语言

Elasticsearch 为很多世界流行语言提供良好的、简单的、开箱即用的语言分析器集合：

阿拉伯语、亚美尼亚语、巴斯克语、巴西语、保加利亚语、加泰罗尼亚语、中文、捷克语、丹麦、荷兰语、英语、芬兰语、法语、加里西亚语、德语、希腊语、北印度语、匈牙利语、印度尼西亚、爱尔兰语、意大利语、日语、韩国语、库尔德语、挪威语、波斯语、葡萄牙语、罗马尼亚语、俄语、西班牙语、瑞典语、土耳其语和泰语。

这些分析器承担以下四种角色：

- 文本拆分为单词：

  `The quick brown foxes` → [ `The`, `quick`, `brown`, `foxes`]

- 大写转小写：

  `The` → `the`

- 移除常用的 _停用词_：

  [ `The`, `quick`, `brown`, `foxes`] → [ `quick`, `brown`, `foxes`]

- 将变型词（例如复数词，过去式）转化为词根：

  `foxes` → `fox`

为了更好的搜索性，每个语言的分析器提供了该语言词汇的具体转换规则：

- `英语` 分析器移除了所有格 `'s`

  `John's` → `john`

- `法语` 分析器移除了 *元音省略* 例如 `l'` 和 `qu'` 和 *变音符号* 例如 `¨` 或 `^` ：

  `l'église` → `eglis`

- `德语` 分析器规范化了切词， 将切词中的 `ä` 和 `ae` 替换为 `a` ， 或将 `ß` 替换为 `ss` ：

  `äußerst` → `ausserst`



### 使用语言分析器

Elasticsearch 的内置分析器都是全局可用的，不需要提前配置， 它们也可以在字段映射中直接指定在某字段上：

```js
PUT /my_index
{
  "mappings": {
    "blog": {
      "properties": {
        "title": {
          "type":     "string",
          "analyzer": "english"             <1>
        }
      }
    }
  }
}
```
>  ![img](assets/1.png)  `title` 字段将会用 `english`（英语）分析器替换默认的 `standard`（标准）分析器   

当然，文本经过 `english` 分析处理，我们会丢失源数据：

```js
GET /my_index/_analyze?field=title      <1>
I'm not happy about the foxes
```
>  ![img](assets/1.png)  切词为: `i'm`，`happi`，`about`，`fox`   

我们无法分辨源文档中是包含单数 `fox` 还是复数 `foxes` ；单词 `not` 因为是停用词所以被移除了， 所以我们无法分辨源文档中是happy about foxes还是not happy about foxes，虽然通过使用 `english` （英语）分析器，使得匹配规则更加宽松，我们也因此提高了召回率，但却降低了精准匹配文档的能力。

为了获得两方面的优势，我们可以使用[multifields](https://www.elastic.co/guide/cn/elasticsearch/guide/current/multi-fields.html)（多字段）对 `title` 字段建立两次索引： 一次使用`english`（英语）分析器，另一次使用 `standard`（标准）分析器:

```js
PUT /my_index
{
  "mappings": {
    "blog": {
      "properties": {
        "title": {                           <1>
          "type": "string",
          "fields": {
            "english": {                     <2>
              "type":     "string",
              "analyzer": "english"
            }
          }
        }
      }
    }
  }
}
```
>  ![img](assets/1.png)  主 `title` 字段使用 `standard`（标准）分析器。   
>  
>  ![img](assets/2.png)  `title.english` 子字段使用 `english`（英语）分析器。  

替换为该字段映射后，我们可以索引一些测试文档来展示怎么在搜索时使用两个字段：

```js
PUT /my_index/blog/1
{ "title": "I'm happy for this fox" }

PUT /my_index/blog/2
{ "title": "I'm not happy about my fox problem" }

GET /_search
{
  "query": {
    "multi_match": {
      "type":     "most_fields",                       <1>
      "query":    "not happy foxes",
      "fields": [ "title", "title.english" ]
    }
  }
}
```
>  ![img](assets/1.png)  使用[`most_fields`](https://www.elastic.co/guide/cn/elasticsearch/guide/current/most-fields.html) query type（多字段搜索语法来）让我们可以用多个字段来匹配同一段文本。   

感谢 `title.english` 字段的切词，无论我们的文档中是否含有单词 `foxes` 都会被搜索到，第二份文档的相关性排行要比第一份高， 因为在 `title` 字段中匹配到了单词 `not` 。



### 配置语言分析器

语言分析器都不需要任何配置，开箱即用， 它们中的大多数都允许你控制它们的各方面行为，具体来说：

- 词干提取排除

  想象下某个场景，用户们想要搜索 `World Health Organization` 的结果， 但是却被替换为搜索 `organ health` 的结果。有这个困惑是因为 `organ` 和 `organization` 有相同的词根： `organ` 。 通常这不是什么问题，但是在一些特殊的文档中就会导致有歧义的结果，所以我们希望防止单词 `organization` 和 `organizations` 被缩减为词干。

- 自定义停用词

  英语中默认的停用词列表如下：`a, an, and, are, as, at, be, but, by, for, if, in, into, is, it, no, not, of, on, or, such, that, the, their, then, there, these, they, this, to, was, will, with`关于单词 `no` 和 `not` 有点特别，这俩词会反转跟在它们后面的词汇的含义。或许我们应该认为这两个词很重要，不应该把他们看成停用词。

为了自定义 `english` （英语）分词器的行为，我们需要基于 `english` （英语）分析器创建一个自定义分析器，然后添加一些配置：

```js
PUT /my_index
{
  "settings": {
    "analysis": {
      "analyzer": {
        "my_english": {
          "type": "english",
          "stem_exclusion": [ "organization", "organizations" ],               <1>
          "stopwords": [                                                       <2>
            "a", "an", "and", "are", "as", "at", "be", "but", "by", "for",
            "if", "in", "into", "is", "it", "of", "on", "or", "such", "that",
            "the", "their", "then", "there", "these", "they", "this", "to",
            "was", "will", "with"
          ]
        }
      }
    }
  }
}
  
GET /my_index/_analyze?analyzer=my_english                                     <3>
The World Health Organization does not sell organs.
```
>  ![img](assets/1.png)  防止 `organization` 和 `organizations` 被缩减为词干       
>  
>  ![img](assets/2.png)  指定一个自定义停用词列表                             
>  
>  ![img](assets/3.png)  切词为 `world` 、 `health` 、 `organization` 、 `does` 、 `not` 、 `sell` 、 `organ`   

我们在 [*将单词还原为词根*](https://www.elastic.co/guide/cn/elasticsearch/guide/current/stemming.html) 和 [*停用词: 性能与精度*](https://www.elastic.co/guide/cn/elasticsearch/guide/current/stopwords.html) 中分别详细讨论了词干提取和停用词。



### 混合语言的陷阱

如果你只需要处理一种语言，那么你很幸运。找到一个正确的策略用于处理多语言文档是一项巨大的挑战。

**在索引的时候**

多语言文档主要有以下三个类型：

- 一种是每份 *document* （文档）有自己的主语言，并包含一些其他语言的片段（参考 [每份文档一种语言](https://www.elastic.co/guide/cn/elasticsearch/guide/current/one-lang-docs.html)。）
- 一种是每个 *field* （域）有自己的主语言, 并包含一些其他语言的片段（参考 [每个域一种语言](https://www.elastic.co/guide/cn/elasticsearch/guide/current/one-lang-fields.html)。）
- 一种是每个 *field* （域）都是混合语言（参考 [混合语言域](https://www.elastic.co/guide/cn/elasticsearch/guide/current/mixed-lang-fields.html)。）

（分词）目标不总是可以实现，我们应当保持将不同语言分隔开。在同一份倒排索引内混合多种语言可能造成一些问题。

**不合理的词干提取**

德语的词干提取规则跟英语，法语，瑞典语等是不一样的。 为不同的语言提供同样的词干提规则 将会导致有的词的词根找的正确，有的词的词根找的不正确，有的词根本找不到词根。 甚至是将不同语言的不同含义的词切为同一个词根，合并这些词根的搜索结果会给用户带来困恼。

提供多种的词干提取器轮流切分同一份文档的结果很有可能得到一堆垃圾，因为下一个词干提取器会尝试切分一个已经被缩减为词干的单词，这加剧了上面提到的问题。

------
>  
>  **每种书写方式一种词干提取器**
>  
>  只有一种情况, *only-one-stemmer* （唯一词干提取器）会发生，就是每种语言都有自己的书写方式。例如，在以色列就有很大的可能一个文档包含希伯来语， 阿拉伯语，俄语（古代斯拉夫语），和英语。
>  
>  ```
אזהרה - Предупреждение - تحذير - Warning
>  ```
>  
>  每种语言使用不同的书写方式，所以一种语言的词干提取器就不会干扰其他语言的，允许为同一>  份文本提供多种词干提取器。
>  
------

**不正确的倒排文档频率**

在 [什么是相关性?](https://www.elastic.co/guide/cn/elasticsearch/guide/current/relevance-intro.html) （相关性教程）中，一个 term （词）在一份文档中出现的频率越高，该term（词）的权重就越低。 为了精确的计算相关性，你需要精确的统计 term-frequency （词频）。

一段德文出现在英语为主的文本中会给与德语单词更高的权重，给那么高权重是因为德语单词相对来说更稀有。 但是如果这份文档跟以德语为主的文档混合在一起，那么这段德文就会有很低的权重。

**在搜索的时候**

然而仅仅考虑你的文档是不够的 。你也需要考虑你的用户会怎么搜索这些文档。 通常你能从用户选择的语言界面来确定用户的主语言，（例如， `mysite.de` 和 `mysite.fr` ） 或者从用户的浏览器的HTTP header（HTTP头文件） [`accept-language`](http://www.w3.org/International/questions/qa-lang-priorities.en.php) 确定。

用户的搜索也注意有三个方面:

- 用户使用他的主语言搜索。
- 用户使用其他的语言搜索，但希望获取主语言的搜索结果。
- 用户使用其他语言搜索，并希望获取该语言的搜索结果。（例如，精通双语的人，或者网络咖啡馆的外国访问者）。

根据你搜索数据的类型，或许会返回单语言的合适结果（例如，一个用户在西班牙网站搜索商品），也可能是用户主语言的搜索结果和其他语言的搜索结果混合。

通常来说，给与用户语言偏好的搜索很有意义。一个使用英语的用户搜索时更希望看到英语 Wikipedia 页面而不是法语 Wikipedia 页面。

**语言识别**

你很可能已经知道你的文档所选用的语言，或者你的文档只是在你自己的组织内编写并被翻译成确定的一系列语言。人类的预识别可能是最可靠的将语言正确归类的方法。

然而，或许你的文档来自第三方资源且没经过语言归类，或者是不正确的归类。这种情况下，你需要一个学习算法来归类你文档的主语言。幸运的是，一些语言有现成的工具包可以帮你解决这个问题。

详细内容是来自 [Mike McCandless](http://blog.mikemccandless.com/2013/08/a-new-version-of-compact-language.html) 的 [chromium-compact-language-detector](https://github.com/mikemccand/chromium-compact-language-detector) 工具包，使用的是google开发的基于 ([Apache License 2.0](http://www.apache.org/licenses/LICENSE-2.0))的开源工具包 [Compact Language Detector](https://code.google.com/p/cld2/) (CLD) 。 它小巧，快速，且精确，并能根据短短的两句话就可以检测 160+ 的语言。 它甚至能对单块文本检测多种语言。支持多种开发语言包括 Python，Perl，JavaScript，PHP，C#/.NET，和 R 。

确定用户搜索请求的语言并不是那么简单。 CLD 是为了至少 200 字符长的文本设计的。字符短的文本，例如搜索关键字，会产生不精确的结果。 这种情况下，或许采取一些简单的启发式算法会更好些，例如该国家的官方语言，用户选择的语言，和 HTTP `accept-language` headers （HTTP头文件）。



### 每份文档一种语言

每个主语言文档 只需要相当简单的设置。 不同语言的文档被分别存放在不同的索引中 — `blogs-en` 、`blogs-fr` ， 如此等等 — 这样每个索引就可以使用相同的类型和相同的域，只是使用不同的分析器：

```js
PUT /blogs-en
{
  "mappings": {
    "post": {
      "properties": {
        "title": {
          "type": "string",               <1>
          "fields": {
            "stemmed": {
              "type":     "string",
              "analyzer": "english"       <2>
            }
}}}}}}

PUT /blogs-fr
{
  "mappings": {
    "post": {
      "properties": {
        "title": {
          "type": "string",              <3>
          "fields": {
            "stemmed": {
              "type":     "string",
              "analyzer": "french"       <4>
            }
}}}}}}
```
>  ![img](assets/1.png)  ![img](assets/3.png)   索引 `blogs-en` 和 `blogs-fr` 的 `post` 类型都有一个包含 `title` 域。   
>  
>  ![img](assets/2.png)  ![img](assets/4.png)   `title.stemmed` 子域使用了具体语言的分析器。  

这个方法干净且灵活。新语言很容易被添加 — 仅仅是创建一个新索引--因为每种语言都是彻底的被分开， 我们不用遭受在 [混合语言的陷阱](https://www.elastic.co/guide/cn/elasticsearch/guide/current/language-pitfalls.html) 中描述的词频和词干提取的问题。

每一种语言的文档都可被独立查询，或者通过查询多种索引来查询多种语言。 我们甚至可以使用 `indices_boost` 参数为特定的语言添加优先权 ：

```js
GET /blogs-*/post/_search                                 <1>
{
    "query": {
        "multi_match": {
            "query":   "deja vu",
            "fields":  [ "title", "title.stemmed" ]       <2>
            "type":    "most_fields"
        }
    }, 
    "indices_boost": {                                    <3>
        "blogs-en": 3,
        "blogs-fr": 2
    }
}
```
>  ![img](assets/1.png)  这个查询会在所有以 `blogs-` 开头的索引中执行。       
>  
>  ![img](assets/2.png)  `title.stemmed` 字段使用每个索引中指定的分析器查询。      
>  
>  ![img](assets/3.png)  也许用户接受语言标头表明，更倾向于英语，然后是法语，所以相应的，我们会为每个索引的结果添加权重。任何其他语言会有一个中性的权重 1 。  



**外语单词**

当然，有些文档含有一些其他语言的单词或句子，且不幸的是这些单词被切为了正确的词根。对于主语言文档，这通常并不是主要的问题。用户经常需要搜索很精确的单词--例如，一个其他语言的引用--而不是语型变化过的单词。召回率 (Recall)可以通过使用 [*归一化词元*](https://www.elastic.co/guide/cn/elasticsearch/guide/current/token-normalization.html) 中讲解的技术提升。

假设有些单词例如地名应当能被主语言和原始语言都能检索，例如 *Munich* 和 *München* 。 这些单词实际上是我们在 [*同义词*](https://www.elastic.co/guide/cn/elasticsearch/guide/current/synonyms.html) 解释过的同义词。

------
>  
>  **不要对语言使用类型**
>  
>  你也许很倾向于为每个语言使用分开的类型， 来代替使用分开的索引。 为了达到最佳效果，你应当避免使用类型。在 [类型和映射](https://www.elastic.co/guide/cn/elasticsearch/guide/current/mapping.html) 解释过，不同类型但有相同域名的域会被索引在 *相同的倒排索引* 中。这意味着不同类型（和不同语言）的词频混合在了一起。
>  
>  为了确保一种语言的词频不会污染其他语言的词频，在后面的章节中会介绍到，无论是为每个语言使用单独的索引，还是使用单独的域都可以。
>  
------



### 每个域一种语言

对于一些实体类，例如:产品、电影、法律声明， 通常这样的一份文本会被翻译成不同语言的文档。虽然这些不同语言的文档可以单独保存在各自的索引中。但另一种更合理的方式是同一份文本的所有翻译统一保存在一个索引中。。

```js
{
   "title":     "Fight club",
   "title_br":  "Clube de Luta",
   "title_cz":  "Klub rváčů",
   "title_en":  "Fight club",
   "title_es":  "El club de la lucha",
   ...
}
```

每份翻译存储在不同的域中，根据域的语言决定使用相应的分析器：

```js
PUT /movies
{
  "mappings": {
    "movie": {
      "properties": {
        "title": {                   <1>
          "type":       "string"
        },
        "title_br": {                <2>
            "type":     "string",
            "analyzer": "brazilian"
        },
        "title_cz": {                <3>
            "type":     "string",
            "analyzer": "czech"
        },
        "title_en": {                <4>
            "type":     "string",
            "analyzer": "english"
        },
        "title_es": {                <5>
            "type":     "string",
            "analyzer": "spanish"
        }
      }
    }
  }
}
```
>  ![img](assets/1.png)  `title` 域含有title的原文，并使用 `standard` （标准）分析器。   
>  
>  ![img](assets/2.png)  ![img](assets/3.png)  ![img](assets/4.png)  ![img](assets/5.png)   其他字段使用适合自己语言的分析器。  

在维持干净的词频方面，虽然 *index-per-language* （一种语言一份索引的方法），不像 *field-per-language* （一种语言一个域的方法）分开索引那么灵活。但是使用 [`update-mapping` API](https://www.elastic.co/guide/cn/elasticsearch/guide/current/mapping-intro.html#updating-a-mapping) 添加一个新域也很简单，那些新域需要新的自定义分析器，这些新分析器只能在索引创建时被装配。有一个变通的方案，你可以先关闭这个索引 [close](https://www.elastic.co/guide/en/elasticsearch/reference/5.6/indices-open-close.html) ，然后使用 [`update-settings` API](https://www.elastic.co/guide/en/elasticsearch/reference/5.6/indices-update-settings.html) ，重新打开这个索引，但是关掉这个索引意味着得停止服务一段时间。

文档的 一种语言可以单独查询，也可以通过查询多个域来查询多种语言。我们甚至可以通过对特定语言设置偏好来提高字段优先级：

```js
GET /movies/movie/_search
{
    "query": {
        "multi_match": {
            "query":    "club de la lucha",
            "fields": [ "title*", "title_es^2" ],    <1>
            "type":     "most_fields"
        }
    }
}
```
>  ![img](assets/1.png)  这个搜索查询所有以 `title` 为前缀的域，但是对 `title_es` 域加权重 `2` 。其他的所有域是中性权重 `1` 。   



### 混合语言域

通常,那些从源数据中获得的多种语言混合在一个域中的文档会超出你的控制， 例如 从网上爬取的页面：

```js
{ "body": "Page not found / Seite nicht gefunden / Page non trouvée" }
```

正确的处理多语言类型文档是非常困难的。即使你简单对所有的域使用 `standard` （标准）分析器， 但你的文档会变得不利于搜索，除非你使用了合适的词干提取器。当然，你不可能只选择一个词干提取器。 词干提取器是由语言具体决定的。或者，词干提取器是由语言和脚本所具体决定的。像在 [每种书写方式一种词干提取器](https://www.elastic.co/guide/cn/elasticsearch/guide/current/language-pitfalls.html#different-scripts) 讨论中那样。 如果每个语言都使用不同的脚本，那么词干提取器就可以合并了。

假设你的混合语言使用的是一样的脚本，例如拉丁文，你有三个可用的选择：

- 切分到不同的域
- 进行多次分析
- 使用 n-grams

**切分到不同的域**

在 [语言识别](https://www.elastic.co/guide/cn/elasticsearch/guide/current/language-pitfalls.html#identifying-language) 提到过的紧凑的语言检测 可以告诉你哪部分文档属于哪种语言。 你可以用 [每个域一种语言](https://www.elastic.co/guide/cn/elasticsearch/guide/current/one-lang-fields.html) 中用过的一样的方法来根据语言切分文本。

**进行多次分析**

如果你主要处理数量有限的语言， 你可以使用多个域，每种语言都分析文本一次。

```js
PUT /movies
{
  "mappings": {
    "title": {
      "properties": {
        "title": {                       <1>
          "type": "string",
          "fields": {
            "de": {                      <2>
              "type":     "string",
              "analyzer": "german"
            },
            "en": {                      <3>
              "type":     "string",
              "analyzer": "english"
            },
            "fr": {                      <4>
              "type":     "string",
              "analyzer": "french"
            },
            "es": {                      <5>
              "type":     "string",
              "analyzer": "spanish"
            }
          }
        }
      }
    }
  }
}
```
>  ![img](assets/1.png)  主域 `title` 使用 `standard` （标准）分析器     
>  
>  ![img](assets/2.png)  ![img](assets/3.png)  ![img](assets/4.png)  ![img](assets/5.png)  每个子域提供不同的语言分析器来对 `title` 域文本进行分析。   



**使用 n-grams**

你可以使用 [Ngrams 在复合词的应用](https://www.elastic.co/guide/cn/elasticsearch/guide/current/ngrams-compound-words.html) 中描述的 方法索引所有的词汇为 n-grams。 大多数语型变化包含给单词添加一个后缀（或在一些语言中添加前缀），所以通过将单词拆成 n-grams，你有很大的机会匹配到相似但不完全一样的单词。 这个可以结合 *analyze-multiple times* （多次分析）方法为不支持的语言提供全域抓取：

```js
PUT /movies
{
  "settings": {
    "analysis": {...}                           <1>
  },
  "mappings": {
    "title": {
      "properties": {
        "title": {
          "type": "string",
          "fields": {
            "de": {
              "type":     "string",
              "analyzer": "german"
            },
            "en": {
              "type":     "string",
              "analyzer": "english"
            },
            "fr": {
              "type":     "string",
              "analyzer": "french"
            },
            "es": {
              "type":     "string",
              "analyzer": "spanish"
            },
            "general": {                       <2>
              "type":     "string",
              "analyzer": "trigrams"
            }
          }
        }
      }
    }
  }
}
```
>  ![img](assets/1.png)  在 `analysis` 章节, 我们按照 [Ngrams 在复合词的应用](https://www.elastic.co/guide/cn/elasticsearch/guide/current/ngrams-compound-words.html) 中描述的定义了同样的 `trigrams` 分析器。   
>  
>  ![img](assets/2.png)  在 `title.general` 域使用 `trigrams` 分析器索引所有的语言。   

当查询抓取所有 `general` 域时，你可以使用 `minimum_should_match` （最少应当匹配数）来减少低质量的匹配。 或许也需要对其他字段进行稍微的加权，给与主语言域的权重要高于其他的在 `general` 上的域：

```js
GET /movies/movie/_search
{
    "query": {
        "multi_match": {
            "query":    "club de la lucha",
            "fields": [ "title*^1.5", "title.general" ],        <1>
            "type":     "most_fields",
            "minimum_should_match": "75%"                       <2>
        }
    }
}
```
>  ![img](assets/1.png)  所有 `title` 或 `title.*` 域给与了比 `title.general` 域稍微高的加权。   
>  
>  ![img](assets/2.png)  `minimum_should_match`（最少应当匹配数） 参数减少了低质量匹配的返回数, 这对 `title.general` 域尤其重要。   












