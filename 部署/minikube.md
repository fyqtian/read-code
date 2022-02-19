https://minikube.sigs.k8s.io/docs/start/

https://minikube.sigs.k8s.io/docs/start/



```
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube


useradd fyq
passwd fyq
usermod -aG docker fyq
su fyq
minikube start


minikube kubectl -- get pods -A


curl -L -O https://dl.k8s.io/v1.20.0/kubernetes-client-linux-amd64.tar.gz

tar  -zxvf   

minikube dashboard

kubectl proxy --port=8001 --address='172.26.0.108' --accept-hosts='^.*' &

http://centos:8001/api/v1/namespaces/kubernetes-dashboard/services/http:kubernetes-dashboard:/proxy/#/overview?namespace=default

//不需要minikube dashboard
kubectl proxy --address='0.0.0.0'  --accept-hosts='^*$' 

```





### kind

https://github.com/kubernetes-sigs/kind

```
curl -Lo ./kind "https://kind.sigs.k8s.io/dl/v0.11.1/kind-$(uname)-amd64"
chmod +x ./kind
mv ./kind /some-dir-in-your-PATH/kind


kind create cluster

helm repo add kubernetes-dashboard https://kubernetes.github.io/dashboard/
helm install dashboard kubernetes-dashboard/kubernetes-dashboard -n kubernetes-dashboard --create-namespace

kubectl proxy --address='0.0.0.0'  --accept-hosts='^*$' 

http://ubuntu:8001/api/v1/namespaces/kubernetes-dashboard/services/https:dashboard-kubernetes-dashboard:https/proxy/#/login




apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kubernetes-dashboard
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: admin-user
  namespace: kubernetes-dashboard
  
  
kubectl describe serviceaccount admin-user -n kubernetes-dashboard


```

