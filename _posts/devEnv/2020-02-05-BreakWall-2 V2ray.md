# BreakWall 2 - V2ray


[参考](https://www.4spaces.org/digitalocean-build-v2ray-0-1/)以及[官方文档](https://www.v2ray.com/chapter_00/install.html)

## 1. 下载安装脚本

``` shell
wget https://install.direct/go.sh # 2020-10-5 : 此方法已经废止
wget https://raw.githubusercontent.com/v2fly/fhs-install-v2ray/master/install-release.sh
```

## 2. 安装

```shell
chmod u+x ./go.sh
./go.sh #如果无法从GitHub下载软件，可以执行 ./go.sh -l ./v2ray-xxx-xx.zip从本地安装

# 2020-10-5 新方法
chmod u+x install-release.sh
sudo ./install-release.sh
```

> Linux服务端和客户端都通过此脚本进行安装，服务端和客户端的区别在于配置文件不同。
>
> Windows和macOS版本客户端都有GUI根据参考1进行配置，主要根据服务端填写 port\id\level\alterId几项。

## 3. 配置

配置文件默认路径： ~~```/etc/v2ray/config.json```~~   `/usr/local/etc/v2ray/config.json`

服务端配置

```json
{
  "inbounds": [{
    "port": ports, 					// 根据需要修改此处port
    "protocol": "vmess",
    "settings": {
      "clients": [
        {
          "id": "uuid", 		// 此处一般由服务器自动生成，需要复制粘贴进客户端
          "level": 1, 
          "alterId": 64
        }
      ]
    }
  }],
  "outbounds": [{
    "protocol": "freedom",
    "settings": {}
  },{
    "protocol": "blackhole",
    "settings": {},
    "tag": "blocked"
  }],
  "routing": {
    "rules": [
      {
        "type": "field",
        "ip": ["geoip:private"],
        "outboundTag": "blocked"
      }
    ]
  }
}
```

客户端配置

```json
{
  "inbounds": [{
    "port": 1080,  													// SOCKS 代理端口，在浏览器中需配置代理并指向这个端口
    "listen": "127.0.0.1",
    "protocol": "socks",
    "settings": {
      "udp": true
    }
  }],
  "outbounds": [{
    "protocol": "vmess",
    "settings": {
      "vnext": [{
        "address": "server_ip", 						// 服务器地址，请修改为你自己的服务器 ip 或域名
        "port": server_port, 	 							// 服务器端口
        "users": [{ "id": "server_uuid" }] 	// 复制服务器配置文件的UUID
      }]
    }
  },{
    "protocol": "freedom",
    "tag": "direct",
    "settings": {}
  }],
  "routing": {
    "domainStrategy": "IPOnDemand",
    "rules": [{
      "type": "field",
      "ip": ["geoip:private"],
      "outboundTag": "direct"
    }]
  }
}
```

## 4. 启动服务

``` shell
启动 ： service  v2ray start 或者 systemctl start v2ray
停止 ： service  v2ray stop 或者 systemctl stop v2ray
重启 ： service  v2ray restart 或者 systemctl restart v2ray
设置开机启动： systemctl enable v2ray
```
### ubuntu关闭ufw

```shell
#查看防火墙规则
$ sudo ufw status verbose
#关闭防火墙，禁用开机自启
$ sudo ufw disable
```

## 5. 设置国内中转

由于运营商国际互连出口拥堵造成访问代理服务器速度巨慢，所以想通过一台国内的VPS走不同的线路访问国外代理服务器的方式实现正常的BreakWall。
以下配置文件是国内的VPS的配置项目。
```json
{
  "inbounds": [
  {
    "port": //监听的端口号,
    "protocol": "保持和客户端一致的协议",
    "settings": {
      "clients": [ 
        { //要和客户端的配置文件中的user保持一致
          "id": "uuid 要和客户端保持一致",
          "level": 1,
          "alterId": 64
        }
      ]
    }
  }],
  "outbounds": [{
    "protocol": "vmess",
    "settings": {
        "vnext": [
            {
                "address" : "国外代理服务器的ip",
                "port" : 国外代理服务的监听端口,
                "users" : [ //保持和国外代理服务器inbounds的user内容一样
                    {
                        "id":"国外代理服务器的uuid",
                        "alterId" : 64 //额外ID也要保持一致
                    }
                ]
            }
        ]
    }
  },{
    "protocol": "blackhole",
    "settings": {},
    "tag": "blocked"
  }],
  "routing": {
    "rules": [
      {
        "type": "field",
        "ip": ["geoip:private"],
        "outboundTag": "blocked"
      }
    ]
  }
}
```
