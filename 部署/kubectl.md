### kubectl

k8s内 ping不通 因为需要通过端口 进行转发

kubectl config view



 kubectl get pod  -A -o wide



只能监听127.0.0.1

kubectl port-forward rs/redis-master 6379:6379 





```
kubectl create service nodeport my-ns --tcp=5678:8080
```

Namespace

```fallback
kubectl create namespace foo
kubectl create rolebinding sam-edit \
    --clusterrole edit \
    --user sam \
    --namespace foo
    
    
    
$ kubectl create clusterrolebinding sam-view \
    --clusterrole view \
    --user sam

$ kubectl create clusterrolebinding sam-secret-reader \
    --clusterrole secret-reader \
    --user sam
```



```
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: secret-reader
rules:
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get", "watch", "list"]
  
 kubectl create -f clusterrole-secret-reader.yaml​
```



```text
kubectl run busybox --rm=true --image=busybox --restart=Never -it
```



### rbac

https://kubernetes.io/docs/reference/access-authn-authz/

https://kubernetes.io/docs/reference/access-authn-authz/rbac/#service-account-permissions





kubectl create -f nginx-deployment.yaml



kubectl get pods -l app=nginx

