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
kubectl proxy --address='0.0.0.0'  --accept-hosts='^*$' &

```

