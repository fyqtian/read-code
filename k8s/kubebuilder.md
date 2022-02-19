### kubebuilder

https://github.com/zq2599/blog_demos 程序员欣宸仓库

https://xinchen.blog.csdn.net/article/details/105086920 程序员欣宸文章汇总(DevOps篇)

https://edu.aliyun.com/roadmap/cloudnative?spm=a2cd1.725B#course

https://cloudnative.to/kubebuilder/				 中文

https://book.kubebuilder.io/quick-start.html 英文

https://github.com/kubernetes-sigs/kubebuilder 仓库

https://www.cnblogs.com/yangyuliufeng/p/14217887.html 实战

https://kubernetes.io/zh/docs/concepts/extend-kubernetes/operator/ 

https://blog.csdn.net/boling_cavalry/category_9280998.html client-go

https://xinchen.blog.csdn.net/article/details/113035349



https://xinchen.blog.csdn.net/article/details/113715847 group version

https://kubernetes.io/docs/reference/kubernetes-api/ k8s api

https://v1-19.docs.kubernetes.io/docs/reference/generated/kubernetes-api/v1.19/





```
kubectl api-resources -o wide
kubectl api-resources --api-group apps -o wide
kubectl explain configmap
kubectl api-versions
```





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

