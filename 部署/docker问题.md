### docker问题

centos 安装docker

```
 sudo yum install -y yum-utils
 sudo yum-config-manager \
    --add-repo \
    https://download.docker.com/linux/centos/docker-ce.repo
    
  yum install docker-ce docker-ce-cli containerd.io
  
  
   yum list docker-ce --showduplicates | sort -r
   
   
   sudo yum install docker-ce-<VERSION_STRING> docker-ce-cli-<VERSION_STRING> containerd.io
   
   
   systemctl start docker
   
   
    curl -fsSL https://get.docker.com -o get-docker.sh
 sudo sh get-docker.sh
```



172网段冲突

https://github.com/docker/compose/issues/4336

https://www.cnblogs.com/lemon-le/p/10531449.html

```
{
    "bip":"192.168.0.1/24",
    "default-address-pools" : [
    {
      "base" : "172.31.0.0/16",
      "size" : 24
    }
  ]
}
```



none镜像

docker images -f dangling=true | head -n 3



删除无效镜像

docker image prune



## cap_add, cap_drop

这部分用于调整容器操作内核权限、能力。这部分有一点点变化，就是在 Swarm 模式中，Compose 会忽略这部分参数的值。



```undefined
  - ALL

cap_drop:
  - NET_ADMIN
  - SYS_ADMIN
```



-v 挂在文件 修改后不生效

```
在启动docker容器时，为了保证一些基础配置与宿主机保持同步，通常需要将这些配置文件挂载进 docker容器，例如/etc/resolv.conf//etc/hosts//etc/localtime等。

当这些配置变化时，我们通常会修改这些文件。但是此时遇到了一个问题：

当在宿主机上修改这些文件后，docker容器内查看时，这些文件并未发生对应的修改。

然后通过查阅相关资料，发现该问题是由docker -v挂载文件和某些编辑器存储文件的行为共同导致 的。

docker 挂载文件时，并不是挂载了某个文件的路径，而是实打实的挂载了对应的文件，即挂载了某 个指定的inode文件。
某些编辑器(vi)在编辑保存文件时，采用了备份、替换的策略，即编辑过程中，将变更写入新文件， 保存时，再将备份文件替换原文件，此时会导致文件的inode发生变化。
原inode对应的文件其实并没有发生修改。
因此，我们从宿主机上修改这些文件时，应该采用echo重定向等操作，避免文件的inode发生变化。

```





docker-compose 容器连接



docker-compose config 解析docker-compose.yml



#### image: yandex/clickhouse-server:${CLICKHOUSE_VERSION:-20.11}

Both `$VARIABLE` and `${VARIABLE}` syntax are supported. Additionally when using the [2.1 file format](https://docs.docker.com/compose/compose-file/compose-versioning/#version-21), it is possible to provide inline default values using typical shell syntax:

- `${VARIABLE:-default}` evaluates to `default` if `VARIABLE` is unset or empty in the environment.
- `${VARIABLE-default}` evaluates to `default` only if `VARIABLE` is unset in the environment.





### 默认存储位置

{
   "data-root": "/data/docker"
}



docker-compose

sudo curl -L "https://github.com/docker/compose/releases/download/1.24.1/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose



```
sudo chmod +x /usr/local/bin/docker-compose
```