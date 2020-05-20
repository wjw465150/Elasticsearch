# 前言

这个世界已然被数据淹没。多年来，我们系统间流转和产生的大量数据已让我们不知所措。 现有的技术都集中在如何解决数据仓库存储以及如何结构化这些数据。 这些看上去都挺美好，直到你实际需要基于这些数据实时做决策分析的时候才发现根本不是那么一回事。

Elasticsearch 是一个分布式、可扩展、实时的搜索与数据分析引擎。 它能从项目一开始就赋予你的数据以搜索、分析和探索的能力，这是通常没有预料到的。 它存在还因为原始数据如果只是躺在磁盘里面根本就毫无用处。

无论你是需要全文搜索，还是结构化数据的实时统计，或者两者结合，这本指南都能帮助你了解其中最基本的概念， 从最基本的操作开始学习 Elasticsearch。之后，我们还会逐渐开始探索更加高级的搜索技术，不断提升搜索体验来满足你的用户需求。

Elasticsearch 不仅仅只是全文搜索，我们还将介绍结构化搜索、数据分析、复杂的语言处理、地理位置和对象间关联关系等。 我们还将探讨如何给数据建模来充分利用 Elasticsearch 的水平伸缩性，以及在生产环境中如何配置和监视你的集群。



## 谁应该读这本书

这本书是写给任何想要把他们的数据拿来干活做点事情的人。不管你是新起一项目从头开始还是为了给遗留系统改造换血， Elasticsearch 都能够帮助你解决现有问题和开发新的功能，有些可能是你之前没有想到的功能。

这本书既适合初学者也适合有经验的用户。我们希望你有一定的编程基础，虽然不是必须的，但有用过 SQL 和关系数据库会更佳。 我们会从原理解释和基本概念出发，帮助新手在复杂的搜索世界里获得一个稳定的基础。

具有搜索背景的读者也会受益于这本书。有经验的用户将懂得其所熟悉搜索的概念在 Elasticsearch 是如何对应和具体实现的。 即使是高级用户，前面几个章节所包含的信息也是非常有用的。

最后，也许你是一名 DevOps，其他部门一直尽可能快的往 Elasticsearch 里面灌数据，而你是那个负责阻止 Elasticsearch 服务器起火的消防员。 只要用户在规则内行事，Elasticsearch 集群扩容相当轻松。不过你需要知道如何在进入生产环境前搭建一个稳定的集群，还能要在凌晨三点钟能识别出警告信号，以防止灾难发生。 前面几章你可能不太感兴趣，但这本书的最后一部分是非常重要的，包含所有你需要知道的用以避免系统崩溃的知识。



## 为什么我们要写这本书

我们写这本书，因为 Elasticsearch 需要更好的阐述。 现有的参考文档是优秀的 — 前提是你知道你在寻找什么。它假定你已经熟悉信息检索的概念、分布式系统原理、Query DSL 和许多其他相关的概念。

这本书没有这样的假设。它的目的是写一本即便是一个完全不懂的初学者（不管是搜索还是分布式系统）也能拿起它简单看完几章，就能开始搭建一个原型。

我们采取一种基于问题求解的方式：这是一个问题，我该怎么解决？ 如何对候选方案进行权衡取舍？我们从基础知识开始，循序渐进，每一章都建立在前一章之上，同时提供必要的实用案例和理论解释。

现有的参考文档解决了 *如何* 使用这些功能，我们希望这本书解决的是 *为什么* 和 *什么时候* 使用这些功能。


<a name="Elasticsearch版本"></a>
## Elasticsearch版本

本书的初始印刷版针对的是 Elasticsearch 1.4.0，不过我们一直在不断更新内容和完善示例 [本书的线上版本](https://www.elastic.co/guide/en/elasticsearch/guide/current/) 针对的是 Elasticsearch 2.x。

你可以访问这本书的 [GitHub 仓库](https://github.com/elastic/elasticsearch-definitive-guide/) 来追踪最新变化。



## 如何读这本书

Elasticsearch 做了很多努力和尝试来让复杂的事情变得简单，很大程度上来说 Elasticsearch 的成功来源于此。 换句话说，搜索以及分布式系统是非常复杂的，不过迟早你也需要掌握一些来充分利用 Elasticsearch。

恩，是有点复杂，但不是魔法。我们倾向于认为复杂系统如同神奇的黑盒子，能响应外部的咒语，但是通常里面的工作逻辑很简单。 理解了这些逻辑过程你就能驱散魔法，理解内在能够让你更加明确和清晰，而不是寄托于黑盒子做你想要做的。

这本权威指南不仅帮助你学习 Elasticsearch，而且带你接触更深入、更有趣的话题，如 [*集群内的原理*](https://www.elastic.co/guide/cn/elasticsearch/guide/current/distributed-cluster.html) 、 [*分布式文档存储*](https://www.elastic.co/guide/cn/elasticsearch/guide/current/distributed-docs.html) 、 [*执行分布式检索*](https://www.elastic.co/guide/cn/elasticsearch/guide/current/distributed-search.html) 和 [*分片内部原理*](https://www.elastic.co/guide/cn/elasticsearch/guide/current/inside-a-shard.html) ，这些虽然不是必要的阅读却能让你深入理解其内在机制。

本书的第一部分应该是在按章节顺序阅读，因为每一章建立在上一章的基础上（尽管你也可以浏览刚才提到的章节）。 后续各章节如 [*近似匹配*](https://www.elastic.co/guide/cn/elasticsearch/guide/current/proximity-matching.html) 和 [*部分匹配*](https://www.elastic.co/guide/cn/elasticsearch/guide/current/partial-matching.html) 相对独立，你可以根据需要选择性参阅。



## 本书导航

这本书分为七个部分：

- 章节 [*你知道的, 为了搜索…*](https://www.elastic.co/guide/cn/elasticsearch/guide/current/intro.html) 到 [*分片内部原理*](https://www.elastic.co/guide/cn/elasticsearch/guide/current/inside-a-shard.html) 主要是介绍 Elasticsearch。介绍了 Elasticsearch 的数据输入输出以及 Elasticsearch 如何处理你的文档数据。 如何进行基本的搜索操作和管理你的索引。 本章结束你将学会如何将 Elasticsearch 与你的应用程序集成。 章节：[*集群内的原理*](https://www.elastic.co/guide/cn/elasticsearch/guide/current/distributed-cluster.html)、[*分布式文档存储*](https://www.elastic.co/guide/cn/elasticsearch/guide/current/distributed-docs.html)、 [*执行分布式检索*](https://www.elastic.co/guide/cn/elasticsearch/guide/current/distributed-search.html) 和 [*分片内部原理*](https://www.elastic.co/guide/cn/elasticsearch/guide/current/inside-a-shard.html) 为附加章节，目的是让你了解分布式处理的过程，不是必读的。
- 章节 [*结构化搜索*](https://www.elastic.co/guide/cn/elasticsearch/guide/current/structured-search.html) 到 [*控制相关度*](https://www.elastic.co/guide/cn/elasticsearch/guide/current/controlling-relevance.html) 让你深入了解搜索，如何索引和查询你的数据，并借助一些更高级的特性，如邻近词（word proximity）和部分匹配（partial matching）。你将了解相关度评分是如何工作的以及如何控制它来确保第一页总是返回最佳的搜索结果。
- 章节 [*开始处理各种语言*](https://www.elastic.co/guide/cn/elasticsearch/guide/current/language-intro.html) 到 [*拼写错误*](https://www.elastic.co/guide/cn/elasticsearch/guide/current/fuzzy-matching.html) 解决如何有效使用分析器和查询来处理语言的头痛问题。我们会从一个简单的语言分析下手，然后逐步深入，如字母表和排序，还会涉及到词干提取、停用词、同义词和模糊匹配。
- 章节 [*高阶概念*](https://www.elastic.co/guide/cn/elasticsearch/guide/current/aggs-high-level.html) 到 [*Doc Values and Fielddata*](https://www.elastic.co/guide/cn/elasticsearch/guide/current/docvalues-and-fielddata.html) 讨论聚合（aggregations）和分析，对你的数据进行摘要化和分组来呈现总体趋势。
- 章节 [*地理坐标点*](https://www.elastic.co/guide/cn/elasticsearch/guide/current/geopoints.html) 到 [*地理形状*](https://www.elastic.co/guide/cn/elasticsearch/guide/current/geo-shapes.html) 介绍 Elasticsearch 支持的两种地理位置检索方式：经纬坐标点和复杂的地理形状（geo-shapes）。
- 章节 [*关联关系处理*](https://www.elastic.co/guide/cn/elasticsearch/guide/current/relations.html) 到 [*扩容设计*](https://www.elastic.co/guide/cn/elasticsearch/guide/current/scale.html) 谈到了如何为你的数据建模来高效使用 Elasticsearch。在搜索引擎里表达实体间的关系可能不是那么容易，因为它不是用来设计做这个的。这些章节还会阐述如何设计索引来匹配你系统中的数据流。
- 最后，章节 [*监控*](https://www.elastic.co/guide/cn/elasticsearch/guide/current/cluster-admin.html) 到 [*部署后*](https://www.elastic.co/guide/cn/elasticsearch/guide/current/post_deploy.html) 将讨论生产环境上线的重要配置、监控点以及如何诊断以避免出现问题。



## 在线资源

因为本书侧重如何在 Elasticsearch 里解决实际问题，而不是语法介绍，所以有时候你需要访问 [Elasticsearch 参考手册](https://www.elastic.co/guide/en/elasticsearch/reference/current/index.html) 来获取详细说明。 你可以访问以下网址获取最新的 Elasticsearch 参考手册和相关文档： <https://www.elastic.co/guide/>

如果你遇到本书或者参考手册没有收录到的问题，我们建议你访问 Elasticsearch 讨论社区来提问，学习别人是如何使用 Elasticsearch 的或者分享你自己的经验：

- [英文社区](https://discuss.elastic.co/c/elasticsearch/)
- [中文社区](http://elasticsearch.cn/)



## 本书协议约定

以下是本书中使用的印刷规范：

- *斜体*

  表示重点、新的术语或概念。

- `等宽字体`

  用于程序列表以及在段落中引用变量或程序元素如：函数名称、数据库、数据类型、环境变量、语句和关键字。

>  ![提示](assets/tip.png)   这个图标代表小贴士，建议。

>  ![注意](assets/note.png)  这个图标代表一般注意事项。

>  ![警告](assets/warning.png)  这个图标代表警告。



## 使用代码示例

本书的目的是为了帮你尽快能完成工作。一般来说，本书提供的示例代码你都可以用于你的程序或文档。 你不需要联系我们来询问许可，除非你打算复用相当大一部分代码。比如，写一个程序用了一段本书的代码不需要许可，但是销售或者是发行一张包含所有 O’Reilly 图书的示例代码的 CD 这个就需要许可。 引用这本书、引用示例代码来回答问题不需要许可，将大量的示例代码从这本书中包含到您的产品的文档中，这个需要许可。

关于署名出处，我们欣赏但不是必须。一个出处通常包含：书名、作者、出版商和 ISBN。如： *Elasticsearch: The Definitive Guide* by Clinton Gormley and Zachary Tong (O’Reilly). Copyright 2015 Elasticsearch BV, 978-1-449-35854-9。

如果你觉得你的示例代码使用超出合理使用或上面给出的许可，可随时与我们联系[permissions@oreilly.com](mailto:permissions@oreilly.com)。



## 鸣谢

为什么配偶总是被放到最后一个？但并非是说最不重要！ 在我们心中毫无疑问，有两个最值得我们感谢的人，他们是 Clinton 长期受苦的老婆和 Zach 的未婚妻。 他们照顾着我们和爱着我们，毫不懈怠，忍受我们的缺席和我们没完没了的抱怨这本书还要多久完成，最重要的是，她们依然还在我们身边。

感谢 Shay Banon 在最开始创建了 Elasticsearch，感谢 Elastic 公司支持本书的工作。 也非常感谢 Elastic 所有的同事，他们帮助我们透彻的了解 Elasticsearch 内部如何工作并且一直负责添加完善和修复与他们相关的部分。

其中两位同事特别值得一提：

- Robert Muir 耐心地分享了他的真知灼见，特别是 Lucene 搜索方面。有几章段落就是直接出自其智慧珠玑。
- Adrien Grand 深入到代码中回答问题，并检查我们的解释，以确保他们合理。

感谢 O’Reilly 承担这个项目和我们一起工作使这本书免费在线阅读，还有一直温柔哄骗我们的编辑 Brian Anderson 和善良而温柔的评论者 Benjamin Devèze、Ivan Brusic 和 Leo Lapworth。你们的鼓励，让我们充满希望。

感谢我们的读者，其中一些我们只有通过各自的 GitHub 才知道他们的身份，他们花时间报告问题、提供修正或提出改进建议：

Adam Canady, Adam Gray, Alexander Kahn, Alexander Reelsen, Alaattin Kahramanlar, Ambrose Ludd, Anna Beyer, Andrew Bramble, Baptiste Cabarrou, Bart Vandewoestyne, Bertrand Dechoux, Brian Wong, Brooke Babcock, Charles Mims, Chris Earle, Chris Gilmore, Christian Burgas, Colin Goodheart-Smithe, Corey Wright, Daniel Wiesmann, David Pilato, Duncan Angus Wilkie, Florian Hopf, Gavin Foo, Gilbert Chang, Grégoire Seux, Gustavo Alberola, Igal Sapir, Iskren Ivov Chernev, Itamar Syn-Hershko, Jan Forrest, Jānis Peisenieks, Japheth Thomson, Jeff Myers, Jeff Patti, Jeremy Falling, Jeremy Nguyen, J.R. Heard, Joe Fleming, Jonathan Page, Joshua Gourneau, Josh Schneier, Jun Ohtani, Keiji Yoshida, Kieren Johnstone, Kim Laplume, Kurt Hurtado, Laszlo Balogh, londocr, losar, Lucian Precup, Lukáš Vlček, Malibu Carl, Margirier Laurent, Martijn Dwars, Matt Ruzicka, Mattias Pfeiffer, Mehdy Amazigh, mhemani, Michael Bonfils, Michael Bruns, Michael Salmon, Michael Scharf , Mitar Milutinović, Mustafa K. Isik, Nathan Peck, Patrick Peschlow, Paul Schwarz, Pieter Coucke, Raphaël Flores, Robert Muir, Ruslan Zavacky, Sanglarsh Boudhh, Santiago Gaviria, Scott Wilkerson, Sebastian Kurfürst, Sergii Golubev, Serkan Kucukbay, Thierry Jossermoz, Thomas Cucchietti, Tom Christie, Ulf Reimers, Venkat Somula, Wei Zhu, Will Kahn-Greene 和 Yuri Bakumenko。

感谢所有参与本书的中文译者与审校人员，他们牺牲了大量宝贵的休息时间，他们对翻译内容仔细斟酌，一丝不苟， 对修改意见认真对待，各抒己见，不厌其烦的进行修改与再次审校，这些默默奉献的可爱的人分别是： [薛杰](http://github.com/xuej)，[骆朗](http://github.com/luotitan)，[彭秋源](http://github.com/pengqiuyuan)，[魏喆](http://github.com/richardwei2008)，[饶琛琳](http://github.com/chenryn)， [风虎](http://github.com/dajyaretakuya)，[路小磊](http://github.com/looly)，[michealzh](http://github.com/michealzh)，[nodexy](http://github.com/node)，[sdlyjzh](http://github.com/sdlyjzh)，[落英流离](http://github.com/wharstr9027)， [sunyonggang](http://github.com/sunyonggang)，[Singham](http://github.com/zhaochenxiao90)，[烧碱](http://github.com/Josephjin)，[龙翔](http://github.com/lephix)，[陈思](http://github.com/lephix)，[陈华](http://github.com/blogsit)， [追风侃侃](http://github.com/calm4wei)，[Geolem](http://github.com/Geolem)，[卷发](http://github.com/JessicaWon)，[kfypmqqw](http://github.com/kfypmqqw)，[袁伟强](http://github.com/weiqiangyuan)，[yichao](http://github.com/yichao2015)， [小彬](http://github.com/rockybean)，[leo](http://github.com/leo650)，[tangmisi](http://github.com/tangmisi)，[Alex](http://github.com/cdma)，[baifan](http://github.com/abia321)，[Evan](http://github.com/EvanYellow)，[fanyer](http://github.com/fanyer)， [wwb](http://github.com/Lywangwenbin)，[瑞星](http://github.com/luoruixing)，[刘碧琴](http://github.com/Miranda21)，[walker](http://github.com/weikuo0506)，[songgl](http://github.com/javasgl)， [吕兵](http://github.com/lvbabc)，[东](http://github.com/kankedong)，[杜宁](http://github.com/smilesfc)，[秦东亮](http://github.com/qindongliang)，[biyuhao](http://github.com/biyuhao)，[刘刚](http://github.com/LiuGangR)， [yumo](http://github.com/lxy4java)，[王秀文](http://github.com/wangxiuwen)，[zcola](http://github.com/zcola)，[gitqh](http://github.com/gitqh)，[blackoon](http://github.com/blackoon)，[David](http://github.com/davidmr_001)，[韩炳辰](http://github.com/stromdush)， [韩陆](http://github.com/feuyeux)，[echolihao](http://github.com/echolihao)，[Xargin](http://github.com/cch123)，[abel-sun](http://github.com/sunzhenya)，[卞顺强](http://github.com/AlixMu)， [bsll](http://github.com/bsll)，[冬狼](http://github.com/donglangdtstack)，[王琦](http://github.com/destinyfortune)，[Medcl](http://github.com/medcl)。



