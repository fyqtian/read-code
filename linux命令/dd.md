### dd



```
dd命令可以创建指定大小的文件

命令：     dd if=/dev/zero of=test bs=1M count=1000

会在当前目录下生成一个大小为1M*1000=1000M大小的test.img文件，它的内容都是0（因从/dev/zero中读取，/dev/zero为0源）

但是这样为实际写入硬盘，文件产生速度取决于硬盘读写的速度，如果要产生超大文件，速度会很慢。

 

在某些场景下，我们只想让文件系统认为存在一个超大文件在此，但是并不实际写入硬盘，可以这样
命令：  dd if=/dev/zero of=test bs=1M count=0 seek=150000

此时创建的文件在文件系统中的显示大小为150000MB，但是并不实际占用block，因此创建速度与内存速度相当。

seek的作用是跳过输出文件中指定大小的部分，这就达到了创建大文件，但是并不实际写入的目的。

当然，因为不实际写入硬盘，所以你在容量只有10G的硬盘上创建100G的此类文件都是可以的。

```

