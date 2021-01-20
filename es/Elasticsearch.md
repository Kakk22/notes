# Elasticsearch

## 简介

Elasticsearch是一个基于Lucene的搜索服务器。它提供了一个分布式的全文搜索引擎，基于restful web接口。Elasticsearch是用Java语言开发的，基于Apache协议的开源项目，是目前最受欢迎的企业搜索引擎。Elasticsearch广泛运用于云计算中，能够达到实时搜索，具有稳定，可靠，快速的特点。

## Linux安装

### Elasticsearch

- 下载elasticsearch 6.4.0的docker镜像；

```bash
docker pull elasticsearch:6.4.0
```

- 修改虚拟内存区域大小，否则会因为过小而无法启动；

```bash
sysctl -w vm.max_map_count=262144
```

- 给挂载目录加权限

```bash
chmod 777 /mydata/elasticsearch/data/
```

- 使用docker命令启动

```bash
docker run -p 9200:9200 -p 9300:9300 --name elasticsearch \
-e "discovery.type=single-node" \
-e "cluster.name=elasticsearch" \
-e  ES_JAVA_OPTS="-Xms256m -Xmx256m" \
-v /mydata/elasticsearch/plugins:/usr/share/elasticsearch/plugins \
-v /mydata/elasticsearch/data:/usr/share/elasticsearch/data \
-d elasticsearch:6.4.0
```

- 安装中文分词器IKAnalyzer，并重新启动；

```bash
docker exec -it elasticsearch /bin/bash
#此命令需要在容器中运行
elasticsearch-plugin install https://github.com/medcl/elasticsearch-analysis-ik/releases/download/v6.4.0/elasticsearch-analysis-ik-6.4.0.zip
docker restart elasticsearch
```

### Kibina

- 下载kibana 6.4.0的docker镜像；

```bash
docker pull kibana:6.4.0Copy to clipboardErrorCopied
```

- 使用docker命令启动；

```bash
docker run --name kibana -p 5601:5601 \
--link elasticsearch:es \
-e "elasticsearch.hosts=http://es:9200" \
-d kibana:6.4.0Copy to clipboardErrorCopied
```

- 访问地址进行测试：http://ip:5601/

![image-20201110210906373](https://gitee.com/chen_yi_fenga/blog-imag/raw/master/image-20201110210906373.png)

## 相关概念

- Near Realtime（近实时）：Elasticsearch是一个近乎实时的搜索平台，这意味着从索引文档到可搜索文档之间只有一个轻微的延迟(通常是一秒钟)。
- Cluster（集群）：群集是一个或多个节点的集合，它们一起保存整个数据，并提供跨所有节点的联合索引和搜索功能。每个群集都有自己的唯一群集名称，节点通过名称加入群集。
- Node（节点）：节点是指属于集群的单个Elasticsearch实例，存储数据并参与集群的索引和搜索功能。可以将节点配置为按集群名称加入特定集群，默认情况下，每个节点都设置为加入一个名为`elasticsearch`的群集。
- Index（索引）：索引是一些具有相似特征的文档集合，类似于MySql中数据库的概念。
- Type（类型）：类型是索引的逻辑类别分区，通常，为具有一组公共字段的文档类型，类似MySql中表的概念。`注意`：在Elasticsearch 6.0.0及更高的版本中，一个索引只能包含一个类型。
- Document（文档）：文档是可被索引的基本信息单位，以JSON形式表示，类似于MySql中行记录的概念。
- Shards（分片）：当索引存储大量数据时，可能会超出单个节点的硬件限制，为了解决这个问题，Elasticsearch提供了将索引细分为分片的概念。分片机制赋予了索引水平扩容的能力、并允许跨分片分发和并行化操作，从而提高性能和吞吐量。
- Replicas（副本）：在可能出现故障的网络环境中，需要有一个故障切换机制，Elasticsearch提供了将索引的分片复制为一个或多个副本的功能，副本在某些节点失效的情况下提供高可用性。

## 集群状态查看

- 查看集群健康状态；

```bash
GET /_cat/health?vCopy to clipboardErrorCopied
epoch      timestamp cluster       status node.total node.data shards pri relo init unassign pending_tasks max_task_wait_time active_shards_percent
1585552862 15:21:02  elasticsearch yellow          1         1     27  27    0    0       25             0                  -                 51.9%Copy to clipboardErrorCopied
```

- 查看节点状态；

```bash
GET /_cat/nodes?vCopy to clipboardErrorCopied
ip        heap.percent ram.percent cpu load_1m load_5m load_15m node.role master name
127.0.0.1           23          94  28                          mdi       *      KFFjkpVCopy to clipboardErrorCopied
```

- 查看所有索引信息；

```bash
GET /_cat/indices?vCopy to clipboardErrorCopied
health status index    uuid                   pri rep docs.count docs.deleted store.size pri.store.size
green  open   pms      xlU0BjEoTrujDgeL6ENMPw   1   0         41            0     30.5kb         30.5kb
green  open   .kibana  ljKQtJdwT9CnLrxbujdfWg   1   0          2            1     10.7kb         10.7kbCopy to clipboardErrorCopied
```

## 索引操作

- 创建索引并查看；

```bash
PUT /customer
GET /_cat/indices?vCopy to clipboardErrorCopied
health status index    uuid                   pri rep docs.count docs.deleted store.size pri.store.size
yellow open   customer 9uPjf94gSq-SJS6eOuJrHQ   5   1          0            0       460b           460b
green  open   pms      xlU0BjEoTrujDgeL6ENMPw   1   0         41            0     30.5kb         30.5kb
green  open   .kibana  ljKQtJdwT9CnLrxbujdfWg   1   0          2            1     10.7kb         10.7kbCopy to clipboardErrorCopied
```

- 删除索引并查看；

```bash
DELETE /customer
GET /_cat/indices?vCopy to clipboardErrorCopied
health status index    uuid                   pri rep docs.count docs.deleted store.size pri.store.size
green  open   pms      xlU0BjEoTrujDgeL6ENMPw   1   0         41            0     30.5kb         30.5kb
green  open   .kibana  ljKQtJdwT9CnLrxbujdfWg   1   0          2            1     10.7kb         10.7kb
```





## 搜索查询

```JSON
#查询所有
GET /bank/_search
{
  "query": {
    "match_all": {}
  }
}

#分页
GET /bank/_search
{
  "query": {
    "match_all": {}
  }, 
  "from": 0,
  "size": 20
}

#排序
GET /bank/_search
{
  "query": {
    "match_all": {}
  }
  , "sort": [
    {
      "account_number": {
        "order": "desc"
      }
    }
  ]
}
#搜索所有
GET /bank/_search
{
  "query": {
    "match_all": {}
  },
  "_source": ["account_number","firstname"]
  
}

#条件搜索
GET /bank/_search
{
  "query": {
    "match": {
      "account_number": "20"
    }
  }
}

#条件搜索 只显示指定
GET /bank/_search
{
  "query": {
    "match": {
      "address": "mill"
    }
  },
  "_source": ["account_number","firstname"]
  
}

#短语匹配搜索
GET /bank/_search
{
  "query": {
    "match_phrase": {
      "address": "mill lane"
    }
  }
}

#组合搜索 must 同时成立
GET /bank/_search
{
  "query": {
    "bool": {
      "must": [
        {
          "match": {
            "address": "mill"
          }
        },
        {
          "match": {
            "address": "lane"
          }
        }
      ]
    }
  }
}
#组合搜索 should 满足其中一个
GET /bank/_search
{
  "query": {
    "bool": {
      "should": [
        {
          "match": {
            "address": "mill"
          }
        },
        {
          "match": {
            "address": "lane"
          }
        }
      ]
    }
  }
}


#组合搜索 must_not
GET /bank/_search
{
  "query": {
    "bool": {
      "must_not": [
        {
          "match": {
            "address": "mill"
          }
        },
        {
          "match": {
            "address": "lane"
          }
        }
      ]
    }
  }
}

#组合搜索，组合must和must_not，
GET /bank/_search
{
  "query": {
    "bool": {
      "must": [
        {
          "match": {
            "age": "40"
          }
        }
      ],
      "must_not": [
        {"match": {
          "state": "ID"
        }}
      ]
    }
  }
}


## 过滤搜索
GET /bank/_search
{
  "query": {
    "bool": {
      "must": [
        {"match_all": {}}
      ],
      "filter": {
        "range": {
          "balance": {
            "gte": 20000,
            "lte": 30000
          }
        }
      }
    }
  }
}

#搜索聚合 对搜索结果进行聚合，使用aggs来表示，
#似于MySql中的group
#by，例如对state字段进行聚合，统计出相同state
#文档数量；
GET /bank/_search
{
  "size": 0,
  "aggs": {
    "group_by_state": {
      "terms": {
        "field": "state.keyword"
      }
    }
  }
}


#嵌套聚合，例如对state字段进行聚合，统计出相同
#state的文档数量，再统计出balance的平均值
GET /bank/_search
{
  "size": 0,
  "aggs": {
    "group_by_state": {
      "terms": {
        "field": "state.keyword"
      },
      "aggs": {
        "average_balance": {
          "avg": {
            "field": "balance"
          }
        }
      }
    }
  }
}


#聚合搜索的结果进行排序，例如按balance的平均值
#序排列；
GET /bank/_search
{
  "size": 0,
  "aggs": {
    "group_by_state": {
      "terms": {
        "field": "state.keyword",
        "order": {
          "average_balance": "desc"
        }
      },
      "aggs": {
        "average_balance": {
          "avg": {
            "field": "balance"
          }
        }
      }
    }
  }
}


#按字段值的范围进行分段聚合，例如分段范围为age
#段的[20,30] [30,40] 
#[40,50]，之后按gender统计文档个数和balance的平
#均值；
GET /bank/_search
{
  "size": 0,
  "aggs": {
    "group_by_age": {
      "range": {
        "field": "age",
        "ranges": [
          {
            "from": 20,
            "to": 30
          },
          {
            "from": 30,
            "to": 40
          },
          {
            "from": 40,
            "to": 50
          }
        ]
      },
      "aggs": {
        "group_by_gender": {
          "terms": {
            "field": "gender.keyword"
          },
          "aggs": {
            "average_balance": {
              "avg": {
                "field": "balance"
              }
            }
          }
        }
      }
    }
  }
}

```

