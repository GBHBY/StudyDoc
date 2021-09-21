# ES运维
## 集群管理

### 集群配置

#### 节点

1. 相同角色节点，不能是用较大差异的配置
2. 避免使用超级服务器，比如128个核心，1T内存，不仅造成资源浪费，并且会降低性能
3. 等量的配置，使用较少的物理机，好于用较多的虚拟机，因为虚拟机本身就消耗性能，并且有数据安全的问题，因为宿主机如果数据出现了损失，那么虚拟机也有可能出现问题
4. 服务器最好是大于等于5台
5. 避免一台服务器部署多个节点，不方便管理

#### 内存、磁盘

1. 根据业务量的不同，对内存的需求也不同，最好是不少于16G，但不要超过64G。ES对内存的消耗非常非常大，本身ES就有jvm，ES对内存的依赖要大于CPU。

   - 百万级设置16G

   - 千万级 32G

   - 亿级 64G

   - 如果还不够，可以增加节点，增加内存效率不高

2. 磁盘

   - 对于ES来说，磁盘是最重要的，因为数据都是存储在磁盘上，这里说的磁盘是指的性能，计算机的性能瓶颈是磁盘
   - 优化
     - 数据进行冷热分离：热节点使用SSD，冷节点可以在需要的时候开着，不需要的时候关着

3. CPU

   - ES对CPU不是依赖特别高，提升CPU没有提升磁盘或者内存的效果那么高，master比其他节点对CPU的要求稍微高一些。服务器的CPU不太需要单核性能，需要更多的是核心数

4. 网络

   - ES天生分布式，ES的分布式基于对等网络，节点与节点之间的通信非常频繁。高延迟对ES是致命的，比如淘宝很慢的时候是用户是很崩溃的。

## 集群规划

- 在集群搭建之前，首先要搞清楚，为什么要使用ES，如果是站内搜索或者搭建日志，或者是数据分析，针对不同的场景，有不同的优化方案
- 集群需要多少种配置，这要看业务量来定

##  配置文件

````yaml
# ======================== Elasticsearch Configuration =========================
#
# NOTE: Elasticsearch comes with reasonable defaults for most settings.
#       Before you set out to tweak and tune the configuration, make sure you
#       understand what are you trying to accomplish and the consequences.
#
# The primary way of configuring a node is via this file. This template lists
# the most important settings you may want to configure for a production cluster.
#
# Please consult the documentation for further information on configuration options:
# https://www.elastic.co/guide/en/elasticsearch/reference/index.html
#
# ---------------------------------- Cluster -----------------------------------
#
# Use a descriptive name for your cluster:
#通过这个来寻找其他集群中的节点
cluster.name: gb-es
#
# ------------------------------------ Node ------------------------------------
#
# Use a descriptive name for the node:
#
node.name: gb-01
#
# Add custom attributes to the node:
#
#node.attr.rack: r1
#
# ----------------------------------- Paths ------------------------------------
#
# Path to directory where to store the data (separate multiple locations by comma):
#如果单台服务器配置了多个节点，那么就不能配置同一路径，否则引导检查不通过
path.data: G:/ES/ES-01/data
#
# Path to log files:
#如果单台服务器配置了多个节点，那么就不能配置同一路径，否则引导检查不通过
path.logs: G:/ES/ES-01/log
#
# ----------------------------------- Memory -----------------------------------
#
# Lock the memory on startup:
#内存锁：非常重要，作用是当内存大小不够，禁用swap，必须配true
bootstrap.memory_lock: true
#
# Make sure that the heap size is set to about half the memory available
# on the system and that the owner of the process is allowed to use this
# limit.
#
# Elasticsearch performs poorly when the system is swapping the memory.
#
# ---------------------------------- Network -----------------------------------
#
# Set the bind address to a specific IP (IPv4 or IPv6):
#
#network.host: localhost
#
# Set a custom port for HTTP:
#ES端口号
http.port: 9200   
#节点间通讯端口，tpc协议的
transport.port: 9300
#
# For more information, consult the network module documentation.
#
# --------------------------------- Discovery ----------------------------------
#
# Pass an initial list of hosts to perform discovery when this node is started:
# The default list of hosts is ["127.0.0.1", "[::1]"]
#配置master候选节点：
discovery.seed_hosts: ["127.0.0.1:9300", "127.0.0.1:9301"]
#
# Bootstrap the cluster using an initial set of master-eligible nodes:
#设置第一次启动的仅master节点，会自动从以下选出一个，可以是ip地址也可以是节点名字
cluster.initial_master_nodes: ["gb-01"]
#
# For more information, consult the discovery and cluster formation module documentation.
#
# ---------------------------------- Gateway -----------------------------------
#
# Block initial recovery after a full cluster restart until N nodes are started:
#
#gateway.recover_after_nodes: 3
#
# For more information, consult the gateway module documentation.
#
# ---------------------------------- Various -----------------------------------
#
# Require explicit names when deleting indices:
#
#action.destructive_requires_name: true
#允许跨域
http.cors.enabled: true
#跳板机的地址
http.cors.allow-origin: "*"
node.master: true
node.data: true
````

### 其他配置

- `max_result_window:10000`,这个配置默认是1w，分页的时候，假设我们要取年龄最大的500个数据，会先从每个分片上排序然后取500，假设分片是3个，然后再从这1500中排序再取前500，而这个参数就是只最终取出的数据条数的最大值，如果改了配置，很有可能造成oom。`(https://www.elastic.co/guide/en/elasticsearch/reference/7.6/search-request-body.html#request-body-search-search-after)`官方给的深度分页
- ES的默认配置基本就是最好的配置，不可能通过修改一项或者几项配置就能提升效率
-  官方推荐JVM不要改

## JVM

- 每个版本的配置不太一样，7.6的JVM堆内存是1G，这个值有点小，应该调大。7.9默认是1G，但推荐的不是1G

- **Java的堆内存不要超过物理机内存的50%，即使堆很大，也不要超过32G**（官网说的），分页、fielddata都会使用堆内存，如果堆内存特别大，那么系统内存就会变得很小，而系统内存过小会直接导致ES性能下降，ES基于Lucene，Lucene是通过系统内存提升性能的
  
  - 修改JVM
    -  在ES进程启动的时候加上参数：ES_JAVA_OPTS="-Xms512m -Xmx512m",**或**在jvm.options中修改-Xms和-Xmx，注意最大值和最小值要相同，避免jvm heap在运行中resize，这是一个非常耗性能的过程。
    - 修改环境变量：ES_HEAP_SIZE，环境变量的优先级高于jvm.options中的设置的数值。
  
  

## 如何避免脑裂

- 候选节点不能少于3个，并且集群的数量一定要是奇数
- `discovery.zen.minimum_master_nodes=N/2+1` 这样配置可以避免脑裂，N是候选节点的数量
- `(https://www.elastic.co/guide/en/elasticsearch/reference/7.6/modules-discovery-voting.html)`，这是关于投票的官方介绍
- 

