# ES

## 倒排索引

- 倒排索引的数据结构

  - 包含该分词的doc list

  - 分词在每个doc中出现的次数 TF term frequency

  - 分词在整个索引中出现的次数 IDF inverse doc frequency

  - 每个doc的长度，长度越长，相关度越低

  - 包含这个关键词所有doc的平均长度

    

## 健康值

````json
{
  "cluster_name" : "elasticsearch", 集群名称
  "status" : "yellow", 健康状态
  "timed_out" : false, 是否超时
  "number_of_nodes" : 1, 节点总数量
  "number_of_data_nodes" : 1, 数据节点数量
  "active_primary_shards" : 4, 活跃的主分片数量
  "active_shards" : 4, 活跃的分片树
  "relocating_shards" : 0, 迁移中的分片数量
  "initializing_shards" : 0, 初始化的分片数量
  "unassigned_shards" : 1, 未分配的分片数量
  "delayed_unassigned_shards" : 0,
  "number_of_pending_tasks" : 0,
  "number_of_in_flight_fetch" : 0,
  "task_max_waiting_in_queue_millis" : 0,
  "active_shards_percent_as_number" : 80.0
}

#设置某个索引的副本数量
PUT product/_settings
{
  "number_of_replicas":1
}


````

## 如何实现高可用

- 当我们进行横向扩容的时候，ES的master节点会自动进行节点分配。 
- 同一个节点ES允许不同的索引的分片同时存在
- ES存放基本规则（**以下这俩个方式对性能提升没有作用，试想：一个节点里有两个相同的数据，有什么作用呢？正常应该是两个相同的数据分布在两个不同的节点上，否则，所有的请求都到这一台机器，这对性能提升无效，如果挂掉了，并且副本只有这一份，那么数据就不会完整**）
  - ES不允许主和副本放在同一个节点中
  - 并且一个节点中不可以存在相同的两个副本。
  - 副本只能读取不能进行写，如果primary shards挂掉，那么，在primary shard重启之前，数据是不能进入到replica shard中，所以这段时间中，是要丢失数据的。
  - 官方推荐，每个分片保持在20-40GB之间

## ES如果宕机，如何恢复数据

- 如果宕机的是master节点：

  1. 开始从剩余**候选节点**进行选举，在选举出新的master之前，集群健康值是**RED**

  2. 假设选举出新master节点是slaver1，此时由于P0挂掉了，那么，slaver1要从R0-1/R0-2中选出一个作为新的P0，假设将R0-1作为新的P0,也叫做Replica容错，或者replica升级。

  3. 新master节点会重启原master节点

  4. 节点重启成功以后，在重启期间原master节点丢失了部分数据，那么此时要从R0-1复制P0没有的数据到P0中。

![ES宕机](.\image\ES宕机.png)

## 节点

- **Master节点**：每个集群的主节点只有一个

  - 对于master节点，es的默认设置是将主节点作为数据节点（node.data = true），也就是承担了搜索任务，但是在实际线上，这是绝对不可以的，因为master还有其他任务要做,要专注于**集群的管理**，当然master也可以作为**协调节点**

- **投票节点**：当master宕机的时候，其他的节点要重新选举出一个主节点，投票节点就是干这个活的，**但仍然可以作为数据节点**，**ES默认投票节点数量和主节点数量一致**，但是如果设置了  **node.voting_only**，那么该节点值进行投票，不做其他事情

  - ``````yaml
    node.voting_only:  true #只有投票节点才能这么设置,意味着这个节点只进行投票，不做其他事情
    data.master: true #如果这两个参数都为true，那么，如果进行选举，该节点仍然不会参选。
    ``````

- **coordinating**:协调节点（负责转发请求，默认每一个节点都是协调节点）

- ````yaml
  #以下两个设置同时设置的时候，那么该节点仅作为协调节点，也就是进行负载均衡
  node.data: false 
  data.master: false
  ````

## 容灾

1. Master选举（如果master节点宕机）

   - 选举过程

     1. 可用的节点会不定期的在集群里做广播，看哪个是主节点。
     2. 如果存在以下情况，就是没有主节点
        1. 没有节点回复自己是主节点	
        2. 发起广播的节点也不是主节点
        3. minimum_master_nodes个节点都联系不上master
           - minimum_master_nodes，这个是需要配置的，指的是**联系不上master的最小节点数量，不是最少主节点数量**，当达到这个数量并且满足以上两个条件，就说明masterGG

     3. 没有主节点的话，就从候选节点中进行投票,满足`node.master: true`配置的节点就是候选节点
     4. 开始投票（默认投票节点的数量和候选节点数量相同，如果投票节点是偶数个，ES会自动去除一个投票节点，使得投票节点的数量成为奇数个）
        - 寻找`clusterStateVersion`比自己高的候选节点，向该节点投票
        - 如果两个节点的`clusterStateVersion`一样，那么就向节点id最小的一个进行投票，id越小，说明该节点更新
        - 如果一个节点的投票数量足够多，超过一半，并且也向自己进行了投票，那么就成为master节点并且告诉其他节点，停止投票。如果是别的节点当选，那么本节点就请求新的master让自己加入集群
     5. 如果投票失败（超时或者没有票数过半的节点），那么重新投票

   - 特殊情况

     - 如果只有两台节点，一台宕机，那么另一台就是master
     - 脑裂：由于某些原因，产生了多台master
       - 如何避免脑裂？
         - 配置`discovery.zen.minumum_master_nodes=N/2+1`,这个配置代表，在选举过程中，票数最小为候选节点数/2+1才能成为master
         - 四/三个节点，可以容忍一台宕机。如果是两台，都必须可用，因为有可能是都是自己投自己，就会无法选出master
         - 集群中最好是奇数个投票节点，如果是偶数个，ES自动去除一个，成为奇数个。因为假设四台机器，每两台在一个机房，那么此时票数最少为4/2+1=3,才能成为master，当机房之间断开连接，这两个小集群都无法选出master，因为票数不可能为3。如果是三个，那么最小票数是2，如果发生机房之间断开连接，两个机房一个有两个投票节点，一个只有一个，那么两个投票节点一定能选出master。 

     

     - 如果有，并且筛选出来的主节点是自己

     - ![Master选举](.\image\Master选举.png)


- `GET /product/_search` 执行这个语句，会查询出product索引的所有数据

  - ````json
    {
      "took" : 2,  #花费的毫秒数
      "timed_out" : false, #是否超时
      "_shards" : {
        "total" : 1,
        "successful" : 1,
        "skipped" : 0,
        "failed" : 0
      },
      "hits" : {
        "total" : {
          "value" : 5,
          "relation" : "eq"
        },
        "max_score" : 1.0,
        "hits" : [
          {
            "_index" : "product",
            "_type" : "_doc",
            "_id" : "2",
            "_score" : 1.0,
            "_source" : {
              "name" : "xiaomi nfc phone",
              "desc" : "zhichi quangongneng nfc,shouji zhong de jianjiji",
              "price" : 4999,
              "tags" : [
                "xingjiabi",
                "fashao",
                "gongjiaoka"
              ]
            }
          },
          {
            "_index" : "product",
            "_type" : "_doc",
            "_id" : "3",
            "_score" : 1.0,
            "_source" : {
              "name" : "nfc phone",
              "desc" : "shouji zhong de hongzhaji",
              "price" : 2999,
              "tags" : [
                "xingjiabi",
                "fashao",
                "menjinka"
              ]
            }
          }
         }
    ````

  - 对于`timed_out`,默认是ES没有进行设置超时限定时间，通过 `GET /_search?timeout=1`可以将超时时间设置为1s，当到达1秒，就会返回1秒内查询出的数据，没查询出来的不显示

- `GET /product/_search?sort=price:desc`,加上sort会进行排序，此时_score为null

  - ````json
     {
            "_index" : "product",
            "_type" : "_doc",
            "_id" : "1",
            "_score" : null, # 这里是空，如果不指定排序，这个表示相关度，默认是按照相关度排序的
            "_source" : {
              "name" : "xiaomi phone",
              "desc" : "shouji zhong de zhandouji",
              "price" : 3999,
              "tags" : [
                "xingjiabi",
                "fashao",
                "buka"
              ]
            },
            "sort" : [
              3999
            ]
          },
    ````

### Query DSL

  - 查询所有

      - ````JSON
        GET /product/_search
        {
          "query": {
            "match_all": {}
          }
        }
        ````

  - 匹配属性

      - ````json
        GET /product/_search
        {
         "query": {
           "match": {
             "name": "xiaomi"
           }
         } 
        }
        ````

  - 排序

      - ````json
        GET /product/_search
        {
          "query": {
            "match_all": {
              
            }
          },
          "sort": [
            {
              "FIELD": {
                "order": "desc"
              }
            }
          ]
        }
        ````

- 多属性匹配

    - ````json
      GET /product/_search
      {
        "query": {
          "multi_match": {
            "query": "erji", #要查询的东西
            "fields": ["name","desc"] #需要匹配的属性，就是这两个属性都要包含 erji
          }
        }
      }
      ````

- 查询某些属性

    - ````json
      GET /product/_search
      {
        "query": {
          "match_all": {}
        },
        "_source": ["name","price"] #查询出的结果只包含这两个字段		
      }
      
      ````

  - 结果：

  - ````json
    {
      "took" : 1,
      "timed_out" : false,
      "_shards" : {
        "total" : 1,
        "successful" : 1,
        "skipped" : 0,
        "failed" : 0
      },
      "hits" : {
        "total" : {
          "value" : 5,
          "relation" : "eq"
        },
        "max_score" : 1.0,
        "hits" : [
          {
            "_index" : "product",
            "_type" : "_doc",
            "_id" : "2",
            "_score" : 1.0,
            "_source" : {
              "price" : 4999,
              "name" : "xiaomi nfc phone"
            }
          },
          {
            "_index" : "product",
            "_type" : "_doc",
            "_id" : "3",
            "_score" : 1.0,
            "_source" : {
              "price" : 2999,
              "name" : "nfc phone"
            }
          },
          {
            "_index" : "product",
            "_type" : "_doc",
            "_id" : "4",
            "_score" : 1.0,
            "_source" : {
              "price" : 999,
              "name" : "xiaomi erji"
            }
          }
          }
        ]
      }
    }
    ````

- 分页

    - ````json
      GET /product/_search
      {
        "query": {
          "match_all": {}
        },
        "from": 1,"size": 2 #从id为1开始，拿出两个，不包含id为1
        , "_source": ["name","price"]
      }
      ````

- 全文检索

  - ````
    #这种方式搜索，搜索的是"nfc phone",但是用match_all,是分别搜索nfc，phone
    GET /product/_search
    {
      "query": {
        "term": {
          "name": "nfc phone"
        }
        
      }
    }
    ````

- 必须、或者

  - 单独的should不计算匹配分数

  - ````json
    GET /product/_search
    {
      "query":{
        "bool": {
          "must": [    #‘must’必须满足
            {"match": {
              "name": "xiaomi"
            }}
          ],
          "should": [
            {
              "range": {
                "price": {"gt": 200}
              }},
              {"range": {
                "price": {"gt": 4999}
               }}
          ],
          "minimum_should_match": 1 #设定‘should’中要满足的个数，如果说查询中，只有should，那么，minimum_should_match默认是1，否则是0
        }
      }
    }
    ````

- **filter**

  - filter查询具有缓存功能，并且不计算分数，也就是匹配程度，计算匹配度是耗费时间的，从而就比query速度更快
  - filter会在query之前执行，先过滤掉一部分数据，在进行查询，这样会加快速度

## 查询语句

-  查询某个index中所有的数据：
  - `GET /product/_search`
- bool
  - must:查询出的结果必须包含该值，并且计算分数
  - filter：子句（查询）必须出现在匹配的文档中。但是不像 `must`查询的分数将被忽略。Filter子句在过滤器中执行，这意味着分数被忽略不被计算分数，并且子句被考虑用于缓存。
    - range:范围查询
      - gte:大于等于
      - lte：小于等于
      - gt：大于
      - lt：小于
  - should：可能会出现在结果中
  - must_not：子句不会出现在查询结果中，并且是在过滤器中执行，所以，不会被计算分数，并且子句被用于缓存

## DeepPaging深查询

- 假设现在有五个分片，每个分片有1w条数据，索引中有一个price属性，我们现在要根据price选出前50条由低到高排序。但是每个分片的前50个不一定甚至不可能是就是全数据的前50，所以要从每个数据中取出前50，然后再排序，这样非常消耗性能，如果不是必要，不要这样做
- 解决办法：
  - 要么避免
  - 要么使用ES推荐使用的**Scoll Search**

### Scoll Search

- ```json
  GET /product/_search?scroll=1m
  {
    "query": {
     "match_all": {}
    },
    "sort": [
      {
        "price": {"order": "desc"}
      }
      
    ],
    "size": 2
  }
  ```

- 在查询的时候加了个**scroll=1m**，此时，结果如下，这比以往的数据多了个**"_scroll_id"**，这个是为了我们再下次查询的时候，只需要带上这个**"_scroll_id"**，ES自动就会给出下一个两条数据，无需重新从所有分片中找出，类似于缓存吧，参数**scroll=1m**中的1m，是指：**在一分钟内，如果没有进行查询，这个scroll_id就会失效**，这个id只能查询一次，再次查询就会失效

  - ````json
    {
      "_scroll_id" : "DXF1ZXJ5QW5kRmV0Y2gBAAAAAAAAOJYWcndfbzFvR0tSaHlQMVYtcDZaRWRNdw==",
      "took" : 1,
      "timed_out" : false,
      "_shards" : {
        "total" : 1,
        "successful" : 1,
        "skipped" : 0,
        "failed" : 0
      },
      "hits" : {
        "total" : {
          "value" : 5,
          "relation" : "eq"
        },
        "max_score" : null,
        "hits" : [
          {
            "_index" : "product",
            "_type" : "_doc",
            "_id" : "2",
            "_score" : null,
            "_source" : {
              "name" : "xiaomi nfc phone",
              "desc" : "zhichi quangongneng nfc,shouji zhong de jianjiji",
              "price" : 4999,
              "tags" : [
                "xingjiabi",
                "fashao",
                "gongjiaoka"
              ]
            },
            "sort" : [
              4999
            ]
          },
          {
            "_index" : "product",
            "_type" : "_doc",
            "_id" : "1",
            "_score" : null,
            "_source" : {
              "name" : "nfc",
              "desc" : "nfc",
              "price" : 3999,
              "tags" : [
                "xingjiabi",
                "fashao",
                "buka"
              ]
            },
            "sort" : [
              3999
            ]
          }
        ]
      }
    }
    
    ````

  - 通过scroll_id查询

    - ````json
      GET /_search/scroll
      {
        "scroll_id": "DXF1ZXJ5QW5kRmV0Y2gBAAAAAAAAOU0WcndfbzFvR0tSaHlQMVYtcDZaRWRNdw=="
      }
      ````

  - 如果第一个id已经使用过了，还想继续使用，那么可以这样

    - ````json
      GET /_search/scroll
      {
        "scroll": "1m", #这相当于重新再加1分钟
        "scroll_id": "DXF1ZXJ5QW5kRmV0Y2gBAAAAAAAAOU0WcndfbzFvR0tSaHlQMVYtcDZaRWRNdw=="
      }
      ````

## mapping

- 动态映射：自动映射类型等

| 示例            | ES中的类型         |
| --------------- | ------------------ |
| “ElasticSearch” | test/keyword       |
| 123456          | long               |
| 123,123         | double             |
| true false      | boolean            |
| 2020-5-12       | date(精确匹配类型) |

- text类型默认不创建正排索引 ，也就是不能进行聚合查询，但通过设置`fileddate=true`可以使text类型的字段进行聚合分析

  - ````json
    put /product3_mapping
    {
    	"properties" :{
    	"tags":{# 这个是字段名名字
    		"type":"text",
    		"fileddate":true
    	}
    	}
    }
    ````
    
  - 
  
- 创建mapping

  - ````json
    PUT /product3
    {
      "mappings": {
        "properties": {
          "date": {
            "type": "text"
          },
          "desc": {
            "type": "text",
            "analyzer": "english"
          },
          "name": {
            "type": "text",
            "index": "false"
          },
          "price": {
            "type": "long"
          },
          "tags": {
            "type": "text",
            "index": "true"
          },
          "parts": {
            "type": "object"
          },
          "partlist": {
            "type": "nested"
          }
        }
      }
    }
    ````

- 创建时候的参数

  - **index**：是否对当前字段创建索引，默认为true，如果是false，那么就是不创建索引，该字段不会通过所以很难被搜索到，但是仍然会zaisource元数据中展示，但是搜索会报错

    - <img src=".\image-20210613174258930.png" alt="image-20210613174258930"  />

  - analyzer:指定分析器

  - search_analyzer:指定搜索时的分词器

  - boost：对当前字段相关度的评分权重，默认为1，5.0已经删掉了

  - **coerce**：是否允许强制转换，默认允许，比如一个字段是interger，如果在创建mapping的没有指定coerce，那么当我们插入一个“123”，也是可以插入进去的，但如果为false，就不会插入进去

  - **doc_value**:正排索引

    - 默认为true，这个参数是为了提升排序和聚合效率，如果确定字段不需要进行排序或者聚合，也不需要通过脚本访问字段值，就可以设置为false，因为正排和倒排都是需要占用磁盘空间的，当时默认这个参数是**不支持text**以及**annotated_text**,因为这两个类型都是比较长的，是非常消耗性能,如果说想要将一个text类型的字段的doc_value打开,那么就可以这样做：

      - ````json
      PUT product
        {
      	"properties":{
        		"tags":{
        			"type":keyword,#也就是将类型换为keyword
        			"doc_values":true
        		}
        	}
        }
        ````
  
      - 
  
  - **eager_global_ordinals：用于聚合的字段上，优化聚合性能**
  
    - Frozen indices（冻结索引）：有些索引使用率很高，会被保存在内存中，有些使用率特别低，宁愿在使用的时候重新创建，在使用完毕后丢弃数据，Frozen indices的数据命中频率小，不适用于高搜索负载，数据不会被保存在内存中，堆空间占用比普通索引少得多，Frozen indices是只读的，请求可能是秒级或者分钟级。**eager_global_ordinals不适用于Frozen indices**
  
  - **fielddata**：查询时**内存**数据结构，在首次用当前字段聚合、排序或者在脚本中使用时，需要字段为fielddata数据结构，并且创建**正排索引**保存到堆中，也就是说，如果某个字段的fieldata为true，那么在进行聚合或者排序的时候，会创建正排索引在JVM内存中
  
  - fileds：给field创建多字段，用于全文检索或者聚合分析
  
    - ````json
         "name" : {
                "type" : "text",
                "fields" : {
                  "keyword" : {
                    "type" : "keyword",
                    "ignore_above" : 256
                  }
                }
            },
      ````
  
    - 

## ES数据类型

- 数字类型：(虽然ES在添加数据的时候会自动匹配类型，但是如果是自己创建mapping，要选择小的数据类型)
  - long,integer,short,byte,double,float,half_float,scaled_float
- 字符串
  - keyword
    -  适用于过滤、排序、聚合。keyword类型的字段只能通过**精确搜索**，id应该用keyword(如果不进行范围查询)，而不是text也不是interger
  - test
    - 当一个字段是要被全文搜索的时候，比如描述、内容等，这些字段应该使用text类型，设置text类型后，字段内容会被分析，字段会被分成一个一个的词项，text类型的字段不用于排序，很少用于聚合
- date（**exact value**，精确查找）
- 还有很多类型

## ES底层原理
- 首先，ES官方建议，ES的查询是通过系统的缓存来提升性能的，不建议用jvm内存来进行缓存，那样会导致一定的gc以及oom问题，在服务器中，一般来说，ES服务器的内存，1/16~1/4的内存分配给jvm，剩下的都给ES，这样可以提升正排和倒排的效率
- fieldata与doc value
  - 如果字段没有设置doc value为false，那么如果想要进行聚合或者排序，那么需要设置fieldata
  - fieldata默认不启用，因为text的字段都比较长，那么长的字段在内存中，很容易把内从撑爆，
  - ES采用了熔断机制避免fieldata一次性超过内存大小从而溢出，如果触发熔断，查询会被终止并返回异常
  - 如果内存充足的时候，doc vlue会被操作系统加载到内存中，如果内存不充足，会在磁盘中

## 批量CRUD

- ````json
  GET /product/_mget
  {
    "docs":[
      {"_index":"product",
        "_id":1
      },{
        "_index":"product",
        "_id":4
      }
      
      
      ]
  }
  
  GET /product/_mget
  {
    "docs":[
      {
        "_id":1
      },{
        
        "_id":4
      }
      
      
      ]
  }
  
  GET /product/_mget
  {
    "ids":[1,2]
  }
  
  GET /product/_mget
  {
    "docs":[
      {
        "_id":1,
       "_source":[
         "name",
         "price"
       ]
      },{
        "_index":"product",
        "_id":4
      }
      
      
      ]
  }
  
  GET /product/_mget
  {
    "docs":[
      {
        "_id":1,
       "_source":{
         "include":["name"],
         "exclude":["tags"]
       }
      },{
        "_index":"product",
        "_id":4
      }
      ]
  }
  
  
  ````

- ````json
  # 批量创建
  POST /_bulk
  {"create":{"_index":"product2","_id":"6"}}
  {"name":"_uld crate6"}
  {"create":{"_index":"product2","_id":"7"}}
  {"name":"_uld crate7"}
  
  #批量删除
  POST /_bulk
  {"delete":{"_index":"product2","_id":"6"}}
  {"delete":{"_index":"product2","_id":"7"}}
  
  #批量更改
  POST /_bulk
  {"update":{"_index":"product2","_id":"2","retry_on_conflict":"3"}}
  {"doc":{"name":"update_name_by_bulk"}}
  {"update":{"_index":"product2" ,"_id":"5"}}
  {"doc":{"name":"update_name_by_bulk"}}
  
  #通过index进行批量增加或者修改
  POST /_bulk
  {"index":{"_index":"product2","_id":"1"}}
  {"doc":{"name":"update_by_index"}}
  {"index":{"_index":"product2","_id":"10"}}
  {"doc":{"name":"craete_by_index"}}
  
  ````

- **为什么要用这种特殊的格式?**

  - 如果用之前的那种格式，ES会序列化为对象，对于批量操作，会非常占用内存，用这种特殊的格式，会减少内存消耗

- ````json
  POST /_bulk?filter_path=items.*.error # 通过这种方式，就只会返回错误的操作，成功的不再返回
  {"create":{"_index":"product2","_id":"9"}}
  {"name":"_uld crate6"}
  {"create":{"_index":"product2","_id":"8"}}
  {"name":"_uld crate7"}
  ````


## ES写入原理

- 

## 前缀、正则、通配符、模糊

- ````json
  #前缀
  GET /my_index/_search
  {
    "query": {
      "prefix": {
        "text": "int"
      }
    }
  }
  #正则
  GET product/_search
  {
    "query": {
      "regexp": {
        "desc": {
          "value": ".*2020-05-20.*",
          "flags": "ALL"  #默认启用所有可选的运算符
        }
      }
    }
  }
  
  #模糊查询
  GET /product/_search 
  {
    "query": {
      "fuzzy": {
        "desc": {
          "value": "quangemneng",
          "fuzziness": 5
        }
      }
    }
  }
  #通配符
  GET my_index/_search
  {
    "query": {
      "wildcard": {
        "text": {
          "value": "eng?ish"
        }
      }
    }
  }
  #模糊查询，可以用这个
  GET my_index/_search
  {
    "query": {
      "match": {
        "text": {
          "query": "30",
          "fuzziness": "AUTO",
          "operator": "and"
          
        }
      }
      
    }
  }
  
  ````

  

- 在创建mapping的时候，可以添加两个参数

  - `min_chars`,`max_chars`,默认是2，5
  - 如果参数的值分别是2，4。假设字段的值是  `String name int `

- 前缀、正则、通配符都是搜索分词后的结果，效率不高

- `match_phrase_prefix`:

  - ![image-20210622223738515](.\image\image-20210622223738515.png)

  - ![image-20210622224003569](.\image\image-20210622224003569.png)




## 评分度算法

- ````json
  #查询一
  GET  /my_index3/_search
  {
    "query": {
      "match": {
        "text": "english good"
      }
    }
  }
  #查询二这个是查询一的标准写法，
  GET  /my_index3/_search
  {
    "query": {
      "match": {
        "text": {
          "query": "english good",
          "operator": "or"，#默认就是or,会将关于english或者good相关的都搜索出来
        }
      }
    }
  }
  
  #查询三
  GET my_index3/_search
  {
    "query": {
      "bool": {
        "should": [
          {"term": {"text":"english"}},
          {"term": {"text": "good"}}
        ]
        
      }
    }
  }
  
  # 以上三个查询相同,分数也是相同的，ES内部会将第一个查询转换为第二个
  ````

- ````json
  
  GET  /my_index3/_search
  {
    "query": {
      "match": {
        "text": {
          "query": "english is  very good",
          "operator": "or",
          "minimum_should_match": "75%"
        }
      }
    }
  }
  
  
  GET my_index3/_search
  {
    "query": {
      "bool": {
        "should": [
          {"term": {"text":"english"}},
          {"term": {"text": "good"}},
          {"term": {"text":"is"}},
          {"term": {"text": "very"}}
        ],
        "minimum_should_match": "75%"
        
      }
    }
  }
  #以上两个查询相同
  ````

- ````json
  GET my_index3/_search
  {
    "query": {
      "bool": {
        "should": [
          {"match":{"text": "chinese"}},
           {"match":  {"text":{"query":"english"}}},
           {"match": {"text": "is"}},
           {"match":   {"text": "very"}}
        ],
        "minimum_should_match": "75%"
      }
    }
  }
  #结果：
  {
    "took" : 1,
    "timed_out" : false,
    "_shards" : {
      "total" : 1,
      "successful" : 1,
      "skipped" : 0,
      "failed" : 0
    },
    "hits" : {
      "total" : {
        "value" : 2,
        "relation" : "eq"
      },
      "max_score" : 3.799707,
      "hits" : [
        {
          "_index" : "my_index3",
          "_type" : "_doc",
          "_id" : "8",
          "_score" : 3.799707,
          "_source" : {
            "text" : "my chinese is very good"
          }
        },
        {
          "_index" : "my_index3",
          "_type" : "_doc",
          "_id" : "4",
          "_score" : 3.3144937,
          "_source" : {
            "text" : "my english is very good"
          }
        }
      ]
    }
  }
  
  
  
  
  
  
  GET my_index3/_search
  {
    "query": {
      "bool": {
        "should": [
          {"match":{"text": "chinese"}},
           {"match":  {"text":{"query":"english","boost":2}}},
           {"match": {"text": "is"}},
           {"match":   {"text": "very"}}
        ],
        "minimum_should_match": "75%"
      }
    }
  }
  #结果:明显能看到english的结果的分数较高了，这是因为设置了boost位2，默认是1
  {
    "took" : 1,
    "timed_out" : false,
    "_shards" : {
      "total" : 1,
      "successful" : 1,
      "skipped" : 0,
      "failed" : 0
    },
    "hits" : {
      "total" : {
        "value" : 2,
        "relation" : "eq"
      },
      "max_score" : 4.419325,
      "hits" : [
        {
          "_index" : "my_index3",
          "_type" : "_doc",
          "_id" : "4",
          "_score" : 4.419325,
          "_source" : {
            "text" : "my english is very good"
          }
        },
        {
          "_index" : "my_index3",
          "_type" : "_doc",
          "_id" : "8",
          "_score" : 3.799707,
          "_source" : {
            "text" : "my chinese is very good"
          }
        }
      ]
    }
  }
  
  ````



### TF—IDF算法：

- TF(词频：term frequency)：每个词在doc中出现的频率,TF越高相关度越高

- IDF(反文档词频 inversed document frequency):关键词在整个索引中出现的次数，IDF越高，相关度越低

- norm：字段长度越长，相关度越低

  - ````json
    
    GET my_index3/_search
    {
      "query": {
        "bool": {
          "must": [
                  {"match": {"text": "is"}}
          
          ]
        }
      }
    }
    #结果
    {
      "took" : 1,
      "timed_out" : false,
      "_shards" : {
        "total" : 1,
        "successful" : 1,
        "skipped" : 0,
        "failed" : 0
      },
      "hits" : {
        "total" : {
          "value" : 3,
          "relation" : "eq"
        },
        "max_score" : 0.9632257,
        "hits" : [
          {
            "_index" : "my_index3",
            "_type" : "_doc",
            "_id" : "4",
            "_score" : 0.9632257,
            "_source" : {
              "text" : "my english is very good"
            }
          },
          {
            "_index" : "my_index3",
            "_type" : "_doc",
            "_id" : "8",
            "_score" : 0.9632257,
            "_source" : {
              "text" : "my chinese is very good"
            }
          },
          {
            "_index" : "my_index3",
            "_type" : "_doc",
            "_id" : "9",
            "_score" : 0.6522291,
            "_source" : {
              "text" : "my chinese is very good dssd  dsd s 22 dw  deqqqeq "
            }
          }
        ]
      }
    }
    
    ````


## 评分原理以及排序规则优化

### 评分原理

- 有如下匹配结果：

  1. {"match": {"name": "吃鸡手机"}},
     doc1:	吃鸡1次，手机1次，计2分	
     doc2:	吃鸡0次，手机1次，计1分
     doc3:	吃鸡0次，手机1次，计1分
  2. {"match": {"desc": "吃鸡手机"}}
     doc1:	吃鸡0次，手机0次，计0分	
     doc2:	吃鸡0次，手机1次，计1分
     doc3:	吃鸡0次，手机1次，计1分
  3. 计算分数：总分：（query1+query2）*matched query / total query
     1. doc1:	query1+query2：2		matched：1	total query：2		result：2*1/2=1
     2. doc2:	query1+query2：2		matched：2	total query：2		result：2*2/2=2
     3. doc3:	query1+query2：2		matched：2	total query：2		result：2*2/2=2

### dix_max

- 现有以下数据：

  - ````json
    PUT /product/_doc/1
    {
      "name": "吃鸡手机，游戏神器，超级",
      "desc": "基于TX深度定制，流畅游戏不发热，物理外挂，快充",
      "price": 3999,
      "createtime": "2020-05-20",
      "collected_num": 99,
      "tags": [
        "性价比",
        "发烧",
        "不卡"
      ]
    }
    
    PUT /product/_doc/2
    {
      "name": "小米NFC手机",
      "desc": "支持全功能NFC,专业吃鸡，快充",
      "price": 4999,
      "createtime": "2020-05-20",
      "collected_num": 299,
      "tags": [
        "性价比",
        "发烧",
        "公交卡"
      ]
    }
    
    PUT /product/_doc/3
    {
      "name": "NFC手机，超级",
      "desc": "手机中的轰炸机",
      "price": 2999,
      "createtime": "2020-05-20",
      "collected_num": 1299,
      "tags": [
        "性价比",
        "发烧",
        "门禁卡"
      ]
    }
    
    PUT /product/_doc/4
    {
      "name": "小米耳机",
      "desc": "耳机中的黄焖鸡",
      "price": 999,
      "createtime": "2020-05-20",
      "collected_num": 9,
      "tags": [
        "低调",
        "防水",
        "音质好"
      ]
    }
    
    PUT /product/_doc/6
    {
      "name": "红米耳机，超",
      "desc": "耳机中的肯德基，快充",
      "price": 399,
      "createtime": "2020-05-20",
      "collected_num": 0,
      "tags": [
        "牛逼",
        "续航长",
        "质量好"
      ]
    }
    #现有一查询
    GET product/_search
    {
      "query": {
        "bool": {
          "should": [
            {"match": {"desc": "吃鸡手机"}},
            {"match": {"name": "吃鸡手机"}} 
          ]
        }
      }
    }
    # 结果为
        "hits" : [
          {
            "_index" : "product",
            "_type" : "_doc",
            "_id" : "2",
            "_score" : 3.0876093,
            "_source" : {
              "name" : "小米NFC手机",
              "desc" : "支持全功能NFC,专业吃鸡，快充",
              "price" : 4999,
              "createtime" : "2020-05-20",
              "collected_num" : 299,
              "tags" : [
                "性价比",
                "发烧",
                "公交卡"
              ]
            }
          },
          {
            "_index" : "product",
            "_type" : "_doc",
            "_id" : "1",
            "_score" : 2.9206116,
            "_source" : {
              "name" : "吃鸡手机，游戏神器，超级",
              "desc" : "基于TX深度定制，流畅游戏不发热，物理外挂，快充",
              "price" : 3999,
              "createtime" : "2020-05-20",
              "collected_num" : 99,
              "tags" : [
                "性价比",
                "发烧",
                "不卡"
              ]
            }
          },
          {
            "_index" : "product",
            "_type" : "_doc",
            "_id" : "3",
            "_score" : 2.4004009,
            "_source" : {
              "name" : "NFC手机，超级",
              "desc" : "手机中的轰炸机",
              "price" : 2999,
              "createtime" : "2020-05-20",
              "collected_num" : 1299,
              "tags" : [
                "性价比",
                "发烧",
                "门禁卡"
              ]
            }
          },
          {
            "_index" : "product",
            "_type" : "_doc",
            "_id" : "4",
            "_score" : 1.1123568,
            "_source" : {
              "name" : "小米耳机",
              "desc" : "耳机中的黄焖鸡",
              "price" : 999,
              "createtime" : "2020-05-20",
              "collected_num" : 9,
              "tags" : [
                "低调",
                "防水",
                "音质好"
              ]
            }
          }
        ]
    
    #最佳结果明明是id为1的，但查出来的结果却是id为2，原因是根据上文所提到的评分原则，假定每匹配到一个词项的得分是1，
    #那么id为2的得分是：name中‘手机’匹配一次，desc中'吃'和'鸡'各匹配一次，一共两个查询，匹配了两个字段，那么得分为：（1+1+1）*2/2=3,
    #id为1的得分是：name中手机匹配一次，'吃'和'鸡'各匹配一次，desc没有匹配到，那么总得分是：（1+1+1）*1/2=1.5
    #那么id为2的得分最高，为了解决这个问题，可以使用dix_max
    GET product/_search
    {
      "query": {
        "dis_max": {
          "queries": [
            {"match": {"name": "吃鸡手机"}},
            {"match": {"desc": "吃鸡手机"}}      
            ]
        }
      }
    }
    #结果
     "hits" : [
          {
            "_index" : "product",
            "_type" : "_doc",
            "_id" : "1",
            "_score" : 2.9206116,
            "_source" : {
              "name" : "吃鸡手机，游戏神器，超级",
              "desc" : "基于TX深度定制，流畅游戏不发热，物理外挂，快充",
              "price" : 3999,
              "createtime" : "2020-05-20",
              "collected_num" : 99,
              "tags" : [
                "性价比",
                "发烧",
                "不卡"
              ]
            }
          },
          {
            "_index" : "product",
            "_type" : "_doc",
            "_id" : "2",
            "_score" : 2.7195241,
            "_source" : {
              "name" : "小米NFC手机",
              "desc" : "支持全功能NFC,专业吃鸡，快充",
              "price" : 4999,
              "createtime" : "2020-05-20",
              "collected_num" : 299,
              "tags" : [
                "性价比",
                "发烧",
                "公交卡"
              ]
            }
          },
         #其他的结果不一一列举
    #这样，id为1的得分就会是最高的
    ````

  - 注意**dix_max是选取，匹配的字段中的某个字段score最高的为准**

### tie_breaker

  - 但问题又来了

    - ````json
      GET product/_search
      {
        "query": {
          "dis_max": {
            "queries": [
              {"match": {"name": "超级快充"}},
              {"match": {"desc": "超级快充"}}
              ]
          }
          }
      }
      #结果为：
       "hits" : [
            {
              "_index" : "product",
              "_type" : "_doc",
              "_id" : "6",
              "_score" : 1.497693,
              "_source" : {
                "name" : "红米耳机，超",
                "desc" : "耳机中的肯德基，快充",
                "price" : 399,
                "createtime" : "2020-05-20",
                "collected_num" : 0,
                "tags" : [
                  "牛逼",
                  "续航长",
                  "质量好"
                ]
              }
            },
            {
              "_index" : "product",
              "_type" : "_doc",
              "_id" : "2",
              "_score" : 1.2683676,
              "_source" : {
                "name" : "小米NFC手机",
                "desc" : "支持全功能NFC,专业吃鸡，快充",
                "price" : 4999,
                "createtime" : "2020-05-20",
                "collected_num" : 299,
                "tags" : [
                  "性价比",
                  "发烧",
                  "公交卡"
                ]
              }
            },
            {
              "_index" : "product",
              "_type" : "_doc",
              "_id" : "3",
              "_score" : 1.093527,
              "_source" : {
                "name" : "NFC手机，超级",
                "desc" : "手机中的轰炸机",
                "price" : 2999,
                "createtime" : "2020-05-20",
                "collected_num" : 1299,
                "tags" : [
                  "性价比",
                  "发烧",
                  "门禁卡"
                ]
              }
            },
            {
              "_index" : "product",
              "_type" : "_doc",
              "_id" : "1",
              "_score" : 1.0533224,
              "_source" : {
                "name" : "吃鸡手机，游戏神器，超级",
                "desc" : "基于TX深度定制，流畅游戏不发热，物理外挂，快充",
                "price" : 3999,
                "createtime" : "2020-05-20",
                "collected_num" : 99,
                "tags" : [
                  "性价比",
                  "发烧",
                  "不卡"
                ]
              }
            }
          ]
      #很明显，结果应该是id为1的，但是结果却不是，至于为什么，因为有了dix_max，从而该查询将desc作为了最佳查询字段，可以看一下以下查询以及结果：
      GET  product/_search
      {
        "query": {
          "bool": {
            "should": [
              {"match": {"desc": "超级快充"}}]
          }
        }
      }
      #结果
       "hits" : [
            {
              "_index" : "product",
              "_type" : "_doc",
              "_id" : "6",
              "_score" : 1.497693,
              "_source" : {
                "name" : "红米耳机，超",
                "desc" : "耳机中的肯德基，快充",
                "price" : 399,
                "createtime" : "2020-05-20",
                "collected_num" : 0,
                "tags" : [
                  "牛逼",
                  "续航长",
                  "质量好"
                ]
              }
            },
            {
              "_index" : "product",
              "_type" : "_doc",
              "_id" : "2",
              "_score" : 1.2683676,
              "_source" : {
                "name" : "小米NFC手机",
                "desc" : "支持全功能NFC,专业吃鸡，快充",
                "price" : 4999,
                "createtime" : "2020-05-20",
                "collected_num" : 299,
                "tags" : [
                  "性价比",
                  "发烧",
                  "公交卡"
                ]
              }
            },
            {
              "_index" : "product",
              "_type" : "_doc",
              "_id" : "1",
              "_score" : 1.0533224,
              "_source" : {
                "name" : "吃鸡手机，游戏神器，超级",
                "desc" : "基于TX深度定制，流畅游戏不发热，物理外挂，快充",
                "price" : 3999,
                "createtime" : "2020-05-20",
                "collected_num" : 99,
                "tags" : [
                  "性价比",
                  "发烧",
                  "不卡"
                ]
              }
            }
          ]
      #但不知道为什么，这个查询出来的结果是3个，而上面的查询结果是4个
      #但为了解决目前name没有参与进来查询，可以通过"tie_breaker"这个字段来让name参与进来，代码如下
      GET product/_search
      {
        "query": {
          "dis_max": {
            "queries": [
              {"match": {"name": "超级快充"}},
              {"match": {"desc": "超级快充"}}
              ]
              , "tie_breaker": 0.7
          }
          }
      }
      #结果：
      "hits" : [
            {
              "_index" : "product",
              "_type" : "_doc",
              "_id" : "1",
              "_score" : 1.6110761,
              "_source" : {
                "name" : "吃鸡手机，游戏神器，超级",
                "desc" : "基于TX深度定制，流畅游戏不发热，物理外挂，快充",
                "price" : 3999,
                "createtime" : "2020-05-20",
                "collected_num" : 99,
                "tags" : [
                  "性价比",
                  "发烧",
                  "不卡"
                ]
              }
            },
            {
              "_index" : "product",
              "_type" : "_doc",
              "_id" : "6",
              "_score" : 1.497693,
              "_source" : {
                "name" : "红米耳机，超",
                "desc" : "耳机中的肯德基，快充",
                "price" : 399,
                "createtime" : "2020-05-20",
                "collected_num" : 0,
                "tags" : [
                  "牛逼",
                  "续航长",
                  "质量好"
                ]
              }
            },
            {
              "_index" : "product",
              "_type" : "_doc",
              "_id" : "2",
              "_score" : 1.2683676,
              "_source" : {
                "name" : "小米NFC手机",
                "desc" : "支持全功能NFC,专业吃鸡，快充",
                "price" : 4999,
                "createtime" : "2020-05-20",
                "collected_num" : 299,
                "tags" : [
                  "性价比",
                  "发烧",
                  "公交卡"
                ]
              }
            },
            {
              "_index" : "product",
              "_type" : "_doc",
              "_id" : "3",
              "_score" : 1.093527,
              "_source" : {
                "name" : "NFC手机，超级",
                "desc" : "手机中的轰炸机",
                "price" : 2999,
                "createtime" : "2020-05-20",
                "collected_num" : 1299,
                "tags" : [
                  "性价比",
                  "发烧",
                  "门禁卡"
                ]
              }
            }
          ]
      #"tie_breaker"最好是0.1-0.7之间
      
      ````


### minimum_should_match

- 观察两个查询区别

  - 查询一

    - ````json
      GET product/_search
      {
        "query": {
           "bool": {
              "should": [
                {"match": {
                  "name": {
                    "query": "吃鸡手机"
                  }
                }}
              ]
          }
        }
      }
      #结果
       "hits" : [
            {
              "_index" : "product",
              "_type" : "_doc",
              "_id" : "1",
              "_score" : 2.9206116,
              "_source" : {
                "name" : "吃鸡手机，游戏神器，超级",
                "desc" : "基于TX深度定制，流畅游戏不发热，物理外挂，快充",
                "price" : 3999,
                "createtime" : "2020-05-20",
                "collected_num" : 99,
                "tags" : [
                  "性价比",
                  "发烧",
                  "不卡"
                ]
              }
            },
            {
              "_index" : "product",
              "_type" : "_doc",
              "_id" : "2",
              "_score" : 0.7361701,
              "_source" : {
                "name" : "小米NFC手机",
                "desc" : "支持全功能NFC,专业吃鸡，快充",
                "price" : 4999,
                "createtime" : "2020-05-20",
                "collected_num" : 299,
                "tags" : [
                  "性价比",
                  "发烧",
                  "公交卡"
                ]
              }
            },
            {
              "_index" : "product",
              "_type" : "_doc",
              "_id" : "3",
              "_score" : 0.7361701,
              "_source" : {
                "name" : "NFC手机，超级",
                "desc" : "手机中的轰炸机",
                "price" : 2999,
                "createtime" : "2020-05-20",
                "collected_num" : 1299,
                "tags" : [
                  "性价比",
                  "发烧",
                  "门禁卡"
                ]
              }
            }
          ]
         
      ````

    - 查询二

      - ````json
        GET product/_search
        {
          "query": {
             "bool": {
                "should": [
                  {"match": {
                    "name": {
                      "query": "吃鸡手机",
                      "minimum_should_match": "100%"
                    }
                  }}
                ]
            }
          }
          
        "hits" : [
              {
                "_index" : "product",
                "_type" : "_doc",
                "_id" : "1",
                "_score" : 2.9206116,
                "_source" : {
                  "name" : "吃鸡手机，游戏神器，超级",
                  "desc" : "基于TX深度定制，流畅游戏不发热，物理外挂，快充",
                  "price" : 3999,
                  "createtime" : "2020-05-20",
                  "collected_num" : 99,
                  "tags" : [
                    "性价比",
                    "发烧",
                    "不卡"
                  ]
                }
              }
            ]
        ````

    - 第二个查询只有一个数据，这是因为`minimum_should_match`是用来控制匹配精准度的，也就是说，第二个查询会严格查询只包含`吃鸡手机`这四个字的，其他的一概不展示。

### 查询替换

- 以下两个查询相同

  - ````json
    GET product/_search
    {
      "query": {
         "bool": {
            "should": [
              {"match": {
                "name": {
                  "query": "吃鸡手机",
                  "minimum_should_match": "100%"
                }
              }}
            ]
        }
      }
    }
    
    GET product/_search
    {
      "query": {
        "bool": {
          "should": [
            {"match": { "name": "吃鸡"}},
            {"match": {"name": "手机"}}
          ],
          "minimum_should_match": "100%"
          }
      }
    }
    ````

### 简化查询

- ````json
  # "type"默认是best_fields
  #在fields上的字段名字^num，就是像上面的boost
  #但是不知道怎么分别设置minimum_should_match
  GET product/_search
  {
    "query": {
      "multi_match": {
        "query": "吃鸡手机",
        "type": "best_fields", 
        "fields": ["name^2","desc^1"],
        "tie_breaker": 0.7,
        "minimum_should_match": "50%"
      }
    }
  }
  ````

### best_fields、most_fields、cross_fields

- best_fields（默认）：对于一个查询，单个field(字段)匹配更多的term(词项),则优先排序

- most_fields：对于一个查询，对于同一个doc，匹配某个词项的fields的越多，则优先排序

- cross_fields:

  - ````json
    #有以下数据
    POST /person/_bulk
    { "index": { "_id": "1"} }
    { "name" : {"first_name" : "史密斯", "last_name" : "威廉姆斯"} }
    { "index": { "_id": "2"} }	
    { "name" : {"first_name" : "史蒂夫", "last_name" : "乔布斯"} }
    { "index": { "_id": "3"} }
    { "name" : { "first_name" : "彼得","last_name" : "史密斯"} }
    { "index": { "_id": "4"} }
    { "name" : { "first_name" : "阿诺德","last_name" : "施瓦辛格"} }
    { "index": { "_id": "5"} }
    { "name" : {"first_name" : "詹姆斯", "last_name" : "卡梅隆"} }
    { "index": { "_id": "6"} }
    { "name" : {"first_name" : "彼得", "last_name" : "詹姆斯"} }
    { "index": { "_id": "7"} }
    { "name" : {"first_name" : "彼得", "last_name" : "史密斯"} }
    { "index": { "_id": "8"} }
    { "name" : {"first_name" : "彼得", "last_name" : "史密斯"} }
    { "index": { "_id": "9"} }
    { "name" : {"first_name" : "彼得", "last_name" : "史密斯"} }
    { "index": { "_id": "10"} }
    { "name" : {"first_name" : "勒布朗", "last_name" : "史密斯"} }
    #cross_fields：彼得必须在姓中出现或者名中出现，且，史密斯必须在姓中出现或者名中出现
    GET person/_search
    {
      "query": {
        "multi_match": {
          "query": "彼得·史密斯",
          "type": "cross_fields", 
          "fields": ["name.first_name","name.last_name"],
          "operator": "and"
        }
        
      }
    }
    
    ````

### field_value_factor根据有个数字字段进行计算分数

- field:要计算的字段

- modifier:以何种方式来进行计算，接受以下参数

  1. none：不处理

  2. log：计算对数

  3. log1p：先将字段值 +1，再计算对数

  4. log2p：先将字段值 +2，再计算对数

  5. ln：计算自然对数

  6. ln1p：先将字段值 +1，再计算自然对数

  7. ln2p：先将字段值 +2，再计算自然对数

  8. square：计算平方

  9. sqrt：计算平方根
  10. reciprocal：计算倒数

- factor：当前分数计算，对整个结果产生的权重比

### boost_mode

- 指定计算后的分数与原始的score如何合并

  - 1. multiply：查询分数和函数分数相乘
    2. sum：查询分数和函数分数相加
    3. avg：取平均值
    4. replace：替换原始分数
    5. min:取查询分数和函数分数的最小值
    6. max：取查询分数和函数分数的最大值

  ​      

### max_boost

- 分数上限

- 例子

  - ````json
    GET product/_search
    {
      "query": {
        "function_score": {
          "query": {
            "match_all": {}
          },
          "field_value_factor": {
            "field": "collected_num",
            "modifier": "log1p",
            "factor": 0.9
          },
          "boost_mode": "multiply",
          "random_score": {
            "": 314159265359
          }, 
          "max_boost": 3
        }
      }
    }
    ````

## 聚合分析深入





















