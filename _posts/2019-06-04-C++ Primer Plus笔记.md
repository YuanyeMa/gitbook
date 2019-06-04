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

