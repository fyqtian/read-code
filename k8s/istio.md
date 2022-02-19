### istio

https://www.orchome.com/2027

安装

```
https://istio.io/latest/docs/setup/getting-started/#download
curl -L https://istio.io/downloadIstio | sh -

// namespace istio-system 需要手工创建
istioctl manifest generate > $HOME/generated-manifest.yaml

istioctl install --set profile=demo -y

istioctl profile dump demo

istioctl profile list

istioctl profile dump --config-path components.pilot demo

istioctl profile diff default demo



kubectl apply -f samples/addons
kubectl rollout status deployment/kiali -n istio-system

istioctl dashboard kiali

// 给namespace打标签 声明自动注入sidecar
kubectl label namespace default istio-injection=enabled

// istio下载包
kubectl apply -f samples/bookinfo/platform/kube/bookinfo.yaml

kubectl apply -f samples/bookinfo/networking/bookinfo-gateway.yaml


istioctl analyze



// 不启用自动注入，手工注入
istioctl kube-inject -f samples/bookinfo/platform/kube/bookinfo.yaml

```



## 删除

```
istioctl x uninstall --purge

kubectl delete namespace istio-system

istioctl manifest generate <your original installation options> | kubectl delete -f -

kubectl delete -f samples/addons
$ istioctl manifest generate --set profile=demo | kubectl delete --ignore-not-found=true -f -

kubectl label namespace default istio-injection-
```



### 查询

```
kubectl get virtualservices -o yaml

kubectl get destinationrules -o yaml
```

