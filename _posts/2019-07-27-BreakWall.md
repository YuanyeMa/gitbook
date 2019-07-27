# 搭建shadowsocks服务以及启用Google BBR加速



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

