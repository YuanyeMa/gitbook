# Docker部署python版本的shadowsocks服务

## ubuntu服务端

- 安装docker
- 使用shadowsocks的docker镜像启动服务
- 更新内核启用Google BBR进行加速

```shell
# 安装docker
apt update
apt install docker 
apt install docker.io
# 拉取镜像
docker pull oddrationale/docker-shadowsocks
# 启动container
docker run -d -p port:port oddrationale/docker-shadowsocks -s 0.0.0 -p port -k passwd -m aes-256-cfb
```
### BBR加速
``` shell
# 下载安装新版本的内核
wget -c http://kernel.ubuntu.com/~kernel-ppa/mainline/v4.11.4/linux-image-4.11.4-041104-generic_4.11.4-041104.201706071003_amd64.deb
# 安装新版本内核
dpkg -i ./linux-image-4.11.4-041104-generic_4.11.4-041104.201706071003_amd64.deb 
# 查看现在使用的内核版本
lsmod | grep bbr
uname -r
# 更新grub使新版本内核开机自启
update-grub
# 卸载多余的软件包
apt autoremove
# 重启系统启用新的内核
reboot
# 查看是否启用了新版本内核，以及是否支持BBR
lsmod | grep bbr
uname -r

# 开启BBR
modprobe  tcp_bbr
echo "tcp_bbr" >> /etc/modules-load.d/modules.conf 
cat /etc/modules-load.d/modules.conf 
echo "net.core.default_qdisc=fq" >>/etc/sysctl.conf 
echo "net.ipv4.tcp_congestion_control=bbr" >>/etc/sysctl.conf 
sysctl  -p 
   
# 查看BBR是否启动
sysctl  net.ipv4.tcp_available_congestion_control  
	# net.ipv4.tcp_available_congestion_control = bbr cubic reno
sysctl  net.ipv4.tcp_congestion_control 
	# net.ipv4.tcp_congestion_control = bbr
sysctl net.core.default_qdisc
	# net.core.default_qdisc = fq
uname -r
	# 4.11.4-041104-generic
lsmod  | grep bbr
	# tcp_bbr                20480  156
```

此外还有[一键安装BBR的脚本](https://raw.githubusercontent.com/teddysun/across/master/bbr.sh)

https://github.com/google/bbr/blob/master/Documentation/bbr-quick-start.md


## CentOS 7的服务端

安装docker
```
yum update 
yum install docker 
service docker start 
```
设置docker自动启动
```
chkconfig docker on
```

使用ncat测试网络是否联通
```
yum install nmap-ncat -y
ncat -l port 
```
在本地使用nc命令连接服务器指定port
```
nc ip port
```
然后可以随便输入一下内容，看另外一端是否能正常收发数据。

拉取镜像并创建container
```
docker pull oddrationale/docker-shadowsocks
docker run -d -p port:port --name name oddrationale/docker-shadowsocks -s 0.0.0.0 -p port -k passwd -m aes-256-cfb
```
查看是否正常监听端口
```
netstat -nalp | grep port
```
设置container随docker自动启动
```
docker update --restart=always <CONTAINER ID>
```
### BBR加速

升级内核

```
rpm --import https://www.elrepo.org/RPM-GPG-KEY-elrepo.org
rpm -Uvh http://www.elrepo.org/elrepo-release-7.0-2.el7.elrepo.noarch.rpm
yum --enablerepo=elrepo-kernel install kernel-ml -y
```
设置grub2默认使用新内核启动
```
egrep ^menuentry /etc/grub2.cfg | cut -f 2 -d \'
sudo grub2-set-default 0
reboot
```
查看是否使用新内核启动
```
uname -r
```
开启BBR
```
echo 'net.core.default_qdisc=fq' | sudo tee -a /etc/sysctl.conf
echo 'net.ipv4.tcp_congestion_control=bbr' | sudo tee -a /etc/sysctl.conf
sysctl -p
```
查看BBR是否开启

```
sysctl net.ipv4.tcp_available_congestion_control
# 输出应为 net.ipv4.tcp_available_congestion_control = bbr cubic reno
sysctl -n net.ipv4.tcp_congestion_control
# 输出应为 bbr
lsmod | grep bbr
# 输出应类似 tcp_bbr  16384  28
```

## 客户端

### sslocal

Linux 安装shadowsocks客户端

```shell
# 使用python版的shadowsocks
$ sudo apt install python3-pip
$ pip install shadowsocks
# 使用
$ sudo vim /etc/shadowsocks/config.json # 创建配置文件，并写入以下内容
{
  "server": "自己shadowsocks服务端ip",
  "server_port": "服务端口",
  "local_port": 1080,
  "password": "密码",
  "timeout": 300,
  "method": "aes-256-cfb" # 加密方式根据自己需要修改
}
$ sslocal -c /etc/shadowsocks/config.json # 启动breakwall
```

此外还有很多种[图形界面版]([https://github.com/Alvin9999/new-pac/wiki/ss%E5%85%8D%E8%B4%B9%E8%B4%A6%E5%8F%B7](https://github.com/Alvin9999/new-pac/wiki/ss免费账号))的。

### Proxychains

`Proxychains`是一个可以让终端命令走代理的软件。

``` shell
# 安装proxychains
$ sudo apt install proxychains
# 修改配置文件
$ sudo vim /etc/proxychains.conf
# 注释掉最后一行的 #socks4 127.0.0.1 9050 
# 添加一行 socks5 127.0.0.1 1080
```

`proxychains`的使用

```shell
$ proxychains wget #后边加一个下载地址
```

关于非`sudo`执行`proxychains`时会有一个错误：
```shell
ERROR: ld.so: object 'libproxychains.so.3' from LD_PRELOAD cannot be preloaded (cannot open shared object file): ignored.
```

解决方法为

- 修改`/usr/bin/proxychains` 中的`export LD_PRELOAD=libproxychains.so.3`为`export /usr/lib/x86_64-linux-gnu/libproxychains.so.3`

- 修改`/usr/lib/proxychains3/proxyresolv` 中的`export LD_PRELOAD=libproxychains.so.3`为`/usr/lib/x86_64-linux-gnu/libproxychains.so.3`

  > 其中`/usr/lib/x86_64-linux-gnu/libproxychains.so.3`可以通过命令`sudo find /usr/ -name "libproxychains.so.3"`找到。

# 安装`simple_obfs`混淆插件

[simple_obfs](https://github.com/shadowsocks/simple-obfs)是一个混淆插件，是把经过`shadowsocks`封装`socks5`协议后的数据包，再封装成正常的`http/tls`数据包的一个插件。

[`shadowsocks-libev`](https://github.com/shadowsocks/shadowsocks-libev)是另一个实现`socks5`代理的软件本质上和前边用的`python`版本的`ssserver/sslocal`没什么区别。但是基于`libev`实现的功能更加强大，不同版本的区别在[这里](https://github.com/shadowsocks/shadowsocks/wiki/Feature-Comparison-across-Different-Versions)。最明显的就是`shadowsocks-libev`同时包含客户端和服务端即`ss-local/ss-server`，而且都可以安装插件。



## Ubuntu

``` shell 
sudo apt update
sudo apt install shadowsocks-libev
```



## CentOS 7

``` shell
yum install git
# 下载shadowsocks-libev的源代码
git clone https://github.com/shadowsocks/shadowsocks-libev.git
cd shadowsocks-libev/
git submodule update --init

# 安装依赖
yum install gcc gettext autoconf libtool automake make pcre-devel asciidoc xmlto c-ares-devel libev-devel libsodium-devel mbedtls-devel -y

# 编译安装
./autogen.sh && ./configure && make && make installl
```

## simple_obfs

```shell
# 安装依赖
sudo yum install gcc autoconf libtool automake make zlib-devel openssl-devel asciidoc xmlto libev-devel
	## Ubuntu 安装依赖： sudo apt-get install --no-install-recommends build-essential autoconf libtool libssl-dev libpcre3-dev libev-dev asciidoc xmlto automake

# 下载源代码
git clone https://github.com/shadowsocks/simple-obfs.git
cd simple-obfs/
git submodule update --init --recursive

# 编译安装
./autogen.sh
./configure && make && make install
```

## 启动配置

服务端：`ss-server -c /etc/shadowsocks-libev/config.json`

客户端： `ss-local -c /etc/shadowsocks-libev/config.json`

## 创建配置文件

``` shell
# 创建配置文件
mkdir /etc/shadowsocks-libev
vim /etc/shadowsocks-libev/config.json

# 服务端写入以下内容
{
     "server":"0.0.0.0",
     "server_port":8388,
     "local_port":1080,
     "password":"passwd",
     "user":"nobody",
     "timeout":600,
     "method":"chacha20-ietf-poly1305",
     "fast_open": false,
     "nameserver":"8.8.8.8",
     "mode":"tcp_and_udp",
     "plugin":"obfs-server",
     "plugin_opts":"obfs=http"
 }

# 客户端写入以下内容
{
    "server":"server_ip",
    "server_port":8388,
    "local_port":1080,
    "password":"passwd",
    "timeout":60,
    "method":"chacha20-ietf-poly1305",
    "plugin":"obfs-local",
    "plugin_opts":"obfs=http;obfs-host=www.bing.com"
}
```

## 设置systemctl托管shadowsocks-libev

``` shell
sudo vim /etc/systemd/system/shadowsocks-libev.service
# 写入以下内容
[Unit]
Description=Shadowsocks-libev Server
Documentation=https://shadowsocks.org/en/
After=network.target

[Service]
Type=simple
ExecStart=/usr/local/bin/ss-server -c /etc/shadowsocks-libev/config.json -u
Restart=on-abort

[Install]
WantedBy=multi-user.target
```

启动服务： `sudo systemctl start shadowsocks-libev`

开机自启动： `sudo systemctl enable shadowsocks-libev`