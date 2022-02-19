### Taint

```
kubectl get nodes --show-labels

# 新增
kubectl taint nodes  node key=val:NoSchedule
# 删
kubectl taint nodes node1 key:NoSchedule-
```



- NoSchedule 表示不能容忍taint的Pod不会被调度到这个节点，属于刚性的限制
- PreferNoSchedule 与上一条相同的效果，但是是柔性的限制，如果集群中没有其他更合适的Node，则会调度到这个节点
- NoExcute 上两条只是影响调度，这条还会对正在运行的Pod产生影响，如果在节点上增加一条 taint，那么如果已经运行的Pod没有设置对应的toleration，则会被立即驱逐