### query

https://www.elastic.co/guide/en/elasticsearch/reference/current/index.html



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

```

