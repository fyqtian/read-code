## kustomize

https://github.com/kubernetes-sigs/kustomize

https://kubectl.docs.kubernetes.io/guides/config_management/introduction/

https://kubectl.docs.kubernetes.io/guides/introduction/kustomize/

https://www.jianshu.com/p/837d7ae77818

[使用kustomize管理声明式Kubernetes对象 (coderdocument.com)](http://www.coderdocument.com/docs/kubernetes/v1.14/tasks/manage_kubernetes_objects/declarative_management_of_kubernetes_objects_using_kustomize.html)

```doc
Kubernetes native configuration management

Kustomize introduces a template-free way to customize application configuration that simplifies the use of off-the-shelf applications. Now, built into kubectl as apply -k.
```

kustomize 是 kubernetes **原生的配置管理**，以无模板方式来定制应用的配置。kustomize 使用 k8s 原生概念帮助创建并复用资源配置(YAML)，允许用户以一个应用描述文件 （YAML 文件）为基础（Base YAML），**然后通过 Overlay 的方式生**成最终部署应用所需的描述文件。



如何管理不同环境或不同团队的应用的 Kubernetes YAML 资源

如何以某种方式管理**不同环境的微小差异，使得资源配置可以复用，减少 copy and change 的工作量**

如何简化维护应用的流程，不需要额外学习模板语法



```
curl -s -o "abc/#1.yaml" "https://raw.githubusercontent.com\
/kubernetes-sigs/kustomize\
/master/examples/helloWorld\
/{configMap,deployment,kustomization,service}.yaml"
```



```
#demo 

https://github.com/fyqtian/go-pakcage-learing/tree/main/k8s/kustomize-example
```

