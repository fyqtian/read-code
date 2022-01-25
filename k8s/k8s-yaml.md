### 一些yaml

**所有namespace**

kubectl get pod -A





kubectl get --raw "/apis/metrics.k8s.io/v1beta1/nodes" | jq .

kubectl get --raw "/apis/metrics.k8s.io/v1beta1/pods" | jq .

#### 所有资源

kubectl api-resources





kubectl proxy --address='0.0.0.0'  --accept-hosts='^*$'

http://{kubectl.proxy}/api/v1/namespaces/default/pods/{pod.name}}:{pod.port}/proxy

http://centos:8001/api/v1/namespaces/default/pods/netdata-parent-765954d64-p4xzn:19999/proxy/#menu_netdata_submenu_web;after=-360;before=0;theme=slate

pod结构源码

https://github.com/kubernetes/kubernetes/blob/master/staging/src/k8s.io/api/core/v1/types.go



**pod 里调用apiserver**

TOKEN=$(kubectl describe secrets $(kubectl get secrets -n kube-system |grep admin |cut -f1 -d ' ') -n kube-system |grep -E '^token' |cut -f2 -d':'|tr -d '\t'|tr -d ' ')

原文链接：https://blog.csdn.net/weixin_44774358/article/details/93743364

APISERVER=$(kubectl config view |grep server|cut -f 2- -d ":" | tr -d " ")

curl -H "Authorization: Bearer $TOKEN" $APISERVER/api  --insecure



证书（没实验成功过）

curl https://192.168.31.61:6443/api/v1/nodes \ --cacert /etc/kubernetes/pki/ca.crt \ --cert /etc/kubernetes/pki/apiserver-kubelet-client.crt \ --key /etc/kubernetes/pki/apiserver-kubelet-client.key   

```
curl --cacert /var/run/secrets/kubernetes.io/serviceaccount/ca.crt -H "Authorization: Bearer $TOKEN" -s  
```



kubectl get --raw /api/v1/nodes









**clusterrolebinding**

kubectl create clusterrolebinding system:anonymous   --clusterrole=cluster-admin   --user=system:anonymous



**Pod**

```
apiVersion: v1
kind: Pod
metadata:
  name: two-containers
spec:
  restartPolicy: Never
  volumes:
  - name: shared-data
    hostPath:      
      path: /data
  containers:
  - name: nginx-container
    image: nginx
    volumeMounts:
    - name: shared-data
      mountPath: /usr/share/nginx/html
  - name: debian-container
    image: debian
    volumeMounts:
    - name: shared-data
      mountPath: /pod-data
    command: ["/bin/sh"]
    args: ["-c", "echo Hello from the debian container > /pod-data/index.html"]
    
    
apiVersion: v1
kind: Pod
metadata:
  name: javaweb-2
spec:
  initContainers:
  - image: geektime/sample:v2
    name: war
    command: ["cp", "/sample.war", "/app"]
    volumeMounts:
    - mountPath: /app
      name: app-volume
  containers:
  - image: geektime/tomcat:7.0
    name: tomcat
    command: ["sh","-c","/root/apache-tomcat-7.0.42-v2/bin/start.sh"]
    volumeMounts:
    - mountPath: /root/apache-tomcat-7.0.42-v2/webapps
      name: app-volume
    ports:
    - containerPort: 8080
      hostPort: 8001 
  volumes:
  - name: app-volume
    emptyDir: {}





pod选择node
apiVersion: v1
kind: Pod
...
spec:
 nodeSelector:
   disktype: ssd
    
    
    
HostAliases：定义了 Pod 的 hosts 文件（比如 /etc/hosts）里的内容，用法如下：    
piVersion: v1
kind: Pod
...
spec:
  hostAliases:
  - ip: "10.1.2.3"
    hostnames:
    - "foo.remote"
    - "bar.remote"


共享命名空间 shareProcessNamespace: true 

apiVersion: v1
kind: Pod
metadata:
  name: nginx
  namespace: default
spec:
  shareProcessNamespace: true
  containers:
  - name: nginx
    image: nginx
  - name: shell
    image: busybox
    stdin: true
    tty: true

kubectl create -f nginx.yaml

kubectl attach -it nginx -c shell       -c 指定那个容器

kubectl delete pod nginx-2 -n namespace





容器要共享宿主机的 Namespace，也一定是 Pod 级别的定义

apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  hostNetwork: true
  hostIPC: true
  hostPID: true
  containers:
  - name: nginx
    image: nginx
  - name: shell
    image: busybox
    stdin: true
    tty: true
    

Lifecycle 字段。它定义的是 Container Lifecycle Hooks。顾名思义，Container Lifecycle Hooks 的作用，是在容器状态发生变化时触发一系列“钩子”。

postStart 定义的操作，虽然是在 Docker 容器 ENTRYPOINT 执行之后，但它并不严格保证顺序。也就是说，在 postStart 启动时，ENTRYPOINT 有可能还没有结束。

preStop 发生的时机，则是容器被杀死之前（比如，收到了 SIGKILL 信号）。而需要明确的是，preStop 操作的执行，是同步的。所以，它会阻塞当前的容器杀死流程，直到这个 Hook 定义操作完成之后，才允许容器被杀死，这跟 postStart 不一样。

apiVersion: v1
kind: Pod
metadata:
  name: lifecycle-demo
spec:
  containers:
  - name: lifecycle-demo-container
    image: nginx
    lifecycle:
      postStart:
        exec:
          command: ["/bin/sh", "-c", "echo Hello from the postStart handler > /usr/share/message"]
      preStop:
        exec:
          command: ["/usr/sbin/nginx","-s","quit"]
```



### 目前4种projected volume

- secret
- configMap
- Downward Api
- ServiceAccountToken



**secret**帮你把pod想要访问的加密数据 存放到etcd，然后在pod用挂载的volume访问

**Secret 可以被 ServerAccount 关联**

Secret 可以存储 docker register 的鉴权信息，用在 ImagePullSecret 参数中，用于拉取私有仓库的镜像

Secret 支持 Base64 加密

Secret 分为 kubernetes.io/service-account-token、kubernetes.io/dockerconfigjson、Opaque 三种类型


通过**Volume挂载到容器内部时，当该Secret的值发生变化时，容器内部具备自动更新的能力，但是通过环境变量设置到容器内部该值不具备自动更新的能力**。所以一般推荐使用Volume挂载的方式使用Secret。



### 真正的加密

你需要在 Kubernetes 中开启 Secret 的加密插件，增强数据的安全性。关于开启 Secret 加密插件的内容，我会在后续专门讲解 Secret 的时候，再做进一步说明。

```
生成base64
echo -n 'admin' | base64
 
创建docker registry的凭证
kubectl create secret docker-registry myregistry --docker-server=DOCKER_SERVER --docker-username=DOCKER_USER --docker-password=DOCKER_PASSWORD --docker-email=DOCKER_EMAIL

kubectl create secret generic user --from-literal=username=admin

从文件创建
kubectl create secret generic user --from-file=./username.txt

创建secret

apiVersion: v1
kind: Secret
metadata: 
  name: mysecret
type: Opaque
data:  
user: YWRtaW4=  
pass: MWYyZDFlMmU2N2Rm

          
拉私有仓库image

apiVersion: v1
kind: Pod
metadata:
  name: foo
spec:
  containers:
  - name: foo
    image: 192.168.1.100:5000/test:v1
  imagePullSecrets:
  - name: myregistry


pod使用secret 文件方式

apiVersion: v1
kind: Pod
metadata:
  name: test-projected-volume 
spec:
  containers:
  - name: test-secret-volume
    image: busybox
    args:
    - sleep
    - "86400"
    volumeMounts:
    - name: mysql-cred
      mountPath: "/projected-volume"
      readOnly: true
  volumes:
  - name: mysql-cred
    projected:
      sources:
      - secret:
          name: user
      - secret:
          name: pass
          
环境变量方式

apiVersion: v1
kind: Pod
metadata:
  name: pod-secret-env
spec:
  containers:
  - name: myapp
	image: busybox
	args:
    - sleep
    - "86400"
	env:
    - name: SECRET_USERNAME
      valueFrom:
        secretKeyRef:
          name: mysecret
          key: user
    - name: SECRET_PASSWORD
      valueFrom:
        secretKeyRef:
          name: mysecret
          key: pass
  restartPolicy: Never




```



### Downward API

它的作用是：**让 Pod 里的容器能够直接获取到这个 Pod API 对象本身的信息。**

当前 Pod 的 Labels 字段的值，就会被 Kubernetes 自动挂载成为容器里的 /etc/podinfo/labels 文件。



```
apiVersion: v1
kind: Pod
metadata:
  name: test-downwardapi-volume
  labels:
    zone: us-est-coast
    cluster: test-cluster1
    rack: rack-22
spec:
  containers:
    - name: client-container
      image: k8s.gcr.io/busybox
      command: ["sh", "-c"]
      args:
      - while true; do
          if [[ -e /etc/podinfo/labels ]]; then
            echo -en '\n\n'; cat /etc/podinfo/labels; fi;
          sleep 5;
        done;
      volumeMounts:
        - name: podinfo
          mountPath: /etc/podinfo
          readOnly: false
  volumes:
    - name: podinfo
      projected:
        sources:
        - downwardAPI:
            items:
              - path: "labels"
                fieldRef:
                  fieldPath: metadata.labels
```



支持的字段

Downward API 能够获取到的信息，**一定是 Pod 里的容器进程启动之前就能够确定下来的信息**。

```
1. 使用 fieldRef 可以声明使用:
spec.nodeName - 宿主机名字
status.hostIP - 宿主机 IP
metadata.name - Pod 的名字
metadata.namespace - Pod 的 Namespace
status.podIP - Pod 的 IP
spec.serviceAccountName - Pod 的 Service Account 的名字
metadata.uid - Pod 的 UID
metadata.labels['<KEY>'] - 指定 <KEY> 的 Label 值
metadata.annotations['<KEY>'] - 指定 <KEY> 的 Annotation 值
metadata.labels - Pod 的所有 Label
metadata.annotations - Pod 的所有 Annotation
 
2. 使用 resourceFieldRef 可以声明使用:
容器的 CPU limit
容器的 CPU request
容器的 memory limit
容器的 memory request
```



### serviceAccount serviceToken

Kubernetes 项目的 Projected Volume 其实只有三种，因为第四种 ServiceAccountToken，只是一种特殊的 Secret 而已。

为了方便使用，Kubernetes 已经为你提供了一个的默认“服务账户”（default Service Account）。并且，任何一个运行在 Kubernetes 里的 Pod，都可以直接使用这个默认的 Service Account，而无需显示地声明挂载它。

**kubectl describe pod busybox 可以看到mounts挂载了一个serviceToken**



 **固定目录下：/var/run/secrets/kubernetes.io/serviceaccount** 



**这种把 Kubernetes 客户端以容器的方式运行在集群里，然后使用 default Service Account 自动授权的方式，被称作“InClusterConfig”，也是我最推荐的进行 Kubernetes API 编程的授权方式。**







### 健康检查

**如果直接设置pod，pod不会从node迁移，只有用deployment才会迁移**

kubectl port-forward  --address 0.0.0.0  pod/nginx-86659f4cfb-29442 49999:8080

```
apiVersion: v1
kind: Pod
metadata:
  labels:
    test: liveness
  name: test-liveness-exec
spec:
  containers:
  - name: liveness
    image: busybox
    args:
    - /bin/sh
    - -c
    - touch /tmp/healthy; sleep 30; rm -rf /tmp/healthy; sleep 600
    livenessProbe:
      exec:
        command:
        - cat
        - /tmp/healthy
      initialDelaySeconds: 5
      periodSeconds: 5
```



### 预先填充pod模版

需要启用api settings.k8s.io/v1alpha1=true

minikube  --extra-config=apiserver.runtime-config=settings.k8s.io/v1alpha1=true

通过select标签选择要填充的pod

```
apiVersion: settings.k8s.io/v1alpha1
kind: PodPreset
metadata:
  name: allow-database
spec:
  selector:
    matchLabels:
      role: frontend
  env:
    - name: DB_PORT
      value: "6379"
  volumeMounts:
    - mountPath: /cache
      name: cache-volume
  volumes:
    - name: cache-volume
      emptyDir: {}
      
apiVersion: v1
kind: Pod
metadata:
  name: website
  labels:
    app: website
    role: frontend
spec:
  containers:
    - name: website
      image: nginx
      ports:
        - containerPort: 80

```





### Deployment

### 实际状态来源于心跳上报，期望状态来与用户提交的yaml，状态保存在etcd

template 代表模版

kubectl create -f nginx-deployment.yaml **--record**

### kubectl scale deployment nginx-deployment --replicas=4



实时查看 Deployment 对象的状态变化

### kubectl rollout status deployment/nginx-deployment

查看deployment对应的replica set

kubectl get rs

kubectl edit deployment/nginx-deployment (**kubectl edit 并不神秘，它不过是把 API 对象的内容下载到了本地文件，让你修改完成后再提交上去。**)



kubectl **set image** deployment/nginx-deployment nginx=nginx:1.91

### **回滚**

kubectl rollout undo deployment/nginx-deployment --to-revision=2

kubectl rollout history deployment/nginx-deployment

kubectl rollout history deployment/nginx-deployment --revision=2



// 暂停更新

kubectl rollout pause deployment/nginx-deployment

// 回复

kubectl rollout resume deploy/nginx-deployment

可以在 Deployment 配置文件中通过 **revisionHistoryLimit** 属性增加 revision 数量。

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 1
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.7.9
        ports:
        - containerPort: 80



滚动策略

maxSurge 指定的是除了 DESIRED 数量之外，在一次“滚动”中，Deployment 控制器还可以创建多少个新 Pod；
maxUnavailable 指的是，在一次“滚动”中，Deployment 控制器可以删除多少个旧 Pod。
maxUnavailable=50%，指的是我们最多可以一次删除“50%*DESIRED 数量”个 Pod。

apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
...
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1

```





### StatefulSet

**第一种方式，是以 Service 的 VIP（Virtual IP，即：虚拟 IP）方式**。

当我访问 10.0.23.1 这个 Service 的 IP 地址时，10.0.23.1 其实就是一个 VIP，它会把请求转发到该 Service 所代理的某一个 Pod 上。



**第二种就是以 Service 的 DNS 方式**

只要我访问“my-svc.my-namespace.svc.cluster.local”这条 DNS 记录，就可以访问到名叫 my-svc 的 Service 所代理的某一个 Pod。



**Headless Service**。这种情况下，访问“my-svc.my-namespace.svc.cluster.local”解析到的，直接就是 my-svc 代理的某一个 Pod 的 IP 地址。**可以看到，这里的区别在于，Headless Service 不需要分配一个 VIP，而是可以直接以 DNS 记录的方式解析出被代理 Pod 的 IP 地址。**



重点**clusterIP: None**

**<pod-name>.<svc-name>.<namespace>.svc.cluster.local**

kubectl create service clusterip my-cs --tcp=9999:80

**headless**

```
apiVersion: v1
kind: Service
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  ports:
  - port: 80
    name: web
  clusterIP: None
  selector:
    app: nginx
```



### busybox

1.28.3 telnet 有问题

kubectl run -i --tty --image busybox:1.28.3 dns-test --restart=Never --rm /bin/sh

​	kubectl run -i --tty --image docker docker-test --restart=Never --rm /bin/sh



kubectl run -i --tty --image curlimages/curl curl-test --restart=Never --rm /bin/sh



kubectl run -i --tty --image widdpim/mysql-client mysql-client-test --restart=Never --rm /bin/sh

### Stateful

和deployment的差别多了 **serviceName: "nginx"**

这个字段的作用，就是告诉 StatefulSet 控制器，在执行控制循环（Control Loop）的时候**，请使用 nginx 这个 Headless Service 来保证 Pod 的“可解析身份”**。

kubectl get pods -w -l app=nginx

```
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  serviceName: "nginx"
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.9.1
        ports:
        - containerPort: 80
          name: web

声明+mount
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  serviceName: "nginx"
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.9.1
        ports:
        - containerPort: 80
          name: web
        volumeMounts:
        - name: www
          mountPath: /usr/share/nginx/html
  volumeClaimTemplates:
  - metadata:
      name: www
    spec:
      accessModes:
      - ReadWriteOnce
      resources:
        requests:
          storage: 1Gi
```



### PVC

**persistent-volumes access-modes** https://kubernetes.io/docs/concepts/storage/persistent-volumes/#access-modes

```
声明pvc

kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: pv-claim
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
      
声明pod

apiVersion: v1
kind: Pod
metadata:
  name: pv-pod
spec:
  containers:
    - name: pv-container
      image: nginx
      ports:
        - containerPort: 80
          name: "http-server"
      volumeMounts:
        - mountPath: "/usr/share/nginx/html"
          name: pv-storage
  volumes:
    - name: pv-storage
      persistentVolumeClaim:
        claimName: pv-claim
        

声明PV

kind: PersistentVolume
apiVersion: v1
metadata:
  name: pv-volume
  labels:
    type: local
spec:
  capacity:
    storage: 10Gi
  rbd:
    monitors:
    - '10.16.154.78:6789'
    - '10.16.154.82:6789'
    - '10.16.154.83:6789'
    pool: kube
    image: foo
    fsType: ext4
    readOnly: true
    user: admin
    keyring: /etc/ceph/keyring
    imageformat: "2"
    imagefeatures: "layering"

```



### DaemonSet

```
apiVersion: apps/v1
kind: DaemonSet
metadata: fluentd-elasticsearch
namespace: kube-system
labels:
	k8s-app: fluentd-logging
spec:
	selector:
		matchLabels:
			name: fluentd-elasticsearch
	template:
		metadata:
			labels:
				name: fluentd-elasticsearch
			spec:
      	tolerations:
      	- key: node-role.kubernetes.io/master
      		effect: NoSchedule
      	containers:
      	- name: fluentd-elasticsearch
      		image: k8s.gcr.io/fluentd-elasticsearch:1.20
      		resources:
      			limits:
      				memory: 200Mi
      			requests:
      				cpu: 100m
      				memory: 200Mi
      		volumeMounts:
      		- name: varlog
      			mountPath: /var/log
      		- name: varlibdockercontainers
      			mountPath: /var/lib/docker/containers
      			readOnly: true
      	terminationGracePeriodSeconds: 30
      	volumes:
        - name: varlog
        	hostPath:
        		path: /var/log
        - name: varlibdockercontainers
        	hostPath:
        		path: /var/lib/docker/containers






```







### Job

kubectl describe jobs/pi

如果你**定义的 restartPolicy=OnFailure，那么离线作业失败后，Job Controller 就不会去尝试创建新的 Pod。但是，它会不断地尝试重启 Pod 里的容器**



**activeDeadlineSeconds** 字段可以设置最长运行时间

```
apiVersion: batch/v1
kind: Job
metadata:
  name: pi
spec:
  template:
    spec:
      containers:
      - name: pi
        image: resouer/ubuntu-bc 
        command: ["sh", "-c", "echo 'scale=10000; 4*a(1)' | bc -l "]
      restartPolicy: Never
  backoffLimit: 4
  activeDeadlineSeconds: 100

```

1. **spec.parallelism**，它定义的是一个 Job 在任意时间最多可以启动多少个 Pod 同时运行；
2. **spec.completions**，它定义的是 Job 至少要完成的 Pod 数目，即 Job 的最小完成数。

```
apiVersion: batch/v1
kind: Job
metadata:
  name: pi
spec:
  parallelism: 2
  completions: 4
  template:
    spec:
      containers:
      - name: pi
        image: resouer/ubuntu-bc
        command: ["sh", "-c", "echo 'scale=5000; 4*a(1)' | bc -l "]
      restartPolicy: Never
  backoffLimit: 4
```





**CronJob**

需要注意的是，由于定时任务的特殊性，**很可能某个 Job 还没有执行完，另外一个新 Job 就产生了**。这时候，你可以通过 spec.concurrencyPolicy 字段来定义具体的处理策略。比如：

1. concurrencyPolicy=Allow，这也是默认情况，这意味着这些 Job 可以同时存在；
2. concurrencyPolicy=Forbid，这意味着不会创建新的 Pod，该创建周期被跳过；
3. concurrencyPolicy=Replace，这意味着新产生的 Job 会替换旧的、没有执行完的 Job。

而如果某**一次 Job 创建失败**，这次创建就会被**标记为“miss”**。当在指定的时间窗口内，miss 的数目达到 100 时，那么 CronJob 会停止再创建这个 Job。

这个时间窗口，**可以由 spec.startingDeadlineSeconds 字段指**定。比如 startingDeadlineSeconds=200，意味着在过去 200 s 里，如果 miss 的数目达到了 100 次，那么这个 Job 就不会被创建执行了。

```
apiVersion: batch/v1beta1
kind: CronJob
metadata:
  name: hello
spec:
  schedule: "*/1 * * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: hello
            image: busybox
            args:
            - /bin/sh
            - -c
            - date; echo Hello from the Kubernetes cluster
          restartPolicy: OnFailure
```



### 声明式

kubectl apply -f nginx.yaml

你可以简单地理解为，kubectl replace 的执行过程，是使用新的 YAML 文件中的 API 对象，**替换原有的 API 对象**；而 kubectl apply，则是执行了一个**对原有 API 对象的 PATCH 操作**。

类似地，kubectl set image 和 kubectl edit 也**是对已有 API 对象的修改**



这意味着 kube-apiserver 在响应命令式请求（比如，kubectl replace）的时候，**一次只能处理一个写请求**，否则会有产生冲突的可能。而对于声明式请求（比如，kubectl apply），**一次能处理多个写操作，并且具备 Merge 能力**。



实际上，**Istio 项目使用的，是 Kubernetes 中的一个非常重要的功能，叫作 Dynamic Admission Control。**

而这个“初始化”操作的实现，**借助的是一个叫作 Admission 的功能**。它其实是 Kubernetes 项目里一组被称为 Admission Controller 的代码，**可以选择性地被编译进 APIServer 中**，在 API 对象创建之后会被立刻调用到。



**首先，Kubernetes 会匹配 API 对象的组。**

需要明确的是，对于 Kubernetes **里的核心 API 对象，比如：Pod、Node 等**，**是不需要 Group 的**（即：它们 Group 是“”）。所以，对于这些 API 对象来说，Kubernetes 会直接在 /api 这个层级进行下一步的匹配过程。

**然后，Kubernetes 会进一步匹配到 API 对象的版本号。**

apiVersion: batch/v1beta1

对于 CronJob 这个 API 对象来说，Kubernetes 在 batch 这个 Group 下，匹配到的版本号就是 v2alpha1。



### Admission Controller

https://kubernetes.io/docs/reference/access-authn-authz/extensible-admission-controllers/#configure-initializers-on-the-fly





### **CRD  Custom Resource Definition**

https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definitions/

https://cloud.redhat.com/blog/kubernetes-deep-dive-code-generation-customresources

允许用户在 Kubernetes 中添加一个跟 Pod、Node 类似的、新的 API 资源类型，**即：自定义 API 资源**。

// example

可以看到，在这个 CRD 中，**我指定了“`group: samplecrd.k8s.io`”“`version: v1`”这样的 API 信息**，也指定了这个 CR 的**资源类型叫作 Network**，复数（plural）是 networks。

然后，我还声明了它的 **scope 是 Namespaced**，即：我们定义的这个 Network 是一个属于 Namespace 的对象，类似于 Pod。

```
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  name: networks.samplecrd.k8s.io
spec:
  group: samplecrd.k8s.io
  version: v1
  names:
    kind: Network
    plural: networks
  scope: Namespaced
  

apiVersion: samplecrd.k8s.io/v1
kind: Network
metadata:
  name: example-network
spec:
  cidr: "192.168.0.0/16"
  gateway: "192.168.0.1"
  
  
  
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  # 名称必须与下面的spec字段匹配，格式为: <plural>.<group>
  name: crontabs.stable.example.com
spec:
  # 用于REST API的组名称: /apis/<group>/<version>
  group: stable.example.com
  # 此CustomResourceDefinition支持的版本列表
  versions:
    - name: v1
      # 每个版本都可以通过服务标志启用/禁用。
      served: true
      # 必须将一个且只有一个版本标记为存储版本。
      storage: true
  # 指定crd资源作用范围在命名空间或集群
  scope: Namespaced
  names:
    # URL中使用的复数名称: /apis/<group>/<version>/<plural>
    plural: crontabs
    # 在CLI(shell界面输入的参数)上用作别名并用于显示的单数名称
    singular: crontab
    # kind字段使用驼峰命名规则. 资源清单使用如此
    kind: CronTab
    # 短名称允许短字符串匹配CLI上的资源，意识就是能通过kubectl 在查看资源的时候使用该资源的简名称来获取。
    shortNames:
    - ct
  

apiVersion: "stable.example.com/v1"
kind: CronTab
metadata:
  name: my-new-cron-object
spec:
  cronSpec: "* * * * */5"
  image: my-awesome-cron-image

   
kubectl create -f xx.yaml
kubectl get crontab
```



#### TODO











### RBAC（Role-Based Access Control）

1. Role：角色，它其实是一组规则，定义了一组对 Kubernetes API 对象的操作权限。
2. Subject：被作用者，既可以是“人”，也可以是“机器”，也可以使你在 Kubernetes 里定义的“用户”。
3. RoleBinding：定义了“被作用者”和“角色”的绑定关系。
4. 

kubectl create serviceaccount username



kubectl get role

kubectl get cluster role

kubectl get clusterrole/admin -o yaml



全部权限 ["get", "list", "watch", "create", "update", "patch", "delete"]

```
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  namespace: mynamespace
  name: example-role
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "watch", "list"]
 
权限细化 
rules:
- apiGroups: [""]
  resources: ["configmaps"]
  resourceNames: ["my-config"]
  verbs: ["get"]
 
 
  
  
段，RoleBinding 对象就可以直接通过名字，来引用我们前面定义的 Role 对象（example-role），从而定义了“被作用者（Subject）”和“角色（Role）”之间的绑定关系。  
  
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: example-rolebinding
  namespace: mynamespace
subjects:
- kind: User
  name: example-user
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: example-role
  apiGroup: rbac.authorization.k8s.io

```





对于非 Namespaced（Non-namespaced）对象（比如：Node），或者，某一个 Role 想要作用于所有的 Namespace 的时候

我们就必须要使用 ClusterRole 和 ClusterRoleBinding 这两个组合了。这两个 API 对象的用法跟 Role 和 RoleBinding 完全一样。只不过，**它们的定义里，没有了 Namespace 字段**，

```
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: example-clusterrole
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "watch", "list"]
  
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: example-clusterrolebinding
subjects:
- kind: User
  name: example-user
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: example-clusterrole
  apiGroup: rbac.authorization.k8s.io
```





### 用户

```
apiVersion: v1
kind: ServiceAccount
metadata:
  namespace: mynamespace
  name: example-sa
  
  
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: example-rolebinding
  namespace: mynamespace
subjects:
- kind: ServiceAccount
  name: example-user
  namespace: mynamespace
roleRef:
  kind: Role
  name: example-role
  apiGroup: rbac.authorization.k8s.io
```



挂载用户的token

/var/run/secrets/kubernetes.io/serviceaccount

**如果一个 Pod 没有声明 serviceAccountName，Kubernetes 会自动在它的 Namespace 下创建一个名叫 default 的默认 ServiceAccount，然后分配给这个 Pod。**

```
apiVersion: v1
kind: Pod
metadata:
  namespace: mynamespace
  name: sa-token-test
spec:
  containers:
  - name: nginx
    image: nginx:1.7.9
  serviceAccountName: example-sa

```

实际上，一个 ServiceAccount，在 Kubernetes 里对应的“用户”的名字是：

system:serviceaccount:<ServiceAccount 名字 >

**而它对应的内置“用户组”的名字，就是：**

system:serviceaccounts:<Namespace 名字 >

**group**

这就意味着这个 Role 的权限规则，作用于 mynamespace 里的所有 ServiceAccount。这就用到了“用户组”的概念。

```
subjects:
- kind: Group
  name: system:serviceaccounts:mynamespace
  apiGroup: rbac.authorization.k8s.io
```

**这个 Role 的权限规则，作用于整个系统里的所有 ServiceAccount。**

```
subjects:
- kind: Group
  name: system:serviceaccounts
  apiGroup: rbac.authorization.k8s.io

```



#### User Accounts 与 Service Accounts

Kubernetes区分普通帐户（user accounts）和服务帐户（service accounts）的原因：

```
-普通帐户是针对（人）用户的，服务账户针对Pod进程。

-普通帐户是全局性。在集群所有namespaces中，名称具有惟一性。

-通常，群集的普通帐户可以与企业数据库同步，新的普通帐户创建需要特殊权限。服务账户创建目的是更轻量化，允许集群用户为特定任务创建服务账户。

-普通帐户和服务账户的审核注意事项不同。

-对于复杂系统的配置包，可以包括对该系统的各种组件的服务账户的定义。

```

Service account是为了方便Pod里面的进程调用Kubernetes API或其他外部服务而设计的。它与User account不同：





### 持久化

**pv**

```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nfs
spec:
  storageClassName: manual
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteMany
  nfs:
    server: 10.244.1.4
    path: "/"
```

**pvc**

```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nfs
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: manual
  resources:
    requests:
      storage: 1Gi
```

**pod**

```
apiVersion: v1
kind: Pod
metadata:
  labels:
    role: web-frontend
spec:
  containers:
  - name: web
    image: nginx
    ports:
      - name: web
        containerPort: 80
    volumeMounts:
        - name: nfs
          mountPath: "/usr/share/nginx/html"
  volumes:
  - name: nfs
    persistentVolumeClaim:
      claimName: nfs
```



动态存储 **StorageClass**

```
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: block-service
provisioner: kubernetes.io/gce-pd
parameters:
  type: pd-ssd
```





- 第一个条件，当然是 PV 和 PVC 的 **spec 字段**。比如，PV 的存储（storage）大小，就必须满足 PVC 的要求。
- 而第二个条件，则是 PV 和 PVC 的 **storageClassName 字段必须一样**



todo







### 网络

在 Linux 中，能够起到虚拟交换机作用的网络设备，**是网桥（Bridge）**。它是一个工作在**数据链路层**（Data Link）的设备，主要功能是**根据 MAC 地址学习来将数据包转发到网桥的不同端口（Port）上**。



容器跨网络

1. VXLAN；
2. host-gw；
3. UDP。



TUN 设备是一种工作在三层（Network Layer）的虚拟网络设备。TUN 设备的功能非常简单，即：**在操作系统内核和用户应用程序之间传递 IP 包。** 

容器的流量通过tun走到flannel，flannel再发送出去（udp方案）

**flanneld 又是如何知道这个 IP 地址对应的容器，是运行在 Node 2 上的呢？** 通过etcd注册的





VXLAN，即 Virtual Extensible LAN（虚拟可扩展局域网），是 Linux 内核本身就支持的一种网络虚似化技术。所以说**，VXLAN 可以完全在内核态实现上述封装和解封装的工作**，从而通过与前面相似的“隧道”机制，构建出覆盖网络（Overlay Network）



VXLAN 的覆盖网络的设计思想是：在现有的三层网络之上，“覆盖”一层虚拟的、由内核 VXLAN 模块负责维护的二层网络，使得连接在这个 VXLAN 二层网络上的“主机”（虚拟机或者容器都可以）之间，**可以像在同一个局域网（LAN）里那样自由通信。当然，实际上，这些“主机”可能分布在不同的宿主机上，甚至是分布在不同的物理机房里。**

而为了能够在二层网络上打通“隧道”，VXLAN 会在宿主机上设置一个特殊的网络设备作为“隧道”的两端。**这个设备就叫作 VTEP**，即：**VXLAN Tunnel End Point（虚拟隧道端点）**



 bridge fdb 



**Flannel host-gw 模式必须要求集群宿主机之间是二层连通的。**，通过路由直接到对方主机，不需要封装

calico bgp





### NetworkPolicy

 **podSelector** 字段。它的作用，就是定义这个 NetworkPolicy 的限制范围，比如：当前 Namespace 里携带了 role=db 标签的 Pod

如果**podSelector 字段留空**：那么这个 NetworkPolicy **就会作用于当前 Namespace 下的所有 Pod。**

**NetworkPolicy 定义的规则，其实就是“白名单”。**





这个 NetworkPolicy 对象，指定的隔离规则如下所示：

1. 该隔离规则只对 **default Namespace 下的，携带了 role=db 标签的 Pod 有效。**限制的请求类型包括 ingress（流入）和 egress（流出）。
2. Kubernetes 会拒绝任何访问被隔离 Pod 的请求，除非这个请求来自于以下“白名单”里的对象，并且访问的是被隔离 Pod 的 6379 端口。这些“白名单”对象包括：
3. default Namespace 里的，携带了 role=fronted 标签的 Pod；
4. 任何 Namespace 里的、携带了 project=myproject 标签的 Pod；
5. 任何源地址属于 172.17.0.0/16 网段，且不属于 172.17.1.0/24 网段的请求。
6. Kubernetes 会拒绝被隔离 Pod 对外发起任何请求，除非请求的目的地址属于 10.0.0.0/24 网段，并且访问的是该网段地址的 5978 端口。

```
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: test-network-policy
  namespace: default
spec:
  podSelector:
    matchLabels:
      role: db
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - ipBlock:
        cidr: 172.17.0.0/16
        except:
        - 172.17.1.0/24
    - namespaceSelector:
        matchLabels:
          project: myproject
    - podSelector:
        matchLabels:
          role: frontend
    ports:
    - protocol: TCP
      port: 6379
  egress:
  - to:
    - ipBlock:
        cidr: 10.0.0.0/24
    ports:
    - protocol: TCP
      port: 5978
```



在 Kubernetes 生态里，目前已经实现了 NetworkPolicy 的网络插件包括 Calico、Weave 和 kube-router 等多个项目，**但是并不包括 Flannel 项目。**



**可以看到，Kubernetes 网络插件对 Pod 进行隔离，其实是靠在宿主机上生成 NetworkPolicy 对应的 iptable 规则来实现的。**







### Service

**Service 是由 kube-proxy 组件，加上 iptables 来共同实现的。**

selector 选择pod

kubectl get endpoints	



1. KUBE-SERVICES 或者 KUBE-NODEPORTS 规则对应的 Service 的入口链，这个规则应该与 VIP 和 Service 端口一一对应；
2. KUBE-SEP-(hash) 规则对应的 DNAT 链，这些规则应该与 Endpoints 一一对应；
3. KUBE-SVC-(hash) 规则对应的负载均衡链，这些规则的数目应该与 Endpoints 数目一致；
4. 如果是 NodePort 模式的话，还有 POSTROUTING 处的 SNAT 链。

```
apiVersion: v1
kind: Service
metadata:
  name: hostnames
spec:
  selector:
    app: hostnames
  ports:
  - name: default
    protocol: TCP
    port: 80
    targetPort: 9376
```

**Headless**

```
apiVersion: v1
kind: Service
metadata:
  name: default-subdomain
spec:
  selector:
    name: busybox
  clusterIP: None
  ports:
  - name: foo
    port: 1234
    targetPort: 1234
---
apiVersion: v1
kind: Pod
metadata:
  name: busybox1
  labels:
    name: busybox
spec:
  hostname: busybox-1
  subdomain: default-subdomain
  containers:
  - image: busybox
    command:
      - sleep
      - "3600"
    name: busybox
```



**nodeport**

```
apiVersion: v1
kind: Service
metadata:
  name: my-nginx
  labels:
    run: my-nginx
spec:
  type: NodePort
  ports:
  - nodePort: 8080
    targetPort: 80
    protocol: TCP
    name: http
  - nodePort: 443
    protocol: TCP
    name: https
  selector:
    run: my-nginx

```

如果你不显式地声明 nodePort 字段，Kubernetes 就会为你分配随机的可用端口来设置代理。**这个端口的范围默认是 30000-32767，你可以通过 kube-apiserver 的–service-node-port-range 参数来修改它。**

NodePort 模式也就非常容易理解了。显然，kube-proxy 要做的，就是在每台宿主机上生成这样一条 iptables 规则：

```
iptables -A KUBE-NODEPORTS -p tcp -m comment --comment "default/my-nginx: nodePort" -m tcp --dport 8080 -j KUBE-SVC-67RL4FN6JRUPOJYM
```





**Loadbalance**

```
kind: Service
apiVersion: v1
metadata:
  name: example-service
spec:
  ports:
  - port: 8765
    targetPort: 9376
  selector:
    app: example
  type: LoadBalancer
```



**ExternalName**

是 Kubernetes 在 1.7 之后支持的一个新特性，叫作 ExternalName。举个例子：

，我指定了一个 externalName=my.database.example.com 的字段。而且你应该会注意到，这个 YAML 文件里不需要指定 selector。

这时候，当你通过 Service 的 DNS 名字访问它的时候，比如访问：my-service.default.svc.cluster.local。那么，Kubernetes 为你返回的就是`my.database.example.com`。所以说，ExternalName 类型的 Service，**其实是在 kube-dns 里为你添加了一条 CNAME 记录**。这时，访问 my-service.default.svc.cluster.local 就和访问 my.database.example.com 这个域名是一个效果

```
kind: Service
apiVersion: v1
metadata:
  name: my-service
spec:
  type: ExternalName
  externalName: my.database.example.com


kind: Service
apiVersion: v1
metadata:
  name: my-service
spec:
  selector:
    app: MyApp
  ports:
  - name: http
    protocol: TCP
    port: 80
    targetPort: 9376
  externalIPs:
  - 80.11.12.10
```





### Ingress

nginx-ingress 

[Installation Guide - NGINX Ingress Controller (kubernetes.github.io)](https://kubernetes.github.io/ingress-nginx/deploy/#minikube)

**IngressRule**。



IngressRule 的 Key，就叫做：host。**它必须是一个标准的域名格式（Fully Qualified Domain Name）的字符串**，而不能是 IP 地址。

而接下来 IngressRule 规则的定义，则依赖于 path 字段。你可以简单地理解为，这里的每一个 path 都对应一个后端 Service。所以在我们的例子里，我定义了两个 path，它们分别对应 coffee 和 tea 这两个 Deployment 的 Service（即：coffee-svc 和 tea-svc）。

不难看到，所谓 Ingress 对象，其实就是 Kubernetes 项目对“反向代理”的一种抽象。



**nginx-ingress**

kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/mandatory.yaml

```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: cafe-ingress
spec:
  tls:
  - hosts:
    - cafe.example.com
    secretName: cafe-secret
  rules:
  - host: cafe.example.com
    http:
      paths:
      - path: /tea
        backend:
          serviceName: tea-svc
          servicePort: 80
      - path: /coffee
        backend:
          serviceName: coffee-svc
          servicePort: 80
```





### **资源控制**

cpu 属于**可压缩资源**，当可压缩资源不足时，Pod 只会“饥饿”，但不会退出。

内存属于**不可压缩资源**，当**不可压缩资源不足时**，Pod 就会因为 **OOM（Out-Of-Memory）被内核杀掉**。

而由于 **Pod 可以由多个 Container 组成**，所以 CPU 和内存资源的限额，**是要配置在每个 Container 的定义上的**。这样，Pod 整体的资源配置，**就由这些 Container 的配置值累加得到**。

所谓 500m，指的就是 500 millicpu，也就是 0.5 个 CPU 的意思。

内存资源来说，它的单位自然就是 bytes。Kubernetes 支持你使用 Ei、Pi、Ti、Gi、Mi、Ki（或者 E、P、T、G、M、K）的方式来作为 bytes 的值



这两者的区别其实非常简单：**在调度的时候**，kube-scheduler **只会按照 requests 的值进行计**算。而在真正设置 Cgroups 限制的时候，kubelet 则会按照 limits 的值来进行设置。





spec.containers[].resources.limits.cpu

spec.containers[].resources.limits.memory

spec.containers[].resources.requests.cpu

spec.containers[].resources.requests.memory



在**调度**的时候，kube-scheduler 只会按照 **requests 的值**进行计算。而在真正**设置 Cgroups 限制**的时候，kubelet 则会按照 l**imits 的值**来进行设置。



当你指定了 requests.cpu=250m 之后，相当于将 Cgroups 的 **cpu.shares 的值设置为 (250/1000)*1024**。而当你没有设置 requests.cpu 的时候，cpu.shares 默认则是 1024。这样，Kubernetes 就通过 cpu.shares 完成了对 CPU 时间的按比例分配。



而如果你指定了 limits.cpu=500m 之后，则相当于将 Cgroups 的 cpu.cfs_quota_us 的值设置为 (500/1000)*100ms，而 cpu.cfs_period_us 的值始终是 100ms。这样，Kubernetes 就为你设置了这个容器只能用到 CPU 的 50%。

```
  containers:
  - name: db
    image: mysql
    env:
    - name: MYSQL_ROOT_PASSWORD
      value: "password"
    resources:
      requests:
        memory: "64Mi"
        cpu: "250m"
      limits:
        memory: "128Mi"
        cpu: "500m"
  - name: wp
    image: wordpress
    resources:
      requests:
        memory: "64Mi"
        cpu: "250m"
      limits:
        memory: "128Mi"
        cpu: "500m"
```



当这个 Pod 创建之后，**它的 qosClass 字段就会被 Kubernetes 自动设置为 Guaranteed**。需要注意的是，**当 Pod 仅设置了 limits 没有设置 requests 的时候，Kubernetes 会自动为它设置与 limits 相同的 requests 值**，所以，这也属于 Guaranteed 情况。



```
apiVersion: v1
kind: Pod
metadata:
  name: qos-demo
  namespace: qos-example
spec:
  containers:
  - name: qos-demo-ctr
    image: nginx
    resources:
      limits:
        memory: "200Mi"
        cpu: "700m"
      requests:
        memory: "200Mi"
        cpu: "700m"
```



**而当 Pod 不满足 Guaranteed 的条件，但至少有一个 Container 设置了 requests。那么这个 Pod 就会被划分到 Burstable 类别**。比如下面这个例子：



```
apiVersion: v1
kind: Pod
metadata:
  name: qos-demo-2
  namespace: qos-example
spec:
  containers:
  - name: qos-demo-2-ctr
    image: nginx
    resources:
      limits
        memory: "200Mi"
      requests:
        memory: "100Mi"

```



**而如果一个 Pod 既没有设置 requests，也没有设置 limits，那么它的 QoS 类别就是 BestEffort**。比如下面这个例子：

```
apiVersion: v1
kind: Pod
metadata:
  name: qos-demo-3
  namespace: qos-example
spec:
  containers:
  - name: qos-demo-3-ctr
    image: nginx
```



三种类型实际上，**QoS 划分的主要应用场景，是当宿主机资源紧张的时候，kubelet 对 Pod 进行 Eviction（即资源回收）时需要用到的。**

具体地说，当 Kubernetes 所管理的宿主机上不可压缩资源短缺时，就有可能触发 Eviction。比如，**可用内存（**memory.available）、**可用的宿主机磁盘空间**（nodefs.available），以及容器运行时镜像存储空间（imagefs.available）等等。

```
memory.available<100Mi
nodefs.available<10%
nodefs.inodesFree<5%
imagefs.available<15%

kubelet --eviction-hard=imagefs.available<10%,memory.available<500Mi,nodefs.available<5%,nodefs.inodesFree<5% --eviction-soft=imagefs.available<30%,nodefs.available<10% --eviction-soft-grace-period=imagefs.available=2m,nodefs.available=2m --eviction-max-pod-grace-period=600

```



在这个配置中，你可以看到**Eviction 在 Kubernetes 里其实分为 Soft 和 Hard 两种模式**。

其中，Soft Eviction 允许你为 Eviction 过程设置一段“优雅时间”，比如上面例子里的 imagefs.available=2m，就意味着当 imagefs 不足的阈值达到 2 分钟之后，kubelet 才会开始 Eviction 的过程。

而 Hard Eviction 模式下，Eviction 过程就会在阈值达到之后立刻开始





而当 Eviction 发生的时候，kubelet 具体会挑选哪些 Pod 进行删除操作，就需要参考这些 Pod 的 QoS 类别了。

- 首当其冲的，自然是 BestEffort 类别的 Pod。
- 其次，是属于 Burstable 类别、并且发生“饥饿”的资源使用量已经超出了 requests 的 Pod。
- 最后，才是 Guaranteed 类别。并且，Kubernetes 会保证只有当 Guaranteed 类别的 Pod 的资源使用量超过了其 limits 的限制，或者宿主机本身正处于 Memory Pressure 状态时，Guaranteed 的 Pod 才可能被选中进行 Eviction 操作。



Kubernetes 里一个非常有用的特性：**cpuset 的设置。**

我们知道，在使用容器的时候，你可以通过设置 **cpuset 把容器绑定到某个 CPU 的核上**，而不是像 cpushare 那样共享 CPU 的计算能力。

这种情况下，由于操作系统在 CPU 之间进行上下文切换的次数大大减少，容器里应用的性能会得到大幅提升。事实上，**cpuset 方式，是生产环境里部署在线应用类型的 Pod 时，非常常用的一种方式。**

- 首先，你的 Pod 必须是 Guaranteed 的 QoS 类型；

- 然后，你只需要将 Pod 的 CPU 资源的 requests 和 limits 设置为同一个相等的整数值即可。

  ```
  spec:
    containers:
    - name: nginx
      image: nginx
      resources:
        limits:
          memory: "200Mi"
          cpu: "2"
        requests:
          memory: "200Mi"
          cpu: "2"
  ```




# LimitRange and ResourceQuota

LimitRange可以：

- 限制namespace中每个pod或container的最小和最大资源用量。
- 限制namespace中每个PVC的资源请求范围。
- 限制namespace中资源请求和限制数量的比例。
- 配置资源的默认限制。



### 最终pod资源限制的处理机制

- 如果pod配置了请求和限制，并且请求和限制的值位于LimitRange对应类型资源的min和max之间。pod资源限制采用pod的配置。
- 如果pod配置了请求但没有配置限制，pod限制会采用LimitRange中的默认配置。
- 如果pod配置了限制但没有配置请求，pod请求会使用和limit相同的配置。
- 如果pod既没有配置请求又没有配置限制，pod限制使用LimitRange定义的默认配置，请求使用defaultRequest的配置。





命名空间下创建,并且在创建的时候没有**指定内存申请值和内存限制值**,则它会被默认分配request=256M的内存请求和limit=512M的

 alikube describe limits

```
apiVersion: v1
kind: 
LimitRange
metadata:
  name: mem-limit-range
spec:
  limits:
  - default:
      memory: 512Mi
    defaultRequest:
      memory: 256Mi
    type: Container
    
   
   
apiVersion: v1
kind: LimitRange
metadata:
  name: mem-limit-range
  namespace: example
spec:
  limits:
  - default:  # default limit
      memory: 512Mi
      cpu: 2
    defaultRequest:  # default request
      memory: 256Mi
      cpu: 0.5
    max:  # max limit
      memory: 800Mi
      cpu: 3
    min:  # min request
      memory: 100Mi
      cpu: 0.3
    maxLimitRequestRatio:  # max value for limit / request
      memory: 2
      cpu: 2
    type: Container # limit type, support: Container / Pod / PersistentVolumeClaim
    
注意：pod类型的LimitRange会检查之后创建的pod中所有container配置的资源限制总和，如果pod内所有container的资源限制总和超过了LimitRange的限制，pod创建会被拒绝。

```



> ResourceQuota 用来限制 namespace 中所有的 Pod 占用的总的资源 request 和 limit
>
> ```
> kubectl describe quota compute-resources --namespace=myspace
> ```

```
apiVersion: v1 
kind: ResourceQuota 
metadata: 
  name: compute-resources 
spec: 
  hard: 
    pods: "4" 
    requests.cpu: "1" 
    requests.memory: 1Gi 
    limits.cpu: "2" 
    limits.memory: 2Gi
```





# 优先级与抢占机制

正常情况下，**当一个 Pod 调度失败后，它就会被暂时“搁置”起来**，**直到 Pod 被更新，或者集群状态发生变化，调度器才会对这个 Pod 进行重新调度**。

这个 YAML 文件，定义的是一个名叫 **high-priority 的 PriorityClass，其中 value 的值是 1000000 （一百万）**。

**Kubernetes 规定，优先级是一个 32 bit 的整数，最大值不超过 1000000000（10 亿，1 billion），并且值越大代表优先级越高。**而超出 10 亿的值，其实是被 Kubernetes 保留下来分配给系统 Pod 使用的。显然，这样做的目的，就是保证系统 Pod 不会被用户抢占掉



 YAML 文件里的 **globalDefault** 被设置为 true 的话，那就意味着这个 PriorityClass 的值会成为系统的默认值。而如果这个值是 false，就表示我们只希望声明使用该 PriorityClass 的 Pod 拥有值为 1000000 的优先级，**而对于没有声明 PriorityClass 的 Pod 来说，它们的优先级就是 0。**

```
apiVersion: scheduling.k8s.io/v1beta1
kind: PriorityClass
metadata:
  name: high-priority
value: 1000000
globalDefault: false
description: "This priority class should be used for high priority service pods only."


apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels:
    env: test
spec:
  containers:
  - name: nginx
    image: nginx
    imagePullPolicy: IfNotPresent
  priorityClassName: high-priority


```







## 日志收集

DaemonSet

**sidecar**

由于 sidecar 跟主容器之间是共享 Volume 的，所以这里的 sidecar 方案的额外性能损耗并不高，**也就是多占用一点 CPU 和内存罢了**。

但需要注意的是，这时候，**宿主机上实际上会存在两份相同的日志文件：一份是应用自己写入的**；另一份则是 sidecar 的 stdout 和 stderr 对应的 JSON 文件。**这对磁盘是很大的浪费**。所以说，除非万不得已或者应用容器完全不可能被修改，否则我还是建议你直接使用方案一，或者直接使用下面的第三种方案。

```
apiVersion: v1
kind: Pod
metadata:
  name: counter
spec:
  containers:
  - name: count
    image: busybox
    args:
    - /bin/sh
    - -c
    - >
      i=0;
      while true;
      do
        echo "$i: $(date)" >> /var/log/1.log;
        echo "$(date) INFO $i" >> /var/log/2.log;
        i=$((i+1));
        sleep 1;
      done
    volumeMounts:
    - name: varlog
      mountPath: /var/log
  - name: count-log-1
    image: busybox
    args: [/bin/sh, -c, 'tail -n+1 -f /var/log/1.log']
    volumeMounts:
    - name: varlog
      mountPath: /var/log
  - name: count-log-2
    image: busybox
    args: [/bin/sh, -c, 'tail -n+1 -f /var/log/2.log']
    volumeMounts:
    - name: varlog
      mountPath: /var/log
  volumes:
  - name: varlog
    emptyDir: {}
```





### DNS

Pod 将看到自己的 FQDN 为 "`busybox-1.default-subdomain.my-namespace.svc.cluster-domain.example`"。

```
apiVersion: v1
kind: Service
metadata:
  name: default-subdomain
spec:
  selector:
    name: busybox
  clusterIP: None
  ports:
  - name: foo # 实际上不需要指定端口号
    port: 1234
    targetPort: 1234
---
apiVersion: v1
kind: Pod
metadata:
  name: busybox1
  labels:
    name: busybox
spec:
  hostname: busybox-1
  subdomain: default-subdomain
  containers:
  - image: busybox:1.28
    command:
      - sleep
      - "3600"
    name: busybox
---
apiVersion: v1
kind: Pod
metadata:
  name: busybox2
  labels:
    name: busybox
spec:
  hostname: busybox-2
  subdomain: default-subdomain
  containers:
  - image: busybox:1.28
    command:
      - sleep
      - "3600"
    name: busybox
    
   
   
指定dns
apiVersion: v1
kind: Pod
metadata:
  namespace: default
  name: dns-example
spec:
  containers:
    - name: test
      image: nginx
  dnsPolicy: "None"
  dnsConfig:
    nameservers:
      - 1.2.3.4
    searches:
      - ns1.svc.cluster-domain.example
      - my.dns.search.suffix
    options:
      - name: ndots
        value: "2"
      - name: edns0
```





### HPA

```
apiVersion: autoscaling/v2beta2
kind: HorizontalPodAutoscaler
metadata:
  name: php-apache
  namespace: default
spec:
  # HPA的伸缩对象描述，HPA会动态修改该对象的pod数量
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: php-apache
  # HPA的最小pod数量和最大pod数量
  minReplicas: 1
  maxReplicas: 10
  # 监控的指标数组，支持多种类型的指标共存
  metrics:
  # Object类型的指标
  - type: Object
    object:
      metric:
        # 指标名称
        name: requests-per-second
      # 监控指标的对象描述，指标数据来源于该对象
      describedObject:
        apiVersion: networking.k8s.io/v1beta1
        kind: Ingress
        name: main-route
      # Value类型的目标值，Object类型的指标只支持Value和AverageValue类型的目标值
      target:
        type: Value
        value: 10k
  # Resource类型的指标
  - type: Resource
    resource:
      name: cpu
      # Utilization类型的目标值，Resource类型的指标只支持Utilization和AverageValue类型的目标值
      target:
        type: Utilization
        averageUtilization: 50
  # Pods类型的指标
  - type: Pods
    pods:
      metric:
        name: packets-per-second
      # AverageValue类型的目标值，Pods指标类型下只支持AverageValue类型的目标值
      target:
        type: AverageValue
        averageValue: 1k
  # External类型的指标
  - type: External
    external:
      metric:
        name: queue_messages_ready
        # 该字段与第三方的指标标签相关联，（此处官方文档有问题，正确的写法如下）
        selector:
          matchLabels:
            env: "stage"
            app: "myapp"
      # External指标类型下只支持Value和AverageValue类型的目标值
      target:
        type: AverageValue
        averageValue: 30
```

