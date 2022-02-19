### netdata

https://learn.netdata.cloud/docs/configure/nodes

https://www.gitiu.com/journal/netdata-install/

安装centos8 装不起来 用ubuntu

bash <(curl -Ss https://my-netdata.io/kickstart.sh)

配置

/etc/netdata/*.conf

/opt/netdata/etc/netdata/

http://NODE:19999 dashboard



启动

`sudo systemctl start netdata`.

service netdata start

killall netdata



store

When you use the database engine to store your metrics, you can always perform a quick backup of a node's `/var/cache/netdata/dbengine/` folder using the tool of your choice.





配置文件/etc/netdata/netdata.conf

curl -o netdata.conf http://127.0.0.1:19999/netdata.conf 下载



启动

/usr/sbin/netdata

killall netdata



netdatacli shutdown-agent

​					reload-health

killall -USR2 netdata



sudo systemctl stop netdata

sudo killall netdata

ps aux| grep netdata

`service netdata start`,





配置stream

https://blog.csdn.net/weixin_42867972/article/details/99832905







### Raised Alarms[#](https://learn.netdata.cloud/docs/agent/web/api/health#raised-alarms)



```
http://NODE:19999/api/v1/alarms?all
all alarm

http://NODE:19999/api/v1/alarms
This API call will return the alarms currently in WARNING or CRITICAL state.


http://NODE:19999/api/v1/alarm_log?after=UNIQUEID
The above returns all the events in the alarm log that occurred after UNIQUEID (you poll it once without after=, remember the last UNIQUEID of the returned set, which you give back to get incrementally the next events).




http://NODE:19999/api/v1/badge.svg?alarm=NAME&chart=CHART
The following will return an SVG badge of the alarm named NAME, attached to the chart named CHART.
```