# 记一次NVIDIA driver/cuda/cudnn升级

系统环境：Ubuntu 18.04 / NVIDIA_Driver version:390 / cuda 9.0 / cudnn 7.1

目标： 升级NVIDIA_Driver version :410.104 /cuda 10.0 / cudnn 7.6 

## NVIDIA Driver

首先关闭桌面系统

``` shell
$ sudo init 3 # 表示以字符界面启动
```

然后屏幕黑了，左上角有个光标在闪烁，没有出来`login`的提示符，这是因为当前窗口是`init 5`即图形界面终端。同时按键盘`Ctrl+Alt+F1`三个按键，进入一个其他终端，然后会出现`login`提示符，输入用户名密码登陆。 因为后续很多操作需要`root`权限，所以可以直接以`root`身份登陆，或者以普通用户身份登陆后用`sudo -i `切换为`root`身份。  

由于我之前安装的390驱动需要gcc-5支持，但是410驱动需要gcc-7，因此需要切换会gcc-7. 

``` shell
# update-alternatives --install /usr/bin/gcc gcc /usr/bin/gcc-7 70
# update-alternatives --install /usr/bin/g++ g++ /usr/bin/g++-7 70
```

> 如果不知道当前gcc是什么版本，可以使用gcc -v命令查看
>
> update-alternatives 是Debian系系统中用于管理不同软件版本的工具。本质上是通过修改软链接的指向来确定使用哪个版本的命令。
>
> 基本用法如下： 
>
> ​		update-alternatives --display <name> # 显示<name>组的情况
>
> ​		update-alternatives --remove <name> <path> # 从<name>组中删除<path>
>
> ​		update-alternatives --remove-all <name> # 删除整个<name>组
>
> ​		update-alternatives --install <link> <name> <path> <priority> # 向<name>组中添加一个路径
>
> ​		update-alternatives --config <name> # 选择一个当前使用的路径
>
> 其中<link>是软连接的位置，对应上边命令中的/usr/bin/gcc； <name> 命令组的名字，可以随便写，对应上边命令中的gcc； <path> 是确切要执行的命令的路径，对应上边命令中的 /usr/bin/gcc-7 ；最后的70是优先级。
>
> 因此上边命令的意思就是创建一个软连接/usr/bin/gcc指向/usr/bin/gcc-7。当执行gcc时调用的是/usr/bin/gcc-7,也就是7.4版本的gcc.

先卸载NVIDIA-390，然后安装NVIDIA-410.

``` shell
# apt purge nvidia* 
进入到驱动安装脚本保存的位置执行安装命令
# ./NVIDIA-Linux-x86_64-410.104.run
```

根据提示下一步就好，其中有一步问是否开启Xorg支持，要选“是”，不然的话，安装完驱动，返回桌面的时候会失败。

安装完成后通过`init 5`命令启动桌面环境。进入桌面环境后可以通过`nvidia-smi`命令查看驱动安装是否正确。

## CUDA 9.0 升级CUDA 10.0

```shell
$sudo cuda_10.0.130_410.48_linux.run
```

接受用户协议，不安装驱动，安装路径默认，安装samples，完成。

设置动态连接库的路径

``` shell 
$ sudo vim /etc/ld.so.conf.d/cuda-10.conf
写入
/usr/local/cuda-10.0/lib64
$ sudo ldconfig
```
> cuda-9.0不用卸载 /usr/local/cuda也是一个软连接安装cuda 10.0的时候已经自动修改指向了cuda-10.0.

## 安装cudnn

``` shell
$ sudo dpkg -i  libcudnn7_7.6.4.38-1+cuda10.0_amd64.deb
$ sudo dpkg -i  libcudnn7-dev_7.6.4.38-1+cuda10.0_amd64.deb
$ sudo dpkg -i  libcudnn7-doc_7.6.4.38-1+cuda10.0_amd64.deb
```

## 测试

``` python 
import tensorflow as tf 
tf.test.gpu_device_name()
```

