# C++ Primer Plus笔记



## 第十章：对象和类



类和结构(class和struct)的区别：结构的默认访问类型是public，而类的默认访问类型是private。    

所创建的每个新对象都有自己的存储空间，用于存储其内部变量和类成员；但同一个类的所有对象共享同一组类方法，即每种方法只有一个副本。  

### 内联函数 

内联函数是C++为提高程序运行速度所做的改进，常规函数和内联函数之间的主要区别不在于编写方式，而是编译方式。常规函数调用的时候需要执行函数调用操作(参数入栈，设置返回点，跳转到函数入口等)，而内联函数在编译的时候直接在调用处展开函数代码，在执行的时候并没有进行函数调用。  

因此，多次调用同一个内联函数的时候，编译器会在内联函数被调用处展开多个函数实例，和常规函数调用相比占用更多的内存空间。  

内联函数适用于函数调用开销比函数执行开销还要大的场合，**内联函数不能递归**。    

与宏的区别：宏只是字符串替换，没有函数传递的性质，内联函数也是函数，可以传递参数。    

其*定义*位于类声明中的函数都将自动成为内联函数(下边的`set_tot`函数)  

``` c++
#ifndef STOCK_H
#define STOCK_H
#include <string>
class Stock
{
  private:
  	std::string company;
  	long shares;
  	double total_val;
  	double share_val;
  	void set_tot() {total_cal = shares * share_val;}
  public:
  	void acquire(const std::string & co, long n, double pr);
}
#endif
```

也可以在类声明外定义内联函数  

```c++
inline void Stock::set_tot()
{
	total_val = shares*share_val;
}
```



### 构造函数

C++提供了两种使用构造函数来初始化对象的形式，第一种是显式的调用构造函数：

```c++
Stock food = Stock("World Cabbage", 250, 1,25); //根据编译器编译规则不同，可能会创建临时对象
stock1 = Stock("Nifty Foods", 10, 50.0); //赋值，总是会在赋值前创建一个临时对象，效率更低
```

另外一种是隐式的调用：

``` c++
Stock garment("Furry Mason", 50, 2.5);
```



此外还有默认构造函数和拷贝构造函数  

默认构造函数：未提供显式初始值时，下边是使用默认构造函数的三种形式。  

``` c++
Stock fluffy_the_cat;
Stock first = Stock();
Stock * prelief = new Stock;
```

> 当且仅当没有定义任何构造函数的时候，编译器才会提供默认构造函数；为类定义了构造函数后就必须为它提供默认构造函数。如果定义了非默认参数的构造函数，而没用定义默认构造函数则上边的声明会报错。

默认构造函数可以通过两种方式定义：直接定义一个不接收参数的构造函数；定义提供默认参数的构造函数。  

``` c++
Stock::Stock()
{
  company = "no name";
  shares = 0;
  share_val = 0.0;
  total_val = 0.0;
}
或者
Stock::Stock(const string & co = "Error", int n = 0, double pr = 0.0);
C++11还可以通过参数列表的形式定义
```

拷贝构造：也叫复制构造，用于将一个对象复制到新创建的对象中。下边四种方式都是拷贝构造。每当程序生成了对象副本时，编译器都将使用拷贝构造函数。编译器生成临时对象时，也将使用拷贝构造函数。

``` c++
StringBad ditto(motto);
StringBad metoo = metoo;
StringBad also = StringBad(motto);
StringBad * pStringBad = new StringBad(const StringBad &);
```

### const成员函数

``` c++
const Stock land = Stock("Kludge");
land.show();
```

第二行的`land.show()`编译的时候会出错，因为`show()`方法可能会更改`land`对象的属性，而`land`对象又被`const`所修饰。正确的做法是，在声明和定义`show()`方法的时候将其定义为`const`成员函数，像下边这样。  

``` c++
void show() const;
void Stock::show() const
{
  ....
}
```

### 对象数组

``` c++
const int STKS = 10;
Stock stocks[STKS] = {
  Stock("aefaf", 12,5, 20),
  Stock(),
  Stock("dsfafefa", 130, 3.25)
};
```

初始化对象数组的方案是，首先使用默认构造函数创建数组元素，然后花括号中的构造函数将创建临时对象，然后将临时对象的内容复制到相应的元素中。因此，要创建对象数组，则**这个类必须有默认构造函数**。

## 第十一章：使用类



### 运算符重载

语法格式：`operatorp(argument-list)`其中最后一个`p`表示要重载的运算符，比如`operator+(Time)`

> 函数重载：函数重载的关键是函数的参数列表—也称为函数特征标(function signature)。如果两个函数的参数数目和类型相同，同时参数的排列顺序也相同，则他们的特征标相同，而变量名是无关紧要的。函数名相同而特征标不同的函数是重载函数。
>
> double func(int a, double b) 和 double func(double a, int b)也是重载函数，后边有实验证明。

运算符重载的时候**左侧**的操作数应该是调用对象，比如`A = B*2.75`等价于`A = B.operator*(2.75)`。而`A = 2.75*B`会出错。  

要实现`A = 2.75*B`可以通过**友元函数**来实现。  

```c++
class Time
{
  private:
  	...
  public:
  	...
    friend Time operator*(double m, const Time & t); 
};
```

这样再调用`A = 2.75*B；` 就等价于` A = operator*(2.75, B)； ` 

关于函数重载和左值是调用对象做了一个实验。  

```c++
#include <iostream>

double func(int a, double b)
{
	std::cout<<"fir i"<<std::endl;
	return a+b;
}

double func(double b, int a)
{
	std::cout<<"fir dou"<<std::endl;
	return a+b;
}

int main()
{
	std::cout<<func(1, 2.3)<<std::endl; 
	std::cout<<func(2.3, 1)<<std::endl; 
	return 0;
}
```

上边程序的输出如下，符合预期。  

```
$ ./a.out
fir i
3.3
fir dou
3.3
```

但是把第二个func函数的定义注释掉之后，使用clang编译的时候会出现`./test.cpp:19:18: warning: implicit conversion from 'double' to 'int' changes value from 2.3 to 2 [-Wliteral-conversion]
        std::cout<<func(2.3, 1)<<std::endl;`的警告。运行的时候会输出

``` c++
$ ./a.out
fir i
3.3
fir i
3 //把2.3转换成了int型的2,所以出错了。
```

由此证明前边说的，参数列表中参数的顺序不同代表的是不同的函数。

### 类的自动转换和强制类型转换

## 第十二章：类和动态内存分配

### 静态类成员函数

使用`static`关键字将类成员函数声明为静态的。  

声明为静态的类成员函数不能通过对象调用，也不能使用this指针。因为静态成员函数不与特定的对象相关联，因此只能使用静态数据成员。可以直接使用`类名::函数名()`调用，静态类数据成员的使用：`类名::变量名`。



---

# C++ Primer

## 第九章：顺序容器(sequential-container)

顺序容器是按照他们在容器中的位置来顺序保存和访问的  

顺序容器 

- vector : 可变大小数组，支持快速随机访问，在尾部之外的位置插入或者删除元素可能会很慢。
- deque：双端队列
- list ： 双向链表，只支持双向顺序访问
- forward_list：单向链表，只支持单向顺序访问
- array：固定大小数组，支持快速随机访问，不能添加或者删除元素
- string：与vector相似的容器，但专门用于保存字符，随机访问快。

> array和forward是C++新标准增加的类型

额外三个顺序容器适配器

- stack
- queue
- Priority_queue

> 适配器（adaptor）是标准库中的一个通用概念。容器、迭代器和函数都有适配器。本质上，一个适配器是一种机制，能使某种事物的行为看起来像另外一种事物一样。一个容器能接受一种已有的容器类型，使其行为看起来像另外一种不同的类型。例如stack适配器使用一种顺序容器，使其操作起来像一个stack一样。每个适配器都在其底层顺序容器类型之上定义了一个新的接口。

## 第十一章：关联容器(associative-container)

关联容器中的元素是按照关键字来保存和访问的  

主要有两个 

​		map: 元素是key-value对

​		set：每个元素只包含一个关键字，支持高效的关键字查询操作：检查一个给定的关键字是否在set中

共有八中形式  

- map : 关联数组：保存关键字-值对
- set ： 关键字即值，即只保存关键字的容器

- multimap ： 关键字可以出现多次的 关键字-值 对

- multiset ： 关键字可以重复出现多次的set

以下四个是无序的集合

- unordered_map : 用哈希函数组织的map

- unordered_set :  用哈希函数组织的set

- unordered_multimap : 哈希函数组织的map，且关键字可以重复出现多次

- unordered_multiset ： 哈希函数组织的set, 且关键字可以重复出现多次

> map和multimap定义在头文件map中  
>
> set和multiset定义在头文件set中  
>
> 无序容器定义在unordered_map和unordered_set中。

使用关联容器的一个例子

```c++
#include <map>
#include <set>
#include <string>
using namespace std;

map<string, size_t> word_count;
set<string> exclude = {"The", "But", "And", "Or", "An", "A", "the"};

string word;
while (cin>>word)
  	if (exclude.find(word) == exclude.end()) //find调用返回一个迭代器，如果给定关键字在set中，迭代器指向该关键字，否则find返回尾后迭代器。
      ++word_count[word];
```





---

# 侯婕-STL标准库和泛型编程

STL的六大部件:

- 容器（Containers）
- 分配器 (Allocators) 
- 算法 (Algorithms)
- 迭代器 (Adapters)
- 仿函式 (Functors)

![image-20190727221102649](/Users/kevin/Library/Application Support/typora-user-images/image-20190727221102649.png)

下边一个程序用了六大组件。

``` c++
# include <vector>
# include <algorithm>
# include <functional>
# include <iostream>
using namespace std;
int main()
{
  int ia[6] = {27, 210, 12, 47, 109, 83};
  vector<int, allocator<int> > vi(ia, ia+6); 
  // vector是容器， allocator是分配器，负责底层内存的分配。
  count<<count_if(vi.begin(), vi.end(), not1(bind2nd(less<int>(), 40)));
  // vi.begin()和vi.end()是迭代器， count_if是算法：满足后边条件的元素数目
  // not1(否1，数字1)和bind2nd(绑定第二个参数：40)都是函数适配器，less是一个仿函数
  // 即输出大于等于40的数字个数
  return 0;
}
```

