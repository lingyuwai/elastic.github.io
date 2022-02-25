#### Elasticsearch基础概念
 - es采用倒排索引的方式进行索引，因此速度很快
 - GET  PUT  DELETE具有幂等性 不允许重复操作
 - POST无幂等性 可重复操作
 - 在Elasticsearch中GET请求是可以携带请求体的!!!
```javascript
//Elasticsearch安装信息
{
    "name": "pawn.local",
    "cluster_name": "elasticsearch", //集群名称
    "cluster_uuid": "POekwKn2SDup_vGyoFOIWA", //集群ID
    "version": {
        "number": "7.16.3", //版本号
        "build_flavor": "default",
        "build_type": "tar",
        "build_hash": "4e6e4eab2297e949ec994e688dad46290d018022",
        "build_date": "2022-01-06T23:43:02.825887787Z",
        "build_snapshot": false,
        "lucene_version": "8.10.1",
        "minimum_wire_compatibility_version": "6.8.0",
        "minimum_index_compatibility_version": "6.0.0-beta1"
    },
    "tagline": "You Know, for Search"
}
```

Elasticsearch与传统数据库对比：

| elasticsearch | 传统数据库 |
|:-------------:|:--------:|
|   index(索引) |   table(table)    |
|   type(类型)默认```_doc```  |   database(数据库) |
|   document(文档) |    row(数据行) |

```声明： 后续命名行中的[param]通常表示该参数位可选```
#### 倒排索引
在讨论Elasticsearch之前我们先来讨论让Elasticserch快速检索的关键算法```倒排索引```。在传统数据库中的我们通常都是根据ID去查询数据,而倒排索引就是通过数据查询ID。怎么讲?例如：```the weather is well today```  ```the weather is cool yesterday```。当我们在Elasticserch中存储这些文本的时候,会对消息进行拆分(分词)。第一句话被处理为

| 分词 | ids|
|:----:|:---:|
| the | 1 |
| weather | 1 |
| is | 1 |
| well | 1 |
| today | 1 |

接着在保存第二句话的进行处理:

| 分词 | ids|
|:----:|:---:|
| the | 1,2 |
| weather | 1,2 |
| is | 1,2 |
| well | 1 |
| today | 1 |
| cool | 2 |
| yesterday | 2 |

这样当我们搜索```today```就会直接从这个表中直接找到today对应的ID是1,然后就直接去找ID为1的数据,这样就很快的找到了数据。这就是Elasticsearch搜索的基本原理，当然真实的Elasticsearch实际还做了很多其他的工作，因为即使是上面的操作,在海量数据面前也顶不住。

#### 创建空索引
```PUT /index_name```

该方式用于创建一个没有内容的索引，即相当于创建一个空表，该操作不携带请求体,该操作主要用于设置一个新的索引，并初始化一些配置信息(高级部分讨论):

```PUT /shopping``` 执行结果：
```javascript
{
    "acknowledged": true,
    "shards_acknowledged": true,
    "index": "shopping"
}
```
elaticsearch也支持通过POST方式创建带数据的索引：
```POST /index_name/type_name/[id]```
```javascript
POST /category/bookmark
{
    "title": "探索科学",
    "desc": "该分类主要用于收录著名的搜索引擎"
}
```
执行结果：
```javascript
{
    "_index": "category",  //创建的索引名称
    "_type": "bookmark",   //类型
    "_id": "WDGgJn8BCzb-oGcCB0y4", //文档ID,记录的唯一标识
    "_version": 1,  //版本号
    "result": "created", //状态
    "_shards": {
        "total": 2,
        "successful": 1,
        "failed": 0
    },
    "_seq_no": 0,
    "_primary_term": 1
}
```

#### 查看索引
查询我们创建的索引主要通过无请求体的GET请求来实现：```GET /index_name```
```javascript
GET /shopping
```
得到响应结果：
```javascript
{
    "shopping": {
        "aliases": {},
        "mappings": {},
        "settings": {
            "index": {
                "routing": {
                    "allocation": {
                        "include": {
                            "_tier_preference": "data_content"
                        }
                    }
                },
                "number_of_shards": "1",
                "provided_name": "shopping",
                "creation_date": "1645618136817",
                "number_of_replicas": "1",
                "uuid": "b0rhuyr0Tzyt-0-9cUWqtg",
                "version": {
                    "created": "7160399"
                }
            }
        }
    }
}
```
查看系统中创建的索引:

```GET /_cat/indices?v```
```
health status index            uuid                   pri rep docs.count docs.deleted store.size pri.store.size
green  open   .geoip_databases LVh5PZP1QXGq4oPP-Dp2Mg   1   0         41            5     38.6mb         38.6mb
yellow open   category         BLXwVuGtSS-Wm1uIhaOCyg   1   1          1            0      4.8kb          4.8kb
yellow open   shopping         b0rhuyr0Tzyt-0-9cUWqtg   1   1          0            0       226b           226b
```
#### 删除索引
删除索引主要通过DELETE操作进行,删除索引分为单体删除,定向删除以及删除所有，

单体删除:```DELETE /index_name```
```javascript
DELETE /category
```
删除结果：
```javascript
{
    "acknowledged": true
}
```

定向删除:```DELETE /index_name,index_name,...```

首先通过```PUT```创建5个索引分别为```shopping1``` ```shopping2``` ```shopping3``` ```shopping4``` ```shopping5```
```javascript
PUT /shopping1
PUT /shopping2
PUT /shopping3
PUT /shopping4
PUT /shopping5
```
删除指定的索引```shopping1,shopping2,shopping3```：```DELETE /shopping1,shopping2,shopping3```
删除所有:```DELETE /_all``` 该指令将删除系统中的所有索引,因此在生产环境通常会禁用该操作。

在经过上述操作之后我们再次查看系统的中的索引,发现除了系统内置的都被我们清空了,为了后续操作方便,我们通过``` PUT /bookmark```再次创建一个索引。

#### 添加数据(添加文档)
添加数据通常采用POST进行操作：
```javascript
POST /index_name/_doc/[id]
{数据结构体}
```
若未指定```id```,则由系统自动生成全局唯一ID：
```javascript
POST /bookmark/_doc
{
    "label": "更好书签",
    "desc": "更实用的书签导航",
    "url": "https://www.bettershuqian.com",
    "created_at": "2022-02-23",
    "click_num": 10,
    "score": 5.8
}
```
添加结果：
```javascript
{
    "_index": "bookmark",
    "_type": "_doc",
    "_id": "WTHGJn8BCzb-oGcCCEzS",
    "_version": 1,
    "result": "created",
    "_shards": {
        "total": 2,
        "successful": 1,
        "failed": 0
    },
    "_seq_no": 0,
    "_primary_term": 1
}
```
在多次提交后我们发现这个系统自动生成的ID在使用的时候存在很多不方便的,于是我们需要自定义一个ID, 指定ID的情况下可以通过```PUT```添加记录:
```javascript
POST|PUT /bookmark/_doc/1
{
    "label": "elasticsearch镜像",
    "desc": "更实用的书签导航",
    "url": "https://es.xingquzu.com",
    "created_at": "2022-02-23",
    "click_num": 10,
    "score": 5.8
}
```
结果：
```javascript
{
    "_index": "bookmark",
    "_type": "_doc",
    "_id": "1", //设置为我们需要的业务ID
    "_version": 1,
    "result": "created",
    "_shards": {
        "total": 2,
        "successful": 1,
        "failed": 0
    },
    "_seq_no": 1,
    "_primary_term": 1
}
```

#### 查看文档
查看上一步添加的ID=1的文档信息：```GET /index_name/type_name/id```
```javascript
GET /index_name/_doc/1
```
结果：
```javascript
{
    "_index": "bookmark",
    "_type": "_doc",
    "_id": "1",
    "_version": 1,
    "_seq_no": 1,
    "_primary_term": 1,
    "found": true,
    "_source": {
        "label": "elasticsearch镜像",
        "desc": "更实用的书签导航",
        "url": "https://es.xingquzu.com",
        "created_at": "2022-02-23",
        "click_num": 10,
        "score": 5.8
    }
}
```

#### 更新文档
既然文档就是数据库中行的概念，那就有需要更新的时候,文档的更新分为全量更新和局部更新,但值得一提的是根据elasticsearch官方文档说明,似乎没有真正的更新,所谓的更新在es内部都是删除再创建的过程。不过无所谓,我们只需要关注结果就好：

```全量更新```

全量更新具有一定的幂等性，因此可以使用PUT或POST来执行,推荐用```PUT```避免混淆

```javascript
PUT /bookmark/_doc/1
{
    "label": "elasticsearch镜像",
    "desc": "更实用的书签导航",
    "url": "https://es.xingquzu.com",
    "created_at": "2022-02-23",
    "click_num": 10,
    "score": 7.2
}
```
执行结果：
```javascript
{
    "_index": "bookmark",
    "_type": "_doc",
    "_id": "1",
    "_version": 2, //更新后版本发生改变
    "result": "updated",   //执行结果为updated 还记得首次创建的时候是什么吗 是created
    "_shards": {
        "total": 2,
        "successful": 1,
        "failed": 0
    },
    "_seq_no": 2,
    "_primary_term": 1
}
```

```局部更新```
```javascript
POST /bookmark/_doc/1
{
    "doc": {
        "desc": "一个优秀的书签导航网站,更多优秀的功能需要你的开发和探索",
        "score": 9
    }
}
```
结果：
```javascript
{
    "_index": "bookmark",
    "_type": "_doc",
    "_id": "1",
    "_version": 3, //更新后版本再次更新
    "result": "updated", //更新
    "_shards": {
        "total": 2,
        "successful": 1,
        "failed": 0
    },
    "_seq_no": 3,
    "_primary_term": 1
}
```

#### 查看文档的所有内容
Elasticsearch基于性能考虑在查看整个文档的时候并不会返回文档的所有内容 仅返回前10条数据,```GET /index_name/[type_name]/_search```,其中```type_name```通常不填写,除非在特殊情况有冲突
```javascript
GET /bookmark/_search
```
```javascript
{
    "took": 27, //查询时间 单位ms
    "timed_out": false,
    "_shards": {
        "total": 1,
        "successful": 1,
        "skipped": 0,
        "failed": 0
    },
    "hits": {
        "total": {
            "value": 2,
            "relation": "eq"
        },
        "max_score": 1.0,
        "hits": [
            {
                "_index": "bookmark",
                "_type": "_doc",
                "_id": "WTHGJn8BCzb-oGcCCEzS",
                "_score": 1.0,
                "_source": {
                    "label": "百度搜索",
                    "url": "http://www.baidu.com",
                    "created_at": "2022-02-23",
                    "click_num": 10,
                    "score": 5.8
                }
            },
            {
                "_index": "bookmark",
                "_type": "_doc",
                "_id": "1",
                "_score": 1.0,
                "_source": {
                    "doc": {
                        "desc": "一个优秀的书签导航网站,更多优秀的功能需要您的探索和开发",
                        "score": 9
                    }
                }
            }
        ]
    }
}
```

#### 删除文档
删除文档的内容主要是根据文档的ID进行删除

```DELETE /index_name/type_name/id```
```javascript
DELETE /bookmark/_doc/WTHGJn8BCzb-oGcCCEzS
```
执行结果：
```javascript
{
    "_index": "bookmark",
    "_type": "_doc",
    "_id": "WTHGJn8BCzb-oGcCCEzS", //被删除记录的ID
    "_version": 2,
    "result": "deleted",  //删除
    "_shards": {
        "total": 2,
        "successful": 1,
        "failed": 0
    },
    "_seq_no": 9,
    "_primary_term": 1
}
```
