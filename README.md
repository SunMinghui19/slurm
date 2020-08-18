# slurm
slurm+openmpi
# 安装规划
使用三台虚拟机ip地址和主机名如下  
```
192.168.133.40 compute1  
192.168.133.41 compute2  
192.168.133.42 compute3
```
系统 centos 7.3   
注：compute1作为我们的控制节点，同时compute1也是计算节点。compute2、compute3只是作为计算节点

# 一、基础配置
## 1.1配置ip与主机名映射（所有节点都要做）
使用 hostnamectl set-hostname compute[1-3]分别设置三台主机名
在compute1上vim /etc/hosts
添加如下内容：
192.168.133.40 compute1
192.168.133.41 compute2
192.168.133.42 compute3

使用如下命令复制到compute2和compute3
```
# scp /etc/hosts root@compute2:/etc/
# scp /etc/hosts root@compute3:/etc/
```

## 1.2 关闭防火墙（所有节点都要做）
```
# systemctl stop firewalld
# systemctl disable firewalld
# systemctl stop iptables
# systemctl disable iptables
```

## 1.3 安装epel源（所有节点都要做）
* EPEL源-是什么?为什么安装？
 EPEL (Extra Packages for Enterprise Linux)是基于Fedora的一个项目，为“红帽系”的操作系统提供额外的软件包，适用于RHEL、CentOS和Scientific Linux.
 我们下面需要用的munge就需要从这个源里面找。
* 使用很简单：
 需要安装一个叫”epel-release”的软件包，这个软件包会自动配置yum的软件仓库。当然你也可以不安装这个包，自己配置软件仓库也是一样的。
 我们这里使用下面的命令直接安装，不再配置软件仓库  
```
# yum install http://mirrors.sohu.com/fedora-epel/epel-release-latest-7.noarch.rpm
```

## 1.4 安装NFS（将nfs服务器安装到compute1即可）
* 1 安装nfs服务器
```
# yum -y install nfs-utils rpcbind
```
* 2 创建共享文件夹software
```
# mkdir /software
```
* 3 编辑/etc/exports文件
```
# vim /etc/exports
```
添加如下内容
/software/ *(rw,async,insecure,no_root_squash)

* 4 启动nfs服务器
```
# systemctl start nfs
# systemctl start rpcbind
# systemctl enable nfs
# systemctl enable rpcbind
```
## 1.5客户端挂载NFS（除了控制节点以外的节点既compute2和compute3）
* 1 安装nfs客户端
```
# yum -y install nfs-utils
```
* 2 创建挂载文件夹software
```
# mkdir /software
```
* 3 临时挂载 和开机自动挂载
临时挂载使用如下命令
```
# mount 192.168.133.40:/software /software
```
开机自动挂载：
vim /etc/fstab
#在该文件中挂载，使系统每次启动时都能自动挂载
添加如下内容
192.168.133.40:/software  /software       nfs    defaults 0 0


## 1.6配置SSH免密登录（在compute1上执行）
```
# ssh-keygen
# ssh-copy-id -i /root/.ssh/id_rsa.pub compute1
# ssh-copy-id -i /root/.ssh/id_rsa.pub compute2
# ssh-copy-id -i /root/.ssh/id_rsa.pub compute3
```

# 二、安装munge
## 1、创建Munge用户
Munge用户要确保Master Node和Compute Nodes的UID和GID相同，所有节点都需要安装Munge；
# groupadd -g 1108 munge
# useradd -m -c "Munge Uid 'N' Gid Emporium" -d /var/lib/munge -u 1108 -g munge -s /sbin/nologin munge
## 2、生成熵池
# yum install rng-tools
# systemctl start rngd
# systemctl enable rngd
使用/dev/urandom来做熵源
# rngd -r /dev/urandom
# vim /usr/lib/systemd/system/rngd.service
修改如下参数
[service]
ExecStart=/sbin/rngd -f -r /dev/urandom

# systemctl daemon-reload
# systemctl restart rngd
## 3、部署Munge
Munge是认证服务，实现本地或者远程主机进程的UID、GID验证

# yum install munge munge-libs munge-devel -y

创建全局密钥
在Master Node创建全局使用的密钥

# /usr/sbin/create-munge-key -r
# dd if=/dev/urandom bs=1 count=1024 > /etc/munge/munge.key

密钥同步到所有计算节点
# scp -p /etc/munge/munge.key root@compute2:/etc/munge
# scp -p /etc/munge/munge.key root@compute3:/etc/munge
# chown munge: /etc/munge/munge.key
# chmod 400 /etc/munge/munge.key

## 启动所有节点
# systemctl start munge
# systemctl status munge
# systemctl enable munge

## 测试Munge服务器
每个计算节点与控制节点进行连接验证
本地查看凭据
# munge -n
本地解码
# munge -n | unmunge
验证compute node，远程解码
# munge -n | ssh compute1 unmunge
Munge凭证基准测试
# remunge

三、配置Slurm

1、创建Slurm用户

# groupadd -g 1109 slurm
# useradd -m -c "Slurm manager" -d /var/lib/slurm -u 1109 -g slurm -s /bin/bash slurm

 

2、安装Slurm依赖

# yum install gcc gcc-c++ readline-devel perl-ExtUtils-MakeMaker pam-devel rpm-build mysql-devel -y

 

编译Slurm

# wget https://download.schedmd.com/slurm/slurm-19.05.7.tar.bz2 
# rpmbuild -ta slurm-19.05.7.tar.bz2 
# cd /root/rpmbuild/RPMS/x86_64/

 

所有节点安装Slurm

# yum localinstall slurm-*

 

3、配置控制节点Slurm
复制代码

# cp /etc/slurm/cgroup.conf.example /etc/slurm/cgroup.conf
# cp /etc/slurm/slurm.conf.example /etc/slurm/slurm.conf
# vim /etc/slurm/slurm.conf
##修改如下部分
ControlMachine=m1
ControlAddr=192.168.1.11
SlurmUser=slurm
SlurmctldLogFile=/var/log/slurm/slurmctld.log
SlurmdLogFile=/var/log/slurm/slurmd.log
SelectType=select/cons_res
SelectTypeParameters=CR_CPU_Memory
NodeName=m[1-3] RealMemory=3400 Sockets=1 CoresPerSocket=2 State=IDLE
PartitionName=all Nodes=m[1-3] Default=YES State=UP

复制代码

 

复制控制节点配置文件到计算节点

# scp /etc/slurm/*.conf m2:/etc/slurm/
# scp /etc/slurm/*.conf m3:/etc/slurm/

 

设置控制节点文件权限

# mkdir /var/spool/slurmctld
# chown slurm.slurm /var/spool/slurmctld

 

设置计算节点文件权限

# mkdir /var/spool/slurmd
# chown slurm: /var/spool/slurmd
# mkdir /var/log/slurm
# chown slurm: /var/log/slurm


5、配置控制节点Slurm Accounting
Accounting records为slurm收集作业步骤的信息，可以写入一个文本文件或数据库，但这个文件会变得越来越大，最简单的方法是使用MySQL来存储信息。

创建数据库的Slurm用户（MySQL自行安装）

mysql> grant all on slurm_acct_db.* to 'slurm'@'%' identified by 'slurm*456' with grant option;

 

配置slurmdbd.conf文件
复制代码

# cp /etc/slurm/slurmdbd.conf.example /etc/slurm/slurm.conf
# cat /etc/slurm/slurmdbd.conf
AuthType=auth/munge
AuthInfo=/var/run/munge/munge.socket.2
DbdAddr=192.168.1.11
DbdHost=m1
SlurmUser=slurm
DebugLevel=verbose
LogFile=/var/log/slurm/slurmdbd.log
PidFile=/var/run/slurmdbd.pid
StorageType=accounting_storage/mysql
StorageHost=10.23.227.111
StorageUser=slrum
StoragePass=slurm*456
StorageLoc=slurm_acct_db

复制代码

 

6、开启节点服务

启动控制节点slurmctld服务

# systemctl start slurmctld
# systemctl status slurmctld
# systemctl enable slurmctld

 

启动控制节点Slurmdbd服务

# systemctl start slurmdbd
# systemctl status slurmdbd
# systemctl enable slurmdbd

 

启动计算节点的服务

# systemctl start slurmd
# systemctl status slurmd
# systemctl enable slurmd

 

 

四、检查Slurm集群

查看集群

# sinfo
# scontrol show partition
# scontrol show node

提交作业    

# srun -N3 hostname
# scontrol show jobs

查看作业

# squeue -a


111
