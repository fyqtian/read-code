### lsof

https://linux.cn/article-4099-1.html



https://www.netadmintools.com/html/lsof.man.html

- **默认** : 没有选项，lsof列出活跃进程的所有打开文件
- **组合** : 可以将选项组合到一起，如-abc，但要当心哪些选项需要参数
- **-a** : 结果进行“与”运算（而不是“或”）
- **-l** : 在输出显示用户ID而不是用户名w
- **-h** : 获得帮助
- **-t** : 仅获取进程ID
- **-U** : 获取UNIX套接口地址
- **-F** : 格式化输出结果，用于其它命令。可以通过多种方式格式化，如-F pcfn（用于进程id、命令名、文件描述符、文件名，并以空终止）



#### lsof -i  获取网络连接

lsof -i 6 获取ip6

lsof -iTCP

lsof -i :22

lsof -i@172.16.12.5

lsof -i@172.16.12.5:22

lsof -i -sTCP:LISTEN  |grep port



用户打开的文件

###  lsof -u root

kill -9 `lsof -t -u daniel` 清理某个用户打开的





### lsof -c syslog-ng 使用-c查看指定的命令正在使用的文件和网络连接



#### 使用-p查看指定进程ID已打开的内容

```
# lsof -p 10075
```



### lsof /var/log/messages/ 通过查看指定文件或目录，你可以看到系统上所有正与其交互的资源——包括用户、进程等。







***#* lsof -u daniel -i @1.1.1.1  显示daniel连接到1.1.1.1所做的一切**

bkdr   1893 daniel 3u  IPv6 3456 TCP 10.10.1.10:1234->1.1.1.1:31337 (ESTABLISHED)