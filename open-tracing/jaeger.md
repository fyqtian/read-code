### jaeger

https://research.google.com/pubs/pub36356.html dapper

https://www.jaegertracing.io/docs/1.22/ 文档



### all in one （用于测试环境）

https://www.jaegertracing.io/docs/1.22/getting-started/#all-in-one

https://blog.csdn.net/sniperking2008/article/details/103762543

```dockerfile
docker run -d -e COLLECTOR_ZIPKIN_HTTP_PORT=9411 -p 5775:5775/udp -p 6831:6831/udp -p 6832:6832/udp -p 5778:5778  -p 16686:16686 -p 14268:14268  -p 14269:14269   -p 9411:9411 jaegertracing/all-in-one:latest

```

端口				协议				组件					功能
5775			  UDP				agent				  接受zipkin.thrift compact thrift 协议（已过时，仅旧客户端使用）
6831			  UDP	   		 agent				  接受jaeger.thrift compact thrift 协议
6832			  UDP				agent				  接受jaeger.thrift binary thrift 协议
5778			  HTTP			   agent				  服务配置
16686			HTTP			   query				 服务前端  dashboard
14268			HTTP			   collector			直接从客户端接受jaeger.thrift协议

14250            HTTP                collector		  proto 

9411			  HTTP				collector			兼容 Zipkin 服务（可选的）





### jaeger+elasticsearch

https://my.oschina.net/u/2548090/blog/1821372 





```json
{
        "_index" : "jaeger-span-2021-04-01",
        "_type" : "_doc",
        "_id" : "Fmv_ingB5T5JhUaVPJVN",
        "_score" : 1.0,
        "_source" : {
          "traceID" : "5691daccfaadf424",
          "spanID" : "0c73af8d30275d93",
          "flags" : 1,
          "operationName" : "span_foo3",
          "references" : [
            {
              "refType" : "CHILD_OF",
              "traceID" : "5691daccfaadf424",
              "spanID" : "5691daccfaadf424"
            }
          ],
          "startTime" : 1617239683660295,
          "startTimeMillis" : 1617239683660,
          "duration" : 503458,
          "tags" : [
            {
              "key" : "request",
              "type" : "string",
              "value" : "Hello foo3"
            },
            {
              "key" : "reply",
              "type" : "string",
              "value" : "foo3Reply"
            },
            {
              "key" : "internal.span.format",
              "type" : "string",
              "value" : "jaeger"
            }
          ],
          "logs" : [ ],
          "process" : {
            "serviceName" : "jaeger-demo",
            "tags" : [
              {
                "key" : "jaeger.version",
                "type" : "string",
                "value" : "Go-2.25.0"
              },
              {
                "key" : "hostname",
                "type" : "string",
                "value" : "yuxiadeMBP.lan"
              },
              {
                "key" : "ip",
                "type" : "string",
                "value" : "192.168.2.135"
              },
              {
                "key" : "client-uuid",
                "type" : "string",
                "value" : "377aab8f919e8d4a"
              }
            ]
          }
        }
      }
```



### jaeger+Cassandra

https://blog.csdn.net/niyuelin1990/article/details/80225305



