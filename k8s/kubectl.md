### kubectl

主意版本

curl -LO https://storage.googleapis.com/kubernetes-release/release/v1.20.0/bin/linux/amd64/kubectl

Config 默认位置 $HOME/.kube/config



nginx ingress

kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v0.46.0/deploy/static/provider/baremetal/deploy.yaml



kubectl describe pod ingress-nginx-admission-create-kdrsd -n ingress-nginx



```csharp
kubectl run --image=nginx:alpine nginx-app --port=80
  
  kubectl delete pod nginx-app  -n default
```



转发端口

kubectl port-forward  --address 0.0.0.0  pod/nginx-86659f4cfb-29442 49999:8080