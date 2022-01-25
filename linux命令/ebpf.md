### ebpf



### 官网

https://ebpf.io/



### BLOG

https://arthurchiao.art/blog/cilium-bpf-xdp-reference-guide-zh/



### linux网络

https://github.com/leandromoreira/linux-network-performance-parameters





### xdp-tutorial

https://github.com/xdp-project/xdp-tutorial



### 介绍

https://www.jianshu.com/p/a9a07855ab15



#### 安装

包安装有404的问题 https证书也有问题，建议源码安装

https://github.com/iovisor/bcc/blob/master/INSTALL.md

#### 教程

https://github.com/iovisor/bcc/blob/master/docs/tutorial.md



https://www.ebpf.top/post/ebpf_intro/



https://github.com/DavadDi/bpf_study



### example

https://github.com/bpftools/linux-observability-with-bpf



```
cat /proc/sys/kernel/unprivileged_bpf_disabled

sudo sysctl kernel.unprivileged_bpf_disabled=1




日志 /sys/kernel/debug/tracing/trace_pipe

```



- **kprobes**：实现内核中动态跟踪。 kprobes 可以跟踪到 Linux 内核中的函数入口或返回点，但是不是稳定 ABI 接口，可能会因为内核版本变化导致，导致跟踪失效。

- **uprobes**：用户级别的动态跟踪。与 kprobes 类似，只是跟踪的函数为用户程序中的函数。

- **tracepoints**：内核中静态跟踪。tracepoints 是内核开发人员维护的跟踪点，能够提供稳定的 ABI 接口，但是由于是研发人员维护，数量和场景可能受限。

- **perf_events**：定时采样和 PMC。

- ##### USDT 用户态程序静态跟踪点







注意 go1.17以前是栈 以后是寄存器

```
arg0, arg1, ..., argN. - Arguments to the traced function; assumed to be 64 bits wide
sarg0, sarg1, ..., sargN. - Arguments to the traced function (for programs that store arguments on the stack); assumed to be 64 bits wide
```





### bpftrace

https://homerl.github.io/2020/07/24/bpftrace/

```
apt-get install -y bpftrace

bpftrace -l

bpftrace -l 'kprobe:*' 



https://github.com/iovisor/bpftrace/blob/master/docs/reference_guide.md#1-hello-world


objdump -tT  main2 | grep ping

nm  main2 | | grep main.say

readelf -n ./hello_usdt



bpftrace -e 'uprobe:./main2:main.say { printf("read a line %d\n",sarg0); }'
bpftrace -e 'uprobe:./main2:main.say { printf("arg %s\n",ustack(perf)); }'



bpftrace -e 'uprobe:./main2:main.say {@reads = count()}'
```





### Tracing point

https://www.kernel.org/doc/Documentation/trace/tracepoint-analysis.txt

```
ll /sys/kernel/debug/tracing/events/
```





```
 bpf_get_current_pid_tgid   bpf_get_current_pid_tgid() >> 32 进行过滤的
 
current->tgid << 32 | current->pid，高 32 位置为 tgid ，低 32 位为 pid(tid)，如果我们计划采用进程空间传统的 pid 过滤那么则可以这样写 tcptop.py：
```





### USDT 需要在用户代码加入追踪点

https://blog.srvthe.net/usdt-report-doc/

```
#include <sys/sdt.h>
int main() {
	DTRACE_PROBE("hello-usdt", "probe-main");
}

```





### network

https://tldp.org/HOWTO/Traffic-Control-HOWTO/index.html

```
网卡设备

ls -la /sys/class/net

tc qdisc ls
```



### PIXIE

https://blog.px.dev/ebpf-function-tracing/
