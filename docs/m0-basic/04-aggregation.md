> 上一篇我们讨论了关于多条件组合的查询,既然是查询,当然就就少不了需要对结果进行聚合统计
#### 聚合查询
在应用数据库的时候我们经常通过```avg``` ```sum``` ```count```等操作进行数据分析，在Elasticsearch中也是有这些操作。例如计算书签的平均分(score)：
```javascript
GET /bookmark/_search
{
    "size": 0,   //es默认会将参与运算的记录进行展示,但是通常聚合运算我们并不关心,因此设置size=0将其关闭
    "aggs": {    //aggs表明该操作是一个聚合操作
        "avg_score": {  //聚合的结果名称, 就像SELECT avg(score) avg_score FROM bookmark中的avg_score一样
            "avg": {   //聚合的函数
                "field": "score" //聚合的字段
            }
        }
    }
}
```
执行结果：
```javascript
{
    "took": 26,
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
        "hits": []   //因为size=0,这里没有返回参与计算的记录信息
    },
    "aggregations": {  //聚合结果
        "avg_score": {   //结果的key
            "value": 6.600000063578288   //结果值
        }
    }
}
```
> 不过需要注意的是,如果参与聚合的字段是字符串类型,默认情况出现报错，至于处理方式后面再谈,先看例子
```javascript
GET /bookmark/_search
{
    "size": 0,
    "aggs": {
        "label_group": { //进行分组操作
            "terms": {  //分组统计函数类似 group by count操作
                "field": "label"
            }
        }
    }
}
```
```javascript
{
    "error": {
        "root_cause": [
            {
                "type": "illegal_argument_exception",
                "reason": "Text fields are not optimised for operations that require per-document field data like aggregations and sorting, so these operations are disabled by default. Please use a keyword field instead. Alternatively, set fielddata=true on [label] in order to load field data by uninverting the inverted index. Note that this can use significant memory."
            }
        ],
        "type": "search_phase_execution_exception",
        "reason": "all shards failed",
        "phase": "query",
        "grouped": true,
        "failed_shards": [
            {
                "shard": 0,
                "index": "bookmark",
                "node": "4aZT2GWlQUCyRcW-9DQNdg",
                "reason": {
                    "type": "illegal_argument_exception",
                    "reason": "Text fields are not optimised for operations that require per-document field data like aggregations and sorting, so these operations are disabled by default. Please use a keyword field instead. Alternatively, set fielddata=true on [label] in order to load field data by uninverting the inverted index. Note that this can use significant memory."
                }
            }
        ],
        "caused_by": {
            "type": "illegal_argument_exception",
            "reason": "Text fields are not optimised for operations that require per-document field data like aggregations and sorting, so these operations are disabled by default. Please use a keyword field instead. Alternatively, set fielddata=true on [label] in order to load field data by uninverting the inverted index. Note that this can use significant memory.",
            "caused_by": {
                "type": "illegal_argument_exception",
                "reason": "Text fields are not optimised for operations that require per-document field data like aggregations and sorting, so these operations are disabled by default. Please use a keyword field instead. Alternatively, set fielddata=true on [label] in order to load field data by uninverting the inverted index. Note that this can use significant memory."
            }
        }
    },
    "status": 400
}
```
不出意外，您应该会出现这样的提示。这个不是语法错误，例如可以将```label```换成```score```试一下,发现结果就出来了：
```javascript
GET /bookmark/_search
{
    "size": 0,
    "aggs": {
        "label_group": {
            "terms": {
                "field": "score" //整个查询结构 只修改了此处的字段的名字,甚至分组的名称都没有改变
            }
        }
    }
}
执行结果：
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
            "value": 3,
            "relation": "eq"
        },
        "max_score": null,
        "hits": []
    },
    "aggregations": {
        "label_group": {
            "doc_count_error_upper_bound": 0,
            "sum_other_doc_count": 0,
            "buckets": [
                {
                    "key": 5.800000190734863,
                    "doc_count": 1
                },
                {
                    "key": 6.0,
                    "doc_count": 1
                },
                {
                    "key": 8.0,
                    "doc_count": 1
                }
            ]
        }
    }
}
```
至于报错信息，我们需要关注```Text fields are not optimised for operations that require per-document field data like aggregations and sorting, so these operations are disabled by default. Please use a keyword field instead. Alternatively, set fielddata=true on [label] in order to load field data by uninverting the inverted index. Note that this can use significant memory.```的消息。至于问题怎么解决,留待后续解密。