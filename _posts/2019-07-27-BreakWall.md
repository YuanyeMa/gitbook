## 服务端

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

## 客户端

- Linux 安装shadowsocks客户端
- Proxychains 配置

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