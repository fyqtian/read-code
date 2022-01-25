### IPVS



https://www.qikqiak.com/post/how-to-use-ipvs-in-kubernetes/



https://github.com/kubernetes/kubernetes/blob/master/pkg/proxy/ipvs/README.md



### vip实现

https://zhaohuabing.com/post/2019-11-27-vip/





### Ipvsadm 命令

https://www.jianshu.com/p/4a3496b8006a

https://www.jianshu.com/p/7cff00e253f4  ipset

```
ipvsadm -A -t 207.175.44.110:80 -s rr && ipvsadm -a -t 207.175.44.110 -r  172.26.0.109 -m
```





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