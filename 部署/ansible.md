### ansible

我们可以修改一下配置文件来修改设置，配置文件被读取的顺序如下：

```
* ANSIBLE_CONFIG (一个环境变量)
* ansible.cfg (位于当前目录中)
* .ansible.cfg (位于家目录中)
* /etc/ansible/ansible.cfg
```

配置文件

[ansible/ansible.cfg at devel · ansible/ansible (github.com)](https://github.com/ansible/ansible/blob/devel/examples/ansible.cfg)



https://www.zsythink.net/archives/2481

http://www.ansible.com.cn/docs/intro_getting_started.html#id3

ansible --version

/etc/ansible/host



https://blog.csdn.net/Kangshuo2471781030/article/details/82665219 系列命令



https://github.com/ansible/ansible-examples。

**ansible2.0版本以后的写法**

ansible -i inventory -m dosomething

127.0.0.1 ansible_port=8888 ansible_user=root ansible_ssh_pass=123456

Aliasname ansible_host=127.0.0.1 ansible_port=8888 ansible_user=root ansible_ssh_pass=123456



ansible all -m ping

ansible group -m ping 



ansible -i ./host all -m ping



ansible -i host test -a "whoami" 

ansible -i host test   -u {user}    -a "whoami" 



```
ansible webservers -m service -a "name=httpd state=restarted"
```

```
ansible atlanta -a "/sbin/reboot" -f 10

ansible atlanta -a "/sbin/reboot" -f 10 -u username --become [--ask-become-pass]

ansible atlanta -m copy -a "src=/etc/hosts dest=/tmp/hosts"

ansible webservers -m file -a "dest=/srv/foo/b.txt mode=600 owner=mdehaan group=mdehaan"


创建文件夹
ansible webservers -m file -a "dest=/path/to/c mode=755 owner=mdehaan group=mdehaan state=directory"
删除文件
ansible webservers -m file -a "dest=/path/to/c state=absent"
```



更新软件

```
ansible webservers -m yum -a "name=acme state=present"
```

### [Managing users and groups](https://docs.ansible.com/ansible/2.9/user_guide/intro_adhoc.html#id9)

You can create, manage, and remove user accounts on your managed nodes with ad-hoc tasks:

```
$ ansible all -m user -a "name=foo password=<crypted password here>"

$ ansible all -m user -a "name=foo state=absent"
```



```
$ ansible webservers -m service -a "name=httpd state=started"
```



ansible-config dump









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







### **playbooks**

http://www.ansible.com.cn/docs/playbooks_intro.html#about-playbooks



Ansible-glaxy

ansible-glaxy init

ansible-playbook --step

Ansible-playbook stie.yml --start-task

facts

[ansible gather_facts配置_地下库-CSDN博客_ansible gather_facts](https://ghostwritten.blog.csdn.net/article/details/113696617)



debug

[Ansible Debug 调试 | 温欣爸比的博客 (wxnacy.com)](http://wxnacy.com/2020/06/04/ansible-debug/)





清华大学源

https://mirrors.tuna.tsinghua.edu.cn/

https://mirrors.ustc.edu.cn/





ansible awx 18版本以前的

https://www.linuxtechi.com/install-ansible-awx-on-ubuntu/



```
ansible  all -i "ip," -m ping -u root

```



ansible 指定某个组



 ansible-playbook -i hosts deploy.yaml -l "ap-chengdu"

 ansible-playbook -i hosts deploy.yaml --limit "ap-chengdu"

**测试**

 ansible-playbook  -i hosts --list-hosts deploy.yaml --limit "ap-chengdu"







ansible-galaxy install xxx 会默认装在用户家目录下



rpm -qi 包 查看安装时间



意思是说存在没有完成的yum事务，建议先运行yum-complete-transaction命令结束它们。

yum-complete-transaction --cleanup-only





### command

```
使用command模块在远程主机中执行命令时，不会经过远程主机的shell处理，在使用command模块时，如果需要执行的命令中含有重定向、管道符等操作时，这些符号也会失效，比如"<", ">", "|", ";" 和 "&" 这些符号，
```



### shell

```
shell模块在远程主机中执行命令时，会经过远程主机上的/bin/sh程序处理。
```





