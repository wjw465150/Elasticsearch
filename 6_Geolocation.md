# 地理位置



我们拿着纸质地图漫步城市的日子一去不返了。得益于智能手机，我们现在总是可以知道 自己所处的准确位置，也预料到网站会使用这些信息。我想知道从当前位置步行 5 分钟内可到的那些餐馆，对伦敦更大范围内的其他餐馆并不感兴趣。

但地理位置功能仅仅是 Elasticsearch 的冰山一角，Elasticsearch 的妙处在于，它让你可以把地理位置、全文搜索、结构化搜索和分析结合到一起。

例如：告诉我提到 *vitello tonnato* 这种食物、步行 5 分钟内可到、且晚上 11 点还营业的餐厅，然后结合用户评价、距离、价格排序。另一个例子：给我展示一幅整个城市8月份可用假期出租物业的地图，并计算出每个区域的平均价格。

Elasticsearch 提供了 两种表示地理位置的方式：用纬度－经度表示的坐标点使用 `geo_point` 字段类型，以 [GeoJSON](http://en.wikipedia.org/wiki/GeoJSON) 格式定义的复杂地理形状，使用 `geo_shape` 字段类型。

*Geo-points* 允许你找到距离另一个坐标点一定范围内的坐标点、计算出两点之间的距离来排序或进行相关性打分、或者聚合到显示在地图上的一个网格。另一方面，*Geo-shapes* 纯粹是用来过滤的。它们可以用来判断两个地理形状是否有重合或者某个地理形状是否完全包含了其他地理形状。



## 地理坐标点

*地理坐标点* 是指地球表面可以用经纬度描述的一个点。 地理坐标点可以用来计算两个坐标间的距离，还可以判断一个坐标是否在一个区域中，或在聚合中。

地理坐标点不能被动态映射 （[dynamic mapping](https://www.elastic.co/guide/cn/elasticsearch/guide/current/dynamic-mapping.html)）自动检测，而是需要显式声明对应字段类型为 `geo-point` ：

```json
PUT /attractions
{
  "mappings": {
    "restaurant": {
      "properties": {
        "name": {
          "type": "string"
        },
        "location": {
          "type": "geo_point"
        }
      }
    }
  }
}
```



### 经纬度坐标格式

如上例，`location` 字段被声明为 `geo_point` 后，我们就可以索引包含了经纬度信息的文档了。 经纬度信息的形式可以是字符串、数组或者对象：

```json
PUT /attractions/restaurant/1
{
  "name":     "Chipotle Mexican Grill",
  "location": "40.715, -74.011"                   <1>
}

PUT /attractions/restaurant/2
{
  "name":     "Pala Pizza",
  "location": {                                   <2>
    "lat":     40.722,
    "lon":    -73.989
  }
}

PUT /attractions/restaurant/3
{
  "name":     "Mini Munchies Pizza",
  "location": [ -73.983, 40.719 ]                 <3>
}
```
>  ![img](assets/1.png)  字符串形式以半角逗号分割，如 `"lat,lon"` 。 
>  
>  ![img](assets/2.png)  对象形式显式命名为 `lat` 和 `lon` 。    
>  
>  ![img](assets/3.png)  数组形式表示为 `[lon,lat]` 。           


>  ![小心](assets/caution.png)  可能所有人都至少一次踩过这个坑：地理坐标点用字符串形式表示时是纬度在前，经度在后（ `"latitude,longitude"` ），而数组形式表示时是经度在前，纬度在后（ `[longitude,latitude]` ）—顺序刚好相反。  

其实，在 Elasticesearch 内部，不管字符串形式还是数组形式，都是经度在前，纬度在后。不过早期为了适配 GeoJSON 的格式规范，调整了数组形式的表示方式。

因此，在使用地理位置的路上就出现了这么一个“捕熊器”，专坑那些不了解这个陷阱的使用者。



### 通过地理坐标点过滤

有四种地理坐标点相关的过滤器 可以用来选中或者排除文档：

- [`geo_bounding_box`](https://www.elastic.co/guide/cn/elasticsearch/guide/current/geo-bounding-box.html)

  找出落在指定矩形框中的点。

- [`geo_distance`](https://www.elastic.co/guide/cn/elasticsearch/guide/current/geo-distance.html)

  找出与指定位置在给定距离内的点。

- [`geo_distance_range`](https://www.elastic.co/guide/cn/elasticsearch/guide/current/geo-distance.html#geo-distance-range)

  找出与指定点距离在给定最小距离和最大距离之间的点。

- `geo_polygon`

  找出落在多边形中的点。 *这个过滤器使用代价很大* 。当你觉得自己需要使用它，最好先看看 [geo-shapes](https://www.elastic.co/guide/cn/elasticsearch/guide/current/geo-shapes.html)。

这些过滤器判断点是否落在指定区域时的计算方法稍有不同，但过程类似。指定的区域被转换成一系列以quad/geohash为前缀的tokens，并被用来在倒排索引中搜索拥有相同tokens的文档。
>  ![提示](assets/tip.png)  地理坐标过滤器使用代价昂贵 — 所以最好在文档集合尽可能少的场景下使用。你可以先使用那些简单快捷的过滤器，比如 `term` 或 `range` ，来过滤掉尽可能多的文档，最后才交给地理坐标过滤器处理。  

布尔型过滤器 [`bool` filter](https://www.elastic.co/guide/cn/elasticsearch/guide/current/combining-filters.html#bool-filter) 会自动帮你做这件事。 它会优先让那些基于“bitset”的简单过滤器(见 [关于缓存](https://www.elastic.co/guide/cn/elasticsearch/guide/current/filter-caching.html) )来过滤掉尽可能多的文档，然后依次才是更昂贵的地理坐标过滤器或者脚本类的过滤器。



### 地理坐标盒模型过滤器

这是目前为止最有效的地理坐标过滤器了，因为它计算起来非常简单。 你指定一个矩形的 `顶部` , `底部` , `左边界` ，和 `右边界` ，然后过滤器只需判断坐标的经度是否在左右边界之间，纬度是否在上下边界之间：

```json
GET /attractions/restaurant/_search
{
  "query": {
    "filtered": {
      "filter": {
        "geo_bounding_box": {
          "location": {               <1>
            "top_left": {
              "lat":  40.8,
              "lon": -74.0
            },
            "bottom_right": {
              "lat":  40.7,
              "lon": -73.0
            }
          }
        }
      }
    }
  }
}
```
>  ![img](assets/1.png)  这些坐标也可以用 `bottom_left` 和 `top_right` 来表示。   



**优化盒模型**

`地理坐标盒模型过滤器` 不需要把所有坐标点都加载到内存里。 因为它要做的 只是简单判断 `lat` 和 `lon` 坐标数值是否在给定的范围内，可以用倒排索引做一个 `range` 过滤来实现目标。

要使用这种优化方式，需要把 `geo_point` 字段 用 `lat` 和 `lon` 的方式分别映射到索引中：

```json
PUT /attractions
{
  "mappings": {
    "restaurant": {
      "properties": {
        "name": {
          "type": "string"
        },
        "location": {
          "type":    "geo_point",
          "lat_lon": true                <1>
        }
      }
    }
  }
}
```
>  ![img](assets/1.png)  `location.lat` 和 `location.lon` 字段将被分别索引。它们可以被用于检索，但是不会在检索结果中返回。   

然后，查询时你需要告诉 Elasticesearch 使用已索引的 `lat` 和 `lon` ：

```json
GET /attractions/restaurant/_search
{
  "query": {
    "filtered": {
      "filter": {
        "geo_bounding_box": {
          "type":    "indexed",              <1>
          "location": {
            "top_left": {
              "lat":  40.8,
              "lon": -74.0
            },
            "bottom_right": {
              "lat":  40.7,
              "lon":  -73.0
            }
          }
        }
      }
    }
  }
}
```
>  ![img](assets/1.png)  设置 `type` 参数为 `indexed` （替代默认值 `memory` ）来明确告诉 Elasticsearch 对这个过滤器使用倒排索引。   


>  ![小心](assets/caution.png)  `geo_point` 类型的字段可以包含多个地理坐标点，但是针对经度纬度分别索引的这种优化方式只对包含单个坐标点的字段有效。



### 地理距离过滤器

地理距离过滤器（ `geo_distance` ）以给定位置为圆心画一个圆，来找出那些地理坐标落在其中的文档 ：

```json
GET /attractions/restaurant/_search
{
  "query": {
    "filtered": {
      "filter": {
        "geo_distance": {
          "distance": "1km",        <1>
          "location": {             <2>
            "lat":  40.715,
            "lon": -73.988
          }
        }
      }
    }
  }
}
```
>  ![img](assets/1.png)  找出所有与指定点距离在 `1km` 内的 `location` 字段。访问 [Distance Units](https://www.elastic.co/guide/en/elasticsearch/reference/5.6/common-options.html#distance-units) 查看所支持的距离表示单位。   
>  
>  ![img](assets/2.png)  中心点可以表示为字符串，数组或者（如示例中的）对象。详见 [经纬度坐标格式](https://www.elastic.co/guide/cn/elasticsearch/guide/current/lat-lon-formats.html)。 

地理距离过滤器计算代价昂贵。为了优化性能，Elasticsearch 先画一个矩形框来围住整个圆形，这样就可以先用消耗较少的盒模型计算方式来排除掉尽可能多的文档。 然后只对落在盒模型内的这部分点用地理距离计算方式处理。
>  ![提示](assets/tip.png)  你需要判断你的用户，是否需要如此精确的使用圆模型来做距离过滤？ 通常使用矩形模型 [bounding box](https://www.elastic.co/guide/cn/elasticsearch/guide/current/geo-bounding-box.html) 是比地理距离更高效的方式，并且往往也能满足应用需求。  



**更快的地理距离计算**

两点间的距离计算，有多种牺牲性能换取精度的算法：

- `arc`

  最慢但最精确的是 `arc` 计算方式，这种方式把世界当作球体来处理。不过这种方式的精度有限，因为这个世界并不是完全的球体。

- `plane`

  `plane` 计算方式把地球当成是平坦的，这种方式快一些但是精度略逊。在赤道附近的位置精度最好，而靠近两极则变差。

- `sloppy_arc`

  如此命名，是因为它使用了 Lucene 的 `SloppyMath` 类。这是一种用精度换取速度的计算方式， 它使用 [Haversine formula](http://en.wikipedia.org/wiki/Haversine_formula) 来计算距离。它比 `arc` 计算方式快 4 到 5 倍，并且距离精度达 99.9%。这也是默认的计算方式。

你可以参考下例来指定不同的计算方式：

```json
GET /attractions/restaurant/_search
{
  "query": {
    "filtered": {
      "filter": {
        "geo_distance": {
          "distance":      "1km",
          "distance_type": "plane",       <1>
          "location": {
            "lat":  40.715,
            "lon": -73.988
          }
        }
      }
    }
  }
}
```
>  ![img](assets/1.png)  使用更快但精度稍差的 `plane` 计算方法。   


>  ![提示](assets/tip.png)  你的用户真的会在意一个餐馆落在指定圆形区域数米之外吗？一些地理位置相关的应用会有较高的精度要求；但大部分实际应用场景中，使用精度较低但响应更快的计算方式可能更好。  



**地理距离区间过滤器**

`geo_distance` 和 `geo_distance_range` 过滤器 的唯一差别在于后者是一个环状的，它会排除掉落在内圈中的那部分文档。

指定到中心点的距离也可以换一种表示方式：指定一个最小距离（使用 `gt` 或者 `gte` ）和最大距离（使用 `lt` 和 `lte` ），就像使用 `range` 过滤器一样：

```json
GET /attractions/restaurant/_search
{
  "query": {
    "filtered": {
      "filter": {
        "geo_distance_range": {
          "gte":    "1km",          <1>
          "lt":     "2km",          <2>
          "location": {
            "lat":  40.715,
            "lon": -73.988
          }
        }
      }
    }
  }
}
```
>  ![img](assets/1.png)  ![img](assets/2.png)  匹配那些距离中心点大于等于 `1km` 而小于 `2km` 的位置。   



### 按距离排序

检索结果可以按与指定点的距离排序 ：
>  ![提示](assets/tip.png)  当你 *可以* 按距离排序时， [按距离打分](https://www.elastic.co/guide/cn/elasticsearch/guide/current/sorting-by-distance.html#scoring-by-distance) 通常是一个更好的解决方案。  

```json
GET /attractions/restaurant/_search
{
  "query": {
    "filtered": {
      "filter": {
        "geo_bounding_box": {
          "type":       "indexed",
          "location": {
            "top_left": {
              "lat":  40.8,
              "lon": -74.0
            },
            "bottom_right": {
              "lat":  40.4,
              "lon": -73.0
            }
          }
        }
      }
    }
  },
  "sort": [
    {
      "_geo_distance": {
        "location": {                <1>
          "lat":  40.715,
          "lon": -73.998
        },
        "order":         "asc",
        "unit":          "km",       <2>
        "distance_type": "plane"     <3>
      }
    }
  ]
}
```
>  ![img](assets/1.png)  计算每个文档中 `location` 字段与指定的 `lat/lon` 点间的距离。   
>  
>  ![img](assets/2.png)  将距离以 `km` 为单位写入到每个返回结果的 `sort` 键中。  
>  
>  ![img](assets/3.png)  使用快速但精度略差的 `plane` 计算方式。  

你可能想问：为什么要制定距离的 `单位` 呢？用于排序的话，我们并不关心比较距离的尺度是英里、公里还是光年。 原因是，这个用于排序的值会设置在每个返回结果的 `sort` 元素中。

```json
...
  "hits": [
     {
        "_index": "attractions",
        "_type": "restaurant",
        "_id": "2",
        "_score": null,
        "_source": {
           "name": "New Malaysia",
           "location": {
              "lat": 40.715,
              "lon": -73.997
           }
        },
        "sort": [
           0.08425653647614346           <1>
        ]
     },
...
```
>  ![img](assets/1.png)  餐厅到我们指定的位置距离是 0.084km。  

你可以通过设置 `单位` （ `unit` ）来让返回值的形式，匹配你应用中需要的。

>  ![提示](assets/tip.png)  地理距离排序可以对多个坐标点来使用，不管（这些坐标点）是在文档中还是排序参数中。使用 `sort_mode` 来指定是否需要使用位置集合的 `最小` （ `min` ） `最大` （ `max` ）或者 `平均`（ `avg` ）距离。 如此就可以返回 “离我的工作地和家最近的朋友” 这样的结果了。  



**按距离打分**

有可能距离是决定返回结果排序的唯一重要因素，不过更常见的情况是距离会和其它因素，比如全文检索匹配度、流行程度或者价格一起决定排序结果。

遇到这种场景你需要在 [功能评分查询](https://www.elastic.co/guide/cn/elasticsearch/guide/current/function-score-query.html) 中指定方式让我们把这些因子处理后得到一个综合分。 [越近越好](https://www.elastic.co/guide/cn/elasticsearch/guide/current/decay-functions.html) 中有个一个例子就是介绍地理距离影响排序得分的。

另外按距离排序还有个缺点就是性能：需要对每一个匹配到的文档都进行距离计算。而 `function_score` 查询，在 [`rescore` 语句](https://www.elastic.co/guide/cn/elasticsearch/guide/current/_Improving_Performance.html#rescore-api) 中可以限制只对前 *n* 个结果进行计算。



## Geohashes  {#Geohashes6}

[Geohashes](http://en.wikipedia.org/wiki/Geohash) 是一种将经纬度坐标（ `lat/lon` ）编码成字符串的方式。 这么做的初衷只是为了让地理位置在 url 上呈现的形式更加友好，但现在 geohashes 已经变成一种在数据库中有效索引地理坐标点和地理形状的方式。

Geohashes 把整个世界分为 32 个单元的格子 —— 4 行 8 列 —— 每一个格子都用一个字母或者数字标识。比如 `g` 这个单元覆盖了半个格林兰，冰岛的全部和大不列颠的大部分。每一个单元还可以进一步被分解成新的 32 个单元，这些单元又可以继续被分解成 32 个更小的单元，不断重复下去。 `gc` 这个单元覆盖了爱尔兰和英格兰， `gcp` 覆盖了伦敦的大部分和部分南英格兰， `gcpuuz94k` 是白金汉宫的入口，精确到约 5 米。

换句话说， geohash 的长度越长，它的精度就越高。如果两个 geohashes 有一个共同的前缀— `gcpuuz`—就表示他们挨得很近。共同的前缀越长，距离就越近。

这也意味着，两个刚好相邻的位置，可能会有完全不同的 geohash 。比如，伦敦 [Millenium Dome](http://en.wikipedia.org/wiki/Millennium_Dome) 的 geohash 是 `u10hbp` ，因为它落在了 `u` 这个单元里，而紧挨着它东边的最大的单元是 `g` 。

地理坐标点可以自动索引相关的 geohashes ，更重要的是，他们也可以索引所有的 geohashes *前缀* 。如索引白金汉宫入口位置——纬度 `51.501568` ，经度 `-0.141257`—将会索引下面表格中列出的所有 geohashes ，表格中也给出了各个 geohash 单元的近似尺寸：

| Geohash        | Level | Dimensions          |
| -------------- | ----- | ------------------- |
| `g`            | `1`   | ~ 5,004km x 5,004km |
| `gc`           | `2`   | ~ 1,251km x 625km   |
| `gcp`          | `3`   | ~ 156km x 156km     |
| `gcpu`         | `4`   | ~ 39km x 19.5km     |
| `gcpuu`        | `5`   | ~ 4.9km x 4.9km     |
| `gcpuuz`       | `6`   | ~ 1.2km x 0.61km    |
| `gcpuuz9`      | `7`   | ~ 152.8m x 152.8m   |
| `gcpuuz94`     | `8`   | ~ 38.2m x 19.1m     |
| `gcpuuz94k`    | `9`   | ~ 4.78m x 4.78m     |
| `gcpuuz94kk`   | `10`  | ~ 1.19m x 0.60m     |
| `gcpuuz94kkp`  | `11`  | ~ 14.9cm x 14.9cm   |
| `gcpuuz94kkp5` | `12`  | ~ 3.7cm x 1.8cm     |

[`geohash单元` 过滤器](https://www.elastic.co/guide/en/elasticsearch/reference/2.4/query-dsl-geohash-cell-query.html) 可以使用这些 geohash 前缀 来找出与指定坐标点（ `lat/lon` ）相邻的位置。



### Geohashes 映射  {#Geohashes映射}

首先，你需要决定使用什么样的精度。 虽然你也可以使用 12 级的精度来索引所有的地理坐标点，但是你真的需要精确到数厘米吗？如果你把精度控制在一个实际一些的值，比如 `1km` ，那么你可以节省大量的索引空间：

```json
PUT /attractions
{
  "mappings": {
    "restaurant": {
      "properties": {
        "name": {
          "type": "string"
        },
        "location": {
          "type":               "geo_point",
          "geohash_prefix":     true,             <1>
          "geohash_precision":  "1km"             <2>
        }
      }
    }
  }
}
```
>  ![img](assets/1.png)  将 `geohash_prefix` 设为 `true` 来告诉 Elasticsearch 使用指定精度来索引 geohash 的前缀。  
>  
>  ![img](assets/2.png)  精度可以是一个具体的数字，代表的 geohash 的长度，也可以是一个距离。 `1km` 的精度对应的 geohash 的长度是 `7` 。   

通过如上设置， geohash 前缀中 1 到 7 的部分将被索引，所能提供的精度大约在 150 米。



### Geohash 单元查询  {#Geohash单元查询}

`geohash_cell` 查询做的事情非常简单： 把经纬度坐标位置根据指定精度转换成一个 geohash ，然后查找所有包含这个 geohash 的位置——这是非常高效的查询。

```json
GET /attractions/restaurant/_search
{
  "query": {
    "constant_score": {
      "filter": {
        "geohash_cell": {
          "location": {
            "lat":  40.718,
            "lon": -73.983
          },
          "precision": "2km" 
        }
      }
    }
  }
}
```
>  ![img](assets/1.png)  `precision` 字段设置的精度不能高于映射时 `geohash_precision` 字段指定的值。   

此查询将 `lat/lon` 坐标点转换成对应长度的 geohash —— 本例中为 `dr5rsk`—然后查找所有包含这个短语的位置。

然而，如上例中的写法可能不会返回 2km 内所有的餐馆。要知道 geohash 实际上仅是个矩形，而指定的点可能位于这个矩形中的任何位置。有可能这个点刚好落在了 geohash 单元的边缘附近，但过滤器会排除那些落在相邻单元的餐馆。

为了修复这个问题，我们可以通过设置 `neighbors`参数为 `true` ，让查询把周围的单元也包含进来：

```json
GET /attractions/restaurant/_search
{
  "query": {
    "constant_score": {
      "filter": {
        "geohash_cell": {
          "location": {
            "lat":  40.718,
            "lon": -73.983
          },
          "neighbors": true,     <1>
          "precision": "2km"
        }
      }
    }
  }
}
```
>  ![img](assets/1.png)  此查询将会寻找对应的 geohash 和包围它的 geohashes 。   

明显的， `2km` 精度的 geohash 加上周围的单元，最终导致一个范围极大的搜索区域。此查询不是为精度而生，但是它非常有效率，而且可以作为更高精度的地理位置过滤器的前置过滤器。
>  ![提示](assets/tip.png)  将 `precision` 参数设置为一个距离可能会有误导性。 `2km` 的 `precision` 会被转换成长度为 6 的 geohash 。实际上它的尺寸是约 1.2km x 0.6km。你可能会发现明确的设置长度为 `5`或 `6` 会更容易理解。  

此查询的另一个优点是，相比 `geo_bounding_box` 查询，它支持一个字段中有多个坐标位置的情况。 我们在 [优化盒模型](https://www.elastic.co/guide/cn/elasticsearch/guide/current/geo-bounding-box.html#optimize-bounding-box) 中讨论过，设置 `lat_lon` 选项也是一个很有效的方式，但是它只在每个字段只有单个坐标点的情况下有效。



## 地理位置聚合

虽然按照地理位置对结果进行过滤或者打分很有用， 但是在地图上呈现信息给用户通常更加有用。一个查询可能会返回太多结果以至于不能单独地展现每一个地理坐标点，但是地理位置聚合可以用来将地理坐标聚集到更加容易管理的 buckets 中。

处理 `geo_point` 类型字段的三种聚合：

- [`地理位置距离`](https://www.elastic.co/guide/cn/elasticsearch/guide/current/geo-distance-agg.html)

  将文档按照距离围绕一个中心点来分组。

- [`geohash 网格`](https://www.elastic.co/guide/cn/elasticsearch/guide/current/geohash-grid-agg.html)

  将文档按照 geohash 范围来分组，用来显示在地图上。

- [`地理位置边界`](https://www.elastic.co/guide/cn/elasticsearch/guide/current/geo-bounds-agg.html)

  返回一个包含所有地理位置坐标点的边界的经纬度坐标，这对显示地图时缩放比例的选择非常有用。



### 地理距离聚合

`geo_distance` 聚合 对一些搜索非常有用，例如找到所有距离我 1km 以内的披萨店。搜索结果应该也的确被限制在用户指定 1km 范围内，但是我们可以添加在 2km 范围内找到的其他结果：

```json
GET /attractions/restaurant/_search
{
  "query": {
    "bool": {
      "must": {                           <1>
        "match": { 
          "name": "pizza"
        }
      },
      "filter": {
        "geo_bounding_box": {
          "location": {                   <2>
            "top_left": {
              "lat":  40.8,
              "lon": -74.1
            },
            "bottom_right": {
              "lat":  40.4,
              "lon": -73.7
            }
          }
        }
      }
    }
  },
  "aggs": {
    "per_ring": {
      "geo_distance": {                   <3>
        "field":    "location",
        "unit":     "km",
        "origin": {
          "lat":    40.712,
          "lon":   -73.988
        },
        "ranges": [
          { "from": 0, "to": 1 },
          { "from": 1, "to": 2 }
        ]
      }
    }
  },
  "post_filter": {                       <4>
    "geo_distance": {
      "distance":   "1km",
      "location": {
        "lat":      40.712,
        "lon":     -73.988
      }
    }
  }
}
```
>  ![img](assets/1.png)   主查询查找名称中含有 `pizza` 的饭店。   
>  
>  ![img](assets/2.png)   `geo_bounding_box` 筛选那些只在纽约区域的结果。    
>  
>  ![img](assets/3.png)   `geo_distance` 聚合统计距离用户 1km 以内，1km 到 2km 的结果的数量。   
>  
>  ![img](assets/4.png)   最后，`post_filter` 将结果缩小至那些在用户 1km 范围内的饭店。 

前面的请求 响应如下：

```json
"hits": {
  "total":     1,
  "max_score": 0.15342641,
  "hits": [                                   <1>
     {
        "_index": "attractions",
        "_type":  "restaurant",
        "_id":    "3",
        "_score": 0.15342641,
        "_source": {
           "name": "Mini Munchies Pizza",
           "location": [
              -73.983,
              40.719
           ]
        }
     }
  ]
},
"aggregations": {
  "per_ring": {                                <2>
     "buckets": [
        {
           "key":       "*-1.0",
           "from":      0,
           "to":        1,
           "doc_count": 1
        },
        {
           "key":       "1.0-2.0",
           "from":      1,
           "to":        2,
           "doc_count": 1
        }
     ]
  }
}
```
>  ![img](assets/1.png)   `post_filter` 已经将搜索结果缩小至仅在用户 1km 范围以内的披萨店。   
>  
>  ![img](assets/2.png)  聚合包括搜索结果加上其他在用户 2km 范围以内的披萨店。  

在这个例子中，我们计算了落在每个同心环内的饭店数量。当然，我们可以在 `per_rings` 聚合下面嵌套子聚合来计算每个环的平均价格、最受欢迎程度，等等。



### Geohash 网格聚合  {#Geohash网格聚合}

通过一个查询返回的结果数量对在地图上单独的显示每一个位置点而言可能太多了。 `geohash_grid` 按照你定义的精度计算每一个点的 geohash 值而将附近的位置聚合在一起。

结果是一个网格—一个单元格表示一个可以显示在地图上的 geohash 。通过改变 geohash 的精度，你可以按国家或者城市街区来概括全世界。

聚合是稀疏的—它 仅返回那些含有文档的单元。 如果 geohashes 太精确，将产生太多的 buckets，它将默认返回那些包含了大量文档、最密集的10000个单元。 然而，为了计算哪些是最密集的 Top10000 ，它还是需要产生 *所有* 的 buckets 。可以通过以下方式来控制 buckets 的产生数量：

1. 使用 `geo_bounding_box` 来限制结果。
2. 为你的边界大小选择一个适当的 `precision` (精度)

```json
GET /attractions/restaurant/_search
{
  "size" : 0,
  "query": {
    "constant_score": {
      "filter": {
        "geo_bounding_box": {
          "location": {                   <1>
            "top_left": {
              "lat":  40.8,
              "lon": -74.1
            },
            "bottom_right": {
              "lat":  40.4,
              "lon": -73.7
            }
          }
        }
      }
    }
  },
  "aggs": {
    "new_york": {
      "geohash_grid": {                   <2>
        "field":     "location",
        "precision": 5
      }
    }
  }
}
```
>  ![img](assets/1.png)  边界框将搜索限制在大纽约区的范围       
>  
>  ![img](assets/2.png)  Geohashes 精度为 `5` 大约是 5km x 5km。   

Geohashes 精度为 `5` ，每个约25平方公里，所以10000个单元按这个精度将覆盖250000平方公里。我们指定的边界范围，约44km x 33km，或约1452平方公里，所以我们的边界在安全范围内；我们绝对不会在内存中创建了太多的 buckets。

前面的请求响应看起来是这样的：

```json
...
"aggregations": {
  "new_york": {
     "buckets": [                 <1>
        {
           "key": "dr5rs",
           "doc_count": 2
        },
        {
           "key": "dr5re",
           "doc_count": 1
        }
     ]
  }
}
...
```
>  ![img](assets/1.png)   每个 bucket 包含作为 `key` 的 geohash 值   

同样，我们也没有指定任何子聚合，所以我们得到是文档计数。如果需要，我们也可以了解这些 buckets 中受欢迎的餐厅类型、平均价格或其他细节。
>  ![提示](assets/tip.png)   要在地图上绘制这些 buckets，你需要一个将 geohash 转换成同等边界框或中心点的库。JavaScript 和其他语言已有的库会为你执行这个转换，但你也可以从使用 [geo-bounds-agg](https://www.elastic.co/guide/cn/elasticsearch/guide/current/geo-bounds-agg.html)的信息来进行类似的工作。  



### 地理边界聚合

在我们[之前的例子](https://www.elastic.co/guide/cn/elasticsearch/guide/current/geohash-grid-agg.html)中，我们通过一个覆盖大纽约区的边框来过滤结果。 然而，我们的结果全部都位于曼哈顿市中心。当为我们的用户显示一个地图的时候，放大包含数据的区域是有意义的；展示大量的空白空间是没有任何意义的。

`geo_bounds` 正好是这样的：它计算封装所有地理位置点需要的最小边界框：

```json
GET /attractions/restaurant/_search
{
  "size" : 0,
  "query": {
    "constant_score": {
      "filter": {
        "geo_bounding_box": {
          "location": {
            "top_left": {
              "lat":  40,8,
              "lon": -74.1
            },
            "bottom_right": {
              "lat":  40.4,
              "lon": -73.9
            }
          }
        }
      }
    }
  },
  "aggs": {
    "new_york": {
      "geohash_grid": {
        "field":     "location",
        "precision": 5
      }
    },
    "map_zoom": {                        <1>
      "geo_bounds": {
        "field":     "location"
      }
    }
  }
}
```
>  ![img](assets/1.png)   `geo_bounds` 聚合将计算封装所有匹配查询文档所需要的最小边界框。   

响应现在包括了一个可以用来缩放地图的边界框。

```json
...
"aggregations": {
  "map_zoom": {
     "bounds": {
        "top_left": {
           "lat":  40.722,
           "lon": -74.011
        },
        "bottom_right": {
           "lat":  40.715,
           "lon": -73.983
        }
     }
  },
...
```

事实上，我们甚至可以在每一个 geohash 单元内部使用 `geo_bounds` 聚合， 以免一个单元内的地理位置点仅集中在单元的一部分上：

```json
GET /attractions/restaurant/_search
{
  "size" : 0,
  "query": {
    "constant_score": {
      "filter": {
        "geo_bounding_box": {
          "location": {
            "top_left": {
              "lat":  40,8,
              "lon": -74.1
            },
            "bottom_right": {
              "lat":  40.4,
              "lon": -73.9
            }
          }
        }
      }
    }
  },
  "aggs": {
    "new_york": {
      "geohash_grid": {
        "field":     "location",
        "precision": 5
      },
      "aggs": {
        "cell": {                          <1>
          "geo_bounds": {
            "field": "location"
          }
        }
      }
    }
  }
}
```
>  ![img](assets/1.png)   `cell_bounds` 子聚合会为每个 geohash 单元计算边界框。  

现在在每个单元里的点有一个边界框。

```json
...
"aggregations": {
  "new_york": {
     "buckets": [
        {
           "key": "dr5rs",
           "doc_count": 2,
           "cell": {
              "bounds": {
                 "top_left": {
                    "lat":  40.722,
                    "lon": -73.989
                 },
                 "bottom_right": {
                    "lat":  40.719,
                    "lon": -73.983
                 }
              }
           }
        },
...
```



## 地理形状

地理形状（ Geo-shapes ）使用一种与地理坐标点完全不同的方法。我们在计算机屏幕上看到的圆形并不是由完美的连续的线组成的。而是用一个个连续的着色像素点画出的一个近似圆。地理形状的工作方式就与此相似。

复杂的形状——比如点集、线、多边形、多多边形、中空多边形——都是通过 geohash 单元 “画出来” 的，这些形状会转化为一个被它所覆盖到的 geohash 的集合。
>  ![注意](assets/note.png)  实际上，两种类型的网格可以被用于 geo-shapes：默认使用我们之前讨论过的 geohash ，另外还有一种是 *四叉树* 。四叉树与 geohash 类似，除了四叉树每个层级只有 4 个单元，而不是 32 。这种不同取决于编码方式的选择。

组成一个形状的 geohash 都作为一个单元被索引在一起。有了这些信息，通过查看是否有相同的 geohash 单元，就可以很轻易地检查两个形状是否有交集。

geo-shapes 有以下作用：判断查询的形状与索引的形状的关系；这些 `关系` 可能是以下之一：

- `intersects`

  查询的形状与索引的形状有重叠（默认）。

- `disjoint`

  查询的形状与索引的形状完全 *不* 重叠。

- `within`

  索引的形状完全被包含在查询的形状中。

Geo-shapes 不能用于计算距离、排序、打分以及聚合。



### 映射地理形状

与 `geo_point` 类型的字段相似， 地理形状 也必须在使用前明确映射：

```json
PUT /attractions
{
  "mappings": {
    "landmark": {
      "properties": {
        "name": {
          "type": "string"
        },
        "location": {
          "type": "geo_shape"
        }
      }
    }
  }
}
```

你需要考虑修改两个非常重要的设置： `精度` 和 `距离误差` 。



**精度**

`精度` （ `precision` ）参数 用来控制生成的 geohash 的最大长度。默认精度为 `9` ，等同于尺寸在 5m x 5m 的[geohash](https://www.elastic.co/guide/cn/elasticsearch/guide/current/geohashes.html) 。这个精度可能比你需要的精确得多。

精度越低，需要索引的单元就越少，检索时也会更快。当然，精度越低，地理形状的准确性就越差。你需要考虑自己的地理形状所需要的精度——即使减少1-2个等级的精度也能带来明显的消耗缩减收益。

你可以使用距离来指定精度 —— 如 `50m` 或 `2km`—不过这些距离最终也会转换成对应的[*Geohashes*](https://www.elastic.co/guide/cn/elasticsearch/guide/current/geohashes.html)等级。



**距离误差**

当索引一个多边形时，中间连续区域很容易用一个短 geohash 来表示。 麻烦的是边缘部分，这些地方需要使用更精细的 geohashes 才能表示。

当你在索引一个小地标时，你会希望它的边界比较精确。让这些纪念碑一个叠着一个可不好。当索引整个国家时，你就不需要这么高的精度了。误差个50米左右也不可能引发战争。

`距离误差` 指定地理形状可以接受的最大错误率。它的默认值是 `0.025` ， 即 2.5% 。也就是说，大的地理形状（比如国家）相比小的地理形状（比如纪念碑）来说，容许更加模糊的边界。

`0.025` 是一个不错的初始值。不过如果我们容许更大的错误率，对应地理形状需要索引的单元就越少。



### 索引地理形状

地理形状通过 [GeoJSON](http://geojson.org/) 来表示，这是一种开放的使用 JSON 实现的二维形状编码方式。 每个形状都包含了形状类型— `point`, `line`, `polygon`, `envelope` —和一个或多个经纬度点集合的数组。
>  ![小心](assets/caution.png)  在 GeoJSON 里，经纬度表示方式通常是 *纬度* 在前， *经度* 在后。  

如，我们用一个多边形来索引阿姆斯特丹达姆广场：

```json
PUT /attractions/landmark/dam_square
{
    "name" : "Dam Square, Amsterdam",
    "location" : {
        "type" : "polygon", 
        "coordinates" : [[ 
          [ 4.89218, 52.37356 ],
          [ 4.89205, 52.37276 ],
          [ 4.89301, 52.37274 ],
          [ 4.89392, 52.37250 ],
          [ 4.89431, 52.37287 ],
          [ 4.89331, 52.37346 ],
          [ 4.89305, 52.37326 ],
          [ 4.89218, 52.37356 ]
        ]]
    }
}
```

| [![img](assets/1-1552025856795.png)](https://www.elastic.co/guide/cn/elasticsearch/guide/current/indexing-geo-shapes.html#CO244-1) | `type` 参数指明了经纬度坐标集表示的形状类型。 |
| ------------------------------------------------------------ | --------------------------------------------- |
| [![img](assets/2-1552025857217.png)](https://www.elastic.co/guide/cn/elasticsearch/guide/current/indexing-geo-shapes.html#CO244-2) | `lon/lat` 列表描述了多边形的形状。            |

上例中大量的方括号可能看起来让人困惑，不过实际上 GeoJSON 的语法非常简单：

1. 用一个数组表示 `经纬度` 坐标点：

   ```
   [lon,lat]
   ```

2. 一组坐标点放到一个数组来表示一个多边形：

   ```
   [[lon,lat],[lon,lat], ... ]
   ```

3. 一个多边形（ `polygon` ）形状可以包含多个多边形；第一个表示多边形的外轮廓，后续的多边形表示第一个多边形内部的空洞：

   ```
   [
     [[lon,lat],[lon,lat], ... ],  # main polygon
     [[lon,lat],[lon,lat], ... ],  # hole in main polygon
     ...
   ]
   ```

参见 [Geo-shape mapping documentation](https://www.elastic.co/guide/en/elasticsearch/reference/5.6/geo-shape.html) 了解更多支持的形状。



### 查询地理形状

[`geo_shape` 查询](https://www.elastic.co/guide/en/elasticsearch/reference/5.6/query-dsl-geo-shape-query.html)不寻常的地方在于，它允许我们使用形状来做查询，而不仅仅是坐标点。

举个例子，当我们的用户刚刚迈出阿姆斯特丹中央火车站时，我们可以用如下方式，查询出方圆 1km 内的所有地标：

```json
GET /attractions/landmark/_search
{
  "query": {
    "geo_shape": {
      "location": {                <1>
        "shape": {                 <2>
          "type":   "circle",      <3>
          "radius": "1km",
          "coordinates": [         <4>
            4.89994,
            52.37815
          ]
        }
      }
    }
  }
}
```
>  ![img](assets/1.png)  查询使用 `location` 字段中的地理形状。  
>  
>  ![img](assets/2.png)  查询中的形状是由 `shape` 键对应的内容表示。  
>  
>  ![img](assets/3.png)  形状是一个半径为 1km 的圆形。  
>  
>  ![img](assets/4.png)  安姆斯特丹中央火车站入口的坐标点。   

默认的，查询（或者过滤器 —— 工作方式相同）会从已索引的形状中寻找与指定形状有交集的部分。 此外，可以把 `relation` 字段设置为 `disjoint` 来查找与指定形状不相交的部分，或者设置为 `within` 来查找完全落在查询形状中的。

举个例子，我们可以查找阿姆斯特丹中心区域所有的地标：

```json
GET /attractions/landmark/_search
{
  "query": {
    "geo_shape": {
      "location": {
        "relation": "within",            <1>
        "shape": {
          "type": "polygon",
          "coordinates": [[              <2>
              [4.88330,52.38617],
              [4.87463,52.37254],
              [4.87875,52.36369],
              [4.88939,52.35850],
              [4.89840,52.35755],
              [4.91909,52.36217],
              [4.92656,52.36594],
              [4.93368,52.36615],
              [4.93342,52.37275],
              [4.92690,52.37632],
              [4.88330,52.38617]
            ]]
        }
      }
    }
  }
}
```
>  ![img](assets/1.png)  只匹配完全落在查询形状中的已索引的形状。   
>  
>  ![img](assets/2.png)  这个多边形表示安姆斯特丹中心区域。     



### 在查询中使用已索引的形状

对于那些经常会在查询中使用的形状，可以把它们索引起来以便在查询中可以方便地直接引用名字。 以之前的阿姆斯特丹中部为例，我们可以把它存储成一个类型为 `neighborhood` 的文档。

首先，我们仿照之前设置 `landmark` 时的方式建立映射：

```json
PUT /attractions/_mapping/neighborhood
{
  "properties": {
    "name": {
      "type": "string"
    },
    "location": {
      "type": "geo_shape"
    }
  }
}
```

然后我们索引阿姆斯特丹中部对应的形状：

```json
PUT /attractions/neighborhood/central_amsterdam
{
  "name" : "Central Amsterdam",
  "location" : {
      "type" : "polygon",
      "coordinates" : [[
        [4.88330,52.38617],
        [4.87463,52.37254],
        [4.87875,52.36369],
        [4.88939,52.35850],
        [4.89840,52.35755],
        [4.91909,52.36217],
        [4.92656,52.36594],
        [4.93368,52.36615],
        [4.93342,52.37275],
        [4.92690,52.37632],
        [4.88330,52.38617]
      ]]
  }
}
```

形状索引好之后，我们就可以在查询中通过 `index` ， `type` 和 `id` 来引用它了：

```json
GET /attractions/landmark/_search
{
  "query": {
    "geo_shape": {
      "location": {
        "relation": "within",
        "indexed_shape": {                    <1>
          "index": "attractions",
          "type":  "neighborhood",
          "id":    "central_amsterdam",
          "path":  "location"
        }
      }
    }
  }
}
```
>  ![img](assets/1.png)  指定 `indexed_shape` 而不是 `shape` ，Elasticesearch 就知道需要从指定的文档和 `path` 检索出对应的形状了。   

阿姆斯特丹中部这个形状没有什么特别的。同样地，我们也可以在查询中使用已经索引好的达姆广场。这个查询可以找出与达姆广场有交集的临近点：

```json
GET /attractions/neighborhood/_search
{
  "query": {
    "geo_shape": {
      "location": {
        "indexed_shape": {
          "index": "attractions",
          "type":  "landmark",
          "id":    "dam_square",
          "path":  "location"
        }
      }
    }
  }
}
```