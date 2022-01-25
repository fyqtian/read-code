### kubebuilder

https://edu.aliyun.com/roadmap/cloudnative?spm=a2cd1.725B#course

https://cloudnative.to/kubebuilder/				 中文

https://book.kubebuilder.io/quick-start.html 英文

https://github.com/kubernetes-sigs/kubebuilder 仓库

https://www.cnblogs.com/yangyuliufeng/p/14217887.html 实战

```
mkdir project
cd project && go mod init project

kubebuilder init --domain tutorial.kubebuilder.io

kubebuilder create api --group webapp --version v1 --kind Guestbook
```



```

 
 kubebuilder init --domain tutorial.kubebuilder.io --repo van --skip-go-version-check
 
 kubebuilder create api --group batch --version v1 --kind VanJob
 
 
#  会创建对应的crd文件
 make install 
```





```
ktest get --raw "/apis/autoscaling.alibabacloud.com/v1beta1/cronhorizontalpodautoscalers?timeout=32s" | jq .
```

