

yum install -y openssl openssh-server



ssh-keygen -q -t rsa -b 2048 -f /etc/ssh/ssh_host_rsa_key -N ''  
ssh-keygen -q -t ecdsa -f /etc/ssh/ssh_host_ecdsa_key -N ''
ssh-keygen -t dsa -f /etc/ssh/ssh_host_ed25519_key -N ''

**vim /etc/ssh/sshd_config**

17行到22行

PermitRootLogin



mkdir -p /var/run/sshd

 /usr/sbin/sshd -D & 



