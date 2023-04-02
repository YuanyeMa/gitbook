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



# 搭建阅读内核环境的一些补充


> 本节内容摘自《ARM Linux源码剖析》

1. 下载解压内核

2. 安装`ctages` 和 `cscope`

   ```shell
   $ sudo apt update 
   $ sudo apt install ctags cscope
   ```

3. 生成标签

   `ctags`程序通过程序代码生成标签，用以生成函数、变量、类成员、宏等的索引，以管理数据库。

   使用`ctags`生成标签的命令是`ctags -R`。内核源码树中提供了用于生成各种架构`ctags`标签的脚本`./scripts/tags.sh`, 使用命令 `make tags ARCH=arm`可以生成`arm`架构的内核源码标签。

   ```shell 
   $ ls ./scripts/tags.sh
   ./scripts/tags.sh
   $ make tags ARCH=arm
     GEN     tags
   $ ls -lah tags
   -rw-rw-r-- 1 kevin kevin 89M Nov 14 16:36 ./tags  # 生成的89MB的tags文件
   ```



> `tags`文件结构
>
> ```shell
> tart_kernel    include/linux/start_kernel.h    /^extern asmlinkage void __init start_kernel(void);$/;" p
> start_kernel    init/main.c     /^asmlinkage void __init start_kernel(void)$/;" f
> 
> tags_name<TAB>file_name<TAB>ex_cmd;<TAB>extension_fields
> # tags_name 	符号名		
> # file_name 	符号所在的文件名	
> # ex_cmd 	在文件中查找符号时，使用vim的ex模式，在此模式中搜索范式的正则表达式
> # extension_field 	符号类型 f=普通C函数， c=类， d=已定义的值
> ```



生成`cscope`数据库

`ctags`只能进行单方向的搜索，无法搜索调用的函数； 无法输出调用该函数的函数；无法输出该函数调用的函数。`cscope`与`ctags`十分相似，但是补充了`ctags`的不足。

```shell
$ make cscope ARCH=arm
  GEN     cscope
$ ls cscope* -lh
-rw-rw-r-- 1 kevin kevin 449K Nov 14 16:39 cscope.files
-rw-rw-r-- 1 kevin kevin 178M Nov 14 16:39 cscope.out
-rw-rw-r-- 1 kevin kevin  34M Nov 14 16:39 cscope.out.in
-rw-rw-r-- 1 kevin kevin 258M Nov 14 16:39 cscope.out.po
# 生成的四个文件
# cscope.files	含有要分析的代码文件列表的文件
# cscope.out 	针对函数位置、函数调用、宏、变量、预处理符号的符号交叉引用文件，是实际的数据库
# cscope.out.in		用-q选项生成数据库时生成的文件，用于提高符号搜索速度的反向索引
# cscope.out.po	用-q选项生成数据库时生成的文件，作用同cscope.in
```



4. vim下插件配置

   `Source Explorer` :  当光标停留在变量名或者函数名时，会显示此变量或者函数所在的文件列表；

   `NERDTree` : 显示源码目录树；

   `Tag List` : 显示文件中的符号列表；

   

​	vim的插件可以到[vim官网](https://www.vim.org/)下载，点击左侧"Scripts" -> "Browse all"在下方出现的搜索窗口中依次搜索需要的插件并下载。

```shell
$ mkdir -p ~/.vim/plugin
$ mv srcexpl.vim ~/.vim/plugin/
$ mv NERD_tree.zip ~/.vim/ && unzip NERD_tree.zip 
$ mv taglist_45.zip ~/.vim/ && unzip taglist_45.zip
```

配置vim插件

``` shell
$ vim ~/.vimrc


"------------------------------------------------"
"ctags database path"
"------------------------------------------------"
set tags=xxxxxx/tags "ctags DB位置"

"------------------------------------------------"
"cscope database path"
"------------------------------------------------"
set csprg=/usr/bin/cscope 	"cscope可执行程序的路径"
set csto=0					"cscope DS search first"
set cst 					"cscope D tag DB search"
set nocsverb				"verbose off"
cs add /home/xxxx/cscope.out /home/code/linux-2.6.30.4
set csverb

"------------------------------------------------"
"Tag List"
"------------------------------------------------"
filetype on
nmap <F7> :TlistToggle<CR>
let Tlist_Ctags_Cmd = "/usr/bin/ctags"
let Tlist_Inc_Winwidth = 0 				"window  width change off"
let Tlist_Exit_OnlyWindow = 0			"tag/file完成选择时taglist window close = off"
let Tlist_Auto_Open = 0
let Tlist_Use_Right_Window = 1


"------------------------------------------------"
"Source Explore"
"------------------------------------------------"
nmap <F8> :SrcExplToggle<CR> 
nmap <C-H> <C-W>h		"向左侧窗口移动"
nmap <C-J> <C-W>j		"向下侧窗口移动"
nmap <C-K> <C-W>k		"向上侧窗口移动"
nmap <C-L> <C-W>l		"向右侧窗口移动"

let g:SrcExpl_winHeight = 8 			"指定SrcExpl Windows高度"
let g:SrcExpl_refreshTime = 100 		"100ms"
let g:SrcExpl_jumpKey = "<Enter>" 		"按Enter跳转到相应的defineition"
let g:SrcExpl_gobackKey = "<SPACE>" 	"back"
let g:SrcExpl_isUpdateTags = 0 			"tag file update = off"


"------------------------------------------------"
"NERD Tree"
"------------------------------------------------"
let NERDTreeWinPos = "left"
nmap <F2> :NERDTreeToggle<CR>	
```



打开`vim`，按`F2`在左侧出现`NERDTree`源码目录树，选择`./init/main.c`。按`Enter`打开源码文件， 按`F8 F9`打开`TagList`和`Source Explorer`窗口。

将光标移动到右侧的`TagList`窗口后，进入vim的ex模式(按Esc)后，输入`:/start_kernel`， `TagList`窗口会自动跳转到`start_kernel`符号处，再按`Enter`，则代码文件窗口会显示定义`start_kernel`的位置。

在代码窗口移动光标到函数或者变量名上时，在下方的`Source Explorer`窗口会显示定义相应符号的文件目录。将光标移动到下方`Source Explorer`窗口,上下选择要浏览的文件，按`Enter`上方代码浏览窗口会显示相应的文件内容， 按`SPACE`会回退。



常用ctags命令

| 命令   | 说明                 |
| ------ | -------------------- |
| ctrl+] | 移动到定义函数的位置 |
| ctrl+t | 回退到跳转前的位置   |

cscope命令格式`: cs find <querytype> <name>`

| querytype | 说明                             |
| --------- | -------------------------------- |
| 0  or s   | 查找c符号                        |
| 1 or g    | 查找定义（definition）           |
| 2 or d    | 查找被该函数调用（called）的函数 |
| 3 or c    | 查找调用该函数的（calling）函数  |
| 4 or t    | 查找文本字符串(text string)      |
| 6 or e    | 查找egrep范式                    |
| 7 or f    | 查找文件                         |
| 8 or i    | 查找用#include包含该文件的文件   |

`：hellp cscope`可以查看更详细的指令用法。

## 参考文献
[cscope官网](http://cscope.sourceforge.net/)
[Cscope的使用（领略Vim + Cscope的强大魅力）](https://blog.csdn.net/dengxiayehu/article/details/6330200)
[ctags](http://ctags.sourceforge.net/)
[Ubuntu下查看Linux内核源码(vim+ctags)](https://blog.csdn.net/w_linux/article/details/72784989)
[vim插件ctags的安装和使用](https://blog.csdn.net/G_BrightBoy/article/details/16830395)
[[VIM使用(二) 浏览内核源代码](https://www.cnblogs.com/linux-sir/p/4675919.html)](https://www.cnblogs.com/linux-sir/p/4675919.html)
[定义cscope快捷键](http://biancheng.dnbcw.net/linux/347492.html)
