# Elasticsearch02

## DSL查询
- ES中提供了DSL查询，就是以JSON格式来定义查询条件
- DSL查询可以分为两大类
  - 叶子查询：一般是在特定的字段里查询特定值，属于简单查询，很少单独使用
  - 复合查询：以逻辑方式组合多个叶子查询或者更改叶子查询的行为方式
- 在查询以后，还可以对查询的结果做处理
  - 排序：按照1个或多个字段值做排序
  - 分页：根据from和size做分页，类似Mysql
  - 高亮：对搜索结果中的关键字添加特殊样式，使其更加醒目
  - 聚合：对搜索结果做数据统计以形成报表
### 入门
- 我们依然在Kibana的DevTools中学习查询的DSL语法。首先来看查询的语法结构：
```Java
GET /{索引库名}/_search
{
  "query": {
    "查询类型": {
      // .. 查询条件
    }
  }
}
```
- GET /{索引库名}/_search：其中的_search是固定路径，不能修改
- 我们以最简单的无条件查询为例，无条件查询的类型是：match_all，因此其查询语句如下：
```
GET /items/_search
{
  "query": {
    "match_all": {
      
    }
  }
}
```
- 你会发现虽然是match_all，但是响应结果中并不会包含索引库中的所有文档，而是仅有10条。这是因为处于安全考虑，elasticsearch设置了默认的查询页数。
### 叶子查询
- 全文检索查询（Full Text Queries）：利用分词器对用户输入搜索条件先分词，得到词条，然后再利用倒排索引搜索词条。例如：
    - match：
    - multi_match
- 精确查询（Term-level queries）：不对用户输入搜索条件分词，根据字段内容精确值匹配。但只能查找keyword、数值、日期、boolean类型的字段。例如：
    - ids
    - term
    - range
- 地理坐标查询：用于搜索地理位置，搜索方式很多，例如：
    - geo_bounding_box：按矩形搜索
    - geo_distance：按点和半径搜索
#### 全文检索查询
以全文检索中的match为例，语法如下：
```Java
GET /{索引库名}/_search
{
  "query": {
    "match": {
      "字段名": "搜索条件"
    }
  }
}
```
与match类似的还有multi_match，区别在于可以同时对多个字段搜索，而且多个字段都要满足，语法示例：
```Java
GET /{索引库名}/_search
{
  "query": {
    "multi_match": {
      "query": "搜索条件",
      "fields": ["字段1", "字段2"]
    }
  }
}
```
#### 精确查询
精确查询，英文是Term-level query，顾名思义，词条级别的查询。也就是说不会对用户输入的搜索条件再分词，而是作为一个词条，与搜索的字段内容精确值匹配。因此推荐查找keyword、数值、日期、boolean类型的字段。例如：
- id
- price
- 城市
- 地名
- 人名
以term查询为例，其语法如下
```Java
GET /{索引库名}/_search
{
  "query": {
    "term": {
      "字段名": {
        "value": "搜索条件"
      }
    }
  }
} 
```
当你输入的搜索条件不是词条，而是短语时，由于不做分词，你反而搜索不到：
- 再来看下range查询，语法如下
```Java
GET /{索引库名}/_search
{
  "query": {
    "range": {
      "字段名": {
        "gte": {最小值},
        "lte": {最大值}
      }
    }
  }
}
```
### 复合查询
复合查询大致可以分为两类：
- 第一类：基于逻辑运算组合叶子查询，实现组合条件，例如
    - bool
- 第二类：基于某种算法修改查询时的文档相关性算分，从而改变文档排名。例如：
    - function_score
    - dis_max
#### 算分函数查询
- 当我们利用match查询时，文档结果会根据与搜索词条的关联度打分（_score），返回结果时按照分值降序排列。
- 从elasticsearch5.1开始，采用的相关性打分算法是BM25算法，公式如下：
![es4.png](es4.png)
- 要想认为控制相关性算分，就需要利用elasticsearch中的function score 查询了。
##### 基本语法
function score 查询中包含四部分内容：
- 原始查询条件：query部分，基于这个条件搜索文档，并且基于BM25算法给文档打分，原始算分（query score)
- 过滤条件：filter部分，符合该条件的文档才会重新算分
- 算分函数：符合filter条件的文档要根据这个函数做运算，得到的函数算分（function score），有四种函数
  - weight：函数结果是常量
  - field_value_factor：以文档中的某个字段值作为函数结果
  - random_score：以随机数作为函数结果
  - script_score：自定义算分函数算法
- 运算模式：算分函数的结果、原始查询的相关性算分，两者之间的运算方式，包括：
  - multiply：相乘
  - replace：用function score替换query score
  - 其它，例如：sum、avg、max、min
###### function score的运行流程
- 1）根据原始条件查询搜索文档，并且计算相关性算分，称为原始算分（query score）
- 2）根据过滤条件，过滤文档
- 3）符合过滤条件的文档，基于算分函数运算，得到函数算分（function score）
- 4）将原始算分（query score）和函数算分（function score）基于运算模式做运算，得到最终结果，作为相关性算分。
- 因此，其中的关键点是：
- 过滤条件：决定哪些文档的算分被修改
- 算分函数：决定函数算分的算法
- 运算模式：决定最终算分结果
##### 实例
示例：给IPhone这个品牌的手机算分提高十倍，分析如下：
- 过滤条件：品牌必须为IPhone
- 算分函数：常量weight，值为10
- 算分模式：相乘multiply
- 对应代码如下：
```Java
GET /hotel/_search
{
  "query": {
    "function_score": {
      "query": {  .... }, // 原始查询，可以是任意条件
      "functions": [ // 算分函数
        {
          "filter": { // 满足的条件，品牌必须是Iphone
            "term": {
              "brand": "Iphone"
            }
          },
          "weight": 10 // 算分权重为2
        }
      ],
      "boost_mode": "multipy" // 加权模式，求乘积
    }
  }
}
```
#### bool查询
bool查询，即布尔查询。就是利用逻辑运算来组合一个或多个查询子句的组合。bool查询支持的逻辑运算有：
- must：必须匹配每个子查询，类似“与”
- should：选择性匹配子查询，类似“或”
- must_not：必须不匹配，不参与算分，类似“非”
- filter：必须匹配，不参与算分 
- bool查询的语法如下：
```Java
GET /items/_search
{
  "query": {
    "bool": {
      "must": [
        {"match": {"name": "手机"}}
      ],
      "should": [
        {"term": {"brand": { "value": "vivo" }}},
        {"term": {"brand": { "value": "小米" }}}
      ],
      "must_not": [
        {"range": {"price": {"gte": 2500}}}
      ],
      "filter": [
        {"range": {"price": {"lte": 1000}}}
      ]
    }
  }
}
```
出于性能考虑，与搜索关键字无关的查询尽量采用must_not或filter逻辑运算，避免参与相关性算分。
例如黑马商城的搜索页面：
![es6.png](es6.png)

### 排序
elasticsearch默认是根据相关度算分（_score）来排序，但是也支持自定义方式对搜索结果排序。不过分词字段无法排序，能参与排序字段类型有：keyword类型、数值类型、地理坐标类型、日期类型等。
```Java
GET /indexName/_search
{
  "query": {
    "match_all": {}
  },
  "sort": [
    {
      "排序字段": {
        "order": "排序方式asc和desc"
      }
    }
  ]
}

```
### 分页
elasticsearch 默认情况下只返回top10的数据。而如果要查询更多数据就需要修改分页参数了。
#### 基础分页
elasticsearch中通过修改from、size参数来控制要返回的分页结果：
- from：从第几个文档开始
- size：总共查询几个文档
  类似于mysql中的limit ?, ?
```Java
GET /items/_search
{
  "query": {
    "match_all": {}
  },
  "from": 0, // 分页开始的位置，默认为0
  "size": 10,  // 每页文档数量，默认10
  "sort": [
    {
      "price": {
        "order": "desc"
      }
    }
  ]
}
```
#### 深度分页
- elasticsearch的数据一般会采用分片存储，也就是把一个索引中的数据分成N份，存储到不同节点上。这种存储方式比较有利于数据扩展，但给分页带来了一些麻烦。
- 比如一个索引库中有100000条数据，分别存储到4个分片，每个分片25000条数据。现在每页查询10条，查询第99页。
- 要知道每一片的数据都不一样，第1片上的第900~1000，在另1个节点上并不一定依然是900~1000名。所以我们只能在每一个分片上都找出排名前1000的数据，然后汇总到一起，重新排序，才能找出整个索引库中真正的前1000名，此时截取990~1000的数据即可。
- 由此可知，当查询分页深度较大时，汇总数据过多，对内存和CPU会产生非常大的压力。
  因此elasticsearch会禁止from+ size 超过10000的请求。
- 针对深度分页，elasticsearch提供了两种解决方案：
  - search after：分页时需要排序，原理是从上一次的排序值开始，查询下一页数据。官方推荐使用的方式。
  - scroll：原理将排序后的文档id形成快照，保存下来，基于快照做分页。官方已经不推荐使用。
### 高亮
#### 高亮原理
我们在百度，京东搜索时，关键字会变成红色，比较醒目，这叫高亮显示
观察页面源码，你会发现两件事情：
- 高亮词条都被加了`<em>`标签
- `<em>`标签都添加了红色样式
- 因此实现高亮的思路就是
  - 用户输入搜索关键字搜索数据
  - 服务端根据搜索关键字到elasticsearch搜索，并给搜索结果中的关键字词条添加html标签
  - 前端提前给约定好的html标签添加CSS样式
#### 实现高亮
事实上elasticsearch已经提供了给搜索关键字加标签的语法，无需我们自己编码。
```Java
GET /{索引库名}/_search
{
  "query": {
    "match": {
      "搜索字段": "搜索关键字"
    }
  },
  "highlight": {
    "fields": {
      "高亮字段名称": {
        "pre_tags": "<em>",
        "post_tags": "</em>"
      }
    }
  }
}

```
注意：
- 搜索必须有查询条件，而且是全文检索类型的查询条件，例如match
- 参与高亮的字段必须是text类型的字段
- 默认情况下参与高亮的字段要与搜索字段一致，除非添加：required_field_match=false
## RestClient 查询
### 查询
```Java
public void testMatchAll() throws IOException {
        // 1.创建请求对象
        SearchRequest request = new SearchRequest(INDEX_NAME);
        request.source().query(QueryBuilders.matchAllQuery());
        SearchResponse response = client.search(request, RequestOptions.DEFAULT);
        // 2.处理响应结果
        SearchHits searchHits = response.getHits();
        //总条数
        System.out.println(searchHits.getTotalHits().value);
        // 命中的数据
        SearchHit[] hits = searchHits.getHits();
        for (SearchHit hit : hits) {
            // 获取结果
            String json = hit.getSourceAsString();
            System.out.println(json);
        }
    }

```
### 构建查询条件
#### 单字段查询
`request.source().query(QueryBuilders.matchQuery("title", "小米"));`
#### 多字段查询
`request.source().query(QueryBuilders.multiMatchQuery("小米", "title", "brand"));`
#### 词条查询
`request.source().query(QueryBuilders.termQuery("title", "小米"));`
#### 范围查询
`request.source().query(QueryBuilders.rangeQuery("price").gte(1000).lte(2000));`
#### bool查询 {id="bool_1"}
`request.source().query(QueryBuilders.boolQuery()
.must(QueryBuilders.termQuery("title", "小米"))
.must(QueryBuilders.rangeQuery("price").gte(1000).lte(2000)));`
### 排序和分页
```Java
// 排序
request.source().sort("price", SortOrder.DESC);
// 分页
request.source().from(0).size(10);
```
### 高亮显示
`request.source().highlighter(SearchSourceBuilder.highlight()
.field("name")
.preTags("<em>").postTags("</em>"));`
- 显示高亮的数据 不能直接getHit 需要获得高亮显示结果
```Java
// 2.处理响应结果
SearchHits searchHits = response.getHits();
//总条数
System.out.println(searchHits.getTotalHits().value);
// 命中的数据
SearchHit[] hits = searchHits.getHits();
for (SearchHit hit : hits) {
    // 获取结果
    String json = hit.getSourceAsString();
    Map<String, HighlightField> map = hit.getHighlightFields();
    if (map.containsKey("name")) {
        HighlightField highlightField = map.get("name");
        String highlight = highlightField.getFragments()[0].toString();
        System.out.println(highlight);
    }
    System.out.println(json);
}
```
## 数据聚合
- 聚合（aggregations）可以让我们极其方便的实现对数据的统计、分析、运算
- 聚合常见的有三类：
  - 桶（Bucket）聚合：用来对文档做分组
    - TermAggregation：按照文档字段值分组，例如按照品牌值分组、按照国家分组
    - Date Histogram：按照日期阶梯分组，例如一周为一组，或者一月为一组
  - 度量（Metric）聚合：用以计算一些值，比如：最大值、最小值、平均值等
    - Avg：求平均值
    - Max：求最大值
    - Min：求最小值
    - Stats：同时求max、min、avg、sum等
  -  管道（pipeline）聚合：其它聚合的结果为基础做进一步运算 
- 注意：参加聚合的字段必须是keyword、日期、数值、布尔类型
### DSL实现聚合 {id="dsl_1"}
#### Bucket聚合
- 例如我们要统计所有商品中共有哪些商品分类，其实就是以分类（category）字段对数据分组。category值一样的放在同一组，属于Bucket聚合中的Term聚合。
```Java
GET /items/_search
{
  "size": 0, 
  "aggs": {
    "category_agg": {
      "terms": {
        "field": "category",
        "size": 20
      }
    }
  }
}
```
语法说明：
- size：设置size为0，就是每页查0条，则结果中就不包含文档，只包含聚合
- aggs：定义聚合
  - category_agg：聚合名称，自定义，但不能重复
    - terms：聚合的类型，按分类聚合，所以用term
      - field：参与聚合的字段名称
      - size：希望返回的聚合结果的最大数量
![es7.png](es7.png)
#### 带条件聚合
- 默认情况下，Bucket聚合是对索引库的所有文档做聚合，例如我们统计商品中所有的品牌
- 可以看到统计出的品牌非常多。
  但真实场景下，用户会输入搜索条件，因此聚合必须是对搜索结果聚合。那么聚合必须添加限定条件。
- 例如，我想知道价格高于3000元的手机品牌有哪些，该怎么统计呢？
```Java
GET /items/_search
{
  "query": {
    "bool": {
      "filter": [
        {
          "term": {
            "category": "手机"
          }
        },
        {
          "range": {
            "price": {
              "gte": 300000
            }
          }
        }
      ]
    }
  }, 
  "size": 0, 
  "aggs": {
    "brand_agg": {
      "terms": {
        "field": "brand",
        "size": 20
      }
    }
  }
}
```
- 结果
```Java
{
  "took" : 2,
  "timed_out" : false,
  "hits" : {
    "total" : {
      "value" : 13,
      "relation" : "eq"
    },
    "max_score" : null,
    "hits" : [ ]
  },
  "aggregations" : {
    "brand_agg" : {
      "doc_count_error_upper_bound" : 0,
      "sum_other_doc_count" : 0,
      "buckets" : [
        {
          "key" : "华为",
          "doc_count" : 7
        },
        {
          "key" : "Apple",
          "doc_count" : 5
        },
        {
          "key" : "小米",
          "doc_count" : 1
        }
      ]
    }
  }
}
```
#### Metric聚合
- 我们统计了价格高于3000的手机品牌，形成了一个个桶。现在我们需要对桶内的商品做运算，获取每个品牌价格的最小值、最大值、平均值。
  这就要用到Metric聚合了，例如stats聚合，就可以同时获取min、max、avg等结果。
```Java
GET /items/_search
{
  "query": {
    "bool": {
      "filter": [
        {
          "term": {
            "category": "手机"
          }
        },
        {
          "range": {
            "price": {
              "gte": 300000
            }
          }
        }
      ]
    }
  }, 
  "size": 0, 
  "aggs": {
    "brand_agg": {
      "terms": {
        "field": "brand",
        "size": 20
      },
      "aggs": {
        "stats_meric": {
          "stats": {
            "field": "price"
          }
        }
      }
    }
  }
}
```
query部分就不说了，我们重点解读聚合部分语法。
可以看到我们在brand_agg聚合的内部，我们新加了一个aggs参数。这个聚合就是brand_agg的子聚合，会对brand_agg形成的每个桶中的文档分别统计。
- stats_meric：聚合名称
  - stats：聚合类型，stats是metric聚合的一种
    - field：聚合字段，这里选择price，统计价格
- 由于stats是对brand_agg形成的每个品牌桶内文档分别做统计，因此每个品牌都会统计出自己的价格最小、最大、平均值。
  结果如下：
![es8.png](es8.png)
- 另外，我们还可以让聚合按照每个品牌的价格平均值排序：
![es9.png](es9.png)
### RestClient实现聚合 {id="restclient_1"}
- 可以看到在DSL中，aggs聚合条件与query条件是同一级别，都属于查询JSON参数。因此依然是利用request.source()方法来设置。
- 不过聚合条件的要利用AggregationBuilders这个工具类来构造。DSL与JavaAPI的语法对比如下：
![es10.png](es10.png)
- 聚合结果与搜索文档同一级别，因此需要单独获取和解析。具体解析语法如下
![es11.png](es11.png)
```Java
@Test
void testAgg() throws IOException {
    // 1.创建Request
    SearchRequest request = new SearchRequest("items");
    // 2.准备请求参数
    BoolQueryBuilder bool = QueryBuilders.boolQuery()
            .filter(QueryBuilders.termQuery("category", "手机"))
            .filter(QueryBuilders.rangeQuery("price").gte(300000));
    request.source().query(bool).size(0);
    // 3.聚合参数
    request.source().aggregation(
            AggregationBuilders.terms("brand_agg").field("brand").size(5)
    );
    // 4.发送请求
    SearchResponse response = client.search(request, RequestOptions.DEFAULT);
    // 5.解析聚合结果
    Aggregations aggregations = response.getAggregations();
    // 5.1.获取品牌聚合
    Terms brandTerms = aggregations.get("brand_agg");
    // 5.2.获取聚合中的桶
    List<? extends Terms.Bucket> buckets = brandTerms.getBuckets();
    // 5.3.遍历桶内数据
    for (Terms.Bucket bucket : buckets) {
        // 5.4.获取桶内key
        String brand = bucket.getKeyAsString();
        System.out.print("brand = " + brand);
        long count = bucket.getDocCount();
        System.out.println("; count = " + count);
    }
}
```