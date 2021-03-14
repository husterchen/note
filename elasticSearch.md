# 交互命令

```cmd
//索引
//put指令需要指定id值，post不需要
PUT /megacorp/_create/3
{
    "first_name" :  "Douglas",
    "last_name" :   "Fir",
    "age" :         35,
    "about":        "I like to build cabinets",
    "interests":  [ "forestry" ]
}
//查询
GET /megacorp/_doc/1
//查询索引下所有记录
GET /megacorp/_search
//条件索引
GET /megacorp/_search?q=last_name:Smith
//部分查询返回指定内容
GET /website/blog/123?_source=title,text
//查询表达式
//条件查询并不是强匹配，相关的记录都会给出，根据匹配度进行排序
GET /megacorp/_search
{
    "query" : {
        "bool": {
            "must": {
                "match" : {
                    "last_name" : "smith" 
                }
            },
            "filter": {
                "range" : {
                    "age" : { "gt" : 30 } 
                }
            }
        }
    }
}
//短语查询，高亮显示
GET /megacorp/_search
{
    "query" : {
        "match_phrase" : {
            "about" : "rock climbing"
        }
    },
    "highlight": {
        "fields" : {
            "about" : {}
        }
    }
}
//聚合索引
//默认没有优化的字段是不能进行聚合的（类似传统数据库没加索引）
PUT /megacorp/_mapping?pretty
{
  "properties": {
    "interests": { 
      "type":     "text",
      "fielddata": true
    }
  }
}
GET /megacorp/_search
{
    "aggs" : {
        "all_interests" : {
            "terms" : { "field" : "interests" },
            "aggs" : {
                "avg_age" : {
                    "avg" : { "field" : "age" }
                }
            }
        }
    }
}
//创建索引，并设置分片数
PUT /blogs
{
   "settings" : {
      "number_of_shards" : 3,
      "number_of_replicas" : 1
   }
}
//判断文档知否存在
HEAD /website/blog/123
//删除文档，即使不存在版本号也会增加
DELETE /website/blog/123
//修改文档
POST /website/blog/1/_update
{
   "doc" : {
      "tags" : [ "testing" ],
      "views": 0
   }
}
//使用脚本更新文档
//不存在就根据给定的参数新增一条
//冲突重试
POST /website/blog/1/_update?retry_on_conflict=5
{
   "script" : "ctx._source.views+=1",
   "upsert": {
       "views": 1
   }
}
//多个文档同时索引
GET /_mget
{
   "docs" : [
      {
         "_index" : "website",
         "_type" :  "blog",
         "_id" :    2
      },
      {
         "_index" : "website",
         "_type" :  "pageviews",
         "_id" :    1,
         "_source": "views"
      }
   ]
}
//批量请求
//单个请求的结果不会影响其他请求 
POST /_bulk
{ "delete": { "_index": "website", "_type": "blog", "_id": "123" }} 
{ "create": { "_index": "website", "_type": "blog", "_id": "123" }}
{ "title":    "My first blog post" }
{ "index":  { "_index": "website", "_type": "blog" }}
{ "title":    "My second blog post" }
{ "update": { "_index": "website", "_type": "blog", "_id": "123", "_retry_on_conflict" : 3} }
{ "doc" : {"title" : "My updated blog post"} } //换行符
//查询索引的模式定义，可以查看字段类型等
GET /gb/_mapping/tweet
//分析器测试
GET /_analyze
{
  "analyzer": "standard",
  "text": "Text to analyze"
}
//创建索引并设置字段映射
PUT /gb 
{
  "mappings": {
    "tweet" : {
      "properties" : {
        "tweet" : {
          "type" :    "string",
          "analyzer": "english"
        },
        "date" : {
          "type" :   "date"
        },
        "name" : {
          "type" :   "string"
        },
        "user_id" : {
          "type" :   "long"
        }
      }
    }
  }
}
//分页(search请求体查询)
GET /_search?size=5&from=5
//查询语句检查
//加上explain会返回错误原因，对于合法的查询会返回es时如何解析该查询的
GET /gb/tweet/_validate/query?expalin
{
   "query": {
      "tweet" : {
         "match" : "really powerful"
      }
   }
}
//多值字段排序
//下面就表示根据dates字段中的最小时间进行排序
"sort": {
    "dates": {
        "order": "asc",
        "mode":  "min"
    }
}
//组合过滤
GET /my_store/products/_search
{
   "query" : {
      "filtered" : { 
         "filter" : {
            "bool" : {
              "should" : [
                 { "term" : {"price" : 20}}, 
                 { "term" : {"productID" : "XHDK-A-1293-#fJ3"}} 
              ],
              "must_not" : {
                 "term" : {"price" : 30} 
              }
           }
         }
      }
   }
}
//查询表达式
GET /my_index/my_type/_search
{
    "query": {
        "match": {
            "title": {      
                "query":    "BROWN DOG!",
                "operator": "and"
            }
        }
    }
}
//控制查询项中需要匹配字段的个数
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
// 控制should需要满足的个数
GET /my_index/my_type/_search
{
  "query": {
    "bool": {
      "should": [
        { "match": { "title": "brown" }},
        { "match": { "title": "fox"   }},
        { "match": { "title": "dog"   }}
      ],
      "minimum_should_match": 2 
    }
  }
}
//通过should和boost可以提升特定文档的匹配度，should匹配的条目越多匹配度越高
//boost大于1会提升权重，置于0-1之间会降低权重
GET /_search
{
    "query": {
        "bool": {
            "must": {
                "match": {  
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
                        "boost": 3 
                    }
                }},
                { "match": {
                    "content": {
                        "query": "Lucene",
                        "boost": 2 
                    }
                }}
            ]
        }
    }
}
//可以利用copy_to参数进行字段值的复制
PUT /my_index
{
    "mappings": {
        "person": {
            "properties": {
                "first_name": {
                    "type":     "string",
                    "copy_to":  "full_name" 
                },
                "last_name": {
                    "type":     "string",
                    "copy_to":  "full_name" 
                },
                "full_name": {
                    "type":     "string"
                }
            }
        }
    }
}
```

# 内部实现

作为用户，我们可以将请求发送到 *集群中的任何节点* ，包括主节点。 每个节点都知道任意文档所处的位置，并且能够将我们的请求直接转发到存储我们所需文档的节点。

一个副本分片只是一个主分片的拷贝。副本分片作为硬件故障时保护数据不丢失的冗余备份，并为搜索和返回文档等读操作提供服务。

在 Elasticsearch 中， *每个字段的所有数据* 都是 *默认被索引的* 。 即每个字段都有为了快速检索设置的专用倒排索引。而且，不像其他多数的数据库，它能在 *同一个查询中* 使用所有这些倒排索引，并以惊人的速度返回结果。

文档元数据：

* _index	文档存放位置
* _type      文档表示的对象类别
* _id。       文档唯一标识

在 Elasticsearch 中每个文档都有一个版本号。当每次对文档进行修改时（包括删除）， `_version` 的值会递增。创建文档时可以自己指定id值，也可以不指定es会自动分配一个id值，此时创建指令改成POST。

在 Elasticsearch 中文档是 *不可改变* 的，不能修改它们。相反，如果想要更新现有的文档，需要 重建索引或者进行替换。在内部，Elasticsearch 已将旧文档标记为已删除，并增加一个全新的文档。 尽管你不能再对旧版本的文档进行访问，但它并不会立即消失。当继续索引更多的数据，Elasticsearch 会在后台清理这些已删除文档。

文档修改流程：

1. 从旧文档构建 JSON
2. 更改该 JSON
3. 删除旧文档
4. 索引一个新文档

当我们使用 `index` API 更新文档 ，可以一次性读取原始文档，做我们的修改，然后重新索引 *整个文档* 。 最近的索引请求将获胜：无论最后哪一个文档被索引，都将被唯一存储在 Elasticsearch 中。如果其他人同时更改这个文档，他们的更改将丢失。当然可以通过指定版本号es会进行乐观锁控制。

es分片策略：shard = hash(routing) % number_of_primary_shards，相同分片的副本不会放在同一节点。

es的副本节点的复制是同步的，一旦索引请求成功返回给用户就表示文档在主分片和副本分片都是可用的。在处理读请求时协调节点在每次请求的时候都会通过轮询所有的副本分片达到负载均衡。

协调节点将多文档请求分解成每个分片的多文档请求，并且将这些请求并行转发到每个参与节点。

```sense
GET /_search?q=mary
这个表示查询所有包含mary的文档，除非特指某个字段否则都是这样，这种查询类型都是string，es在进行这种查询时会将所有字段拼接进行索引。
```

es中的数据可以分为：精确值和全文，对于精确值字段的索引要求精确匹配。索引全文域时，索引的内容也需要进行和索引域相同的分析流程。

数组类型也是全文域，需要经过分析进行倒排索引的创建，es会将数组的第一个值的类型作为该域的类型。

空值域不能被索引。

内部对象类型es会进行下面的转化：

```js
{
    "tweet":            [elasticsearch, flexible, very],
    "user.id":          [@johnsmith],
    "user.gender":      [male],
    "user.age":         [26],
    "user.name.full":   [john, smith],
    "user.name.first":  [john],
    "user.name.last":   [smith]
}
```

es分析器的工作内容：

* 字符过滤器
* 分词器
* Token过滤器

常用查询：

* match_all
* match
* multi_natch
* range
* term（精确查询）
* terms（精确查询）
* exists
* missing

可以通过bool语句进行组合查询，它接受以下参数：

* must
* must_not
* should
* filter（不评分查询，性能较好）

有时候我们需要相同字段存在不同的定义，可以使用下面的命令为同一个字段创建多个映射。例如当我们需要根据tweet字段进行检索，但同时也要根据该字段进行排序，但是字段为analyzed的时候才能进行全文索引，但是该类型的字段进行排序时性能较差。

```cmd
"tweet": { 
    "type":     "string",
    "analyzer": "english",
    "fields": {
        "raw": { 
            "type":  "string",
            "index": "not_analyzed"
        }
    }
}
```

es的相关性算法：

* 检索词频率
* 反向文档频率
* 字段长度准则

分布式查询的流程：

1. 客户端发送一个 `search` 请求到 `Node 3` ， `Node 3` 会创建一个大小为 `from + size` 的空优先队列。
2. `Node 3` 将查询请求转发到索引的每个主分片或副本分片中。每个分片在本地执行查询并添加结果到大小为 `from + size` 的本地有序优先队列中。
3. 每个分片返回各自优先队列中所有文档的 ID 和排序值给协调节点，也就是 `Node 3` ，它合并这些值到自己的优先队列中来产生一个全局排序后的结果列表。

偏好这个参数 `preference` 允许 用来控制由哪些分片或节点来处理搜索请求。 它接受像 `_primary`, `_primary_first`, `_local`, `_only_node:xyz`, `_prefer_node:xyz`, 和 `_shards:2,3` 这样的值。

参数 `timeout` 告诉 分片允许处理数据的最大时间。如果没有足够的时间处理所有数据，这个分片的结果可以是部分的，甚至是空数据。

参数routing可以在查询时指定分片。

在底层Lucene中同一索引的不同类型共享一份映射关系，所以不同类型中不能对相同的字段创建不同的映射。

字段 _all表示一个把其它字段值当作一个大字符串来索引的特殊字段。 可以通过enable字段设置是否启用_all字段，也可通过include_in_all字段指定哪些字段包含在_all中。

动态映射：可以直接导入数据到es，es会创建相应的索引和映射关系。

自定义动态映射：日期检测，自定义动态映射模板。

倒排索引被持久化到磁盘后是不可改变的。

translog 也被用来提供实时 CRUD 。当你试着通过ID查询、更新、删除一个文档，它会在尝试从相应的段中检索之前， 首先检查 translog 任何最近的变更。这意味着它总是能够实时地获取到文档的最新版本。

es数据持久化：

新增的索引必须写入到segment后才能被搜索到。

![img](https://tva1.sinaimg.cn/large/008eGmZEgy1go26sqgru1j30ix0cf757.jpg)

![img](https://tva1.sinaimg.cn/large/008eGmZEgy1go26v1hmddj316w0q279l.jpg)

refresh：（refresh_interval参数就是用来控制refresh操作，默认是1秒）

- 所有在内存缓冲区中的文档被写入到一个新的segment中，但是没有调用fsync，因此内存中的数据可能丢失
- segment被打开使得里面的文档能够被搜索到
- 清空内存缓冲区

flush：

* 通过refresh操作把所有在内存缓冲区中的文档写入到一个新的segment中
* 清空内存缓冲区
* 往磁盘里写入commit point信息
* 文件系统的page cache(segments) fsync到磁盘
* 删除旧的translog文件，因此此时内存中的segments已经写入到磁盘中,就不需要translog来保障数据安全了

精确查询：term，他可以对数字，布尔值，日期和文本进行精确查询。对文本进行term查询时要将查询字段设置成not_analyzed，因为如果字段为analyzed，es会对字段进行分析拆分。

多个精确值查询：terms，term和terms都表示一种包含关系。

es无法处理空值，如果一个字段的值为空，那么es不会存储任何值。

存在查询：exists；缺失查询：missing。

最佳字段匹配：当我们进行多字段搜索时，我们希望按照最佳匹配字段进行排序返回，可以使用下面的查询方式。

对于2，3插入的文档，如果我们通过4进行查询，那么因为1中的title和body字段都含有索引字段，所以1最后的评分会高于2，但是2中的body字段包含最佳的字段匹配，所以如果我们期望的是2的评分高的话，可以通过dis_max参数进行查询，例如1。dis_max查询只是简单的使用最佳匹配语句的评分最为最终评分。

但是如果对于quick，pet进行查询，两个文档中都包含quick，但是只有文档2含有pets，照理应该是文档2评分会高一点，但是因为dis_max语义，会导致两个文档的评分相同，可以通过tie_breaker参数解决这个问题。

```cmd
1:
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
2:
PUT /my_index/my_type/1
{
    "title": "Quick brown rabbits",
    "body":  "Brown rabbits are commonly seen."
}
3:
PUT /my_index/my_type/2
{
    "title": "Keeping pets healthy",
    "body":  "My quick brown fox eats rabbits on a regular basis."
}
4:
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

multi_match查询：

```cmd
{
    "multi_match": {
        "query":                "Quick brown fox",
        "type":                 "best_fields", //最佳字段匹配
        "fields":               [ "title", "body" ],//支持模糊匹配，也可以通过chapter_title^2增加字段权重
        "tie_breaker":          0.3,
        "minimum_should_match": "30%" 
    }
}
//most_fields 多数字段匹配
//cross_fields 类似all字段
```

短语查询：（临近搜索词查询），`match_phrase` 查询首先将查询字符串解析成一个词项列表，然后对这些词项进行搜索，但只保留那些包含 *全部* 搜索词项，且位置与搜索词项相同的文档。当一个字符串被分词后，分析器不但会返回词项列表，而且还会返回各词项在原始字符串中的位置和顺序，这些顺序信息会被存储在倒排索引中。

一个被认定为和短语 `quick brown fox` 匹配的文档，必须满足以下这些要求：

- `quick` 、 `brown` 和 `fox` 需要全部出现在域中。
- `brown` 的位置应该比 `quick` 的位置大 `1` 。
- `fox` 的位置应该比 `quick` 的位置大 `2` 。

可以通过加入参数slop来选择词条之间相隔多远还能匹配到文档。

对于多值字段我们使用短语查询时，其实es对于多值字段的分析和字符串是一样的，所以进行短语查询时即使查询短语中的字段存在不同的字段中依然可以进行匹配。随遇多值字段我们可以在配置映射时添加参数position_increment_gap，这个参数用来设置多值字段之间的位置间隔。词项之间的临近读会增加查询匹配分数。

```cmd
GET /my_index/my_type/_search
{
    "query": {
        "match_phrase": {
            "title": "quick brown fox"
        }
    }
}
```

结果集重新评分：

相当于在查询结果的基础上进行二次查询排序。

```cmd
GET /my_index/my_type/_search
{
    "query": {
        "match": {  
            "title": {
                "query":                "quick brown fox",
                "minimum_should_match": "30%"
            }
        }
    },
    "rescore": {
        "window_size": 50, （指定每个分片二次查询的结果集数量）
        "query": {         
            "rescore_query": {
                "match_phrase": {
                    "title": {
                        "query": "quick brown fox",
                        "slop":  50
                    }
                }
            }
        }
    }
}
```

创建倒排索引时除了对于单个单词进行创建，也可以对相邻单词对进行索引创建，通过创建特定的分析器。就相当于将相同字段按照不同的方式存储多份。

部分匹配：（类似模糊匹配，针对于单个词）

prefix查询，查询过程：

1. 扫描词列表并查找到第一个以 `W1` 开始的词。
2. 搜集关联的文档 ID 。
3. 移动到下一个词。
4. 如果这个词也是以 `W1` 开头，查询跳回到第二步再重复执行，直到下一个词不以 `W1` 为止。

```sense
GET /my_index/address/_search
{
    "query": {
        "prefix": {
            "postcode": "W1"
        }
    }
}
```

wildcard：通配符，regexp：正则表达式，匹配流程和前缀匹配一样。

输入即搜索：

使用match_phrase_prefix，类似短语搜索，只是会讲最后一个词进行部分匹配，和短语查询一样可以指定slop，并且为了防止前缀搜索匹配的词过大，可以通过参数max_expansions指定最大匹配词数。

相关性计算：

相关性计算的影响因素：

* 词频
* 逆向文档频率
* 字段长度归一值（字段越短，字段的权重值越高）

在创建索引时可以通过index_options设置成docs来禁用词频统计和词频位置。可以通过参数norms禁用字段长度归一值。要求精确查询的字段一般这两个都是默认禁用的。

在查询多个索引时，可以通过indices_boost参数配置索引的权重值：

```cmd
GET /docs_2014_*/_search 
{
  "indices_boost": { 
    "docs_2014_10": 3,
    "docs_2014_09": 2
  },
  "query": {
    "match": {
      "text": "quick brown fox"
    }
  }
}
```

权重提升查询：（boosting查询）

它接受 `positive` 和 `negative` 查询。只有那些匹配 `positive` 查询的文档罗列出来，对于那些同时还匹配 `negative` 查询的文档将通过文档的原始 `_score` 与 `negative_boost` 相乘的方式降级后的结果。

```cmd
GET /_search
{
  "query": {
    "boosting": {
      "positive": {
        "match": {
          "text": "apple"
        }
      },
      "negative": {
        "match": {
          "text": "pie tart fruit crumble tree"
        }
      },
      "negative_boost": 0.5
    }
  }
}
```

function_score查询：用来干预评分计算：

* weight：

  它接受 `positive` 和 `negative` 查询。只有那些匹配 `positive` 查询的文档罗列出来，对于那些同时还匹配 `negative` 查询的文档将通过文档的原始 `_score` 与 `negative_boost` 相乘的方式降级后的结果。

* field_value_factor

  它接受 `positive` 和 `negative` 查询。只有那些匹配 `positive` 查询的文档罗列出来，对于那些同时还匹配 `negative` 查询的文档将通过文档的原始 `_score` 与 `negative_boost` 相乘的方式降级后的结果。

* random_score

  为每个用户都使用一个不同的随机评分对结果排序，但对某一具体用户来说，看到的顺序始终是一致的。

* 衰减函数（linear， exp， gauss）

  将浮动值结合到评分 `_score` 中，例如结合 `publish_date` 获得最近发布的文档，结合 `geo_location` 获得更接近某个具体经纬度（lat/lon）地点的文档，结合 `price` 获得更接近某个特定价格的文档。

* script_score

