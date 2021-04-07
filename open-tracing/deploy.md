### Deploy

https://www.jianshu.com/p/ffc597bb4ce8

https://github.com/jaegertracing/jaeger/blob/master/docker-compose/jaeger-docker-compose.yml

https://www.elastic.co/guide/cn/kibana/current/docker.html kibana

https://www.elastic.co/guide/cn/kibana/current/connect-to-elasticsearch.html



https://www.elastic.co/guide/en/kibana/7.12/get-started.html

https://docs.docker.com/compose/gettingstarted/ 

https://deepzz.com/post/docker-compose-file.html#toc_31



https://github.com/deviantony/docker-elk











```sh
docker run -d   --net=dockerelk_elk  -e SPAN_STORAGE_TYPE=elasticsearch -e ES_SERVER_URLS=http://elasticsearch:9200  -e ES_USERNAME=elastic -e ES_PASSWORD=changeme  -p 14268:14268 -p 9411:9411 jaegertracing/jaeger-collector


docker run -d  \
  --net=dockerelk_elk \
  -p 16686:16686 \
  -p 16687:16687 \
  -e SPAN_STORAGE_TYPE=elasticsearch \
  -e ES_SERVER_URLS=http://elasticsearch:9200 \
  -e ES_PASSWORD=changeme \
  -e ES_USERNAME=elastic \
  jaegertracing/jaeger-query
```