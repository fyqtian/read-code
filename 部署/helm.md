### Helm	

```fallback

https://helm.sh/zh/docs/intro/install/

mac
brew install helm

linux
https://github.com/helm/helm/releases


 curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3
$ chmod 700 get_helm.sh
$ ./get_helm.sh
```





```
初始化
helm repo add bitnami https://charts.bitnami.com/bitnami

helm search repo bitnami

helm search hub 从 Artifact Hub 中查找并列出 helm charts。 Artifact Hub中存放了大量不同的仓库。
helm search repo 从你添加（使用 helm repo add）到本地 helm 客户端中的仓库中进行查找。该命令基于本地数据进行搜索，无需连接互联网。


helm repo update



helm install bitnami/mysql --generate-name

helm show chart bitnami/mysql  命令简单的了解到这个chart的基本信息

helm show all bitnami/mysql  获取关于该chart的所有信息。

helm ls

helm uninstall mysql-1612624192

如果您在执行 helm uninstall 的时候提供 --keep-history 选项， Helm将会保存版本历史。 您可以通过命令查看该版本的信息

helm get -h


使用 helm install 命令来安装一个新的 helm 包。最简单的使用方法只需要传入两个参数：你命名的release名字和你想安装的chart的名称。
helm install happy-panda bitnami/wordpress

现在wordpress chart 已经安装。注意安装chart时创建了一个新的 release 对象。上述发布被命名为 happy-panda。 （如果想让Helm生成一个名称，删除发布名称并使用--generate-name。）

helm status happy-panda

使用 helm show values 可以查看 chart 中的可配置选项
helm show values bitnami/wordpress



// 动态配置
然后，你可以使用 YAML 格式的文件覆盖上述任意配置项，并在安装过程中使用该文件。
$ echo '{mariadb.auth.database: user0db, mariadb.auth.username: user0}' > values.yaml
$ helm install -f values.yaml bitnami/wordpress --generate-name

上述命令将为 MariaDB 创建一个名称为 user0 的默认用户，并且授予该用户访问新建的 user0db 数据库的权限。chart 中的其他默认配置保持不变。

安装过程中有两种方式传递配置数据：

--values (或 -f)：使用 YAML 文件覆盖配置。可以指定多次，优先使用最右边的文件。
--set：通过命令行的方式对指定项进行覆盖。



// 安装

上述命令将为 MariaDB 创建一个名称为 user0 的默认用户，并且授予该用户访问新建的 user0db 数据库的权限。chart 中的其他默认配置保持不变。

安装过程中有两种方式传递配置数据：

--values (或 -f)：使用 YAML 文件覆盖配置。可以指定多次，优先使用最右边的文件。
--set：通过命令行的方式对指定项进行覆盖。
```





```
三大概念
Chart 代表着 Helm 包。它包含在 Kubernetes 集群内部运行应用程序，工具或服务所需的所有资源定义。你可以把它看作是 Homebrew formula，Apt dpkg，或 Yum RPM 在Kubernetes 中的等价物。

Repository（仓库） 是用来存放和共享 charts 的地方。它就像 Perl 的 CPAN 档案库网络 或是 Fedora 的 软件包仓库，只不过它是供 Kubernetes 包所使用的。

Release 是运行在 Kubernetes 集群中的 chart 的实例。一个 chart 通常可以在同一个集群中安装多次。每一次安装都会创建一个新的 release。以 MySQL chart为例，如果你想在你的集群中运行两个数据库，你可以安装该chart两次。每一个数据库都会拥有它自己的 release 和 release name。
```



helm安装顺序

```
Helm按照以下顺序安装资源：

Namespace
NetworkPolicy
ResourceQuota
LimitRange
PodSecurityPolicy
PodDisruptionBudget
ServiceAccount
Secret
SecretList
ConfigMap
StorageClass
PersistentVolume
PersistentVolumeClaim
CustomResourceDefinition
ClusterRole
ClusterRoleList
ClusterRoleBinding
ClusterRoleBindingList
Role
RoleList
RoleBinding
RoleBindingList
Service
DaemonSet
Pod
ReplicationController
ReplicaSet
Deployment
HorizontalPodAutoscaler
StatefulSet
Job
CronJob
Ingress
APIService

```





## helm upgrade' 和 'helm rollback'：升级 release 和失败时恢复

当你想升级到 chart 的新版本，或是修改 release 的配置，你可以使用 `helm upgrade` 命令。

一次升级操作会使用已有的 release 并根据你提供的信息对其进行升级。由于 Kubernetes 的 chart 可能会很大而且很复杂，Helm 会尝试执行最小侵入式升级。即它只会更新自上次发布以来发生了更改的内容。

```fallback
$ helm upgrade -f panda.yaml happy-panda bitnami/wordpress
```



我们可以使用 `helm get values` 命令来看看配置值是否真的生效了：

```fallback
$ helm get values happy-panda
mariadb:
  auth:
    username: user1
```



现在，假如在一次发布过程中，发生了不符合预期的事情，也很容易通过 `helm rollback [RELEASE] [REVISION]` 命令回滚到之前的发布版本。

```fallback
$ helm rollback happy-panda 1
```



```fallback
helm repo list
```



创建charts

```fallback
helm create deis-workflow
```

当准备将 chart 打包分发时，你可以运行 `helm package` 命令

```go
helm package deis-workflow

helm install deis-workflow ./deis-workflow-0.1.0.tgz
```







### helm3移除tiler

```
创建名称为tiller的Service Account
创建tiller对Service Account具有集群管理员权限的ClusterRoleBinding。
```





helm get manifest test1



helm install --debug --dry-run goodly-guppy ./test1/

