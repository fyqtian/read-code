### influx



### influx

[Install InfluxDB | InfluxDB OSS 2.0 Documentation (influxdata.com)](https://docs.influxdata.com/influxdb/v2.0/install/?t=Docker)

```
docker run -d --rm --name influxdb -p 8086:8086 influxdb:2.0.7

docker run \
    --name influxdb \
    -p 8086:8086 \
    --volume $PWD:/var/lib/influxdb2 \
    influxdb:2.0.7
    
docker run \
  --rm influxdb:2.0.7 \
  influxd print-config > config.yml
  
  
docker run -p 8086:8086 \
  -v $PWD/config.yml:/etc/influxdb2/config.yml \
  influxdb:2.0.7
  
docker exec -it influxdb /bin/bash

docker run -p 8086:8086 influxdb:2.0.7 --reporting-disabled
```









### telegraf



https://github.com/influxdata/telegraf



```
默认生成config
telegraf config > telegraf-mysql.conf  

生成指定输出
telegraf --input-filter <pluginname>[:<pluginname>] --output-filter <outputname>[:<outputname>] config > telegraf.conf

telegraf --input-filter cpu:mem:disk:diskio:net --output-filter influxdb:opentsdb config > telegraf.conf


telegraf --input-filter cpu:mem:disk:diskio:net --output-filter file config > telegraf.conf

telegraf  -config telegraf.conf 

telegraf 日志目录 /var/log/telegraf/telegraf.log
```



```bash
git clone https://github.com/influxdata/sandbox.git


cd sandbox

# Start the sandbox
./sandbox up
```