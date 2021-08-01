### Prometheus



[First steps | Prometheus](https://prometheus.io/docs/introduction/first_steps/)

```
docker run --name prometheus -d -v  prometheus.yml:/etc/prometheus/prometheus.yml -p 9090:9090 prom/prometheus
```



```
docker run -p 9090:9090 -v /etc/prometheus/prometheus.yml:/etc/prometheus/prometheus.yml prom/prometheus
```



```bash
./prometheus --config.file=prometheus.yml
```



```
docker run --name node-exporter -d -p 9100:9100 prom/node-exporter:latest

docker run --name alertmanager -d -p 9093:9093 prom/alertmanager:latest

docker run -d
	--name alertmanager
	-p 9093:9093 
	-v /root/prometheus/alertmanager.yml:/etc/alertmanager/alertmanager.yml
	prom/alertmanager:latest

```



[Prometheus Time Series Collection and Processing Server](http://centos:9090/targets)



### 查询语法

[Querying basics | Prometheus](https://prometheus.io/docs/prometheus/latest/querying/basics/)

```
promhttp_metric_handler_requests_total

promhttp_metric_handler_requests_total{code="200"}

count(promhttp_metric_handler_requests_total)

http_requests_total{instance!="localhost:9090"}


使用label=~regx表示选择那些标签符合正则表达式定义的时间序列；
反之使用label!~regx进行排除；
http_requests_total{environment=~"staging|testing|development",method!="GET"}

最近5分钟
s - 秒
m - 分钟
h - 小时
d - 天
w - 周
y - 年
选择以当前时间为基准，5分钟内的数据
http_request_total{}[5m]


分钟前的瞬时样本数据，或昨天一天的区间内的样本数据呢? 这个时候我们就可以使用位移操作，位移操作的关键字为offset。

http_request_total{} offset 5m
http_request_total{}[1d] offset 1d
rate(prometheus_http_requests_total[5m] offset 1d)


# 查询系统所有http请求的总量
sum(prometheus_http_requests_total)

# 按照mode计算主机CPU的平均使用时间
avg(node_cpu) by (mode)

# 按照主机查询各个主机的CPU使用率
sum(sum(irate(node_cpu{mode!='idle'}[5m]))  / sum(irate(node_cpu[5m]))) by (instance)
```





### 函数

```
sum (求和)
min (最小值)
max (最大值)
avg (平均值)
stddev (标准差)
stdvar (标准差异)
count (计数)
count_values (对value进行计数)
bottomk (后n条时序)
topk (前n条时序)
quantile (分布统计)

<aggr-op>([parameter,] <vector expression>) [without|by (<label list>)]
其中只有count_values, quantile, topk, bottomk支持参数(parameter)。


without用于从计算结果中移除列举的标签，而保留其它标签。by则正好相反，结果向量中只保留列出的标签，其余标签则移除。通过without和by可以按照样本的问题对数据进行聚合。

sum(prometheus_http_requests_total) without (instance)
等价
sum(prometheus_http_requests_total) by (code,handler,job,method)


count_values("count", prometheus_http_requests_total)
```



















## Counter：只增不减的计数器

```
增长率
rate(http_requests_total[5m])

查询当前系统中，访问量前10的HTTP地址：
topk(10, http_requests_total)
```



## Gauge：可增可减的仪表盘

```
delta(cpu_temp_celsius{host="zeus"}[2h])

predict_linear(node_filesystem_free{job="node"}[1h], 4 * 3600)
```



## Histogram和Summary分析数据分布情况











### graph

```
rate(promhttp_metric_handler_requests_total{code="200"}[1m])
```



# Grafana

admin/admin

[Kalasearch/grafana-tutorial: Grafana 使用教程 (github.com)](https://github.com/Kalasearch/grafana-tutorial)

[Grafana 教程 - 构建你的第一个仪表盘 | 卡拉搜索 (kalasearch.cn)](https://kalasearch.cn/blog/grafana-with-prometheus-tutorial/)

[Prometheus 入门 - 奇妙的 Linux 世界 (hi-linux.com)](https://www.hi-linux.com/posts/25047.html)

```
docker run -d -p 3000:3000 grafana/grafana
```



### Prometheus.yaml

### Reload

This endpoint triggers a reload of the Prometheus configuration and rule files. It's **disabled** by default and can be enabled via the `--web.enable-lifecycle` flag.

Alternatively, a configuration reload can be triggered by sending a `SIGHUP` to the Prometheus process.

```
PUT  /-/reload
POST /-/reload
```

```
- job_name: 'node-exporter'
    scrape_interval: 5s
    file_sd_configs:
      - files: ['/usr/local/prometheus/groups/nodegroups/*.json']
```





### Alert rule

````
$ vim node-up.rules
groups:
- name: node-up
  rules:
  - alert: node-up
    expr: up{job="node-exporter"} == 0
    for: 15s
    labels:
      severity: 1
      team: node
    annotations:
      summary: "{{ $labels.instance }} 已停止运行超过 15s！"
   
通过$labels.<labelname>变量可以访问当前告警实例中指定标签的值。$value则可以获取当前PromQL表达式计算的样本值。   

      
alert：告警规则的名称。

expr：基于PromQL表达式告警触发条件，用于计算是否有时间序列满足该条件。

for：评估等待时间，可选参数。用于表示只有当触发条件持续一段时间后才发送告警。在等待期间新产生告警的状态为pending。

labels：自定义标签，允许用户指定要附加到告警上的一组附加标签。

annotations：用于指定一组附加信息，比如用于描述告警详细信息的文字等，annotations的内容在告警产生时会一同作为参数发送到Alertmanager。

````

**Inactive**：非活动状态，表示正在监控，但是还未有任何警报触发。

**Pending**：表示这个警报必须被触发。由于警报可以被分组、压抑/抑制或静默/静音，所以等待验证，一旦所有的验证都通过，则将转到 Firing 状态。

**Firing**：将警报发送到 AlertManager，它将按照配置将警报的发送给所有接收者。一旦警报解除，则将状态转到 Inactive，如此循环。



### AlertManager

```
global:
  resolve_timeout: 5m

route:
  group_by: ['alertname']
  group_wait: 10s
  group_interval: 10s
  repeat_interval: 1h
  receiver: 'web.hook'
receivers:
- name: 'web.hook'
  webhook_configs:
  - url: 'http://127.0.0.1:5001/'
inhibit_rules:
  - source_match:
      severity: 'critical'
    target_match:
      severity: 'warning'
    equal: ['alertname', 'dev', 'instance']
    
    
    
global:
  [ resolve_timeout: <duration> | default = 5m ]
  [ smtp_from: <tmpl_string> ] 
  [ smtp_smarthost: <string> ] 
  [ smtp_hello: <string> | default = "localhost" ]
  [ smtp_auth_username: <string> ]
  [ smtp_auth_password: <secret> ]
  [ smtp_auth_identity: <string> ]
  [ smtp_auth_secret: <secret> ]
  [ smtp_require_tls: <bool> | default = true ]
  [ slack_api_url: <secret> ]
  [ victorops_api_key: <secret> ]
  [ victorops_api_url: <string> | default = "https://alert.victorops.com/integrations/generic/20131114/alert/" ]
  [ pagerduty_url: <string> | default = "https://events.pagerduty.com/v2/enqueue" ]
  [ opsgenie_api_key: <secret> ]
  [ opsgenie_api_url: <string> | default = "https://api.opsgenie.com/" ]
  [ hipchat_api_url: <string> | default = "https://api.hipchat.com/" ]
  [ hipchat_auth_token: <secret> ]
  [ wechat_api_url: <string> | default = "https://qyapi.weixin.qq.com/cgi-bin/" ]
  [ wechat_api_secret: <secret> ]
  [ wechat_api_corp_id: <string> ]
  [ http_config: <http_config> ]

templates:
  [ - <filepath> ... ]

route: <route>

receivers:
  - <receiver> ...

inhibit_rules:
  [ - <inhibit_rule> ... ]
```



Alertmanager的配置主要包含两个部分：**路由(route)以及接收器(receivers)**。所有的告警信息都会从配置中的顶级路由(route)进入路由树，根据路由规则将告警信息发送给相应的接收器。













### API

[HTTP API | Prometheus](https://prometheus.io/docs/prometheus/latest/querying/api/)

```
curl 'http://localhost:9090/api/v1/query?query=up&time=2015-07-01T20:10:51.781Z'
{
  "status": "success",
  "data": {
    "resultType": "vector",
    "result": [
      {
        "metric": {
          "__name__": "up",
          "instance": "172.26.0.109:9100",
          "job": "mynodeport"
        },
        "value": [
          1627011246.537,
          "1"
        ]
      },
      {
        "metric": {
          "__name__": "up",
          "instance": "localhost:19999",
          "job": "netdata"
        },
        "value": [
          1627011246.537,
          "1"
        ]
      },
      {
        "metric": {
          "__name__": "up",
          "instance": "localhost:9090",
          "job": "prometheus"
        },
        "value": [
          1627011246.537,
          "1"
        ]
      },
      {
        "metric": {
          "__name__": "up",
          "instance": "localhost:9100",
          "job": "mynodeport"
        },
        "value": [
          1627011246.537,
          "1"
        ]
      }
    ]
  }
}

curl 'http://localhost:9090/api/v1/query_range?query=up&start=2015-07-01T20:10:30.781Z&end=2015-07-01T20:11:00.781Z&step=15s'






 curl -g 'http://localhost:9090/api/v1/series?' \
 --data-urlencode 'match[]=up' \
 --data-urlencode 'match[]=process_start_time_seconds{job="prometheus"}'
 
 up or process_start_time_seconds{job="prometheus"}:
 
 
 
 curl 'localhost:9090/api/v1/labels'
 
 
 curl http://localhost:9090/api/v1/label/job/values
 
 curl http://localhost:9090/api/v1/targets?state=active
 
curl http://localhost:9090/api/v1/rules
  
curl http://localhost:9090/api/v1/alerts


-G, --get           Send the -d data with a HTTP GET (H)
curl -G http://localhost:9091/api/v1/targets/metadata \
    --data-urlencode 'metric=go_goroutines' \
    --data-urlencode 'match_target={job="prometheus"}' \
    --data-urlencode 'limit=2'
    
    
curl -G http://localhost:9091/api/v1/targets/metadata \
    --data-urlencode 'match_target={instance="127.0.0.1:9090"}'
    
curl -G http://localhost:9090/api/v1/metadata?limit=2


curl -G http://localhost:9090/api/v1/metadata?metric=http_requests_total

curl http://localhost:9090/api/v1/alertmanagers
 
curl http://localhost:9090/api/v1/status/config

curl http://localhost:9090/api/v1/status/flags

curl http://localhost:9090/api/v1/status/runtimeinfo

curl http://localhost:9090/api/v1/status/buildinfo

curl http://localhost:9090/api/v1/status/tsdb


curl -XPOST http://localhost:9090/api/v1/admin/tsdb/snapshot

curl -X POST \
  -g 'http://localhost:9090/api/v1/admin/tsdb/delete_series?match[]=up&match[]=process_start_time_seconds{job="prometheus"}'
  
  
curl -XPOST http://localhost:9090/api/v1/admin/tsdb/clean_tombstones
```

