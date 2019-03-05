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



## 词汇识别

英语单词相对而言比较容易辨认：单词之间都是以空格或者（一些）标点隔开。 然而即使在英语词汇中也会有一些争议： *you’re* 是一个单词还是两个？ *o’clock* ， *cooperate* ， *half-baked* ，或者 *eyewitness* 这些呢？

德语或者荷兰语把独立的单词合并起来创造一个长的合成词如 *Weißkopfseeadler* (white-headed sea eagle) , 但是为了在查询 `Adler` (eagle)的时候返回查询 `Weißkopfseeadler` 的结果，我们需要懂得怎么将合并词拆成词组。

亚洲的语言更复杂：很多语言在单词，句子，甚至段落之间没有空格。 有些词可以用一个字来表达，但是同样的字在另一个字旁边的时候就是不同意思的长词的一部分。

显而易见的是没有能够奇迹般处理所有人类语言的万能分析器，Elasticsearch 为很多语言提供了专用的分析器， 其他特殊语言的分析器以插件的形式提供。

然而并不是所有语言都有专用分析器，而且有时候你甚至无法确定处理的是什么语言。这种情况，我们需要一些忽略语言也能合理工作的标准工具包。



### 标准分析器

任何全文检索的字符串域都默认使用 `standard` 分析器。 如果我们想要一个 [`自定义` 分析器](https://www.elastic.co/guide/cn/elasticsearch/guide/current/custom-analyzers.html) ，可以按照如下定义方式重新实现 `标准` 分析器：

```js
{
    "type":      "custom",
    "tokenizer": "standard",
    "filter":  [ "lowercase", "stop" ]
}
```

在 [*归一化词元*](https://www.elastic.co/guide/cn/elasticsearch/guide/current/token-normalization.html) （标准化词汇单元）和 [*停用词: 性能与精度*](https://www.elastic.co/guide/cn/elasticsearch/guide/current/stopwords.html) （停用词）中，我们讨论了 `lowercase` （小写字母）和 `stop` （停用词） *词汇单元过滤器* ，但是现在，我们专注于 `standard` *tokenizer* （标准分词器）。



### 标准分词器

*分词器* 接受一个字符串作为输入，将 这个字符串拆分成独立的词或 *语汇单元（token）* （可能会丢弃一些标点符号等字符），然后输出一个 *语汇单元流（token stream）* 。

有趣的是用于词汇 *识别* 的算法。 `whitespace` （空白字符）分词器按空白字符 —— 空格、tabs、换行符等等进行简单拆分 —— 然后假定连续的非空格字符组成了一个语汇单元。例如：

```js
GET /_analyze?tokenizer=whitespace
You're the 1st runner home!
```

这个请求会返回如下词项（terms）： `You're` 、 `the` 、 `1st` 、 `runner` 、 `home!`

`letter` 分词器 ，采用另外一种策略，按照任何非字符进行拆分， 这样将会返回如下单词： `You` 、 `re` 、 `the` 、 `st` 、 `runner` 、 `home` 。

`standard` 分词器使用 Unicode 文本分割算法 （定义来源于 [Unicode Standard Annex #29](http://unicode.org/reports/tr29/)）来寻找单词 *之间* 的界限，并且输出所有界限之间的内容。 Unicode 内含的知识使其可以成功的对包含混合语言的文本进行分词。

标点符号 可能是单词的一部分，也可能不是，这取决于它出现的位置：

```js
GET /_analyze?tokenizer=standard
You're my 'favorite'.
```

在这个例子中，`You're` 中的撇号被视为单词的一部分，然而 `'favorite'` 中的单引号则不会被视为单词的一部分， 所以分词结果如下： `You're` 、 `my` 、 `favorite` 。
>  ![提示](assets/tip.png)  `uax_url_email` 分词器和 `standard` 分词器工作方式极其相同。 区别只在于它能识别 email 地址和 URLs 并输出为单个语汇单元。 `standard` 分词器则不一样，会将 email 地址和 URLs 拆分成独立的单词。 例如，email 地址 `joe-bloggs@foo-bar.com` 的分词结果为 `joe` 、 `bloggs` 、 `foo` 、 `bar.com` 。  

`standard` 分词器是大多数语言分词的一个合理的起点，特别是西方语言。 事实上，它构成了大多数特定语言分析器的基础，如 `english` 、`french` 和 `spanish` 分析器。 它也支持亚洲语言，只是有些缺陷，你可以考虑通过 ICU 插件的方式使用 `icu_tokenizer` 进行替换。



### 安装 ICU 插件  {#安装ICU插件}

Elasticsearch的 [ICU 分析器插件](https://github.com/elasticsearch/elasticsearch-analysis-icu) 使用 *国际化组件 Unicode* (ICU) 函数库（详情查看 [site.project.org](http://site.icu-project.org/) ）提供丰富的处理 Unicode 工具。 这些包含对处理亚洲语言特别有用的 `icu_分词器` ，还有大量对除英语外其他语言进行正确匹配和排序所必须的分词过滤器。
>  ![注意](assets/note.png)  ICU 插件是处理英语之外语言的必需工具，非常推荐你安装并使用它，不幸的是，因为是基于额外的 ICU 函数库， 不同版本的ICU插件可能并不兼容之前的版本，当更新插件的时候，你需要重新索引你的数据。  

安装这个插件，第一步先关掉你的Elasticsearch节点，然后在Elasticsearch的主目录运行以下命令：

```sh
./bin/plugin -install elasticsearch/elasticsearch-analysis-icu/$VERSION 
```
>  ![img](assets/1.png)  当前 `$VERSION` （版本）可以在以下地址找到 *https://github.com/elasticsearch/elasticsearch-analysis-icu*.   

一旦安装后，重启Elasticsearch，你将会看到类似如下的一条启动日志：

```
[INFO][plugins] [Mysterio] loaded [marvel, analysis-icu], sites [marvel]
```

如果你有很多节点并以集群方式运行的，你需要在集群的每个节点都安装这个插件。



### icu_分词器  {#icu分词器}

`icu_分词器` 和 `标准分词器` 使用同样的 Unicode 文本分段算法， 只是为了更好的支持亚洲语，添加了泰语、老挝语、中文、日文、和韩文基于词典的词汇识别方法，并且可以使用自定义规则将缅甸语和柬埔寨语文本拆分成音节。

例如，分别比较 `标准分词器` 和 `icu_分词器` 在分词泰语中的 `'Hello. I am from Bangkok.'` 产生的词汇单元：

```js
GET /_analyze?tokenizer=standard
สวัสดี ผมมาจากกรุงเทพฯ
```

`标准分词器` 产生了两个词汇单元，每个句子一个： `สวัสดี` ， `ผมมาจากกรุงเทพฯ` 。这个只是你想搜索整个句子 `'I am from Bangkok.'` 的时候有用，但是如果你仅想搜索 `'Bangkok.'` 则不行。

```js
GET /_analyze?tokenizer=icu_tokenizer
สวัสดี ผมมาจากกรุงเทพฯ
```

相反， `icu_分词器` 可以把文本分成独立的单词（ `สวัสดี` ， `ผม` ， `มา` ， `จาก` ， `กรุงเทพฯ` ），这使得文档更容易被搜索到。

相较而言, `标准分词器` 分词中文和日文的时候“过度分词”了，经常将一个完整的词拆分为独立的字符，因为单词之间并没有空格，很难区分连续的字符是间隔的单词还是一个句子中的单字：

- 向的意思是 *facing* （面对）， 日的意思是 *sun* （太阳），葵的意思是 *hollyhock* （蜀葵）。当写在一起的时候, 向日葵的意思是 *sunflower* （向日葵）。
- 五的意思是 *five* （五）或者 *fifth* （第五）， 月的意思是 *month* （月份），雨的意思是 *rain* （下雨）。 第一个和第二个字符写在一起成了五月，意思是 *the month of May_（一年中的五月）， 然而添加上第三个字符, 五月雨的意思是 _continuous rain* （连续不断的下雨,梅雨）。当在合并第四个字符， 式， 意思是 *style* （样式），五月雨式这个单词则成了一种不屈不挠持续不断的东西的形容词。

虽然每个字符本身可以是一个单词，但使词汇单元保持更大的原始概念比使其仅作为一个词组的一部分要有意义的多：

```js
GET /_analyze?tokenizer=standard
向日葵

GET /_analyze?tokenizer=icu_tokenizer
向日葵
```

`标准分词器` 在前面的例子中将每个字符输出为单独的词汇单元： `向` ， `日` ， `葵` 。 `icu_分词器` 会输出单个词汇单元 `向日葵` （sunflower） 。

`标准分词器` 和 `icu_分词器` 的另一个不同的地方是后者会将不同书写方式的字符（例如，`βeta` ）拆分成独立的词汇单元 — `β` 和 `eta`— ，而前者则会输出单个词汇单元： `βeta` 。



### 整理输入文本

当输入文本是干净的时候分词器提供最佳分词结果，有效文本，这里 *有效* 指的是遵从 Unicode 算法期望的标点符号规则 。 然而很多时候，我们需要处理的文本会是除了干净文本之外的任何文本。在分词之前整理文本会提升输出结果的质量。

**HTML 分词**

将 HTML 通过 `标准分词器` 或 `icu_分词器` 分词将产生糟糕的结果。这些分词器不知道如何处理 HTML 标签。例如：

```js
GET /_analyze?tokenizer=standard
<p>Some d&eacute;j&agrave; vu <a href="http://somedomain.com>">website</a>
```

`标准分词器` 会混淆 HTML 标签和实体，并且输出以下词汇单元： `p` 、 `Some` 、 `d` 、 `eacute` 、 `j` 、 `agrave` 、 `vu` 、 `a` 、 `href` 、 `http` 、 `somedomain.com` 、 `website` 、 `a` 。这些词汇单元显然不知所云！

*字符过滤器* 可以添加进分析器中，在将文本传给分词器之前预处理该文本。在这种情况下，我们可以用 `html_strip` 字符过滤器 移除 HTML 标签并编码 HTML 实体如 `&eacute;` 为一致的 Unicode 字符。

字符过滤器可以通过 `analyze` API 进行测试，这需要在查询字符串中指明它们：

```js
GET /_analyze?tokenizer=standard&char_filters=html_strip
<p>Some d&eacute;j&agrave; vu <a href="http://somedomain.com>">website</a>
```

想将它们作为分析器的一部分使用，需要把它们添加到 `custom` 类型的自定义分析器里：

```js
PUT /my_index
{
    "settings": {
        "analysis": {
            "analyzer": {
                "my_html_analyzer": {
                    "tokenizer":     "standard",
                    "char_filter": [ "html_strip" ]
                }
            }
        }
    }
}
```

一旦自定义分析器创建好之后， 我们新的 `my_html_analyzer` 就可以用 `analyze` API 测试：

```js
GET /my_index/_analyze?analyzer=my_html_analyzer
<p>Some d&eacute;j&agrave; vu <a href="http://somedomain.com>">website</a>
```

这次输出的词汇单元才是我们期望的： `Some` ， `déjà` ， `vu` ， `website` 。

**整理标点符号**

`标准分词器` 和 `icu_分词器` 都能理解单词中的撇号应当被视为单词的一部分，然而包围单词的单引号在不应该。 分词文本 `You're my 'favorite'` ， 会被输出正确的词汇单元 `You're ， my ， favorite` 。

不幸的是， Unicode 列出了一些有时会被用为撇号的字符：

- `U+0027`

  撇号标记为 (`'`)— 原始 ASCII 符号

- `U+2018`

  左单引号标记为 (`‘`)— 当单引用时作为一个引用的开始

- `U+2019`

  右单引号标记为 (`’`)— 当单引用时座位一个引用的结束，也是撇号的首选字符。

当这三个字符出现在单词中间的时候， `标准分词器` 和 `icu_分词器` 都会将这三个字符视为撇号（这会被视为单词的一部分）。 然而还有另外三个长得很像撇号的字符：

- `U+201B`

  Single high-reversed-9 （高反单引号）标记为 (`‛`)— 跟 `U+2018` 一样，但是外观上有区别

- `U+0091`

  ISO-8859-1 中的左单引号 — 不会被用于 Unicode 中

- `U+0092`

  ISO-8859-1 中的右单引号 — 不会被用于 Unicode 中

`标准分词器` 和 `icu_分词器` 把这三个字符视为单词的分界线 — 一个将文本拆分为词汇单元的位置。不幸的是，一些出版社用 `U+201B` 作为名字的典型书写方式例如 `M‛coy` ， 第二个俩字符或许可以被你的文字处理软件打出来，这取决于这款软件的年纪。

即使在使用可以“接受”的引号标记时，一个用单引号书写的词 — `You’re` — 也和一个用撇号书写的词 — `You're` — 不一样，这意味着搜索其中的一个变体将会找不到另一个。

幸运的是，可以用 `mapping` 对这些混乱的字符进行分类， 该过滤器可以运行我们用另一个字符替换所有实例中的一个字符。这种情况下，我们可以简单的用 `U+0027` 替换所有的撇号变体：

```js
PUT /my_index
{
  "settings": {
    "analysis": {
      "char_filter": {                        <1>
        "quotes": {
          "type": "mapping",
          "mappings": [                       <2>
            "\\u0091=>\\u0027",
            "\\u0092=>\\u0027",
            "\\u2018=>\\u0027",
            "\\u2019=>\\u0027",
            "\\u201B=>\\u0027"
          ]
        }
      },
      "analyzer": {
        "quotes_analyzer": {
          "tokenizer":     "standard",
          "char_filter": [ "quotes" ]         <3>
        }
      }
    }
  }
}
```
>  ![img](assets/1.png)  我们自定义了一个 `char_filter` （字符过滤器）叫做 `quotes` ，提供所有撇号变体到简单撇号的映射。   
>  
>  ![img](assets/2.png)  为了更清晰，我们使用每个字符的 JSON Unicode 转义语句，当然我们也可以使用他们本身字符表示： `"‘=>'"` 。  
>  
>  ![img](assets/3.png)  我们用自定义的 `quotes` 字符过滤器创建一个新的分析器叫做 `quotes_analyzer` 。   

像以前一样，我们需要在创建了分析器后测试它：

```js
GET /my_index/_analyze?analyzer=quotes_analyzer
You’re my ‘favorite’ M‛Coy
```

这个例子返回如下词汇单元，其中所有的单词中的引号标记都被替换为了撇号： `You're`, `my`, `favorite`, `M'Coy` 。

投入更多的努力确保你的分词器接收到高质量的输入，你的搜索结果质量也将会更好。



## 归一化词元

把文本切割成词元(token)只是这项工作 的一半。为了让这些词元(token)更容易搜索, 这些词元(token)需要被 *归一化*(*normalization*)--这个过程会去除同一个词元(token)的无意义差别，例如大写和小写的差别。可能我们还需要去掉有意义的差别, 让 `esta`、`ésta` 和 `está` 都能用同一个词元(token)来搜索。你会用 `déjà vu` 来搜索，还是 `deja vu`?

这些都是语汇单元过滤器的工作。语汇单元过滤器接收来自分词器(tokenizer)的词元(token)流。还可以一起使用多个语汇单元过滤器，每一个都有自己特定的处理工作。每一个语汇单元过滤器都可以处理来自另一个语汇单元过滤器输出的单词流。



### 举个例子

用的最多的语汇单元过滤器(token filters)是 `lowercase` 过滤器，它的功能正和你期望的一样；它将每个词元(token)转换为小写形式：

```js
GET /_analyze?tokenizer=standard&filters=lowercase
The QUICK Brown FOX!                                  <1>
```
>  ![img](assets/1.png)  得到的词元(token)是 `the`, `quick`, `brown`, `fox`   

只要查询和检索的分析过程是一样的，不管用户搜索 `fox` 还是 `FOX` 都能得到一样的搜索结果。`lowercase`过滤器会将查询 `FOX` 的请求转换为查询 `fox` 的请求， `fox` 和我们在倒排索引中存储的是同一个词元(token)。

为了在分析过程中使用 token 过滤器 ，我们可以创建一个 `custom` 分析器 ：

```js
PUT /my_index
{
  "settings": {
    "analysis": {
      "analyzer": {
        "my_lowercaser": {
          "tokenizer": "standard",
          "filter":  [ "lowercase" ]
        }
      }
    }
  }
}
```

我们可以通过 `analyze` API 来验证:

```js
GET /my_index/_analyze?analyzer=my_lowercaser
The QUICK Brown FOX!                                   <1>
```
>  ![img](assets/1.png)  得到的词元是 `the`, `quick`, `brown`, `fox`   



### 如果有口音

英语用变音符号(例如 `´`, `^`, 和 `¨`) 来强调单词—例如 `rôle`, `déjà`, 和 `däis` —但是是否使用他们通常是可选的. 其他语言则通过变音符号来区分单词。当然，只是因为在你的索引中拼写正确的单词并不意味着用户将搜索正确的拼写。 去掉变音符号通常是有用的，让 `rôle` 对应 `role`, 或者反过来。 对于西方语言，可以用 `asciifolding` 字符过滤器来实现这个功能。 实际上，它不仅仅能去掉变音符号。它会把Unicode字符转化为ASCII来表示:

- `ß` ⇒ `ss`
- `æ` ⇒ `ae`
- `ł` ⇒ `l`
- `ɰ` ⇒ `m`
- `⁇` ⇒ `??`
- `❷` ⇒ `2`
- `⁶` ⇒ `6`

像 `lowercase` 过滤器一样, `asciifolding` 不需要任何配置，可以被 `custom` 分析器直接使用:

```js
PUT /my_index
{
  "settings": {
    "analysis": {
      "analyzer": {
        "folding": {
          "tokenizer": "standard",
          "filter":  [ "lowercase", "asciifolding" ]
        }
      }
    }
  }
}

GET /my_index?analyzer=folding
My œsophagus caused a débâcle                                 <1>
```
>  ![img](assets/1.png)  得到的词元 `my`, `oesophagus`, `caused`, `a`, `debacle`   

**保留原意**

理所当然的，去掉变音符号会丢失原意。 例如, 参考 这三个 西班牙单词:

- `esta`

  形容词 *this* 的阴性形式, 例如 *esta silla* (this chair) 和 *esta* (this one).

- `ésta`

  `esta` 的古代用法.

- `está`

  动词 *estar* (to be) 的第三人称形式, 例如 *está feliz* (he is happy).

通常我们会合并前两个形式的单词，而去区分和他们不相同的第三个形式的单词。类似的:

- `sé`

  动词 *saber* (to know) 的第一人称形式 例如 *Yo sé* (I know).

- `se`

  与许多动词使用的第三人称反身代词, 例如 *se sabe* (it is known).

不幸的是，没有简单的方法，去区分哪些词应该保留变音符号和哪些词应该去掉变音符号。而且很有可能，你的用户也不知道.

相反， 我们对文本做两次索引: 一次用原文形式，一次用去掉变音符号的形式 :

```js
PUT /my_index/_mapping/my_type
{
  "properties": {
    "title": {                               <1>
      "type":           "string",
      "analyzer":       "standard",
      "fields": {
        "folded": {                          <2>
          "type":       "string",
          "analyzer":   "folding"
        }
      }
    }
  }
}
```
>  ![img](assets/1.png)  在 `title` 字段用 `standard` 分析器，会保留原文的变音符号.   
>  
>  ![img](assets/2.png)  在 `title.folded` 字段用 `folding` 分析器，会去掉变音符号   

你可以使用 `analyze` API 分析 *Esta está loca* (This woman is crazy)这个句子，来验证字段映射:

```js
GET /my_index/_analyze?field=title                <1>
Esta está loca
 
GET /my_index/_analyze?field=title.folded         <2>
Esta está loca
```
>  ![img](assets/1.png)  得到的词元 `esta`, `está`, `loca`   
>  
>  ![img](assets/2.png)  得到的词元 `esta`, `esta`, `loca`   

可以用更多的文档来测试:

```js
PUT /my_index/my_type/1
{ "title": "Esta loca!" }

PUT /my_index/my_type/2
{ "title": "Está loca!" }
```

现在，我们可以通过联合所有的字段来搜索。在`multi_match`查询中通过 <<most-fields,`most_fields`mode>> 模式来联合所有字段的结果:

```js
GET /my_index/_search
{
  "query": {
    "multi_match": {
      "type":     "most_fields",
      "query":    "esta loca",
      "fields": [ "title", "title.folded" ]
    }
  }
}
```

通过 `validate-query` API 来执行这个查询可以帮助你理解查询是如何执行的:

```js
GET /my_index/_validate/query?explain
{
  "query": {
    "multi_match": {
      "type":     "most_fields",
      "query":    "está loca",
      "fields": [ "title", "title.folded" ]
    }
  }
}
```

`multi-match` 查询会搜索在 `title` 字段中原文形式的单词 (`está`)，和在 `title.folded` 字段中去掉变音符号形式的单词 `esta`:

```
(title:está        title:loca       )
(title.folded:esta title.folded:loca)
```

无论用户搜索的是 `esta` 还是 `está`; 两个文档都会被匹配，因为去掉变音符号形式的单词在 `title.folded`字段中。然而，只有原文形式的单词在 `title` 字段中。此额外匹配会把包含原文形式单词的文档排在结果列表前面。

我们用 `title.folded` 字段来 *扩大我们的网* (*widen the net*)来匹配更多的文档，然后用原文形式的 `title`字段来把关联度最高的文档排在最前面。在可以为了匹配数量牺牲文本原意的情况下，这个技术可以被用在任何分析器里。
>  ![提示](assets/tip.png)  `asciifolding` 过滤器有一个叫做 `preserve_original` 的选项可以让你这样来做索引 ，把词的原文词元(original token)和处理--折叠后的词元(folded token)放在同一个字段的同一个位置。开启了这个选项，结果会像这样:   

```
Position 1     Position 2
--------------------------
(ésta,esta)    loca
--------------------------
```

虽然这个是节约空间的好办法，但是也意味着没有办法再说“给我精确匹配的原文词元”(Give me an exact match on the original word)。包含去掉和不去掉变音符号的词元，会导致不可靠的相关性评分。

所以，正如我们这一章做的，把每个字段的不同形式分开到不同的字段会让索引更清晰。



### Unicode的世界  {#Unicode的世界}

当Elasticsearch在比较词元(token)的时候，它是进行字节(byte)级别的比较。 换句话说，如果两个词元(token)被判定为相同的话，他们必须是相同的字节(byte)组成的。然而，Unicode允许你用不同的字节来写相同的字符。

例如， *é* 和 *é* 的不同是什么？这取决于你问谁。对于Elasticsearch，第一个是由 `0xC3 0xA9` 这两个字节组成的，第二个是由 `0x65 0xCC 0x81` 这三个字节组成的。

对于Unicode，他们的差异和他们的怎么组成没有关系，所以他们是相同的。第一个是单个单词 `é` ，第二个是一个简单 `e` 和重音符 +´+。

如果你的数据有多个来源，就会有可能发生这种状况：因为相同的单词使用了不同的编码，导致一个形式的 `déjà` 不能和它的其他形式进行匹配。

幸运的是，这里就有解决办法。这里有4种Unicode *归一化形式* (*normalization forms*) : `nfc`, `nfd`, `nfkc`, `nfkd`，它们都把Unicode字符转换成对应标准格式，把所有的字符 进行字节(byte)级别的比较。

------

>  **Unicode归一化形式 (Normalization Forms)**
>
>  ```
>  _组合_ (_composed_) 模式—`nfc` 和 `nfkc`—用尽可能少的字节(byte)来代表字符。 ((("composed forms (Unicode normalization)"))) 所以用 `é` 来代表单个字母 `é` 。  _分解_ （_decomposed_） 模式—`nfd` and `nfkd`—用字符的每一部分来代表字符。所以 `é` 分解为 `e` 和 `´`。 ((("decomposed forms (Unicode normalization)")))
>  ```
>
>  *规范* (*canonical*) 模式—`nfc` 和 `nfd`&—把连字作为单个字符，例如 `ﬃ` 或者 `œ` 。 *兼容*(*compatibility*) 模式—`nfkc` 和 `nfkd`—将这些组合的字符分解成简单字符的等价物，例如： `f` + `f` + `i` 或者 `o` + `e`.
------

无论你选择哪一个归一化(normalization)模式，只要你的文本只用一种模式，那你的同一个词元(token)就会由相同的字节(byte)组成。例如，*兼容* (*compatibility*) 模式 可以用连词 `ﬃ` 的简化形式 `ffi`来进行对比。

你可以使用 `icu_normalizer` 语汇单元过滤器(token filters) 来保证你的所有词元(token)是相同模式：

```js
PUT /my_index
{
  "settings": {
    "analysis": {
      "filter": {
        "nfkc_normalizer": {                       <1>
          "type": "icu_normalizer",
          "name": "nfkc"
        }
      },
      "analyzer": {
        "my_normalizer": {
          "tokenizer": "icu_tokenizer",
          "filter":  [ "nfkc_normalizer" ]
        }
      }
    }
  }
}
```
>  ![img](assets/1.png)  用 `nfkc` 归一化(normalization)模式来归一化(Normalize)所有词元(token).   

>  ![提示](assets/tip.png)  包括刚才提到过的 `icu_normalizer` 语汇单元过滤器(token filters)在内，这里还有 `icu_normalizer` *字符* 过滤器(*character* filters)。虽然它和语汇单元过滤器做相同的工作，但是会在文本到达过滤器之前做。到底是用`standard` 过滤器，还是 `icu_tokenizer` 过滤器，其实并不重要。因为过滤器知道怎么来正确处理所有的模式。
>  
>  但是，如果你使用不同的分词器，例如： `ngram`, `edge_ngram`, 或者 `pattern` 分词器，那么在语汇单元过滤器(token filters)之前使用 `icu_normalizer` 字符过滤器就变得有意义了。  

通常来说，你不仅仅想要归一化(normalize)词元(token)的字节(byte)规则，还需要把他们转成小写字母。这个可以通过 `icu_normalizer` 和定制的归一化(normalization)的模式 `nfkc_cf` 来实现。下一节我们会具体讲这个。



### Unicode 大小写折叠  {#Unicode大小写折叠}

人类没有创造力的话就不会是人类， 而人类的语言就恰恰反映了这一点。

处理一个单词的大小写看起来是一个简单的任务，除非遇到需要处理多语言的情况。

那就举一个例子：转换小写德国单词 `ß`。把它转换成大写是 `SS`，然后在转换成小写就成了 `ss`。还有一个例子：转换希腊字母 `ς` (sigma, 在单词末尾使用)。把它转换成大写是 `Σ`，然后再转换成小写就成了 `σ`。

把词条小写的核心是让他们看起来更像，而不是更不像。在Unicode中，这个工作是大小写折叠(case folding)来完成的，而不是小写化(lowercasing)。 *大小写折叠_ (_Case folding*) 把单词转换到一种(通常是小写)形式，是让写法不会影响单词的比较，所以拼写不需要完全正确。

例如：单词 `ß`，已经是小写形式了，会被_折叠_(_folded_)成 `ss`。类似的小写的 `ς` 被折叠成 `σ`，这样的话，无论 `σ`， `ς`， 和 `Σ`出现在哪里， 他们就都可以比较了。

```
`icu_normalizer` 语汇单元过滤器默认的归一化(normalization)模式是 `nfkc_cf`。它像 `nfkc` 模式一样：
```

- *组合* (*Composes*) 字符用最短的字节来表示。
- 用 *兼容* (_compatibility_）模式，把像 `ﬃ` 的字符转换成简单的 `ffi`

但是，也会这样做：

- *大小写折叠_ (_Case-folds*) 字符成一种适合比较的形式

换句话说， `nfkc_cf`等价于 `lowercase` 语汇单元过滤器(token filters)，但是却适用于所有的语言。 *on-steroids* 等价于 `standard` 分析器，例如：

```js
PUT /my_index
{
  "settings": {
    "analysis": {
      "analyzer": {
        "my_lowercaser": {
          "tokenizer": "icu_tokenizer",
          "filter":  [ "icu_normalizer" ]      <1>
        }
      }
    }
  }
}
```
>  ![img](assets/1.png)  `icu_normalizer` 默认是 `nfkc_cf` 模式.   

我们来比较 `Weißkopfseeadler`和 `WEISSKOPFSEEADLER`(大写形式) 分别通过 `standard`分析器和我们的Unicode自识别(Unicode-aware)分析器处理得到的结果：

```js
GET /_analyze?analyzer=standard                     <1>
Weißkopfseeadler WEISSKOPFSEEADLER

GET /my_index/_analyze?analyzer=my_lowercaser       <2>
Weißkopfseeadler WEISSKOPFSEEADLER
```
>  ![img](assets/1.png)  得到的词元(token)是 `weißkopfseeadler`, `weisskopfseeadler`    
>  
>  ![img](assets/2.png)  得到的词元(token)是 `weisskopfseeadler`, `weisskopfseeadler```standard`分析器得到了两个不同且不可比较的词元(token)，而我们定制化的分析器得到了两个相同但是不符合原意的词元(token)。`     



### Unicode 字符折叠  {#Unicode 字符折叠}

```
在多语言((("Unicode", "character folding")))((("tokens", "normalizing", "Unicode character folding")))处理中，`lowercase` 语汇单元过滤器(token filters)是一个很好的开始。但是作为对比的话，也只是对于整个巴别塔的惊鸿一瞥。所以 <<asciifolding-token-filter,`asciifolding` token filter>> 需要更有效的Unicode _字符折叠_ (_character-folding_)工具来处理全世界的各种语言。((("asciifolding token filter")))
`icu_folding` 语汇单元过滤器(token filters) (provided by the <<icu-plugin,`icu` plug-in>>)的功能和 `asciifolding` 过滤器一样， ((("icu_folding token filter")))但是它扩展到了非ASCII编码的语言，例如：希腊语，希伯来语，汉语。它把这些语言都转换对应拉丁文字，甚至包含它们的各种各样的计数符号，象形符号和标点符号。
`icu_folding` 语汇单元过滤器(token filters)自动使用 `nfkc_cf` 模式来进行大小写折叠和Unicode归一化(normalization)，所以不需要使用 `icu_normalizer` ：
PUT /my_index
{
  "settings": {
    "analysis": {
      "analyzer": {
        "my_folder": {
          "tokenizer": "icu_tokenizer",
          "filter":  [ "icu_folding" ]
        }
      }
    }
  }
}

GET /my_index/_analyze?analyzer=my_folder
١٢٣٤٥                                              <1>
```
>  ![img](assets/1.png)  阿拉伯数字 `١٢٣٤٥` 被折叠成等价的拉丁数字: `12345`.   

如果你有指定的字符不想被折叠，你可以使用 [*UnicodeSet*](http://icu-project.org/apiref/icu4j/com/ibm/icu/text/UnicodeSet.html)(像字符的正则表达式) 来指定哪些Unicode才可以被折叠。例如：瑞典单词 `å`,`ä`, `ö`, `Å`, `Ä`, 和 `Ö` 不能被折叠，你就可以设定为： `[^åäöÅÄÖ]` (`^` 表示 *不包含*)。这样就会对于所有的Unicode字符生效。

```js
PUT /my_index
{
  "settings": {
    "analysis": {
      "filter": {
        "swedish_folding": {                                  <1>
          "type": "icu_folding",
          "unicodeSetFilter": "[^åäöÅÄÖ]"
        }
      },
      "analyzer": {
        "swedish_analyzer": {                                  <2>
          "tokenizer": "icu_tokenizer",
          "filter":  [ "swedish_folding", "lowercase" ]
        }
      }
    }
  }
}
```
>  ![img](assets/1.png)   `swedish_folding`语汇单元过滤器(token filters) 定制了 `icu_folding`语汇单元过滤器(token filters)来不处理那些大写和小写的瑞典单词。   
>  
>  ![img](assets/2.png)  `swedish` 分析器首先分词，然后用`swedish_folding`语汇单元过滤器来折叠单词，最后把他们走转换为小写，除了被排除在外的单词： ++Å++, `Ä`, 或者 `Ö`。  



### 排序和整理

本章到目前为止，我们已经了解了怎么以搜索为目的去规范化词汇单元。 本章节中要考虑的最终用例是字符串排序。

在 [字符串排序与多字段](https://www.elastic.co/guide/cn/elasticsearch/guide/current/multi-fields.html) （复数域）中，我们解释了 Elasticsearch 为什么不能在 `analyzed` （分析过）的字符串字段上排序，并演示了如何为同一个域创建 *复数域索引* ，其中 `analyzed` 域用来搜索， `not_analyzed` 域用来排序。

`analyzed` 域无法排序并不是因为使用了分析器，而是因为分析器将字符串拆分成了很多词汇单元，就像一个 *词汇袋* ，所以 Elasticsearch 不知道使用那一个词汇单元排序。

依赖于 `not_analyzed` 域来排序的话不是很灵活：这仅仅允许我们使用原始字符串这一确定的值排序。然而我们 *可以* 使用分析器来实现另外一种排序规则，只要你选择的分析器总是为每个字符串输出有且仅有一个的词汇单元。

**大小写敏感排序**

想象下我们有三个 `用户` 文档，文档的 `姓名` 域分别含有 `Boffey` 、 `BROWN` 和 `bailey` 。首先我们将使用在 [字符串排序与多字段](https://www.elastic.co/guide/cn/elasticsearch/guide/current/multi-fields.html) 中提到的技术，使用 `not_analyzed` 域来排序：

```js
PUT /my_index
{
  "mappings": {
    "user": {
      "properties": {
        "name": {                           <1>
          "type": "string",
          "fields": {
            "raw": {                        <2>
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
>  ![img](assets/1.png)  `analyzed` `name` 域用来搜索。   
>  
>  ![img](assets/2.png)  `not_analyzed` `name.raw` 域用来排序。   

我们可以索引一些文档用来测试排序：

```js
PUT /my_index/user/1
{ "name": "Boffey" }

PUT /my_index/user/2
{ "name": "BROWN" }

PUT /my_index/user/3
{ "name": "bailey" }

GET /my_index/user/_search?sort=name.raw
```

运行这个搜索请求将会返回这样的文档排序： `BROWN` 、 `Boffey` 、 `bailey` 。 这个是 *词典排序* 跟 *字符串排序* 相反。基本上就是大写字母开头的字节要比小写字母开头的字节权重低，所以这些姓名是按照最低值优先排序。

这可能对计算机是合理的，但是对人来说并不是那么合理，人们更期望这些姓名按照字母顺序排序，忽略大小写。为了实现这个，我们需要把每个姓名按照我们想要的排序的顺序索引。

换句话来说，我们需要一个能输出单个小写词汇单元的分析器：

```js
PUT /my_index
{
  "settings": {
    "analysis": {
      "analyzer": {
        "case_insensitive_sort": {
          "tokenizer": "keyword",          <1>
          "filter":  [ "lowercase" ]       <2>
        }
      }
    }
  }
}
```
>  ![img](assets/1.png)  `keyword` 分词器将输入的字符串原封不动的输出。  
>  
>  ![img](assets/2.png)   `lowercase` 分词过滤器将词汇单元转化为小写字母。   

使用 `大小写不敏感排序` 分析器替换后，现在我们可以将其用在我们的复数域：

```js
PUT /my_index/_mapping/user
{
  "properties": {
    "name": {
      "type": "string",
      "fields": {
        "lower_case_sort": {                    <1>
          "type":     "string",
          "analyzer": "case_insensitive_sort"
        }
      }
    }
  }
}

PUT /my_index/user/1
{ "name": "Boffey" }

PUT /my_index/user/2
{ "name": "BROWN" }

PUT /my_index/user/3
{ "name": "bailey" }

GET /my_index/user/_search?sort=name.lower_case_sort
```
>  ![img](assets/1.png)  `name.lower_case_sort` 域将会为我们提供大小写不敏感排序。   

运行这个搜索请求会得到我们想要的文档排序： `bailey` 、 `Boffey` 、 `BROWN` 。

但是这个顺序是正确的么？它符合我门的期望所以看起来像是正确的， 但我们的期望可能受到这个事实的影响：这本书是英文的，我们的例子中使用的所有字母都属于到英语字母表。

如果我们添加一个德语姓名 *Böhm* 会怎样呢？

现在我们的姓名会返回这样的排序： `bailey` 、 `Boffey` 、 `BROWN` 、 `Böhm` 。 `Böhm` 会排在 `BROWN` 后面的原因是这些单词依然是按照它们表现的字节值排序的。 `r` 所存储的字节为 `0x72` ，而 `ö` 存储的字节值为 `0xF6` ，所以 `Böhm` 排在最后。每个字符的字节值都是历史的意外。

显然，默认排序顺序对于除简单英语之外的任何事物都是无意义的。事实上，没有完全“正确”的排序规则。这完全取决于你使用的语言。

**语言之间的区别**

每门语言都有自己的排序规则，并且 有时候甚至有多种排序规则。 这里有几个例子，我们前一小节中的四个名字在不同的上下文中是怎么排序的：

- 英语： `bailey` 、 `boffey` 、 `böhm` 、 `brown`
- 德语： `bailey` 、 `boffey` 、 `böhm` 、 `brown`
- 德语电话簿： `bailey` 、 `böhm` 、 `boffey` 、 `brown`
- 瑞典语： `bailey`, `boffey`, `brown`, `böhm`
>  ![注意](assets/note.png)  德语电话簿将 `böhm` 放在 `boffey` 的原因是 `ö` 和 `oe` 在处理名字和地点的时候会被看成同义词，所以 `böhm` 在排序时像是被写成了 `boehm` 。  

**Unicode 归类算法**

归类是将文本按预定义顺序排序的过程。 *Unicode 归类算法* 或称为 UCA （参见[*www.unicode.org/reports/tr10*](http://www.unicode.org/reports/tr10/) ） 定义了一种将字符串按照在归类单元表中定义的顺序排序的方法（通常称为排序规则）。

UCA 还定义了 *默认 Unicode 排序规则元素表* 或称为 *DUCET* ， *DUCET* 为无论任何语言的所有 Unicode 字符定义了默认排序。如你所见，没有惟一一个正确的排序规则，所以 DUCET 让更少的人感到烦恼，且烦恼尽可能的小，但它还远不是解决所有排序烦恼的万能药。

而且，明显几乎每种语言都有 自己的排序规则。大多时候使用 DUCET 作为起点并且添加一些自定义规则用来处理每种语言的特性。

UCA 将字符串和排序规则作为输入，并输出二进制排序键。 将根据指定的排序规则对字符串集合进行排序转化为对其二进制排序键的简单比较。

**Unicode 排序**
>  ![提示](assets/tip.png)  本节中描述的方法可能会在未来版本的 Elasticsearch 中更改。请查看 [`icu` plugin](https://www.elastic.co/guide/cn/elasticsearch/guide/current/icu-plugin.html) 文档的最新信息。  

`icu_collation` 分词过滤器默认使用 DUCET 排序规则。这已经是对默认排序的改进了。想要使用 `icu_collation` 我们仅需要创建一个使用默认 `icu_collation` 过滤器的分析器：

```js
PUT /my_index
{
  "settings": {
    "analysis": {
      "analyzer": {
        "ducet_sort": {
          "tokenizer": "keyword",
          "filter": [ "icu_collation" ]      <1>
        }
      }
    }
  }
}
```
>  ![img](assets/1.png)  使用默认 DUCET 归类。   

通常，我们想要排序的字段就是我们想要搜索的字段， 因此我们使用与在 [大小写敏感排序](https://www.elastic.co/guide/cn/elasticsearch/guide/current/sorting-collations.html#case-insensitive-sorting) 中使用的相同的复数域方法：

```js
PUT /my_index/_mapping/user
{
  "properties": {
    "name": {
      "type": "string",
      "fields": {
        "sort": {
          "type": "string",
          "analyzer": "ducet_sort"
        }
      }
    }
  }
}
```

使用这个映射， `name.sort` 域将会含有一个仅用来排序的键。我们没有指定某种语言，所以它会默认会使用 [DUCET collation](https://www.elastic.co/guide/cn/elasticsearch/guide/current/sorting-collations.html#uca) 。

现在，我们可以重新索引我们的案例文档并测试排序：

```js
PUT /my_index/user/_bulk
{ "index": { "_id": 1 }}
{ "name": "Boffey" }
{ "index": { "_id": 2 }}
{ "name": "BROWN" }
{ "index": { "_id": 3 }}
{ "name": "bailey" }
{ "index": { "_id": 4 }}
{ "name": "Böhm" }

GET /my_index/user/_search?sort=name.sort
```
>  ![注意](assets/note.png)  注意，每个文档返回的 `sort` 键，在前面的例子中看起来像 `brown` 和 `böhm` ，现在看起来像天书： `ᖔ乏昫တ倈⠀\u0001` 。原因是 `icu_collation` 过滤器输出键 仅用于有效分类，不用于任何其他目的。  

运行这个搜索请求反问的文档排序为： `bailey` 、 `Boffey` 、 `Böhm` 、 `BROWN` 。这个排序对英语和德语来说都正确，这已经是一种进步，但是它对德语电话簿和瑞典语来说还不正确。下一步我们为不同的语言自定义映射。

**指定语言**

可以为特定的语言配置 使用归类表的 `icu_collation` 过滤器，例如一个国家特定版本的语言，或者像德语电话簿之类的子集。 这个可以按照如下所示通过 使用 `language` 、 `country` 、 和 `variant` 参数来创建自定义版本的分词过滤器：

- 英语

  `{ "language": "en" }`

- 德语

  `{ "language": "de" }`

- 奥地利德语

  `{ "language": "de", "country": "AT" }`

- 德语电话簿

  `{ "language": "de", "variant": "@collation=phonebook" }`
>  ![提示](assets/tip.png)  你可以在一下网址阅读更多的 ICU 本地支持： <http://userguide.icu-project.org/locale>.  

这个例子演示怎么创建德语电话簿排序规则：

```js
PUT /my_index
{
  "settings": {
    "number_of_shards": 1,
    "analysis": {
      "filter": {
        "german_phonebook": {                  <1>
          "type":     "icu_collation",
          "language": "de",
          "country":  "DE",
          "variant":  "@collation=phonebook"
        }
      },
      "analyzer": {
        "german_phonebook": {                  <2>
          "tokenizer": "keyword",
          "filter":  [ "german_phonebook" ]
        }
      }
    }
  },
  "mappings": {
    "user": {
      "properties": {
        "name": {
          "type": "string",
          "fields": {
            "sort": {                          <3>
              "type":     "string",
              "analyzer": "german_phonebook"
            }
          }
        }
      }
    }
  }
}
```
>  ![img](assets/1.png)  首先我们为德语电话薄创建一个自定义版本的 `icu_collation` 。  
>  
>  ![img](assets/2.png)  之后我们将其包装在自定义的分析器中。  
>  
>  ![img](assets/3.png)  并且为我们的 `name.sort` 域配置它。   

像我们之前那样重新索引并重新搜索：

```js
PUT /my_index/user/_bulk
{ "index": { "_id": 1 }}
{ "name": "Boffey" }
{ "index": { "_id": 2 }}
{ "name": "BROWN" }
{ "index": { "_id": 3 }}
{ "name": "bailey" }
{ "index": { "_id": 4 }}
{ "name": "Böhm" }

GET /my_index/user/_search?sort=name.sort
```

现在返回的文档排序为： `bailey` 、 `Böhm` 、 `Boffey` 、 `BROWN` 。在德语电话簿归类中， `Böhm` 等同于 `Boehm` ，所以排在 `Boffey` 前面。

**多排序规则**

每种语言都可以使用复数域 来支持对同一个域进行多规则排序：

```js
PUT /my_index/_mapping/_user
{
  "properties": {
    "name": {
      "type": "string",
      "fields": {
        "default": {
          "type":     "string",
          "analyzer": "ducet"                <1>
        },
        "french": {
          "type":     "string",
          "analyzer": "french"               <2>
        },
        "german": {
          "type":     "string",
          "analyzer": "german_phonebook"     <3>
        },
        "swedish": {
          "type":     "string",
          "analyzer": "swedish"              <4>
        }
      }
    }
  }
}
```
>  ![img](assets/1.png)  ![img](assets/2.png)  ![img](assets/3.png)  ![img](assets/4.png)   我们需要为每个排序规则创建相应的分析器。   

使用这个映射，只要按照 `name.french` 、 `name.german` 或 `name.swedish` 域排序，就可以为法语、德语和瑞典语用户正确的排序结果了。不支持的语言可以回退到使用 `name.default` 域，它使用 DUCET 排序顺序。

**自定义排序**

`icu_collation` 分词过滤器提供很多 选项，不止 `language` 、 `country` 、和 `variant` ，这些选项可以用于定制排序算法。可用的选项有以下作用：

- 忽略变音符号
- 顺序大写排先或排后，或忽略大小写
- 考虑或忽略标点符号和空白
- 将数字按字符串或数字值排序
- 自定义现有归类或定义自己的归类

这些选项的详细信息超出了本书的范围，更多的信息可以查询 [ICU plug-in documentation](https://github.com/elasticsearch/elasticsearch-analysis-icu) 和 [ICU project collation documentation](http://userguide.icu-project.org/collation/concepts) 。



## 将单词还原为词根

大多数语言的单词都可以 *词形变化* ，意味着 下列单词可以改变它们的形态用来表达不同的意思：

- *单复数变化* ： fox 、foxes
- *时态变化* ： pay 、 paid 、 paying
- *性别变化* ： waiter 、 waitress
- *动词人称变化* ： hear 、 hears
- *代词变化* ： I 、 me 、 my
- *不规则变化* ： ate 、 eaten
- *情景变化* ： so be it 、 were it so

虽然词形变化有助于表达，但它干扰了检索，一个单一的词根 *词义* （或意义）可能被很多不同的字母序列表达。 英语是一种弱词形变化语言（你可以忽略词形变化并且能得到合理的搜索结果），但是一些其他语言是高度词形变化的并且需要额外的工作来保证高质量的搜索结果。

*词干提取* 试图移除单词的变化形式之间的差别，从而达到将每个词都提取为它的词根形式。 例如 `foxes`可能被提取为词根 `fox` ，移除单数和复数之间的区别跟我们移除大小写之间的区别的方式是一样的。

单词的词根形式甚至有可能不是一个真的单词，单词 `jumping` 和 `jumpiness` 或许都会被提取词干为 `jumpi`。 这并没有什么问题--只要在索引时和搜索时产生相同的词项，搜索会正常的工作。

如果词干提取很容易的话，那只要一个插件就够了。不幸的是，词干提取 是一种遭受两种困扰的模糊的技术：词干弱提取和词干过度提取。

*词干弱提取* 就是无法将同样意思的单词缩减为同一个词根。例如， `jumped` 和 `jumps` 可能被提取为 `jump`， 但是 `jumping` 可能被提取为 `jumpi` 。弱词干提取会导致搜索时无法返回相关文档。

*词干过度提取* 就是无法将不同含义的单词分开。例如， `general` 和 `generate` 可能都被提取为 `gener` 。 词干过度提取会降低精准度：不相干的文档会在不需要他们返回的时候返回。

------

> **词形还原**
>
> 原词是一组相关词的规范形式，或词典形式 — `paying` 、 `paid` 和 `pays` 的原词是 `pay` 。 通常原词很像与其相关的词，但有时也不像 — `is` 、 `was` 、 `am` 和 `being` 的原词是 `be` 。
>
> 词形还原，很像词干提取，试图归类相关单词，但是它比词干提取先进一步的是它企图按单词的 *词义* ，或意义归类。 同样的单词可能表现出两种意思—例如， *wake* 可以表现为 *to wake up* 或 *a funeral* 。然而词形还原试图区分两个词的词义，词干提取却会将其混为一谈。
>
> 词形还原是一种更复杂和高资源消耗的过程，它需要理解单词出现的上下文来决定词的意思。实践中，词干提取似乎比词形还原更高效，且代价更低。

------

首先我们会讨论下两个 Elasticsearch 使用的经典词干提取器 — [词干提取算法](https://www.elastic.co/guide/cn/elasticsearch/guide/current/algorithmic-stemmers.html) 和 [字典词干提取器](https://www.elastic.co/guide/cn/elasticsearch/guide/current/dictionary-stemmers.html) — 并且在 [选择一个词干提取器](https://www.elastic.co/guide/cn/elasticsearch/guide/current/choosing-a-stemmer.html) 讨论了怎么根据你的需要选择合适的词干提取器。 最后将在 [控制词干提取](https://www.elastic.co/guide/cn/elasticsearch/guide/current/controlling-stemming.html) 和 [原形词干提取](https://www.elastic.co/guide/cn/elasticsearch/guide/current/stemming-in-situ.html) 中讨论如何裁剪词干提取。



### 词干提取算法

Elasticsearch 中的大部分 stemmers （词干提取器）是基于算法的，它们提供了一系列规则用于将一个词提取为它的词根形式，例如剥离复数词末尾的 `s` 或 `es` 。提取单词词干时并不需要知道该词的任何信息。

这些基于算法的 stemmers 优点是：可以作为插件使用，速度快，占用内存少，有规律的单词处理效果好。缺点是：没规律的单词例如 `be` 、 `are` 、和 `am` ，或 `mice` 和 `mouse` 效果不好。

最早的一个基于算法 的英文词干提取器是 Porter stemmer ，该英文词干提取器现在依然推荐使用。 Martin Porter 后来为了开发词干提取算法创建了 [Snowball language](http://snowball.tartarus.org/) 网站， 很多 Elasticsearch 中使用的词干提取器就是用 Snowball 语言写的。
>  ![提示](assets/tip.png)  [`kstem` token filter](https://www.elastic.co/guide/en/elasticsearch/reference/5.6/analysis-kstem-tokenfilter.html) 是一款合并了词干提取算法和内置词典的英语分词过滤器。为了避免模糊词不正确提取，这个词典包含一系列根词单词和特例单词。 `kstem` 分词过滤器相较于 Porter 词干提取器而言不那么激进。  

**使用基于算法的词干提取器**

你 可以使用 [`porter_stem`](https://www.elastic.co/guide/en/elasticsearch/reference/5.6/analysis-porterstem-tokenfilter.html) 词干提取器或直接使用 [`kstem`](https://www.elastic.co/guide/en/elasticsearch/reference/5.6/analysis-kstem-tokenfilter.html) 分词过滤器，或使用 [`snowball`](https://www.elastic.co/guide/en/elasticsearch/reference/5.6/analysis-snowball-tokenfilter.html) 分词过滤器创建一个具体语言的 Snowball 词干提取器。所有基于算法的词干提取器都暴露了用来接受 `语言` 参数的统一接口： [`stemmer` token filter](https://www.elastic.co/guide/en/elasticsearch/reference/5.6/analysis-stemmer-tokenfilter.html) 。

例如，假设你发现 `英语` 分析器使用的默认词干提取器太激进并且 你想使它不那么激进。首先应在 [language analyzers](https://www.elastic.co/guide/en/elasticsearch/reference/5.6/analysis-lang-analyzer.html) 查看 `英语` 分析器配置文件，配置文件展示如下：

```js
{
  "settings": {
    "analysis": {
      "filter": {
        "english_stop": {
          "type":       "stop",
          "stopwords":  "_english_"
        },
        "english_keywords": {
          "type":       "keyword_marker",          <1>
          "keywords":   []
        },
        "english_stemmer": {
          "type":       "stemmer",
          "language":   "english"                  <2>
        },
        "english_possessive_stemmer": {
          "type":       "stemmer",
          "language":   "possessive_english"       <3>
        }
      },
      "analyzer": {
        "english": {
          "tokenizer":  "standard",
          "filter": [
            "english_possessive_stemmer",
            "lowercase",
            "english_stop",
            "english_keywords",
            "english_stemmer"
          ]
        }
      }
    }
  }
}
```
>  ![img](assets/1.png)  `keyword_marker` 分词过滤器列出那些不用被词干提取的单词。这个过滤器默认情况下是一个空的列表。   
>  
>  ![img](assets/2.png)  ![img](assets/3.png)  `english` 分析器使用了两个词干提取器： `possessive_english` 词干提取器和 `english` 词干提取器。所有格词干提取器会在任何词传递到 `english_stop` 、 `english_keywords` 和 `english_stemmer` 之前去除 `'s` 。   

重新审视下现在的配置，添加上以下修改，我们可以把这份配置当作新分析器的基本配置：

- 修改 `english_stemmer` ，将 `english` （[`porter_stem`](https://www.elastic.co/guide/en/elasticsearch/reference/5.6/analysis-porterstem-tokenfilter.html) 分词过滤器的映射）替换为 `light_english`（非激进的 [`kstem`](https://www.elastic.co/guide/en/elasticsearch/reference/5.6/analysis-kstem-tokenfilter.html) 分词过滤器的映射）。
- 添加 [`asciifolding`](https://www.elastic.co/guide/cn/elasticsearch/guide/current/asciifolding-token-filter.html) 分词过滤器用以移除外语的附加符号。
- 移除 `keyword_marker` 分词过滤器，因为我们不需要它。（我们会在 [控制词干提取](https://www.elastic.co/guide/cn/elasticsearch/guide/current/controlling-stemming.html) 中详细讨论它）

新定义的分析器会像下面这样:

```js
PUT /my_index
{
  "settings": {
    "analysis": {
      "filter": {
        "english_stop": {
          "type":       "stop",
          "stopwords":  "_english_"
        },
        "light_english_stemmer": {
          "type":       "stemmer",
          "language":   "light_english"      <1>
        },
        "english_possessive_stemmer": {
          "type":       "stemmer",
          "language":   "possessive_english"
        }
      },
      "analyzer": {
        "english": {
          "tokenizer":  "standard",
          "filter": [
            "english_possessive_stemmer",
            "lowercase",
            "english_stop",
            "light_english_stemmer",        <2>
            "asciifolding"                  <3>
          ]
        }
      }
    }
  }
}
```
>  ![img](assets/1.png)  ![img](assets/2.png)  将 `english` 词干提取器替换为非激进的 `light_english` 词干提取器   
>  
>  ![img](assets/3.png)  添加 `asciifolding` 分词过滤器     



### 字典词干提取器

*字典词干提取器* 在工作机制上与 [算法化词干提取器](https://www.elastic.co/guide/cn/elasticsearch/guide/current/algorithmic-stemmers.html) 完全不同。 不同于应用一系列标准规则到每个词上，字典词干提取器只是简单地在字典里查找词。理论上可以给出比算法化词干提取器更好的结果。一个字典词干提取器应当可以：

- 返回不规则形式如 `feet` 和 `mice` 的正确词干
- 区分出词形相似但词义不同的情形，比如 `organ` and `organization`

实践中一个好的算法化词干提取器一般优于一个字典词干提取器。应该有以下两大原因：

- 字典质量

  一个字典词干提取器再好也就跟它的字典一样。 据牛津英语字典网站估计，英语包含大约75万个单词（包含变音变形词）。电脑上的大部分英语字典只包含其中的 10% 。词的含义随时光变迁。`mobility` 提取词干 `mobil` 先前可能讲得通，但现在合并进了手机可移动性的含义。字典需要保持最新，这是一项很耗时的任务。通常等到一个字典变得好用后，其中的部分内容已经过时。字典词干提取器对于字典中不存在的词无能为力。而一个基于算法的词干提取器，则会继续应用之前的相同规则，结果可能正确或错误。

- 大小与性能

  字典词干提取器需要加载所有词汇、 所有前缀，以及所有后缀到内存中。这会显著地消耗内存。找到一个词的正确词干，一般比算法化词干提取器的相同过程更加复杂。依赖于不同的字典质量，去除前后缀的过程可能会更加高效或低效。低效的情形可能会明显地拖慢整个词干提取过程。另一方面，算法化词干提取器通常更简单、轻量和快速。
>   ![提示](assets/tip.png)  如果你所使用的语言有比较好的算法化词干提取器，这通常是比一个基于字典的词干提取器更好的选择。对于算法化词干提取器效果比较差（或者压根没有）的语言，可以使用拼写检查（Hunspell）字典词干提取器，下一个章节会讨论。  



### Hunspell 词干提取器  {#Hunspell词干提取器}

Elasticsearch 提供了基于词典提取词干的 [`hunspell` 语汇单元过滤器（token filter）](https://www.elastic.co/guide/en/elasticsearch/reference/5.6/analysis-hunspell-tokenfilter.html). Hunspell [*hunspell.github.io*](http://hunspell.github.io/) 是一个 Open Office、LibreOffice、Chrome、Firefox、Thunderbird 等众多其它开源项目都在使用的拼写检查器。

可以从这里获取 Hunspell 词典 ：

- [*extensions.openoffice.org*](http://extensions.openoffice.org/): 下载解压 `.oxt` 后缀的文件。
- [*addons.mozilla.org*](http://mzl.la/157UORf): 下载解压 `.xpi` 扩展文件。
- [OpenOffice archive](http://download.services.openoffice.org/contrib/dictionaries/): 下载解压 `.zip` 文件。

一个 Hunspell 词典由两个文件组成 — 具有相同的文件名和两个不同的后缀 — 如 `en_US`—和下面的两个后缀的其中一个：

- `.dic`

  包含所有词根，采用字母顺序，再加上一个代表所有可能前缀和后缀的代码表 【集体称之为词缀( *affixes* 】

- `.aff`

  包含实际 `.dic` 文件每一行代码表对应的前缀和后缀转换

**安装一个词典**

Hunspell 语汇单元过滤器 在特定的 Hunspell 目录里寻找词典， 默认目录是 `./config/hunspell/` 。 `.dic` 文件和 `.aff` 文件应该要以子目录且按语言/区域的方式来命名。 例如，我们可以为美式英语创建一个 Hunspell 词干提取器，目录结构如下：

```text
config/
  └ hunspell/                    <1>
      └ en_US/                   <3>
          ├ en_US.dic
          ├ en_US.aff
          └ settings.yml         <3>
```
>  ![img](assets/1.png)  Hunspell 目录位置可以通过编辑 `config/elasticsearch.yml` 文件的：`indices.analysis.hunspell.dictionary.location` 设置来修改。  
>  
>  ![img](assets/2.png)   `en_US` 是这个区域的名字，也是我们传给 `hunspell` 语汇单元过滤器参数 `language` 值。   
>  
>  ![img](assets/3.png)   一个语言一个设置文件，下面的章节会具体介绍。   

**按语言设置**

在语言的目录设置文件 `settings.yml` 包含适用于所有字典内的语言目录的设置选项。

```yaml
---
ignore_case:          true
strict_affix_parsing: true
```

这些选项的意思如下：

- `ignore_case`

  Hunspell 目录默认是区分大小写的，如，姓氏 `Booker` 和名词 `booker` 是不同的词，所以应该分别进行词干提取。 也许让 `hunspell` 提取器区分大小写是一个好主意，不过也可能让事情变得复杂：一个句子的第一个词可能会被大写，因此感觉上会像是一个名词。输入的文本可能全是大写，如果这样那几乎一个词都找不到。用户也许会用小写来搜索名字，在这种情况下，大写开头的词将找不到。一般来说，设置参数 `ignore_case` 为 `true` 是一个好主意。

- `strict_affix_parsing`

  词典的质量千差万别。 一些网上的词典的 `.aff` 文件有很多畸形的规则。 默认情况下，如果 Lucene 不能正常解析一个词缀(affix)规则， 它会抛出一个异常。 你可以通过设置 `strict_affix_parsing` 为 `false` 来告诉 Lucene 忽略错误的规则。

------

**自定义词典**

>  如果一个目录放置了多个词典 (`.dic` 文件)， 他们会在加载时合并到一起。这可以让你以自定义的词典的方式对下载的词典进行定制：
>  
>  ```text
config/
  └ hunspell/
      └ en_US/  
          ├ en_US.dic
          ├ en_US.aff 
          ├ custom.dic
          └ settings.yml
>  ```
>  ![img](assets/1.png)  `custom` 词典和 `en_US` 词典将合并到一起。   
>  
>  ![img](assets/2.png)  多个 `.aff` 文件是不允许的，因为会产生规则冲突。  
>  
>  `.dic` 文件和 `.aff` 文件的格式在这里讨论： [Hunspell 词典格式](https://www.elastic.co/guide/cn/elasticsearch/guide/current/hunspell.html#hunspell-dictionary-format) 。  

------

**创建一个 Hunspell 语汇单元过滤器**

一旦你在所有节点上安装好了词典，你就能像这样定义一个 `hunspell` 语汇单元过滤器 ：

```json
PUT /my_index
{
  "settings": {
    "analysis": {
      "filter": {
        "en_US": {
          "type":     "hunspell",
          "language": "en_US"                     <1>
        }
      },
      "analyzer": {
        "en_US": {
          "tokenizer":  "standard",
          "filter":   [ "lowercase", "en_US" ]
        }
      }
    }
  }
}
```
>  ![img](assets/1.png)  参数 `language` 和目录下对应的名称相同。   

你可以通过 `analyze` API 来测试这个新的分析器， 然后和 `english` 分析器比较一下它们的输出：

```json
GET /my_index/_analyze?analyzer=en_US     <1>
reorganizes

GET /_analyze?analyzer=english            <2>
reorganizes
```
>  ![img](assets/1.png)   返回 `organize`   
>  
>  ![img](assets/2.png)   返回 `reorgan`   

在前面的例子中，`hunspell` 提取器有一个有意思的事情，它不仅能移除前缀还能移除后缀。大多数算法词干提取仅能移除后缀。
>  ![提示](assets/tip.png)  Hunspell 词典会占用几兆的内存。幸运的是，Elasticsearch 每个节点只会创建一个词典的单例。 所有的分片都会使用这个相同的 Hunspell 分析器。  

**Hunspell 词典格式**

尽管使用 `hunspell` 不必了解 Hunspell 词典的格式， 不过了解格式可以帮助我们编写自己的自定义的词典。其实很简单。

例如，在美式英语词典（US English dictionary），`en_US.dic` 文件包含了一个包含词 `analyze` 的实体，看起来如下：

```text
analyze/ADSG
```

`en_US.aff` 文件包含了一个针对标记 `A` 、 `G` 、`D` 和 `S` 的前后缀的规则。 其中应该只有一个能匹配，每一个规则的格式如下：

```text
[type] [flag] [letters to remove] [letters to add] [condition]
```

例如，下面的后缀 (`SFX`) 规则 `D` 。它是说，当一个词由一个辅音 (除了 `a` 、`e` 、`i` 、`o` 或 `u` 外的任意音节) 后接一个 `y` ，那么它可以移除 `y` 和添加 `ied` 结尾 （如，`ready` → `readied` ）。

```text
SFX    D      y   ied  [^aeiou]y
```

前面提到的 `A` 、 `G` 、`D` 和 `S` 标记对应规则如下：

```text
SFX D Y 4
SFX D   0     d          e                  <1>
SFX D   y     ied        [^aeiou]y
SFX D   0     ed         [^ey]
SFX D   0     ed         [aeiou]y

SFX S Y 4
SFX S   y     ies        [^aeiou]y
SFX S   0     s          [aeiou]y
SFX S   0     es         [sxzh]
SFX S   0     s          [^sxzhy]          <2>

SFX G Y 2
SFX G   e     ing        e                 <3>
SFX G   0     ing        [^e]

PFX A Y 1
PFX A   0     re         .                 <4>
```
>  ![img](assets/1.png)   `analyze` 以一个 `e` 结尾，所以它可以添加一个 `d` 变成 `analyzed` 。   
>  
>  ![img](assets/2.png)   `analyze` 不是由 `s` 、`x` 、`z` 、`h` 或 `y` 结尾，所以，它可以添加一个 `s` 变成 `analyzes` 。   
>  
>  ![img](assets/3.png)   `analyze` 以一个 `e` 结尾，所以，它可以移除 `e` 和添加 `ing` 然后变成 `analyzing` 。   
>  
>  ![img](assets/4.png)  可以添加前缀 `re` 来形成 `reanalyze` 。这个规则可以组合后缀规则一起形成： `reanalyzes` 、`reanalyzed` 、 `reanalyzing` 。  

了解更多关于 Hunspell 的语法，可以前往 [Hunspell 文档](http://sourceforge.net/projects/hunspell/files/Hunspell/Documentation/) 。



### 选择一个词干提取器

在文档 [`stemmer`](https://www.elastic.co/guide/en/elasticsearch/reference/5.6/analysis-stemmer-tokenfilter.html) token filter 里面列出了一些针对语言的若干词干提取器。 就英语来说我们有如下提取器：

- `english`

  [`porter_stem`](https://www.elastic.co/guide/en/elasticsearch/reference/5.6/analysis-porterstem-tokenfilter.html) 语汇单元过滤器（token filter）。

- `light_english`

  [`kstem`](https://www.elastic.co/guide/en/elasticsearch/reference/5.6/analysis-kstem-tokenfilter.html) 语汇单元过滤器（token filter）。

- `minimal_english`

  Lucene 里面的 `EnglishMinimalStemmer` ，用来移除复数。

- `lovins`

  基于 [Snowball](https://www.elastic.co/guide/en/elasticsearch/reference/5.6/analysis-snowball-tokenfilter.html) 的 [Lovins](http://snowball.tartarus.org/algorithms/lovins/stemmer.html) 提取器, 第一个词干提取器。

- `porter`

  基于 [Snowball](https://www.elastic.co/guide/en/elasticsearch/reference/5.6/analysis-snowball-tokenfilter.html) 的 [Porter](http://snowball.tartarus.org/algorithms/porter/stemmer.html) 提取器。

- `porter2`

  基于 [Snowball](https://www.elastic.co/guide/en/elasticsearch/reference/5.6/analysis-snowball-tokenfilter.html) 的 [Porter2](http://snowball.tartarus.org/algorithms/english/stemmer.html) 提取器。

- `possessive_english`

  Lucene 里面的 `EnglishPossessiveFilter` ，移除 `'s`

Hunspell 词干提取器也要纳入到上面的列表中，还有多种英文的词典可用。

有一点是可以肯定的：当一个问题存在多个解决方案的时候，这意味着没有一个解决方案充分解决这个问题。 这一点同样体现在词干提取上 — 每个提取器使用不同的方法不同程度的对单词进行了弱提取或是过度提取。

在 `stemmer` 文档 中，使用粗体高亮了每一个语言的推荐的词干提取器， 通常是因为它提供了一个在性能和质量之间合理的妥协。也就是说，推荐的词干提取器也许不适用所有场景。 关于哪个是最好的词干提取器，不存在一个唯一的正确答案 — 它要看你具体的需求。 这里有3个方面的因素需要考虑在内： 性能、质量、程度。

**提取性能**

算法提取器一般来说比 Hunspell 提取器快4到5倍。 “Handcrafted” 算法提取器通常（不是永远） 要比 Snowball 快或是差不多。 比如，`porter_stem` 语汇单元过滤器（token filter）就明显要比基于 Snowball 实现的 Porter 提取器要快的多。

Hunspell 提取器需要加载所有的词典、前缀和后缀表到内存，可能需要消耗几兆的内存。而算法提取器，由一点点代码组成，只需要使用很少内存。

**提取质量**

所有的语言，除了世界语（Esperanto）都是不规范的。 最日常用语使用的词往往不规则，而更正式的书面用语则往往遵循规律。 一些提取算法经过多年的开发和研究已经能够产生合理的高质量的结果了，其他人只需快速组装做很少的研究就能解决大部分的问题了。

虽然 Hunspell 提供了精确地处理不规则词语的承诺，但在实践中往往不足。 一个基于词典的提取器往往取决于词典的好坏。如果 Hunspell 碰到的这个词不在词典里，那它什么也不能做。 Hunspell 需要一个广泛的、高质量的、最新的词典以产生好的结果；这样级别的词典可谓少之又少。 另一方面，一个算法提取器，将愉快的处理新词而不用为新词重新设计算法。

如果一个好的算法词干提取器可用于你的语言，那明智的使用它而不是 Hunspell。它会更快并且消耗更少内存，并且会产生和通常一样好或者比 Hunspell 等价的结果.

如果精度和可定制性对你很重要，那么你需要（和有精力）来维护一个自定义的词典，那么 Hunspell 会给你比算法提取器更大的灵活性。 (查看 [控制词干提取](https://www.elastic.co/guide/cn/elasticsearch/guide/current/controlling-stemming.html) 来了解可用于任何词干提取器的自定义技术。)

**提取程度**

不同的词干提取器会将词弱提取或过度提取到一定的程度 。 `light_` 提取器提干力度不及标准的提取器。 `minimal_` 提取器同样也不那么积极。Hunspell 提取力度要激进一些。

是否想要积极提取还是轻量提取取决于你的场景。如果你的搜索结果是要用于聚类算法，你可能会希望匹配的更广泛一点（因此，提取力度要更大一点）。 如果你的搜索结果是面向最终用户，轻量的提取一般会产生更好的结果。对搜索来说，将名称和形容词提干比动词提干更重要，当然这也取决于语言。

另外一个要考虑的因素就是你的文档集的大小。 一个只有 10,000 个产品的小集合，你可能要更激进的提干来确保至少匹配到一些文档。 如果你的文档集很大，使用轻量的弱提取可能会得到更好的匹配结果。

**做一个选择**

从推荐的一个词干提取器出发，如果它工作的很好，那没有什么需要调整的。如果不是，你将需要花点时间来调查和比较该语言可用的各种不同提取器， 来找到最适合你目的的那一个。



### 控制词干提取

开箱即用的词干提取方案永远也不可能完美。 尤其是算法提取器，他们可以愉快的将规则应用于任何他们遇到的词，包含那些你希望保持独立的词。 也许，在你的场景，保持独立的 `skies` 和 `skiing` 是重要的，你不希望把他们提取为 `ski` （正如 `english` 分析器那样）。

语汇单元过滤器 [`keyword_marker`](https://www.elastic.co/guide/en/elasticsearch/reference/5.6/analysis-keyword-marker-tokenfilter.html) 和 [`stemmer_override`](https://www.elastic.co/guide/en/elasticsearch/reference/5.6/analysis-stemmer-override-tokenfilter.html) 能让我们自定义词干提取过程。

**阻止词干提取**

语言分析器（查看 [配置语言分析器](https://www.elastic.co/guide/cn/elasticsearch/guide/current/configuring-language-analyzers.html)）的参数 [`stem_exclusion`](https://www.elastic.co/guide/cn/elasticsearch/guide/current/configuring-language-analyzers.html#stem-exclusion) 允许我们指定一个词语列表，让他们不被词干提取。

在内部，这些语言分析器使用 [`keyword_marker` 语汇单元过滤器](https://www.elastic.co/guide/en/elasticsearch/reference/5.6/analysis-keyword-marker-tokenfilter.html) 来标记这些词语列表为 *keywords* ，用来阻止后续的词干提取过滤器来触碰这些词语。

例如，我们创建一个简单自定义分析器，使用 [`porter_stem`](https://www.elastic.co/guide/en/elasticsearch/reference/5.6/analysis-porterstem-tokenfilter.html) 语汇单元过滤器，同时阻止 `skies` 的词干提取：

```json
PUT /my_index
{
  "settings": {
    "analysis": {
      "filter": {
        "no_stem": {
          "type": "keyword_marker",
          "keywords": [ "skies" ]         <1>
        }
      },
      "analyzer": {
        "my_english": {
          "tokenizer": "standard",
          "filter": [
            "lowercase",
            "no_stem",
            "porter_stem"
          ]
        }
      }
    }
  }
}
```
>  ![img](assets/1.png)  参数 `keywords` 可以允许接收多个词语。 

使用 `analyze` API 来测试，可以看到词 `skies` 没有被提取：

```json
GET /my_index/_analyze?analyzer=my_english
sky skies skiing skis                            <1>
```
>  ![img](assets/1.png)  返回: `sky`, `skies`, `ski`, `ski`  
>  
>  ![提示](assets/tip.png)  虽然语言分析器只允许我们通过参数 `stem_exclusion` 指定一个词语列表来排除词干提取，不过 `keyword_marker` 语汇单元过滤器同样还接收一个 `keywords_path` 参数允许我们将所有的关键字存在一个文件。 这个文件应该是每行一个字，并且存在于集群的每个节点。查看 [更新停用词（Updating Stopwords）](https://www.elastic.co/guide/cn/elasticsearch/guide/current/using-stopwords.html#updating-stopwords) 了解更新这些文件的提示。  

**自定义提取**

在上面的例子中，我们阻止了 `skies` 被词干提取，但是也许我们希望他能被提干为 `sky` 。 The[`stemmer_override`](https://www.elastic.co/guide/en/elasticsearch/reference/5.6/analysis-stemmer-override-tokenfilter.html) 语汇单元过滤器允许我们指定自定义的提取规则。 与此同时，我们可以处理一些不规则的形式，如：`mice` 提取为 `mouse` 和 `feet` 到 `foot` ：

```json
PUT /my_index
{
  "settings": {
    "analysis": {
      "filter": {
        "custom_stem": {
          "type": "stemmer_override",
          "rules": [                        <1>
            "skies=>sky",
            "mice=>mouse",
            "feet=>foot"
          ]
        }
      },
      "analyzer": {
        "my_english": {
          "tokenizer": "standard",
          "filter": [
            "lowercase",
            "custom_stem",                   <2>
            "porter_stem"
          ]
        }
      }
    }
  }
}

GET /my_index/_analyze?analyzer=my_english
The mice came down from the skies and ran over my feet     <3>
```
>  ![img](assets/1.png)  规则来自 `original=>stem` 。    
>  
>  ![img](assets/2.png)  `stemmer_override` 过滤器必须放置在词干提取器之前。     
>  
>  ![img](assets/3.png)   返回 `the`, `mouse`, `came`, `down`, `from`, `the`, `sky`, `and`, `ran`, `over`, `my`, `foot` 。   
>  
>  ![提示](assets/tip.png)  正如 `keyword_marker` 语汇单元过滤器，规则可以被存放在一个文件中，通过参数 `rules_path` 来指定位置。  



### 原形词干提取

为了完整地 完成本章的内容，我们将讲解如何将已提取词干的词和原词索引到同一个字段中。举个例子，分析句子 *The quick foxes jumped* 将会得到以下词项：

```text
Pos 1: (the)
Pos 2: (quick)
Pos 3: (foxes,fox)      <1>
Pos 4: (jumped,jump)    <2>
```
>  ![img](assets/1.png)  ![img](assets/2.png)  已提取词干的形式和未提取词干的形式位于相同的位置。   

Warning：使用此方法前请先阅读 [原形词干提取是个好主意吗](https://www.elastic.co/guide/cn/elasticsearch/guide/current/stemming-in-situ.html#stemming-in-situ-good-idea) 。

为了归档词干提取出的 *原形* ，我们将使用 [`keyword_repeat`](http://www.elastic.co/guide/en/elasticsearch/reference/current/analysis-keyword-repeat-tokenfilter.html) 过滤器，跟 `keyword_marker` 过滤器 ( see [阻止词干提取](https://www.elastic.co/guide/cn/elasticsearch/guide/current/controlling-stemming.html#preventing-stemming) ) 一样，它把每一个词项都标记为关键词，以防止后续词干提取器对其修改。但是，它依然会在相同位置上重复词项，并且这个重复的词项 **是** 提取的词干。

单独使用 `keyword_repeat` token 过滤器将得到以下结果：

```text
Pos 1: (the,the)           <1>
Pos 2: (quick,quick)       <2>
Pos 3: (foxes,fox)
Pos 4: (jumped,jump)
```
>  ![img](assets/1.png)  ![img](assets/2.png)  提取词干前后的形式一样，所以只是不必要的重复。   

为了防止提取和未提取词干形式相同的词项中的无意义重复，我们增加了组合的 [`unique`](https://www.elastic.co/guide/en/elasticsearch/reference/5.6/analysis-unique-tokenfilter.html) 语汇单元过滤器：

```json
PUT /my_index
{
  "settings": {
    "analysis": {
      "filter": {
        "unique_stem": {
          "type": "unique",
          "only_on_same_position": true    <1>
        }
      },
      "analyzer": {
        "in_situ": {
          "tokenizer": "standard",
          "filter": [
            "lowercase",
            "keyword_repeat",              <2>
            "porter_stem",
            "unique_stem"                  <3>
          ]
        }
      }
    }
  }
}
```
>  ![img](assets/1.png)  设置 `unique` 类型语汇单元过滤器，是为了只有当重复语汇单元出现在相同位置时，移除它们。   
>  ![img](assets/2.png)  语汇单元过滤器必须出现在词干提取器之前。   
>  
>  ![img](assets/3.png)  `unique_stem` 过滤器是在词干提取器完成之后移除重复词项。   



**原形词干提取是个好主意吗**

用户喜欢 *原形* 词干提取这个主意：“如果我可以只用一个组合字段，为什么还要分别存一个未提取词干和已提取词干的字段呢？” 但这是一个好主意吗？答案一直都是否定的。因为有两个问题：

第一个问题是无法区分精准匹配和非精准匹配。本章中，我们看到了多义词经常会被展开成相同的词干词：`organs` 和 `organization` 都会被提取为 `organ` 。

在 [使用语言分析器](https://www.elastic.co/guide/cn/elasticsearch/guide/current/using-language-analyzers.html) 我们展示了如何整合一个已提取词干属性的查询(为了增加召回率)和一个未提取词干属性的查询（为了提升相关度）。 当提取和未提取词干的属性相互独立时，单个属性的贡献可以通过给其中一个属性增加boost值来优化(参见 [语句的优先级](https://www.elastic.co/guide/cn/elasticsearch/guide/current/multi-query-strings.html#prioritising-clauses) )。相反地，如果已提取和未提取词干的形式置于同一个属性，就没有办法来优化搜索结果了。

第二个问题是，必须搞清楚 相关度分值是否如何计算的。在 [什么是相关性?](https://www.elastic.co/guide/cn/elasticsearch/guide/current/relevance-intro.html) 我们解释了部分计算依赖于逆文档频率（IDF）—— 即一个词在索引库的所有文档中出现的频繁程度。 在一个包含文本 `jump jumped jumps` 的文档上使用原形词干提取，将得到下列词项：

```text
Pos 1: (jump)
Pos 2: (jumped,jump)
Pos 3: (jumps,jump)
```

`jumped` 和 `jumps` 各出现一次，所以有正确的IDF值；`jump` 出现了3次，作为一个搜索词项，与其他未提取词干的形式相比，这明显降低了它的IDF值。

基于这些原因，我们不推荐使用原形词干提取。


