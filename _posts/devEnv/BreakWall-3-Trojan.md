

Trojan的主页：https://trojan-gfw.github.io/trojan/

# 步骤一： 购买、部署VPS

推荐使用**[Vultr](https://www.vultr.com/?ref=8838164)**, 因为“便宜”。

我安装的是“CentOS 8” ，自带了BBR，因此不用再手动设置。

# 步骤二：申请域名，设置DNS解析，申请证书

我使用的是**腾讯云**的域名和免费的证书。

# 步骤三：部署并配置Nginx

```shell
yum install nginx
nginx -h # 查看nginx的命令帮助
nginx # 启动nginx
nginx -t # 检查配置文件是否有效
nginx -s stop # 关闭nginx
nginx -s reload # 重新加载配置文件
```

nginx默认的配置文件在`/etc/nginx/nginx.conf`

## 设置防火墙

```c
firewall-cmd  --permanent --add-port=80/tcp
firewall-cmd  --permanent --add-port=443/tcp
firewall-cmd --list-port 
    80/tcp 443/tcp  # 配置生效
```

# 步骤四：部署并配置Trojan

```shell
sudo bash -c "$(curl -fsSL https://raw.githubusercontent.com/trojan-gfw/trojan-quickstart/master/trojan-quickstart.sh)"
或者
sudo bash -c "$(wget -O- https://raw.githubusercontent.com/trojan-gfw/trojan-quickstart/master/trojan-quickstart.sh)"
```

安装完成后配置文件路径： `/usr/local/etc/trojan/config.json`

```json
# 配置文件中需要修改的地方
    "password": [
        "trojan的连接密码"
    ],
# 以及
"ssl": {
        "cert": "证书crt文件的路径",
        "key": "证书key文件的路径",
        "key_password": "证书的密码",
}
```

启动：`systemctl  start trojan`

重启：`systemctl  restart trojan`

查看状态：`systemctl  status  trojan`

关闭：`systemctl  stop trojan`

开机自启：`systemctl  enable  trojan`



# 步骤五 :  客户端设置

我Windows端使用的是**V2rayN V3.27**，

服务器 -> 添加[Trojan]服务器 -> 

- 服务器地址：VPS的IP地址
- 服务器端口 ： 443
- 密码 ： 步骤四中配置的“trojan的连接密码”
- 域名(SNI) : 填写证书的域名



