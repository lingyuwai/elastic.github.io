> 前面我们学习了关于es的一些基础的增删改查的操作,现在我们需要来讨论稍微复杂一些的查询 例如多字段的查询 精确查询
#### 多字段查询
在前面的操作中我们多数都是在根据某个字段进行查询，但是实际开发过程中事情没有那么容易，很多时候我们需要执行```SELECT * FROM article WHERE title like '%elasticsearch%' or `desc` like '%elasticsearch%'```这样的查询,Elasticsearch中怎么实现呢?
先通过批量操作创建一些数据
```javascript
PUT /article/_bulk
{"create": {"_id": 1}}
{"title": "ElasticSearch快速开始", "desc": "本篇内容主要讨论关于elasticsearch的下载安装和简单的索引增删和文档的增删改查操作"}
{"create": {"_id": 2}}
{"title": "redis快速入门", "desc": "本文主要讨论关于redis的快速入门,和大家一起快速进行redis的世界"}
{"create": {"_id": 3}}
{"title": "如何快速入门一门新的语言", "desc": "本篇我们一起来探讨如何快速的掌握一门语言或工具,例如elasticsearch"}
//这里有一个空行
```
用elasticsearch实现上面的SQL查询：
```javascript
GET /article/_search
{
    "query": {
        "multi_match": {
            "query": "elasticsearch",
            "fields": ["title", "desc"]
        }
    }
}
```
查询结果：
```javascript
{
    "took": 135,
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
        "max_score": 1.1276041,
        "hits": [
            {
                "_index": "article",
                "_type": "_doc",
                "_id": "1",
                "_score": 1.1276041,
                "_source": {
                    "title": "ElasticSearch快速开始",
                    "desc": "本篇内容主要讨论关于elasticsearch的下载安装和简单的索引增删和文档的增删改查操作"
                }
            },
            {
                "_index": "article",
                "_type": "_doc",
                "_id": "3",
                "_score": 0.49077308,
                "_source": {
                    "title": "如何快速入门一门新的语言",
                    "desc": "本篇我们一起来探讨如何快速的掌握一门语言或工具,例如elasticsearch"
                }
            }
        ]
    }
}
```

#### 精确匹配
这里的精确匹配的意思经过测试(v7.16.3)为被搜索的内容(即关键字)不能进行分词的匹配。有点懵，上例子：
```javascript
GET /article/_search
{
    "query": {
        "term": {
            "title": "redis" //这里的redis是不能被分词的一个单词
        }
    }
}
```
这样就可以搜出结果，但如果按照我们想象的在传统数据库中的搜索执行：
```javascript
GET /article/_search
{
    "query": {
        "term": {
            "title": "redis快速入门"  //这里的关键字会被进行拆词处理
        }
    }
}
```
因此我们这个查询实际上是不会有返回结果的，但是如果我们改成
```javascript
GET /article/_search
{
    "query": {
        "term": {
            "title": "速" //一个中文字符无法被拆词
        }
    }
}
```
这样就又匹配出结果了。这个场景似乎有点眼熟，这个不是之前在映射中讨论过的keyword吗?
> 这里的搜索字段能不能被拆分其实和mapping中字段的type有一定的关系。总结起来就是如果type是text等会进行拆词处理的类型，那么在查询时字段就只能携带最小单位的关键词，英文就是一个单词,中文就是一个汉字,当然数值型天然具有不可分词的特性,自身就已经是一个整体了。而如果是keyword,查询字段携带的关键词就需要和mapping中的字段一致。

还记得我们设置的user的mapping吗
```javascript
{
    "properties": {
        "nickname": {
            "type": "text",
            "index": true
        },
        "sex": {
            "type": "keyword",
            "index": true
        },
        "tel": {
            "type": "text",
            "index": false
        },
        "username": {
            "type": "keyword",
            "index": false
        }
    }
}
```
我们设置了```sex```为keyword类型,并且记录中的值为```女士```,因此我们如果进行精确匹配就需要：
```javascript
GET /user/_search
{
    "query": {
        "term": {
            "sex": "女士"
        }
    }
}
```
执行结果：
```javascript
{
    "took": 5,
    "timed_out": false,
    "_shards": {
        "total": 1,
        "successful": 1,
        "skipped": 0,
        "failed": 0
    },
    "hits": {
        "total": {
            "value": 1,
            "relation": "eq"
        },
        "max_score": 0.13353139,
        "hits": [
            {
                "_index": "user",
                "_type": "_doc",
                "_id": "5",
                "_score": 0.13353139,
                "_source": {
                    "nickname": "那个人很优秀",
                    "sex": "女士",
                    "tel": "1111111",
                    "username": "优秀的女人"
                }
            }
        ]
    }
}
```

#### 多关键词精确查询
多关键词查询用的是```terms```进行设置：
```javascript
GET /article/_search
{
    "query": {
        "terms": {
            "title": ["elasticsearch", "reids"]
        }
    }
}
```
执行结果
```javascript
{
    "took": 6,
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
                "_index": "article",
                "_type": "_doc",
                "_id": "1",
                "_score": 1.0,
                "_source": {
                    "title": "ElasticSearch快速开始",
                    "desc": "本篇内容主要讨论关于elasticsearch的下载安装和简单的索引增删和文档的增删改查操作"
                }
            },
            {
                "_index": "article",
                "_type": "_doc",
                "_id": "2",
                "_score": 1.0,
                "_source": {
                    "title": "redis快速入门",
                    "desc": "本文主要讨论关于redis的快速入门,和大家一起快速进行redis的世界"
                }
            }
        ]
    }
}
```
> 由于精确查询的特点,因此精确查询更多的应用场景是根据数值型字段进行查询,实现```=```或```in```的查询效果

#### 过滤查询结果
前面我们已经讨论了，通过```_source```可以对查询的字段进行限制,这个和我们在用SQL指定字段查询一致,不过大家在开发过程中应该会有一个痛处就是如果我们需要查询的字段特别多,就比较麻烦了,我们需要挨个列出来十几个字段。现在Elasticsearch帮我们解决了这个困难,它允许设置需要排除```excludes```的字段和需要查询```includes```的字段,这样在控制字段的时候就更加轻松愉快了
```javascript
GET /player/_search
{
    "query": {
        "match": {
            "username": "faker"
        }
    },
    "_source": {
        "excludes": ["champions"] //排除的字段
    }
}
```
查询结果：
```javascript
{
    "took": 5,
    "timed_out": false,
    "_shards": {
        "total": 1,
        "successful": 1,
        "skipped": 0,
        "failed": 0
    },
    "hits": {
        "total": {
            "value": 1,
            "relation": "eq"
        },
        "max_score": 0.6931471,
        "hits": [
            {
                "_index": "player",
                "_type": "_doc",
                "_id": "1",
                "_score": 0.6931471,
                "_source": {
                    "nickname": "大魔王",
                    "hero": "影流之主",
                    "team": "SKT",
                    "age": "26",
                    "username": "faker"
                }
            }
        ]
    }
}
```
当然如果两个都指定了,那么includes的优先级会高于excludes,如果一个属性既被排除,又被包含,最终将会排除该字段

#### 范围查询
范围查询采用```range```来表示范围查询。先添加以下的数据
```javascript
PUT /products/_bulk
{"create": {"_id": 1}}
{"name": "小米2", "category": "小米", "price": 2999.00}
{"create": {"_id": 2}}
{"name": "红米9", "category": "小米", "price": 3563.00}
{"create": {"_id": 3}}
{"name": "华为p9", "category": "华为", "price": 4999.00}
{"create": {"_id": 4}}
{"name": "iphone 12", "category": "苹果", "price": 7999.00}
{"create": {"_id": 5}}
{"name": "诺基亚N97", "category": "诺基亚", "price": 2500.00}
{"create": {"_id": 6}}
{"name": "海尔", "category": "海尔", "price": 800.00}
```
查询价格小于2500的手机：
```javascript
GET /products/_search
{
    "query": {
        "range": {
            "price": {
                "lte": 2500
            }
        }
    }
}
```
查询结果：
```javascript
{
    "took": 2,
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
                "_index": "products",
                "_type": "_doc",
                "_id": "5",
                "_score": 1.0,
                "_source": {
                    "name": "诺基亚N97",
                    "category": "诺基亚",
                    "price": 2500.00
                }
            },
            {
                "_index": "products",
                "_type": "_doc",
                "_id": "6",
                "_score": 1.0,
                "_source": {
                    "name": "海尔",
                    "category": "海尔",
                    "price": 800.00
                }
            }
        ]
    }
}
```
```range```的关键字主要包括：```lt小于``` ```lte小于等于``` ```gt大于``` ```gte大于等于```

#### 模糊查询
正如我们平时用搜索引擎(谷歌、百度等)的时候,我们并不会真正记得词条的真正长相,很多时候我们甚至输入的是错别字,但是搜索引擎依然能够帮我们搜出内容来，有时候这些内容刚好还是我们误打误撞搜出来的。这其实就是全文检索的模糊匹配。搜索引擎根据我们输入的内容进行纠错处理然后检索并返回内容。Elasticsearch也提供了类似的功能，即用关键字```fuzzy```实现即可。而其中的错误纠正被称为编辑距离```fuzziness```。
```javascript
GET /player/_search
{
    "query": {
        "fuzzy": {
            "username": {"value": "fake"}
        }
    }
}
```
设置编辑距离：
```javascript
GET /player/_search
{
    "query": {
        "fuzzy": {
            "username": {
                "value": "fake",
                "fuzziness": 0  //尝试纠正错误的次数 不纠错即相当于按照当前字段值进行搜索,默认由系统根据语境来决定
            }
        }
    }
}
```

#### 数据排序
之前我们讨论了数据的排序,现在我们来研究排序的简化写法:
```javascript
GET /products/_search
{
    "sort": [
        {"price": "desc"} //简化排序设置
    ]
}
```
或者更简化
```javascript
GET /products/_search
{
    "sort": {
        "price": "desc"
    }
}
```

#### 高亮查询
前面我们讨论了简单的高亮操作
```javascript
GET /products/_search
{
    "query": {
        "match": {
            "name": "小米"
        }
    },
    "highlight": {
        "fields": {"name": {}} //设置需要高亮的字段
    }
}
```
这是我们之前设置的高亮,但是系统生成的高亮标签有时不一定符合我们的预期,因此我们需要进行自定义高亮,例如
```javascript
GET /products/_search
{
    "query": {
        "multi_match": {
            "query": "小米",
            "fields": ["name", "category"]
        }
    },
    "highlight": {
        "fields": {
            "name": {},
            "category": {"pre_tags": "<p>", "post_tags": "</p>"}
        },
        "pre_tags": "<i class=\"red\">",  //修改高亮标签为class为red的i标签 这个标记可以和具体的字段进行绑定,即每个字段设置不同的tag
        "post_tags": "</i>"
    }
}
```

#### 聚合操作
在前面我们讨论了基础的聚合操作```max``` ```min``` ```avg``` ```sum``` ```cardinality```等操作,但是我们都是单体执行，如果要合并在一起,似乎停麻烦的,有什么简化的办法没?这就是```state```操作
```javascript
GET /products/_search
{
    "size": 0,
    "aggs": {
        "result": {
            "stats": {"field": "price"}
        }
    }
}
```
执行结果：
```javascript
{
    "took": 8,
    "timed_out": false,
    "_shards": {
        "total": 1,
        "successful": 1,
        "skipped": 0,
        "failed": 0
    },
    "hits": {
        "total": {
            "value": 6,
            "relation": "eq"
        },
        "max_score": null,
        "hits": []
    },
    "aggregations": {
        "result": {
            "count": 6,   //总记录数
            "min": 800.0, //最低价格
            "max": 7999.0, //最高价格
            "avg": 3810.0, //平均价格
            "sum": 22860.0 //总价格
        }
    }
}
```

#### 桶聚合查询
对于桶聚合查询这个词的翻译已经不太可考,但是这个词大致表示的是SQL中的```group by```操作。因此按照词我们可以在products中再添加一条数据
```javascript
PUT /products/_bulk
{"create": {}} //在创建文档时可以不添加ID 由系统自动生成
{"name": "iphone 5s", "category": "苹果", "price": 2999.00}
```
统计当前的价格分组情况：
```javascript
GET /products/_search
{
    "aggs": {
        "price_group": {
            "terms": {"field": "price"}
        }
    },
    "size": 0
}
```
查询结果：
```javascript
{
    "took": 3,
    "timed_out": false,
    "_shards": {
        "total": 1,
        "successful": 1,
        "skipped": 0,
        "failed": 0
    },
    "hits": {
        "total": {
            "value": 7,
            "relation": "eq"
        },
        "max_score": null,
        "hits": []
    },
    "aggregations": {
        "price_group": {
            "doc_count_error_upper_bound": 0,
            "sum_other_doc_count": 0,
            "buckets": [
                {
                    "key": 2999.0,
                    "doc_count": 2  //该价格有两件商品
                },
                {
                    "key": 800.0,
                    "doc_count": 1
                },
                {
                    "key": 2500.0,
                    "doc_count": 1
                },
                {
                    "key": 3563.0,
                    "doc_count": 1
                },
                {
                    "key": 4999.0,
                    "doc_count": 1
                },
                {
                    "key": 7999.0,
                    "doc_count": 1
                }
            ]
        }
    }
}
```
