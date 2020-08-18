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
```
192.168.133.40 compute1
192.168.133.41 compute2
192.168.133.42 compute3
```
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
```
/software/ *(rw,async,insecure,no_root_squash)
```
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
* 3 临时挂载和开机自动挂载 

临时挂载使用如下命令:
```
# mount 192.168.133.40:/software /software
```
开机自动挂载：   
#在该文件中挂载，使系统每次启动时都能自动挂载
```
vim /etc/fstab
```
添加如下内容
```
192.168.133.40:/software  /software       nfs    defaults 0 0
```

## 1.6配置SSH免密登录（在compute1上执行）
ssh-keygen  产生公钥与私钥对.  
ssh-copy-id 将本机的公钥复制到远程机器的authorized_keys文件中，ssh-copy-id也能让你有到远程机器的home, ~./ssh , 和 ~/.ssh/authorized_keys的权利   
```
#执行如下命令后一路enter就可以
# ssh-keygen
# 将公钥复制到其他机器上
# ssh-copy-id -i /root/.ssh/id_rsa.pub compute2
# ssh-copy-id -i /root/.ssh/id_rsa.pub compute3
```

# 二、安装munge
## 简介
munge是认证服务，用于生成和验证证书。应用于大规模的HPC集群中。  
它允许进程在【具有公用的用户和组的】主机组中，对另外一个【本地或者远程的】进程的UID和GID进行身份验证。  
这些主机构成由共享密钥定义的安全领域。在此领域中的客户端能够在不使用root权限，不保留端口，或其他特定平台下进行创建凭据和验证。  
简而言之，在集群中，munge能够实现本地或者远程主机进程的GID和UID验证  
注：此处的GID为GroupID，即组ID，用来标识用户组的唯一标识符  
    UID为UserID，即用户ID，用来标识每个用户的唯一标识符
    
## 2.1、创建Munge用户（所有节点都要做）

Munge用户要确保Master Node和Compute Nodes的UID和GID相同，所有节点都需要安装Munge；
```
# groupadd -g 1108 munge
# useradd -m -c "Munge Uid 'N' Gid Emporium" -d /var/lib/munge -u 1108 -g munge -s /sbin/nologin munge
```

*注：groupadd和useradd的用法
```
groupadd命令用于将新组加入系统。
groupadd [－g gid] [－o]] [－r] [－f] groupname
－g gid：指定组ID号。
－o：允许组ID号，不必惟一。
－r：加入组ID号，低于499系统账号。
－f：加入已经有的组时，发展程序退出。

useradd命令用来建立用户帐号和创建用户的起始目录，使用权限是root用户。
useradd [－d home] [－s shell] [－c comment] [－m [－k template]] [－f inactive] [－e expire ] [－p passwd] [－r] name
－c：加上备注文字，备注文字保存在passwd的备注栏中。 
－d：指定用户登入时的启始目录。
－D：变更预设值。
－e：指定账号的有效期限，缺省表示永久有效。
－f：指定在密码过期后多少天即关闭该账号。
－g：指定用户所属的起始群组。
－G：指定用户所属的附加群组。
－m：自动建立用户的登入目录。
－M：不要自动建立用户的登入目录。
－n：取消建立以用户名称为名的群组。
－r：建立系统账号。
－s：指定用户登入后所使用的shell。
－u：指定用户ID号。
```
## 2.2、生成熵池即安装rngd服务（若安装只在compute1安装即可，不安装也没有问题）
### 简介
Linux内核采用熵来描述数据的随机性。熵（entropy）是描述系统混乱无序程度的物理量，一个系统的熵越大则说明该系统的有序性越差，即不确定性越大。在信息学中，熵被用来表征一个符号或系统的不确定性，熵越大，表明系统所含有用信息量越少，不确定度越大。
生成随机数是密码学中的一项基本任务，是生成加密密钥、加密算法和加密协议所必不可少的，随机数的质量对安全性至关重要。最近报道有人利用随机数缺点成功 攻击了某网站，获得了管理员的权限。美国和法国的安全研究人员最近也评估了两个 Linux 内核 PRNG——/dev/random 和/dev/urandom 的安全性，认为 Linux 的伪随机数生成器不满足鲁棒性的安全概念，没有正确积累熵。

rngd主要用于增加内核中的熵数量以使其/dev/random更快。默认情况下，/dev/random它非常慢，因为它仅从设备驱动程序和其他（慢速）源收集熵。rngd允许使用更快的熵源，主要是现代随机硬件（如最近的AMD / Intel处理器，Via Nano或Raspberry Pi ）中存在的硬件随机数生成器（TRNG）

* 1安装rng-tools并启动rngd服务
```
# yum install rng-tools
# systemctl start rngd
# systemctl enable rngd
```
* 2使用/dev/urandom来做熵源
```
# rngd -r /dev/urandom
# vim /usr/lib/systemd/system/rngd.service
```
修改如下参数
```
[service]
ExecStart=/sbin/rngd -f -r /dev/urandom
```
* 3重新加载配置文件并重启rngd服务
```
# systemctl daemon-reload
# systemctl restart rngd
```
## 2.3、部署Munge
Munge是认证服务，实现本地或者远程主机进程的UID、GID验证
* 1安装munge（所有节点都要做）
```
# yum install munge munge-libs munge-devel -y
```
* 2创建全局密钥（compute1上执行）
在compute1创建全局秘钥配置权限后，发送到其它节点
```
# /usr/sbin/create-munge-key -r
# dd if=/dev/urandom bs=1 count=1024 > /etc/munge/munge.key
# chown munge: /etc/munge/munge.key
# chmod 400 /etc/munge/munge.key
```
密钥同步到所有计算节点
```
# scp -p /etc/munge/munge.key root@compute2:/etc/munge
# scp -p /etc/munge/munge.key root@compute3:/etc/munge

```
*注
```
利用 chown 将指定文件的拥有者改为指定的用户或组，用户可以是用户名或者用户ID；组可以是组名或者组ID
chown [-cfhvR] [--help] [--version] user[:group] file

    user : 新的文件拥有者的使用者 ID
    group : 新的文件拥有者的使用者组(group)
    -c : 显示更改的部分的信息
    -f : 忽略错误信息
    -h :修复符号链接
    -v : 显示详细的处理信息
    -R : 处理指定目录以及其子目录下的所有文件
    --help : 显示辅助说明
    --version : 显示版本
    
格式：chmod [0-7][0-7][0-7]
第1个[0-7]:表示该档案的拥有者

第2个[0-7]:表示与该档案的拥有者属于同一个群体(group)者

第3个[0-7]:表示其他以外的人(other)
数字权限是基于二进制数字系统而创建的，读（read，r）的值是4，写（write，w）的值是2，执行（execute，x）的值是1，没有授权的值是0。
```
* 3启动所有节点（所有节点都执行）
```
# systemctl start munge
# systemctl status munge
# systemctl enable munge
```

## 2.4测试Munge服务器
每个计算节点与控制节点进行连接验证

本地查看凭据
```
# munge -n
```
本地解码
```
# munge -n | unmunge
```
验证compute node，远程解码
```
# munge -n | ssh compute1 unmunge
```
Munge凭证基准测试
```
# remunge
```

# 三、配置Slurm

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
