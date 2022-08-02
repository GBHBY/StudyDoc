# ElasticSearch

- 在学习过程中，不要修改集群相关的配置，修改过后，ES会进行健康检查，如果配置的有问题，ES会无启动。而不修改集群的相关配置，ES不会进行健康检查，即使配置有问题，也只是出现警告。比如配置文件中以`cluster`开头等配置。
- ES的日志都是输出到logs中的目录中，健康检查会检查，GC、线程、内存，权限、网络配置等配置，任何一项配置的不对都会引起ES无法启动，

## 配置

- [配置文件详解](ElasticSearchConfig.md)

## SQL

- SearchTimeout:
  - 默认没有TimeOut，如果设置了time_out，那么就会执行time_out机制.
  - 如果设置了time_out机制，假设设置的时间为1秒，那么如果一个查询消耗的时间是10s，那么查询结果只会返回一秒的数据，其余的数据不会返回。
  - 通过 `GET /_search?timeout=1s`来设置超时时间

- 查看集群健康值
  - `GET /_cluster/health`
  
  - ```json
    { 
      "cluster_name" : "elasticsearch",
      "status" : "yellow",
      "timed_out" : false,#是否超时
      "number_of_nodes" : 1,  #节点数 
      "number_of_data_nodes" : 1, #数据节点数
      "active_primary_shards" : 32, #活跃的主分片
      "active_shards" : 32,#活跃的分片数
      "relocating_shards" : 0,# 迁移中的分片数
      "initializing_shards" : 0,#初始化的分片数量
      "unassigned_shards" : 22,#未分配的分片数量
      "delayed_unassigned_shards" : 0,
      "number_of_pending_tasks" : 0,
      "number_of_in_flight_fetch" : 0,
      "task_max_waiting_in_queue_millis" : 0,
      "active_shards_percent_as_number" : 59.25925925925925
    }
    ```

- 插入数据：

  -  ````json
     
     PUT /product/_doc/1
     {
         "name" : "xiaomi phone",
         "desc" :  "shouji zhong de zhandouji",
         "price" :  3999,
         "tags": [ "xingjiabi", "fashao", "buka" ]
     }
     PUT /product/_doc/2
     {
         "name" : "xiaomi nfc phone",
         "desc" :  "zhichi quangongneng nfc,shouji zhong de jianjiji",
         "price" :  4999,
         "tags": [ "xingjiabi", "fashao", "gongjiaoka" ]
     }
     
     
     
     ````

- 修改id为1的某一个字段值

  - `````json
    POST /product/_doc/1/_update
    {
     "doc":{
       "price":5000
     }
     ## "doc"是必须要加的，"price"是字段
    `````

- 查询name字段包含nfc的数据

  - ````sql
    GET /product/_search?q=name:nfc
    ````

- 在上个查询加入分页以及排序，按照升序排序，但注意，**加入sort之后的score是null，但是不加排序，score是有值的，因为ES默认是按照score从高到低排序的，现在加入排序条件，就不会再有score了**不同的条件之间使用`&`来连接

  - ```
    GET /product/_search?q=name:nfc&from=0&size=2&sort=price:asc
    ```

  -  ````josn
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
         "max_score" : null,
         "hits" : [
           {
             "_index" : "product",
             "_type" : "_doc",
             "_id" : "3",
             "_score" : null,
             "_source" : {
               "name" : "nfc phone",
               "desc" : "shouji zhong de hongzhaji",
               "price" : 2999,
               "tags" : [
                 "xingjiabi",
                 "fashao",
                 "menjinka"
               ]
             },
             "sort" : [
               2999
             ]
           },
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
           }
         ]
       }
     }
     ````

    

### DSL

- 查询索引

  - ````json
    查询全部
    GET /_cat/indices?v
    
    模糊搜索
    GET /_cat/indices/indexName*?v
    ````

- 查询所有

  - ````json
    GET /product/_search
    {
      "query": {
        "match_all": {}
      }
      
    }
    ````

- 查询name字段包含xiaomi的数据
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

- 查询name字段包含xiaomi的数据,并且按照price降序排序

  - ````
    
    GET /product/_search
    {
      "query": {
        "match": {
          "name": "xiaomi"
        }
      },
      "sort": [
        {
          "price":"desc"
        }
      ]
      
    }
    
    ````

- 查询name和desc字段中包含nfc的数据

  - ```json
    GET /product/_search
    {
      "query": {
        "multi_match": {
          "query": "nfc",
          "fields": ["name","desc"]
        }
        
      }  
      
      
    }
    
    ```

- 查询name和desc字段中包含nfc的数据，但只返回名字和价格字段

  - `````json
    GET /product/_search
    {
      "query": {
        "multi_match": {
          "query": "nfc",
          "fields": ["name","desc"]
        }
        
      },
      "_source": ["name","price"]
      
    }
    
    `````

- 查询name和desc字段中包含nfc的数据，但只返回名字和价格字段,并加上分页

  - ````json
    
     GET /product/_search
    {
      "query": {
        "multi_match": {
          "query": "nfc",
          "fields": ["name","desc"]
        }
        
      },
      "_source": ["name","price"],
      
      "from": 0,
      "size": 1
      
    }
    
    ````

- 多值匹配,**terms**，会返回name字段中，有nfc或者phone的，可以理解为in

  - ````json
    GET /product/_search
    {
      "query": {
        "terms":{
          "name":["nfc","phone"]
        }
      } 
    }
    ````

- 短语搜索：只会返回包含 ’nfc phone的‘的数据

  - ````json
    GET /product/_search
    {
      "query": {
        "match_phrase": {
          "name": "nfc phone"
        }
      } 
    }
    ````

#### 多条件查询 bool

- must

  - ```
    
    GET /product/_search
    {
      "query": {
        "bool": {
          "must": [
            {"match":{"name":"nfc phone"}}
          ]
        }
      }
    }
      
    ```

  - 

- must_not(不计算相关度分数)

- filer(不计算相关度分数,但是如果filter和must或者should联合使用，是会计算分数的)，由于不计算分数，所以性能要比其他计算分数的要高一些，并且filter支持**缓存**

- should

#### **minimum_should_match**

- ````json
  GET /product/_search
  {
    "query": {
      "bool": {
        "should": [
          {"match": {
            "name": "xiaomi"
          }},
          
          {"match": {
            "desc": "shouji"
          }}
        ],
        "minimum_should_match": 1
        
      }
      
      
    }
    
  ````

- 先忽略上面的`minimum_should_match`，当执行后，会出现两个match只符合一个或者两个的结果，当时加上`minimum_should_match`的时候，如果指定的是1，那么就会出现只符合其中一个match的结果，如果设定的是2，那么只会出现符合两个match的结果
- 如果只有查询语句中，只有should，就像是上面的sql，那么minimum_should_match默认就是1，如果再上面的语句的基础上再加一个must或者其他的语句，那么默认就是0，这里就不再演示了

#### filter

- 具有缓存功能

- 不计算分数

- `````json
  
  GET /product/_search
  {
    "query": {
      "bool": {
      
      "filter":{
      "bool": {
        
        
        "should": [
          {"term": {"name": {"value": "xiaomi"}}},
          {"term": {"name": {"value": "nfc"}}}
        ],
        "must_not": [
          {"term": {"name": {"value": "erji"}}}
        ]
        
        
      }
      
      
    }
      }
    },
    "_source": []
    
    
  }
  
  `````

  

#### constant_score

- 下面的语句和上面的语句的结果是一样的

- `````json
  GET /product/_search
  {
    "query": {
      "constant_score": {
      
      "filter":{
      "bool": {
        
        
        "should": [
          {"term": {"name": {"value": "xiaomi"}}},
          {"term": {"name": {"value": "nfc"}}}
        ],
        "must_not": [
          {"term": {"name": {"value": "erji"}}}
        ]
        
        
      }
      
      
    }
      }
    },
    "_source": []
    
    
  }
  `````

- 









#### 比较term和match区别

- match，有结果

  - ````json
    
    GET /product/_search
    {
      "query": {
        "match": {
          "name": "xiaomi phone"
        }
      }
      
    }
    ````

- term，无结果

  - ````json
    GET /product/_search
    {
      "query":{
        "term": {"name": {"value": "xiaomi phone"}
        }
        
      }
    }
    ````

- 分析

  - match：ES会将‘xiaomi phone’进行分词，所以是有结果的
  - term：ES不会将‘xiaomi phone’分词，而doc中不含有xiaomi phone的数据，所以没有数据

 



#### 查看分词情况

````json
GET /_analyze
{
  "analyzer": "standard",
  "text" :"xiaommi nfc zhinenng phone" 
}
````

## Deep Paging( 深度分页)

### 解释

- 由于数据分布在不同的分片，查询大量的数据并且排序 是非常消耗性能的，，当数据超过1w条或者返回数据超过1千条的时候尽量不要使用	
- 比如说一个index一共是5w条数据，有5个分片，每个分片1w条，如果我们想要取5001-5050条数据，那么ES会从每个分片先排序然后各取5050条，然后再行排序

### 解决办法

- 使用scroll search

### Scroll Search

- ````json
  GET /product/_search?scroll=1m
  {
    "query": {
      "match": {
        "name": "phone"
      }
    },
    "sort": [
      {
        "price":"asc"
      }
    ],
    "size": 2
    
    
  }
    
  GET /_search/scroll
  {
     "scroll":"1m",##这个指的是加上这个参数，下面的id会延迟一分钟  "scroll_id":"FGluY2x1ZGVfY29udGV4dF91dWlkDXF1ZXJ5QW5kRmV0Y2gBFnUzVmo1RFBDUlhpUFJwTGRmV2NfYkEAAAAAAAAfMxZwaFBwVDlTeVE0eVk2eGtLeWxpMUJ3"
    
    
  }
  ````

- 1m指的是一分钟内有效

## filter缓存原理

1. filter并不是每次执行都会i进行cache，而是当执行一定次数的时候才会进行cache一个二进制数组，1表示匹配，2表示不匹配。这个次数是不固定的
2. filter会优先过滤掉稀疏的数据，保留匹配的cache数组
3. filter cache保存的是匹配的结果，不需要再从倒排索引中去查找比对，大大提高了查询速度
4. filter一般会再query之前执行，过滤掉一部分数据，从而提高query速度
5. filter不会计算相关度分数，在执行效率上比query高
6. 当原数据发生改变时，cache也会更新

## mapping

- 对于一个从未出过的index，如果第一次插入数据，在没有mapping的情况下，es会自动创建索引，对于text类型的字段，还有再创建一个fields的字段，类型是keyword,并且会自动只取前256个字符，超过256会截掉

  - ````json
    {
      "product" : {
        "mappings" : {
          "properties" : {
            "desc" : {
              "type" : "text",
              "fields" : {
                "keyword" : {
                  "type" : "keyword",
                  "ignore_above" : 256
                }
              }
            },
            "name" : {
              "type" : "text",
              "fields" : {
                "keyword" : {
                  "type" : "keyword",
                  "ignore_above" : 256
                }
              }
            },
            "price" : {
              "type" : "long"
            },
            "tags" : {
              "type" : "text",
              "fields" : {
                "keyword" : {
                  "type" : "keyword",
                  "ignore_above" : 256
                }
              }
            }
          }
        }
      }
    }
    
    ````


## 正排索引（doc values）

#### 为什么倒排索引进行聚合分析性能低下

- 如下图所示，ES会将index中的数据分词，并且建立下图右侧的表，聚合的时候，每个buket都要进行全表扫描，所以性能是极低的

![image-20220412223229208](..\image\为什么倒排索引进行聚合性能低.png)

#### 为什么要用正排索引进行聚合

-  ![image-20220412224329559](..\image\为什么要用正派索引进行聚合.png)

#### 正排和倒排的区别

1. 倒排索引的优势是：确定哪些doc包含了当前term，而正排索引的优势是：doc包含了哪些term
2. 正排和倒排都是在index-time（索引创建的时候）创建的

#### doc_value 和 fieldata:

- ![image-20220423211150774](..\image\image-20220423211150774.png)

- 在创建mapping的时候，如果类型是text，那么doc_value 是关闭的，要想实现聚合，只能通过打开fielddata

## 批量crud

#### 查询

- ````json
  ## 这种方式可以查询不同索引中的数据
  GET /_mget
  {
    "docs":[
      {
        "_index":"product",
        "_id":1
      },
      {
        "_index":"shirts",
        "_id":1 
      }
      ]
  }
  
  
  ##如果查询一个索引中的数据，那么就这样写
  GET /product/_mget
  {
    "docs":[
      {
       
        "_id":2
      },
      {
        
        "_id":1
      }
      ]
  }
  
  GET /product/_mget
  {
    "ids":[1,2,3
  } 
  ## 这样id是1的就只会出现name字段，其他字段不会出现         
  GET /product/_mget
  {
    "docs":[
      {
       
        "_id":2,
        "_source":false
      },
      {
        
        "_id":1,
        "_source":["name"]
      }
      ]
  }
  
  ## 这样id是1的就不会出现name字段
  GET /product/_mget
  {
    "docs":[
      {
       
        "_id":2,
        "_source":false
      },
      {
        
        "_id":1,
        "_source":{
          
          "exclude":["name"]
          
        }
      }
      ]
  }
    
     
    
  ````


#### bulk

- ES在更新的时候，历史的数据，依旧是保存在磁盘中，当累计到一定程度的历史数据的时候，ES会自动删除

  - ````json
    原来插入一条数据：
    PUT /product2/_doc/5
    {
        "name" : "hongmi erji",
        "desc" :  "erji zhong de kendeji",
        "price" :  399,
        "tags": [ "lowbee", "xuhangduan", "zhiliangx" ]
    }
    
    返回是：
    {
      "_index" : "product2",
      "_type" : "_doc",
      "_id" : "6",
      "_version" : 1,
      "result" : "created",
      "_shards" : {
        "total" : 2,
        "successful" : 1,
        "failed" : 0
      },
      "_seq_no" : 6,
      "_primary_term" : 1
    }
    进行修改的时候：
    PUT /product2/_doc/5
    {
        "name" : "hongmi erji",
        "desc" :  "erji zhong de kendeji",
        "price" :  399,
        "tags": [ "lowbee", "xuhangduan", "zhiliangx" ]
    }
    返回的是：
    {
      "_index" : "product2",
      "_type" : "_doc",
      "_id" : "5",
      "_version" : 2,注意这个version，已经改为了2
      "result" : "updated",
      "_shards" : {
        "total" : 2,
        "successful" : 1,
        "failed" : 0
      },
      "_seq_no" : 7,
      "_primary_term" : 1
    }
    
    ````

    

##### 插入

````json
POST /_bulk
{"create":{"_index":"product3","_id":"1"}}
{"name":"_buld crateBulk"}


POST /_bulk 
{"create":{"_index":"product3","_id":"1"}}
{"name":"_bul2d crateBulk"}
{"create":{"_index":"product3","_id":"2"}}
{"name":"_bul2d cratjioijoeBulk"}

````

##### 删除

``````json
POST /_bulk
{"delete":{"_index":"product3","_id":"2"}}
``````

##### 修改

- `retay_on_conflict`:意思是：当发生冲突的时候，会自动重试三次

`````json
POST /_bulk
{"update":{"_index":"product3","_id":"1","retay_on_conflict":"3"}}
{"doc":{"name":"uodateName"}}
`````

##### index

- `index`既可以是新增操作，也可以是全量替换操作

  `````json
  #id为1的doc原来没有grade字段，但经过index，就会将原来的数据替换掉
  #而原来没有id为12的数据，就会新增一个数据
  POST /_bulk
  {"index":{"_index":"product3","_id":"1"}} 
  {"name":"laalalala","grade":"male"}
  {"index":{"_index":"product3","_id":"12"}}
  {"name":"我是你爸爸","grade":"female"}
  `````

##### 过滤掉成功的数据

- 当需要使用bulk的时候，一般都是大量插入数据或者修改的场景，我们不会关心成功的返回状态，只会关心失败的返回状态，所以可以在bulk后面加上`filter_path=items.*.error`参数

- `````json
  POST /_bulk?filter_path=items.*.error
  {"create":{"_index":"product3","_id":"1"}}
  {"name":"huidsahduisa"}
  {"create":{"_index":"product3","_id":"14"}}
  {"name":"uidfhdiufd"}
  
  ##返回结果
  {
    "items" : [
      {
        "create" : {
          "error" : {
            "type" : "version_conflict_engine_exception",
            "reason" : "[1]: version conflict, document already exists (current version [3])",
            "index_uuid" : "EOYW7xNOTuCLLDwalM8Ksw",
            "shard" : "0",
            "index" : "product3"
          }
        }
      }
    ]
  }
  ## 由于之前id为1的数据已经存在了，所以，创建id为1的数据会报错，但是14没有存在，所以只返回了1的报错
  `````

  



## ES的并发操作

### 乐观锁

### 悲观锁





## 分词器

- 概念：将搜索的语句进行分词
  - 作用：
    - 分词
    - 提升召回率（normaliazation）
      - 对于英文，es会自动将一些搜索关键字进行替换或者更改时态，比如：将mom转为mother，has转为have等，这样就能提升es搜索的内容以及准确性
- 分析器
  - character filter :字符过滤器，分此前预处理，将一些无用字符或者词语进行转换或者去除，比如：= + - ！ @ # 等符号 或者 将 ‘的’ ‘啦’ ‘了’ 等词语去除，将<ElasticSearch>转为 ElasticSearch

````json

PUT my_index
{
  "settings": {
      ##自定义分析器
    "analysis": {
      ##字符过滤器
      "char_filter": {
        "my_char_filter": {
          "type": "html_strip",
      ##保证a标签不被去除
          "escaped_tags":["<a>"]
        }
      },
      "analyzer": {
        "my_analyzer": {
            ##分词器
          "tokenizer": "standard",
            ##使用自定义字符过滤器
          "char_filter": "my_char_filter"
        }
      }
    }
  }
}
 
  

````

- 测试

  - ````json
    GET /my_index/_analyze
    {
      "analyzer": "my_analyzer",
      "text": ["mashibing<a> <b>dsad <c>"]
      
    }
    {
      "tokens" : [
        {
          "token" : "mashibing",
          "start_offset" : 0,
          "end_offset" : 12,
          "type" : "<ALPHANUM>",
          "position" : 0
        },
        {
          "token" : "dsad",
          "start_offset" : 16,
          "end_offset" : 24,
          "type" : "<ALPHANUM>",
          "position" : 1
        }
      ]
    }
    
    ````

  - 

### ik分词器

#### 安装

- 直接编译对应ES版本的ik，然后放入ES的plugin文件夹中，然后重启ES

### 查看某个索引的分词器

- `GET /my_index2/_settings`

### 查看某个词语的分词状况

- ```json
  ## 粗粒度
  GET _analyze
  {
    "analyzer": "ik_smart",
    "text": "中韩渔警冲突调查：韩警平均每天扣1艘中国渔船"
    
  }
  ## 细粒度
  GET _analyze
  {
    "analyzer": "ik_max_word",
    "text": "北京市"
    
  }
  ```

### 添加新的分词

1. 当添加一个新的分词的时候，首先去`main.dic`文件中添加想要添加的分词，比如`老铁`

   - ![image-20220604191902674](..\image\添加分词.png)

2. 重启ES

3. ```json
   GET _analyze
   {
     "analyzer": "ik_max_word",
     "text": "老铁兄弟"
     
   }
   ## 现在会将老铁作为一个词语去分词
   {
     "tokens" : [
       {
         "token" : "老铁",
         "start_offset" : 0,
         "end_offset" : 2,
         "type" : "CN_WORD",
         "position" : 0
       },
       {
         "token" : "兄弟",
         "start_offset" : 2,
         "end_offset" : 4,
         "type" : "CN_WORD",
         "position" : 1
       }
     ]
   }
   
   ```

### 分词器

- 分词器包含：
  - character filter：用于分词前的预处理
    - 三种类型
      - html_strip
      - mapping
      - pattern_replace(正则)
  - tokenizer：分词
  - token filter：停用词、时态转换、大小写转换、语气词处理等

#### 自定义分词器

- 分词器一旦创建就不能修改，只能删除索引，再创建

- ````json
  PUT my_index
  {
    "settings": {
      "analysis": {
        "char_filter": {
          "my_char_filter": {
            "type": "html_strip",   
            "escaped_tags":["<a>"]  ## 这个是指将保留<a>标签
          }
        },
        "analyzer": {
          "my_analyzer": {
            "tokenizer": "keyword",
            "char_filter": "my_char_filter"
          }
        }
      }
    }
  }
  ````


## 前缀搜索 通配等

- **不建议使用这种方式，因为没有bitcache，会非常慢，尤其大数据量，并且不计算相关度评分**

### 前缀

- ````json
  
  GET /my_index2/_search
  {
    "query": {
      "prefix": {
        "text": {
          "value": "老"
        }
      }
     
    }
  }
  ````

- 

 

[ElasticSearch.yml]: 
