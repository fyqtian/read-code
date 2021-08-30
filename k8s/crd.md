### crd

https://github.com/owenliang/k8s-client-go  

https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definitions/

https://blog.csdn.net/github_35614077/article/details/98749285 crd

https://kubernetes.feisky.xyz/ k8s文档

https://book.kubebuilder.io/quick-start.html

### kubebuilder

kubebuilder能帮我们节省大量工作，让开发CRD和adminsion webhook变得异常简单

```bash
git clone https://github.com/kubernetes-sigs/kubebuilder
cd kubebuilder
make build
cp bin/kubebuilder $GOPATH/bin
```



下载二进制：

```bash
os=$(go env GOOS)
arch=$(go env GOARCH)
 
# download kubebuilder and extract it to tmp
curl -sL https://go.kubebuilder.io/dl/2.0.0-beta.0/${os}/${arch} | tar -xz -C /tmp/
 
# move to a long-term location and put it on your path
# (you'll need to set the KUBEBUILDER_ASSETS env var if you put it somewhere else)
sudo mv /tmp/kubebuilder_2.0.0-beta.0_${os}_${arch} /usr/local/kubebuilder
export PATH=$PATH:/usr/local/kubebuilder/bin
```



还需要装下[kustomize](https://github.com/kubernetes-sigs/kustomize) 这可是个渲染yaml的神器，让helm颤抖。

```bash
go install sigs.k8s.io/kustomize/v3/cmd/kustomize
```



创建CRD

```bash
kubebuilder init --domain sealyun.com --license apache2 --owner "fanux"
kubebuilder create api --group infra --version v1 --kind VirtulMachine

make install # 安装CRD
make run # 启动controller
```



然后我们就可以看到创建的CRD了

```bash
# kubectl get crd
NAME                                           AGE
virtulmachines.infra.sealyun.com                  52m
```



来创建一个虚拟机：

```bash
# kubectl apply -f config/samples/
# kubectl get virtulmachines.infra.sealyun.com 
NAME                   AGE
virtulmachine-sample   49m
```

看一眼yaml文件

```bash
# cat config/samples/infra_v1_virtulmachine.yaml 
apiVersion: infra.sealyun.com/v1
kind: VirtulMachine
metadata:
  name: virtulmachine-sample
spec:
  # Add fields here
  foo: bar
```

这里仅仅是把yaml存到etcd里了，**我们controller监听到创建事件时啥事也没干**



把controller部署到集群中

Controller的逻辑其实是很简单的：监听CRD实例（以及关联的资源）的CRUD事件，然后执行相应的业务逻辑

```
make docker-build docker-push IMG=fanux/infra-controller
make deploy
```

