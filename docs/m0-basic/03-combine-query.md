> 上一篇我们讨论了了查询,但是似乎有点太简单了,毕竟只有一个条件,这在实际生产中,似乎不太实用,因此我们需要来讨论更加有用的组合查询。
##### 组合查询
在elasticsearch的条件组合用```bool```来完成组合,配合```must``` ```must_not``` ```should```来完成对条件的组装,其中```must```必须满足,相当于```AND```操作,```must_not```即相当于```<>```,当然```should```即相当于```OR```。例如:现在我们查询label中带有```更好书签```和```google```的书签信息：
```javascript
GET /bookmark/_search
{
    "query": {
        "bool": {
            "must": [
                {"match": {"label": "更好书签"}},
                {"match": {"label": "google"}}
            ]
        }
    }
}
```
该语句即相当于```SELECT * FROM bookmark WHERE label='更好书签' and label='google'```,当前该语句什么查不到,因为这个条件根本不成立:
```javascript
{
    "took": 4,
    "timed_out": false,
    "_shards": {
        "total": 1,
        "successful": 1,
        "skipped": 0,
        "failed": 0
    },
    "hits": {
        "total": {
            "value": 0,
            "relation": "eq"
        },
        "max_score": null,
        "hits": []
    }
}
```
但是如果改成```must_not```,即查询非```更好书签```和```google```的书签:
```javascript
GET /bookmark/_search
{
    "query": {
        "bool": {
            "must_not": [
                {"match": {"label": "更好书签"}},
                {"match": {"label": "google"}}
            ]
        }
    }
}
```
执行结果：
```javascript
{
    "took": 19,
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
        "max_score": 0.0,
        "hits": [
            {
                "_index": "bookmark",
                "_type": "_doc",
                "_id": "3",
                "_score": 0.0,
                "_source": {
                    "label": "bing搜索",
                    "url": "https://www.bing.com",
                    "created_at": "2022-02-23",
                    "score": 6.0
                }
            }
        ]
    }
}
```
如果改为```should```，则查询的是```更好书签```或者```google```:
```javascript
GET /bookmark/_search
{
    "query": {
        "bool": {
            "should": [
                {"match": {"label": "更好书签"}},
                {"match": {"label": "google"}}
            ]
        }
    }
}
```
执行结果：
```javascript
{
    "took": 4,
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
        "max_score": 4.0607862,
        "hits": [
            {
                "_index": "bookmark",
                "_type": "_doc",
                "_id": "1",
                "_score": 4.0607862,
                "_source": {
                    "label": "更好书签",
                    "desc": "更实用的书签导航",
                    "url": "https://www.bettershuqian.com",
                    "created_at": "2022-02-23",
                    "click_num": 10,
                    "score": 5.8
                }
            },
            {
                "_index": "bookmark",
                "_type": "_doc",
                "_id": "2",
                "_score": 1.6277175,
                "_source": {
                    "label": "google",
                    "url": "https://www.google.com",
                    "created_at": "2022-02-23",
                    "score": 8.0
                }
            }
        ]
    }
}
```
现在我们还需要更精确下查询范围,例如查询高质量```score>6```的书签：
```javascript
GET /bookmark/_search
{
    "query": {
        "bool": {
            "should": [
                {"match": {"label": "更好书签"}},
                {"match": {"label": "google"}}
            ],
            "filter": { //filter节点
                "range": {
                    "score": {
                        "gt": 6
                    }
                }
            }
        }
    }
}
```
执行结果：
```javascript
{
    "took": 14,
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
        "max_score": 1.6277175,
        "hits": [
            {
                "_index": "bookmark",
                "_type": "_doc",
                "_id": "2",
                "_score": 1.6277175,
                "_source": {
                    "label": "google",
                    "url": "https://www.google.com",
                    "created_at": "2022-02-23",
                    "score": 8.0
                }
            }
        ]
    }
}
```
> 小结：目前```bool```中包含了```must``` ```must_not``` ```should``` ```filter```节点信息用于组合查询条件

#### 短语查询
现在加入我们有这样一个查询,索引中保存的label分别为：```更好书签``` ```google``` ```bing```，思考一下会查询出结果吗
```javascript
GET /bookmark/_search
{
    "query": {
        "match": {
            "label": "更好google"
        }
    }
}
```
凭我们使用数据库的经验来说，这个查询会转换为```SELECT * FROM bookmark WHERE label='更好google'```，所以这个查询没有结果。但是经过试验发现
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
        "max_score": 2.0303931,
        "hits": [
            {
                "_index": "bookmark",
                "_type": "_doc",
                "_id": "1",
                "_score": 2.0303931,
                "_source": {
                    "label": "更好书签",
                    "desc": "更实用的书签导航",
                    "url": "https://www.bettershuqian.com",
                    "created_at": "2022-02-23",
                    "click_num": 10,
                    "score": 5.8
                }
            },
            {
                "_index": "bookmark",
                "_type": "_doc",
                "_id": "2",
                "_score": 1.6277175,
                "_source": {
                    "label": "google",
                    "url": "https://www.google.com",
                    "created_at": "2022-02-23",
                    "score": 8.0
                }
            }
        ]
    }
}
```
丫的，不仅查处了数据，竟然还有两条。这是为什么呢？其实这是Elaticsearch的分词的结果，Elasticsearch默认情况下按照英文单词，单个中文字符进行查词，因此在倒排索引的结构上就出现了更、好、google的三个拆词，因此搜索的时候以这三个词分别进行搜索，于是就搜出了前面看到的结果。但是这个并不是我们想要的结果，因此我们就需要按照度短语进行匹配了：```match_phrase```
```javascript
GET /bookmark/_search
{
    "query": {
        "match_phrase": {
            "label": "更好google"
        }
    }
}
```
此时的执行结果就是：
```javascript
{
    "took": 41,
    "timed_out": false,
    "_shards": {
        "total": 1,
        "successful": 1,
        "skipped": 0,
        "failed": 0
    },
    "hits": {
        "total": {
            "value": 0,
            "relation": "eq"
        },
        "max_score": null,
        "hits": []  //未找到结果
    }
}
```

#### 查询高亮
使用过搜索引擎的同学可能都注意到了，搜索引擎搜索出的结果中往往会将搜索的关键字进行高亮显示，这个是怎么玩的呢？其实这就是在匹配的关键字上添加了一些css样式而已。那么Elasticsearch怎么实现这个功能呢？
```javascript
GET /bookmark/_search
{
    "query": {
        "match_phrase": {
            "label": "更好"
        }
    },
    "highlight": {
        "fields": {
            "label": {}
        }
    }
}
```
执行的结果：
```javascript
{
    "took": 4,
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
        "max_score": 2.0303931,
        "hits": [
            {
                "_index": "bookmark",
                "_type": "_doc",
                "_id": "1",
                "_score": 2.0303931,
                "_source": {
                    "label": "更好书签",
                    "desc": "更实用的书签导航",
                    "url": "https://www.bettershuqian.com",
                    "created_at": "2022-02-23",
                    "click_num": 10,
                    "score": 5.8
                },
                "highlight": {
                    "label": [
                        "<em>更</em><em>好</em>书签" //添加了特殊标签的label
                    ]
                }
            }
        ]
    }
}
```