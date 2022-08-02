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
#
#cluster.name: my-application
#
# ------------------------------------ Node ------------------------------------
#
# Use a descriptive name for the node:
#
#node.name: node-1
#
# Add custom attributes to the node:
#
#node.attr.rack: r1
#
# ----------------------------------- Paths ------------------------------------
#
# Path to directory where to store the data (separate multiple locations by comma):
#
#path.data: /path/to/data
#配置数据和日志的存放位置，生产环境必须配置，因为升级ES可能造成ES丢失
path.data: /path/to/data
#数据的路径可以设置多个，如下：
path:
    data:
        - path1
        - path2
        - path3
# ES会将每个分片的数据放在同一个路径上。但是ES不会平衡一个节点内不同分片的数据（个人理解意思是，如果一个节点的数据过大，大到引起 ）
# Path to log files:
#
#path.logs: /path/to/logs
#
# ----------------------------------- Memory -----------------------------------
#
# Lock the memory on startup:
#
#bootstrap.memory_lock: true
#
# Make sure that the heap size is set to about half the memory available
# on the system and that the owner of the process is allowed to use this
# limit.
#
# Elasticsearch performs poorly when the system is swapping the memory.
#
# ---------------------------------- Network -----------------------------------
#
# By default Elasticsearch is only accessible on localhost. Set a different
# address here to expose this node on the network:
#
#network.host: 192.168.0.1
#
# By default Elasticsearch listens for HTTP traffic on the first free port it
# finds starting at 9200. Set a specific HTTP port here:
#
#http.port: 9200
#
# For more information, consult the network module documentation.
#
# --------------------------------- Discovery ----------------------------------
#
# Pass an initial list of hosts to perform discovery when this node is started:
# The default list of hosts is ["127.0.0.1", "[::1]"]
#
#discovery.seed_hosts: ["host1", "host2"]
#
# Bootstrap the cluster using an initial set of master-eligible nodes:
#
#cluster.initial_master_nodes: ["node-1", "node-2"]
#
# For more information, consult the discovery and cluster formation module documentation.
#
# ---------------------------------- Various -----------------------------------
#
# Require explicit names when deleting indices:
#
#action.destructive_requires_name: true
#
# ---------------------------------- Security ----------------------------------
#
#                                 *** WARNING ***
#
# Elasticsearch security features are not enabled by default.
# These features are free, but require configuration changes to enable them.
# This means that users don’t have to provide credentials and can get full access
# to the cluster. Network connections are also not encrypted.
#
# To protect your data, we strongly encourage you to enable the Elasticsearch security features. 
# Refer to the following documentation for instructions.
#
# https://www.elastic.co/guide/en/elasticsearch/reference/7.16/configuring-stack-security.html

````

## 路径配置

### 日志和数据的存放位置

- 以下是配置数据和日志的存放位置
- **数据和日志的位置在生产环境下必须配置，因为升级ES可能造成ES丢失**

````yaml
#path.data: /path/to/data
# Path to log files:
#path.logs: /path/to/logs
````

### 多路径配置（7.13版本中已经被弃用，以后会删除）

````yaml
#数据的路径可以设置多个，如下：
path:
    data:
        - path1
        - path2
        - path3
````

- ES会将每个分片的数据放在同一个路径上
- 但是ES不会平衡一个节点内不同分片的数据
  - 个人理解意思是，如果一个节点的某一个路径的数据过大，大到引起 [high disk usage watermark](https://www.elastic.co/guide/en/elasticsearch/reference/7.16/modules-cluster.html#disk-based-shard-allocation),即使，其他路径还有空闲空间，ES也不会新增分片。**如果`high disk usage watermark`发生了，ES官方建议是，横向扩容，而不是将节点的硬盘新增。** 

## 发现节点配置（discovery.seed_hosts）

````yaml
discovery.seed_hosts:
   - 192.168.1.10:9300
   - 192.168.1.11 
   - seeds.mydomain.com #如果一个域名会被解析成多个ip地址，那么ES会在这些ip地址中寻找节点
   - [0:0:0:0:0:ffff:c0a8:10c]:9301 
````





