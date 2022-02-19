### IPVS



https://www.qikqiak.com/post/how-to-use-ipvs-in-kubernetes/



https://github.com/kubernetes/kubernetes/blob/master/pkg/proxy/ipvs/README.md





ip addr add 10.222.0.1/24 dev enp0s3

### vip实现

https://zhaohuabing.com/post/2019-11-27-vip/





### Ipvsadm 命令

https://www.jianshu.com/p/4a3496b8006a

https://www.jianshu.com/p/7cff00e253f4  ipset

```
ipvsadm -A -t 207.175.44.110:80 -s rr && ipvsadm -a -t 207.175.44.110 -r  172.26.0.109 -m

iptables -t nat -A POSTROUTING -m ipvs  --vaddr 10.222.0.1 --vport 8080 -j MASQUERADE
```

```text
sysctl net.ipv4.vs.conntrack=1

apt install conntrack
```

### ip 命令

https://access.redhat.com/sites/default/files/attachments/rh_ip_command_cheatsheet_1214_jcs_print.pdf



### k8s service ping  问题

https://segmentfault.com/a/1190000039349716

```
IPVS：clusterIP 会绑定到虚拟网卡 kube-ipvs0，配置了 route 路由到回环网卡，icmp 包是 lo 网卡回复的。可以 ping 通。

```



### netshoot工具

https://github.com/nicolaka/netshoot





### kind k8s in docker

https://kind.sigs.k8s.io/docs/user/quick-start/#installation

https://www.cnblogs.com/weihanli/p/12831225.html

https://github.com/kubernetes-sigs/cri-tools





### ipvs

https://www.hwchiu.com/ipvs-1.html

https://www.qikqiak.com/post/how-to-use-ipvs-in-kubernetes/

https://www.cnblogs.com/luozhiyun/p/13782077.html

https://cloud.tencent.com/developer/article/1607796

https://zhuanlan.zhihu.com/p/94418251



https://www.cnblogs.com/blxt/p/13099424.html

https://blog.dianduidian.com/post/lvs-snat%E5%8E%9F%E7%90%86%E5%88%86%E6%9E%90/





### dummy网卡

https://www.cnblogs.com/FengZeng666/p/15583434.html

https://www.kancloud.cn/chunyu/php_basic_knowledge/2137337

https://man7.org/linux/man-pages/man8/ip-link.8.html

```
 ip link add nodelocaldns type dummy
 ip addr add 169.254.20.10 dev nodelocaldns
 ip addr add 10.96.0.10 dev nodelocaldns
 
 
  ethtool -i nodelocaldns
```

