# C++程序设计（2）

切勿在浮沙

## 一、导读

（1）**泛型编程**和**面向对象编程**分属不同的思维，

（2）由继承所形成的**对象模型**，含`this`指针，`vptr`指针，`vtbl`虚表，**虚机制**，及**虚函数**造成的**多态**。

## 二、conversion function 转换函数

（1）如`operator type() const {}` 的**成员函数**。分数`Fraction`型转成`double`，**无返回类型**，**没有形参**。

```cpp
class Fraction{
    int m_num;    //分子
    int m_den;  //分母
public:
    Fraction(int num, int den=1): m_num(num), m_den(den){ }
    operator double() const { return (double)..;}//转出，const
};
Fraction f(3,5);
double d= 4 + f;//f对象调用operator double() 转成0.6，调用double的+
```

## 三、non-explicit-one-argument constructor 

（1）`explicit`**构造函数明确地调用**，**限制自动转换操作**。

```cpp
class Fraction{
public:
    //单参数构造函数
    Fraction(int num, int den=1): m_num(num), m_den(den){ }
    operator double() const { return (double)..;}//转出，const
    Fraction operator+(const Fraction& f) {return f;}
};
Fraction f(3,5);
double d= 4 + f;//调用non-explicit constructor将4转成Fraction，再operator+
//// explicit Fraction(int num, int den=1)
 explicit Fraction(int num, int den=1){} //该构造函数只做构造函数使用.让编译器不要自动去调用它。
double d= 4 + f;//不能调用成员版+操作
```

（2）**vector<bool>问题**

> `vector<bool>`有两个问题．第一，它不是一个真正STL容器，第二，它并不保存bool类型．

![image-20200419214634490](C++%E9%9D%A2%E5%90%91%E5%AF%B9%E8%B1%A1%E7%AC%94%E8%AE%B0_%E4%B8%8B.assets/image-20200419214634490.png)

## 四、pointer-like classes （关于智能指针与迭代器）

### 4.1关于智能指针

> 类含一个指针，在C++中，->这个符号比较特殊，这个符号除了第一次作用在shared_ptr对象上，返回原始指针px，还会继续作用在px上，用来调用函数method。链式传递的固定写法

```cpp
template<class T>
class shared_ptr{
    T *px;
public:
    T& operator* () const {return *px;}
    T* operator->() const {return px;}
    shared_ptr(T* p): px(p) {}
};//shared_ptr

struct Foo{void method(void) {}}
//call
shared_ptr<Foo> sp(new Foo);
Foo f(*sp);
sp->method();//传递调用px->method();
```

### 4.2关于迭代器

> 作为迭代器，将（结构体或类产生的对象）的指针**包装为一个迭代器**（类指针对象），如链表节点指针，在内部重载多个操作符。如“*”、“->”、“++”、“--”、“==”、“!=”等，分别对应链表节点之间的取值、调用、右移、左移、判断等于、判断不等于。

```cpp
__list_iterator<Foo, Foo&, Foo*> iter;
*iter;  // 获得一个Foo对象
iter->method();  
// 调用Foo::method()
// 相当于(*iter).method()
// 相当于(&(*iter))->method()
```

![img](C++%E9%9D%A2%E5%90%91%E5%AF%B9%E8%B1%A1%E7%AC%94%E8%AE%B0_%E4%B8%8B.assets/1244144-20190626190908080-1756617049.png)

![](C++%E9%9D%A2%E5%90%91%E5%AF%B9%E8%B1%A1%E7%AC%94%E8%AE%B0_%E4%B8%8B.assets/image-20200419215901372.png)

（1）C++程序设计中使用**堆内存是非常频繁**的操作，堆内存的**申请和释放**都由程序员自己管理。程序员自己管理堆内存可以提高了程序的效率，但是整体来说堆内存的管理是麻烦的，C++11中引入智能指针，方便管理堆内存。

（2）智能指针在C++11版本之后提供，包含在头文件<memory>中，`shared_ptr`、`unique_ptr`、`weak_ptr`，**迭代器也可以认为特殊的智能指针**，除了提供`*`和`->`操作外，还需要提供`++`，`--`的操作。

## 五、function-like classes  仿函数

（1）使用`struct`或者`class`重载运算符`operator()`, 就是**行为类似于函数的对象.**

- 根据仿函数的操作数可分为 : `unary_function`和`binary_function`.

（2）仿函数与函数指针

> STL采用**对象重载操作实现函数行为**并没有直接定义函数指针来代替仿函数, 仿函数使STL灵活, 也可以**自定义**
>

- 函数指针不能满足STL的抽象性

- 函数指针不能与STL的其他组件搭配, 不够灵活

- 仿函数具有可配接性, 可以满足traits编程, 也是能在**编译期间**就能完成, 不会有运行时的开销.

  C++ 11 lambda相似。 

### 1.unary_function

```cpp
template <class Arg, class Result>
struct unary_function {
    typedef Arg argument_type;	// 参数类型别名
    typedef Result result_type;	// 返回值类型别名
};

template <class T>
struct TJudegNagetive : public unary_function<T, bool> {
    bool operator()(const T& x) const { return x <T(); }
};
```

### 2.binary_function

```cpp
template <class Arg1, class Arg2, class Result>
struct binary_function {
    typedef Arg1 first_argument_type;	// 参数类型别名
    typedef Arg2 second_argument_type;	// 参数类型别名
    typedef Result result_type;	// 返回值类型别名
}; 
//等于
template <class T>
struct equal_to : public binary_function<T, T, bool> {
    bool operator()(const T& x, const T& y) const { return x == y; }
};
```

## 六、namespace经验谈 

### inline namespace （C++11）

```cpp
namespace Program {
  namespace Version1 {
    int getVersion() { return 1; }
  }
  inline namespace Version2 {
    int getVersion() { return 2; } //namespace Program 
  }
}
int version {Program::getVersion()};              // Uses getVersion() from Version2
int oldVersion {Program::Version1::getVersion()}; // Uses getVersion() from Version1
```

## 七、class template  类模板(通用)

```cpp
template<class T1,class T2>//通用模板
class Compare //在这里不能写类型
{   
    T1 num1;
    T2 num2;
public:
    Compare();
};
template<class T1,class T2> //这里必须写
Compare<T1,T2>::Compare  (){  cout << "通用模板" << endl;}
//call
Compare <int   ,int> p1;    //打印通用函数
```

## 八、Funtion Template  函数模板

```cpp
template <typename T>
inline T& max(T &a,T &b){
	return a >b ? a : b;
}
max(1,3);//不必指明T，编译器进行实参推导
```

## 九、Member Template 成员模板

（1）模板里面再定义模板

```cpp
template <class T1, class T2>
struct pair
{
    T1 first;
    T2 second;
    pair(): first(), second() {}  //用无参构造函数初始化 first 和 second
    pair(const T1 &a, const T2 &b): first(a), second(b) {}
    template <class U1, class U2>
    pair(const pair <U1, U2> &p): first(p.first), second(p.second) {}
};

template <class T1, class T2>
pair<T1, T2 > make_pair(T1 x, T2 y)
{
    return ( pair<T1, T2> (x, y) );
}

//call


class Base1 {};//Base1理解为鱼类
class derived1 :public Base1 {};//derived1理解为鲫鱼
class Base2 {};//Base2理解为鸟类
class derived2 :public Base2 {};//derived2理解为麻雀

Pair<derived1, derived2> p;
//将p，即鲫鱼和麻雀组成的Pair作为参数传入Pair的构造函数
//形成的p2里面，first是鱼类，second是鸟类。但是他们的具体内容是鲫鱼和麻雀
//Pair<Base1, Base2> p2(p);
//因为普通的指针都可以指向自己子类的对象，那么作为这个指针的封装（智能指针），那必须能够指向子类对象。所以，可以模板T1就是父类类型，U1就是子类类型。
```

## 十、specialization  模板特化

```cpp
template<>   //类模板全特化,参数类型在后面绑定
class Compare<char*,char*>    //在这里写上所需要的类型
{  
    char* num1;
    char* num2;
public: 
    Compare();
};
//template<>   //全特化这行代码不能写!!!
Compare<char* ,char*>::Compare(){ cout  << "全特化" << endl;}
//call
Compare <char*,char*> p2;   //打印全特化
```

## 十一、partial specialization 模板偏特化

```cpp
template<class T2>  //类模板偏特化
class Compare<float,T2>     //偏特化类型T1 特化为float类型
{
    float num1;
    T2 num2;
public:    
    Compare();
};
template<class T2>  //偏特化这里必须要写
Compare<float,T2>::Compare(){ cout  << "偏特化" << endl;}
//call
Compare <float,int> p3;     //打印偏特化
```

（1）emplate <class Allocator>

```cpp
class vector<bool, Allocator> { //…//};
```
这个偏特化的例子中，一个参数被绑定到bool类型，而另一个参数仍未绑定需要由用户指定。

(2)偏特化分：类模板偏特化、 函数模板偏特化

```cpp
template<class T>
class  A {
public:
    void operator()(T t)const {
        cout << "泛化" << endl;
    }
};

template<class T>
class A<T*> {
public:
    void operator()(T* tp)const {
        cout << "范围偏特化" << endl;
    }
};
//called
int num = 5;
int *p_num = &num;
A<int> a1;
A<int*> a2;
a1(num);    //输出泛化
a2(p_num);    //输出范围偏特化
```

## 十二、模板的模板参数 

（1）模板的参数类型也是模板

（2）  template <class T> 和  template <typename T> ，仅此class和 typename可以互相替换。

```cpp
template<typename T, 
                template <typename T>
                class Container
              >
class XCls
{
private:
    Container<T> c;
public:
    ....
};

template<typename T>
using Lst = list<T,allocator<T>>;

XCls<string,list> mylst1;   //语法错误，容器list有第二模板参数
XCls<string,Lst> mylst2;
```



```cpp
template<typename T,
               template<typename T>
                class SmartPtr
             >
 
class XCls
{
private:
    SmartPtr<T> sp;
public:
    XCls() :sp(new T) { }
};
XCls<string,shared_ptr> p1;
XCls<double,unique_ptr> p2;  //错误，指针特性
XCls<int,weak_ptr> p3;  //错误，指针特性
XCls<long,auto_ptr> p4;
```

不是模板模板参数

```cpp
template<class T,class Sequence = deque<T>>  //Sequence = deque<T>绑定了参数
class stack
{
    friend bool operator==<>(const stack&,const stack&);
    frend bool operator< <>(const stack&,const stack&);
protected:
    Sequence c;
    ....
};

stack<int> s1; //第二个参数默认值
stack<int,list<int>> s2;//第二参数已经绑定了
```

## 十三、关于C++标准库

STL六大组件：容器（`containers`）、算法（`algorithms`）、迭代器（`iterators`）、函数对象（`functors`）、适配器（`adapters`）、分配器（`allocators`）

## 十四、三个主题（ 和C++11有关）

（1）`variadic templates` 数量不定的模板参数，即模板参数可变化

```cpp
template <typename T，typename...Types>
```

（2）`auto` 编译器可以指导变量的类型，就在变量前面加上`auto`

```cpp
for(auto a:vec){} //pass by value
for(auto & a:vec){}//pass by reference
//auto写法，编译器从右边的find返回值推导其变量类型
auto ite = find(c.begin(), c.end(), string("leo"));
```

（3）ranged-base for 即`for`循环的新形式

```cpp
for( int i : {1, 2, 3, 4, 5, } )  {  }

list<int> c;
c.push_back(33);
//遍历一个容器里的元素    
for (int i : c) {
    cout << i << endl;
}
```

## 十五、Reference和Point

![img](C++%E9%9D%A2%E5%90%91%E5%AF%B9%E8%B1%A1%E7%AC%94%E8%AE%B0_%E4%B8%8B.assets/1244144-20190627021704182-655246197.png)

- > ####   指针和引用的区别：
  >
  > 1. 都是地址的概念；指针是一个实体,指向一块内存，它的内容是所指内存的地址；引用则是某块内存的别名。 
  >
  > 2. 使用sizeof指针本身大小一般是（4），而引用则是被引用对象的大小； 
  >
  > 3. 引用不能为空，指针可以为空；指针可以被初始化为NULL，而引用必须被初始化且必须是一个已有对象的引用,之后不可变；指针可变；引用“从一而终”，指针可以“见异思迁”； 
  >
  > 4. 作为参数传递时，指针需要被解引用才可以对对象进行操作，而直接对引用的修改都会改变引用所指向的对象； 
  >
  > 5.  引用没有`const`，指针有`const`，`const`的指针不可变；
  >     具体指没有`int& const a`这种形式，而`const int& a`是有的，前者指引用本身即别名不可以改变，这是当然的，所以不需要这种形式，后者指引用所指的值不可以改变）
  >     
  > 6. 指针在使用中可以指向其它对象，但是引用只能是一个对象的引用，不能 被改变；
  >
  > 7. 指针可以有多级指针（`**p`），而引用至于一级； 
  >
  > 8. 指针和引用使用自增(`++`)运算符的意义不一样； 
  >
  > 9. 如果返回动态内存分配的对象或者内存，必须使用指针，引用可能引起内存泄露。 
  >
  > 10. 引用是类型安全的，而指针不是 (引用比指针多了类型检查）
  >
  >     **所以，对引用最好的理解方式，就是认为引用就是变量，不用关心为什么，编译器给我们制造了一个假象，就像一个人有大名和小名一样。你对引用做的任何操作，就相当于对变量本身做操作。**
  >
  >     ```cpp
  >     double imag(const double& im){} //不能并存
  >     double imag(const double im){}  //不能并存
  >     //对于调用来说，是一样的imag(im),编译器无法处理
  >     ```
  >
