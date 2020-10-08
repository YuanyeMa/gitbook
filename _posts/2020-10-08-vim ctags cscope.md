---
title: ctags or cscope
---

# vim ctags cscope

## ctags or cscope

[为什么要同时用ctags和cscope](https://www.itranslater.com/qa/details/2131655261931701248)

ctags启用了两个功能：允许您从函数调用跳转到其定义，以及omni完成。

​	 第一个功能意味着当您调用方法时，按`cscope_gen.sh`或`g]`将跳转到定义或实现该方法的位置。 

​	第二个功能意味着当您键入`foo.`或`foo->`时，如果foo是结构，则将显示具有字段完成的弹出菜单。

cscope也有第一个功能 - 使用`cscope_gen.sh` 。 但是，cscope还增加了跳转到调用函数的任何地方的能力。

因此，就跳转代码库而言，ctags只会引导您前往实现函数的位置，而cscope也可以向您显示调用函数的位置。

ctags更容易设置，运行速度更快，如果你只关心单向跳跃，它会显示更少的线条。 您只需运行`cscope_gen.sh`和`g]`即可。 它还可以实现全方位的完整。

Cscope非常适合更大，未知的代码库。 设置很痛苦，因为cscope需要一个包含要解析的文件名列表的文件。 同样在vim中，默认情况下没有设置键绑定 - 您需要手动运行`cscope_gen.sh`。

---



## 安装

```shell
sudo apt-get install ctags 
sudo apt-get install cscope 
```



## ctags 

1. 生成tags

   ```shell
   $ cd  linux/
   $ ctags -R ./
   # 在当前目录生成了tags文件
   kevin→ $ ls -lh tags
   -rw-rw-r-- 1 kevin kevin 392M 10月  8 20:50 tags
   ```

2. 将tags添加到vimrc中

   ```shell
   $ vim ~/.vimrc
   # 写入以下内容
   set tag = /home/kevin/linux/tags
   ```

3. 简单使用

    	光标停留在符号上按`ctrl+]`自动跳转到定义处， `ctrl+t`跳转回调用处。

   ​	 `vim -t start_kernel` 会自动打开打开并跳转。
   
   vim中输入`tselect tag`会列出tag的表单，输入数字选择跳转到哪一处

> ## HOW TO USE WITH VI
>
> Vi will, by default, expect a tag file by the name "tags" in the current directory. Once the tag file is built, the following commands exercise the tag indexing feature:
>
> **vi −t tag** 	Start vi and position the cursor at the file and line where "tag" is defined. 
> **:ta tag**    Find a tag.   
> **Ctrl-]**    Find the tag under the cursor. 
> **Ctrl-T**   Return to previous location before jump to tag (not widely implemented). 

## cscope

配置`cscope`

> 在vim中输入`:help cscope-suggestions`可以查看帮助

```shell
$ vim ~/.vimrc
# 写入以下内容
if has("cscope")
    set csprg=/usr/bin/cscope   "指定用来执行 cscope 的命令
    set csto=0
    set cst					"使用|:cstag|(:cs find g)，而不是缺省的:tag
    set nocsverb 			"不显示添加数据库是否成功
    " add any database in current directory
    if filereadable("cscope.out")
    	cs add cscope.out   "添加cscope数据库
    " else add database pointed to by environment
    elseif $CSCOPE_DB != ""
    	cs add $CSCOPE_DB
    endif
    set csverb 	"显示添加成功与否
endif
```

生成符号数据库

```shell 
$ cd linux
$ make cscope
```



```shell
#使用下面的命令生成代码的符号索引文件：
$ cscope -Rbkq
#这个命令会生成三个文件：cscope.out, cscope.in.out, cscope.po.out。

#其中cscope.out是基本的符号索引，后两个文件是使用"-q"选项生成的，可以加快cscope的索引速度。上面命令的参数含义如下：
# -R: 在生成索引文件时，搜索子目录树中的代码
# -b: 只生成索引文件，不进入cscope的界面
# -d: 只调出cscope gui界面，不跟新cscope.out
# -k: 在生成索引文件时，不搜索/usr/include目录
# -q: 生成cscope.in.out和cscope.po.out文件，加快cscope的索引速度
# -i: 如果保存文件列表的文件名不是cscope.files时，需要加此选项告诉cscope到哪儿去找源文件列表。可以使用"-"，表示由标准输入获得文件列表。
# -I dir: 在-I选项指出的目录中查找头文件
# -u: 扫描所有文件，重新生成交叉索引文件
# -C: 在搜索时忽略大小写
# -P path: 在以相对路径表示的文件前加上的path，这样，你不用切换到你数据库文件所在的目录也可以使用它了。    

#在缺省情况下，cscope在生成数据库后就会进入它自己的查询界面，一般不用这个界面，所以使用了"-b"选项。如果已经进入了这个界面，按CTRL-D退出。 
```



```shell
cscope 命令:
add  ：添加一个新的数据库             (用法： add file|dir [pre-path] [flags])
find ：查询一个模式                   (用法： find a|c|d|e|f|g|i|s|t name)
       a: Find assignments to this symbol
       c: Find functions calling this function
       d: Find functions called by this function
       e: Find this egrep pattern
       f: Find this file
       g: Find this definition
       i: Find files #including this file
       s: Find this C symbol
       t: Find this text string
help ：显示此信息                     (用法： help)
kill ：结束一个连接                   (用法： kill #)
reset：重置所有连接                   (用法： reset)
show ：显示连接                       (用法： show)
```





```shell
有英文注释的我就不说明了，我就说一下里边的键map映射
如： nmap <C-\>s :cs find s <C-R>=expand("<cword>")<CR><CR>
nmap 表示在vim的普通模式下，即相对于：编辑模块和可视模式，以下是几种模式
        :map            普通，可视模式及操作符等待模式
        :vmap           可视模式
        :omap           操作符等待模式
        :map!           插入和命令行模式
        :imap           插入模式
        :cmap           命令行模式
<C-\>表示：Ctrl+\
s表示输入(即按：s)s
: 表示输入':'
“cs find s"表示输入"cs find s"也即是要输入的命令
<C-R>=expand("cword")总体是为了得到：光标下的变量或函数。cword 表示：cursor word, 类似的还有：cfile表示光标所在处的文件名
将下面的内容添加到~/.vimrc中, 并重启vim:
nmap <C-_>s :cs find s <C-R>=expand("<cword>")<CR><CR>
nmap <C-_>g :cs find g <C-R>=expand("<cword>")<CR><CR>
nmap <C-_>c :cs find c <C-R>=expand("<cword>")<CR><CR>
nmap <C-_>t :cs find t <C-R>=expand("<cword>")<CR><CR>
nmap <C-_>e :cs find e <C-R>=expand("<cword>")<CR><CR>
nmap <C-_>f :cs find f <C-R>=expand("<cfile>")<CR><CR>
nmap <C-_>i :cs find i ^<C-R>=expand("<cfile>")<CR>$<CR>
nmap <C-_>d :cs find d <C-R>=expand("<cword>")<CR><CR>
当光标停在某个你要查找的词上时, 按下<C-_>g, 就是查找该对象的定义, 其他的同理.
```





## 参考文献
[cscope官网](http://cscope.sourceforge.net/)
[Cscope的使用（领略Vim + Cscope的强大魅力）](https://blog.csdn.net/dengxiayehu/article/details/6330200)
[ctags](http://ctags.sourceforge.net/)
[Ubuntu下查看Linux内核源码(vim+ctags)](https://blog.csdn.net/w_linux/article/details/72784989)
[vim插件ctags的安装和使用](https://blog.csdn.net/G_BrightBoy/article/details/16830395)
[[VIM使用(二) 浏览内核源代码](https://www.cnblogs.com/linux-sir/p/4675919.html)](https://www.cnblogs.com/linux-sir/p/4675919.html)
[定义cscope快捷键](http://biancheng.dnbcw.net/linux/347492.html)