> 在上一篇我们简单讨论了聚合操作，现在我们来讨论映射
#### 映射 - mapping
什么是映射呢?在传统数据库例如MySQL在创建字段的时候，为了加速业务查询，我们通常都会添加一些额外的索引信息，这些个额外的索引信息就和这里的映射有一点相似的地方。只是Elasticsearch添加映射信息，不只是为了加速查询，有时候还会阻止查询。例如：

创建映射前需要手动创建索引：
```javascript
PUT /user
```
配置映射
```javascript
PUT /user/_mapping  //通过_mapping关键字配置映射信息
{
    "properties": {  //属性组
        "nickname": {  //属性1
            "type": "text",  //属性类型 text=>文本,进行分词处理
            "index": true    //是否能够进行索引 true => 是  false => 否
        },
        "sex": {
            "type": "keyword", //keyword 进行特殊分词,通常表示这个字段值是一个整体
            "index": true
        },
        "tel": {
            "type": "keyword",
            "index": false
        },
        "username": {
            "type": "text",
            "index": false
        }
    }
}
```
查询映射结果：
```javascript
GET /bookmark/_mapping
```
结果：
```javascript
{
    "user": {
        "mappings": {
            "properties": {
                "nickname": {
                    "type": "text"
                },
                "sex": {
                    "type": "keyword"
                },
                "tel": {
                    "type": "keyword",
                    "index": false
                },
                "username": {
                    "type": "text",
                    "index": false
                }
            }
        }
    }
}
```
添加一条记录(文档):
```javascript
PUT /user/_create/1
{
    "nickname": "那个人很优秀",
    "sex": "女士",
    "tel": "1111111",
    "username": "优秀的女人"
}
```
通过```nickname```查询:
```javascript
GET /user/_search
{
    "query": {
        "match": {
            "nickname": "优秀"
        }
    }
}
```
```javascript
{
    "took": 377,
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
        "max_score": 0.5753642,
        "hits": [
            {
                "_index": "user",
                "_type": "_doc",
                "_id": "1",
                "_score": 0.5753642,
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
搜出结果,和之前的搜索似乎没有什么不同。接下来试下通过```sex```搜索呢？
```javascript
GET /user/_search
{
    "query": {
        "match": {
            "sex": "女"
        }
    }
}
```
执行后并没有返回结果,因为keyword表示sex的值是一个整体,在分词过程中保存的是```女士```。因此
```javascript
GET /user/_search
{
    "query": {
        "match": {
            "sex": "女士"
        }
    }
}
```
此时也就搜索到结果了。接下来搜索电话呢?
```javascript
GET /user/_search
{
    "query": {
        "match": {
            "tel": "1111111"
        }
    }
}
```
执行结果：
```javascript
{
    "error": {
        "root_cause": [
            {
                "type": "query_shard_exception",
                "reason": "failed to create query: Cannot search on field [tel] since it is not indexed.",
                "index_uuid": "UQBe89XoSRebl1vCthwQCA",
                "index": "user"
            }
        ],
        "type": "search_phase_execution_exception",
        "reason": "all shards failed",
        "phase": "query",
        "grouped": true,
        "failed_shards": [
            {
                "shard": 0,
                "index": "user",
                "node": "4aZT2GWlQUCyRcW-9DQNdg",
                "reason": {
                    "type": "query_shard_exception",
                    "reason": "failed to create query: Cannot search on field [tel] since it is not indexed.",
                    "index_uuid": "UQBe89XoSRebl1vCthwQCA",
                    "index": "user",
                    "caused_by": {
                        "type": "illegal_argument_exception",
                        "reason": "Cannot search on field [tel] since it is not indexed."
                    }
                }
            }
        ]
    },
    "status": 400
}
```
报错了,是的,Elasticsearch中没有索引的字段是不能进行搜索,正如提示消息```failed to create query: Cannot search on field [tel] since it is not indexed.```,不能搜索未添加索引的tel字段。同理对username也没法进行搜索

#### 批量操作
一直以来我们每次都只能操作单个数据,挺麻烦的。现在让我们来研究批量操作神器 - ```bulk```。

##### 批量新增 - create
新增用的是```create```关键字,表示没有记录才会进行创建
```javascript
POST /players/_bulk
{"create": {"_id": 1}} //设置记录在索引(数据库表)中的ID
{"username": "faker", "team": "SKT", "hero": "影流之主", "champions": ["2013年全球总决赛冠军", "2015年全球总决赛冠军", "2016年全球总决赛冠军"]}
{"create": {"_id": 2}}
{"username": "pawn", "team": "SSW", "hero": "刀锋之影", "champions": ["2014年全球总决赛冠军"]}
{"create": {"_id": 3}}
{"username": "mata", "team": "SSW", "hero": "魂锁典狱长", "champions": ["2014年全球总决赛冠军"]}
//注意：这一行是空白的 语法要求最后一组数据的后面必须要留出一行空行表示后面没有数据了 和http协议的空行相同的含义
```
> 以上的操作添加了3组数据,且在最后一组数后边必须留一个换行\n ```create```表示创建
以下是成功添加的数据,如果数据没有添加成功,则不会出现在下方的```items```节点中
```javascript
{
    "took": 321,
    "errors": false,
    "items": [
        {
            "create": {
                "_index": "player",
                "_type": "_doc",
                "_id": "1",
                "_version": 1,
                "result": "created",
                "_shards": {
                    "total": 2,
                    "successful": 1,
                    "failed": 0
                },
                "_seq_no": 0,
                "_primary_term": 1,
                "status": 201
            }
        },
        {
            "create": {
                "_index": "player",
                "_type": "_doc",
                "_id": "2",
                "_version": 1,
                "result": "created",
                "_shards": {
                    "total": 2,
                    "successful": 1,
                    "failed": 0
                },
                "_seq_no": 1,
                "_primary_term": 1,
                "status": 201
            }
        },
        {
            "create": {
                "_index": "player",
                "_type": "_doc",
                "_id": "3",
                "_version": 1,
                "result": "created",
                "_shards": {
                    "total": 2,
                    "successful": 1,
                    "failed": 0
                },
                "_seq_no": 2,
                "_primary_term": 1,
                "status": 201
            }
        }
    ]
}
```

##### 批量新增 - index
在数据批量新增的过程中,elasticsearch设计了另一种新增方式,就是覆盖,如果用```index```的方式,如果没有数据则新建,如果目标数据已经存在了,则会利用现有数据进行覆盖
```javascript
POST /player/_bulk
{"index": {"_id": 3}}
{"username": "mata", "team": "SSW", "hero": "魂锁典狱长", "champions": ["2014年全球总决赛冠军"], "honor": "2014年全球总局赛FMVP"}
{"index": {"_id": 4}}
{"username": "deft", "team": "EDG", "hero": "金克斯", "honor": "2015年季中邀请赛MSI冠军"}
//这里也有一个空行
```
执行结果:
```javascript
{
    "took": 105,
    "errors": false,
    "items": [
        {
            "index": {
                "_index": "player",
                "_type": "_doc",
                "_id": "3",
                "_version": 2,
                "result": "updated",   //这里表示id=3的数据被更新了
                "_shards": {
                    "total": 2,
                    "successful": 1,
                    "failed": 0
                },
                "_seq_no": 3,
                "_primary_term": 1,
                "status": 200
            }
        },
        {
            "index": {
                "_index": "player",
                "_type": "_doc",
                "_id": "4",
                "_version": 1,
                "result": "created",  //没有记录的是创建
                "_shards": {
                    "total": 2,
                    "successful": 1,
                    "failed": 0
                },
                "_seq_no": 4,
                "_primary_term": 1,
                "status": 201
            }
        }
    ]
}
```

##### 批量更新
批量更新的关键正如我们预料一样使用```update```作为关键字,但是在数据格式上有一些小不同,因为```update```执行的是局部更新,因此：
```javascript
POST /player/_bulk
{"update": {"_id": 3}} //更新的记录ID相当于where
{"doc": {"hero": "魂锁典狱长·锤石", "year": "2014"}} //这里指定需要更新的信息
{"update": {"_id": 1}}
{"doc": {"nickname": "大魔王", "age": "26"}}
//按照惯例 这里空一行
```
执行结果：
```javascript
{
    "took": 164,
    "errors": false,
    "items": [
        {
            "update": {
                "_index": "player",
                "_type": "_doc",
                "_id": "3",
                "_version": 3,    //注意这里因为 index方式创建数据导致版本更新了一次 因此在update一次,版本号变成了3
                "result": "updated",
                "_shards": {
                    "total": 2,
                    "successful": 1,
                    "failed": 0
                },
                "_seq_no": 5,
                "_primary_term": 1,
                "status": 200
            }
        },
        {
            "update": {
                "_index": "player",
                "_type": "_doc",
                "_id": "1",
                "_version": 2,
                "result": "updated",
                "_shards": {
                    "total": 2,
                    "successful": 1,
                    "failed": 0
                },
                "_seq_no": 6,
                "_primary_term": 1,
                "status": 200
            }
        }
    ]
}
```
> 仔细的同学可能在之前的操作过程可能发现了 我们在批量update操作的时候 id=1的记录跑到了后面,明明我们之前查询的时候还在前面的，这其实就是elasticsearch的伪更新操作,其实它内部每次都是将数据删除了重新建立的关系

##### 批量删除
批量删除用的是```delete```关键字
```javascript
POST /player/_bulk
{"delete": {"_id": 3}}
{"delete": {"_id": 4}}
//这里按照老规矩需要空一个空行出来
```
```javascript
{
    "took": 20,
    "errors": false,
    "items": [
        {
            "delete": {
                "_index": "player",
                "_type": "_doc",
                "_id": "3",
                "_version": 4,
                "result": "deleted", //数据被标识为删除状态
                "_shards": {
                    "total": 2,
                    "successful": 1,
                    "failed": 0
                },
                "_seq_no": 7,
                "_primary_term": 1,
                "status": 200
            }
        },
        {
            "delete": {
                "_index": "player",
                "_type": "_doc",
                "_id": "4",
                "_version": 2,
                "result": "deleted",
                "_shards": {
                    "total": 2,
                    "successful": 1,
                    "failed": 0
                },
                "_seq_no": 8,
                "_primary_term": 1,
                "status": 200
            }
        }
    ]
}
```

##### 条件删除
前面的操作我们都是根据ID进行数据删除，那如果有时候我们不方便根据ID或者说不能单纯根据ID进行删除，需要根据一些条件来进行删除此时就需要使用```_delete_by_query```来进行删除
```javascript
POST /user/_delete_by_query
{
    "query": {
        "match": {
            "nickname": "那个人"
        }
    }
}
```
执行结果：
```javascript
{
    "took": 59,
    "timed_out": false,
    "total": 1,
    "deleted": 1, //被删除的数据量
    "batches": 1,
    "version_conflicts": 0,
    "noops": 0,
    "retries": {
        "bulk": 0,
        "search": 0
    },
    "throttled_millis": 0,
    "requests_per_second": -1.0,
    "throttled_until_millis": 0,
    "failures": []
}
```
至此我们的批量操作就告一个段落了
