---
Title: Ubuntu部署NIS服务
---



# Ubuntu 部署NIS服务

在一个大型的网域当中，如果有多部 Linux 主机，万一要每部主机都需要设定相同的账号与密码时，你该怎么办？复制 /etc/passwd ？应该没有这么呆吧？如果能够有一部账号主控服务器来管理网域中所有主机的账号， 当其他的主机有用户登入的需求时，才到这部主控服务器上面要求相关的账号、密码等用户信息， 如此一来，如果想要增加、修改、删除用户数据，只要到这部主控服务器上面处理即可， 这样就能够降低重复设定使用者账号的步骤了。

[CentOS的配置方法在这里](http://cn.linux.vbird.org/linux_server/0430nis.php)

```
实验环境
virtualbox : 6.1 ,网络设置为桥接模式
master : ubuntu 18.04.03 LTS 192.168.50.79
slave : ubuntu 18.04.03	LTS 192.168.50.185
```



## 服务端设置

``` shell
# 1. 安装NIS
apt update 
apt install nis 
# 安装的时候会让设置域名，可以使用nisdomainname 查看/设置域名
# 2. 配置hosts文件
vim /etc/hosts
192.168.50.185	slave
192.168.50.79		master

# 3. 修改配置文件
vim /etc/default/nis
修改如下
NISSERVER=false => NISSERVER=master

vim /etc/ypserv.conf
最后写入
127.0.0.1/255.0.0.0	:*	:*	:none
192.168.50.0/255.255.255.0	:*	:*	:none
*	:*	:*	:deny
# 其中各字段的意义如下
# Host		: 需要设置权限的主机或网络	
# Domain	: 允许访问的域
# Map			: 密码数据库
# Security:	权限， none即不设置限制，deny为拒绝，port为只允许小于1024端口连接NIS

# 4. 启动ypserv
service ypserv start
# 5. 初始化数据库
/usr/lib/yp/ypinit -m
确认域名无误后按<Ctrl D>
然后输入y，之后会初始化数据库，数据库会保存在 /var/yp/<domain> 

```





## 客户端设置

``` shell
# 1. 安装NIS
apt update 
apt install nis

# 2. 修改hosts文件
vim /etc/hosts
192.168.50.185	slave
192.168.50.79		master

# 3. 修改DNS信息
vim /etc/nsswitch.conf
在以下行后添加nis字段
passwd:	compat	systemd nis # 此处的nis都是自己加的
shadow: compat	nis
group:	compat	systemd	nis
gshadow:	files nis

# 4. 修改配置文件
vim /etc/yp.conf
写入以下内容
#domain <此处写自己的域名> server <此处写nis服务器的ip>
domain master server 192.168.50.79

# 5. 启动服务
service ypbind start
```



## 修改密码

```shell
yppasswd nisuser1
# 先输入旧密码， 然后输入两次新密码
/usr/lib/yp/ypinit -m # 更新数据库
```



## 使用autofs自动挂载home目录

Autofs与Mount/Umount的不同之处在于，它是一种守护程序。如果它检测到用户正试图访问一个尚未挂接的文件系统，它就会自动检测该文件系统，如果存在，那么Autofs会自动将其挂接。另一方面， 如果它检测到某个已挂接的文件系统在一段时间内没有被使用，那么Autofs会自动将其卸载。因此一旦运行了Autofs后，用户就不再需要手动完成文件系统的挂接和卸载。

```shell
# 1. 在所有需要挂载home目录的主机上安装autofs
apt update 
apt install autofs
# 2. 创建配置文件
vim /etc/auto.master
写入以下内容
/home/nishome	/etc/auto.nishome # 前边是挂载点，后边是指定挂载时用的配置文件
vim /etc/auto.nishome
写入以下内容
# 挂载目录 挂载文件类型及权限 :设备名称
*		-rw		192.168.50.76:/mnt/home/home/&
# * 代表所有主机都可以挂载此目录
# rw是权限
# 192.168.50.76是nfs服务器，实验中用FreeNAS分享了一个NFS数据集
# /mnt/home/home/&是要挂载的目录在NFS服务器上的路径，其中'&'指代用户名

# 3. 启动autofs
service autofs start

最后效果就是在登录某台主机时，会自动将NFS服务器上自己用户名的home目录挂载到 /home/nishome/下边去，路径前缀在auto.master文件中给出
```



## 添加用户流程

1. 在NIS master上创建用户 `useradd  -d /home/nishome/<nisusername> <nisusername> /bin/bash`
2. 在NIS master上为新创建的用户创建密码 `passwd <nisusername>` 
3. 在NIS master上更新数据库 `/usr/lib/yp/ypinit -m`
4. 在FreeNAS上创建用户家目录
   1. 目录服务(Directory Serices)->网卡(NIS)->重建目录服务缓存(REBUILD DIRECTORY SERVICE CACHE)
   2. 存储(Storage)->池(Pools)->ADD Dataset， 添加一个数据集 -> Edit Permissions-> User和Group都设置为前边创建的用户，并选中 Apply permissions recursively-> SAVE
   3. 共享(Sharing)->Unix (NFS) Shares -> ADD, 选中前边创建的数据集路径，高级模式，Maproot User和Maproot Group都设置为前边创建的用户-> SAVE