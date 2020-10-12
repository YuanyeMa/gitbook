# shell中fork、exec和source的区别


## 前言
在脚本中引用其他脚本有多种方式，直接执行./xx.sh，source xx.sh或者exec xx.sh,但是三种方式的本质是不通的。学过linux高级编程 或者对linux进程创建方式比较熟悉的理解起来可能比较容易一些。

## 简介
###  ./xx.sh
fork 创建一个子进程（sub-shell），然后在子进程中运行调用的脚本，子进程继承父进程的资源（环境变量等等），在子进程中修改父进程的环境变量后，子进程退出后，并不影响父进程。写脚本的时候直接./xx.sh就是采用这种方式。

### . ./xx.sh 或者 source xx.sh
source   让 script 在当前 shell 内执行、 而不是产生一个 sub-shell 来执行。因此source执行的脚本中可以更改父进程的环境变量。

### exec ./xx.sh
exec 也是在原有进程中执行，但是，原有进程被终止。exec是用子进程替换父进程进行运行，所有父进程中exec后边的程序得不到运行。

## 注意
source和exec区别在于，原有进程是否被终止

## 实例分析
`vim 1.sh` 写入以下内容
``` shell
1#!/bin/bash                                                                                       
2 A=B
3 echo "PID for 1.sh before exec/source/fork:$$"
4 export A
5 echo "1.sh: \$A is $A"
6 case $1 in
7     exec)
8         echo "using exec..."
9         exec ./2.sh ;;
10     source)
11         echo "using source..."
12         #. ./2.sh ;;
13         source ./2.sh;;
14     *)
15         echo "using fork by default..."
16         ./2.sh ;;
17 esac
18 echo "PID for 1.sh after exec/source/fork:$$"
19 echo "1.sh: \$A is $A"

```
`vim 2.sh` 写入以下内容
```shell
1 #!/bin/bash                                                                                        
2 echo "PID for 2.sh: $$"
3 echo "2.sh get \$A=$A from 1.sh"
4 A=C
5 export A
6 echo "2.sh: \$A is $A"
```
分别运行  
`./1.sh  fork`  
`./1.sh  source`  
`./1.sh  exec`  
运行结果
>\* ~/dep/selfcode  *  
  root→ # ./1.sh  fork    
PID for 1.sh before exec/source/fork:31099  
1.sh: $A is B  
using fork by default...  
PID for 2.sh: 31100  
2.sh get $A=B from 1.sh  
2.sh: $A is C  
PID for 1.sh after exec/source/fork:31099  
1.sh: $A is B  


>\* ~/dep/selfcode  *  
  root→ # ./1.sh  source  
PID for 1.sh before exec/source/fork:31105  
1.sh: $A is B  
using source...  
PID for 2.sh: 31105  
2.sh get $A=B from 1.sh  
2.sh: $A is C  
PID for 1.sh after exec/source/fork:31105  
1.sh: $A is C  

>\* ~/dep/selfcode  *  
  root→ # ./1.sh  exec   
PID for 1.sh before exec/source/fork:31110  
1.sh: $A is B  
using exec...  
PID for 2.sh: 31110  
2.sh get $A=B from 1.sh  
2.sh: $A is C     
  注意：父进程后边的代码没有运行

以上除了fork，父进程和子进程的PID不同外，source和exec父进程和子进程的PID相同，  
但是，source相当于在父进程执行过程中插入了一段子进程的代码运行，  
而exec是用子进程的代码替换掉了父进程的空间。
