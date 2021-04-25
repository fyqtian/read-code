### ansible

https://www.zsythink.net/archives/2481



ansible --version

/etc/ansible/host



**ansible2.0版本以后的写法**

ansible -i inventory -m dosomething

127.0.0.1 ansible_port=8888 ansible_user=root ansible_ssh_pass=123456

Aliasname ansible_host=127.0.0.1 ansible_port=8888 ansible_user=root ansible_ssh_pass=123456



ansible all -m ping

ansible group -m ping 



[a]

127.0.0.1

[pro:children]

a







**免密登陆**

ssh-keygen

ssh-copy-id -i /root/.ssh/id_rsa.pub root@10.1.1.60



通过ssh-agent帮助我们管理密钥。https://www.zsythink.net/archives/2407



查看模块

ansible-doc  -l

ansible-doc -s ping





**文件操作**

https://www.zsythink.net/archives/2542

https://www.zsythink.net/archives/2560

拉取文件 **fetch**

ansible testA -m fetch -a “src=/etc/fstab dest=/testdir/ansible/”



上传文件 **copy**

ansible 127.0.0.1 -m copy -a "src=/opt/a.txt dest=/opt"

**src参数**   ：用于指定需要copy的文件或目录

**dest参数** ：用于指定文件将被拷贝到远程主机的哪个目录中，dest为必须参数

**content参数** ：当不使用src指定拷贝的文件时，可以使用content直接指定文件内容，src与content两个参数必有其一，否则会报错。

**force参数** :  当远程主机的目标路径中已经存在同名文件，并且与ansible主机中的文件内容不同时，是否强制覆盖，可选值有yes和no，默认值为yes，表示覆盖，如果设置为no，则不会执行覆盖拷贝操作，远程主机中的文件保持不变。

**backup参数** :  当远程主机的目标路径中已经存在同名文件，并且与ansible主机中的文件内容不同时，是否对远程主机的文件进行备份，可选值有yes和no，当设置为yes时，会先备份远程主机中的文件，然后再将ansible主机中的文件拷贝到远程主机。

**owner参数** : 指定文件拷贝到远程主机后的属主，但是远程主机上必须有对应的用户，否则会报错。

**group参数** : 指定文件拷贝到远程主机后的属组，但是远程主机上必须有对应的组，否则会报错。

**mode参数** : 指定文件拷贝到远程主机后的权限，如果你想将权限设置为”rw-r–r–“，则可以使用mode=0644表示，如果你想要在user对应的权限位上添加执行权限，则可以使用mode=u+x表示。





## file模块

## blockinfile模块

## lineinfile模块