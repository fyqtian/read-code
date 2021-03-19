### Docker面试题

Docker与虚拟机的不同点在哪里？

答：Docker不是虚拟化方法。它依赖于实际实现基于容器的虚拟化或操作系统级虚拟化的其他工具。为此，Docker最初使用LXC驱动程序，然后移动到libcontainer现在重命名为runc。Docker主要专注于在应用程序容器内自动部署应用程序。应用程序容器旨在打包和运行单个服务，而系统容器则设计为运行多个进程，如虚拟机。因此，Docker被视为容器化系统上的容器管理或应用程序部署工具。



Dockerfile中的命令COPY和ADD命令有什么区别？

答：一般而言，虽然ADD并且COPY在功能上类似，但是首选COPY。

那是因为它比ADD更易懂。COPY仅支持将本地文件复制到容器中，**而ADD具有一些功能（如仅限本地的tar提取和远程URL支持），这些功能并不是很明显**。因此，ADD的最佳用途是将本地tar文件自动提取到镜像中，如ADD rootfs.tar.xz /。



如何在生产中监控Docker？

cadvisor

答：Docker提供docker stats和docker事件等工具来监控生产中的Docker。我们可以使用这些命令获取重要统计数据的报告。

Docker统计数据：当我们使用容器ID调用docker stats时，我们获得容器的CPU，内存使用情况等。它类似于[Linux](https://www.wkcto.com/courses/linux.html)中的top命令。





如何查看镜像支持的环境变量？
使用sudo docker run IMAGE env



本地的镜像文件都存放在哪里
 于Docker相关的本地资源存放在/var/lib/docker/目录下，其中container目录存放容器信息，graph目录存放镜像信息，aufs目录下存放具体的镜像底层文件。

