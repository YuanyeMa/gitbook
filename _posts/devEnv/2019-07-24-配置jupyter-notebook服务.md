# 配置jupyter-notebook服务

```shell
$jupyter notebook --generate-config #生成配置文件
$conda install nb_conda #
```

设置`jupyter Notebook`密码的方式

```shell
# 先打开python终端
$ python
# 然后生成密码
In[1]: from IPython.lib import passwd
In[2]: passwd()
Enter password:
Verify password:
Out[2]: 'sha1编码后的密码'
# 复制上边编码后的密码，退出python终端
```

设置服务器配置文件

``` shell
$vim ~/.jupyter/jupyter_notebook_config.py
# 配置文件中有几项内容需要修改。
c.NotebookApp.allow_remote_access = True
c.NotebookApp.allow_root = False
c.NotebookApp.ip = '*'
c.NotebookApp.notebook_dir = '这里填jupyter notebook的工作目录'
c.NotebookApp.open_browser = False
c.NotebookApp.password = u'这里填上边复制的加密后的密码'
c.NotebookApp.port = 8888
c.NotebookApp.terminals_enabled = False
```

安装完`Anaconda`利用`conda`创建的虚拟环境，但是启动`jupyter notebook`之后却找不到虚拟环境。

实际上是由于在虚拟环境中缺少`kernel.json`文件，解决方法如下

```shell
# 首先安装ipykernel
$conda install ipykernel
# 然后在虚拟环境中安装ipykernel
$conda install -n 环境名称 ipykernel
# 激活虚拟环境
$conda activate 环境名称
# 将环境写入notebook的kernel中
$python -m ipykernel install --user --name 环境名字 --display-name "要显示的环境名字"

# 此外，以后再创建虚拟环境的时候，最好直接装上ipykernel
$conda create -n 环境名称 python=版本 ipylernel

# 删除kernel环境的命令如下
$jupyter kernelspec remove 环境名称
```

安装完成后可以通过命令`jupyter notebook`启动服务，然后在工作的电脑的浏览器中输入`ip:port`输入密码就可以登录使用了。

