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



### jaeger+Cassandra

https://blog.csdn.net/niyuelin1990/article/details/80225305