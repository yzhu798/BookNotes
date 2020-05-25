## 01 模板类型推断机制
* auto 推断的基础是模板类型推断机制，但部分特殊情况下，模板推断机制不适用于 auto。
```cpp
template<typename T>
void f(ParamType x); // ParamType即x的类型
f(expr); //调用
```
* 编译期间，编译器用 expr 推断 T 和 ParamType，实际上两者通常不一致，比如
```cpp
template<typename T>
void f(const T& x);

int x; 
f(x);  //   T被推断为int，ParamType被推断为const int&
```
* T 的类型推断与 **expr 和 ParamType 相关**，可分为三种情况

### 情形1：ParamType 不是引用或指针
* **丢弃** expr 的 **top-level const和&**限定符，最后得到的 expr 类型就是 T 和 ParamType 的类型
```cpp
template<typename T>
void f(T x); //ParamType 不是引用或指针
//丢弃的是top-level const | &
int a;          f(a); //T和ParamType都是int
const int b;	f(b); //T和ParamType都是int
const int& c;	f(c); //T和ParamType都是int

// char数组会退化为指针
char s1[] = "downdemo";
const char s2[] = "downdemo";
f(s1); // T和ParamType都是char*
f(s2); // T和ParamType都是const char*

// 指针类型   丢弃的是top-level const（即指针本身的const）
// low-level const（即所指对象的const）会保留
int* p1;
const int* p2;
int* const p3;
const int* const p4;
f(p1); // T和ParamType都是int*
f(p2); // T和ParamType都是const int*
f(p3); // T和ParamType都是int*
f(p4); // T和ParamType都是const int*

```

### 情形2：ParamType 是引用类型
#### ParamType = T&

* 如果 expr 的类型是引用，保留 cv 限定符，ParamType 一定是左值引用类型，ParamType 去掉引用符就是 T 的类型，即 T 一定不是引用类型
```cpp
template<typename T>
void f(T& x);
// &相关
int a;			f(a); // ParamType是int&，T是int
int& b;			f(b); // ParamType是int&，T是int
int&& c;		f(c); // ParamType是int&，T是int
const int d;	f(d); // ParamType是const int&，T是const int
const int& e;	f(e); // ParamType是const int&，T是const int

// 数组类型对于T&的情况比较特殊，不会退化到指针
char s1[] = "downdemo";
const char s2[] = "downdemo";
f(s1); // ParamType是char(&)[9]，T是char[9]
f(s2); // ParamType是const char(&)[9]，T是const char[9]

// 因为top-level const和low-level const都保留
// aramType是T &，T 就是参数类型.
int* p1;		      f(p1); // ParamType是T &，T是int*
const int* p2;		  f(p2); // ParamType是T &，T是const int*
int* const p3;		  f(p3); // ParamType是T &，T是int* const
const int* const p4;  f(p4); // ParamType是T &，T是const int* const

```
#### ParamType =  const T&

*  ParamType 是 top-level const，去掉 top-level const 和引用符就是 T 的类型。
```cpp
template<typename T>
void f(const T& x);
// 以下情况ParamType都是const int&，T都是int
int a;
int& b;
int&& c;
const int d;
const int& e;
f(x);
// 对于指针只要记住，T的指针符后一定无const、&
int* p1;      f(p1);  // ParamType是  (T + const &),T是 int*
const int* p2;
int* const p3;
const int* const p4;

char s1[] = "downdemo";
const char s2[] = "downdemo";
// 数组类型类似
f(s1); // ParamType是const char(&)[9]，T是char[9]
f(s2); // ParamType是const char(&)[9]，T是char[9]
```
#### 数组类型的模板参数

* 对应数组类型的模板参数类型应声明为 `T (&) [N]`，即数组类型 `T[N]` 的引用
```cpp
template<typename T, std::size_t N>
constexpr std::size_t f(T (&) [N]) noexcept
{
    return N;
}
const char s[] = "downdemo";
int a[f(s)]; // int a[9]
```

### 情形3：ParamType 是指针类型

#### ParamType 是 non-const 指针

* ParamType是(T*)，是 non-const 指针（传参忽略 top-level const）。
```cpp
template<typename T>
void f(T* x);
// ParamType是T*
int a;
const int b;
f(&a); // T是int
f(&b); // T是const int

int* p1;  //T是int
const int* p2;//T是const int

int* const p3; // 传参时与p1类型一致
const int* const p4; // 传参时与p2类型一致

char s1[] = "downdemo";
const char s2[] = "downdemo";
// 数组类型会转为指针类型
f(s1); // ParamType是char*，T是char
f(s2); // ParamType是const char*，T是const char
```
#### ParamType 是 const-pointer （ top-level ）

* ParamType 多出 top-level const，T 不变
```cpp
template<typename T>
void f(T* const x);
// ParamType是T +(* const)，
int a;
const int b;

int* p1; // 传参时与p3类型一致
const int* p2; // 传参时与p4类型一致
int* const p3;
const int* const p4;

char s1[] = "downdemo";
const char s2[] = "downdemo";

f(&a); // ParamType是int* const，T是int
f(&b); // ParamType是const int* const，T是const int

f(p1); // ParamType是int* const，T是int
f(p2); // ParamType是const int* const，T是const int
f(p3); // ParamType是int* const，T是int
f(p4); // ParamType是const int* const，T是const int

f(s1); // ParamType是char* const，T是char
f(s2); // ParamType是const char* const，T是const char
```
#### ParamType 是 pointer to const （ low-level ）

* 如果 ParamType 是 pointer to const，则只有一种结果，**T 一定是不带 const 的非指针类型**。
```cpp
template<typename T>
void f(const T* x);     //non-array ,以下情况ParamType都是const int*，T都是int

template<typename T>
void g(const T* const x);//non-array, 以下情况ParamType都是const int* const，T都是int

int a;
const int b;

int* p1;
const int* p2;
int* const p3;
const int* const p4;

char s1[] = "downdemo";
const char s2[] = "downdemo";

// array
// 以下情况ParamType都是const char*，T都是char
f(s1);
f(s2);
g(s1);
g(s2);
```

### 情形4：ParamType 是转发引用
* 如果 **expr 是左值**，**T 和 ParamType 都推断为左值引用**。这有两点非常特殊
  * 这是 **T 被推断为引用的唯一情形**
  * **ParamType 使用右值引用语法，却被推断为左值引用**
* 如果 **expr 是右值**，则 ParamType 推断为右值引用类型，去掉 && 就是 T 的类型，即 T 一定不为引用类型
```cpp
template<typename T>
void f(T&& x);

int a;
const int b;
const int& c;
int&& d = 1; // d是右值引用，也是左值，右值引用是只能绑定右值的引用而不是右值

char s1[] = "downdemo";
const char s2[] = "downdemo";

f(a); // ParamType和T都是int&
f(b); // ParamType和T都是const int&
f(c); // ParamType和T都是const int&
f(d); // ParamType和T都是const int&
f(1); // ParamType是int&&，T是int ,1是右值 

f(s1); // ParamType和T都是char(&)[9]
f(s2); // ParamType和T都是const char(&)[9]
```

### 特殊情形：expr 是函数名
```cpp
template<typename T> void f1(T x);
template<typename T> void f2(T& x);
template<typename T> void f3(T&& x);

void g(int);

f1(g); // T和ParamType都是void(*)(int)
f2(g); // ParamType是void(&)(int)，T是void()(int)
f3(g); // T和ParamType都是void(&)(int)
```

## 02 [auto](https://en.cppreference.com/w/cpp/language/auto)类型推断机制

### C++11的auto

* auto 类型推断几乎和模板类型推断一致

* 调用模板时，编译器根据 expr 推断 T 和 ParamType 的类型。

  > 当变量用 auto 声明时，auto 就扮演了模板中的T的角色，变量的类型修饰符则扮演 ParamType 的角色

* 为了推断变量类型，编译器表现得好比每个声明对应一个模板，模板的调用就相当于对应的初始化表达式
```cpp
auto x = 1;
const auto cx = x;
const auto& rx = x;

template<typename T> // 用来推断x类型的概念上假想的模板
void func_for_x(T x);

func_for_x(1); // 假想的调用: param的推断类型就是x的类型

template<typename T> // 用来推断cx类型的概念上假想的模板
void func_for_cx(const T x);

func_for_cx(x); // 假想的调用: param的推断类型就是cx的类型

template<typename T> // 用来推断rx类型的概念上假想的模板
void func_for_rx(const T& x);

func_for_rx(x); // 假想的调用: param的推断类型就是rx的类型
```
* **auto 的推断适用模板推断机制的三种情形：T&、T&& 和 T**
```cpp
auto x = 1; // int x
const auto cx = x; // const int cx
const auto& rx = x; // const int& rx
auto&& uref1 = x; // int& uref1
auto&& uref2 = cx; // const int& uref2
auto&& uref3 = 1; // int&& uref3
```
* **auto 对数组和指针**的推断也和模板一致
```cpp
const char name[] = "downdemo"; // 数组类型是const char[9]
auto arr1 = name; // const char* arr1
auto& arr2 = name; // const char (&arr2)[9]

void g(int, double); // 函数类型是void(int, double)
auto f1 = g; // void (*f1)(int, double)
auto& f2 = g; // void (&f2)(int, double)
```
* auto 推断**唯一不同于**模板实参推断的情形是 **C++11 的初始化列表**。下面是同样的赋值功能
```cpp
// C++98
int x1 = 1;
int x2(1);
// C++11
int x3 = { 1 };
int x4{ 1 };
```
* 但换成 auto 声明，这些赋值的意义就不一样了
```cpp
auto x1 = 1; // int x1
auto x2(1); // int x2
auto x3 = { 1 }; // std::initializer_list<int> x3
auto x4{ 1 }; // C++11为std::initializer_list<int> x4，C++14为int x4
```
* 如果初始化列表中元素类型不同，则无法推断
```cpp
auto x5 = { 1, 2, 3.0 }; // 错误：不能为std::initializer_list<T>推断T
```
* **C++14 禁止对 auto 用[std::initializer_list](https://en.cppreference.com/w/cpp/utility/initializer_list)直接初始化**，而必须用 =，除非列表中只有一个元素，这时不会将其视为[std::initializer_list](https://en.cppreference.com/w/cpp/utility/initializer_list)
```cpp
auto x1 = { 1, 2 }; // C++14中必须用=，否则报错
auto x2 { 1 }; // 允许单元素的直接初始化，不会将其视为initializer_list
```
* **模板不支持模板参数为 T 而 expr 为初始化列表**的推断，不会将其假设为[std::initializer_list](https://en.cppreference.com/w/cpp/utility/initializer_list)，这就是 auto 推断和模板推断唯一的不同之处
```cpp
auto x = { 1, 2, 3 }; // x类型是std::initializer_list<int>

template<typename T> // 等价于x声明的模板
void f(T x);

f({ 1, 2, 3 }); // 错误：不能推断T的类型
```
* 不过将模板参数为[std::initializer_list](https://en.cppreference.com/w/cpp/utility/initializer_list)则可以推断 T
```cpp
template<typename T>
void f(std::initializer_list<T> initList);

f({ 11, 23, 9 }); // T被推断为int，initList类型是std::initializer_list<int>
```
* 对于 C++11，auto的介绍就到此为止了

### C++14的auto
* C++14中，auto 可以作为**函数返回**类型，并且 lambda 可以将**参数**声明为 auto， 称为泛型 lambda
```cpp
auto f() { return 1; }
auto g = [](auto x) { return x; };
```
* 但此时 auto 仍是模板实参推断的机制，因此**不能为 auto 返回类型返回一个初始化列表**，即使是单元素
```cpp
auto newInitList() { return { 1 }; } // 错误
```
* 泛型 lambda 同理
```cpp
std::vector<int> v { 2, 4, 6 };
auto resetV = [&v](const auto& newValue) { v = newValue; };
resetV({ 1, 2, 3 }); // 错误
```

### C++17的auto
* C++17中，auto 可以作为**非类型模板参数**
```cpp
template<auto N>
struct X {
    void f() { std::cout << N; }
};

X<1> x; //1不是类型，是一个值
x.f(); // 1
```

## 03 [decltype](https://en.cppreference.com/w/cpp/language/decltype)
* decltype 会推断出直觉预期的类型
```cpp
const int i = 0; // decltype(i)为const int

struct Point {
    int x, y; // decltype(Point::x)和decltype(Point::y)为int
};

A a; // decltype(a)为A
bool f(const A& x); // decltype(x)为const A&，decltype(f)为bool(const A&)
if (f(a)) … // decltype(f(a))为bool

int a[] {1, 2, 3}; // decltype(a)为int[3]
```
* **decltype** 一般用来声明**返回类型**。比如下面模板的参数是容器和索引，而返回类型取决于元素类型
```cpp
template<typename Container, typename Index>
auto f(Container& c, Index i) -> decltype(c[i]) //尾置返回类型 C++ 11
{ // auto不做任何事，只是表示使用类型推断，推断使用的是decltype
    return c[i]; //int&
}
```
```cpp
template<typename Container, typename Index>
auto f(Container& c, Index i)//尾置返回类型 C++ 14
{
    return c[i];//operator[] 返回元素引用，类型为 int&，但 auto 推断为 int.
}
// call
std::vector<int> v;
f(v, 5) = 10; // 返回v[5]然后赋值为10，但不能通过编译
//修改为 decltype(auto)
decltype(auto) f(Container& c, Index i) //C++14 
```
* decltype(auto) 声明变量
```cpp
int i = 1;
const int& j = i;
decltype(auto) x = j; // const int& x = j;
```
* 但还有一些问题，容器传的是 non-const 左值引用，这就无法接受右值
```cpp
std::vector<int> makeV(); // 工厂函数
auto i = f(makeV(), 5);
```
* 同时**匹配左值和右值**而非重载，只需要**模板参数写为转发引用**
```cpp
template<typename Container, typename Index>
decltype(auto) f(Container&& c, Index i) // C++14版本
{
    return std::forward<Container>(c)[i]; // 传入的实参是右值时，std::forward将c转为右值
}
// C++11版本
template<typename Container, typename Index>
auto f(Container&& c, Index i) -> decltype(std::forward<Container>(c)[i])
{
    authenticateUser();
    return std::forward<Container>(c)[i];
}
```

### decltype的特殊情况
* 如果**表达式是解引用**，decltype 会**推断为引用类型**
```cpp
int* p; // decltype(*p)是int&
```
* **赋值表达式会产生引用**，decltype 会推断类型为**左值的引用类型**
```cpp
int a = 0;
int b = 1;
decltype(a=1) c = b; // int&
c = 3;
std::cout << a << b << c; // 033
```
* 如果**表达式加上括号**，变量作为赋值语句左值的特殊表达式；推断为引用类型。
* **decltype((variable)) 结果永远是引用**，declytpe(variable) 只有当变量**本身是引用时**才是引用
```cpp
int i; // decltype((i))是int&
```
* 在返回类型为 decltype(auto) 时，这**可能导致返回局部变量的引用**
```cpp
decltype(auto) f1()
{
    int x = 0;
    return x; // decltype(x)是int，因此返回int
}

decltype(auto) f2()
{
    int x = 0;
    return (x); // decltype((x))是int&，因此返回了局部变量的引用
}
```

## 04 查看推断类型的方法
* **最简单直接的方法是在 IDE 中将鼠标停放在变量**上

![](./images/1-1.png)

* 利用报错信息，比如写一个声明但不定义的类模板，用这个模板创建实例时将出错，编译将提示错误原因
```cpp
template<typename T>
class A;

A<decltype(x)> xType; // 未定义类模板，错误信息将提示x类型
// 比如对int x报错如下
error C2079: “xType”使用未定义的 class“A<int>”
```
* 使用[type_id运算符](https://en.cppreference.com/w/cpp/language/typeid)和[std::type_info::name](https://en.cppreference.com/w/cpp/types/type_info/name)获取类型，但得到的类型会忽略cv和引用限定符
```cpp
template<typename T>
void f(T& x)
{
    std::cout << "T = " << typeid(T).name() << '\n';
    std::cout << "x = " << typeid(x).name() << '\n';
}
```
* 使用[Boost.TypeIndex](https://www.boost.org/doc/libs/1_71_0/doc/html/boost_typeindex_header_reference.html#header.boost.type_index_hpp)可以得到精确类型
```cpp
#include <boost/type_index.hpp>

template<typename T>
void f(const T& x)
{
    using boost::typeindex::type_id_with_cvr;
    std::cout << "T = " << type_id_with_cvr<T>().pretty_name() << '\n';
    std::cout << "x = " << type_id_with_cvr<decltype(x)>().pretty_name() << '\n';
}
```

## 05 用[auto](https://en.cppreference.com/w/cpp/language/auto)替代显式类型声明
* auto 声明的变量必须初始化，因此使用 **auto 可以避免忘记初始化的问题**
```cpp
int a;  // 潜在的未初始化风险
auto b; // 错误：必须初始化
```
* **对于名称非常长的类型**，如迭代器相关的类型，用 **auto 声明简化工作**
```cpp
template<typename It>
void f(It b, It e)
{
    while (b != e)
    {
        auto currentValue = *b;
        // typename std::iterator_traits<It>::value_type currentValue = *b;
        ...
    }
}
```
* lambda 生成的闭包类型是编译期内部的匿名类型，无法得知，使用 auto 推断就没有这个问题
```cpp
auto f = [](auto& x, auto& y) { return x < y; };
```
* 如果不使用 auto，可以改用[std::function](https://en.cppreference.com/w/cpp/utility/functional/function)
```cpp
// std::function的模板参数中不能使用auto
std::function<bool(int&, int&)> f = [](auto& x, auto& y) { return x < y; };
```
* 除了明显的语法冗长和不能利用 auto 参数的缺点，[std::function](https://en.cppreference.com/w/cpp/utility/functional/function)与 auto 的最大区别在于，auto 和闭包类型一致，内存量和闭包相同.

  **而[std::function](https://en.cppreference.com/w/cpp/utility/functional/function)是类模板，它的实例有一个固定大小**，这个大小不一定能容纳闭包，于是会分配堆上的内存以存储闭包，导致占用更多内存。

  此外，编译器一般会限制内联，[std::function](https://en.cppreference.com/w/cpp/utility/functional/function)调用闭包会比 auto 慢

* auto 可以避免简写类型存在的潜在问题。比如如下代码有潜在隐患
```cpp
std::vector<int> v;
unsigned sz = v.size(); // v.size()类型实际为std::vector<int>::size_type
// 在32位机器上std::vector<int>::size_type与unsigned尺寸相同
// 但在64位机器上，std::vector<int>::size_type是64位，而unsigned是32位
```
* 如下代码也有潜在问题
```cpp
std::unordered_map<std::string, int> m; // m的元素类型std::pair<const std::string, int>
for(const std::pair<std::string, int>& p : m) ... // 类型不一致，转换，期间要构造大量临时对象
```
* 如果**显式类型声明能让代码更清晰**或有其他好处就不用强行 auto，此外 IDE 的类型提示也能缓解不能直接看出对象类型的问题

## 06 [auto](https://en.cppreference.com/w/cpp/language/auto)推断出非预期类型时，先强制转换出预期类型
* 如下代码没有问题
```cpp
std::vector<bool> f()
{
    return std::vector<bool>{ true, false };
}

bool x = f()[0];
if(x) std::cout << "OK";
```
* 但如果把显式声明改为 auto 则会出现非预期行为
```cpp
std::vector<bool> f()
{
    return std::vector<bool>{ true, false };
}

auto x = f()[0]; // 改用auto声明
if(x) std::cout << "OK"; // 错误：未定义行为,临时对象，空悬指针
```
* 原因在于实际上得到的**类型不是 bool**
```cpp
auto x = f()[0]; // x类型为std::vector<bool>::reference
```
* [std::vector\<bool\>](https://en.cppreference.com/w/cpp/container/vector_bool)不是真正的 STL 容器，也不包含 bool 类型元素。它是[std::vector](https://en.cppreference.com/w/cpp/container/vector)对于 bool 类型的特化，**为了节省空间，每个元素用一个 bit（而非一个bool）表示**，于是[operator[]](https://en.cppreference.com/w/cpp/container/vector/operator_at)返回的应该是单个 bit 的引用，但 C++ 中不存在指向单个 bit 的指针，因此也不能获取单个 bit 的引用
```cpp
std::vector<bool> v { true, false };
bool* p = &v[0]; // 错误
std::vector<bool>::reference* q = &v[0]; // 正确
```
* 因此需要一个行为类似单个 bit 并可以被引用的对象，就是[std::vector\<bool\>::reference](https://en.cppreference.com/w/cpp/container/vector_bool/reference)，**隐式转换**为 bool
```cpp
bool x = f()[0];
```
* 而对于 **auto 推断则不会隐式转换**
```cpp
auto x = f()[0]; // std::vector<bool>::reference x = f()[0];
// x不一定指向std::vector<bool>的第0个bit，这取决于std::vector<bool>::reference的实现
// 一种实现是含有一个指向一个machine word的指针，word持有被引用的bit和这个bit相对word的offset
// 于是x持有一个由opeartor[]返回的临时的machine word的指针和bit的offset
// 这条语句结束后临时对象被析构，于是x含有一个空悬指针，导致后续的未定义行为
if(x) ... // 相当于int* p; if(p) ...
```
* [std::vector\<bool\>::reference](https://en.cppreference.com/w/cpp/container/vector_bool/reference)是一个代理类（proxy class，模拟或扩展其他类型的类）的例子，比如[std::shared_ptr](https://en.cppreference.com/w/cpp/memory/shared_ptr)和[std::unique_ptr](https://en.cppreference.com/w/cpp/memory/unique_ptr)是很明显的代理类。还有一些为了提高数值计算效率而使用表达式模板技术开发的类，比如给定一个 Matrix 类和它的对象
```cpp
Matrix sum = m1 + m2 + m3 + m4;
```
* Matrix 对象的 operator+ 返回的是**结果的代理**而非结果本身，这样可以使得表达式的计算更为高效
```cpp
auto x = m1 + m2; // x可能是Sum<Matrix, Matrix>而不是Matrix对象
```
* auto 推断出代理类的问题实际很容易解决，事先做一次到**预期类型的强制转换**即可
```cpp
auto x = static_cast<bool>(f()[0]);
```

## 07 创建对象时注意区分()和{}
* 值初始化方式
```cpp
int a(0);      // 初始化值在小括号中
int b = 0;     // 初始化值在等号后
int c{ 0 };    // 初始化值在大括号中
int d = { 0 }; // 按int d{ 0 }处理，后续讨论将忽略这种用法
```
* 使用等号不一定是赋值，也可能是拷贝。
* 对于内置类型来说，初始化和赋值的区别只是学术争议，但对于类类型则不同
```cpp
X a;     // 默认构造
X b = a; // 拷贝而非赋值
a = b;   // 拷贝而非赋值
```
* C++11引入了统一初始化（uniform initialization），也可以叫**大括号初始化**（braced initialization）。大括号初始化可以方便地为容器指定初始元素
```cpp
std::vector<int> v{ 1, 2, 3 };
```
* **大括号初始化同样能为 non-static 数据成员指定默认值**，也可以用=指定，但不能用小括号初始化指定
```cpp
class A {
    int x{ 0 }; // OK
    int y = 0;  // OK
    int z(0);   // 错误
};
```
* **大括号初始化禁止内置类型的隐式收缩转换**（implicit narrowing conversions），而小括号初始化和 = 不会
```cpp
double x = 1.1;
double y = 2.2;
int a{ x + y }; // 错误：大括号初始化不允许double到int的收缩转换
int b(x + y); // OK：double被截断为int
int c = x + y; // OK：double被截断为int
```
* 大括号初始化不用担心 C++ 的最令人苦恼的解析（C++'s most vexing parse）
```cpp
class A {
public:
    A() { std::cout << 1; }
};

class B{
public:
    B(std::string) { std::cout << 2; }
};
//--> 函数声明
A a(); // 不调用A的构造函数，而是被解析成一个函数声明：A a();
std::string s("hi");
B b(std::string(s)); // 不调用B的构造函数，而是被解析成一个函数声明：B b(std::string);

A a2{}; // 调用A的构造函数
B b2{ std::string(s) }; // 调用B的构造函数

// C++11之前的解决办法
A a3;
B b3((std::string(s)));
```
* 大括号初始化的缺陷在于，只要类型转换后可以匹配，大括号初始化总会**优先匹配参数类型为[std::initializer_list](https://en.cppreference.com/w/cpp/utility/initializer_list)的构造函数**，即使收缩转换会导致调用错误
```cpp
class A {
public:
    A(int) { std::cout << 1; }
    A(std::string) { std::cout << 2; }
    A(std::initializer_list<int>) { std::cout << 3; }
};

A a{ 0 }; // 3
A b{ 3.14 }; // 错误：大括号初始化不允许double到int的收缩转换
A c{"hi"}; // 2
```
* 但特殊的是，参数为空的大括号初始化只会调用**默认构造函数**。
* 如果想传入真正的空[std::initializer_list](https://en.cppreference.com/w/cpp/utility/initializer_list)作为参数，则要额外添加一层大括号或小括号
```cpp
class A {
public:
    A() { std::cout << 1; }
    A(std::initializer_list<int>) { std::cout << 2; }
};

A a{}; // 1
A b{{}}; // 2
A c({}); // 3
```
* 上述问题带来的实际影响很大，比如[std::vector](https://en.cppreference.com/w/cpp/container/vector)就存在参数为参数[std::initializer_list](https://en.cppreference.com/w/cpp/utility/initializer_list)的构造函数，这导致了参数相同时，**大括号初始化和小括号初始化**调用的却是**不同版本的构造函数**
```cpp
std::vector<int> v1(3, 6); // 元素为6、6、6（3个6）
std::vector<int> v2{3, 6}; // 元素为3和6
```
* 这是一种失败的设计，并给模板作者带来了对大括号初始化和小括号初始化的选择困惑
```cpp
template<typename T, typename... Ts>
decltype(auto) f(Ts&&... args)
{
    T x(std::forward<Ts>(args)...); // 用小括号初始化创建临时对象
    return x;
}

template<typename T, typename... Ts>
decltype(auto) g(Ts&&... args)
{
    T x{ std::forward<Ts>(args)... }; // 用大括号初始化创建临时对象
    return x;
}

// 模板作者不知道调用者希望得到哪个结果
auto v1 = f<std::vector<int>>(3, 6); // v1元素为6、6、6
auto v2 = g<std::vector<int>>(3, 6); // v2元素为3、6
```
* [std::make_shared](https://en.cppreference.com/w/cpp/memory/shared_ptr/make_shared)和[std::make_unique](https://en.cppreference.com/w/cpp/memory/unique_ptr/make_unique)就面临了这个问题，而它们的选择是**使用小括号初始化**并在接口文档中写明这点
```cpp
auto p = std::make_shared<std::vector<int>>(3, 6);
for (auto x : *p) std::cout << x; // 666
```

## 08 用[nullptr](https://en.cppreference.com/w/cpp/language/nullptr)替代0和[NULL](https://en.cppreference.com/w/cpp/types/NULL)
* **字面值0本质是 int 而非指针**，只有在使用指针的语境中发现0才会解释为空指针
* **[NULL](https://en.cppreference.com/w/cpp/types/NULL)的本质是宏**，没有规定的实现标准，**一般在 C++ 中定义为0，在 C 中定义为 void\***
```cpp
// VS2017中的定义
#ifndef NULL
    #ifdef __cplusplus
        #define NULL 0
    #else
        #define NULL ((void *)0)
    #endif
#endif
```
* **在重载解析时，[NULL](https://en.cppreference.com/w/cpp/types/NULL)作为参数不会优先匹配指针类型**。而[nullptr](https://en.cppreference.com/w/cpp/language/nullptr)的类型是[std::nullptr_t](https://en.cppreference.com/w/cpp/types/nullptr_t)，[std::nullptr_t](https://en.cppreference.com/w/cpp/types/nullptr_t)可以转换为任何原始指针类型
```cpp
void f(bool) { std::cout << 1; }
void f(int) { std::cout << 2; }
void f(void*) { std::cout << 3; }

f(0); // 2
f(NULL); // 2
f(nullptr); // 3
```
* 这点也会**影响模板实参推断**
```cpp
template<typename T>
void f() {}

f(0); // T推断为int
f(NULL); // T推断为int
f(nullptr); // T推断为std::nullptr_t
```
* 使用[nullptr](https://en.cppreference.com/w/cpp/language/nullptr)就可以避免推断出非指针类型
```cpp
void f1(std::shared_ptr<int>) {}
void f3(int*) {}

template<typename F, typename T>
void g(F f, T x)
{
    f(x);
}

g(f1, 0); // 错误
g(f1, NULL); // 错误
g(f1, nullptr); // OK

g(f3, 0); // 错误
g(f3, NULL); // 错误
g(f3, nullptr); // OK
```
* 使用[nullptr](https://en.cppreference.com/w/cpp/language/nullptr)也能使代码意图更清晰
```cpp
auto res = f();
if (res == nullptr) ... // 很容易看出res是指针类型
```

## 09 用[using别名声明](https://en.cppreference.com/w/cpp/language/type_alias)替代[typedef](https://en.cppreference.com/w/cpp/language/typedef)
* [using别名声明](https://en.cppreference.com/w/cpp/language/type_alias)比[typedef](https://en.cppreference.com/w/cpp/language/typedef)可读性更好，尤其是对于**函数指针类型**
```cpp
typedef void (*F)(int);
using F = void (*)(int);
```
* C++11 还引入了[别名模板](https://en.cppreference.com/w/cpp/language/type_alias)，它只能使用[using别名声明](https://en.cppreference.com/w/cpp/language/type_alias)
```cpp
template<typename T>
using X = std::vector<T>; // X<int>等价于std::vector<int>

// C++11之前的做法是在模板内部typedef
template<typename T>
struct Y { // Y<int>::type等价于std::vector<int>
    typedef std::vector<T> type;
};

// 在其他类模板中使用这两个别名的方式
template<typename T>
class A {
    X<T> x;
    typename Y<T>::type y;
};
```
* C++11 引入了[type traits](https://en.cppreference.com/w/cpp/header/type_traits)，为了方便使用，C++14 为每个[type traits](https://en.cppreference.com/w/cpp/header/type_traits)都定义了[别名模板](https://en.cppreference.com/w/cpp/language/type_alias)
```cpp
// std::remove_reference的实现
template<typename T>
struct remove_reference {
    using type = T;
};

template<typename T>
struct remove_reference<T&> {
    using type = T;
};

template<typename T>
struct remove_reference<T&&> {
    using type = T;
};

// std::remove_reference_t的实现
template<typename T>
using remove_reference_t = typename remove_reference<T>::type;
```
* 为了简化生成值的[type traits](https://en.cppreference.com/w/cpp/header/type_traits)，C++14 还引入了[变量模板](https://en.cppreference.com/w/cpp/language/variable_template)
```cpp
// std::is_same的实现
template<typename T, typename U>
struct is_same {
    static constexpr bool value = false;
};

// std::is_same_v的实现
template<typename T>
constexpr bool is_same_v = is_same<T, U>::value;
```

## 10 用[enum class](https://en.cppreference.com/w/cpp/language/enum#Scoped_enumerations)替代[enum](https://en.cppreference.com/w/cpp/language/enum#Unscoped_enumeration)
* 一般在大括号中声明的名称，只在大括号的作用域内可见，但这对 enum 成员例外。**enum 成员属于 enum 所在的作用域**。
```cpp
enum X { a, b, c };
int a = 1; // 错误：a已在作用域内声明过
```
* C++11 引入了**限定作用域的枚举类型**，用 enum class 关键字表示。
```cpp
enum class X { a, b, c };
int a = 1;  // OK
X x = X::a; // OK
X y = b;    // 错误
```
* enum class **不会进行隐式转换**。
```cpp
enum X { a, b, c };
X x = a;
if(x < 3.14) ... // 不应该将枚举与浮点数进行比较，但这里合法

enum class Y { a, b, c };
Y y = Y::a;
if(x < 3.14) ... // 报错：不允许比较
// 但enum class允许强制转换为其他类型
if(static_cast<double>(x) < 3.14) ... // OK
```
* C++11 之前的 enum 不允许前置声明，而 **C++11 的 enum 和 enum class 都可以前置声明**。
```cpp
enum Color;   // C++11之前错误
enum class X; // OK
```
* C++11 之前不能前置声明 enum 的原因是，编译器为了节省内存，要在 enum 被使用前选择一个足够容纳成员取值的**最小整型作为底层类型**
```cpp
enum X { a, b, c }; // 编译器选择底层类型为char
enum Status { // 编译器选择比char更大的底层类型
    good = 0,
    failed = 1,
    incomplete = 100,
    corrupt = 200,
    indeterminate = 0xFFFFFFFF
};
```
* 不能前置声明的一个弊端是，由于编译依赖关系，在 enum 中仅仅添加一个成员可能就要重新编译整个系统。如果在头文件中包含前置声明，修改 enum class 的定义时就不需要重新编译整个系统，如果 enum class 的修改不影响函数的行为，则函数的实现也不需要重新编译
* C++11 支持前置声明的原因很简单，底层类型是已知的，用[std::underlying_type](https://en.cppreference.com/w/cpp/types/underlying_type)即可获取。也可以指定枚举的底层类型，如果不指定，**enum class 默认为 int，enum 则不存在默认类型**
```cpp
enum class X : std::uint32_t;
// 也可以在定义中指定
enum class Y: std::uint32_t { a, b, c };
```
* C++11 中使用 enum 更方便的场景，即**希望 enum 的隐式转换时**
```cpp
enum X { name, age, number };
auto t = std::make_tuple("downdemo" , 6, "13312345678");
auto x = std::get<name>(t); // get的模板参数类型是std::size_t，name可隐式转换为std::size_t
```
* **如果用 enum class，则需要强制转换**
```cpp
enum class X { name, age, number };
auto t = std::make_tuple("downdemo" , 6, "13312345678");
auto x = std::get<static_cast<std::size_t>(X::name)>(t);
```
* 可以用一个函数来封装转换的过程，但也不会简化多少
```cpp
template<typename E>
constexpr auto f(E e) noexcept
{
    return static_cast<std::underlying_type_t<E>>(e);
}

auto x = std::get<f(X::name)>(t);
```

## 11 用=delete替代private作用域来禁用函数
* **C++11 之前**禁用拷贝的方式是将拷贝构造函数和拷贝赋值运算符声明在 **private 作用域中**
```cpp
class A {
private:
    A(const A&); // 不需要定义，只声明，调用时候会报错
    A& operator(const A&);
};
```
* **C++11 中**可以直接将要**删除的函数用 =delete 声明**，**习惯上会声明在 public 作用域中**，这样在使用删除的函数时，会先检查访问权再检查删除状态，出错时能得到更明确的诊断信息
```cpp
class A {
public:
    A(const A&) = delete;
    A& operator(const A&) = delete;
};
```
* **private 作用域中的函数还可以被成员和友元调用**，而 =delete 是真正禁用了函数，无法通过任何方法调用
* **任何函数都可以用 =delete 声明**，比如函数不想接受某种类型的参数，就可以删除对应类型的重载
```cpp
void f(int);
void f(double) = delete; // 拒绝double和float类型参数

f(3.14); // 错误
```
* **=delete 还可以禁止模板**对某个类型的实例化
```cpp
template<typename T>
void f(T x) {}

template<>
void f<int>(int) = delete;

f(1); // 错误：使用已删除的函数
template<typename T>
void processPointer(T* ptr);
```
* 类内的函数模板也可以用这种方式禁用
```cpp
class A {
public:
    template<typename T>
    void f(T x) {}
};

template<>
void A::f<int>(int) = delete;
```
* 当然，写在 private 作用域也可以起到禁用的效果
```cpp
class A {
public:
    template<typename T>
    void f(T x) {}
private:
    template<>
    void f<int>(int);
};
```
* 但把**模板和特化置于不同的作用域不太合逻辑**，与其效仿 =delete 的效果，不如直接用 =delete

## 12 用[override](https://en.cppreference.com/w/cpp/language/override)标记被重写的虚函数
* **虚函数的重写（override）很容易出错**，因为要在派生类中重写虚函数，必须满足一系列要求
  * 基类中必须有此虚函数
  * 基类和派生类的函数名相同（析构函数除外）
  * 函数参数类型相同
  * const属性相同
  * 函数返回值和异常说明相同
* C++11 多出一条要求：引用修饰符相同。引用修饰符的作用是，指定成员函数仅在对象为左值（成员函数标记为 &）或右值（成员函数标记为 &&）时可用
```cpp
class A {
public:
    void f() & { std::cout << 1; } // *this是左值时才使用
    void f() &&{ std::cout << 2; } // *this是右值时才使用
};

A makeA() { return A{}; }

A a;
a.f(); // 1
makeA().f(); // 2
```
* 对于这么多的要求难以面面俱到，比如下面代码没有任何重写但可以通过编译
```cpp
class A {
public:
    virtual void f1() const;
    virtual void f2(int x);
    virtual void f3() &;
    void f4() const;
};

class B : public A {
public:
    virtual void f1();
    virtual void f2(unsigned int x);
    virtual void f3() &&;
    void f4() const;
};
```
* 为了保证正确性，C++11 提供了[override](https://en.cppreference.com/w/cpp/language/override)来**标记要重写的虚函数**，如果未重写就不能通过编译
```cpp
class A {
public:
    virtual void f1() const;
    virtual void f2(int x);
    virtual void f3() &;
    virtual void f4() const;
};

class B : public A {
public:
    virtual void f1() const override;
    virtual void f2(int x) override;
    virtual void f3() & override;
    void f4() const override;
};
```
* [override](https://en.cppreference.com/w/cpp/language/override)是一个 contextual keyword，只在特殊语境中保留，[override](https://en.cppreference.com/w/cpp/language/override)只有出现在**成员函数声明末尾**才有保留意义，因此如果以前的遗留代码用到了[override](https://en.cppreference.com/w/cpp/language/override)作为名字，不用改名就可以升到 C++11
```cpp
class A {
public:
    void override(); // 在C++98和C++11中都合法
};
```
* C++11 还提供了另一个 contextual keyword：[final](https://en.cppreference.com/w/cpp/language/final)，[final](https://en.cppreference.com/w/cpp/language/final)可以用来指定**虚函数禁止被重写**
```cpp
class A {
public:
    virtual void f() final;
    void g() final; // 错误：final只能用于指定虚函数
};

class B : public A {
public:
    virtual void f() override; // 错误：f不可重写
};
```
* [final](https://en.cppreference.com/w/cpp/language/final)还可以用于**指定某个类禁止被继承**
```cpp
class A final {};
class B : public A {}; // 错误：A禁止被继承
```

## 13 用[std::cbegin](https://en.cppreference.com/w/cpp/iterator/begin)和[std::cend](https://en.cppreference.com/w/cpp/iterator/end)获取const_iterator
* 需要迭代器但**不修改值使用 const_iterator**
```cpp
std::vector<int> v{ 2, 3 };
auto it = std::find(std::cbegin(v), std::cend(v), 2); // C++14
v.insert(it, 1);
```
* 上述功能很容易扩展成模板
```cpp
template<typename C, typename T>
void f(C& c, const T& x, const T& y)
{
    auto it = std::find(std::cbegin(c), std::cend(c), x);
    c.insert(it, y);
}
```
* C++11 没有[std::cbegin](https://en.cppreference.com/w/cpp/iterator/begin)和[std::cend](https://en.cppreference.com/w/cpp/iterator/end)，手动实现即可
```cpp
template<class C>
auto cbegin(const C& c)->decltype(std::begin(c))
{
    return std::begin(c); // c是const所以返回const_iterator
}
```

## 14 用[noexcept](https://en.cppreference.com/w/cpp/language/noexcept_spec)标记不抛异常的函数
* C++98 中，必须指出一个函数**可能抛出的所有异常类型**，如果函数有所改动则[exception specification](https://en.cppreference.com/w/cpp/language/except_spec)也要修改，而这可能破坏代码，因为调用者可能依赖于原本的[exception specification](https://en.cppreference.com/w/cpp/language/except_spec)，所以 C++98 中的[exception specification](https://en.cppreference.com/w/cpp/language/except_spec)被认为不值得使用
* C++11 中达成了一个共识，真正需要关心的是函数会不会抛出异常。**一个函数要么可能抛出异常，要么绝对不抛异常**，这种 maybe-or-never 形成了 C++11 [exception specification](https://en.cppreference.com/w/cpp/language/except_spec)的基础，C++98 的[exception specification](https://en.cppreference.com/w/cpp/language/except_spec)在 C++17 移除
* 函数是否要加上 noexcept 声明与接口设计相关，调用者可以查询函数的 noexcept 状态，查询结果将影响代码的异常安全性和执行效率。**因此函数是否要声明为 noexcept 就和成员函数是否要声明为 const 一样重要**，如果一个函数不抛异常却不为其声明 noexcept，这就是接口规范缺陷
* **noexcept 的一个额外优点是，它可以让编译器生成更好的目标代码**。为了理解原因只需要考虑 C++98 和 C++11 表达函数不抛异常的区别
```cpp
int f(int x) throw(); // C++98
int f(int x) noexcept; // C++11
```
* 如果一个异常在运行期逃出函数，则[exception specification](https://en.cppreference.com/w/cpp/language/except_spec)被违反。
*  C++98 中，**调用栈会展开到函数调用者**，执行一些无关的动作后中止程序。
* C++11 的一个微小区别是是，在程序中止前只是**可能展开栈**。这微小的区别将对代码生成造成巨大的影响
* noexcept 声明的函数中，**如果异常传出函数，优化器不需要保持栈在运行期的展开状态**，也不需要在异常逃出时，保证其中所有的对象按构造顺序的逆序析构。
```cpp
RetType function(params) noexcept; // most optimizable
RetType function(params) throw(); // less optimizable
RetType function(params); // less optimizable
```
* 这个理由已经足够支持给任何已知**不会抛异常的函数加上 noexcept**，**比如移动操作**
* [std::vector::push_back](https://en.cppreference.com/w/cpp/container/vector/push_back)在**容器空间不够容纳元素时，会扩展新内存块**，**再元素转移**到新的内存块。C++98 的做法是**先逐个拷贝，后析构**旧内存的对象，这使得[push_back](https://en.cppreference.com/w/cpp/container/vector/push_back)提供强异常安全保证：如果拷贝元素的过程中抛出异常，则[std::vector](https://en.cppreference.com/w/cpp/container/vector)保持原样，因为旧内存元素还未被析构
* [std::vector::push_back](https://en.cppreference.com/w/cpp/container/vector/push_back)在 C++11 中的优化是把拷贝替换成移动，但为了不违反强异常安全保证，只有确保元素的**移动操作不抛异常时才会用移动替代拷贝**
* swap 函数是需要 noexcept 声明的另一个例子，不过标准库的 swap 用[noexcept操作符](https://en.cppreference.com/w/cpp/language/noexcept)的结果决定
```cpp
// 数组的swap
template <class T, size_t N>
void swap(T (&a)[N], T (&b)[N]) noexcept(noexcept(swap(*a, *b))); // 由元素类型决定noexcept结果
// 比如元素类型是class A，如果swap(A, A)不抛异常则该数组的swap也不抛异常

// std::pair的swap
template <class T1, class T2>
struct pair {
    …
    void swap(pair& p) noexcept(noexcept(swap(first, p.first)) &&
        noexcept(swap(second, p.second)));
    …
};
```
* 虽然 noexcept 有优化的好处，但将函数声明为 noexcept 的前提是，**保证函数长期具有 noexcept 性质**，如果之后随意移除 noexcept 声明，就有破坏客户代码的风险
* 大多数函数是异常中立的，它们**本身不抛异常，但它们调用的函数可能抛异常**，这样它们就允许抛出的异常传到调用栈的更深一层，因此**异常中立函数天生永远不具备 noexcept 性质**
* 如果为了强行加上 noexcept 而修改实现就是本末倒置，比如调用一个会抛异常的函数是最简单的实现，为了不抛异常而环环相扣地来隐藏这点（比如捕获所有异常，将其替换成状态码或特殊返回值），大大增加了理解和维护的难度，并且这些复杂性的时间成本可能超过 noexcept 带来的优化
* 对某些函数来说，noexcept 性质十分重要，**内存释放函数**和**所有的析构函数都隐式 noexcept，**这样就不必加 noexcept 声明。析构函数唯一未隐式 noexcept 的情况是，类中有数据成员的类型显式将析构函数声明 noexcept(false)。但这样的析构函数很少见，标准库中一个也没有
* 有些库的接口设计者会把函数区分为 wide contract 和 narrow contract
* wide contract 函数没有前置条件，不用关心程序状态，对传入的实参没有限制，一定不会有未定义行为，如果知道不会抛异常就可以加上 noexcept
* narrow contract 函数有前置条件，如果条件被违反则结果未定义。但函数没有义务校验这个前置条件，它断言前置条件一定满足（调用者负责保证断言成立），因此加上 noexcept 声明也是合理的
```cpp
// 假设前置条件是s.length() <= 32
void f(const std::string& s) noexcept;
```
* 但如果想在违反前置条件时抛出异常，由于函数的 noexcept 声明，异常就会导致程序中止，因此一般只为 wide contract 函数声明 noexcept
* 在 noexcept 函数中调用可能抛异常的函数时，编译器不会帮忙给出警告
```cpp
void start();
void finish();
void f() noexcept
{
    start();
    … // do the actual work
    finish();
}
```
* 带 noexcept 声明的函数调用了不带 noexcept 声明的函数，这看起来自相矛盾，但也许被调用的函数在文档中写明了不会抛异常，也许它们来自 C 语言的库，也许来自还没来得及根据 C++11 标准做修订的 C++98 库

## 15 用[constexpr](https://en.cppreference.com/w/cpp/language/constexpr)表示编译期常量
*  c**onstexpr 用于对象时就是一个加强版的 const**，表面上看 constexpr 表示值是 const，且在编译期（严格来说是翻译期，包括编译和链接，如果不是编译器或链接器作者，无需关心这点区别）已知，但用于函数则有不同的意义
* **编译期已知的值可能被放进只读内存**，这对嵌入式开发是一个很重要的语法特性
```cpp
int i = 42;
constexpr auto j = i; // 错误：i的值在编译期未知
std::array<int, i> v1; // 错误：同上
constexpr auto n = 10; // OK：10是一个编译期常量
std::array<int, n> v2; // OK：n的值是在编译期已知
```
* **constexpr 函数**在调用时若**传入的是编译期常量**，则产出编译期常量，**传入运行期才知道的值**，则产出运行期值。constexpr 函数可以满足所有需求，因此不必为了有非编译期值的情况而写两个函数
```cpp
constexpr int pow(int base, int exp) noexcept
{
    … // 实现见后
}

constexpr auto n = 5;
std::array<int, pow(3, n)> results; // pow(3, n)在编译期计算出结果
```
* 上面的 **constexpr 并不表示函数要返回 const 值**，而是表示，如果参数都是编译期常量，则返回结果就可以当编译期常量使用，如果有一个不是编译期常量，返回值就在运行期计算
```cpp
auto base = 3; // 运行期获取值
auto exp = 10; // 运行期获取值
auto baseToExp = pow(base, exp); // pow在运行期被调用
```
* C++11 中，constexpr 函数只能包含一条语句，即一条 return 语句。有两个应对限制的技巧：用条件运算符 ?: 替代 if-else、用递归替代循环
```cpp
constexpr int pow(int base, int exp) noexcept
{
    return (exp == 0 ? 1 : base * pow(base, exp - 1));
}
```
* C++14 解除了此限制
```cpp
// C++14
constexpr int pow(int base, int exp) noexcept
{
    auto result = 1;
    for (int i = 0; i < exp; ++i) result *= base;
    return result;
}
```
* constexpr 函数必须传入和返回[literal type](https://en.cppreference.com/w/cpp/named_req/LiteralType)。constexpr 构造函数可以让自定义类型也成为[literal type](https://en.cppreference.com/w/cpp/named_req/LiteralType)
```cpp
class Point {
public:
    constexpr Point(double xVal = 0, double yVal = 0) noexcept
    : x(xVal), y(yVal) {}
    constexpr double xValue() const noexcept { return x; }
    constexpr double yValue() const noexcept { return y; }
    void setX(double newX) noexcept { x = newX; } // 修改了对象所以不能声明为constexpr
    void setY(double newY) noexcept { y = newY; } // 另外C++11中constexpr函数返回类型不能是void
private:
    double x, y;
};

constexpr Point p1(9.4, 27.7); // 编译期执行constexpr构造函数
constexpr Point p2(28.8, 5.3); // 同上

// 通过constexpr Point对象调用xValue和yValue也会在编译期获取值
// 于是可以再写出一个新的constexpr函数
constexpr Point midpoint(const Point& p1, const Point& p2) noexcept
{
    return { (p1.xValue() + p2.xValue()) / 2, (p1.yValue() + p2.yValue()) / 2 };
}
constexpr auto mid = midpoint(p1, p2); // mid在编译期创建
```
* 因为 mid 是编译期已知值，这就意味着如下表达式可以用于模板形参
```cpp
mid.xValue()*10
// 因为上式是浮点型，浮点型不能用于模板实例化，因此还要如下转换一次
static_cast<int>(mid.xValue()*10)
```
* C++14 允许对值进行了**修改**或**无返回值的函数**声明为 constexpr
```cpp
// C++14
class Point {
public:
    constexpr Point(double xVal = 0, double yVal = 0) noexcept
    : x(xVal), y(yVal) {}
    constexpr double xValue() const noexcept { return x; }
    constexpr double yValue() const noexcept { return y; }
    constexpr void setX(double newX) noexcept { x = newX; }
    constexpr void setY(double newY) noexcept { y = newY; }
private:
    double x, y;
};

// 于是C++14允许写出下面的代码
constexpr Point reflection(const Point& p) noexcept // 返回p关于原点的对称点
{
    Point res;
    res.setX(-p.xValue());
    res.setY(-p.yValue());
    return res;
}

constexpr Point p1(9.4, 27.7);
constexpr Point p2(28.8, 5.3);
constexpr auto mid = midpoint(p1, p2);
constexpr auto reflectedMid = reflection(mid); // 值为(-19.1, -16.5)，且在编译期已知
```
* 使用 constexpr 的前提是必须长期保证需要它，因为如果后续要删除 constexpr 可能会导致许多错误

## 16 用[std::mutex](https://en.cppreference.com/w/cpp/thread/mutex)或[std::atomic](https://en.cppreference.com/w/cpp/atomic/atomic)保证const成员函数线程安全
* 假设有一个表示多项式的类，它包含一个返回根的 const 成员函数
```cpp
class Polynomial {
public:
    std::vector<double> roots() const
    { // 实际仍需要修改值，所以将要修改的成员声明为mutable
        if (!rootsAreValid)
        {
            … // 计算根
            rootsAreValid = true;
        }
        return rootVals;
    }
private:
    mutable bool rootsAreValid{ false };
    mutable std::vector<double> rootVals{};
};
```
* 假如此时**有两个线程对同一个对象调用成员函数**，虽然函数声明为 const，但由于函数内部修改了数据成员，就**可能产生数据竞争**。最简单的解决方法是引入一个[std::mutex](https://en.cppreference.com/w/cpp/thread/mutex)
```cpp
class Polynomial {
public:
    std::vector<double> roots() const
    {
        std::lock_guard<std::mutex> l(m);
        if (!rootsAreValid)
        {
            … // 计算根
            rootsAreValid = true;
        }
        return rootVals;
    }
private:
    mutable std::mutex m; // std::mutex是move-only类型，因此这个类只能移动不能拷贝
    mutable bool rootsAreValid{ false };
    mutable std::vector<double> rootVals{};
};
```
* 对一些简单的情况，**使用原子变量[std::atomic](https://en.cppreference.com/w/cpp/atomic/atomic)可能开销更低**（取决于机器及[std::mutex](https://en.cppreference.com/w/cpp/thread/mutex)的实现）
```cpp
class Point {
public:
    double distanceFromOrigin() const noexcept
    {
        ++callCount; // 计算调用次数
        return std::sqrt((x * x) + (y * y));
    }
private:
    mutable std::atomic<unsigned> callCount{ 0 }; // std::atomic也是move-only类型
    double x, y;
};
```
* 因为[std::atomic](https://en.cppreference.com/w/cpp/atomic/atomic)的开销比较低，**多个原子变量来同步，巨坑！！！**
```cpp
class A {
public:
    int f() const
    {
        if (flag) return res;
        else
        {
            auto x = expensiveComputation1();
            auto y = expensiveComputation2();
            res = x + y;
            flag = true; // 设置标记
            return res;
        }
    }
private:
    mutable std::atomic<bool> flag{ false };
    mutable std::atomic<int> res;
};
```
* 这样做可行，但如果多个线程同时观察到标记值为 false，每个线程都要继续进行运算，这个标记反而没起到作用。先设置标记再计算可以消除这个问题，但会引起一个更大的问题
```cpp
class A {
public:
    int f() const
    {
        if (flag) return res;
        else
        {
            flag = true; // 在计算前设置标记值为true
            auto x = expensiveComputation1();
            auto y = expensiveComputation2();
            res = x + y;
            return res;
        }
    }
private:
    mutable std::atomic<bool> flag{ false };
    mutable std::atomic<int> res;
};
```
* 假如线程1刚设置好标记，线程2此时正好检查到标记值为 true 并直接返回数据值，然后线程1接着计算结果，这样线程2的返回值就是错的
* 因此如果要**同步多个变量或内存区**，最好还是使用[std::mutex](https://en.cppreference.com/w/cpp/thread/mutex)
```cpp
class A {
public:
    int f() const
    {
        std::lock_guard<std::mutex> l(m);
        if (flag) return res;
        else
        {
            auto x = expensiveComputation1();
            auto y = expensiveComputation2();
            res = x + y;
            flag = true;
            return res;
        }
    }
private:
    mutable std::mutex m;
    mutable bool flag{ false };
    mutable int res;
};
```

## 17 特殊成员函数的隐式合成与抑制机制
* C++11 中的特殊成员函数多了两个：**移动构造**函数和**移动赋值**运算符
```cpp
class A {
public:
    A(A&& rhs); // 移动构造函数 ，右值移动
    A& operator=(A&& rhs); // 移动赋值运算符 ，右值移动
};
```
* **移动操作同样会在需要时生成**，执行的是对 **non-static 成员**的移动操作，也会对**基类部分**执行移动操作
* 移动操作并不确保真正移动，其核心是把[std::move](https://en.cppreference.com/w/cpp/utility/move)用于每个要移动的对象，根据**返回值的重载解析决定执行移动还是拷贝**。因此按成员移动分为两部分：对支持移动操作的类型进行移动，对不可移动的类型执行拷贝
* 两种拷贝操作（**拷贝构造**函数和**拷贝赋值**运算符）是独立的，声明其中一个不会阻止编译器生成另一个
* **两种移动操作是不独立的**，声明其中一个将阻止编译器生成另一个。理由是如果声明了移动构造函数，可能意味着实现上与编译器默认按成员移动的移动构造函数有所不同，从而可以推断移动赋值操作也应该与默认行为不同
* 显式声明拷贝操作（**即使声明为 =delete**）**会阻止自动生成移动操作**（但声明为 =default 不阻止生成）。理由类似上条，声明拷贝操作可能意味着默认的拷贝方式不适用，从而推断移动操作也应该会默认行为不同
* 反之亦然，声明移动操作也会阻止生成拷贝操作
* C++11 规定，**显式声明析构函数会阻止生成移动操作**。这个规定源于 Rule of Three，即两种拷贝函数和析构函数应该一起声明。这个规则的推论是，如果声明了析构函数，则说明默认的拷贝操作也不适用，但 C++98 中没有重视这个推论，因此仍可以生成拷贝操作，而在 C++11 中为了保持不破坏遗留代码，保留了这个规则。由于析构函数和拷贝操作需要一起声明，加上声明了拷贝操作会阻止生成移动操作，于是 C++11 就有了这条规定
* 最终，**生成移动操作的条件**必须满足：该类**没有声明的拷贝、移动、析构**中的任何一个函数
* 总有一天这个规则会扩展到拷贝操作，因为 C++11 规定存在拷贝操作或析构函数时，仍能生成拷贝操作是被废弃的行为。**C++11 提供了 =default 来表示使用默认行为，而不抑制生成其他函数**
* 这种手法对于**多态基类很有用**，多态基类一般会有虚析构函数，虚析构函数的默认实现一般是正确的，**为了使用默认行为而不阻止生成移动操作，则应该使用 =default**，同理，如果要使用默认的移动操作而不阻止生成拷贝操作，则应该给移动操作加上 =default
```cpp
class A {
public:
    virtual ~A() = default;
    A(A&&) = default; // support moving
    A& operator=(A&&) = default;
    A(const A&) = default; // support copying
    A& operator=(const A&) = default;
};
```
* 事实上不需要思考太多限制，如果需要**默认操作就使用 =default**，虽然麻烦一些，但可以避免许多问题
```cpp
class StringTable {
public:
    … // 实现插入、删除、查找等函数
private:
    std::map<int, std::string> values;
};
```
* 上面的类没有声明任何特殊成员函数，编译器将在需要时自动合成。假设过了一段时间后，想扩充一些行为，比如记录构造和析构日志
```cpp
class StringTable {
public:
    StringTable() { makeLogEntry("Creating StringTable object"); }
    ~StringTable() { makeLogEntry("Destroying StringTable object"); }
    …
private:
    std::map<int, std::string> values;
};
```
* **这时析构函数就会阻止生成移动操作**，但针对移动操作的测试可以通过编译，因为在不可移动时会使用拷贝操作，而这很难被察觉。执行移动的代码实际变成了拷贝，而这一切只是源于添加了一个析构函数。避免这个问题也不是难事，只需要一开始把**拷贝和移动操作声明为 =default**
* 另外还有默认构造函数和析构函数的生成未被提及，这里将统一总结
  * **默认构造函数**：和 C++98 相同，只在类中不存在用户声明的构造函数时生成
  * **析构函数**：
    * 和 C++98 基本相同，唯一的区别是**默认为 noexcept**
    * 和 C++98 相同，只有**基类的析构函数为虚函数**，**派生类的析构函数才为虚函数**
  * 拷贝构造函数：
    * 仅当类中不存在用户声明的拷贝构造函数时生成
    * 如果**声明了移动操作**，**则拷贝构造函数被删除**
    * 如果声明了拷贝赋值运算符或析构函数，仍能生成拷贝构造函数，但这是被废弃的行为
  * 拷贝赋值运算符：
    * 仅当类中不存在用户声明的拷贝赋值运算符时生成
    * 如果**声明了移动操作，则拷贝赋值运算符被删除**
    * 如果声明了拷贝构造函数或析构函数，仍能生成拷贝赋值运算符，但这是**被废弃的行为**
  * **移动操作：仅当类中不存在任何用户声明的拷贝操作、移动操作、析构函数时生成**
* 注意，**这些机制中提到的是成员函数而非成员函数模板**，模板并不会影响特殊成员函数的合成
```cpp
class A {
public:
    template<typename T>
    A(const T& rhs); // 从任意类型构造
    template<typename T>
    A& operator=(const T& rhs); // 从任意类型赋值
    …
};
```
* 上述模板不会阻止编译器生成拷贝和移动操作，即使模板的实例化和拷贝操作签名相同（即 T 是 A）

* **原始指针的缺陷有**：
  * 声明中**未指出指向**的是**单个**对象还是**数组**
  * 没有提示**使用完对象后是否需要析构**，从声明中**无法看出指针是否拥有对象**
  * **不知道析构该使用 delete 还是其他方式**（比如传入一个专门用于析构的函数）
  * 即使知道了使用 delete，**也不知道 delete 的是单个对象还是数组**（使用 delete[]）
  * **难以保证**所有路径上**只产生一次析构**
  * **没有检查空悬指针的办法**
* **智能指针解决了这些问题**，它封装了原始指针，行为看起来和原始指针类似但大大减少了犯错的可能
* C++17 中有三种智能指针：[std::shared_ptr](https://en.cppreference.com/w/cpp/memory/shared_ptr)、[std::unique_ptr](https://en.cppreference.com/w/cpp/memory/unique_ptr)、[std::weak_ptr](https://en.cppreference.com/w/cpp/memory/weak_ptr)

## 18 用[std::unique_ptr](https://en.cppreference.com/w/cpp/memory/unique_ptr)管理所有权唯一的资源
* 使用智能指针时一般**首选**[std::unique_ptr](https://en.cppreference.com/w/cpp/memory/unique_ptr)，默认情况下它和原始指针尺寸相同
* [std::unique_ptr](https://en.cppreference.com/w/cpp/memory/unique_ptr)对资源拥有**唯一所有权**，因此**它是move-obly类型，不允许拷贝**。它常用**作工厂函数的返回类型**，这样工厂函数生成的对象在需要销毁时会被自动析构，而不需要手动析构
```cpp
class A {};

std::unique_ptr<A> makeA()
{
    return std::unique_ptr<A>(new A);
}

auto p = makeA();
```
* [std::unique_ptr](https://en.cppreference.com/w/cpp/memory/unique_ptr)的**析构默认通过 delete** 内部的原始指针完成，但**也可以自定义删除器**，删除器需要一个[std::unique_ptr](https://en.cppreference.com/w/cpp/memory/unique_ptr)**内部指针类型的参数**
```cpp
class A {};

auto f = [](A* p) { std::cout << "destroy\n"; delete p; };//删除器

std::unique_ptr<A, decltype(del)> makeA()
{
    std::unique_ptr<A, decltype(f)> p(new A, f);
    return p;
}
```
* 使用 C++14 的 **auto 返回类型**，可以将删除器的 **lambda 定义在工厂函数内**，封装性更好一些
```cpp
class A {};

auto makeA()
{
    auto f = [](A* p) { std::cout << "destroy\n"; delete p; };
    std::unique_ptr<A, decltype(f)> p(new A, f);
    return p;
}
```
* 可以进一步扩展成支持**继承体系的工厂函数**
```cpp
class A {
public:
    virtual ~A() {}    // 删除器对任何对象调用的是基类的析构函数，因此必须声明为虚函数
};
class B : public A {}; // 基类的析构函数为虚函数，则派生类的析构函数默认为虚函数
class C : public A {};
class D : public A {};

auto makeA(int i)
{
    auto f = [](A* p) { std::cout << "destroy\n"; delete p; };
    std::unique_ptr<A, decltype(f)> p(nullptr, f);
    if(i == 1) p.reset(new B);
    else if(i == 2) p.reset(new C);
    else p.reset(new D);
    return p;
}
```
* 默认情况下，[std::unique_ptr](https://en.cppreference.com/w/cpp/memory/unique_ptr)和原始指针尺寸相同，如果自定义删除器则[std::unique_ptr](https://en.cppreference.com/w/cpp/memory/unique_ptr)会加上删除器的尺寸。一般无状态的函数对象（如无捕获的 lambda）不会浪费任何内存，**lambda作为删除器可以节约空间**
```cpp
class A{};

auto f = [](A* p) { delete p; };
void g(A* p) { delete p; }
struct X {
    void operator()(A* p) const { delete p; }
};

std::unique_ptr<A> p1(new A);
std::unique_ptr<A, decltype(f)> p2(new A, f); //无捕获lambda
std::unique_ptr<A, decltype(g)*> p3(new A, g);//函数指针
std::unique_ptr<A, decltype(X())> p4(new A, X());

// 机器为64位
std::cout 
    << sizeof(p1)  // 8：默认尺寸，即一个原始指针的尺寸
    << sizeof(p2)  // 8：无捕获lambda不会浪费尺寸
    << sizeof(p3)  // 16：函数指针占一个原始指针尺寸
    << sizeof(p4); // 8：无状态的函数对象。但如果X中存储了状态（如数据成员、虚函数）就会增加尺寸
```
* [std::unique_ptr](https://en.cppreference.com/w/cpp/memory/unique_ptr)作为返回类型的另一个方便之处是，**可以转**为[std::shared_ptr](https://en.cppreference.com/w/cpp/memory/shared_ptr)
```cpp
// std::make_unique的返回类型是std::unique_ptr
std::shared_ptr<int> p = std::make_unique<int>(42); // OK
```
* [std::unique_ptr](https://en.cppreference.com/w/cpp/memory/unique_ptr)针对数组提供了一个**特化版本**，此版本提供[operator[]](https://en.cppreference.com/w/cpp/memory/unique_ptr/operator_at)，但不提供[单元素版本的operator\*和operator-\>](https://en.cppreference.com/w/cpp/memory/unique_ptr/operator*)，这样对其指向的对象就不存在二义性
```cpp
std::unique_ptr<int[]> p(new int[3]{11, 22, 33});
for(int i = 0; i < 3; ++i) std::cout << p[i];
```

## 19 用[std::shared_ptr](https://en.cppreference.com/w/cpp/memory/shared_ptr)管理所有权可共享的资源
* [std::shared_ptr](https://en.cppreference.com/w/cpp/memory/shared_ptr)内部有一个引用计数，用来存储资源被共享的次数。因为内部**多了一个指向引用计数的指针**，所以[std::shared_ptr](https://en.cppreference.com/w/cpp/memory/shared_ptr)的尺寸是原始指针的两倍
```cpp
int* p = new int(42);
auto q = std::make_shared<int>(42);
std::cout << sizeof(p) << sizeof(q); // 816
```
* [std::shared_ptr](https://en.cppreference.com/w/cpp/memory/shared_ptr)**保证线程安全**，因此引用计数的递增和递减是原子操作，原子操作一般比非原子操作慢
* [std::shared_ptr](https://en.cppreference.com/w/cpp/memory/shared_ptr)默认析构方式和[std::unique_ptr](https://en.cppreference.com/w/cpp/memory/unique_ptr)一样，**也是 delete 内部的原始指针**，同样可以自定义删除器，不过不需要在模板参数中指明删除器类型
```cpp
class A {};
auto f = [](A* p) { delete p; };

std::unique_ptr<A, decltype(f)> p(new A, f);
std::shared_ptr<A> q(new A, f);
```
* 模板参数不使用删除器的设计在使用上带来了一些弹性
```cpp
std::shared_ptr<A> p(new A, f);
std::shared_ptr<A> q(new A, g);
// 使用不同的删除器但具有相同的类型，因此可以放进同一容器
std::vector<std::shared_ptr<A>> v{ p, q };
```
* 删除器不影响[std::shared_ptr](https://en.cppreference.com/w/cpp/memory/shared_ptr)的尺寸，因为删除器不是[std::shared_ptr](https://en.cppreference.com/w/cpp/memory/shared_ptr)的一部分，而是位于堆上或自定义分配器的内存位置。[std::shared_ptr](https://en.cppreference.com/w/cpp/memory/shared_ptr)有一个 control block，它包含了引用计数的指针和自定义删除器的拷贝，以及一些其他数据（比如弱引用计数）

![](./images/4-1.png)

* [std::shared_ptr](https://en.cppreference.com/w/cpp/memory/shared_ptr)内部实现如下
```cpp
template<typename T>
struct sp_element {
    using type = T;
};

template<typename T>
struct sp_element<T[]> {
    using type = T;
};

template<typename T, std::size_t N>
struct sp_element<T[N]> {
    using type = T;
};

template<typename T>
class shared_ptr {
    using elem_type = typename sp_element<T>::type;
    elem_type* px; // 内部指针
    shared_count pn; // 引用计数
    template<typename U> friend class shared_ptr;
    template<typename U> friend class weak_ptr;
};

class shared_count {
    sp_counted_base* pi;
    int shared_count_id;
    friend class weak_count;
};

class weak_count {
    sp_counted_base* pi;
};

class sp_counted_base {
    int use_count; // 引用计数
    int weak_count; // 弱引用计数
};

template<typename T>
class sp_counted_impl_p: public sp_counted_base {
    T* px; // 删除器
};
```
* control block 在创建第一个[std::shared_ptr](https://en.cppreference.com/w/cpp/memory/shared_ptr)时确定，因此 **control block 的创建**发生在如下时机
  * 调用[std::make_shared](https://en.cppreference.com/w/cpp/memory/shared_ptr/make_shared)时：调用时生成一个新对象，此时显然不会有关于该对象的 control block
  * 从[std::unique_ptr](https://en.cppreference.com/w/cpp/memory/unique_ptr)构造[std::shared_ptr](https://en.cppreference.com/w/cpp/memory/shared_ptr)时：因为[std::unique_ptr](https://en.cppreference.com/w/cpp/memory/unique_ptr)没有 control block
  * 用原始指针**构造**[std::shared_ptr](https://en.cppreference.com/w/cpp/memory/shared_ptr)时
* 这意味着用**同一个原始指针构造多个**[std::shared_ptr](https://en.cppreference.com/w/cpp/memory/shared_ptr)，将创建多个 control block，即有多个引用指针，当引用指针变为零时就**会出现多次析构的错误**
```cpp
int main()
{
    {
        int* i = new int(42);
        std::shared_ptr<int> p(i);
        std::shared_ptr<int> q(i);
    } // 错误
}
```
* 使用[std::make_shared](https://en.cppreference.com/w/cpp/memory/shared_ptr/make_shared)就不会有这个问题
```cpp
auto p = std::make_shared<int>(42);
```
* 但[std::make_shared](https://en.cppreference.com/w/cpp/memory/shared_ptr/make_shared)不支持自定义删除器，这时**应该直接传递 new 的结果**
```cpp
auto f = [](int*) {};
std::shared_ptr<int> p(new int(42), f);
```
* 用类的 this 指针构造[std::make_shared](https://en.cppreference.com/w/cpp/memory/shared_ptr/make_shared)时，**\*this 的所有权不会被共享**
```cpp
class A {
public:
    std::shared_ptr<A> f() { return std::shared_ptr<A>(this); }
};

auto p = std::make_shared<A>();
auto q = p->f();
std::cout << p.use_count() << q.use_count(); // 11
```
* 为了解决这个问题，需要继承[std::enable_shared_from_this](https://en.cppreference.com/w/cpp/memory/enable_shared_from_this)，通过其提供的[shared_from_this](https://en.cppreference.com/w/cpp/memory/enable_shared_from_this/shared_from_this)**获取 \*this 的所有权**
```cpp
class A : public std::enable_shared_from_this<A> {
public:
    std::shared_ptr<A> f() { return shared_from_this(); }
};

auto p = std::make_shared<A>();
auto q = p->f();
std::cout << p.use_count() << q.use_count(); // 22
```
* [shared_from_this](https://en.cppreference.com/w/cpp/memory/enable_shared_from_this/shared_from_this)的原理是为 \*this 的 control block 创建一个新的[std::shared_ptr](https://en.cppreference.com/w/cpp/memory/shared_ptr)，因此 **\*this 必须有一个已关联的 control block**，即有一个指向 \*this 的[std::shared_ptr](https://en.cppreference.com/w/cpp/memory/shared_ptr)，否则行为未定义，抛出[std::bad_weak_ptr](https://en.cppreference.com/w/cpp/memory/bad_weak_ptr)异常
```cpp
class A : public std::enable_shared_from_this<A> {
public:
    std::shared_ptr<A> f() { return shared_from_this(); }
};

auto p = new A;
auto q = p->f(); // 抛出std::bad_weak_ptr异常
```
* 为了**只允许创建用[std::shared_ptr](https://en.cppreference.com/w/cpp/memory/shared_ptr)指向的对象**，可以将**构造函数放进 private 作用域**，并提供一个返回[std::shared_ptr](https://en.cppreference.com/w/cpp/memory/shared_ptr)对象的工厂函数
```cpp
class A : public std::enable_shared_from_this<A> {
public:
    static std::shared_ptr<A> create() { return std::shared_ptr<A>(new A); }
    std::shared_ptr<A> f() { return shared_from_this(); }
private:
    A() = default;
};

auto p = A::create(); // 构造函数为private，auto p = new A将报错
auto q = p->f(); // OK
```

## 20 用[std::weak_ptr](https://en.cppreference.com/w/cpp/memory/weak_ptr)观测[std::shared_ptr](https://en.cppreference.com/w/cpp/memory/shared_ptr)的内部状态
* [std::weak_ptr](https://en.cppreference.com/w/cpp/memory/weak_ptr)不能解引用，它不是一种独立的智能指针，而是[std::shared_ptr](https://en.cppreference.com/w/cpp/memory/shared_ptr)的一种扩充，**它用[std::shared_ptr](https://en.cppreference.com/w/cpp/memory/shared_ptr)初始化**，**共享对象但不改变引用计数**，主要作用是观察[std::shared_ptr](https://en.cppreference.com/w/cpp/memory/shared_ptr)的内部状态
```cpp
std::weak_ptr<int> w;

void f(std::weak_ptr<int> w)
{
    if (auto p = w.lock()) std::cout << *p;
    else std::cout << "can't get value";
}

int main()
{
    {
        auto p = std::make_shared<int>(42);
        w = p;
        assert(p.use_count() == 1);
        assert(w.expired() == false);
        f(w); // 42
        auto q = w.lock();
        assert(p.use_count() == 2);
        assert(q.use_count() == 2);
    }
    f(w); // can't get value
    assert(w.expired() == true);
    assert(w.lock() == nullptr);
}
```
* [std::weak_ptr](https://en.cppreference.com/w/cpp/memory/weak_ptr)的另一个作用是**解决循环引用**问题
```cpp
class B;
class A {
public:
    std::shared_ptr<B> b;
};

class B{
public:
    std::shared_ptr<A> a; // std::weak_ptr<A> a;
};

int main()
{
    {
        std::shared_ptr<A> x(new A);
        x->b = std::shared_ptr<B>(new B);
        x->b->a = x;
    } // x.use_count由2减为1，不会析构，于是x->b也不会析构，导致两次内存泄漏
    // 如果B::a改为std::weak_ptr，则use_count不会为2而保持1，此处就会由1减为0，从而正常析构
}
```

##  21 用[std::make_unique](https://en.cppreference.com/w/cpp/memory/unique_ptr/make_unique)（[std::make_shared](https://en.cppreference.com/w/cpp/memory/shared_ptr/make_shared)）创建[std::unique_ptr](https://en.cppreference.com/w/cpp/memory/unique_ptr)（[std::shared_ptr](https://en.cppreference.com/w/cpp/memory/shared_ptr)）


* C++14 提供了[std::make_unique](https://en.cppreference.com/w/cpp/memory/unique_ptr/make_unique)，C++11 可以手动实现一个基础功能版
```cpp
template<typename T, typename... Ts>
std::unique_ptr<T> make_unique(Ts&&... params)
{
    return std::unique_ptr<T>(new T(std::forward<Ts>(params)...));
}
```
* 这个基础函数不支持数组和自定义删除器，但这些不难实现。从这个基础函数可以看出，make函数把实参完美转发给构造函数并返回构造出的智能指针。除了[std::make_shared](https://en.cppreference.com/w/cpp/memory/shared_ptr/make_shared)和[std::make_unique](https://en.cppreference.com/w/cpp/memory/unique_ptr/make_unique)，还有一个make函数是[std::allocate_shared](https://en.cppreference.com/w/cpp/memory/shared_ptr/allocate_shared)，它的行为和[std::make_shared](https://en.cppreference.com/w/cpp/memory/shared_ptr/make_shared)一样，只不过第一个实参是分配器对象
* 优先使用make函数的一个明显原因就是**只需要写一次类型**
```cpp
auto p = std::make_unique<int>(42);
std::unique_ptr<int> q(new int(42);
```
* 另一个原因与**异常安全**相关
```cpp
void f(std::shared_ptr<A> p, int n) {}
int g() { return 1; }
f(std::shared_ptr<A>(new A), g()); // 潜在的内存泄露隐患
// g可能运行于new A还未返回给std::shared_ptr的构造函数时
// 此时如果g抛出异常，则new A就发生了内存泄漏
f(std::make_shared<A>(), g()); // 不会发生内存泄漏，且只需要一次内存分配
```
* make 函数有两个限制，一是它**无法定义删除器**
```cpp
auto f = [] (A* p) { delete p; };
std::unique_ptr<A, decltype(f)> p(new A, f);
std::shared_ptr<A> q(new A, f);
```
* 使用**自定义删除器**，但又想**避免内存泄漏**，解决方法是**单独用一条语句来创建**[std::shared_ptr](https://en.cppreference.com/w/cpp/memory/shared_ptr)
```cpp
auto d = [] (A* p) { delete p; };
std::shared_ptr<A> p(new A, d); // 如果发生异常，删除器将析构new创建的对象
f(std::move(p), g());
```
* make 函数的第二个限制是，**make 函数中的完美转发使用的是小括号初始化**，在持有[std::vector](https://en.cppreference.com/w/cpp/container/vector)类型时，设置初始化值不如大括号初始化方便。一个不算直接的解决方法是，先构造一个[std::initializer_list](https://en.cppreference.com/w/cpp/utility/initializer_list)再传入
```cpp
auto p = std::make_unique<std::vector<int>>(3, 6); // vector中是3个6
auto q = std::make_shared<std::vector<int>>(3, 6); // vector中是3个6

auto x = { 1, 2, 3, 4, 5, 6 };
auto p2 = std::make_unique<std::vector<int>>(x);
auto q2 = std::make_shared<std::vector<int>>(x);
```
* [std::make_unique](https://en.cppreference.com/w/cpp/memory/unique_ptr/make_unique)只存在这两个限制，但[std::make_shared](https://en.cppreference.com/w/cpp/memory/shared_ptr/make_shared)和[std::allocate_shared](https://en.cppreference.com/w/cpp/memory/shared_ptr/allocate_shared)还有两个限制
  * 如果类重载了[operator new](https://en.cppreference.com/w/cpp/memory/new/operator_new)和[operator delete](https://en.cppreference.com/w/cpp/memory/new/operator_delete)，其针对的内存尺寸一般为类的尺寸，而[std::shared_ptr](https://en.cppreference.com/w/cpp/memory/shared_ptr)还要**加上 control block 的尺寸**，因此[std::make_shared](https://en.cppreference.com/w/cpp/memory/shared_ptr/make_shared)不适用重载了[operator new](https://en.cppreference.com/w/cpp/memory/new/operator_new)和[operator delete](https://en.cppreference.com/w/cpp/memory/new/operator_delete)的类
  * [std::make_shared](https://en.cppreference.com/w/cpp/memory/shared_ptr/make_shared)使[std::shared_ptr](https://en.cppreference.com/w/cpp/memory/shared_ptr)的 control block 和管理的对象在同一内存上分配（**比用new构造智能指针在尺寸和速度上更优的原因**），对象在引用计数为0时被析构，但其占用的内存直到 control block 被析构时才被释放，比如[std::weak_ptr](https://en.cppreference.com/w/cpp/memory/weak_ptr)会持续指向control block（为了检查引用计数以检查自身是否失效），control block 直到最后一个[std::shared_ptr](https://en.cppreference.com/w/cpp/memory/shared_ptr)和[std::weak_ptr](https://en.cppreference.com/w/cpp/memory/weak_ptr)被析构时才释放
* 假如对象尺寸很大，且最后一个[std::shared_ptr](https://en.cppreference.com/w/cpp/memory/shared_ptr)和[std::weak_ptr](https://en.cppreference.com/w/cpp/memory/weak_ptr)析构之间的时间间隔不能忽略，就会产生对象析构和内存释放之间的延迟
```cpp
auto p = std::make_shared<ReallyBigType>();
… // 创建指向该对象的多个std::shared_ptr和std::weak_ptr并做一些操作
… // 最后一个std::shared_ptr被析构，但std::weak_ptr仍存在
… // 此时，大尺寸对象占用内存仍未被回收
… // 最后一个std::weak_ptr被析构，control block和对象占用的同一内存块被释放
```
* 如果 new 构造[std::shared_ptr](https://en.cppreference.com/w/cpp/memory/shared_ptr)，最后一个[std::shared_ptr](https://en.cppreference.com/w/cpp/memory/shared_ptr)被析构时，内存就能立即被释放
```cpp
std::shared_ptr<ReallyBigType> p(new ReallyBigType);
… // 创建指向该对象的多个std::shared_ptr和std::weak_ptr并做一些操作
… // 最后一个std::shared_ptr被析构，std::weak_ptr仍存在，但ReallyBigType占用的内存立即被释放
… // 此时，仅control block内存处于分配而未回收状态
… // 最后一个std::weak_ptr被析构，control block的内存块被释放
```

## 22 用[std::unique_ptr](https://en.cppreference.com/w/cpp/memory/unique_ptr)实现[pimpl手法](https://en.cppreference.com/w/cpp/language/pimpl)必须在.cpp文件中提供析构函数定义

* [pimpl手法](https://en.cppreference.com/w/cpp/language/pimpl)就是把数据成员提取到类中，**用指向该类的指针替代原来的数据成员**。因为数据成员会影响内存布局，将数据成员用一个指针替代可以减少编译期依赖，保持 ABI 兼容
* 比如对如下类
```cpp
// A.h
#include <string>
#include <vector>

class A {
    int i;
    std::string s;
    std::vector<double> v;
};
```
* 使用[pimpl手法](https://en.cppreference.com/w/cpp/language/pimpl)后
```cpp
// A.h
class A {
    struct X {
    int i;
    std::string s;
    std::vector<double> v;
    };
public:
    A(): x(new X) {}
    ~A() { delete x; }
private:
    struct X;
    X* x;//原始指针
};
```
* 现在使用[std::unique_ptr](https://en.cppreference.com/w/cpp/memory/unique_ptr)**替代原始指针**，不再需要使用析构函数释放指针
```cpp
// A.h
#include <memory>

class A {
public:
    A();
private:
    struct X;
    std::unique_ptr<X> x; //替代原始指针
};

// A.cpp
#include "A.h"
#include <string>
#include <vector>

struct A::X {
    int i;
    std::string s;
    std::vector<double> v;
};

A::A() : x(std::make_unique<X>()) {}
```
* 但调用上述代码会出错
```cpp
// main.cpp
#include "A.h"

int main()
{
    A a; // 错误：A::X是不完整类型
}
```
* 原因在于[std::unique_ptr](https://en.cppreference.com/w/cpp/memory/unique_ptr)**析构时会在内部调用默认删除器**，默认删除器的 delete 语句之前会用[static_assert](https://en.cppreference.com/w/cpp/language/static_assert)**断言指针指向的不是非完整类型**
```cpp
// 删除器的实现
template<class T>
struct default_delete // default deleter for unique_ptr
{
    constexpr default_delete() noexcept = default;
    
    template<class U, enable_if_t<is_convertible_v<U*, T*>, int> = 0>
    default_delete(const default_delete<U>&) noexcept
    { // construct from another default_delete
    }

    void operator()(T* p) const noexcept
    {
        static_assert(0 < sizeof(T), "can't delete an incomplete type");
        delete p;
    }
};
```

![](./images/4-2.png)

* 解决方法就是让析构[std::unique_ptr](https://en.cppreference.com/w/cpp/memory/unique_ptr)的代码看见完整类型，即让**析构函数的定义位于要析构的类型的定义之后**
```cpp
// A.h
#include <memory>

class A {
public:
    A();
    ~A();
private:
    struct X;
    std::unique_ptr<X> x;
};

// A.cpp
#include "A.h"
#include <string>
#include <vector>

struct A::X { //A::X的定义
    int i;
    std::string s;
    std::vector<double> v;
};

A::A() : x(std::make_unique<X>()) {}
A::~A() = default; // 必须位于A::X的定义之后
```
* 使用[pimpl手法](https://en.cppreference.com/w/cpp/language/pimpl)的类自然应该支持移动操作，但定义**析构函数会阻止默认生成移动操作**，因此会想到添加默认的移动操作声明
```cpp
// A.h
#include <memory>

class A {
public:
    A();
    ~A();
    A(A&&) = default;
    A& operator=(A&&) = default;
private:
    struct X;
    std::unique_ptr<X> x;
};

// A.cpp
#include "A.h"
#include <string>
#include <vector>

struct A::X {
    int i;
    std::string s;
    std::vector<double> v;
};

A::A() : x(std::make_unique<X>()) {}
A::~A() = default; // 必须位于A::X的定义之后
```
* 但调用移动操作会出现相同的问题
```cpp
// main.cpp
#include "A.h"

int main()
{
    A a;
    A b(std::move(a)); // 错误：使用了未定义类型A::X
    A c = std::move(a); // 错误：使用了未定义类型A::X
}
```
* 原因也一样，移动操作会先析构原有对象，调用删除器时触发断言。解决方法也一样，**让移动操作的定义位于要析构的类型**的定义之后
```cpp
// A.h
#include <memory>

class A {
public:
    A();
    ~A();
    A(A&&);
    A& operator=(A&&);
private:
    struct X;
    std::unique_ptr<X> x;
};

// A.cpp
#include "A.h"
#include <string>
#include <vector>

struct A::X {
    int i;
    std::string s;
    std::vector<double> v;
};

A::A() : x(std::make_unique<X>()) {}
A::A(A&&) = default;
A& A::operator=(A&&) = default;
A::~A() = default;
```
* **编译器不会为[std::unique_ptr](https://en.cppreference.com/w/cpp/memory/unique_ptr)这类move-only类型生成拷贝操作**，即使可以生成也只是拷贝指针本身（浅拷贝），因此**如果要提供拷贝操作，则需要自己编写**
```cpp
// A.h
#include <memory>

class A {
public:
    A();
    ~A();
    A(A&&);
    A& operator=(A&&);
    A(const A&);
    A& operator=(const A&);
private:
    struct X;
    std::unique_ptr<X> x;
};

// A.cpp
#include "A.h"
#include <string>
#include <vector>

struct A::X {
    int i;
    std::string s;
    std::vector<double> v;
};

A::A() : x(std::make_unique<X>()) {}
A::A(A&&) = default;
A& A::operator=(A&&) = default;
A::~A() = default;
A::A(const A& rhs) : x(std::make_unique<X>(*rhs.x)) {}
A& A::operator=(const A& rhs)
{
    *x = *rhs.x;
    return* this;
}
```
* 如果使用[std::shared_ptr](https://en.cppreference.com/w/cpp/memory/shared_ptr)，则不需要关心上述所有问题
```cpp
// A.h
#include <memory>

class A {
public:
    A();
private:
    struct X;
    std::shared_ptr<X> x;
};

// A.cpp
#include "A.h"
#include <string>
#include <vector>

struct A::X {
    int i;
    std::string s;
    std::vector<double> v;
};

A::A() : x(std::make_shared<X>()) {}
```
* 实现[pimpl手法](https://en.cppreference.com/w/cpp/language/pimpl)时，[std::unique_ptr](https://en.cppreference.com/w/cpp/memory/unique_ptr)尺寸更小，运行更快一些，但必须在实现文件中指定特殊成员函数，[std::shared_ptr](https://en.cppreference.com/w/cpp/memory/shared_ptr)开销大一些，但不需要考虑因为删除器引发的一系列问题。但对于[pimpl手法](https://en.cppreference.com/w/cpp/language/pimpl)来说，主类和数据成员类之间是专属所有权的关系，[std::unique_ptr](https://en.cppreference.com/w/cpp/memory/unique_ptr)更合适。如果在需要共享所有权的特殊情况下，[std::shared_ptr](https://en.cppreference.com/w/cpp/memory/shared_ptr)更合适

* 移动语义使编译器可以用开销较低的移动操作替换昂贵的拷贝操作（*但不是所有情况下移动都会比拷贝快*），是 move-only 类型对象的支持基础
* 完美转发可以将某个函数模板的实参转发给其他函数，转发后的实参保持完全相同的值类型（*左值、右值*）
* 右值引用是移动语义和完美转发的实现基础，它引入了一种新的引用符号（*&&*）来区别于左值引用
* 这些名词很直观，但概念上容易与名称类似的函数混淆
  * **移动操作**的函数要求传入的实参是右值，无法传入左值，因此需要一个能把左值转换为右值的办法，这就是[std::move](https://en.cppreference.com/w/cpp/utility/move)做的事。[std::move](https://en.cppreference.com/w/cpp/utility/move)本身不进行移动，只是将**实参强制转换为右值**，以允许把转换的结果传给移动函数
  * **完美转发**指的是，将**函数模板的实参转发给另一个函数**，同时保持实参传入给模板时的值类型（*传入的实参是左值则转发后仍是左值，是右值则转发后仍是右值*）。如果不做任何处理的话，不论是传入的是左值还是右值，在传入之后都会变为左值，因此需要一个转换到右值的操作。[std::move](https://en.cppreference.com/w/cpp/utility/move)可以做到这点，但它对任何类型都会一视同仁地转为右值。这就需要一个折衷的办法，对左值实参不处理，对右值实参（*传入后会变为左值*）转换为右值，这就是[std::foward](https://en.cppreference.com/w/cpp/utility/forward)所做的事
  * 如果要表示参数是右值，则需要引入一种区别于左值的符号，这就是右值引用符号（*&&*）。右值引用即只能绑定到右值的引用，但其本身是左值（*引用都是左值*）。它只是为了区别于左值引用符号（*&*）而引入的一种符号标记
  * 在模板中，带右值引用符号（*T&&*）并不表示一定是右值引用（*这种不确定类型的引用称为转发引用*），因为模板参数本身可以带引用符号（*int&*），此时为了使结果合法（*int& && 是不合法的*），就引入了**引用折叠机制**（*int& && 折叠为 int&*）

## 23 [std::move](https://en.cppreference.com/w/cpp/utility/move)和[std::forward](https://en.cppreference.com/w/cpp/utility/forward)只是一种强制类型转换
* [std::move](https://en.cppreference.com/w/cpp/utility/move)不完全符合标准的实现如下
```cpp
template<typename T>
decltype(auto) move(T&& x)
{
    using ReturnType = remove_reference_t<T>&&;
    return static_cast<ReturnType>(x);
}
```
* [std::move](https://en.cppreference.com/w/cpp/utility/move)会保留 cv 限定符
```cpp
void f(int&&)
{
    std::cout << 1;
}

void f(const int&)
{
    std::cout << 2;
}

const int i = 1;
f(std::move(i)); // 2
```
* 这可能导致的一个问题是，传入**const右值却执行拷贝操作**
```cpp
class A {
public:
    explicit A(const std::string x)
    : s(std::move(x)) {} // 转为const std::string&&，调用std::string(const std::string&)
private:
    std::string s;
};
```
* 因此如果**希望移动[std::move](https://en.cppreference.com/w/cpp/utility/move)生成的值**，传给[std::move](https://en.cppreference.com/w/cpp/utility/move)的就不要是 const
```cpp
class A {
public:
    explicit A(std::string x)
    : s(std::move(x)) {} // 转为std::string&&，调用std::string(std::string&&)
private:
    std::string s;
};
```
* C++11 之前的转发很简单
```cpp
void f(int&) { std::cout << 1; }
void f(const int&) { std::cout << 2; }

// 用多个重载转发给对应版本比较繁琐
void g(int& x)
{
    f(x);
}

void g(const int& x)
{
    f(x);
}

// 同样的功能可以用一个模板替代
template<typename T>
void h(T& x)
{
    f(x);
}

int main()
{
    int a = 1;
    const int b = 1;

    g(a); h(a); // 11
    g(b); h(b); // 22
    g(1); // 2
    h(1); // 错误
}
```
* C++11 引入了右值引用，但原有的模板无法转发右值。如果使用[std::move](https://en.cppreference.com/w/cpp/utility/move)则无法转发左值，因此为了方便引入了[std::forward](https://en.cppreference.com/w/cpp/utility/forward)
```cpp
void f(int&) { std::cout << 1; }
void f(const int&) { std::cout << 2; }
void f(int&&) { std::cout << 3; }

// 用多个重载转发给对应版本比较繁琐
void g(int& x)
{
    f(x);
}

void g(const int& x)
{
    f(x);
}

void g(int&& x)
{
    f(std::move(x));
}

// 同样可以用一个模板来替代上述功能
template<typename T>
void h(T&& x)
{
    f(std::forward<T>(x)); // 注意std::forward的模板参数是T
}

int main()
{
    int a = 1;
    const int b = 1;

    g(a); h(a); // 11
    g(b); h(b); // 22
    g(std::move(a)); h(std::move(a)); // 33
    g(1); h(1); // 33
}
```
* 看起来完全可以用[std::forward](https://en.cppreference.com/w/cpp/utility/forward)取代[std::move](https://en.cppreference.com/w/cpp/utility/move)，**但[std::move](https://en.cppreference.com/w/cpp/utility/move)的优势在于清晰简单**
```cpp
h(std::forward<int>(a)); // 3
h(std::move(a)); // 3
```
* 结合可变参数模板，完美转发可以转发任意数量的实参
```cpp
template<typename... Ts>
void f(Ts&&... args)
{
    g(std::forward<Ts>(args)...); // 把任意数量的实参转发给g
}
```

## 24 转发引用与右值引用的区别
* **带右值引用符号不一定就是右值引用**，这种不确定类型的引用称为转发引用
```cpp
template<typename T>
void f(T&&) {} // T&&不一定是右值引用

int a = 1;
f(a); // T推断为int&，T&&是int& &&，折叠为int&，是左值引用
f(1); // T推断为int，T&&是int&&，右值引用
auto&& b = a; // int& b = a，左值引用
auto&& c = 1; // int&& c = 1，右值引用
```
* **转发引用必须严格按 T&& 的形式**涉及类型推断
```cpp
template<typename T>
void f(std::vector<T>&&) {} // 右值引用而非转发引用

std::vector<int> v;
f(v); // 错误

template<typename T>
void g(const T&&) {} // 右值引用而非转发引用

int i = 1;
g(i); // 错误
```
* T&& 在模板中也可能不涉及类型推断
```cpp
template<class T, class Allocator = allocator<T>>
class vector {
public:
    void push_back(T&& x); // 右值引用
    
    template <class... Args>
    void emplace_back(Args&&... args); // 转发引用
    ...
};

std::vector<A> v; // 实例化指定了T

// 对应的实例化为
class vector<A, allocator<A>> {
public:
    void push_back(A&& x); // 不涉及类型推断，右值引用
    
    template <class... Args>
    void emplace_back(Args&&... args); // 转发引用
    ...
};
```
* **auto&& 都是转发引用**，**因为一定涉及类型推断**。完美转发中，如果想在转发前修改要转发的值，可以用 auto&& 存储结果，**修改后再转发**
```cpp
template<typename T>
void f(T x)
{
    auto&& res = doSomething(x);
    doSomethingElse(res);//修改
    set(std::forward<decltype(res)>(res));
}
```
* lambda 中也可以使用完美转发
```cpp
auto f = [](auto&& x) { return g(std::forward<decltype(x)>(x)); };

// 转发任意数量实参
auto f = [](auto&&... args) {
    return g(std::forward<decltype(args)>(args)...);
};
```

## 25 对右值引用使用[std::move](https://en.cppreference.com/w/cpp/utility/move)，对转发引用使用[std::forward](https://en.cppreference.com/w/cpp/utility/forward)
* **右值引用只会绑定到可移动对象上**，因此应该使用[std::move](https://en.cppreference.com/w/cpp/utility/move)。转发引用用右值初始化时才是右值引用，因此应当使用[std::forward](https://en.cppreference.com/w/cpp/utility/forward)
```cpp
class A {
public:
    A(A&& rhs) : s(std::move(rhs.s)), p(std::move(rhs.p)) {}
    
    template<typename T>
    void f(T&& x)
    {
        s = std::forward<T>(x);
    }
private:
    std::string s;
    std::shared_ptr<int> p;
};
```
* 如果希望只有**在移动构造函数保证不抛异常时才能转为右值**，则可以用[std::move_if_noexcept](https://en.cppreference.com/w/cpp/utility/move_if_noexcept)替代[std::move](https://en.cppreference.com/w/cpp/utility/move)
```cpp
class A {
public:
    A() {}
    A(const A&) { std::cout << 1; }
    A(A&&) { std::cout << 2; }
};

class B {
public:
    B() {}
    B(const B&) noexcept { std::cout << 3; }
    B(B&&) noexcept { std::cout << 4; }
};

int main()
{
    A a;
    A a2 = std::move_if_noexcept(a); // 1
    B b;
    B b2 = std::move_if_noexcept(b); // 4
}
```
* 如果返回对象传入时是右值引用或转发引用，在返回时要用[std::move](https://en.cppreference.com/w/cpp/utility/move)或[std::forward](https://en.cppreference.com/w/cpp/utility/forward)转换。**返回类型不需要声明为引用，按值传递即可**
```cpp
A f(A&& a)
{
    doSomething(a);
    return std::move(a);
}


template<typename T>
A g(T&& x)
{
    doSomething(x);
    return std::forward<T>(x);
}
```
* **返回局部变量时，不需要使用[std::move](https://en.cppreference.com/w/cpp/utility/move)来优化**
```cpp
A makeA()
{
    A a;
    return std::move(a); // 画蛇添足
}
```
* **局部变量会直接创建在为返回值分配的内存上，从而避免拷贝**，这是 C++ 标准诞生时就有的[RVO（return value optimization）](https://en.cppreference.com/w/cpp/language/copy_elision)。RVO 的要求十分严谨，它要求局部对象类型与返回值类型相同，且返回的就是局部对象本身，而使用了[std::move](https://en.cppreference.com/w/cpp/utility/move)反而不满足 RVO 的要求。此外 RVO 只是种优化，编译器可以选择不采用，但标准规定，即使编译器不省略拷贝，返回对象也会被作为右值处理，所以[std::move是多余的](https://www.ibm.com/developerworks/community/blogs/5894415f-be62-4bc0-81c5-3956e82276f3/entry/RVO_V_S_std_move?lang=en)
```cpp
A makeA()
{
    return A{};
}

auto x = makeA(); // 只需要调用一次A的默认构造函数
```

## 26 避免重载使用转发引用的函数
* 如果函数参数接受左值引用，则传入右值时执行的仍是拷贝
```cpp
std::vector<std::string> v;

void f(const std::string& s)
{
    v.emplace_back(s);
}

// 传入右值，执行的依然是拷贝
f(std::string("hi"));
f("hi");
```
* **让函数接受转发引用**即可解决此问题
```cpp
std::vector<std::string> v;

template<typename T>
void f(T&& s)
{
    v.emplace_back(std::forward<T>(s));
}

// 现在传入右值时变为移动操作
f(std::string("hi"));
f("hi");
```
* 但如果重载这个转发引用版本的函数，就会导致新的问题
```cpp
std::vector<std::string> v;

template<typename T>
void f(T&& s)
{
    v.emplace_back(std::forward<T>(s));
}

std::string makeString(int n)
{
    return std::string("hi");
}

void f(int n) // 新的重载函数
{
    v.emplace_back(makeString(n));
}

// 之前的调用仍然正常
f(std::string("hi"));
f("hi");
// 对于重载版本的调用也没问题
f(1); // 调用重载版本
// 但对于非int（即使能转换到int）参数就会出现问题
unsigned i = 1;
f(i); // 转发引用是比int更精确的匹配
// 为std::vector<std::string>传入short，用short构造std::string导致错误
```
* 转发引用几乎可以匹配任何类型，因此应该避免对其重载。此外，**如果在构造函数中使用转发引用，会导致拷贝构造函数不能被正确匹配**
```cpp
std::string makeString(int n)
{
    return std::string("hi");
}

class A {
public:
    A() {}
    template<typename T>
    explicit A(T&& n) : s(std::forward<T>(n)) {}

    explicit A(int n) : s(makeString(n)) {}
private:
    std::string s;
};

unsigned i = 1;
A a(i); // 依然调用模板而出错，但还有一个更大的问题
A b("hi"); // OK
A c(b); // 错误：调用的仍是模板，用A初始化std::string出错
```
* 模板构造函数不会阻止合成拷贝和移动构造函数（会阻止合成默认构造函数），上述问题的实际情况如下
```cpp
class A {
public:
    template<typename T>
    explicit A(T&& n) : s(std::forward<T>(n)) {}

    A(const A& rhs) = default;
    A(A&& rhs) = default;
private:
    std::string s;
};

A a("hi"); // OK
A b(a); // 错误：调用的仍是模板，用A初始化std::string出错
// 传入的是A&，T&&比const A&更匹配
const A c("hi");
A d(c); // OK
```
* 上述问题在继承中会变得更为复杂，如果派生类的拷贝和移动操作调用基类的构造函数，同样会匹配到使用了转发引用的模板，从而导致编译错误
```cpp
class A {
public:
    template<typename T>
    explicit A(T&& n) : s(std::forward<T>(n)) {}
private:
    std::string s;
};

class B : public A {
public:
    B(const B& rhs) : A(rhs) {} // 错误：调用基类模板而非拷贝构造函数，const B不能转为std::string
    B(B&& rhs) : A(std::move(rhs)) noexcept {} // 错误：调用基类模板而非移动构造函数，B不能转为std::string
};
```

## 27 重载转发引用的替代方案
* 上述问题的最直接解决方案是，不使用重载。其次是使用 C++98 的做法，不使用转发引用
```cpp
class A {
public:
    template<typename T>
    explicit A(const T& n) : s(n) {}
private:
    std::string s;
};

A a("hi");
A b(a); // OK
```
* 直接按值传递也是一种简单的方式，而且解决了之前的问题
```cpp
std::string makeString(int n)
{
    return std::string("hi");
}

class A {
public:
    explicit A(std::string n) : s(std::move(n)) {}
    explicit A(int n) : s(makeString(n)) {}
private:
    std::string s;
};

unsigned i = 1;
A a(i); // OK，调用int版本的构造函数
```
* 不过上述方法实际上是规避了使用转发引用，下面是几种允许转发引用的重载方法

### 标签分派（tag dispatching）
* 标签分派的思路是，额外引入一个参数来打破转发引用的万能匹配
```cpp
std::vector<std::string> v;

template<typename T>
void g(T&& s, std::false_type)
{
    v.emplace_back(std::forward<T>(s));
}

std::string makeString(int n)
{
    return std::string("hi");
}

void g(int n, std::true_type)
{
    v.emplace_back(makeString(n));
}

template<typename T>
void f(T&& s)
{
    g(std::forward<T>(s), std::is_integral<std::remove_reference_t<T>>());
}

unsigned i = 1;
f(i); // OK：调用int版本
```

### 使用[std::enable_if](https://en.cppreference.com/w/cpp/types/enable_if)在特定条件下禁用模板
* 标签分派用在构造函数上不太方便，这时可以使用[std::enable_if](https://en.cppreference.com/w/cpp/types/enable_if)，它可以强制编译器在满足特定条件时禁用模板
```cpp
class A {
public:
    template<typename T, typename =
        std::enable_if_t<!std::is_same_v<A, std::decay_t<T>>>>
    explicit A(T&& n) {}
private:
    std::string s;
};
```
* 但这只是在参数具有和类相同的类型时禁用模板，派生类调用基类的构造函数时，派生类和基类也是不同类型，不会禁用模板，因此还需要使用[std::is_base_of](https://en.cppreference.com/w/cpp/types/is_base_of)
```cpp
class A {
public:
    template<typename T, typename =
        std::enable_if_t<!std::is_base_of_v<A, std::decay_t<T>>>>
    explicit A(T&& n) {}
private:
    std::string s;
};

class B : public A {
public:
    B(const B& rhs) : A(rhs) {} // OK：不再调用模板
    B(B&& rhs) : A(std::move(rhs)) noexcept {} // OK：不再调用模板
};
```
* 接着在参数为整型时禁用模板，即可解决之前的所有问题
```cpp
std::string makeString(int n)
{
    return std::string("hi");
}

class A {
public:
    template<typename T, typename =
        std::enable_if_t<!std::is_base_of_v<A, typename std::decay_t<T>>
        && !std::is_integral_v<std::remove_reference_t<T>>>>
    explicit A(T&& n) : s(std::forward<T>(n)) {}

    explicit A(int n) : s(makeString(n)) {}
private:
    std::string s;
};

unsigned i = 1;
A a(1); // OK：调用int版本的构造函数
A b("hi"); // OK
A c(b); // OK
```
* 为了更方便调试，可以用[static_assert](https://en.cppreference.com/w/c/error/static_assert)预设错误信息，这个错误信息将在不满足预设条件时出现在诊断信息中
```cpp
std::string makeString(int n)
{
    return std::string("hi");
}

class A {
public:
    template<typename T, typename =
        std::enable_if_t<!std::is_base_of_v<A, typename std::decay_t<T>>
        && !std::is_integral_v<std::remove_reference_t<T>>>>
    explicit A(T&& n) : s(std::forward<T>(n))
    {
        static_assert(std::is_constructible_v<std::string, T>,
            "Parameter n can't be used to construct a std::string");
    }

    explicit A(int n) : s(makeString(n)) {}
private:
    std::string s;
};
```

## 28 引用折叠
* **引用折叠**会出现在四种语境中：模板实例化、auto 类型推断、decltype 类型推断、typedef 或 using 别名声明
* 引用的引用是非法的
```cpp
int a = 1;
int& & b = a; // 错误
```
* 当**左值传给接受转发引用**的模板时，模板参数就会**推断为引用的引用**
```cpp
template<typename T>
void f(T&&);

int i = 1;
f(i); // T为int&，T& &&变成了引用的引用，于是需要引用折叠的机制
```
* 为了使实例化成功，编译器生成引用的引用时，将使用**引用折叠**的机制，规则如下
```cpp
& + & → &
& + && → &
&& + & → &
&& + && → &&
```
* **引用折叠是[std::forward](https://en.cppreference.com/w/cpp/utility/forward)的支持基础**
```cpp
// 不完整的实现
template<typename T>
T&& forward(remove_reference_t<T>& x)
{
    return static_cast<T&&>(x); // 如果传递左值A，T推断为A&，此时需要引用折叠
}

// 传递左值A时相当于
A& && forward(remove_reference_t<A&>& x)
{
    return static_cast<A& &&>(x);
}
// 简化后
A& forward(A& x)
{
    return static_cast<A&>(x);
}

// 传递右值A相当于
A&& forward(remove_reference_t<A>& x)
{
    return static_cast<A&&>(x);
}
// 简化后
A&& forward(A& x)
{
    return static_cast<A&&>(x);
}
```
* auto&& 与使用转发引用的模板原理一样
```cpp
int a = 1;
auto&& b = a; // a是左值，auto被推断为int&，int& &&折叠为int&
```
* decltype 同理，如果推断中出现了引用的引用，就会发生引用折叠
* 如果在 typedef 的创建或求值中出现了引用的引用，就会发生引用折叠
```cpp
template<typename T>
struct A {
    typedef T&& RvalueRef;
};

int a = 1;
A<int&>::RvalueRef b = a; // int& &&折叠为int&，int& b = a
```
* 并且 top-level cv 限定符会被丢弃
```cpp
using A = const int&; // low-level
using B = int&&; // low-level
static_assert(std::is_same_v<volatile A&&, const int&>);
static_assert(std::is_same_v<const B&&, int&&>);
```

## 29 移动不比拷贝快的情况
* 在如下场景中，C++11 的移动语义没有优势
  * **无移动操作**：待移动对象不提供移动操作，移动请求将变为拷贝请求
  * 移动不比拷贝快：待移动对象虽然有移动操作，但不比拷贝操作快
  * 移动不可用：本可以移动时，要求移动操作不能抛异常，但未加上 noexcept 声明
* 除了上述情况，还有一些特殊场景无需使用移动语义，比如之前提到的[RVO](https://en.cppreference.com/w/cpp/language/copy_elision)
* **移动不一定比拷贝代价小得多**。比如[std::array](https://en.cppreference.com/w/cpp/container/array)实际是带 STL 接口的内置数组。不同于其他容器的是，其他容器把元素存放于堆上，自身只持有一个指向堆内存的指针，移动容器时只需要移动指针，在常数时间内即可完成移动

![](./images/5-1.png)

* **而[std::array](https://en.cppreference.com/w/cpp/container/array)自身存储了内容**，没有这样的指针，**移动或拷贝对元素逐个执行**，需要线性时间复杂度，所以移动并不比拷贝快多少

![](./images/5-2.png)

* 另一个移动不一定比拷贝快的例子是[std::string](https://en.cppreference.com/w/cpp/string/basic_string)，一种实现是使用[small string optimization（SSO）](https://blogs.msmvps.com/gdicanio/2016/11/17/the-small-string-optimization/)，在字符串很小时（一般是15字节）存储在自身内部，而不使用堆上分配的内存，因此**对小型字符串的移动并不比拷贝快**

## 30 无法完美转发的类型
* 用相同实参调用原函数和转发函数，如果两者执行不同的操作，则称**完美转发失败**。完美转发失败源于**模板类型推断不符合预期**，会导致这个问题的类型包括：大括号初始化值、作为空指针的0和 NULL、只声明但未定义的整型 static const 数据成员、重载函数的名称和函数模板名称、位域

### 大括号初始化
```cpp
void f(const std::vector<int>& v) {}

template<typename T>
void fwd(T&& x)
{
    f(std::forward<T>(x));
}

f({ 1, 2, 3 }); // OK，{1, 2, 3}隐式转换为std::vector<int>
fwd({ 1, 2, 3 }); // 无法推断T，导致编译错误

// 解决方法是借用auto推断出std::initializer_list类型再转发
auto x = { 1, 2, 3 };
fwd(x); // OK
```

### 作为空指针的0或NULL
* **0和 NULL 作为空指针传递给模板时，会推断为 int** 而非指针类型
```cpp
void f(int*) {}

template<typename T>
void fwd(T&& x)
{
    f(std::forward<T>(x));
}

fwd(NULL); // T推断为int，转发失败
```

### 只声明但未定义的static const整型数据成员
* **类内的 static 成员的声明不是定义**，如果 static 成员声明为 const，则编译器会为这些成员值执行 const propagation，从而不需要为它们保留内存。对整型 static const 成员取址可以通过编译，但会导致链接期的错误。转发引用也是引用，在编译器生成的机器代码中，**引用一般会被当成指针处理**。程序的二进制代码中，**从硬件角度看，指针和引用的本质相同**
```cpp
class A {
public:
    static const int n = 1; // 仅声明
};

void f(int) {}

template<typename T>
void fwd(T&& x)
{
    f(std::forward<T>(x));
}

f(A::n); // OK：等价于f(1)
fwd(A::n); // 错误：fwd形参是转发引用，需要取址，无法链接
```
* 但并非所有编译器的实现都有此要求，上述代码可能可以链接。考虑到移植性，最好还是提供定义
```cpp
// A.h
class A {
public:
    static const int n = 1;
};

// A.cpp
const int A::n;
```

### 重载函数的名称和函数模板名称
* 如果转发的是函数指针，可以直接将函数名作为参数，函数名会转换为函数指针
```cpp
void g(int) {}

void f(void(*pf)(int)) {}

template<typename T>
void fwd(T&& x)
{
    f(std::forward<T>(x));
}

f(g); // OK
fwd(g); // OK
```
* 但如果要转发的函数名对应多个重载函数，则无法转发，因为模板无法从单独的函数名推断出函数类型
```cpp
void g(int) {}
void g(int, int) {}

void f(void(*)(int)) {}

template<typename T>
void fwd(T&& x)
{
    f(std::forward<T>(x));
}

f(g); // OK
fwd(g); // 错误：不知道转发的是哪一个函数指针
```
* 转发函数模板名称也会出现同样的问题，因为函数模板可以看成一批重载函数
```cpp
template<typename T>
void g(T x)
{
    std::cout << x;
}

void f(void(*pf)(int)) { pf(1); }

template<typename T>
void fwd(T&& x)
{
    f(std::forward<T>(x));
}

f(g); // OK
fwd(g<int>); // 错误
```
* 要让转发函数接受重载函数名称或模板名称，只能手动指定需要转发的重载版本或模板实例。不过完美转发本来就是为了接受任何实参类型，而要传入的函数指针类型一般是未知的
```cpp
template<typename T>
void g(T x)
{
    std::cout << x;
}

void f(void(*pf)(int)) { pf(1); }

template<typename T>
void fwd(T&& x)
{
    f(std::forward<T>(x));
}

using PF = void(*)(int);
PF p = g;
fwd(p); // OK
fwd(static_cast<PF>(g)); // OK
```

### 位域
* 转发引用也是引用，实际上需要取址，但**位域不允许直接取址**
```cpp
struct A {
    int a : 1;
    int b : 1;
};

void f(int) {}

template<typename T>
void fwd(T&& x)
{
    f(std::forward<T>(x));
}

A x{};
f(x.a); // OK
fwd(x.a); // 错误
```
* 实际上接受位域实参的函数**也只能收到位域值的拷贝**，因此不需要使用完美转发，换用传值或传 const 引用即可。完美转发中也可以通过强制转换解决此问题，虽然转换的结果是一个临时对象的拷贝而非原有对象，但位域本来就无法做到真正的完美转发
```cpp
fwd(static_cast<int>(x.a)); // OK
```

## 31 捕获的潜在问题
* **值捕获**只保存捕获时的对象状态
```cpp
int x = 1;
auto f = [x] { return x; };
auto g = [x]() mutable { return ++x; };
x = 33; // 不改变lambda中已经捕获的x
std::cout << f() << g(); // 12
```
* **引用捕获**会保持与被捕获对象状态一致
```cpp
int x = 1;
auto f = [&x] { return x; };
x = 42;
std::cout << f(); // 42
```
* 引用捕获时，在**捕获的局部变量析构后调用 lambda**，将出现空悬引用
```cpp
std::function<void()> f()
{
    int x = 0;
    return [&]() { std::cout << x; };
}

f()(); // x已被析构，值未定义
```
* 值捕获的问题比较隐蔽
```cpp
struct A {
    auto f()
    {
        return [=] { std::cout << i; };
    };
    int i = 1;
};

A a;
a.f()(); // 1
```
* 上述代码看似没有问题，但**如果把=去掉，或者显式捕获**，就会出现无法捕获的错误。原因在于数**据成员位于 lambda 作用域外，不能被捕获**
```cpp
struct A {
    auto f()
    {
        return [i] { std::cout << i; }; // 错误：无法捕获i
    };
    int i = 1;
};
```
* 之前不出错的原因在于，**= 捕获的是 this 指针**，实际效果相当于引用捕获
```cpp
struct A {
    auto f()
    {
        return [this] { std::cout << i; }; // i被视为this->i
    };
    int i = 1;
};

A a;
auto x = a.f(); // 内部的i与a.i绑定
a.i = 2;
x(); // 2
```
* 因此值捕获也可以出现空悬引用
```cpp
struct A {
    auto f()
    {
        return [this] { std::cout << i; };
    };
    int i = 1;
};

auto g()
{
    auto p = std::make_unique<A>();
    return p->f();
} // p被析构

g()(); // A对象p已被析构，lambda捕获的this失效，i值未定义
```
* this 指针会被析构，**可以值捕获 \*this 对象，这才是真正的值捕获**
```cpp
struct A {
    auto f()
    {
        return [*this] { std::cout << i; };
    };
    int i = 1;
};

A a;
auto x = a.f(); // 只保存此刻的a.i
a.i = 2;
x(); // 1

// 现在lambda捕获的是对象的一个拷贝，不会被原对象的析构影响
auto g()
{
    auto p = std::make_unique<A>();
    return p->f();
}

g()(); // 1
```
* **更细致的解决方法是，捕获数据成员的一个拷贝**
```cpp
struct A {
    auto f()
    {
        int j = i;//数据成员
        return [j] { std::cout << j; };//拷贝
    };
    int i = 1;
};

auto g()
{
    auto p = std::make_unique<A>();
    return p->f();
}

g()(); // 1
```
* C++14 中提供了广义 lambda 捕获（generalized lambda-capture），也叫**初始化捕获（init-capture）**，可以直接在捕获列表中拷贝
```cpp
struct A {
    auto f()
    {
        return [i = i] { std::cout << i; };
    };
    int i = 1;
};

auto g()
{
    auto p = std::make_unique<A>();
    return p->f();
}
// 或者直接写为
auto g = [p = std::make_unique<A>()]{ return p->f(); };

g()(); // 1
```
* 值捕获似乎表明闭包是独立的，与闭包外的数据无关，但并非如此，比如 **lambda 可以直接使用 static 变量（但无法捕获）**
```cpp
static int i = 1;
auto f = [] { std::cout << i; }; // OK：可以直接使用i
auto g = [i] {}; // 错误：无法捕获static变量
```

## 32 用初始化捕获将对象移入闭包
* **move-only 类型对象**不支持拷贝，只能采用**引用捕获**
```cpp
auto p = std::make_unique<int>(42);
auto f = [&p]() {std::cout << *p; };
```
* **初始化捕获则支持把 move-only 类型对象移动**进 lambda 中
```cpp
auto p = std::make_unique<int>(42);
auto f = [p = std::move(p)]() {std::cout << *p; };
assert(p == nullptr);
```
* 还可以**直接在捕获列表中初始化** move-only 类型对象
```cpp
auto f = [p = std::make_unique<int>(42)]() {std::cout << *p; };
```
* 如果不使用 lambda，C++11 中可以封装一个类来模拟 lambda 的初始化捕获
```cpp
class A {
public:
    A(std::unique_ptr<int>&& q) : p(std::move(q)) {}
    void operator()() const { std::cout << *p; }
private:
    std::unique_ptr<int> p;
};

auto f = A(std::make_unique<int>(42));
```
* 如果要在 C++11 中使用 lambda 并模拟初始化捕获，需要借助[std::bind](https://en.cppreference.com/w/cpp/utility/functional/bind)
```cpp
auto f = std::bind(
    [](const std::unique_ptr<int>& p) { std::cout << *p; },
    std::make_unique<int>(42));
```
* bind 对象（[std::bind](https://en.cppreference.com/w/cpp/utility/functional/bind)返回的对象）中包含**传递给[std::bind](https://en.cppreference.com/w/cpp/utility/functional/bind)的所有实参的拷贝**，对于每个**左值实参**，bind 对象中的对应部分由**拷贝构造**，对于**右值实参则是移动构造**。上例中第二个实参是右值，采用的是移动构造，这就是把右值移动进 bind 对象的手法
```cpp
std::vector<int> v; // 要移动到闭包的对象
// C++14：初始化捕获
auto f = [v = std::move(v)] {};
// C++11：模拟初始化捕获
auto g = std::bind([] (const std::vector<int>& v) {}, std::move(v));
```
* 默认情况下，**lambda 生成的闭包类的 operator() 默认为 const**，闭包中的所有成员变量也会为 const，因此上述模拟中使用的 lambda 形参都为 const
```cpp
auto f = [](auto x, auto y) { return x < y; };

// 上述lambda相当于生成如下匿名类
struct X {
    template<typename T, typename U>
    auto operator() (T x, U y) const { return x < y; }
};
```
* 如果是**可变 lambda**，则闭包中的 operator() 就不会为 const，因此模拟可变 lambda 则模拟中使用的 lambda 形参就不必声明为 const
```cpp
std::vector<int> v;
// C++14：初始化捕获
auto f = [v = std::move(v)] () mutable {};
// C++11：模拟可变lambda的初始化捕获
auto g = std::bind([] (std::vector<int>& v) {}, std::move(v));
```
* 因为bind对象的生命期和闭包相同，所以对bind对象中的对象和闭包中的对象可以用同样的手法处理

## 33 用[decltype](https://en.cppreference.com/w/cpp/language/decltype)获取auto&&参数类型以[std::forward](https://en.cppreference.com/w/cpp/utility/forward)
* 对于泛型lambda
```cpp
auto f = [](auto x) { return g(x); };
```
* 同样可以使用完美转发
```cpp
// 传入参数是auto，类型未知，std::forward的模板参数应该是什么？
auto f = [](auto&& x) { return g(std::forward<???>(x)); };
```
* 此时可以用 decltype 判断传入的实参是左值还是右值
  * 如果传递给auto&&的**实参是左值**，则x为左值引用类型，**decltype(x)**为左值引用类型
  * 如果传递给auto&&的**实参是右值**，则x为右值引用类型，decltype(x)为右值引用类型
```cpp
auto f = [](auto&& x) { return g(std::forward<decltype(x)>(x)); };
```
* 转发任意数量的实参
```cpp
auto f = [](auto&&... args) {
    return g(std::forward<decltype(args)>(args)...);
};
```

## 34 用lambda替代[std::bind](https://en.cppreference.com/w/cpp/utility/functional/bind)
* lambda 代码更简洁
```cpp
auto f = [l, r] (const auto& x) { return l <= x && x <= r; };

// 用std::bind实现相同效果
using namespace std::placeholders;
// C++14
auto f = std::bind(
    std::logical_and<>(),
    std::bind(std::less_equal<>(), l, _1),
    std::bind(std::less_equal<>(), _1, r));
// C++11
auto f = std::bind(
    std::logical_and<bool>(),
    std::bind(std::less_equal<int>(), l, _1),
    std::bind(std::less_equal<int>(), _1, r));
```
* lambda 可以指定值捕获和引用捕获，而[std::bind](https://en.cppreference.com/w/cpp/utility/functional/bind)总会**按值拷贝实参**，要按**引用传递**则需要使用[std::ref](https://en.cppreference.com/w/cpp/utility/functional/ref)
```cpp
void f(const A&);

using namespace std::placeholders;
A a;
auto g = std::bind(f, std::ref(a), _1);
```
* lambda 中可以正常使用**重载函数**，而[std::bind](https://en.cppreference.com/w/cpp/utility/functional/bind)**无法区分重载版本**，为此必须指定对应的函数指针类型。**lambda 闭包类的 operator()** 采用的是能被**编译器内联**的常规的函数调用，而[std::bind](https://en.cppreference.com/w/cpp/utility/functional/bind)采用的是一般不会被内联的函数指针调用，**这意味着 lambda 比[std::bind](https://en.cppreference.com/w/cpp/utility/functional/bind)运行得更快**
```cpp
void f(int) {}
void f(double) {}

auto g = [] { f(1); }; // OK
auto g = std::bind(f, 1); // 错误 无法区分重载版本
auto g = std::bind(static_cast<void(*)(int)>(f), 1); // OK
```
* **实参绑定的是[std::bind](https://en.cppreference.com/w/cpp/utility/functional/bind)返回的对象**，而非内部的函数
```cpp
void f(std::chrono::steady_clock::time_point t, int i)
{
    std::this_thread::sleep_until(t);
    std::cout << i;
}

auto g = [](int i)
{
    f(std::chrono::steady_clock::now() + std::chrono::seconds(3), i);
};

g(1); // 3秒后打印1
// 用std::bind实现相同效果，但存在问题
auto h = std::bind(
    f,
    std::chrono::steady_clock::now() + std::chrono::seconds(3),
    std::placeholders::_1);

h(1); // 3秒后打印1，但3秒指的是调用std::bind后的3秒，而非调用f后的3秒
```
* 上述代码的问题在于，计算时间的表达式作为实参被传递给[std::bind](https://en.cppreference.com/w/cpp/utility/functional/bind)，因此计算发生在调用[std::bind](https://en.cppreference.com/w/cpp/utility/functional/bind)的时刻，而非调用其绑定的函数的时刻。解决办法是延迟到调用绑定的函数时再计算表达式值，这可以通过在内部**再嵌套一个[std::bind](https://en.cppreference.com/w/cpp/utility/functional/bind)来实现**
```cpp
auto h = std::bind(
    f,
    std::bind(std::plus<>(), std::chrono::steady_clock::now(), std::chrono::seconds(3)),
    std::placeholders::_1);
```
* C++14 中没有需要使用[std::bind](https://en.cppreference.com/w/cpp/utility/functional/bind)的理由，C++11 由于特性受限存在两个使用场景，一是模拟 C++11 缺少的移动捕获，二是函数对象的 operator() 是模板时，若要将此函数对象作为参数使用，用[std::bind](https://en.cppreference.com/w/cpp/utility/functional/bind)绑定才能接受任意类型实参
```cpp
struct X {
    template<typename T>
    void operator()(const T&) const;
};

X x;
auto f = std::bind(x, _1); // f可以接受任意参数类型
```
* C++11 的 lambda 无法达成上述效果，但 C++14 可以
```cpp
X a;
auto f = [a](const auto& x) { a(x); };
```

### [std::bind](https://en.cppreference.com/w/cpp/utility/functional/bind)
* 占位符
```cpp
void f(int a, int b, int c) { std::cout << a << b << c; }

using namespace std::placeholders;
auto x = std::bind(f, _2, _1, 3);
// _n表示f的第n个参数
// x(a, b, c)相当于f(b, a, 3);

x(4, 5, 6); // 543
```
* 传引用
```cpp
void f(int& n) { ++n; }

int n = 1;

auto x = std::bind(f, n);
x(); // n == 1
auto y = std::bind(f, std::ref(n));
y(); // n == 2
```
* 传递占位符给其他函数
```cpp
void f(int a, int b) { std::cout << a << b; }
int g(int n) { return n + 1; }

using namespace std::placeholders;
auto x = std::bind(f, _1, std::bind(g, _1));

x(1); // 12
```
* 绑定成员函数指针
```cpp
struct A {
    void f(int n) { std::cout << n; }
};

A a;
auto x = std::bind(&A::f, &a, std::placeholders::_1); // &a也可以换成a
x(42); // 42
```
* 绑定数据成员
```cpp
struct A {
    int i = 1;
};

auto x = std::bind(&A::i, std::placeholders::_1);
A a;
std::cout << x(a); // 1
std::cout << x(&a); // 1
std::cout << x(std::make_unique<A>(a)); // 1
std::cout << x(std::make_shared<A>(a)); // 1
```
* 生成随机数
```cpp
// 打印20个数字
std::default_random_engine e;
std::uniform_int_distribution<> d(0, 9);
auto x = std::bind(d, e);
for (int i = 0; i < 20; ++i) std::cout << x();
```

### [std::function](https://en.cppreference.com/w/cpp/utility/functional/function)
* **存储函数**
```cpp
void f(int i) { std::cout << i; }
std::function<void(int)> g = f;
g(1); // 1
```
* **存储函数对象**
```cpp
struct X {
    void operator()(int n) const
    {
        std::cout << n;
    }
};

std::function<void(int)> f = X();
f(1); // 1
```
* **存储 lambda**
```cpp
std::function<void(int)> f = [] (int i) { std::cout << i; };
f(1); // 1
```
* **存储 bind 对象**
```cpp
void f(int i) { std::cout << i; }
std::function<void(int)> g = std::bind(f, std::placeholders::_1);
g(1); // 1
```
* 存储绑定**成员函数指针的 bind 对象**
```cpp
struct A {
    void f(int n) { std::cout << n; }
};

A a;
std::function<void(int)> g = std::bind(&A::f, &a, _1);
g(1);
```
* **存储成员函数**
```cpp
struct A {
    void f(int n) { std::cout << n; }
};

std::function<void(A&, int)> g = &A::f;
A a;
g(a, 1); // 1
```
* **存储 const 成员函数**
```cpp
struct A {
    void f(int n) const { std::cout << n; }
};

std::function<void(const A&, int)> g = &A::f;

A a;
g(a, 1); // 1
const A b;
g(b, 1); // 1
```
* **存储数据成员**
```cpp
struct A {
    int i = 1;
};

std::function<int(const A&)> g = &A::i;

A a;
std::cout << g(a); // 1
```

## 35 用[std::async](https://en.cppreference.com/w/cpp/thread/async)替代[std::thread](https://en.cppreference.com/w/cpp/thread/thread)
* **异步运行函数**的一种选择是，创建一个[std::thread](https://en.cppreference.com/w/cpp/thread/thread)来运行
```cpp
int f();
std::thread t(f);
```
* 另一种方法是使用[std::async](https://en.cppreference.com/w/cpp/thread/async)，它返回一个持有**计算结果**的[std::future](https://en.cppreference.com/w/cpp/thread/future)
```cpp
int f();
std::future<int> ft = std::async(f);
```
* **如果函数有返回值**，[std::thread](https://en.cppreference.com/w/cpp/thread/thread)无法直接获取该值，而[std::async](https://en.cppreference.com/w/cpp/thread/async)返回的[std::future](https://en.cppreference.com/w/cpp/thread/future)提供了[get](https://en.cppreference.com/w/cpp/thread/future/get)来获取该值。
* 如果函数抛出异常，[get](https://en.cppreference.com/w/cpp/thread/future/get)能访问异常，而[std::thread](https://en.cppreference.com/w/cpp/thread/thread)会调用[std::terminate](https://en.cppreference.com/w/cpp/error/terminate)终止程序
```cpp
int f() { return 1; }
auto ft = std::async(f);
int res = ft.get();
```
* 在并发的C++软件中，线程有三种含义：
  * hardware thread 是实际执行计算的线程，计算机体系结构中会为每个CPU内核提供一个或多个**硬件线程**
  * software thread（OS thread或system thread）是操作系统实现跨进程管理，并执行**硬件线程调度线程**
  * [std::thread](https://en.cppreference.com/w/cpp/thread/thread)是 C++ 进程中的对象，用作**底层 OS thread 的 handle**
* OS thread 是一种有限资源，如果试图创建的线程超出系统所能提供的数量，就会抛出[std::system_error](https://en.cppreference.com/w/cpp/error/system_error)异常。这在任何时候都是确定的，即使要运行的函数不能抛异常
```cpp
int f() noexcept;
std::thread t(f); // 若无线程可用，仍会抛出异常
```
* 解决这个问题的一个方法是在当前线程中运行函数，但这会导致负载不均衡，而且如果当前线程是一个 GUI 线程，将导致无法响应。另一个方法是等待已存在的软件线程完成工作后再新建[std::thread](https://en.cppreference.com/w/cpp/thread/thread)，但一种可能的问题是，已存在的软件线程在等待函数执行某个动作
* 即使没有用完线程也可能发生 oversubscription 的问题，即准备运行（非阻塞）的 OS thread 数量超过了 hardware thread，此时线程调度器会为 OS thread 在 hardware thread 上分配 CPU 时间片。当一个线程的时间片用完，另一个线程启动时，就会发生语境切换。这种语境切换会增加系统的线程管理开销，尤其是调度器切换到不同的 CPU core 上的硬件线程时会产生巨大开销。此时，OS thread 通常不会命中 CPU cache（即它们几乎不含有对该软件线程有用的数据和指令），CPU core 运行的新软件线程还会污染 cache 上为旧线程准备的数据，旧线程曾在该 CPU core 上运行过，并很可能再次被调度到此处运行
* 避免 oversubscription 很困难，因为 OS thread 和 hardware thread 的最佳比例取决于软件线程变为可运行状态的频率，而这是会动态变化的，比如一个程序从 I/O 密集型转换计算密集型。软件线程和硬件线程的最佳比例也依赖于语境切换的成本和使用 CPU cache 的命中率，而硬件线程的数量和 CPU cache 的细节（如大小、速度）又依赖于计算机体系结构，因此即使在一个平台上避免了 oversubscription 也不能保证在另一个平台上同样有效
* **使用[std::async](https://en.cppreference.com/w/cpp/thread/async)则可以把 oversubscription 的问题丢给库作者解决**
```cpp
auto ft = std::async(f); // 由标准库的实现者负责线程管理
```
* 这个调用把线程管理的责任转交给了标准库实现。如果申请的软件线程多于系统可提供的，系统不保证会创建一个新的软件线程。相反，它允许调度器把函数运行在对返回的[std::future](https://en.cppreference.com/w/cpp/thread/future)调用[get](https://en.cppreference.com/w/cpp/thread/future/get)或[wait](https://en.cppreference.com/w/cpp/thread/future/wait)的线程中
* 即使使用[std::async](https://en.cppreference.com/w/cpp/thread/async)，GUI 线程的响应性也仍然存在问题，因为调度器无法得知哪个线程迫切需要响应。这种情况下，**可以将[std::async](https://en.cppreference.com/w/cpp/thread/async)的启动策略设定为[std::launch::async](https://en.cppreference.com/w/cpp/thread/launch)，这样可以保证函数会在调用[get](https://en.cppreference.com/w/cpp/thread/future/get)或[wait](https://en.cppreference.com/w/cpp/thread/future/wait)的线程中运行**
```cpp
auto ft = std::async(std::launch::async, f);//函数在get /wait线程中执行
```
* [std::async](https://en.cppreference.com/w/cpp/thread/async)分担了手动管理线程的负担，并提供了检查异步执行函数的结果的方式，但仍有几种不常见的情况**需要使用[std::thread](https://en.cppreference.com/w/cpp/thread/thread)：**
  * 需要访问**底层线程 API**：并发 API 通常基于系统的底层 API（pthread、Windows线程库）实现，通过[std::thread::native_handle](https://en.cppreference.com/w/cpp/thread/thread/native_handle)即可获取底层线程 handle
  * 需要**为应用优化线程**用法：比如开发一个服务器软件，运行时的 profile 已知并作为唯一的主进程部署在硬件特性固定的机器上
  * **实现**标准库未提供的线程技术，比如**线程池**

## 36 用[std::launch::async](https://en.cppreference.com/w/cpp/thread/launch)指定异步求值
* [std::async](https://en.cppreference.com/w/cpp/thread/async)有两种标准启动策略：
  * [std::launch::async](https://en.cppreference.com/w/cpp/thread/launch)：函数必须**异步运行**，即运行在不同的线程上
  * [std::launch::deferred](https://en.cppreference.com/w/cpp/thread/launch)：函数只在对返回的[std::future](https://en.cppreference.com/w/cpp/thread/future)调用[get](https://en.cppreference.com/w/cpp/thread/future/get)或[wait](https://en.cppreference.com/w/cpp/thread/future/wait)时运行。即**执行会推迟**，调用[get](https://en.cppreference.com/w/cpp/thread/future/get)或[wait](https://en.cppreference.com/w/cpp/thread/future/wait)时函数会**同步运行，调用方会阻塞至函数运行结束**
* [std::async](https://en.cppreference.com/w/cpp/thread/async)的默认启动策略不是二者之一，**而是对二者求或的结果**
```cpp
auto ft1 = std::async(f); // 意义同下
auto ft2 = std::async(std::launch::async | std::launch::deferred, f);
```
* 默认启动策略允许异步或同步运行函数，这种灵活性使得[std::async](https://en.cppreference.com/w/cpp/thread/async)和标准库的线程管理组件能负责线程的创建和销毁、负载均衡以及避免 oversubscription
* 但默认启动策略存在一些潜在问题，比如**给定线程 t** 执行如下语句
```cpp
auto ft = std::async(f);
```
* **潜在的问题有：**
  * 无法预知 f 和 t **是否会并发运行**，因为 f 可能被调度为推迟运行
  * **无法预知运行 f 的线程是否不同于对 ft 调用[get](https://en.cppreference.com/w/cpp/thread/future/get)或[wait](https://en.cppreference.com/w/cpp/thread/future/wait)的线程**，如果调用[get](https://en.cppreference.com/w/cpp/thread/future/get)或[wait](https://en.cppreference.com/w/cpp/thread/future/wait)的线程是 t，就说明无法预知 f 是否会运行在与 t 不同的某线程上
  * 甚至很可能**无法预知 f 是否会运行**，因为无法保证在程序的每条路径上，ft 的[get](https://en.cppreference.com/w/cpp/thread/future/get)或[wait](https://en.cppreference.com/w/cpp/thread/future/wait)会被调用
* 默认启动策略在调度上的灵活性会在使用[thread_local](https://en.cppreference.com/w/cpp/keyword/thread_local)变量时导致混淆，这意味着如果 f 读写此 thread-local
storage（TLS）时，**无法预知哪个线程的局部变量将被访问**
```cpp
auto ft = std::async(f); // f的TLS可能和一个独立线程相关，但也可能与对ft调用get或wait的线程相关
```
* 它也会影响使用 **timeout 的 wait-based 循环**，因为对返回的[std::future](https://en.cppreference.com/w/cpp/thread/future)调用[wait_for](https://en.cppreference.com/w/cpp/thread/future/wait_for)或[wait_until](https://en.cppreference.com/w/cpp/thread/future/wait_until)会产生[std::future_status::deferred](https://en.cppreference.com/w/cpp/thread/future_status)值。这意味着以下循环看似最终会终止，但实际可能永远运行
```cpp
using namespace std::literals;

void f()
{
    std::this_thread::sleep_for(1s);
}

auto ft = std::async(f);
while (ft.wait_for(100ms) != std::future_status::ready)//deferred 延期
{ // 循环至f运行完成，但这可能永远不会发生
    …
}
```
* 如果选用了[std::launch::async](https://en.cppreference.com/w/cpp/thread/launch)启动策略，f 和调用 [std::async](https://en.cppreference.com/w/cpp/thread/async)的线程并发执行，则没有问题。但如果 f 被推迟执行，则 [wait_for](https://en.cppreference.com/w/cpp/thread/future/wait_for)总会返回[std::future_status::deferred](https://en.cppreference.com/w/cpp/thread/future_status)，于是循环永远不会终止
* 这类 bug 在开发和单元测试时很容易被忽略，只有在运行负载很重时才会被发现。解决方法很简单，检查返回的[std::future](https://en.cppreference.com/w/cpp/thread/future)，确定任务是否被推迟。但没有直接检查是否推迟的方法，替代的手法是，先调用一个 timeout-based 函数，比如 [wait_for](https://en.cppreference.com/w/cpp/thread/future/wait_for)，这并不表示想等待任何事，而只是为了查看返回值是否为[std::future_status::deferred](https://en.cppreference.com/w/cpp/thread/future_status)
```cpp
auto ft = std::async(f);
if (ft.wait_for(0s) == std::future_status::deferred) // 任务被推迟
{
    … // 使用ft的wait或get异步调用f
}
else // 任务未被推迟
{
    while (ft.wait_for(100ms) != std::future_status::ready)
    {
        … // 任务未被推迟也未就绪，则做并发工作直至结束
    }
    … // ft准备就绪
}
```
* 综上，[std::async](https://en.cppreference.com/w/cpp/thread/async)使用**默认启动策略创建**要满足以下所有条件：
  * 任务不需要与对返回值调用[get](https://en.cppreference.com/w/cpp/thread/future/get)或[wait](https://en.cppreference.com/w/cpp/thread/future/wait)的线程并发执行
  * 读写哪个线程的 thread_local 变量没有影响
  * 要么保证对**返回值调用**[get](https://en.cppreference.com/w/cpp/thread/future/get)或[wait](https://en.cppreference.com/w/cpp/thread/future/wait)，要么接受任务可能永远不执行
  * 使用[wait_for](https://en.cppreference.com/w/cpp/thread/future/wait_for)或[wait_until](https://en.cppreference.com/w/cpp/thread/future/wait_until)的代码要考虑**任务被推迟的可能**
* 只要一点不满足，就可能意味着想确保异步执行任务，这只需要指定启动策略为[std::launch::async](https://en.cppreference.com/w/cpp/thread/launch)
```cpp
auto ft = std::async(std::launch::async, f); // 异步执行f
```
* 默认使用[std::launch::async](https://en.cppreference.com/w/cpp/thread/launch)启动策略的[std::async](https://en.cppreference.com/w/cpp/thread/async)将会是一个很方便的工具，实现如下
```cpp
template<typename F, typename... Ts>
inline
auto // std::future<std::invoke_result_t<F, Ts...>>
reallyAsync(F&& f, Ts&&... args)
{
    return std::async(std::launch::async,
        std::forward<F>(f),
        std::forward<Ts>(args)...);
}
```
* 这个函数的用法和[std::async](https://en.cppreference.com/w/cpp/thread/async)一样
```cpp
auto ft = reallyAsync(f); // 异步运行f，如果std::async抛出异常则reallyAsync也抛出异常
```

## 37 RAII线程管理
* 每个[std::thread](https://en.cppreference.com/w/cpp/thread/thread)对象都处于可合并（joinable）或不可合并（unjoinable）的状态。
* 一个**可合并**的[std::thread](https://en.cppreference.com/w/cpp/thread/thread)对应于一个底层**异步运行的线程**，若底层线程处于**阻塞、等待调度或已运行结束**的状态，则此[std::thread](https://en.cppreference.com/w/cpp/thread/thread)可合并，否则不可合并。
* 不可合并的[std::thread](https://en.cppreference.com/w/cpp/thread/thread)包括：
  * 默认构造的[std::thread](https://en.cppreference.com/w/cpp/thread/thread)：此时**没有要运行的函数**，因此没有对应的底层运行线程
  * 已移动的[std::thread](https://en.cppreference.com/w/cpp/thread/thread)：**移动操作**导致底层线程被转用于另一个[std::thread](https://en.cppreference.com/w/cpp/thread/thread)
  * 已[join](https://en.cppreference.com/w/cpp/thread/thread/join)或已[join](https://en.cppreference.com/w/cpp/thread/thread/join)的[std::thread](https://en.cppreference.com/w/cpp/thread/thread)
* 如果**可合并的[std::thread](https://en.cppreference.com/w/cpp/thread/thread)对象的析构函数被调用，则程序的执行将终止**
```cpp
void f() {}

void g()
{
    std::thread t(f); // t.joinable() == true
}

int main()
{
    g(); // g运行结束时析构t，导致整个程序终止
    ...
}
```
* **析构可合并的[std::thread](https://en.cppreference.com/w/cpp/thread/thread)时，隐式[join](https://en.cppreference.com/w/cpp/thread/thread/join)或隐式[detach](https://en.cppreference.com/w/cpp/thread/thread/detach)的带来问题更大**。隐式[join](https://en.cppreference.com/w/cpp/thread/thread/join)导致 g 运行结束时仍要保持等待 f 运行结束，这就会导致性能问题，且调试时难以追踪原因。隐式[detach](https://en.cppreference.com/w/cpp/thread/thread/detach)导致的调试问题更为致命
```cpp
void f(int&) {} //引用

void g()
{
    int i = 1;
    std::thread t(f, i);
} // 如果隐式detach，局部变量i被销毁，但f仍在使用局部变量的引用
```

* 完美销毁一个可合并的[std::thread](https://en.cppreference.com/w/cpp/thread/thread)十分困难，因此规定销毁将导致终止程序。**要避免程序终止，只要让可合并的线程在销毁时变为不可合并状态**即可，使用RAII手法就能实现这点
```cpp
class A {
public:
    enum class DtorAction { join, detach };
    A(std::thread&& t, DtorAction a) : action(a), t(std::move(t)) {}
    ~A()
    {
        if (t.joinable())
        {
            if (action == DtorAction::join) t.join();
            else t.detach();
        }
    }
    A(A&&) = default;
    A& operator=(A&&) = default;
    std::thread& get() { return t; }
private:
    DtorAction action;
    std::thread t;
};
void f() {}

void g()
{
    A t(std::thread(f), A::DtorAction::join); // 析构前使用join
}

int main()
{
    g(); // g运行结束时将内部的std::thread置为join，变为不可合并状态
    // 析构不可合并的std::thread不会导致程序终止
    // 这种手法带来了隐式join和隐式detach的问题，但可以调试
    ...
}
```

## 38 [std::future](https://en.cppreference.com/w/cpp/thread/future)的析构行为
* 可合并的[std::thread](https://en.cppreference.com/w/cpp/thread/thread)对应一个底层系统线程，采用[std::launch::async](https://en.cppreference.com/w/cpp/thread/launch)启动策略的[std::async](https://en.cppreference.com/w/cpp/thread/async)返回的[std::future](https://en.cppreference.com/w/cpp/thread/future)和系统线程也有类似的关系，因此**可以认为[std::thread](https://en.cppreference.com/w/cpp/thread/thread)和[std::future](https://en.cppreference.com/w/cpp/thread/future)相当于系统线程的handle**
* 销毁[std::future](https://en.cppreference.com/w/cpp/thread/future)有时表现为隐式 join，有时表现为隐式 detach，有时表现为既不隐式 join 也不隐式 detach，但它不会导致程序终止。这种不同表现行为是值得需要思考的。想象[std::future](https://en.cppreference.com/w/cpp/thread/future)处于信道的一端，callee 将[std::promise](https://en.cppreference.com/w/cpp/thread/promise)对象传给 caller，caller 用一个[std::future](https://en.cppreference.com/w/cpp/thread/future)来读取结果
```cpp
std::promise<int> ps;
std::future<int> ft = ps.get_future();
```

![](./images/7-1.png)

* callee 的结果存储在哪？caller 调用[get](https://en.cppreference.com/w/cpp/thread/future/get)之前，callee 可能已经执行完毕，因此结果不可能存储在 callee 的[std::promise](https://en.cppreference.com/w/cpp/thread/promise)对象中。但结果也不可能存储在caller的[std::future](https://en.cppreference.com/w/cpp/thread/future)中，因为[std::future](https://en.cppreference.com/w/cpp/thread/future)可以用来创建[std::shared_future](https://en.cppreference.com/w/cpp/thread/shared_future)
```cpp
std::shared_future<int> sf(std::move(ft));
// 更简洁的写法是用std::future::share返回std::shared_future
auto sf = ft.share();
```
* 而[std::shared_future](https://en.cppreference.com/w/cpp/thread/shared_future)在原始的[std::future](https://en.cppreference.com/w/cpp/thread/future)析构后仍然可以复制
```cpp
auto sf2 = sf;
auto sf3 = sf;
```
* 因此结果只能存储在外部某个位置，这个位置称为 shared state

![](./images/7-2.png)

* shared state 通常用堆上的对象表示，但类型、接口和具体实现由标准库作者决定。shared state 决定了[std::future](https://en.cppreference.com/w/cpp/thread/future)的析构函数行为：
  * 采用[std::launch::async](https://en.cppreference.com/w/cpp/thread/launch)启动策略的[std::async](https://en.cppreference.com/w/cpp/thread/async)返回的[std::future](https://en.cppreference.com/w/cpp/thread/future)中，最后一个引用 shared state的，析构函数会保持阻塞至任务执行完成。本质上，这样一个[std::future](https://en.cppreference.com/w/cpp/thread/future)的析构函数是对**异步运行**的底层线程执行了一次**隐式 join**
  * 其他所有[std::future](https://en.cppreference.com/w/cpp/thread/future)的析构函数**只是简单地析构对象**。对底层异步运行的任务，这相当于对线程执行了一次隐式 detach。对于被推迟的任务来说，如果这是最后一个[std::future](https://en.cppreference.com/w/cpp/thread/future)，就意味着被推迟的任务将不会再运行
* 这些规则看似复杂，但本质就是一个正常行为和一个特殊行为。正常行为是析构函数会销毁[std::future](https://en.cppreference.com/w/cpp/thread/future)对象，它不 join 或 detach 任何东西，也没有运行任何东西，它只是销毁[std::future](https://en.cppreference.com/w/cpp/thread/future)的成员变量。不过实际上它确实多做了一件事，就是减少了一次 shared state 中的引用计数，shared state 由 caller 的 [std::future](https://en.cppreference.com/w/cpp/thread/future)和 callee 的[std::promise](https://en.cppreference.com/w/cpp/thread/promise)共同操控。引用计数让库得知何时能销毁 shared state
* [std::future](https://en.cppreference.com/w/cpp/thread/future)的析构函数只在满足以下所有条件时发生特殊行为（**阻塞**至异步运行的任务结束）：
  * [std::future](https://en.cppreference.com/w/cpp/thread/future)引用的 shared state 由调用[std::async](https://en.cppreference.com/w/cpp/thread/async)创建
  * 任务的启动策略是[std::launch::async](https://en.cppreference.com/w/cpp/thread/launch)，这可以是运行时系统选择的或显式指定的
  * 这个[std::future](https://en.cppreference.com/w/cpp/thread/future)是最后一个引用 shared state 的。对于[std::shared_future](https://en.cppreference.com/w/cpp/thread/shared_future)，如果其他[std::shared_future](https://en.cppreference.com/w/cpp/thread/shared_future)和要被销毁的[std::shared_future](https://en.cppreference.com/w/cpp/thread/shared_future)引用同一个 shared state，则被销毁的[std::shared_future](https://en.cppreference.com/w/cpp/thread/shared_future)遵循正常行为（即简单地销毁数据成员）
* 阻塞至异步运行的任务结束的特殊行为，在效果上相当于对运行着[std::async](https://en.cppreference.com/w/cpp/thread/async)创建的任务的线程执行了一次隐式 join。特别制定这个规则的原因是，标准委员会想避免隐式 detach 相关的问题，但又不想对可合并的线程一样直接让程序终止，于是妥协的结果就是执行一次隐式 join
* [std::future](https://en.cppreference.com/w/cpp/thread/future)没有提供 API 来判断 shared state 是否产生于[std::async](https://en.cppreference.com/w/cpp/thread/async)的调用，即无法得知析构时是否会阻塞至异步任务执行结束，因此含有[std::future](https://en.cppreference.com/w/cpp/thread/future)的类型都可能在析构函数中阻塞
```cpp
std::vector<std::future<void>> v; // 该容器可能在析构函数中阻塞

class A { // 该类型对象可能会在析构函数中阻塞
    std::shared_future<int> ft;
};
```
* 只有在[std::async](https://en.cppreference.com/w/cpp/thread/async)调用时出现的 shared state 才可能出现特殊行为，但还有其他创建 shared state，也就是说其他创建方式生成的[std::future](https://en.cppreference.com/w/cpp/thread/future)将可以正常析构
```cpp
int f() { return  1; }
std::packaged_task<int()> pt(f);
auto ft = pt.get_future(); // ft可以正常析构
std::thread t(std::move(pt)); // 创建一个线程来执行任务
int res = ft.get();
```
* **析构行为正常**的原因很简单
```cpp
{
    std::packaged_task<int()> pt(f);
    auto ft = pt.get_future(); // ft可以正常析构
    std::thread t(std::move(pt));
    ... // t.join() 或 t.detach() 或无操作
} // 如果t不join不detach，则此处t的析构程序终止
// 如果t已经join了，则ft析构时就无需阻塞
// 如果t已经detach了，则ft析构时就无需detach
// 因此std::packaged_task生成的ft一定可以正常析构
```

## 39 用[std::promise](https://en.cppreference.com/w/cpp/thread/promise)和[std::future](https://en.cppreference.com/w/cpp/thread/future)之间的通信实现一次性通知
* 让一个任务**通知**另一个**异步任务发生了特定事件**，一种实现方法是使用**条件变量**
```cpp
std::condition_variable cv;
std::mutex m;
bool flag(false);
std::string s("hello");

void f()
{
    std::unique_lock<std::mutex> l(m);
    cv.wait(l, [] { return flag; }); // lambda返回false则阻塞，并在收到通知后重新检测
    std::cout << s; // 若返回true则继续执行
}

int main()
{
    std::thread t(f);
    {
        std::lock_guard<std::mutex> l(m);
        s += " world";
        flag = true;
        cv.notify_one(); // 发出通知
    }
    t.join();
}
```
* 另一种方法是用[std::promise::set_value](https://en.cppreference.com/w/cpp/thread/promise/set_value)通知[std::future::wait](https://en.cppreference.com/w/cpp/thread/future/wait)
```cpp
std::promise<void> p;

void f()
{
    p.get_future().wait(); // 阻塞至p.set_value
    std::cout << 1;
}

int main()
{
    std::thread t(f);
    p.set_value(); // 解除阻塞
    t.join();
}
```
* 这种方法非常简单，但也有缺点，[std::promise](https://en.cppreference.com/w/cpp/thread/promise)和[std::future](https://en.cppreference.com/w/cpp/thread/future)之间的**shared state是动态分配的**，存在堆上的分配和回收成本。更重要的是，[std::promise](https://en.cppreference.com/w/cpp/thread/promise)只能设置一次，因此它和[std::future](https://en.cppreference.com/w/cpp/thread/future)的之间的通信**只能使用一次**，而**条件变量可以重复通知**。因此这种方法**一般用来创建暂停状态**的[std::thread](https://en.cppreference.com/w/cpp/thread/thread)
```cpp
std::promise<void> p;

void f()
{
    std::cout << 1;
}

int main()
{
    std::thread t([] { p.get_future().wait(); f(); });
    p.set_value();
    t.join();
}
```
* 此时可能会想到使用 RAII 版的[std::thread](https://en.cppreference.com/w/cpp/thread/thread)，但这并不安全
```cpp
int main()
{
    A t(
        std::thread([] { p.get_future().wait(); f(); }),
        A::DtorAction::join
    );
    ... // 如果此处抛异常，则set_value不会被调用，wait将永远不返回
    // 而RAII会在析构时调用join，join将一直等待线程完成，但wait使线程永不完成
    // 因此如果此处抛出异常，析构函数永远不会完成，程序将失去效应
    p.set_value();
}
```
* [std::condition_variable::notify_all](https://en.cppreference.com/w/cpp/thread/condition_variable/notify_all)可以**一次通知多个任务**，这也可以通过[std::promise](https://en.cppreference.com/w/cpp/thread/promise)和[std::shared_future](https://en.cppreference.com/w/cpp/thread/shared_future)之间的通信实现
```cpp
std::promise<void> p;

void f(int x)
{
    std::cout << x;
}

int main()
{
    std::vector<std::thread> v;
    auto sf = p.get_future().share();
    for(int i = 0; i < 10; ++i) 
        v.emplace_back([sf, i] { sf.wait(); f(i); });
    p.set_value();
    for(auto& x : v) x.join();
}
```

## 40 [std::atomic](https://en.cppreference.com/w/cpp/atomic/atomic)提供原子操作，volatile禁止优化内存
* Java 中的 volatile 变量提供了同步机制，C++ 的 volatile 变量和并发没有任何关系
* **[std::atomic](https://en.cppreference.com/w/cpp/atomic/atomic)是原子类型**，提供了原子操作
```cpp
std::atomic<int> i(0);

void f()
{
    ++i; // 原子自增
    ++i; // 原子自增
}
    
void g()
{
    std::cout << i;
}

int main()
{
    std::thread t1(f);
    std::thread t2(g); // 结果只能是0或1或2
    t1.join();
    t2.join();
}
```
* volatile 变量是普通的**非原子类型**，则不保证原子操作
```cpp
volatile int i(0);

void f()
{
    ++i; // 读改写操作，非原子操作
    ++i; // 读改写操作，非原子操作
}
    
void g()
{
    std::cout << i;
}

int main()
{
    std::thread t1(f);
    std::thread t2(g); // 存在数据竞争，值未定义
    t1.join();
    t2.join();
}
```
* 编译器或底层硬件对于**不相关的赋值会重新排序**以提高代码运行速度，[std::atomic](https://en.cppreference.com/w/cpp/atomic/atomic)可以限制重排序以**保证顺序一致性**
```cpp
std::atomic<bool> a(false);
int x = 0;

void f()
{
    x = 1; // 一定在a赋值为true之前执行
    a = true;
}

void g()
{
    if(a) std::cout << x;
}

int main()
{
    std::thread t1(f);
    std::thread t2(g);
    t1.join();
    t2.join(); // 不打印，或打印1
}
```
* **volatile 不会限制代码的重新排序**
```cpp
volatile bool a(false);
int x = 0;

void f()
{
    x = 1; // 可能被重排在a赋值为true之后
    a = true;
}

void g()
{
    if(a) std::cout << x;
}

int main()
{
    std::thread t1(f);
    std::thread t2(g);
    t1.join();
    t2.join(); // 不打印，或打印0或1
}
```
* volatile 的用处是告诉编译器**正在处理的是特殊内存，不要对此内存上的操作进行优化**。所谓优化指的是，如果把一个值写到内存某个位置，值会保留在那里，直到被覆盖，**因此冗余的赋值就能被消除**
```cpp
int x = 42;
int y = x;
y = x; // 冗余的初始化

// 优化为
int x = 42;
int y = x;
```
* 如果把一个值写到内存某个位置但从不读取，然后再次写入该位置，则**第一次的写入可被消除**
```cpp
int x;
x = 10;
x = 20;

// 优化为
int x;
x = 20；
```
* 结合上述两者
```cpp
int x = 42;
int y = x;
y = x;
x = 10;
x = 20;

// 优化为
int x = 42;
int y = x;
x = 20;
```
* 原子类型的读写也是可优化的
```cpp
std::atomic<int> y(x.load());
y.store(x.load());

// 优化为
register = x.load(); // 将x读入寄存器
std::atomic<int> y(register); // 用寄存器值初始化y
y.store(register); // 将寄存器值存入y
```
* 这种冗余的代码不会直接被写出来，但往往会隐含在大量代码之中。这种优化只在常规内存中合法，特殊内存则不适用。一般主存就是常规内存，**特殊内存一般用于 memory-mapped I/O** ，即与外部设备（如外部传感器、显示器、打印机、网络端口）通信。这个需求的原因在于，看似冗余的操作可能是有实际作用的
```cpp
int currentTemperature; // 传感器中记录当前温度的变量
currentTemperature = 25; // 更新当前温度，这条语句不应该被消除
currentTemperature = 26;
```
## 41 对于可拷贝的形参，如果移动成本低且一定会被拷贝则考虑传值
* 一些函数的形参本身就是用于拷贝的，考虑性能，对**左值实参应该执行拷贝**，对**右值**实参应该执行移动
```cpp
class A {
public:
    void f(const std::string& s) { v.push_back(s); }
    void f(std::string&& s) { v.push_back(std::move(s)); }
private:
    std::vector<std::string> v;
};
```
* 为同一个功能写两个函数太过麻烦，因此**改用为参数为转发引用的模板**
```cpp
class A {
public:
    template<typename T>
    void f(T&& s)
    {
        v.push_back(std::forward<T>(s));
    }
private:
    std::vector<std::string> v;
};
```
* 但模板会带来复杂性，一是模板一般要在头文件中实现，它可能在对象代码中产生多个函数，二是如果传入了不正确的实参类型，将出现十分冗长的错误信息，难以调试。
* 所以最好的方法是，**针对左值拷贝，针对右值移动**，并且在源码和目标代码中**只需要处理一个函数**，还能避开转发引用，而这种方法就是**按值传递**
```cpp
class A {
public:
    void f(std::string s) { v.push_back(std::move(s)); }
private:
    std::vector<std::string> v;
};
```
* C++98 中，按值传递一定是拷贝构造，但**在 C++11 中，只在传入左值时拷贝，如果传入右值则移动**
```cpp
A a;
std::string s("hi");
a.f(s); // 以传左值的方式调用
a.f("hi"); // 以传右值的方式调用
```
* 对比不同方法的开销，**重载和模板的成本是**，对左值一次拷贝，对右值一次移动（此外模板可以用转发实参直接构造，可能一次拷贝或移动都不要）。
* **传值**一定会对形参有一次拷贝（左值）或移动构造（右值），之后**再移动进容器**，因此对左值一次拷贝一次移动，对右值两次移动。对比之下，**传值只多出了一次移动，虽然成本高一些**，**但极大避免了麻烦**
* **可拷贝的形参才考虑传值**，因为 move-only 类型**只需要一个处理右值**类型的函数
```cpp
class A {
public:
    void f(std::unique_ptr<std::string>&& q) { p = std::move(q); }//构造拷贝
private:
    std::unique_ptr<std::string> p;
};
```
* 如果使用**传值**，则同样的调用需要**先移动构造形参**，**多出了一次移动**
```cpp
class A {
public:
    void f(std::unique_ptr<std::string> q/*构造*/) { p = std::move(q); }//构造拷贝
private:
    std::unique_ptr<std::string> p;
};
```
* 只有当移动成本低时，多出的一次移动才值得考虑，因此应该**只对一定会被拷贝的形参传值**
```cpp
class A {
public:
    void f(std::string s)
    {
        if (s.length() <= 15)
        { // 不满足条件则不添加，但多出了一次析构，不如直接传引用
            v.push_back(std::move(s));
        }
    }
private:
    std::vector<std::string> v;
};
```
* **之前的函数通过构造拷贝**，如果**通过赋值来拷贝**，**按值传递可能存在其他额外开销**，这取决于很多方面，比如传入类型是否使用动态分配内存、使用动态分配内存时赋值运算符的实现、赋值目标和源对象的内存大小、是否使用 SSO
```cpp
class A {
public:
    explicit A(std::string x) : s(std::move(x)) {}
    void f(std::string x) { s = std::move(x); }
private:
    std::string s;
};

std::string s("hello");
A a(s);

std::string x("hi");
a.f(x); // 额外的分配和回收成本，可能远高于std::string的移动成本
// 传引用则不会有此成本，因为现在a.s的长度比之前小

std::string y("hello world");
a.f(y); // a.s比之前长，传值和传引用都有额外的分配和回收成本，开销区别不大
```
* 还有一个与效率无关的问题，不同于按引用传递，按值传递容易遇见切片问题（the slicing problem）
```cpp
class A {};
class B : public A {};
void f(A); // 接受A或B
B b;
f(b); // 只能看到一个A而不是一个B，B的独有特征被切割掉
```

## 42 用emplace操作替代insert操作
* [std::vector::push_back](https://en.cppreference.com/w/cpp/container/vector/push_back)对左值和右值的重载为，直接传入字面值时，**会创建一个临时对象**。
```cpp
template <class T, class Allocator = allocator<T>>
class vector {
public:
    void push_back(const T& x);
    void push_back(T&& x);
};
```
* 使用[std::vector::emplace_back](https://en.cppreference.com/w/cpp/container/vector/emplace_back)则可以**直接用传入的实参调用元素的构造函数**，而不会创建任何临时对象
```cpp
class A {
public:
    A(int i) { std::cout << 1; }
    A(const A&) { std::cout << 2; }
    A(A&&) { std::cout << 3; }
    ~A() { std::cout << 4; }
};

std::vector<A> v;
v.push_back(1); // 134
// 先创建一个临时对象右值A(1)
// 随后将临时对象传入push_back，需要一次移动（不支持则拷贝）
// 最后析构临时对象

v.emplace_back(1); // 1：直接构造
```
* **所有 insert 操作都有对应的 emplace 操作**
```cpp
push_back => emplace_back // std::list、std::deque、std::vector
push_front => emplace_front // std::list、std::deque、std::forward_list
insert_after => emplace_after // std::forward_list
insert => emplace // 除std::forward_list、std::array外的所有容器
insert => try_emplace// std:map、std::unordered_map
emplace_hint // 所有关联容器
```
* 即使 insert 函数不需要创建临时对象，也可以用 emplace 函数替代，此时两者本质上做的是同样的事。因此 **emplace 函数就能做到 insert 函数能做的所有事**，有时甚至做得更好
```cpp
std::vector<std::string> v;
std::string s("hi");
// 下面两个调用的效果相同
v.push_back(s);
v.emplace_back(s);
```
* **emplace 不一定比 insert 快**。

  之前 emplace 添加元素到**容器末尾**，该位置**不存在对象**，因此新值会使用**构造方式**。

  但如果添加值到已有对象占据的位置，则会**采用赋值的方式**，于是必须**创建一个临时对象**作为移动的源对象，此时 emplace 并不会比 insert 高效
```cpp
std::vector<std::string> v { "hhh", "iii" };
v.emplace(v.begin(), "hi"); // 创建一个临时对象后移动赋值
```
* 对于[std::set](https://en.cppreference.com/w/cpp/container/set)和[std::map](https://en.cppreference.com/w/cpp/container/map)，**为了检查值是否已存在，emplace 会为新值创建一个 node**，以便能与容器中已存在的 node 进行比较。如果值不存在，则将 node 链接到容器中。如果值已存在，emplace 就会中止，node 会被析构，这意味着构造和析构的成本被浪费了，**此时 emplace 不如 insert 高效**
```cpp
#include <chrono>
#include <functional>
#include <iostream>
#include <set>
#include <iomanip>

class A {
    int a, b, c;
public:
    A(int _a, int _b, int _c) : a(_a), b(_b), c(_c) {}
    bool operator<(const A &other) const
    {
        if (a < other.a)  return true;
        if (a == other.a && b < other.b)  return true;
        return (a == other.a && b == other.b && c < other.c);
    }
};

constexpr int n = 100;

void set_emplace() {
    std::set<A> set;
    for (int i = 0; i < n; ++i)
        for (int j = 0; j < n; ++j)
            for (int k = 0; k < n; ++k)  set.emplace(i, j, k);
}

void set_insert() {
    std::set<A> set;
    for (int i = 0; i < n; ++i)
        for (int j = 0; j < n; ++j)
            for (int k = 0; k < n; ++k)   set.insert(A(i, j, k));
}

void test(std::function<void()> f)
{
    auto start = std::chrono::system_clock::now();
    f();
    auto stop = std::chrono::system_clock::now();
    std::chrono::duration<double, std::milli> time = stop - start;
    std::cout << std::fixed << std::setprecision(2) << time.count() << "  ms" << '\n';
}
int main()
{
    test(set_insert);  // 7640.89 ms
    test(set_emplace); // 7662.36 ms
    test(set_insert);  // 7575.74 ms
    test(set_emplace); // 7661.48 ms
    test(set_insert);  // 7575.80 ms
    test(set_emplace); // 7664.50 ms
}
```
* 创建临时对象并非总是坏事。若给存储[std::shared_ptr](https://en.cppreference.com/w/cpp/memory/shared_ptr)的容器添加一个自定义删除器的[std::shared_ptr](https://en.cppreference.com/w/cpp/memory/shared_ptr)对象
```cpp
std::list<std::shared_ptr<A>> v;

void f(A*);
v.push_back(std::shared_ptr<A>(new A, f));
// 或者如下，意义相同
v.push_back({ new A, f });
```
* **如果使用[emplace_back](https://en.cppreference.com/w/cpp/container/list/emplace_back)会禁止创建临时对象**。但这里临时对象带来的收益远超其成本。考虑如下可能发生的事件序列：
  * 创建一个[std::shared_ptr](https://en.cppreference.com/w/cpp/memory/shared_ptr)临时对象
  * [push_back](https://en.cppreference.com/w/cpp/container/list/push_back)以**引用方式接受临时对象**，分配内存时抛出了内存不足的**异常**
  * 异常传到[push_back](https://en.cppreference.com/w/cpp/container/list/push_back)之外，**临时对象被析构，于是删除器被调用，A被释放**
* 即使发生异常，也没有资源泄露。[push_back](https://en.cppreference.com/w/cpp/container/list/push_back)的调用中，由 new 构造的 A 会在临时对象被析构时释放。如果使用的是[emplace_back](https://en.cppreference.com/w/cpp/container/list/emplace_back)，new 创建的原始指针被完美转发到[emplace_back](https://en.cppreference.com/w/cpp/container/list/emplace_back)分配内存的执行点。
* 如果内存分配失败，抛出内存不足的异常，**异常传到[emplace_back](https://en.cppreference.com/w/cpp/container/list/emplace_back)外**，唯一可以获取堆上对象的原始指针丢失，于是就产生了**资源泄漏**
* 实际上不应该把 new A 这样的表达式直接传递给函数：
* **应该单独用一条语句来创建[std::shared_ptr](https://en.cppreference.com/w/cpp/memory/shared_ptr)**
* **将对象作为右值传递给函数**
```cpp
std::shared_ptr<A> p(new A, f);
v.push_back(std::move(p));
// emplace_back的写法相同，此时两者开销区别不大
v.emplace_back(std::move(p));
```
* emplace 函数在**调用[explicit](https://en.cppreference.com/w/cpp/language/explicit)构造函数时存在一个隐患**
```cpp
std::vector<std::regex> v;
v.push_back(nullptr); // 编译出错
v.emplace_back(nullptr); // 能通过编译，运行时抛出异常，难以发现此问题
```
* 原因在于[std::regex](https://en.cppreference.com/w/cpp/regex/basic_regex)接受 `const char　*`参数的构造函数被声明为[explicit](https://en.cppreference.com/w/cpp/language/explicit)，用[nullptr](https://en.cppreference.com/w/cpp/language/nullptr)赋值要求[nullptr](https://en.cppreference.com/w/cpp/language/nullptr)到[std::regex](https://en.cppreference.com/w/cpp/regex/basic_regex)的隐式转换，因此不能通过编译
```cpp
std::regex r = nullptr; // 错误
```
* 而[emplace_back](https://en.cppreference.com/w/cpp/container/vector/emplace_back)直接**传递实参给构造函数**，这个行为在编译器看来等同于
```cpp
std::regex r(nullptr); // 能编译但会引发异常，未定义行为
```

