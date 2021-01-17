## makefile的自动化变量

```makefile
$< 表示第一个依赖文件,即依赖目标中的第一个目标名字。如果依赖目标是以模式（即%）定义的，那么$<将是符合模式的一系列的文件集。注意是一个一个取出来的。
$^ 表示所有的依赖目标的集合。以空格分隔。如果在依赖目标中有多个重复的，会自动去除重复的依赖目标，只保留一份。
$@ 表示规则中的目标集。在模式规则中，如果有多个目标，那么$@就是匹配于目标中模式定义的集合。
```

## 静态模式

```makefile
<targets ...> : <target-pattern> : <prereq-patterns>
	<commands>
	...
其中 
	targets: 定义了一系列的目标文件，可以有通配符，是目标的一个集合
	target-parrtern 是指明了targets的模式，也就是目标集模式
	prereq-parrterns 是目标的依赖模式，它对target-parrtern形成的模式再进行一次依赖目标的定义。
```
下面看一个例子
```makefile
objs = foo.o bar.o
all: $(objs)

$(objs): %.o : %c
	$(CC) -c $(CFLAGS) &< -o $@ 
	
可以写成下边的样子
%.o:%.c:
	$(CC) -c $(CFLAGS) &< -o $@ 
```

例子中目标从$objs中获取， %.o是变量$objs的模式，及所有以.o结尾的目标。依赖模式是%.c，也就是所有.o文件替换为.c作为依赖的文件名。

## 书写命令

@:省略命令

```makefile
exec:
	@echo "xxx正在编译"
make exec时输出 "xxx正在编译" 而隐藏了命令echo
```

\-：忽略错误，标记不管命令出不出错都认为是成功的

```makefile
clean :
	-rm -f *.o
```

## 使用变量

在makefile中变量是可以使用后边的变量来定义的。

```makefile
foo = $(bar)
bar = $(ugh)
ugh = Huh?

all:
	echo $(foo) 
#执行make输出Huh?
```

好处就是可以把变量的真实值推到后边定义；

缺点是一旦发生嵌套定义就会报错，或者导致不可预知的错误。

### :=

这种方法定义的变量，前面的变量不能使用后边的变量，只能使用前边已经定义的变量。

```makefile
x := foo
y := $(x) bar
x := later
其等价于
y := foo bar
x := later

a := $(b) bar
b := foo
则 a的值是 bar b的值是 foo
```

### ?=

```makefile
FOO ?= bar
其含义是如果FOO没有被定义过，那么变量FOO的值就是bar，如果FOO先前被定义过，那么这条语句什么都不干。
```

### +=

追加变量值

```makefile
objects = main.o foo.o bar.o utils.o
objects += another.o
则$(objects)的值是 main.o foo.o bar.o utils.o another.o
```

## 隐含规则

### 老式风格的“后缀规则”

双后缀: 双后缀规则定义了一对后缀：依赖目标(源文件)的后缀和目标文件的后缀。比如 .c.o相当于 %.o:%.c。

```makefile
.c.o:
	$(CC) -c $(CFLAGS) $(CPPFLAGS) -o $@ $<
```

单后缀：但后缀规则只定义一个后缀。

后缀规则不允许任何的依赖文件，如果有依赖文件的话，那就不是后缀规则，那些后缀统统被认为是文件名。