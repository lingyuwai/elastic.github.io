上一篇我们讨论了Elasticsearch的基础增删改查,Elasticsearch最核心的功能就是查询,因此接下来我们来讨论更详细的查询

#### match匹配查询
匹配和传统数据库的```like %***%```操作类似，但是性能远高于传统数据库的like模糊匹配，匹配操作有两种形式：

第一种：
```javascript
GET /bookmark/_search?q=label:更好书签
```
搜索结果：
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
            }
        ]
    }
}
```
经过操作我们也发现了,这种方式不太好操作,尤其当出现中文的时候,因此我们常用的是第二种方式：
```javascript
GET /bookmark/_search
{
    "query": {
        "match": {
            "label": "更好书签"
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
            "value": 1,
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
            }
        ]
    }
}
```

#### match_all匹配所有
查询总会有不带查询条件的时候,此时就需要进行全量查询了即：
```javascript
GET /bookmark/_search
{
    "query": {
        "match_all": {}
    }
}
```
通过添加数据我们发现，这个操作会查询出超过我们预期的数据,因此在执行该类操作时,通常都会选择进行分页处理：
```javascript
GET /bookmark/_search
{
    "query": {
        "match_all": {}
    },
    "from": 2, //查询的起始位置 现在查询的是第2页的数据
    "size": 2  //单次查询条数
}
```
```from```的算法和数据库分页的算法一致：(page-1)*pageSize,也就是```(page-1)*size```
查询结果:
```javascript
{
    "took": 7,
    "timed_out": false,
    "_shards": {
        "total": 1,
        "successful": 1,
        "skipped": 0,
        "failed": 0
    },
    "hits": {
        "total": {
            "value": 3,   //数据总量
            "relation": "eq"
        },
        "max_score": 1.0,
        "hits": [
            {
                "_index": "bookmark",
                "_type": "_doc",
                "_id": "1",
                "_score": 1.0,
                "_source": {
                    "label": "更好书签",
                    "desc": "更实用的书签导航",
                    "url": "https://www.bettershuqian.com",
                    "created_at": "2022-02-23",
                    "click_num": 10,
                    "score": 5.8
                }
            }
        ]
    }
}
```

#### 过滤查询结果
通过操作我们发现，似乎结果包含了整个文档,但是文档中的某些信息我们不希望看到,毕竟浪费流量,还有可能导致数据泄露,怎么办呢,用```_source```进行过滤即可：
```javascript
GET /bookmark/_search
{
    "query": {
        "match_all": {}
    },
    "from": 0,
    "size": 2,
    "_source": ["label", "url"] //指定要查询的字段信息
}
```
执行结果：
```javascript
{
    "took": 23,
    "timed_out": false,
    "_shards": {
        "total": 1,
        "successful": 1,
        "skipped": 0,
        "failed": 0
    },
    "hits": {
        "total": {
            "value": 3,
            "relation": "eq"
        },
        "max_score": 1.0,
        "hits": [
            {
                "_index": "bookmark",
                "_type": "_doc",
                "_id": "2",
                "_score": 1.0,
                "_source": { //这里的返回结果就是指定的字段了
                    "label": "google",
                    "url": "https://www.google.com"
                }
            },
            {
                "_index": "bookmark",
                "_type": "_doc",
                "_id": "3",
                "_score": 1.0,
                "_source": {
                    "label": "bing搜索",
                    "url": "https://www.bing.com"
                }
            }
        ]
    }
}
```

#### 排序查询结果
通常我们在执行列表查询的时候都会按照一定的顺序来显示数据：
```javascript
GET /bookmark/_search
{
    "query": {
        "match_all": {}
    },
    "from": 0,
    "size": 2,
    "sort": {  //表示排序
        "score": { //排序的字段
            "order": "desc" //排序方式
        }
    }
}
```
执行结果：
```javascript
{
    "took": 168,
    "timed_out": false,
    "_shards": {
        "total": 1,
        "successful": 1,
        "skipped": 0,
        "failed": 0
    },
    "hits": {
        "total": {
            "value": 3,
            "relation": "eq"
        },
        "max_score": null,
        "hits": [
            {
                "_index": "bookmark",
                "_type": "_doc",
                "_id": "2",
                "_score": null,
                "_source": {
                    "label": "google",
                    "url": "https://www.google.com",
                    "created_at": "2022-02-23",
                    "score": 8.0
                },
                "sort": [
                    8.0
                ]
            },
            {
                "_index": "bookmark",
                "_type": "_doc",
                "_id": "3",
                "_score": null,
                "_source": {
                    "label": "bing搜索",
                    "url": "https://www.bing.com",
                    "created_at": "2022-02-23",
                    "score": 6.0
                },
                "sort": [
                    6.0
                ]
            }
        ]
    }
}
```