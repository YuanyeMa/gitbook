# 使用gitbook记笔记
```shell
$ sudo apt update
$ sudo apt install nodejs npm

$ nodejs  --version
v10.19.0

$ npm config set registry http://registry.npm.taobao.org/ # 设置npm源为淘宝的源
# 设置回官方的源 npm config set registry http://registry.npmjs.org/
$ sudo npm install gitbook-cli -g
```

# 常用命令

```shell
git clone blog
cd  blog
gitbook install ./
cd ../
gitbook build /root/blog  /var/www/html/_book
```

