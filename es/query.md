### query

https://www.elastic.co/guide/en/elasticsearch/reference/current/index.html

Copy_to的字段需要显示申明store

```
 https://www.elastic.co/guide/en/elasticsearch/reference/current/copy-to.html
 "region": {
            "type": "text",
            "store": true
  }
  查询的时候
  GET twitter/_doc/1?stored_fields=region
```



```
{
    "mappings": {
        "properties": {
			"provider": { "type": "keyword" },
			"accountId": { "type": "keyword" },
			"regionId": { "type": "keyword" },
			"id": { "type": "keyword" },
			"name": { "type": "text" },
			"type": { "type": "keyword" },
			"tags": { "type": "text" },
			"imageId": { "type": "keyword" },
			"ipAddress": {
				"type": "nested",
				"properties": {
					"type": { "type": "keyword" },
					"ip": { "type": "ip" }
				},
				"dynamic": true 
			},
			"status": { "type": "keyword" },
			"vpcId": { "type": "keyword" },
			"secretKey": { "type": "keyword" },
			"detail": { "type": "text" },
			"ips": {"type": "ip"}
        }
    }
}`
```



```
分词查
GET x-devops-ecs/_search
{
  "query": {
    "match": {
      "detail": {
        "query": "192.168.1.11"
      }
    }
  }
}


嵌套查询
GET x-devops-ecs/_search
{
  "query": {
    "nested": {
      "path": "ipAddress",
      "query": {
        "bool": {
          "must": [
            { "match": { "ipAddress.ip": "192.168.1.11" }}
          ]
        }
      }
    }
  }
}


GET x-devops-slb/_search
{
  "query": {
   "match": {
     "name": {
       "query": "test-eden",
       "operator": "and"
     }
   }
  }
}


```



_id查询

```
GET x-devops-ecs/_doc/aliyunus-east-1i-99ihm6h0e
```







更新某个字段

```
POST x-devops-ecs/_update
{
  "doc": {
   "lastSyncTime" : "2021-05-05 11:08:33"
  }
}

```

增加字段

```

PUT  /x-devops-security-group/_mapping
{
   "properties": {
        "status":{
          "type":"keyword"
        },
        "lastSyncTime": {
            "type": "date",
            "format": "yyyy-MM-dd HH:mm:ss"
        }
    }

}
```



```
GET x-devops-slb/_search
{
  "query": {
    "bool": {
      "must": [
        {"term": {
          "provider": {
            "value": "gcp"
          }
        }},{"match": {
          "search_all": "INTERNAL"
        }}
      ]
    }
  }
}


删除

POST /x-devops-slb/_delete_by_query
{
  "query": {
    "term": {
      "provider": {
        "value": "gcp"
      }
    }
  }
}


```



distinct

```
{
  "query": {
    "term": {
      "user_id_type": 3
    }
  },
  "collapse": {
    "field": "user_id"
  }
}




{
  "query": {
    "term": {
      "status": "release"
    }
  },
  "aggs": {
    "count": {
      "cardinality": {
        "field": "_id"
      }
    }
  }
}



count 


GET x-devops-ecs/_count
{
  "query": {
    "bool": {
      "must": [
        {"term": {
          "status": {
            "value": "release"
          }
        }}
      ]
    }
  }
}



```

