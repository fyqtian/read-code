### Deploy

https://www.jianshu.com/p/ffc597bb4ce8

https://github.com/jaegertracing/jaeger/blob/master/docker-compose/jaeger-docker-compose.yml



https://www.elastic.co/guide/cn/kibana/current/docker.html

```yaml
version: '2.1'
services:

  elasticsearch:
    image: elasticsearch:5.6.4
    environment:
      - "ES_JAVA_OPTS=-Xms1024m -Xmx1024m"

  collector:
    image: jaegertracing/jaeger-collector
    environment:
      - SPAN_STORAGE_TYPE=elasticsearch
      - ES_SERVER_URLS=http://elasticsearch:9200
      - ES_USERNAME=elastic
      - LOG_LEVEL=debug
    depends_on:
      - elasticsearch

      #agent:
      #  image: jaegertracing/jaeger-agent
      # environment:
      #  - COLLECTOR_HOST_PORT=collector:14267
      # - LOG_LEVEL=debug
      #ports:
      #  - "5775:5775/udp"
      # - "5778:5778"
      # - "6831:6831/udp"
      # - "6832:6832/udp"
    # depends_on:
    #  - collector
  query:
    image: jaegertracing/jaeger-query
    environment:
      - SPAN_STORAGE_TYPE=elasticsearch
      - ES_SERVER_URLS=http://elasticsearch:9200
      - ES_USERNAME=elastic
      - LOG_LEVEL=debug
    ports:
      - 16686:16686
    depends_on:
      - elasticsearch

  kibana:
    image: docker.elastic.co/kibana/kibana:5.6.4
    environment:
      - ELASTICSEARCH_URL=http://elasticsearch:9200
      - ELASTICSEARCH_USERNAME=elastic
    ports:
      - 5601:5601
    depends_on:
      - elasticsearch

```

