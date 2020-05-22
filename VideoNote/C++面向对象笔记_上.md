



# C++面向对象开发上

> 培养正规的、大气的编程习惯

## 0. 面向对象三大特征 —— 封装、继承、多态

### 封装
- 把客观事物封装成抽象的类，并且类可以把自己的**数据和方法**只让**可信的类或者对象操作**，对**不可信的进行信息隐藏**。

###  继承
- 基类（父类）——> 派生类（子类）

### 多态
- 多态，是**以封装和继承为基础**，使得消息可以**多种形式显示**。

#### 0) C++ 多态分类及实现：

1. **重载多态**（Ad-hoc Polymorphism，编译期）：**函数、运算符**重载
2. **子类型多态**（Subtype Polymorphism，运行期）：**虚函数**
3. **参数多态性**（Parametric Polymorphism，编译期）：**类模板、函数模板**
4. **强制多态**（Coercion Polymorphism，编译期/运行期）：基本**类型转换**、自定义类型转换
   The Four Polymorphisms in C++

#### 1) 静态多态（编译期/早绑定） 函数重载
```cpp
class A
{
public:
    void do(int a);
    void do(int a, int b);
};
//参数多态性
template <typename T>
class Complex{
	T re, im;
public:
    Complex(T re, T im) {}
};
Complex<int>(5, 4);
Complex<float>(5.0, 4.0);
```
#### 2) 动态多态（运行期期/晚绑定）虚函数
- 虚函数：

  用 virtual 修饰**成员函数（含基类的虚构函数）**，使其成为虚函数.

- 非虚函数：

> 1. 普通（全局）函数（非类成员函数）
> 2. 静态函数（static）
> 3. 构造函数（因为在调用构造函数时，虚表指针并没有在对象的内存空间中，必须要**构造函数调用完成后才会形成虚表指针**）
> 4. **内联函数**不能是表现多态性时的虚函数，解释见：虚函数（virtual）可以是内联函数（inline）吗？
>

##  一、C++编程简介

### 基础知识

> 曾学过procedural language (C 语言最佳)，知道如何对程序编译、链接、执行。
>
> 变量(variables)
>
> 类型(types) : int, float, char, struct …
>
> 作用域(scope)
>
> 循环(loops) : while, for,
>
> 流程控制: if-else, switch-case



#### 基于对象分类：

1. 基于对象：一个class的编程 object based

2. 面向对象：几个class的编程 object oriented




#### class的经典分类：

1. class without pointer members ——>e.g: **complex 复数**
2. class with pointer members  ——>e.g: **string 字符串**

#### class之间的关系：

1. 继承inheritance、
2. 复合composition
3. 委托delegation

#### C++书籍（STL是标准库的前身）

基础：Language
《C++Primer》
《C++programming Language》
提高：Standard Library
《Effective C++ Third Edition》及中文
《The C++ Standard Library》
《STL源码剖析》

##  二、头文件与类的声明
```cpp

//flie XXX.h
#ifndef __complex__
#define __complex__


#program once //编译器宏


// class的布局：

class base; //前置声明

class header{// class header
    // class body
     ...
}

#endif //XXX.h end

// XXX.cpp
#include"XXX.h"   //c
#include<cstdio>  //C++ 
#include<iostream>
```
## 三、构造函数

### （1）inline内联函数：

>特征（程序员期望能inline，编译器决定是否inline）
>1. 相当于把内联函数里面的内容**写在调用内联函数处**；
>1. 相当于不用执行进入函数的步骤，**直接执行函数体**；
>1. 相当于宏，却比宏多了类型检查，真正**具有函数特性**；
>1. 编译器一般**不内联**包含**循环、递归、switch** 等复杂操作的内联函数；
>1. 在类声明中定义的函数，**除了虚函数**的其他函数都会自动隐式地当成内联函数。

```cpp
//--------------------------声明-----------------------------------------------
// 声明1（加 inline，建议使用）
inline int functionName(int first, int second,...);
// 声明2（不加 inline）
int functionName(int first, int second,...);’

//--------------------------定义-----------------------------------------------
// 类内定义并实现，隐式内联
class A {
    int doA() { return 0; }         // 隐式内联
}
// 类外定义，需要显式内联
class A {
    int doA();
    int functionName(int first, int second,...);
}
inline int A::doA() { return 0; }   // 需要显式内联
inline int functionName(int first, int second,...) {/** ...**/};// 需要显式内联
```
####  编译器对 `inline` 函数的处理步骤
>1.	将 `inline` 函数体**复制**到 `inline` 函数**调用点处**；
>2.	为所用` inline `函数中的局部变量**分配内存**空间；
>3.	将` inline `函数的输入**参数和返回值映射到调用方法的局部变量**空间中；
>4.	如果` inline `函数有多个返回点，将其转变为` inline `函数代码块末尾的分支（使用GOTO）

#### 优缺点 **主要与宏定义**比较
>优点
    >1.	内联函数同宏函数一样将在被调用处进行代码展开，省去了参数压栈、栈帧开辟与回收，结果返回等，从而**提高程序运行速度**。
    >2.	内联函数相比宏函数来说，在代码展开时，会做**安全检查或自动类型转换**（同普通函数），而宏定义则不会。
    >3.	在类中声明同时定义的成员函数，自动转化为内联函数，因此内联函数**可以访问类的成员变量**，宏定义则不能。
    >4.	内联函数在运行时**可调试**，而宏定义不可以。

>缺点
    >1.	**代码膨胀**。内联是以代码膨胀（复制）为代价，消除函数调用带来的开销。如果执行函数体内代码的时间，相比于函数调用的开销较大，那么效率的收获会很少。另一方面，每一处内联函数的调用都要复制代码，将使程序的**总代码量增大，消耗更多的内存空间**。
    >2.	inline 函数**无法随着函数库升级**而升级。inline函数的改变需要重新编译，不像 non-inline 可以直接链接。
    >3.	是否内联，**程序员不可控**。内联函数只是对编译器的建议，是否对函数内联，决定权在于编译器。

###  （2）access level 访问级别 

> `public` 成员：可以被任意实体访问
> `private` 成员：只允许被**本类**的成员函数访问
> `protected` 成员：只允许被**子类**及本类的成员函数访问（虚函数使用较多）
>
> `private`：数据的部分用尽量用`private`
> `public`：函数的部分，大部分用`public`

### （3）构造函数 

创建一个对象的时候，构造函数自动被调用，构造函数可设置默认参数，并设置参数初始化列表：

```cpp
pair(const T1& a, const T2& b) : first(a), second(b) {}
```
参数`initializition list`和在`body`里对参数赋值的区别：

> 一个是参数初始化；
> 一个是赋值，是一个执行的过程，多了计算量；

创建一个对象，可以有参数，也可以无参数，也可动态创建：

```cpp
class complex *p = new complex(4); //动态创建
```
class定义了多个构造函数，就是重载`overloading`

### （4）friend 友元类和友元函数：
>1. **能访问私有成员**
>2. 破坏封装性
>3. 友元关系不可传递
>4. 友元关系的单向性
>5. 友元声明的形式及数量不受限制
>6. 相同class 的各objects 互friends (友元)
>
>```cpp
>#include <iostream> 
>class A { 
>   	friend class B; // Friend Class 
>   }; 
>   
>   class B { 
>   public: 
>   	void showA(A& x) { 
>   		std::cout << "A::a=" << x.a; 
>   	} 
>   }; 
>   //OR 
>   class B; 
>   class A { 
>   public: 
>   	void showB(B&); 
>   }; 
>   
>   class B { 
>   	friend void A::showB(B& x); // Friend function 
>   }; 
>   
>   void A::showB(B& x) { 
>   	std::cout << "B::b = " << x.b; 
>   } 
> ```
> 

item23.	宁以 non-member、non-friend 替换 member 函数（可增加封装性、包裹弹性（packaging flexibility）、机能扩充性）

```cpp
//member
class WebBrowser {
public:
    void clearCache();
    void clearHistory();
    void removeCookies();

    void clearEverything();  //调用上述的三个函数
};
//non-member
void clearBrowser (WebBrowser& wb)
{
    wb.clearCache();
    wb.clearHistory();
    wb.removeCookies();
}
```
### （5）static ：

```cpp
class Account {
public:
    static double m_rate;
    static void set_rate(const double& x) { m_rate = x; }
};
double Account::m_rate = 8.0; //static value

int main() {
    Account::set_rate(5.0);
    Account a;
    a.set_rate(7.0);
}
```

### 利用static实现单例模式（Singleton）

```cpp
// Meyers Singleton
class A {
public:
    static A& getInstance();
    setup() { ... }
    A() = delete;    // c++ 11
	A(const A& rhs) = delete;  // c++ 11
private:
    A();   //before c++ 11
    A(const A& rhs);  //before c++ 11
};
A& A::getInstance()
{
    static A a;
    return a;
}

// Singleton
class A {
public:
    static A& getInstance(){ return a;} //???? 猜测结果
    static A& getInstance( return a; ); //???? ppt中
    setup() { ... }
private:
    A();
    A(const A& rhs);
    static A a;
};


//call  多线程中不安全.
A::getInstance().setup();

A& p = A::getInstance();
p.setup();

```

## 四、参数传递和返回值以及const

**尽量使用**传引用的方式才传递参数，因为引用的底层是指针（C语言中指针作为参数传递类似），传递的数据大小为4个字节，速度会很快。同样，**值的返回也尽量返回引用**（如果可以的话）。

> 1. 数据放在`private`里
> 2. 参数用`reference`，是否用`const` 
> 3. 在类的`body`里的函数是否加`const`
> 4. 构造函数的 `initial list`
> 5. return by reference,不能为local object.
>

####   指针&引用：
> 1.  都是地址的概念；指针是一个实体,指向一块内存，它的内容是所指内存的地址；引用则是某块内存的别名。 
> 2.  使用sizeof指针本身大小一般是（4），而引用则是被引用对象的大小； 
> 3.  引用不能为空，指针可以为空；指针可以被初始化为NULL，而引用必须被初始化且必须是一个已有对象的引用,之后不可变；指针可变；引用“从一而终”，指针可以“见异思迁”； 
> 4.  作为参数传递时，指针需要被解引用才可以对对象进行操作，而直接对引用的修改都会改变引用所指向的对象； 
> 5.  引用没有`const`，指针有`const`，`const`的指针不可变；
> 具体指没有`int& const a`这种形式，而`const int& a`是有的，前者指引用本身即别名不可以改变，这是当然的，所以不需要这种形式，后者指引用所指的值不可以改变）
> 6.  指针在使用中可以指向其它对象，但是引用只能是一个对象的引用，不能 被改变；
> 7.  指针可以有多级指针（**p），而引用至于一级； 
> 8.  指针和引用使用自增(++)运算符的意义不一样； 
> 9.  如果返回动态内存分配的对象或者内存，必须使用指针，引用可能引起内存泄露。 
> 10. 引用是类型安全的，而指针不是 (引用比指针多了类型检查）

####  const作用

当一个方法不会对数据进行修改时，尽量将方法指定为const。细分顶层const、底层const

> "effective c++"第三条讲到： 只需要判断const是在 * 的左边还是右边即可。左边则是修饰被指物，即被指物是常量，不可以修改它的值；右边则是修饰指针，即指针是常量，不可以修改它的指向；在左右两边，则被指物和指针都是常量，都不可以修改。

> 1. 修饰变量，说明该变量不可以被改变；
> 1. 修饰指针，分为指向常量的指针和指针常量；
> 1. 常量引用，经常用于形参类型，即避免了拷贝，又避免了函数对值的修改；
> 1. 修饰成员函数，说明该成员函数内不能修改成员变量。
>

```cpp
 //const 使用
 // 类
 class A
 {
 private:
     const int a;                // 常对象成员，只能在初始化列表赋值
 public:
     // 构造函数
     A() : a(0) { };	        // 初始化列表
     A(int x) : a(x) { };        // 初始化列表
     // const可用于对重载函数的区分
     int getValue();             // 普通成员函数
     int getValue() const;       // 常成员函数，不得修改类中的任何数据成员的值
     // 可以供常对象使用，否则报错，调用有可能被改变值，编译器报错。
 };
 
 void function()
 {
     // 对象
     A b;                        // 普通对象，可以调用全部成员函数
     const A a;                  // 常对象，只能调用常成员函数、更新常成员变量
     const A *p = &a;            // 常指针
     const A &q = a;             // 常引用
     // 指针
     char greeting[] = "Hello";   
     char* p1 = greeting;                // 指针变量，指向字符数组变量
     const char* p2 = greeting;          // 指针变量，指向字符数组常量
     char* const p3 = greeting;          // 常指针，指向字符数组变量
     const char* const p4 = greeting;    // 常指针，指向字符数组常量
 }
 // 函数
 void function1(const int Var);           // 传递过来的参数在函数内不可变
 void function2(const char* Var);         // 参数指针所指内容为常量
 void function3(char* const Var);         // 参数指针为常指针
 void function4(const int& Var);          // 引用参数在函数内为常量
 // 函数返回值
 const int function5();      // 返回一个常数
 const int* function6();     // 返回一个指向常量的指针变量，使用：const int *p = function6();
 int* const function7();     // 返回一个指向变量的常指针，使用：int* const p = function7();
 
```


## 五、操作符重载与临时对象

> 1、成员函数带有**隐藏的参数`this`**，谁调用这个函数谁就是 this.**临时对象** complex() ->typename()；
> 2、在类外Complex Complex::operator+=(complex &c2)  这个是成员函数 `operator+=` 的实现，所以需要 Complex:: 具有this指针。例如：

返回& 为了链式连续操作，在stream流的类中非常常见

### 运算符重载——成员函数

```cpp
inline complex&
__doapl(complex* ths, const complex& r)
{ //第一參數將會被改動   //第二參數不會被改動
   ths->re += r.re;
   ths->im += r.im;
return *ths;
}

inline complex&
class complex::operator += (const complex& r) //成员函数 this
{
  return __doapl (this, r);
}
```

### 运算符重载——全局函数

```cpp
//而下面属于运算符重载，不是成员函数的时候，就没有Complex:: 。
inline complex
operator - (const complex& x, double y)
{
  return complex (real (x) - y, imag (x));
}
```

## 六、总结

1.使用**初始化列表**，构造函数中，Complex(double r = 0, double i = 0) : re(r), im(i) {}

2.成员函数是否指明const，若不修改数据，则尽可能指明const。**const对象无法调用非const成员方法**。

3.**参数传递和结果返回，尽量使用传递和返回引用**。效率更高（不要返回局部变量的引用）。

4.**数据private，对外接口public，内部方法private**（注意封装来隐藏数据）。

5.同一个类衍生出的**对象之间都互为友元**，可以直接调用对方的私有变量。

6.**操作符重载**都是作用在左边的变量上的（即调用该操作符函数的对象），注意**连续调用**的情况，操作符重载可以是成员方法，也可全局方法。

7.成员方法在类定义中实现，默认声明为inline方法（视编译器的决定）。若为全局函数，需加inline关键字，让编译器尽量的将其变为inline函数。

## 七、三大函数：拷贝构造，拷贝赋值，析构
构造函数（可以重载，类内创建private，C++11有delete）
拷贝构造函数
拷贝赋值函数
析构函数

*常量成员函数 const修饰成员函数，防止常量对象进行调用出错。*

class with pointer members 必須有copy ctor 和copy op=

一定要在operator= 中檢查是否self assignment

```cpp
inline String& String::operator=(const String& str) //后面有String的整体代码
{
    if (this == &str)
        return *this;
    delete[] m_data;
    m_data = new char[ strlen(str.m_data) + 1 ];
    strcpy(m_data, str.m_data);
    return *this;
}

#include <iostream.h>
//定义成员方法，获取m_data指针，不修改数据，加上const
inline char * String::get_c_str() const {
    return this->m_data;
}
//重载操作符<<
ostream& operator<<(ostream& os, const String& str)
{
    os << str.get_c_str();
    return os;
}
```

——》**类含指针**，就需要 **拷贝构造、 拷贝赋值函数**

>浅拷贝——》拷贝构造函数 ，浅拷贝的影响：
1、造成内存泄漏；	
2、造成有两个指针 指向同一块内存

>深拷贝——》拷贝赋值函数 
>步骤：delete ;new; strcpy;
>class里面有默认的拷贝构造和拷贝赋值函数。如果自己不定义一个拷贝构造函数，在调用拷贝构造函数的时候，就会调用**默认的浅拷贝构造函数**，就会造成问题，所以一定要自己定义拷贝构造函数——深拷贝。
>
>![img](C++%E9%9D%A2%E5%90%91%E5%AF%B9%E8%B1%A1%E7%AC%94%E8%AE%B0_%E4%B8%8A.assets/%E6%9E%84%E9%80%A0%E5%87%BD%E6%95%B0%E6%B5%85%E6%8B%B7%E8%B4%9D.png)

## 八、堆，栈与内存管理
1、static local objects的生命周期

> - static的生命周期 ：object的对象在scope结束以后仍然存在，直到整个程序结束；
> - 非static 的生命周期：object的对象在在scope结束以后就结束了。
> - global objects的生命周期：对象 objects 生命结束，就是什么时候析构函数被调用。
>

2、new——》operator new。

new动态创建对象，分三步：

> 1. 先转化为operator new 函数，申请分配内存。
> 2. 做类型转化。
> 3. 调用构造函数

delete ——》operator delete。删除对象，分两步：

> 1. 先调用析构函数，
> 2. 再调用operator delete函数。

3、带中括号[ ]的new[ ]叫做array new，带中括号[ ]的delete[ ]  叫做array delete。

>  动态分配所得到的数组array：complex *p = new complex[3];
>
> new []  ——》delete[] ——》表示调用几次析构函数
>  new 字符串 ——》delete 指针
>
> delete[n] :array new一定要调用array delete，delete[n]会调用n次析构函数，而delete仅调用一次。
>

**Stack**，是存在于某作用域(scope) 的一块内存空间(memory space)。例如当你调用函数，函数本身即会形成一个stack 用来放置它所接收的**参数**，以及**返回地址**。在函数本体(function body) **内声明的任何变量**，其所使用的内存块都取自上述stack。

**Heap**，是指由操作系统提供的一块**global 内存空间**，程序可动态分配(dynamic allocated) 从某中获得若干区块(blocks),记得释放，尽量用RAII管理。

### new和delete关键字的工作流程

```cpp
{
    class Complex* p = new Complex;
    ...
    delete p; //若未删除，内存泄露，再也无法删除了，p退出作用域，作用域外无法看到p。
    
    //编译器实现如下
    Complex *pc;
    void* mem = operator new( sizeof(Complex) ); //分配内存
    pc = static_cast<Complex*>(mem); //转型，static_cast多了类型检查
    pc->Complex::Complex(1,2); //构造函数
    
    Complex::~Complex(pc); // 析构函數
    operator delete(pc); // 释放內存
}
```

```cpp
{
    String* ps = new String("Hello"); 
    //编译器实现如下
    String* ps;
    void* mem = operator new( sizeof(String) ); //分配内存
    ps = static_cast<String*>(mem); //转型
    ps->String::String("Hello"); //构
}
```

### VC当中的内存分配

**注：仅限new的情况下。**

**Debug（左）Release（右）模式下：**

**![img](C++%E9%9D%A2%E5%90%91%E5%AF%B9%E8%B1%A1%E7%AC%94%E8%AE%B0_%E4%B8%8A.assets/VC%E5%BD%93%E4%B8%AD%E7%9A%84%E5%86%85%E5%AD%98%E5%88%86%E9%85%8Ddebug-1590131562489.png)    ![img](C++%E9%9D%A2%E5%90%91%E5%AF%B9%E8%B1%A1%E7%AC%94%E8%AE%B0_%E4%B8%8A.assets/VC%E5%BD%93%E4%B8%AD%E7%9A%84%E5%86%85%E5%AD%98%E5%88%86%E9%85%8Drelease-1590131565296.png)**

**浅绿色：**Complex对象所占实际空间，大小为8bytes。

**上下砖红色：**各4bytes，一共8bytes。是cookie，用来保存总分配内存大小，以及标志是给出去还是收回来。例如00000041，该数为16进制，4表示64，即总分配内存大小为64,1表示给出去（0表示收回来）。

**灰色：**Debug模式下使用的额外空间，前面32bytes，后面1bytes，一共36bytes。

**深绿色：**内存分配大小必须是16的倍数（这样砖红色部分里的数字最后都是0，可以用来借位表示给出去还是收回来），所以用了12byte的填充（padding）。

**同样，String对象的空间分配，如图：（左Debug，右Release）**

![img](C++%E9%9D%A2%E5%90%91%E5%AF%B9%E8%B1%A1%E7%AC%94%E8%AE%B0_%E4%B8%8A.assets/string%E5%86%85%E5%AD%98%E5%88%86%E9%85%8Ddebug.png)    ![img](C++%E9%9D%A2%E5%90%91%E5%AF%B9%E8%B1%A1%E7%AC%94%E8%AE%B0_%E4%B8%8A.assets/string%E5%86%85%E5%AD%98%E5%88%86%E9%85%8Drelease.png)

**Debug（左）Release（右）模式下，数组空间的分配：**

![img](C++%E9%9D%A2%E5%90%91%E5%AF%B9%E8%B1%A1%E7%AC%94%E8%AE%B0_%E4%B8%8A.assets/%E6%95%B0%E7%BB%84%E5%86%85%E5%AD%98%E5%88%86%E9%85%8Ddebug-1590131593574.png)    ![img](C++%E9%9D%A2%E5%90%91%E5%AF%B9%E8%B1%A1%E7%AC%94%E8%AE%B0_%E4%B8%8A.assets/%E6%95%B0%E7%BB%84%E5%86%85%E5%AD%98%E5%88%86%E9%85%8Drelease.png)

**灰色：**即3个Complex对象的大小，每个是8bytes，一共24bytes。

**深绿色：**填充为16的倍数。

**前后白色：**51表示80bytes，“给出去”。

**黄色：**Debug模式额外占用空间。

**中间白色：**用一个整数表示数组中对象个数。

## 九、复习String类的实现过程

```cpp
class String
{
public:
    String(const char* cstr = 0)
    {
        if (cstr) {
            m_data = new char[strlen(cstr)+1];
            strcpy(m_data, cstr);
        }else { // 未指定初值
            m_data = new char[1];
            *m_data = '\0';
        }
    }
    String(const String& str)
    {
        m_data = new char[ strlen(str.m_data) + 1 ];
        strcpy(m_data, str.m_data);
    }
    String& operator=(const String& str);
    ~String()
    {
        delete[] m_data;
    }
    char* get_c_str() const { return m_data; }
private:
    char* m_data;
};


inline String& String::operator=(const String& str)
{
    if (this == &str)
        return *this;
    delete[] m_data;
    m_data = new char[ strlen(str.m_data) + 1 ];
    strcpy(m_data, str.m_data);
    return *this;
}
```

## 十、扩展补充：类模板，函数模板，及其他

```cpp
//class template,类模板
template<typename T>
class complex
{ public:
    complex (T r = 0, T i = 0): re (r), im (i){ }
    complex& operator += (const complex&);
    T real () const { return re; }
    T imag () const { return im; }
private:
    T re, im;
    friend complex& __doapl (complex*, const complex&);
};
//函数模板
template <class T>
inline const T& min(const T& a, const T& b)
{
    return b < a ? b : a;
}

class stone
{ public:
    stone(int w, int h, int we)
        : _w(w), _h(h), _weight(we){ }
    bool operator< (const stone& rhs) const
    { return _weight < rhs._weight; }
private:
    int _w, _h, _weight;
};

//引数推导的结果，T 为stone，于是调用stone::operator<
//调用函数，那么就会传实参，编译器就会进行实参推导。
stone r1(2,3), r2(3,3), r3;
r3 = min(r1, r2);
```

    （1）static ：静态数据 属于所有对象（类）。

            静态函数 **没有this pointer**，而非静态函数有 this pointer，可以用this去取数据，**静态函数要处理数据只能处理静态数据**。**静态数据一定要在class外面定义**。

    （2）template：类模板  函数模板
    （3）inline namespace：(C++11)

```cpp
namespace Program {
  namespace Version1 {
    int getVersion() { return 1; }
    bool isFirstVersion() { return true; }
  }
  inline namespace Version2 {//inline
    int getVersion() { return 2; }
  }
}

int version {Program::getVersion()};              // Uses getVersion() from Version2
int oldVersion {Program::Version1::getVersion()}; // Uses getVersion() from Version1
bool firstVersion {Program::isFirstVersion()};    // Does not compile when Version2 is added
```

## 十 一、组合  has-a 与继承 is-a

组合表示has-a。即A类里有B类的对象（非指针）**实心，实在成员**对象（UML）。

![image-20200518193613371](C++%E9%9D%A2%E5%90%91%E5%AF%B9%E8%B1%A1%E7%AC%94%E8%AE%B0_%E4%B8%8A.assets/has-a.png)

```cpp
由内而外template <class T>
struct Itr {
    T* cur;
    T* first;
    T* last;
    T** node;
    //Sizeof : 4 * 4
};

template <class T>
class deque {
protected:
    Itr<T> start;
    Itr<T> finish;
    T** map;
    unsigned int map_size;
    //Sizeof : 16 * 2 + 4 + 4
};

//适配器模式(queue 适配器，deque成熟的类)
template <class T>
class queue {
protected:
    deque<T> c;
    //Sizeof : 40
};
//组合
//构造顺序： Itr->deque->queue 由内到外
//析构顺序： queue->deque->Itr 由外到内
//queue内存大小：queue的内存大小+deque的内存大小

```
（2）委托`delegation`，即composition by reference：在body中声明一个带指针的另一个类 composition by reference 生命时间：  classA 用一个**指针**指向classB，需要的时候才调用classB，而不是一直拥有classB。叫做**“Copy on write”**

![img](C++%E9%9D%A2%E5%90%91%E5%AF%B9%E8%B1%A1%E7%AC%94%E8%AE%B0_%E4%B8%8A.assets/%E5%A7%94%E6%89%98%E5%9F%BA%E4%BA%8E%E5%BC%95%E7%94%A8%E7%9A%84%E5%A4%8D%E5%90%88.png)

**指针有点虚（UML）**

```cpp
//委托delegation
// file String.hpp
class StringRep;
class String {//Handle,稳定
private:
    StringRep* rep; // pimpl ，指针指向实现的类（Handle/Body）
};

// file String.cpp
#include "String.hpp"
namespace {
class StringRep {//Body ,有弹性变化
    friend class String;  //friend
    StringRep(const char* s);
    ~StringRep();
    int count;
    char* rep;
};
}
String::String(){ ... }
```
  （3）继承Inheritance：（三种继承方式：public protected private）is-a，继承主要搭配**虚函数**来使用函数的继承：指的是继承函数的调用权，**子类可以调用父类的函数**，如B继承A，则说明B是A的一种。

![img](C++%E9%9D%A2%E5%90%91%E5%AF%B9%E8%B1%A1%E7%AC%94%E8%AE%B0_%E4%B8%8A.assets/InheritanceUML.png)

```cpp
//继承中的     构造函数与析构调用顺序
//构造顺序： 父类->子类  由内到外
//析构顺序： 子类->父类  由外到内

//继承+组合模式  构造函数与析构调用顺序
Derived::Derived(...):Base(),Component() { ... };
Derived::~Derived(...){ ...  ~Component(), ~Base() };
```

继承+组合模式

```cpp
//继承+组合模式  构造函数与析构调用顺序
Derived::Derived(...):Base(),Component() { ... };
Derived::~Derived(...){ ...  ~Component(), ~Base() };
```

## 十二、虚函数与多态

```cpp
//虚函数 你期待derived class的行为
non-virtual 函数：不重新定义(override, 覆写它).
virtual 函数：重新定义(override, 覆写) 它，且你对它已有默认定义。
pure virtual 函数：必须重新定义(override 覆写)它，你对它没有默认定义，不能直接实例化对象。
```

（1）虚函数：virtual 纯虚函数：一定要重新定义。

 （A）Inheritance + composition下的构造和析构

 （B）delegation + Inheritance ——》 功能最强大的一种

## 十三、设计模式

> 建议看李建忠老师视频23种C++设计模式

图形展示

![image-20200519233845305](C++%E9%9D%A2%E5%90%91%E5%AF%B9%E8%B1%A1%E7%AC%94%E8%AE%B0_%E4%B8%8A.assets/%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8FTemplate%20Method.png)

![image-20200519235739912](C++%E9%9D%A2%E5%90%91%E5%AF%B9%E8%B1%A1%E7%AC%94%E8%AE%B0_%E4%B8%8A.assets/%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8FObserver.png)

![image-20200520000157555](C++%E9%9D%A2%E5%90%91%E5%AF%B9%E8%B1%A1%E7%AC%94%E8%AE%B0_%E4%B8%8A.assets/%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F_%E7%BB%84%E5%90%88%E6%A8%A1%E5%BC%8F.png)

![image-20200520000235749](C++%E9%9D%A2%E5%90%91%E5%AF%B9%E8%B1%A1%E7%AC%94%E8%AE%B0_%E4%B8%8A.assets/%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8FPrototype.png)



### Template Method模式

步骤：

> 1.在父类CDocument中，实现共同的方法，例如OpenFile、CloseFile等。
>
> 2.CDocument中，将读文件内容的方法Serialize设计为虚函数或纯虚函数。
>
> 3.CMyDoc继承CDocument，实现Serialize()。
>
> 4.使用子类CMyDoc调用父类方法OnFileOpen()，按图中灰色曲线的顺序来调用内部函数。

这样就实现了关键功能的延迟实现，实现应用与架构分离（Application framework）这就是典型的Template Method。

**为什么会有灰色曲线的调用过程：**

1.当子类myDoc调用OnFileOpen()的时候，实际上对于编译器是CDocument::OnFileOpen(&myDoc);因为谁调用，this指针就指向谁，所以调用这个函数，myDoc的地址被传进去了。

2.当OnFileOpen()函数运行到Serilize()的时候，实际上是运行的this->Serialize()；由于this指向的是myDoc，所以调用的是子类的Serilize()函数。

```cpp
//框架开发人员
class CDocument {
public:
    void OnFileOpen() { 
        cout << "dialog..." << endl;
        cout << "check file status..." << endl;
        cout << "open file..." << endl;
        Serialize();                       //子类再实现
        cout << "close file..." << endl;
        cout << "update status..." << endl;
    }
    //父类的虚函数，当然这里是纯虚函数也是可以的，virtual void Serialize() = 0
    virtual void Serialize() {}
};
//应用开发人员
class CMyDoc :public CDocument {
public:
    //这里实现了父类的虚函数Serialize()
    virtual void Serialize() {
        cout << "MyDoc Serialize..." << endl;
    }
};

//call
#include "CDocument.h"
int main() {
    CMyDoc mc;
    mc.OnFileOpen();
    return 0;
}
```

output

```
dialog...
check file status...
open file...
MyDoc Serialize...
close file...
update status...
```



```cpp
//程序库开发人员
class Library
{
  public:
    //稳定 template method
    void Run()
    {
        Step1();
        Step2();//支持变化 ==> 虚函数的多态调用
        Step3();
        Step4(); //支持变化 ==> 虚函数的多态调用
        Step5();
    }
    virtual ~Library() {}

  protected:
    void Step1(){}//稳定
    void Step3(){}//稳定
    void Step5(){}//稳定

    virtual bool Step2() = 0; //变化
    virtual void Step4() = 0; //变化
};

//应用程序开发人员
class Application : public Library
{
  protected:
	virtual bool Step2(){    //... 子类重写实现
		return true;
	}

	virtual void Step4(){     //... 子类重写实现
	}
};

int main()
{
	Library *pLib = new Application();
	pLib->Run();
	delete pLib;
}
```

### Observer模式

我们的数据设计在类Subject中，窗口（观察者）设计为Observer，这是一个父类，可以被继承（即可以支持派生出不同类型的观察者）。

![image-20200519234006414](C++%E9%9D%A2%E5%90%91%E5%AF%B9%E8%B1%A1%E7%AC%94%E8%AE%B0_%E4%B8%8A.assets/%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8FObserver-1590131631061.png)

**用如下代码来实现：**

```cpp
//数据类
class Subject {
    int m_value;
    vector<Observer*> m_views;
public:
    void attach(Observer* obs) { m_views.push_back(obs);
    }
    void set_value(int value) {
        m_value = value;
        notify();
    }
    void notify() {
        for (int i = 0;i < m_views.size();++i) {
            m_views[i]->update(this, m_value);
        }
    }
};
//观察者基类
class Observer {
public:
    //纯虚函数，提供给不同的实际观察者类来实现不同的特性
    virtual void update(Subject*, int value) = 0;
};
```

用图形来描述：

![img](C++%E9%9D%A2%E5%90%91%E5%AF%B9%E8%B1%A1%E7%AC%94%E8%AE%B0_%E4%B8%8A.assets/%E8%A7%82%E5%AF%9F%E8%80%85%E7%B1%BBUML.png)

### 组合模式

　　在计算机文件系统中，有**文件夹**的概念，文件夹里面既可以放入文件也可以放入文件夹，但是文件中却不能放入任何东西。文件夹和文件构成了一种递归结构和容器结构。
　　虽然文件夹和文件是不同的对象，但是他们都可以被放入到文件夹里，所以一定意义上，文件夹和文件又可以看作是同一种类型的对象，所以我们可以把文件夹和文件统称为目录条目（directory entry）。在这个视角下，文件和文件夹是同一种对象。
　　所以，我们可以将文件夹和文件都看作是目录的条目，将容器和内容作为同一种东西看待，可以方便我们递归的处理问题，在容器中既可以放入容器，又可以放入内容，然后在小容器中，又可以继续放入容器和内容，这样就构成了容器结构和递归结构。
　　这就引出了composite模式，也就是组合模式，组合模式就是用于创造出这样的容器结构的。是容器和内容具有一致性，可以进行递归操作。

图中Primitive代表基本的东西，即文件。Composite代表合成物，即文件夹。Component表示目录条目。

Primitive和Composite都是一种Component，而Composite中可以存放其他的Composite和Primitive，所以Composite中的Vector存放的类型时Component指针，也就包含了Primitive和Composite两种对象的指针。

**代码框架如下：**

```cpp
//一个比较抽象的类，相当于目录条目
class Component {
    int value;
public:
    Component(int val) :value(val){}
    virtual void add(Component*) {}
};
//相当于 文件类
class Primitive {
public:
    Primitive(int val):Component(val){}
};
//相当于 文件夹类
class Composite {
    vector<Component*> c;
public:
    Composite(int val) :Component(val){}

    void add(Component* elem) {
        c.push_back(elem);
    }
};
```

### Prototype模式

**设计应用架构时，并不知道以后实现的子类名称，但有要提供给Client调用子类的功能怎么办？**

例如十年前设计的架构，子类在十年后继承父类并实现功能。Client只能调用架构中的父类，如何通过父类调用到不知道名字的子类对象。使用Prototype模式：

```cpp
#include <iostream>
#include <vector>
using namespace std;

//可能是十年前写的框架，我们不知道子类的名字，但又希望通过该基类来产生子类对象
class Prototype {
    //用于保存子类对象的指针（让子类自己上报）
    static vector<Prototype *> vec;
public:
    //纯虚函数clone，让以后继承的子类来实现，也是获取子类对象的关键
    virtual Prototype* clone() = 0;
    //子类上报自己模板用的方法
    static void addPrototype(Prototype* se) {
        vec.push_back(se);
    }
    //利用该基类在vec中查找子类模板，并且通过模板来克隆更多的子类对象
    static Prototype* findAndClone(int idx) {
        return vec[idx]->clone();
    }

    //子类实现自己操作的函数，hello()只是个例子
    virtual void hello() const = 0;
};
//定义静态vector，很重要，class定义中只是声明
vector<Prototype *> Prototype::vec;

//十年后实现的子类，继承了Prototype
class ConcreatePrototype : public Prototype{
public:
    //用于在Prototype.findAndClone()中克隆子类对象用
    Prototype * clone() {
        //使用另一个构造函数，为了区分创建静态对象的构造函数，添加了一个无用的int参数
        return new ConcreatePrototype(1);
    }

    //子类实现的具体操作
    void hello() const {
        cout << "hello" << endl;
    }

private:
    //静态属性，自己创建自己，并上报给父类Prototype
    static ConcreatePrototype se;
    //上报静态属性给父类
    ConcreatePrototype() {
        addPrototype(this);
    }
    //clone时用的构造方法，参数a无用，只是用来区分两个构造方法
    ConcreatePrototype(int a) {}
};
//定义静态属性，很重要，有了这句，才会创建静态子类对象se
ConcreatePrototype ConcreatePrototype::se;
```

步骤：

1.子类继承Prototype父类，定义静态属性的时候，自己创建一个自己的对象，此时调用的是无参数的构造函数。并将创建好的自己的指针通过addPrototype(this)上传给基类的vector容器保存。

2.基类定义好的纯虚函数clone()，由子类实现，并在其中通过另一个构造函数产生对象并返回。

3.在Client端，使用基类的findAndClone()，获取vector中的子类对象模板的指针，来调用子类对象的clone功能，返回一个新的子类对象，调用多次则可创建多个对象供用户使用。

4.创建出的子类对象可以调用在子类中实现的hello()方法，进行想要的操作。

```cpp
Prototype* p = Prototype::findAndClone(0);
p->hello();
```

## 附录\*、overload override overwrite的区别

> 深入请参考C++ primer

**1. Overload（重载）**

　在同一作用域中，定义了多个同名不同参数（类型或者个数）函数。特征：

**（1）**同一作用域；
**（2）**函数名字相同；
**（3）**参数不同，底层const算重载；
**（4）**virtual 关键字可有可无。

**2. Override（覆盖）**

用来实现C++多态性的，子类改写父类的virtual函数。

**通过`override`显示声明，**使得程序员的意图更加清晰的同时让编译器可以为我们发现一些错误。

**（1）**不同的范围（分别位于派生类与基类）；
**（2）**函数名字相同；
**（3）**参数列表完全相同；
**（4）**基类函数必须有virtual 关键字。

**通常情况下，覆盖函数必须与虚函数的参数类型及返回类型相同**：例外是，当类的虚函数返回类型是类本身的**指针或引用**时

**3. Overwrite（改写）**

基类与派生类之间同名函数重载：派生类重写函数名**屏蔽了基类**中的同名函数。

> 解决办法：**在派生类中通过using为父类函数成员提供声明**

**（1）**若派生类的函数与基类的函数同名，不同参数。基类的函数将被隐藏。
**（2）**若派生类的函数与基类的函数同名，也同参数，但基类函数无virtual关键字。基类函数被隐藏。

```cpp
class Derived : public Base {
 public:
  using Base::print;//解决办法
  void print() {
    cout << "print() in Derived." << endl;
  }
};
```

【1】大纲是Gayhub上热心网友的，进行了部分补充，非常感谢共享 。

​      因个人水平有限，欢迎大家指导。   yzhu798#gmail.com 。2020.03.07