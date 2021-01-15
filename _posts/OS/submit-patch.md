# 提交kernel patch

## 创建工作分支
```shell
# 添加linux-next远程分支
git remote add linux-next https://git.kernel.org/pub/scm/linux/kernel/git/next/linux-next.git

git  fetch --tags linux-next 
# 基于next-20210112标签创建工作分支
git branch mybranch next-20210112
# 检出到分支mybranch
git checkout mybranch
```

修改代码

## 提交修改
```shell

# 添加修改的文件
git add Documentation/fpga/dfl.rst
# 查看log 学写 commit
git log Documentation/fpga/dfl.rst
# 提交
git commit 
```

## 制作patch


```shell
# 针对最近1个commit制作patch 
git format-patch -s -1
# 检查patch是否符合要求
./scripts/checkpatch.pl  0001-Documentation-fpga-dfl-fix-syntax-errors-for-dfl.patch 
# 如果不符合要求就撤销commit，重新修改。
git reset HEAD~1
```

## 发送patch
### 配置msmtp

```shell
sudo apt install msmtp
vim ~/.msmtprc
写入以下内容
# default
account gmail
protocol smtp
host smtp.gmail.com
from xxxxxx@gmail.com
user xxxxxx@gmail.com
password xxxxxxx
port 587 
auth on
tls on
tls_trust_file /etc/ssl/certs/ca-certificates.crt
syslog LOG_MAIL

# set a default account
account default : gmail
```
> 此外，还需在 Google账号->安全性->安全性较低的应用的访问权限->启用

### 发送patch
```shell 
# 查找maintainer
./scripts/get_maintainer.pl  0001-Documentation-fpga-dfl-fix-syntax-errors-for-dfl.patch

# 使用邮件发送patch
git send-email -to xxx@xxx.com ./0001-Documentation-fpga-dfl-fix-syntax-errors-for-dfl.patch
```

