# Effective-STL 笔记

* 本书针对性地介绍标准库容器和算法的使用准则，但不是STL的参考教程，读者需要有一定STL使用经验。

[TOC]

## 参考
* [The C++ Reference](./Reference_STL/algorithm)
* [The C++ Resources Network](http://www.cplusplus.com/reference/)
* [algorithm](./Reference_STL/algorithm.md)
* [numeric](./Reference_STL/numeric.md)
* [string类的六个查找函数](./Reference_STL/string%E7%B1%BB%E7%9A%84%E5%85%AD%E4%B8%AA%E6%9F%A5%E6%89%BE%E5%87%BD%E6%95%B0.md)
* [C++17 STL](./Reference_STL/Cpp_17_STL.md)
* [std::sort源码](./Reference_STL/std::sort%E6%BA%90%E7%A0%81.md)

### 前言:

本书讨论的STL是指:标准容器,`iostream`库的一部分，函数对象和算法
不包含标准容器适配器(`stack`、`deque`、`priority_queue`，因为没有迭代器支持)，数组(属于C++语言)
不包含标准C++库的扩展，比如散列容器、单链表和多种非标准函数对象等。

### 术语:

**标准序列容器**:`vector`,`string`,`deque`,`list`
**标准关联容器**:`set`,`multiset`,`map`,`multimap`
**迭代器**:输入、输出、前向、双向、随机
**仿函数类**:重载了`operator()`的类  使用仿函数对象的地方大部分可以用真函数替代.

# 一、容器

## 01 仔细选择你的容器

**连续内存容器**:vector,string,deque，**插入、删除**操作会使**迭代器失效** （移动导致）

**基于Node的容器**:`list`,**插入、删除不会使迭代器失效**

## 02 小心对“容器无关代码”的幻想

STL是建立在泛化的基础上的。

- 数组泛化为容器，参数泛化所包含对象的类型。
- 函数泛化为算法，参数泛化所用的迭代器类型。
- 指针泛化为迭代器，参数泛化所指向的对象的类型。

不同容器是不同的，优点和缺点大不相同，不要去对它们做包装

* **序列容器**支持`push_front`、`push_back`，但关联容器不支持
* **关联容器提供logN**复杂度的`lower_bound`、`upper_bound`和`equal_range`，**（N叉树）**
* 不同的容器是不同的，优缺点有重大不同。它们不被设计成可互换的，而且你做不了什么包装的工作
* **尽量用`typedef`来代替**冗长的`container<class>` 以及`container<class>::iterator`代码,使用typedef的好处还有，**换另一种容器方便**(以及更换`allocator`等其他`template`参数的时候)

```cpp
class Widget { ... };
typedef vector<Widget> WidgetContainer; //只修改一处
typedef WidgetContainer::iterator WCIterator; //只修改一处
//using WCIterator =WidgetContainer::iterator C++11
WidgetContainer cw;
Widget bestWidget;
...
WCIterator i = find(cw.begin(), cw.end(), bestWidget);
```
* 如果问题的改变是简单的加上用户的`allocator`时特别方便
```cpp
class Widget { ... };
template<typename T> // 关于为什么这里需要一个template
SpecialAllocator { ... }; // 请参见条款10
typedef vector<Widget, SpecialAllocator<Widget> > WidgetContainer;
typedef WidgetContainer::iterator WCIterator;
WidgetContainer cw; // 仍然能用
Widget bestWidget;
...
WCIterator i = find(cw.begin(), cw.end(), bestWidget); // 仍然能用
```
* 2.如果**不想对用户暴露**所使用容器的类型，则把容器进行封装，把容器类型定义在`private`域，**只提供相应的接口给用户**
```cpp
class CustomerList {
private:
    typedef list<Customer> CustomerContainer;
    typedef CustomerContainer::iterator CCIterator;
    CustomerContainer customers;//隐藏
public: // 通过这个接口
    ... // 限制list特殊信息的可见性
};
```

## 03 使容器里对象的拷贝操作轻量而正确
* STL里的容器，所有的操作，都是**基于拷贝**的，插入，读取，删除(导致移动)

* **分割问题**表明把派生类对象插入基类对象的容器几乎总是错的

  解决办法是是建立**智能指针的容器**，拷贝指针很快

## 04 用empty来代替检查size()是否为0
* 事实上empty的典型实现是一个返回size是否返回0的内联函数，对所有的标准容器
* `empty()`总是**常数时间(**因为只检查有没有)
  `size()`**不一定是常数时间**(可能需要遍历所有的成员比如list)

## 05 用区间成员函数代替单元素操作
`assign()`对于所有**标准序列容器**（`vector`，`string`，`deque`和`list`）都有效

目标区间是通过**迭代器指定的`copy`**都可以由**区间成员函数代替**,如`insert`

```cpp
int data[numValues]; // 假设numValues在其他地方定义
vector<int> v;
...
v.insert(v.begin(), data, data + numValues); // 把data中的int插入v前部
```
**连续内存序列容器**：使用区间版本insert相对于单元素insert的好处:

- (1)减少函数调用开销
- (2)单元素版本当把一个区间插入到头部的时候，**反复移动元素**，区间版本计算一次，移动一次
- (3)单元素版本每次insert可能多次内存分配，区间版本不会。
  (4)对于list来说，反复prev和next赋值也是开销
  (5)代码更少，程序含义更加明确,利于后期的维护

```cpp
vector<int>::iterator insertLoc(v.begin());
for (int i = 0; i < numValues; ++i) {
    insertLoc = v.insert(insertLoc, data[i]);
    ++insertLoc;
}

// 或者用copy，本质上和上面代码是一样的
copy(data, data + numValues, inserter(v, v.begin()));
```
* 用区间版本代替单元素插入的方法时，不要忘记**有些单元素变量函数伪装**。如果一个循环调用`push_front`或`push_back`，或一个算法，如`copy`，参数是`front_inserter`或者`back_inserter`.

* 采用`insert`的区间形式作为优先策略。

  **所有标准容器都支持的区间成员函数**
```cpp
// （1）区间构造
container::container(InputIterator begin, InputIterator end); 

// （2）区间插入，所有序列容器提供
void container::insert(iterator position, // 区间插入的位置（关联容器省略了position参数）
    InputIterator begin, // 插入区间的起点
    InputIterator end); // 插入区间的终点

// （3）区间删除
iterator container::erase(iterator begin, iterator end);//序列容器
void container::erase(iterator begin, iterator end);// 关联容器

// （4）区间赋值
void container::assign(InputIterator begin, InputIterator end);
```

## 06 警惕C++最令人恼怒的解析
* 假设有一个`int`文件，将这些`int`拷贝到一个`list`中
```cpp
ifstream dataFile("ints.dat");
list<int> data(istream_iterator<int>(dataFile), istream_iterator<int>());
//list<int> 是类型，声明名为data的函数
//不要在参数内递临时构建对象再来传入，而是先构建，再传入
```
* 解决办法是在数据声明中从使用匿名`istream_iterator`对象后退一步，仅仅给迭代器名字
```cpp
ifstream dataFile("ints.dat");
istream_iterator<int> dataBegin(dataFile);
istream_iterator<int> dataEnd;//不能加括号，否则又是函数声明了
list<int> data(dataBegin, dataEnd);
```

## 07 当使用new得指针的容器时，记得在销毁容器前delete那些指针
* 当一个指针的容器被销毁时，会销毁每个元素，**但`new`的对象不会调用`delete**`
```cpp
void f()
{
    vector<A*> v;
    for (int i = 0; i < 10; ++i)
    v.push_back(new A);
    ...
} // 出了作用域A泄漏！
```
* 可以这样
```cpp
void f()
{
    vector<A*> v; //不是异常安全
    ...
    for (vector<A*>::iterator i = v.begin(); i != v.end(); ++i) 
        delete *i;//删除前抛出异常,要把delete放到一个函数对象中
}
```
* 解决办法是把模板化从DeleteObject移到内部的`operator()`，编译器知道DeleteObject::operator()的指针类型，通过指针类型自动实例化一个`operator()`
```cpp
struct DeleteObject {
    template<typename T> // 模板化加在这里
    void operator()(const T* ptr) const
    {
        delete ptr;
    }
};
// 使用时无需再指定类型
for_each(v.begin(), v.end(), DeleteObject());
for_each(v.begin(), v.end(), [](string *p) {delete p; }); //版本3 lambda表达式
```
* 但仍不是异常安全的，解决方案是**用智能指针容器代替指针容器**
```cpp
void f()
{
    vector<shared_ptr<A>> v;
    for (int i = 0; i < 10; ++i) 
        v.push_back(shared_ptr<A>(new A));
    ...
} // 这里没有泄漏
```

## 08 永不建立auto_ptr的容器

拷贝auto_ptr所有权转移，值被置为NULL

## 09 在删除选项中仔细选择(没有通用方法)
* 删除容器`Container<int>` c中的**所有值为1963的对象**，删除方法因容器类型不同而不同，没有通用方法
```cpp
// 连续内存容器（vector、deque、string）最好的删除方法是erase-remove惯用法
c.erase(remove(c.begin(), c.end(), 1963), c.end()); 

// list的成员函数直接remove更高效
c.remove(1963);

// 关联容器没有remove，方法是调用erase，只花费对数时间（序列容器为线性时间）
c.erase(1963);
```
* 如果问题修改为删除`bool f(int x)`返回值为`true`的对象，序列容器只要把`remove`**替换为`remove_if`**
```cpp
c.erase(remove_if(c.begin(), c.end(), f), c.end());//序列容器 迭代器有效
c.remove_if(f); // c为list 更高效
```
* **关联容器**有两种方法，一种容易编码但**效率低的方法**是用`remove_copy_if`把需要的值拷贝到一个新容器，然后把原容器的内容和新的交换
```cpp
remove_copy_if(c.begin(), c.end(), 
                inserter(c2, c2.end()), f);
c.swap(c2);
```
* 另一种方法是避免拷贝开销，直接**遍历原容器删除，迭代器就失效**
* **解决方法是`erase`前得到下一个元素的迭代器**
```cpp
for (AssocContainer<int>::iterator it = c.begin(); it != c.end();) {//移除 for的++i
    if (f(*it)) 
        c.erase(it++);//后置++，跳到下一个元素
    else 
        ++it;
}
```
* 再增加问题，每次删除后要把一条消息写到日志文件中，**对关联容器很简单，只要加个打印**
```cpp
ofstream logFile;
AssocContainer<int> c;
for (AssocContainer<int>::iterator it = c.begin(); it !=c.end();) {
    if (f(*it)) {
        logFile << "Erasing " << *it <<'\n'; //add log
        c.erase(it++); // 删除元素,关联容器，使用it++保持迭代器有效
        it = c.erase(i);//删除元素,序列容器，使用`erase`的返回值保持迭代器有效
    }
    else ++it;
}
```
* 但`vector`、`string`和`deque`不能再使用`erase-remove`惯用法，因为没有办法让`erase`或`remove`写日志文件，也不能仿照关联容器用`erase`，**因为`erase`会使序列容器的被删元素和之后所有的元素迭代器失效**，再用自增显然会出错。
* **对于list，关联容器和序列容器的做法都可行**

## 10 注意分配器的协定和约束（建议跳过）
* 分配器最初被设想为抽象内存模型，在定义的内存模型中提供指针和引用的`typedef`才有意义。C++标准中类型T的对象的默认分配器（allocator<T>）提供typedef allocator<T>::pointer和allocator<T>::reference，而且也希望用户定义的分配器也提供这些typedef，**问题是在C++里没有办法捏造引用**
* 标准允许库实现假设每个分配器的`pointer typedef`是T*的同义词，每个分配器的`reference typedef`与`T&`相同。库实现可以忽视`typedef`并直接使用原始指针和引用，所以写出提供新指针和引用类型的分配器的方法也没用，因为**STL实现将忽视`typedef`**
* 标准允许STL实现认为相同类型的分配器等价，原因是，假设有两个容器v1和v2，把v2赋值到v1，销毁v1时v1的分配器要能回收由v2的分配器分配的内存，如果两者不等价，接合操作就很难实现
* 相同类型的分配器等价是十分严厉的约束，这代表如果要可移植，分配器就不能有状态，即不能有任何非静态数据成员，这表明不能有从两个不同的堆分配的SpecialAllocator<int>，它们是不等价的
* 分配器在分配原始内存方面类似`operator new`，但它们的接口不同，两者都带有一个指定要分配多少内存的参数，但对于`operator new`，这个参数指定的是字节数，而对于`allocator<T>::allocate`指定的是内存里能容纳多少个T对象。在`sizeof(int) == 4`的平台上，容纳一个int的内存得把4传给`operator new`，把1传给`allocator<int>::allocate`
```cpp
void* operator new(size_t bytes);
pointer allocator<T>::allocate(size_type numObjects);
// 本质上pointer就是T*的typedef
```
* 大多数标准容器从未调用它们例示的分配器，对list和所有标准关联容器都是如此（set、multiset、map和multimap）。原因是这些是基于节点的容器，这些容器所基于的数据结构是每当值被储存就动态分配一个新节点
```cpp
// list的可能实现
template<typename T, typename Allocator = allocator<T>>
class list {
private:
    Allocator alloc; // 用于T类型对象的分配器
    struct ListNode { // 链表里的节点
        T data:
        ListNode *prev;
        ListNode *next;
    };
    ...
};
```
* 添加一个新节点到list时需要从分配器获取内存，需要的不是T的内存而是包含了一个T的ListNode的内存，那使Allocator对象没用了，因为它为T分配内存而不为ListNode分配内存。list需要的是从它的分配器类型那里获得用于ListNode的对应分配器的方法，而分配器不能提供list需要的，这就是list不让Allocator做任何分配的原因
* 分配器模板A（例如，std::allocator，SpecialAllocator，等）都被认为有一个叫做rebind的内嵌结构体模板。rebind带有一个类型参数U，并且只定义一个typedef，other。 other是A<U>的一个简单名字。list<T>可以通过Allocator::rebind<ListNode>::other从它用于T对象的分配器（Allocator）获取对应的ListNode对象分配器
```cpp
template<typename T>
class allocator {
public:
    template<typename U>
    struct rebind {
        typedef allocator<U> other;
    }
...
};
```
* 如果你想要写自定义分配器，必须
  * 把分配器做成一个模板，模板参数T代表要分配内存的对象类型。
  * 提供pointer和reference的typedef，让pointer是`T*`，reference是`T&`。
  * 不要给你的分配器对象状态，分配器不能有非静态的数据成员
  * 传给分配器的allocate成员函数需要分配的对象个数而不是字节数，函数返回T*指针（通过pointer typedef），即使还没有T对象被构造
  * 提供标准容器依赖的内嵌rebind模板

## 11 理解自定义分配器的正确用法
* 假如有一个仿效malloc和free的程序，用于管理共享内存的堆
```cpp
void* mallocShared(size_t bytesNeeded);
void freeShared(void *ptr);
```
* 希望能把STL容器的内容放在共享内存中
```cpp
template<typename T>
class SharedMemoryAllocator {
public:
    ...
    pointer allocate(size_type numObiects, const void *localityHint = 0)
    {
        return static_cast<pointer>(mallocShared(numObiects * sizeof(T)));
    }
    void deallocate(pointer ptrToMemory, size_ type numObjects)
    {
        freeShared(ptrToMiemory);
    }
    ...
};
```
* 使用SharedMemoryAllocator
```cpp
typedef vector<double, SharedMemoryAllocator<double>> SharedDoubleVec;
...
{ // 开始一个块
    SharedDoubleVec v; // 建立一个元素在共享内存中的vector
    ...
}
```
* v分配来容纳它元素的内存将来自共享内存，但v本身只是一个普通的基于堆的对象，所以它将被放在运行时系统为基于堆的对象使用的任何内存，而非共享内存。为了把v的内容放进共享内存，必须这样
```cpp
// 分配足够的共享内存
void *pVectorMemory = mallocShared(sizeof(SharedDoubleVec));
// 用placement new建立一个SharedDoubleVec对象
SharedDoubleVec *pv = new (pVectorMemory) SharedDoubleVec;
...
pv->~SharedDoubleVec(); // 销毁共享内存中的对象
freeShared(pVectorMemory); // 销毁原来的共享内存块
```
* 除非真的要让一个容器（与它的元素相反）在共享内存里，否则避免手工的分配/建造/销毁/回收的过程。另外上述代码忽略了mallocShared可能返回一个null指针，产品代码必须考虑这种可能性。共享内存中的vector的建立由placement new完成而不是基本的new
* 再举一个例子，有两个堆，命名为Heap1和Heap2类。每个堆类有用于进行分配和回收的静态成员函数
```cpp
class Heap1 {
public:
    ...
    static void* alloc(size_t numBytes, const void *memoryBlockToBeNear);
    static void dealloc(void *ptr);
    ...
};
class Heap2 { ... }; // 有相同的alloc/dealloc接口
```
* 要在不同的堆里联合定位一些STL容器的内容，首先，设计一个分配器，使用像Heap1和Heap2那样用于真实内存管理的类
```cpp
template<typenameT, typename Heap>
class SpecificHeapAllocator {
public:
    pointer allocate(size_type numObjects, const void *localityHint = 0)
    {
        return static_cast<pointer>(Heap::alloc(numObjects * sizeof(T),
        localityHint));
    }
    void deallocate(pointer ptrToMemory, size_type numObjects)
    {
        Heap::dealloc(ptrToMemory);
    }
    ...
};
```
* 然后使用SpecificHeapAllocator来把容器的元素集合在一起
```cpp
// 把v和s的元素放进Heap1
// 把L和m的元素放进Heap2
vector<int, SpecificHeapAllocator<int, Heap1>> v;
set<int, SpecificHeapAllocator<int Heap1>> s;

list<Widget,
    SpecificHeapAllocator<Widget, Heap2>> L;
map<int, string, less<int>,
    SpecificHeapAllocator<pair<const int, string>, Heap2>> m;
```

## 12 对STL容器线程安全性的期待现实一些
* SGI定义的STL对多线程支持的黄金规则
  * **多个读取者是安全的**。多线程可能同时读取一个容器的内容，这将正确地执行。当然，在读取时不能
有任何写入者操作这个容器
  * 对不同容器的多个写入者是安全的。**多线程可以同时写不同的容器**
* 下列代码查找vector<int>中第一次出现5的位置，如果找到了，就把这个值改为0
```cpp
vector<int> v;
vector<int>::iterator first5(find(v.begin(), v.end(), 5)); // 行1
if (first5 != v.end()){ // 行2
    *first5 = 0; // 行3
}
```
* 多线程环境中，另一个线程可能在行1完成后修改v中的数据，这样行2对first5和v.end的检测就没有意义，同理行3中对*first5的赋值是不安全的，因为另一个线程可能在行2和行3之间执行并以某种方式使first5失效。要让上述代码线程安全，v必须从行1到行3保持锁定，STL实现很难，同步原语（如信号灯，互斥量）通常开销很大。不能期望任何STL实现解决问题，**必须手工对付这些情况中的同步控制**
```cpp
vector<int> v;
...
getMutexFor(v);
vector<int>::iterator first5(find(v.begin(), v.end(), 5));
if (first5 != v.end()) { // 这里现在安全了
    *first5 = 0; // 这里也是
}
releaseMutexFor(v);
```
* **更面向对象的解决方案是创建一个Lock模板类**，在**构造函数里获得互斥量并在析构函数里释放**，使getMutexFor和releaseMutexFor的调用不匹配的机会减到最小。使用一个类（像Lock）来管理资源的生存期（例如互斥量）的办法通常称为**资源获得即初始化**，即RAII
```cpp
// 获取和释放容器的互斥量的类的模板核心
// 忽略了很多细节
template<typename Container>
class Lock {
public:
    Lock(const Containers container) : c(container)
    {
        getMutexFor(c);
    }
    ~Lock()
    {
        releaseMutexFor(c);
    }
private:
    const Container& c;
};
```
* 使用方法
```cpp
vector<int> v;
...
{ // 建立新块
    Lock<vector<int> > lock(v);//Lock
    vector<int>::iterator first5(find(v.begin(), v.end(), 5));
    if (first5 != v.end()) {
    *first5 = 0;
    }
    //～Lock
}
```
# 二、 vector和string

## 13 尽量使用vector和string来代替动态分配的数组

1.new进行分配的话，要确保(**安全性**)

- (1)有`delete`
- (2)`delete`是正确的形式
- (3)只`delete`一次(所以通常`delete`接把指针=`nullptr`)

2.容器还有很多算法
3.`vector`和`string`也都是兼容C风格的遗留代码的，因为是**基于数组实现**的
4.唯一可能的问题是,`string`**采用了引用计数**,可以用`vector<char>`来代替

## 14 使用reserve来避免不必要的重新分配
* 只要不超过STL容器的最大大小，就可以自动增长到足以容纳放进去的数据，这个最大值可以调用max_size成员函数查看
* 对于vector和string，需要更多空间时就以realloc等价的思想来增长，这个类似于realloc的操作有四个部分，这些步骤发生时所有指向vector或string中的迭代器、指针和引用都会失效
  * 分配新的内存块，它有容器目前容量的几倍。大部分实现中vector和string的容量每次以2为因数增长，即翻倍
  * 把所有元素从容器的旧内存拷贝到新内存
  * 销毁旧内存中的对象
  * 回收旧内存
* resize把size改为n，n\<size则尾部元素被销毁，否则重新分配，默认构造的新元素会添加到容器尾部。reserve把capacity改为n，如果n\<capacity，对vector这个调用什么都不做，string把容量减少为size和n中大的数，但size不变
```cpp
size();
capacity();
resize(Container::size_type n);
reserve(Container::size_type n);
```
* 假定不使用reserve建立一个容纳1-1000vector\<int\>，循环过程中将会导致2到10次重新分配（1000约等于2^10）
```cpp
vector<int> v;
for (int i = 1; i <= 1000; ++i) v.push_back(i);
```
* 代码改为使用reserve则不会有重新分配
```cpp
vector<int> v;
v.reserve(1000);
for (int i = 1; i <= 1000; ++i) v.push_back(i);
```
* 注意，reserve只是改动空间大小，而不能直接用下标赋值
```cpp
vector<int> v;
v.reserve(4);
v[0] = 2; // 下标溢出
v[3] = 1; // 下标溢出
```
* 用下标赋值的正确做法如下
```cpp
vector<int> v(4); // v中包含4个0
v[0] = 2;
v[3] = 1;
```
* 大小和容量之间的关系可以预测重新分配的时机，避免插入使指向容器中的迭代器、指针和引用失效
```cpp
string s;
...
if (s.size() < s.capacity()) {
    s.push_back('x'); // 不会使容器的迭代器失效
}
```

## 15 小心string实现的多样性
* string对象的大小可能是1到至少7倍char\*指针的大小，为了理解存在差别的原因，必须知道string可能存的数据和保存的位置
* 实际上每个string实现都容纳了下面的信息，不同的string实现以不同的方式把这些信息放在一起
  * 字符串的大小
  * 容纳字符串字符的内存容量
  * 这个字符串的值，即构成这个字符串的字符。
  * 一个string可能容纳它的配置器的拷贝
  * 依赖引用计数的string实现包含了这个值的引用计数
* 实现A中，每个string对象包含一个配置器的拷贝，字符串的大小，容量，和一个指向包含引用计数和字符串值的动态分配的缓冲区的指针。这里一个使用默认配置器的字符串对象是指针大小的四倍，对于一个自定义的配置器，string对象会随配置器对象的增大而变大

![](Effective-STL.assets/2-1.png)

* 实现B的string对象和指针一样大，因为在结构体中只包含一个指针。这里仍然假设使用默认配置器。正如实现A，如果使用自定义配置器，这个string对象的大小会增加大约配置器对象的大小。实现B中，使用默认配置器不占用空间，这归功于这里用了一个在实现A中没有的使用优化。B的string指向的对象包含字符串的大小、容量和引用计数，以及容纳字符串值的动态分配缓冲区的指针，也包含在多线程系统中与并发控制有关的一些附加数据，用于并发控制的数据是一个指针大小的6倍

![](Effective-STL.assets/2-2.png)

* 实现C的string对象总是等于指针的大小，但是这个指针指向一个包含所有与string相关的东西的动态分配缓冲器：它的大小、容量、引用计数和值。没有per-object allocator的支持。缓冲区也容纳一些关于值可共享性的数据，我们在这里不考虑这个主题，标记为“X”

![](Effective-STL.assets/2-3.png)

* 实现D的string对象是一个指针大小的七倍（仍然假设使用了默认配置器）。这个实现没有使用引用计数，但每个string包含了一个足以表现最多15个字符的字符串值的内部缓冲区，因此小的字符串可以被整个保存在string对象中，这是一种优化策略，当一个string的容量超过15时，缓冲器的第一部分被用作指向动态分配内存的一个指针，而字符串的值存放在那块内存中。在VS中空string的size就是15，sizeof(string)是28

![](Effective-STL.assets/2-4.png)

* 实现D没有动态分配，实现A和C下一次，实现B下两次（一次是string对象指向的对象，一次是那个对象指向的字符缓冲区），因此新字符串值的建立可能需要0、1或2次动态分配。如果关心动态分配和回收内存的次数，或伴随这样分配的内存开销，避开实现B

## 16 如何将vector和string的数据传给遗留的API
* 如果有一个vector对象v，需要得到一个指向v中数据的指针，使得它可以被当作一个数组，只要使用&v[0]就可以了，对于string对象s，相应的是的s.c_str()
* 如果对于数组
```cpp
void f(const int* pInts, size_t numInts);
```
* 改成vector要考虑v.size()为0的情况，因为&v[0]将是未定义的
```cpp
if (!v.empty()) {
    doSomething(&v[0], v.size());
}
```
* begin的返回类型是iterator，而不是一个指针，需要一个指向vector内部数据的指针时绝不该使begin，如果键入v.begin()，就应该键入&\*v.begin()，而这产生和&v[0]相同的指针，却显得更晦涩
* 对string来说则不用考虑，因为string长度为0也能工作，c_str()将返回一个指向null字符的指针，但这会被解释为字符串结束，对char*有影响
```cpp
void doSomething(const char *pString);
doSomething(s.c_str());
```
* 如果想用C风格函数返回的元素初始化一个vector，可以利用vector和数组内存分布的兼容性将存储vector元素的空间传给函数
```cpp
size_t fillArray(double *pArray, size_t arraySize);
vector<double> vd(maxNumDoubles);
vd.resize(fillArray(&vd[0], vd.size()));
```
* 这个技巧只能用于vector，因为只有vector承诺了与数组具有相同的潜在内存分布。如果想初始化string对象，只要让API将数据放入一个vector<char>，然后从vector中将数据拷到string
```cpp
size_t fillString(char *pArray, size_t arraySize);
vector<char> vc(maxNumChars);
size_t charsWritten = fillString(&vc[0], vc.size());
string s(vc.begin(), vc.begin()+charsWritten);
```
* 让C风格API把数据放入一个vector，然后拷到STL容器的做法总是有效的
```cpp
size_t fillArray(double *pArray, size_t arraySize);
vector<double> vd(maxNumDoubles);
vd.resize(fillArray(&vd[0], vd.size()));
deque<double> d(vd.begin(), vd.end());
list<double> l(vd.begin(), vd.end());
set<double> s(vd.begin(), vd.end());
```
* 反之，vector和string以外的STL容器要将数据传给C风格API，只要把数据拷到vector再传给API
```cpp
void doSomething(const int* pints, size_t numInts);
set<int> intSet;
...
vector<int> v(intSet.begin(), intSet.end());
if (!v.empty()) doSomething(&v[0], v.size());
```

## 17 使用“交换技巧”来修整过剩容量
* 要避免vector持有不再需要的内存，需要把它从曾经最大的容量减少到现在需要的容量，这样减少容量的方法常被称为shrink to fit，C++11加入了shrink_to_fit函数，实现起来其实很简单
```cpp
vector<Contestant>(contestants).swap(contestants);
```
* 新建一份临时vector，只会拷贝已有元素，所以临时vector没有多余容量，再交换，接着就被销毁，这样以前的vector就收缩完成了。string也是同理
```cpp
string s;
... // 使s变大，然后删除所有字符
string(s).swap(s);
```
* 同理，交换技巧的变体可以用于清除容器和减少它的容量到实现提供的最小值
```cpp
vector<Contestant> v;
string s;
... // 使用v和s
vector<Contestant>().swap(v); // 清除v并最小化容量
string().swap(s); // 清除s并最小化容量
```

## 18 避免使用vector\<bool\>
* vector\<bool\>只有两个问题。第一，它不是一个STL容器。第二，它并不容纳bool
* STL容器就必须满足所有在C++标准23.1节中列出的容器必要条件。这些要求中有这样一条：如果c是一个T类型对象的容器，且c支持operator[]，那么以下代码必须能够编译
```cpp
T *p = &c[0]; // 无论operator[]返回什么都可以用这个地址初始化一个T*
```
* 换句话说，如果用operator[]来得到Container\<T\>中的一个T对象，可以通过取它的地址而获得指向那
个对象的指针。因此如果vector<bool>是一个容器，以下代码必须能够编译
```cpp
vector<bool> v;
bool *pb = &v[0]; // 用vector<bool>::operator[]返回值的的地址初始化一个bool*
```
* 但它不能编译，vector\<bool\>是一个伪容器，并不保存真正的bool，而是打包bool以节省空间。在一个典型的实现中，每个保存在“vector”中的“bool”占用一个单独的bit，而一个8bit的字节将容纳8个bool。在内部，vector\<bool\>使用了与位域等价的思想来表示它假装容纳的bool
* vector\<bool\>::operator[]需要返回指向一个比特的引用，而并不存在这样的东西。为了解决这个难题，vector\<bool\>::operator[]返回一个对象，其行为类似于比特的引用，也称为代理对象
```cpp
template <typename Allocator>
vector<bool, Allocator> {
public:
    class reference {...}; // 用于产生引用独立比特的代理类
    reference operator[](size_type n); // operator[]返回一个代理
    ...
}
```
* 而这样，代码不能编译的原因就很明显了，因为&v[0]是vector\<bool\>::reference\*类型而非bool\*
* vector\<bool\>存在于标准中，而它并不是一个容器，标准库提供了两个替代品，分别是deque\<bool\>和bitset。deque\<bool\>是一个STL容器，它保存真正的bool值。bitset不是一个STL容器，大小（元素数量）在编译期固定，因此它不支持插入和删除元素，但就像vector\<bool\>，它使用一个压缩的表示法，使得它包含的每个值只占用一比特，提供vector\<bool\>特有的flip成员函数，还有一系列其他操作位集（collection of bits）所特有的成员函数。如果不在乎没有迭代器和动态改变大小，可以使用bitset

# 三、 关联容器

## 19 了解相等和等价的区别

* find算法和set::insert是判断两个值是否相同的函数代表，它们以不同的方式完成，find的相同基于operator==，set::insert对相同的定义是等价，通常基于operator<
* 相等基于operator==，在类中重载时，即使两个类不完全相同也可以相等
* 等价基于一个有序区间中对象值的相对位置，标准关联容器保持有序，所以每个容器必须定义一个保持有序的比较函数（默认是less），如果一个元素不在另一个之前（关于某个排序标准），则这两个元素是等价的（按照这个标准）。如果用相等决定两个对象是否有相同的值，除了排序的比较函数还需要一个用于判断两个值是否相等的比较函数（习惯用operator==）
* 一般关联容器的比较函数不是operator<或less，而是用户定义的判断式，标准关联容器通过key_comp成员函数访问排序判断式，key_comp默认是一个 std::less 对象，类似操作符 operator<，返回容器中用来比较主键的比较对象的一份拷贝
```cpp
!c.key_comp()(x, y) && !c.key_comp()(y, x) // 为true则xy等价
```
* 为了更理解相等和等价的区别，考虑一个忽略大小写的set<string>，item35实现了忽略大小写比较的ciStringCompare函数，这里写一个仿函数类，类的operator()调用ciStringCompare
```cpp
struct CIStringCompare : public binary_function<string, string, bool>
{ // 这个基类信息见item40
    bool operator()(const string& lhs, const string& rhs) const
    {
        return ciStringCompare(lhs, rhs);
    }
}
```
* 利用这个函数对象建立一个忽略大小写的set<string>
```cpp
set<string, CIStringCompare> ciss; // case-insensitive string set
ciss.insert("Persephone"); // 添加到set中
ciss.insert("persephone"); // 未添加到set中
```
* 用set的find成员函数搜索字符串“persephone”会成功，但用find算法会失败，因为前者基于等价，后者基于相等，“persephone”和"Persephone"在此定义中等价但不相等
```cpp
if (ciss.find("persephone") != ciss.end())... // true
if (find(ciss.begin(), ciss.end(), "persephone") != ciss.end())... // false
```

## 20 为指针的关联容器指定比较类型
* 假定有一个string*指针的set，把一些动物的名字插入进set
```cpp
set<string*> ssp; // set of string ptrs
ssp.insert(new string("Anteater"));
ssp.insert(new string("Wombat"));
ssp.insert(new string("Lemur"));
ssp.insert(new string("Penguin"));
```
* 接着希望用下列代码使字符串按字母顺序出现
```cpp
// 希望看到“Anteater”，“Lemur”，“Penguin”，"Wombat"
for (set<string*>::const_iterator i = ssp.begin(); i != ssp.end(); ++i)
    cout << *i << endl;
```
* 然而结果只能看见四个十六进制数，因为set元素为指针，\*i是一个string的指针。马上会想出把\*i改为\**i，但这样也不能保证输出按字母顺序出现，因为set中保存的是指针，以指针值排序而非string值
* 为了解决这个问题，首先应要知道set<string*> ssp是下列代码的简写
```cpp
set<string*, less<string*>, allocator<string*>> ssp;
```
* 因此要string\*指针以字符串值顺序存储在set中，不能用默认的仿函数类less<string\*>，必须改为自己的比较仿函数类，它的对象带有string\*指针并按指向的字符串值排序
```cpp
struct StringPtrLess : public binary_function<const string*, const string*, bool>
{ // 这个基类信息见item40
    bool operator()(const string *ps1, const string *ps2) const
    {
        return *ps1 < *ps2;
    }
};
```
* 然后用StringPtrLess作为比较类型，循环就可以得到按字母顺序排序的输出了
```cpp
typedef set<string*, StringPtrLess> StringPtrSet;
StringPtrSet ssp;
for (StringPtrSet::const_iterator i = ssp.begin(); i != ssp.end(); ++i)
    cout << **i << endl;
```
* 可以改为使用算法，写一个函数给for_each联用
```cpp
void print(const string *ps)
{
    cout << *ps << endl;
}
for_each(ssp.begin(), ssp.end(), print); // 对ssp的每个元素上调用print
```
* 或者用泛型的解引用仿函数类，然后让它和transform与ostream_iterator联用
```cpp
struct Dereference {
    template <typename T>
    const T& operator()(const T *ptr) const
    {
        return *ptr;
    }
};
// 通过解引用“转换”ssp中的每个元素，把结果写入cout
transform(ssp.begin(), ssp.end(), ostream_iterator<string>(cout, "\n"), Dereference());
```
* 需要一个仿函数类而不是一个简单的比较函数的原因是，set不要一个函数，它要的是能在内部用实例化建立函数的一种类型
* 建立指针的关联容器得指定容器的比较类型，大多数时候，比较类型只是解引用指针并比较所指向的对象，因此手头最好有一个这样的仿函数模板
```cpp
struct DereferenceLess {
    template <typename PtrType>
    bool operator()(PtrType pT1, PtrType pT2) const
    {
        return *pT1 < *pT2;
    }
};

set<string*, DereferenceLess> ssp; // 行为就像set<string*, StringPtrLess>
```
* 对于智能指针或迭代器的关联容器，也得指定比较类型。指针的解决方案也可以用于类似指针的对象，正如DereferenceLess适合作为T*的关联容器的比较类型一样，它也可以作为T对象的迭代器和智能指针容器的比较类型

## 21 永远让比较函数对相等的值返回false
* 建立一个set，比较类型用less_equal，然后插入两次10
```cpp
set<int, less_equal<int> > s; // s以“<=”排序
s.insert(10);
s.insert(10);
```
* 对于后一个insert调用，set必须先要判断出10是否已经位于其中，为了区分，将前一个10称为10A，后一个称为10B。set遍历内部数据结构以查找适合插入10B的位置，最终使用比较函数检查10B是否与10A等价，这里比较函数为less_equal，而less_equal意思就是operator<=。于是，set将计算这个表达式
```cpp
!(10A <= 10B) && !(10B <= 10A)
```
* 这个表达式结果显然是false，于是set认为10A与10B不等价，然后将10B插入到10A旁边，于是set有了两个10，因此用less_equal作为比较类型破坏了容器，同理任何对相等的值返回true的比较函数都会做同样的事情
```cpp
// 对item20的StringPtrLess中的operator()结果取反实现StringPtrGreater
struct StringPtrGreater : public binary_function<const string*, const string*, bool> {
    bool operator()(const string *ps1, const string *ps2) const
    {
        return !(*ps1 < *ps2);
    }
};
// 这里取反返回的是>=而不是>，因此这是一个无效的比较函数
// 需要改为 return *ps2 < *ps1;
```
* 不仅仅对于map和set如此，对于multiset和multimap也是同理。将开头的代码改为multiset
```cpp
multiset<int, less_equal<int>> s; // s仍然以“<=”排序
s.insert(10);
s.insert(10);
```
* s有两个10，对它做一个equal_range，希望得到一对指出包含这两个拷贝的范围的迭代器，而equal_range指示等价而非相等的值的范围，比较函数认为10A和10B是不等价的，所以不会同时出现在equal_range指示的范围内。因此，除非比较函数总是为相等的值返回false，否则将会打破所有的标准关联型容器

## 22 避免原地修改set和multiset的键
* 对于于map和multimap，试图改变容器里的一个键值的程序将不能编译，因为map<K, V>或multimap<K, V>元素的类型是pair<const K, V>
* 原地修改键对map和multimap来说是不可能的（除非用const_cast去掉const属性，但没有人会这样做），但对set和multiset却是可能的，因为set<T>或multiset<T>的元素类型是T而非const T
* 首先要理解为什么set里的类型不是const
```cpp
// 一个雇员类
class Employee {
public:
    ...
    const string& name() const; // 获取雇员名
    void setName(const string& name); // 设置雇员名
    const string& getTitle() const; // 获取雇员头衔
    void setTitle(string& title); // 设置雇员头衔
    int idNumber() const; // 获取雇员ID号
    ...
};
// 以ID号来排序
struct IDNumberLess : public binary_function<Employee, Employee, bool> {
    bool operator()(const Employees lhs, const Employee& rhs) const
    {
        return lhs.idNumber() < rhs.idNumber();
    }
};
// 建立雇员类的set
typedef set<Employee, IDNumberLess> EmpIDSet;
EmpIDSet se; // se按ID号排序
// ID是主键不能修改，但对雇员的头衔做修改是合法的
// 而这种行为的合法则决定了set元素不能是const
Employee selectedID; // 容纳被选择雇员的ID号
...
EmpIDSet::iterator i = se.find(selectedID);
if (i != se.end())
{
    i->setTitle("Corporate Deity"); // 给雇员新头衔
}
```
* 上面的原因对于map来说其实也是适用的，但标准委员会认为map/multimap键应该是const而set/multiset的值不是
* 即使set和multiset的元素不是const，仍然有很多阻止修改方式，比如让用于set<T>::iterator的operator*返回一个const T&，不过这样的实现不一定合法，根据不同编译器有不同的表现结果，既然标准模棱两可，试图修改set或multiset中元素的代码就不可移植
* 如果不关心移植性，要改变set或multiset中元素的值，编译器可以通过，那就保持这样，但要确定不要改变元素的键部分，即影响容器有序性的元素部分。如果在乎移植性，就认为set和multiset中的元素在没有const_cast的情况下不能被修改
```cpp
EmpIDSet::iterator i = se.find(selectedID);
if (i != se.end()) {
    i->setTitle("Corporate Deity"); // 有些编译器不能通过，因为*i是const
}
// 为了编译通过使用const_cast
if (i != se.end()) {
   const_cast<Employee&>(*i).setTitle("Corporate Deity");
}
// static_cast不可行，因为修改的只是临时对象
if (i != se.end()) {
    static_cast<Employee>(*i).setTitle("Corporate Deity");
}
// 等价于
if (i != se.end()){
    Employee tempCopy(*i); // 把*i拷贝到tempCopy
    tempCopy.setTitle("Corporate Deity"); // 修改tempCopy
}
```
* 要安全地改变set、multiset、map或multimap里的元素，按五个简单的步骤去做
  * 定位想要改变的容器元素
  * 拷贝一份要被修改的元素
  * 修改副本，使它有你想要在容器里的值
  * 调用erase删除容器中的元素
  * 把新值插入容器。如果新元素在容器的排序顺序中的位置正好相同或相邻于删除的元素，使用insert
的“提示”形式（第一步获得的迭代器作为提示）把插入的效率从对数时间改进到分摊的常数时间
```cpp
typedef set<Employee, IDNumberLess> EmpIDSet;
EmpIDSet se;
Employee selectedID;
...
EmpIDSet::iterator i = se.find(selectedID); // 第一步：找到要改变的元素
if (i!=se.end())
{
    Employee e(*i); // 第二步：拷贝元素
    se.erase(i++); // 第三步：删除元素，自增保持迭代器有效
    e.setTitle("Corporate Deity"); // 第四步：修改副本
    se.insert(i, e); // 第五步：插入新值，提示它的位置和原先元素的一样
}
```

## 23 考虑用有序vector代替关联容器
* 标准关联容器的典型实现是平衡二叉查找树，平衡二叉查找树是对插入、删除和查找的混合操作优
化的数据结构，因此它的设计目标是优化这些混合操作
* 很多应用中的数据结构没有那么混乱，它们有三个不同的阶段
  * 建立。通过插入很多元素建立一个新的数据结构。在这个阶段，几乎所有的操作都是插入和删除，几
乎没有查找
  * 查找。在数据结构中查找指定的信息片。在这个阶段，几乎所有的操作都是查找，几乎没有插入和删除
  * 重组。通过删除所有现有数据或在原地插入新数据修改内容。这个阶段的行为等价于阶段1，一旦这个阶段完成，程序返回阶段2
* 对于使用上述数据结构的应用来说，vector可能比关联容器性能高（时间和空间上都是）。但必须是有序vector，因为有序容器才能使用查找算法binary_search、lower_bound、equal_range等
* vector二分查找比二叉树的二分查找性能好的原因是，平衡二叉树的每个类对象要附加左右孩子和父节点三个指针，而vector没有这种开销。假如数据结构足够大，分为多个内存页面，类的大小为12个字节，指针为4个字节，一个内存页面为4096（4K）字节，用vector可以保存4096/12=341个对象，而关联容器只能保存一半
* 当然，有序vector的最大缺点是必须保持有序，插入和删除开销大，新元素插入时，大于新元素的所有元素都必须向上移一位，删除同理。因此数据结构使用查找而几乎不用插入和删除时，有序vector代替关联容器才有意义
* 使用有序vector代替set的代码框架，其中最难的是选择搜索算法，见item45
```cpp
vector<Widget> vw;
... // 建立阶段
sort(vw.begin(), vw.end()); // 结束建立阶段，模拟multiset时用stable_sort，见item31
Widget w; // 用于查找的值的对象
... // 开始查找阶段
if (binary_search(vw.begin(), vw.end(), w))... // 通过binary_search查找
vector<Widget>::iterator i = lower_bound(vw.begin(), vw.end(), w); // 通过lower_bound查找
if (i != vw.end() && !(w < *i))...
pair<vector<Widget>::iterator, vector<Widget>::iterator> range =
    equal_range(vw.begin(), vw.end(), w); // 通过equal_range查找
if (range.first != range.second)...
... // 结束查找阶段，开始重组阶段
sort(vw.begin(), vw.end()); // 开始新的查找阶段...
```
* map中的元素类型是pair<const K, V>，要用vector模拟map或者multimap时必须去掉const，因为对vector排序时，元素值会通过赋值移动
* map和multimap排序只作用于元素的key，而pair的operator<作用于pair的两个组件，所以排序vector时要给pair自定义比较函数，排序的比较函数的参数是两个pair。另外还需要第二个比较函数用于查找，查找只用到key，需要传给查找的比较函数一个key和一个pair，因为不知道key还是pair是作为第一个实参传递的，所以需要写两个函数
```cpp
typedef pair<string, int> Data;
class DataCompare { // 用于比较的类
public:
    bool operator()(const Data& lhs, const Data& rhs) const // 用于排序的比较函数
    {
        return keyLess(lhs.first, rhs.first);
    }
    bool operator()(const Data& Ihs, const Data::first_type& k) const // 用于查找的比较函数（形式1）
    {
        return keyLess(lhs.first, k);
    }
    bool operator()(const Data::first_type& k, const Data& rhs) const // 用于查找的比较函数（形式2）
    {
        return keyLessfk, rhs.first);
    }
private:
    bool keyLess(const Data::first_type& k1, const Data::first_type& k2) const // 真正的比较函数
    {
        return k1 < k2;
    }
};

vector<Data> vd; // 代替map<string, int>
... // 建立阶段
sort(vd.begin(), vd.end(), DataCompare()); // 结束建立阶段，模拟multimap时用stable_sort
string s; // 用于查找的值的对象
... // 开始查找阶段
if (binary_search(vd.begin(), vd.end(), s, DataCompare()))... // 通过binary_search查找
vector<Data>::iterator i = lower_bound(vd.begin(), vd.end(), s, DataCompare()); // 通过lower_bound查找
if (i != vd.end() && !DataCompare()(s, *i))...
pair<vector<Data>::iterator, vector<Data>::iterator> range =
    equal_range(vd.begin(), vd.end(), s, DataCompare()); // 通过equal_range查找
if (range.first != range.second)...
... // 结束查找阶段，开始重组阶段
sort(vd.begin(), vd.end(), DataCompare()); // 开始新的查找阶段...
```

## 24 当关乎效率时应该在map::operator[]和map-insert之间仔细选择
* map::operator[]被设计为简化“添加或更新”功能，对于m[k] = v，检查键k，如果k已经在map里，关联值被更新成v，否则就添加上，以v作为对应值。原理是operator[]返回一个与k关联的值对象的引用，然后v赋值给所引用对象，如果k不在map里，operator[]就没有可以引用的值对象，于是值类型的默认构造函数重新建立一个，operator[]返回新建立对象的引用
```cpp
map<int, Widget> m;
m[1] = 1.50;
// 等价于
typedef map<int, Widget> IntWidgetMap;
pair<IntWidgetMap::iterator, bool> result = m.insert(IntWidgetMap::value_type(1, Widget())); 
result.first->second = 1.50;
```
* 由上述代码可知，用想要的值构造Widget比默认构造Widget再赋值显然更高效。用insert替换operator[]，节省了三次函数调用：默认构造Widget对象，销毁临时对象和一个赋值操作
```cpp
m.insert(IntWidgetMap::value_type(1, 1.50));
```
* 添加时insert比operator[]更高效，当等价的键已经在map里再更新时正好相反。insert需要IntWidgetMap::value_type类型实参（即pair<int, Widget>），调用insert时必须构造和析构一个该类型对象，耗费了一对构造函数和析构函数，因为pair<int, Widget>本身包含了一个Widget对象，还会造成一个Widget的构造和析构，operator[]没有使用pair对象，所以没有构造和析构pair和Widget
```cpp
m[k] = v; // 使用operator[]
m.insert(IntWidgetMap::value_type(k, v)).first->second = v; // 使用insert
```
* 可以自己实现一个对于添加和更新都适用的函数
```cpp
template<typename MapType, typename KeyArgType, typename ValueArgtype>
typename MapType::iterator efficientAddOrUpdate
    (MapType& m, const KeyArgType& k, const ValueArgtype& v)
{
    typename MapType::iterator Ib = m.lower_bound(k);
    if(Ib != m.end() && !(m.key_comp()(k, Ib->first)))
    {
        Ib->second = v;
        return Ib;
    }
    else
    {
        typedef typename MapType::value_type MVT;
        return m.insert(Ib, MVT(k, v)); // 把pair(k, v)添加到m并返回指向新map元素的迭代器
    }
}

efficientAddOrUpdate(m, 10, 1.5);
// 如果m已经包含键是10的元素，推断出ValueArgType是double
// 函数体直接把1.5作为double赋给与10相关的那个Widget
```

## 25 熟悉非标准散列容器
* 兼容STL的散列关联容器可以从多个来源获得，而且它们的名字通常是：hash_set、hash_multiset、hash_map和hash_multimap。但因为没有遵循一个标准实现，为了避开这些名字，在C++标准委员会的议案中，散列容器的名字是unordered_set、unordered_multiset、unordered_map和unordered_multimap，C++11中引入了这些容器
![](Effective-STL.assets/4-1.png)

* 输入迭代器：只能用来读取指向的值。当该迭代器自加时，之前指向的值就不可访问。也就是说，不能使用这个迭代器在一个范围内遍历多次。std::istream_iterator就是这样的迭代器
* 前向迭代器：类似于输入迭代器，不过其可以在指示范围内迭代多次。`std::forward_list`就是这样的迭代器。就像一个单向链表一样，只能向前遍历，不能向后遍历，但可以反复迭代
* 双向迭代器：从名字就能看出来，这个迭代器可以自增，也可以自减，迭代器可以向前或向后迭代。`std::list`，`std::set`和`std::map`都支持双向迭代器
* 随机访问迭代器：与其他迭代器不同，随机访问迭代器一次可以跳转到任何容器中的元素上，而非之前的迭代器，一次只能移动一格。`std::vector`和`std::deque`的迭代器就是这种类型
* 连续迭代器：这种迭代器具有前述几种迭代器的所有特性，不过需要容器内容在内存上是连续的，类似一个数组或`std::vector`
* 输出迭代器：该迭代器与其他迭代器不同。因为这是一个单纯用于写出的迭代器，其只能增加，并且将对应内容写入文件当中。如果要读取这个迭代中的数据，那么读取到的值就是未定义的
* 可变迭代器：如果一个迭代器既有输出迭代器的特性，又有其他迭代器的特性，那么这个迭代器就是可变迭代器。该迭代器可读可写。如果我们从一个非常量容器的实例中获取一个迭代器，那么这个迭代器通常都是可变迭代器

# 四、 迭代器

## 26 尽量用iterator代替const_iterator，reverse_iterator和const_reverse_iterator

* insert和erase的一些版本要求iterator，而不能用const_iterator或reverse_iterator
* 从reverse_iterator转换而来的iterator在转换之后可能需要相应的调整
* 不可能把const_iterator隐式转换成iterator
```cpp
typedef deque<int> IntDeque;
typedef IntDeque::iterator Iter;
typedef IntDeque::const_iterator ConstIter;
Iter i;
ConstIter ci;
if (i == ci) ... // iterator隐式转换成const_iterator再比较
// 而有时编译器把operator==作为const_iterator的一个成员函数
// 上面的代码就无法编译，解决方法是交换两个迭代器位置
if (ci == i)...
```
* 只要混用iterator和const_iterator，上面的问题就可能出现，避免这类问题的最简单的方法是减少混用不同类型的迭代器的机会
```cpp
if (i - ci >= 3) ...
// 如果无法编译则需要把iterator转为const_iterator
if (static_cast<ConstIter>(i) - ci >= 3) ...
// 最简单的做法就是不要混用iterator和const_iterator
```

## 27 用distance和advance把const_iterator转化成iterator
* 如果只有一个const_iterator，而要在它指向的容器位置上插入新元素，需要把const_iterator转化为iterator。并不存在从const_iterator到iterator之间的隐式转换，用const_cast也不行，因为iterator和const_iterator是完全不同的类
* 下面是解决思路
```cpp
typedef deque<int> IntDeque;
typedef IntDeque::iterator Iter;
typedef IntDeque::const_iterator ConstIter;
IntDeque d;
ConstIter ci;
... // 让ci指向d
Iter i(d.begin()); // 初始化i为d.begin()
advance(i, distance(i, ci)); // 把i移到指向ci位置
```
* 但这段代码还不能编译。先看distance的定义
```cpp
template<typename InputIterator>
typename iterator_traits<InputIterator>::difference_type
distance(InputIterator first, InputIterator last);
```
* 可见传递给distance的两个参数类型必须相同，因此要顺利地调用distance，显式指定类型即可
```cpp
advance(i, distance<ConstIter>(i, ci));
```
* 对于随机访问的迭代器（比如vector、string和deque的）而言，这是常数时间的操作。对于双向迭代器（list、set、multiset、map、multimap）而言，这是线性时间的操作

## 28 了解如何通过reverse_iterator的base得到iterator
* reverse_iterator::base指向反向迭代器的base iterator，即当前所在位置往右一个位置
```cpp
vector<int> v;
v.reserve(5);
for(int i = 0；i < 5; ++ i) v.push_back(i);
vector<int>::reverse_iterator ri = find(v.rbegin(), v.rend(), 3);
vector<int>::iterator i(ri.base()); // 使i和ri的base一样
```

![](Effective-STL.assets/4-2.png)

* 有些容器的成员函数只接受iterator类型参数，比如要在ri所指的位置插入元素时，vector的insert函数会拒绝reverse_iterator，删除同理，因此必须先通过base函数将reverse_iterator转换成iterator
* 比如要在ri的位置插入99，因为是从右往左遍历，即倒数第三个位置，插入后内容如下

![](Effective-STL.assets/4-3.png)

* ri指向3时，ri.base()指向4，因此直接插入到ri.base()即可
* 如果要删除ri，则删除的是ri.base()的往左一个元素
```cpp
vector<int> v;
v.reserve(5);
for(int i = 0；i < 5; ++ i) v.push_back(i);
vecot<int>::reverse_iterator ri = find(v.rbegin(), v.rend(), 3);
v.erase(--ri.base());
```
* 但上述代码对于大多数vector和string的实现，无法通过编译，因为这样的实现下，iterator和const_iterator会采用内建指针实现，所以ri.base()的结果是一个指针。为了避免修改base的返回值，只需要先增加reverse_iterator的值，然后再调用base
```cpp
v.erase((++ri).base());
```

## 29 需要一个一个字符输入时考虑使用istreambuf_iterator
* 把一个文本文件拷贝到一个字符串对象
```cpp
ifstream inputFile("interestingData.txt");
string fileData((istream_iterator<char>(inputFile)),  istream_iterator<char>());
```
* 但这样无法把文件中的空格拷贝到字符串中，因为istream_iterators使用operator>>函数读取，默认情况下忽略空格，想保留空格，覆盖默认情况，清除输入流的skipws标志即可
```cpp
ifstream inputFile("interestingData.txt");
inputFile.unset(ios::skipws); // 关闭inputFile的忽略空格标志
string fileData((istream_iterator<char>(inputFile)), istream_iterator<char>());
```
* 现在所有字符都能拷贝了，但拷贝速度可能不快。istream_iterators依靠的operator>>函数进行的是格式化输入，每次调用时要做大量工作。它们必须建立和销毁岗哨（sentry）对象（为每个operator>>调用进行建立和清除活动的特殊的iostream对象），检查可能影响行为的流标志（比如skipws），进行全面的读取错误检查，如果遇到问题必须检查流的异常掩码决定是否抛出异常。如果进行格式化输入，这些活动很重要，但只是从输入流中抓取下一个字符就过度了
* 更高效的方法是使用istreambuf_iterators。istreambuf_iterator<char> 对象从一个istream s中读取会调用s.rdbuf()->sgetc()读s的下一个字符。istreambuf_iterator不忽略任何字符，它们只抓取流缓冲区的下一个字符，因此不需要unset skipws标志符
```cpp
ifstream inputFile("interestingData.txt");
string fileData((istreambuf_iterator<char>(inputFile)), istreambuf_iterator<char>());
```
# 五、算法

## 30 确保目标区间足够大

* STL容器在被添加时（通过insert、push_front、push_back等）自动扩展来容纳新对象，许多人便认为不必为容器中的对象腾出空间担心
```cpp
int f(int x); // 此函数从x产生一些新值
vector<int> values;
... // 把数据放入values
vector<int> results;
transform(values.begin(), values.end(), results.end(), f);
```
* 上述代码存在bug，transform的目的区间从results.end()开始，而*results.end()没有对象，给不存在的对象赋值是错误的，犯这种错误的程序员几乎总是以为他们调用算法的结果能插入目标容器。这里把transform的结果放入results容器的结尾的方法是调用back_inserter来产生指定目标区间起点的迭代器
```cpp
vector<int> results;
transform(values.begin(), values.end(), back_inserter(results), f);
```
* 在内部，back_inserter返回的迭代器会调用push_back，所以可以在任何提供push_back的容器上使用back_inserter（即标准序列容器：vector、string、deque和list）。如果想让一个算法在容器的前端插入，可以使用front_inserter。在内部，front_inserter利用了push_front，所以front_insert只能和有push_front的容器配合（也就是deque和list）
```cpp
list<int> results; // results现在是list
// 在results前端反序插入
transform(values.begin(), values.end(), front_inserter(results), f);
```
* 如果要transform把输出结果放在前端，且输出和values中的顺序相同，以相反的顺序迭代values即可
```cpp
list<int> results;
transform(values.rbegin(), values.rend(), front_inserter(results), f);
```
* inserter允许算法把结果插入容器中的任意位置
```cpp
vector<int> values;
...
vector<int> results;
...
// 插入到results中间
transform(values.begin(), values.end(), inserter(results, results.begin() + results.size()/2), f);
```
* 插入vector或string时，可以预先调用reserve，虽然仍要承受每次发生插入时移动元素的开销，但至少避免了重新分配容器的内在内存
```cpp
vector<int> values;
vector<int> results;
...
results.reserve(results.size() + values.size()); // 保证results至少还能装得下values.size()个元素
transform(values.begin(), values.end(), inserter(results, results.begin() + results.size() / 2), f);
```
* reserve只增加容器的容量，大小未变。忽视这点容易再次犯最初的错误
```cpp
vector<int> values;
vector<int> results;
...
results.reserve(results.size() + values.size());
transform(values.begin(), values.end(), results.end(), f);
```
* 正确的做法仍是使用插入迭代器
```cpp
vector<int> values;
vector<int> results;
results.reserve(results.size() + values.size());
transform(values.begin(), values.end(), back_inserter(results), f);
```
* 有时要覆盖现有容器元素而不是插入新的，此时不需要插入迭代器，但仍要确保目的区间足够大
```cpp
vector<int> values;
vector<int> results;
...
if (results.size() < values.size()) results.resize(values.size());
transform(values.begin(), values.end(), results.begin(), f);
```
* 或者清空results再用通常方式使用插入迭代器
```cpp
...
results.clear();
results.reserve(values.size());
transform(values.begin(), values.end(), pack_inserter(results), transmogrify);
```

## 31 了解你的排序选择
* 很多程序员排序对象时，只会想到sort算法，但有时候不需要完全排序。如果有一个Widget的vector，要选出20个质量最高的Widget，只要排序以鉴别出20个最好的Widget，剩下的可以保持无序，这样的部分排序可以使用partial_sort算法
```cpp
bool qualityCompare(const Widget& lhs, const Widget& rhs)
{
    // 返回lhs的质量是不是比rhs的质量好
}
// 把最好的20个元素按顺序放在widgets的前端
partial_sort(widgets.begin(), widgets.begin() + 20, widgets.end(), qualityCompare);
```
* partial_sort后，widgets的前20个元素是容器中最好的且按顺序排列，质量最高的是widgets[0]，但如果只要选出前20个最好的而不需要排序，使用nth_element算法
```cpp
nth_element(widgets.begin(), widgets.begin() + 19, widgets.end(), qualityCompare);
```
* 当有元素有同样质量的时候，比如有12个质量最好，15个质量其次，此时选择20个的做法是选12个最好的和8个其次的，而从15个中选8个的做法是随机的，partial_sort、nth_element、sort都是不稳定的。排序时使用stable_sort可以稳定排序，但STL不包含partial_sort和nth_element的稳定版本
* nth_element除了能找到区间顶部的n个元素，还可以用于找到区间的中值或者找到在指定百分点的元素
```cpp
vector<Widget>::iterator begin(widgets.begin());
vector<Widget>::iterator end(widgets.end()); 
vector<Widget>::iterator goalPosition;
goalPosition = begin + widgets.size() / 2;
nth_element(begin, goalPosition, end, qualityCompare);
... // goalPosition现在指向中等质量等级的Widget
// 下面的代码能找到质量等级为75%的Widget
vector<Widget>::size_type goalOffset = 0.25 * widgets.size();
nth_element(begin, begin + goalOffset, end,
qualityCompare);
... // begin + goalOffset现在指向质量等级为75%的Widget
```
* 如果不需要鉴别出20个质量最高的Widget，而是鉴别出所有质量等级为1或2的，使用partition算法，它重排区间中的元素使所有满足条件的元素都在区间的开头
```cpp
bool hasAcceptableQuality(const Widget& w)
{
    // 返回w质量等级是否是2或更高;
}
// 把所有满足hasAcceptableQuality的widgets移动到widgets前端
// 并且返回一个指向第一个不满足的widget的迭代器
vector<Widget>::iterator goodEnd = 
    partition(widgets.begin(), widgets.end(), hasAcceptableQuality);
```
* sort、stable_sort、partial_sort和nth_element需要随机访问迭代器，所以它们只能用于vector、string、deque和数组，对关联容器没有意义，唯一我们可能会但不能使用这些算法的容器是list，list提供sort成员函数作为补偿（且list::sort提供了稳定排序），而如果你想要对list中的对象进行partial_sort或nth_element，必须间接完成，一个方法是把元素拷贝到一个支持随机访问迭代器的容器中再对其用算法，另一个方法是建立一个list::iterator容器，对容器使用算法，然后通过迭代器访问list元素，第三种方法是使用有序的迭代器容器的信息来迭代地把list的元素接合到想让它们所处的位置
* partition和stable_partition与sort、stable_sort、partial_sort和nth_element不同，它们只需要双向迭代器，因此可以在任何标准序列迭代器上使用partition和stable_partition

## 32 如果你真的想删除东西的话就在类似remove的算法后接上erase
* remove接收一对迭代器而不接收一个容器，所以remove不知道它作用于哪个容器，从容器中删除元素的唯一方法是调用容器的一个成员函数，几乎都是erase的某个形式（list有几个除去元素的成员函数不叫erase） ，从一个容器中remove元素不会改变容器中元素的个数
```cpp
vector<int> v;
v.reserve(10);
for (int i = 1; i <= 10; ++i) v.push_back(i);
cout << v.size(); // 打印10
v[3] = v[5] = v[9] = 99;
remove(v.begin(), v.end(), 99); // 删除所有等于99的元素
cout << v.size(); // 仍然是10
```
* remove并不真的删除东西，remove不知道它要从哪个容器删除东西，而没有容器就没有办法调用成员函数
* remove移动区间中的元素直到所有“不删除的”元素在区间的开头，返回指向最后一个“不删除的”元素的下一个位置的迭代器，即区间的“新逻辑终点”

![调用remove前](Effective-STL.assets/5-1.png)

![调用后返回newEnd](Effective-STL.assets/5-2.png)

* remove并没有改变区间中元素的顺序，不会把“删除的”元素放在结尾

![新逻辑终点后的元素保持原值](Effective-STL.assets/5-3.png)

* 实际上可以remove完成了一种压缩，对于上述vector，remove过程如下
  * 检查v[0]，发现它的值不是要被删除的，继续后移
  * 直到发现应该被删除的v[3]，便认为v[3]是一个洞，继续移到v[4]
  * 发现v[4]不用删除，就把v[4]赋给v[3]填充v[3]的洞，然后把v[4]看作一个洞
  * 发现v[5]应该被删除，忽略它并移动到v[6]，仍然记得v[4]是一个等待填充的洞
  * 发现v[6]不用删除，所以把v[6]赋给v[4]，记得v[5]现在是下一个要被填充的洞，后续同理

![移动过程](Effective-STL.assets/5-4.png)

* erase的目标很容易看出来，就是新逻辑终点到原来的区间终点之间的部分
```cpp
v.erase(remove(v.begin(), v.end(), 99), v.end()); // 真正删除所有等于99的元素
cout << v.size(); // 打印7
```
* 把remove的返回值作为erase区间形式第一个实参传递是个惯用法。remove和erase整合到了list成员函数remove中，这是STL中唯一名叫remove又能从容器中除去元素的函数
```cpp
list<int> li;
...
li.remove(99); // 删除所有等于99的元素
```

## 33 提防在指针的容器上使用类似remove的算法
* 管理一堆动态分配的Widgets，每一个都可能通过检验，把结果指针保存在一个vector中
```cpp
class Widget{
public:
    ...
    bool isCertified() const; // 这个Widget是否通过检验
    ...
};
vector<Widget*> v;
...
v.push_back(new Widget);
```
* 现在要除去未通过检验的Widget
```cpp
v.erase(remove_if(v.begin(), v.end(), not1(mem_fun(&Widget::isCertified))), 
    v.end());
```
* 这样会产生内存泄漏，下列过程实际上没有释放BC

![调用remove_if前](Effective-STL.assets/5-5.png)

![调用remove_if后](Effective-STL.assets/5-6.png)

![erase后明显的内存泄漏](Effective-STL.assets/5-7.png)

* 如果无法避免在指针的容器上使用remove，一种方法是在应用erase-remove惯用法之前先删除指针并设为空，然后除去容器中的所有空指针
```cpp
void delAndNullifyUncertified(Widget*& pWidget)
{
    if (!pWidget->isCertified())
    {
        delete pWidget;
        pWidget = 0;
    }
}
for_each(v.begin(), v.end(), delAndNullifyUncertified);
v.erase(remove(v.begin(), v.end(), static_cast<Widget*>(0)), v.end());
```
* 如果把指针的容器替换成智能指针的容器，可以直接使用erase-remove惯用法
```cpp
typedef shared_ptr<Widget> RCSPW; // RCSP to Widget
vector<RCSPW > v;
...
v.push_back(RCSPW(new Widget));
...
v.erase(remove_if(v.begin(), v.end(), not1 (mem_fun(&Widget::isCertified))), v.end());
```

## 34 注意哪个算法需要有序区间
* 只能操作有序数据的算法
```
// 下列使用二分查找
binary_search
lower_bound
upper_bound
equal_range
// 下列提供了线性时间复杂度
set_union
set_intersection
set_difference
set_symmetric_difference
// 归并排序
merge
inplace_merge
// 检测是否一个区间的所有对象也在另一个区间
includes
```
* unique和unique_copy虽然不要求，但一般用于有序区间，功能是从每个相等元素的**连续组**中去除第一个以外所有的元素，去除方式和remove类似

## 35 通过mismatch或lexicographical比较实现简单的忽略大小写字符串比较
* 首先需要一种方法来确定两个字符除了大小写之外是否相等
```cpp
int ciCharCompare(char c1, char c2)
{
    int Ic1 = tolower(static_cast<unsigned char>(c1));
    int Ic2 = tolower(static_cast<unsigned char>(c2));
    if (Ic1 < Ic2) return -1;
    if (lc1 > Ic2) return 1;
    return 0;
}
```
* 调用mismatch前必须先满足它的前提，要确定一个字符串是否比另一个短，短的字符串作为第一个区间传递
```cpp
int ciStringCompareImpl(const string& s1, const string& s2);
int ciStringCompare(const string& s1, const string& s2)
{
    if (s1.size() <= s2.size()) return ciStringCompareImpl(s1, s2);
    else return -ciStringCompareImpl(s2, s1);
}
```
* 在ciStringCompareImpl中，大部分工作由mismatch来完成。它返回一对迭代器，表示了区间中第一个对应的字符不相同的位置
```cpp
int ciStringCompareImpl(const string& si, const strings s2)
{
    typedef pair<string::const_iterator, string::const_iterator> PSCI; // pair of string::const_iterator
    PSCI p = mismatch(s1.begin(), s1.end(), s2.begin(), not2(ptr_fun(ciCharCompare))); 
    if (p.first== s1.end())
    {
        if (p.second == s2.end()) return 0;
        else return -1;
    }
    return ciCharCompare(*p.first, *p.second);
}
```
* 唯一可能感到奇怪的是传给mismatch的判断式not2(ptr_fun(ciCharCompare))，当字符匹配时这个判断式返回true，因为当判断式返回false时mismatch会停止。如果把ciCharCompare作为判断式传给mismatch，字符串匹配时返回0，转换为bool是false
* 第二个方法是把ciCharCompare修改为一个有判断式接口的字符比较函数，然后把进行字符串比较的工作交给lexicographical_compare
```cpp
bool ciCharLess(char c1, char c2) // 函数对象
{
    tolower(static_cast<unsigned char>(c1)) < 
        tolower(static_cast<unsigned char>(c2));
}
bool ciStringCompare(const string& s1, const string& s2)
{
    return lexicographical_compare(s1.begin(), s1.end(),
        s2.begin(), s2.end(), ciCharLess);
}
```
* lexicographical_compare是strcmp的泛型版本，strcmp只对字符数组起作用，但lexicographical_compare对任何类型的值的区间都起作用

## 36 了解copy_if的正确实现
* STL有11个名字带copy的算法，但没有copy_if
```cpp
copy
copy_backward
replace_copy
reverse_copy
replace_copy_if
unique_copy
remove_copy
rotate_copy
remove_copy_if
partial_sort_copy
unintialized_copy
```
* 如果只是简单地拷贝一个区间中满足某个判断式的元素，只能自己实现
```cpp
// 一个copy_if的正确实现
template<typename InputIterator, typename OutputIterator, typename Predicate>
OutputIterator copy_if(InputIterator begin, InputIterator end,
    OutputIterator destBegin, Predicate p)
{
    while (begin != end)
    {
        if (p(*begin))*destBegin++ = *begin;
        ++begin;
    }
    return destBegin;
}
```

## 37 用accumulate或for_each来统计区间
* 有时需要用一些自定义的方式统计区间，比如对一个容器中的字符串长度求和，求数的区间的乘积，求point区间的平均坐标，STL提供了accumulate算法来实现，不同于大部分算法，它不存在于<algorithm>而是和其他三个数值算法（inner_product、adjacent_difference和partial_sum）存在于<numeric>中
* accumulate存在两种形式，其中一种是带有一对迭代器和初始值的形式，可以返回初始值加上迭代器划分的区间的·值的和
```cpp
list<double> ld;
...
double sum = accumulate(ld.begin(), Id.end(), 0.0); // 从0.0开始计算它们的和
```
* 注意初始值指定为0.0而不是0，0.0的类型是double，所以accumulate内部使用了一个double类型的变量来存储计算的和，如果写0则可能得到错误的结果，因为每次计算都会把元素值转为int
```cpp
double sum = accumulate(ld.begin(), Id.end(), 0); // 结果不是double
```
* accumulate只需要输入迭代器，所以可以使用istream_iterator和istreambuf_iterator
```cpp
cout << "The sum of the ints on the standard input is" // 打印cin中int的和
<< accumulate(istream_iterator<int>(cin), istream_iterator<int>(), 0);
```
* accumulate的另一种形式是带有一个初始值和一个统计函数，比如计算容器中的字符串的长度和
```cpp
string::size_type stringLengthSum(string::size_type sumSoFar, const string& s)
{
    return sumSoFar + s.size();
}
set<string> ss;
...
string::size_type lengthSum = accumulate(ss.begin(), ss.end(), 0, stringLengthSum); 
```
* 计算数值区间的积更简单，不用写求和函数，直接使用标准multiplies仿函数类
```cpp
vector<float> vf;
...
// 因为是乘法，记得把1.0而不是0.0作为初始统计值
float product = accumulate(vf.begin(), vf.end(), 1.0f, multiplies<float>());
```
* 求point的区间的平均值
```cpp
struct Point {
    Point(double initX, double initY): x(initX), y(initY) {}
    double x, y;
};
list<Point> lp;
...
// 求和函数应该是一个叫做PointAverage的仿函数类的对象
Point avg = accumulate(lp.begin(), lp.end(), Point(0, 0), PointAverage());

class PointAverage : public binary_function<Point, Point, Point> {
public:
    PointAverage(): numPoints(0), xSum(0), ySum(0) {}
    const Point operator()(const Point& avgSoFar, const Point& p) {
        ++numPoints;
        xSum += p.x;
        ySum += p.y;
        return Point(xSum/numPoints, ySum/numPoints);
    }
private:
    size_t numPoints;
    double xSum;
    double ySum;
};
```
* PointAverage存在的问题是，成员变量numPoints、xSum和ySum的修改造成了一个副作用，这是accumulate禁止的，所以上述代码会导致结果未定义。很难理解，但标准就是这么规定的。为了避免这个问题，使用for_each而不是accumulate，for_each的函数参数允许有副作用。至于为什么accumulate和for_each之间有差别，作者也不理解
```cpp
struct Point {
    Point(double initX, double initY): x(initX), y(initY) {}
    double x, y;
};
class PointAverage : public unary_function<Point, void> {
public:
    PointAverage(): xSum(0), ySum(0), numPoints(0) {}
    void operator()(const Point& p)
    {
        ++numPoints;
        xSum += p.x;
        ySum += p.y;
    }
    Point result() const
    {
        return Point(xSum/numPoints, ySum/numPoints);
    }
private:
    size_t numPoints;
    double xSum;
    double ySum;
};
list<Point> Ip;
...
Point avg = for_each(lp.begin(), lp.end(), PointAverage()).result;
```
# 六、 仿函数、仿函数类、函数等

## 38 把仿函数类设计为用于值传递

* STL函数对象在函数指针之后成型，因此STL习惯传给函数和从函数返回时，函数对象是值传递的，比如for_each算法通过值传递获取和返回函数对象
```cpp
// for_each声明
template<class InputIterator, class Function>
Function for_each(InputIterator first, InputIterator last, Function f);
```
* 值传递的情况并不是完全打不破的，for_each可以显式指定参数类型，下面的代码使for_each通过引用传递和返回它的仿函数
```cpp
class DoSomething : public unary_function<int, void> {
public:
    void operator()(int x) {...}
    ...
};
typedef deque<int>::iterator DequeIntIter;
deque<int> di;
...
DoSomething d; // 建立一个函数对象
...
for_each<DequeIntIter, DoSomething&>(di.begin(), di.end(), d);
```
* 但STL的用户不能这样做，如果函数对象是引用传递，有些STL算法不能编译
* 函数对象以值传递和返回，因此需要确保两点，一是函数对象应该很小避免开销，二是函数对象不能用虚函数
* 但不是所有的仿函数都是小的、单态的。函数对象比函数优越的原因之一是仿函数可以包含需要的所有状态。为了实现多态，把虚函数放到另一个类中，给仿函数一个指向这个类的指针。比如希望实现下面这个包含多态的仿函数（这个实现是不对的，因为仿函数不能包含虚函数）
```cpp
template<typename T> // Big Polymorphic Functor Class
class BPFC : public unary_function<T, void> {
private:
    Widget w; // 这个类包含很多数据
    Int x;
```
public:
    virtual void operator()(const T& val) const;
    ...
};
```
* 正确的做法是建立一个包含一个指向实现类的指针的小而单态的类，然后把所有数据和虚函数放到实现类。这个做法在许多地方提及过，参考Effective C++ item34、Exceptional C++中的pimpl、设计模式中的Bridge模式
​```cpp
template<typename T>
class BPFCImpl : public unary_function<T, void> {
private:
    Widget w; // 以前在BPFC里的所有数据现在在这里
    int x;
    ...
    virtual ~BPFCImpl();
    virtual void operator()(const T& val) const;
    friend class BPFC<T>; // 让BPFC可以访问这些数据
};
template<typename T>
class BPFC : public unary_function<T, void> {
private:
    BPFCImpl<T> *pImpl; // BPFC唯一的数据
public:
    void operator()(const T& val) const
    {
        pImpl->operator() (val);
    }
    ...
};
```

## 39 用纯函数做判断式
* 返回一个bool的就叫判断式
* 纯函数是返回值只依赖于参数的函数，比如纯函数f(x, y)的返回值仅当x或y的值改变的时候才会改变
* 仿函数类的operator()函数是一个判断式
* 为了明白判断式函数必须是纯函数的原因，参考下面的判断式类，第三次调用时返回true，其他时候返回false，用这个类从一个vector<Widget>中除去第三个Widget
```cpp
class BadPredicate : public unary_function<Widget, bool> {
public:
    BadPredicate(): timesCalled(0) {}
    bool operator()(const Widget&)
    {  
        return ++timesCalled == 3;
    }
private:
    size_t timesCalled;
};

vector<Widget> vw;
...
vw.erase(remove_if(vw.begin(), vw.end(), BadPredicate()), vw.end());
```
* 看似合理，但它不仅会除去第三个元素，还会除去第六个，参考remove_if的实现
```cpp
template <typename FwdIterator, typename Predicate>
FwdIterator remove_if(FwdIterator begin, FwdIterator end, Predicate p)
{
    begin = find_if(begin, end, p);
    if (begin == end) return begin;
    else
    {
        FwdIterator next = begin;
        return remove_copy_if(++next, end, begin, p);
    }
}
```
* 判断式p先传给find_if，后传给remove_copy_if。find_if先接收了一个timesCalled为0的BadPredicate对象，find_if调用三次后对象返回true，然后才调用remove_copy_if，传p的另一个拷贝作为判断式，p的timesCalled成员仍为0。结果第三次remove_copy_if调用判断式也会返回true，因此remove_if最终会删除两个Widgets
* 避免碰到这种问题的方法是把判断式类的operator()函数声明为const，防止改变状态
```cpp
class BadPredicate : public unary_function<Widget, bool> {
public:
    bool operator()(const Widget&) const
    {
        return ++timesCalled == 3; // 错误，不能改变局部数据
    }
};
```
* 把operator()声明为const是必要的，但还不够充分。行为良好的operator()当然是const，但它也得是一个纯函数，纯函数没有状态
```cpp
bool anotherBadPredicate(const Widget&, const Widget&)
{
    static int timesCalled = 0; // 错误，纯函数不能有状态
    return ++timesCalled == 3;
}
```

## 40 使仿函数类可适配
* 在list中找到第一个符合条件的元素很简单
```cpp
list<Widget*> widgetPtrs;
bool f(const Widget *pw);
list<Widget*>::iterator i = find_if(widgetPtrs.begin(), widgetPtrs.end(), f);
if (i != widgetPtrs.end()) { ... }
```
* 但如果要找第一个不符合条件的
```cpp
list<Widget*>::iterator i =
    find_if(widgetPtrs.begin(), widgetPtrs.end(), not1(f));
```
* 这样做会编译失败，因为not1不能直接对函数使用，所以要先用ptr_fun把函数转为函数对象
```cpp
list<Widget*>::iterator i =
    find_if(widgetPtrs.begin(), widgetPtrs.end(), not1(ptr_fun(f)));
```
* ptr_fun做的唯一的事是使一些typedef有效，四个函数适配器not1、not2、bind1st和bind2nd都需要这些typedef。这些typedef是argument_type、first_argument_type、second_argument_type和result_type
* std::unary_function和std::binary_function提供了这些typedef，用于给仿函数类继承。仿函数类的operator()带一个实参则继承std::unary_function，带两个实参则继承是std::binary_function
* unary_function和binary_function是模板，所以不能直接继承，必须从它们产生的类继承，而这需要指定实参类型。对于unary_function必须指定operator()的参数类型和返回类型，对于binary_function要指定operator()的两个参数类型和返回类型
```cpp
template<typename T>
class MeetsThreshold : public std::unary_function<Widget, bool> {
private:
    const T threshold;
public:
    MeetsThreshold(const T& threshold);
    bool operator() (const Widget&) const;
...
};
struct WidgetNameCompare : public std::binary_function<Widget, Widget, bool> {
    bool operator()(const Widget& lhs, const Widget& rhs) const;
};
```
* 一般unary_function或binary_function的非指针类型都去掉了const和引用， 因此后一个operator()的实参类型传给binary_function的是Widget。现在可以直接对上面的仿函数类使用函数适配器
```cpp
list<Widget> widgets;
...
list<Widget>::reverse_iterator i1 = 
    find_if(widgets.rbegin(), widgets.rend(), not1(MeetsThreshold<int>(10)));
Widget w(构造函数实参);
list<Widget>::iterator i2 = 
    find_if(widgets.begin(), widgets.end(), bind2nd(WidgetNameCompare(), w);
```
* 如果仿函数类没有继承unary_function或binary_function，上述代码就不能编译，因为not1和bind2nd都只和可适配的函数对象合作

## 41 了解使用ptr_fun、mem_fun和mem_fun_ref的原因
* 如果有一个函数f和一个对象x，要位于x的成员函数之外在x上调用f，有三种不同的语法实现这个调用
```cpp
f(x); // 语法#1：f是一个非成员函数
x.f(); // 语法#2：f是一个成员函数且x是一个对象或对象的引用
p->f(); // 语法#3：当f是一个成员函数且p是一个对象的指针
```
* 假设有一个测试Widget的函数和一个Widget的容器，用for_each测试容器中的每个元素
```cpp
void test(Widget& w);
vector<Widget> vw;
for_each(vw.begin(), vw.end(), test); // 调用#1（可以编译）
```
* 但如果test是Widget的成员函数，编译将失败
```cpp
class Widget {
public:
    void test(); // 失败则把*this标记为failed
};
for_each(vw.begin(), vw.end(), &Widget::test); // 调用#2（不能编译）
// 对指针的容器测试
list<Widget*> lpw;
for_each(lpw.begin(), lpw.end(), &Widget::test); // 调用#3（不能编译）
```
* 原因是for_each用的是语法#1，函数和函数对象使用非成员函数的语法形式调用是STL里的一个习惯
```cpp
template<typename InputIterator, typename Function>
Function for_each(InputIterator begin, InputIterator end, Function f)
{
    while (begin != end) f(*begin++);
}
```
* mem_fun和mem_fun_ref的作用就是让成员函数使用句法1调用
```cpp
// 用于不带参数的non-const成员函数的mem_fun声明
template<typename R, typename C> // C是类，R是成员函数的返回类型
mem_fun_t<R,C> mem_fun(R(C::*pmf)());
```
* 返回的mem_fun_t类型的对象是一个仿函数类，容纳成员函数指针pmf，提供一个调用成员函数的operator()
```cpp
for_each(lpw.begin(), lpw.end(), mem_fun(&Widget::test)); // 可以编译
```
* mem_fun适配语法#3到语法#1，mem_fun_ref函数适配语法#2到语法#1，只要传一个成员函数给STL组件就必须使用它们
* for_each没有使用ptr_fun增加的typedef，所以不必使用ptr_fun，当然加了也不会有变化。如果不清楚什么时候使用ptr_fun，就在每次传递函数给STL组件时都用上，除了降低可读性并不会有其他影响
```cpp
for_each(vw.begin(), vw.end(), ptr_fun(test)); // 同调用#1
```

## 42 确定less<T>表示operator<
* 如果一个类A有x和y两个成员，用于A的operator<按x排序
```cpp
class A {
public:
    ...
    int x() const;
    int y() const;
    ...
};
bool operator<(const A& lhs, const A& rhs)
{
    return lhs.x() < rhs.x();
}
```
* 现在想建立一个按y排序的multiset<A>，而multiset的默认比较函数是less<A>，默认的less<A>通过调用A的operator<来工作，于是明显想到特化less<A>使其按y排序
```cpp
template<>
struct std::less<A> : public std::binary_function<A, A, bool> {
    bool operator() (const A& lhs, const A& rhs) const
    {
        return lhs.y() < rhs.y();
    }
}
```
* 这种特化并不罕见，比如智能指针想让排序行为表现得像内建指针
```cpp
namespace std {
    template<typename T>
    struct less<shared_ptr<T>>
    : public binary function<boost::shared_ptr<T>, shared_ptr<T> bool> {
        bool operator()(const boost::shared_ptr<T>& a, const boost::shared_ptr<T>& b) const
        {
            return less<T*>()(a.get(),b.get());
        }
    };
}
```
* 但operator<是实现less的默认方式，也是我们希望less做的，没有理由时不应该让less做其他事，回到最初的例子，如果希望按y排序，应当建立一个不叫less的仿函数类用于比较
```cpp
struct yCompare : public binary_function<A, A, bool> {
    bool operator()(const A& lhs, const A& rhs) const
    {
        return lhs.y() < rhs.y();
    }
};
 multiset<A, yCompare> a;
```
# 七、使用STL编程

## 43 尽量用算法调用代替手写循环

* 很多要用循环来实现的任务可以改用算法来实现，算法内部也包含一个循环
* 使用算法有三个原因：高效、正确、可维护（直观易读）

## 44 尽量用成员函数代替同名的算法
* 有些容器拥有和STL算法同名的成员函数，关联容器提供了count、find、lower_bound、upper_bound和
equal_range，list提供了remove、remove_if、unique、sort、merge和reverse
* 尽量用成员函数代替算法，一是成员函数更快，二是比器算法它们与容器结合得更好（尤其是关联容器）
```cpp
set<int> s; // 建立set，放入1,000,000个数据
...
set<int>::iterator i = s.find(727); // 使用find成员函数
if (i != s.end()) ...
set<int>::iterator i = find(s.begin(), s.end(), 727); // 使用find算法
if (i != s.end()) ...
```
* find成员函数花费对数时间，不管727是否存在于此set中，set::find查找不超过40次比较，平均大约20次。find算法运行花费线性时间，如果727不在set中则需要执行1,000,000次比较，在此set中平均需要执行500,000次比较
* 算法判断两个对象相同基于相等，而关联容器基于等价。 因此，find算法搜索727用的是相等，而find成员函数用的是等价。这一差别对map和multimap尤其明显，它们的成员函数只关心key，比如count成员函数只统计key值匹配的元素数，而count算法的寻找基于相等和对象对的全部组成部分
* 每个被list作了特化的算法（remove、remove_if、unique、sort、merge和reverse）都要拷贝对象，而list的特化版本没有拷贝，只是简单地操纵连接list的节点的指针。算法和成员函数的算法复杂度是相同的，但如果操纵指针比拷贝对象的代价小的话，list的版本应该提供更好的性能
* list成员函数的行为和同名算法的行为经常不相同，比如list的remove、remove_if和unique成员函数真正删除了元素，不需要再调用erase。sort算法不能用于list，但list有sort成员函数

## 45 注意count、find、binary_search、lower_bound、upper_bound和equal_range的区别
* 把count用来作为是否存在的检查。count返回零或者一个正数，可以把非零转化为true而把零转化为false
```cpp
list<Widget> lw; // Widget的list
Widget w;
if (count(lw.begin(), lw.end(), w) != 0) ...
```
* 使用find则难懂一些，但效率高，因为find找到匹配目标后就停止了，count会继续搜索
```cpp
if (find(lw.begin(), lw.end(), w) != lw.end()) ... // 找到了
```
* count和find都是线性时间的，而有序区间的搜索算法（binary_search、lower_bound、upper_bound和equal_range）是对数时间的，对于有序区间应当使用后者，另外前者基于相等，后者基于等价
* 要测试有序区间中是否存在一个值，使用binary_search
* lower_bound查找一个值的第一个拷贝，它返回一个迭代器，如果找到，迭代器指向这个值的第一个拷贝，否则返回可以插入这个值的位置。必须测试lower_bound的结果所标示出的对象是不是目标值
```cpp
vector<Widget>::iterator i = lower_bound(vw.begin(), vw.end(), w);
if (i != vw.end() && *i == w) ...
```
* 但lower_bound搜索用的是等价，大多数情况下等价和相等的结果是一样的，但如果想完全完成，必须检测lower_bound指向的对象值是否和目标值等价
* 一个更好的方法是使用equal_range。equal_range返回一对迭代器，第一个等于lower_bound返回的迭代器，第二个等于upper_bound返回的，如果两个迭代器相同就说明没找到。equal_range只用等价，所以总是正确的
```cpp
vector<Widget> vw;
...
sort(vw.begin(), vw.end());
typedef vector<Widget>::iterator VWIter;
typedef pair<VWIter, VWIter> VWIterPair;
VWIterPair p = equal_range(vw.begin(), vw.end(), w);
if (p.first != p.second) ... // 找到了
```
* 对equal_range返回的两个迭代器作distance就等于区间中对象的数目，完成了计数
```cpp
VWIterPair p = equal_range(vw.begin(), vw.end(), w);
cout << "There are " << distance(p.first, p.second)
<< " elements in vw equivalent to w.";
```
* 有时要在区间中寻找一个位置，比如有一个Timestamp类和一个Timestamp的vector，它按照老的timestamp放在前面的方法排序
```cpp
class Timestamp { ... };
bool operator<(const Timestamp& lhs, const Timestamp& rhs);
vector<Timestamp> vt;
...
sort(vt.begin(), vt.end());
```
* 现在有一个特殊的timestamp对象ageLimit，要删除所有比ageLimit老的timestamp，首先要找到第一个不比ageLimit更老的元素的位置，使用lower_bound
```cpp
Timestamp ageLimit;
...
vt.erase(vt.begin(), lower_bound(vt.begin(), vt.end(), ageLimit));
```
* 如果要删除所有至少和ageLimit一样老的timestamp，即要找到第一个比ageLimit年轻的timestamp的位置，使用upper_bound
```cpp
vt.erase(vt.begin(), upper_bound(vt.begin(), vt.end(), ageLimit));
```
* 要把东西插入一个有序区间，对象的插入位置是在有序的等价关系下它应该在的地方时，upper_bound也很有用
```cpp
class Person {
public:
    ...
    const string& name() const;
    ...
};

struct PersonNameLess : public binary_function<Person, Person, bool> {
    bool operator()(const Person& lhs, const Person& rhs) const
    {
        return lhs.name() < rhs.name();
    }
};

list<Person> lp;
...
lp.sort(PersonNameLess());
```
* 要保持list顺序，用upper_bound来指定插入位置
```cpp
Person newPerson;
...
lp.insert(upper_bound(lp.begin(), lp.end(), newPerson, PersonNameLess()), newPerson);
```
* binary_search算法没有提供对应的成员函数，要测试在set或map中是否存在某个值，使用count成员函数
```cpp
set<Widget> s;
...
Widget w;
...
if (s.count(w)) ... // 找到了
```
* 要测试某个值在multiset或multimap中是否存在，find往往比count好，因为count要测试每个对象，但用count给关联容器计数比调用equal_range然后distance更快也更简单清晰

|目标|无序区间算法|有序区间算法|set或map的成员函数|multiset或multimap的成员函数|
|:-:|:-:|:-:|:-:|:-:|
|值是否存在|find|binary_search|count|find|
|值是否存在，第一个等于这个值的对象在哪里|find|equal_range|find|find或lower_bound|
|第一个不在期望值之前的对象在哪里|find_if|lower_bound|lower_bound|lower_bound|
|第一个在期望值之后的对象在哪里|find_if|upper_bound|upper_bound|upper_bound|
|有多少对象等于期望值|count|equal_range再distance count|count|count|
|等于期望值的所有对象在哪里|find（迭代）|equal_range|equal_range|equal_range|

## 46 考虑使用函数对象代替函数作算法的参数
* 把函数对象传递给算法所产生的代码一般比传递函数高效，比如要降序排序一个double的vector，最直接的STL方式是通过sort算法和greater<double>类型的函数对象实现
```cpp
vector<double> v;
...
sort(v.begin(), v.end(), greater<double>());
```
* 可能你会使用内联函数来减少调用开销
```cpp
inline
bool doubleGreater(double d1, double d2)
{
    return dl > d2;
}
...
sort(v.begin(), v.end(), doubleGreater);
```
* 但两者相比，用函数对象的那个更快，因为greater<double>::operator()是一个内联函数，编译器在实例化sort时内联展开，于是sort没有包含一次函数调用，而且编译器可以对这个没有调用操作的代码进行优化
* 而把函数作为参数时，编译器把函数转化为一个指向那个函数的指针，传递的是这个指针，因此第二个sort调用传了一个doubleGreater的指针，sort模板实例化时，声明为
```cpp
void sort(vector<double>::iterator first,
    vector<double>::iterator last, 
    bool (*comp)(double, double));
```
* 因为comp是一个指向函数的指针，每次在sort中用到时，编译器产生一个通过指针的间接函数调用，编译器不会试图去内联通过函数指针调用的函数
* 把函数指针作为参数会抑制内联也解释了C++的sort实际上总比C的qsort快的原因
* 用函数对象代替函数还可以避免细微的语言陷阱
```cpp
// 返回两个浮点数的平均值
template<typename FPType>
FPTypeaverage(FPType val1, FPType val2)
{
    return (val1 + val2) / 2;
}
// 把两个序列的成对的平均值写入一个流
template<typename InputIter1, typename InputIter2>
void writeAverages(InputIter1 begin1, InputIter1 end1, InputIter2 begin2, ostream& s)
{
    transform(begin1, end1, begin2,
        ostream_iterator<typename iterator_traits
            <lnputIter1>::value_type> (s, "\n"),
        average<typename iteraror_traits
            <InputIter1>::value_type>
    );
}
```
* 有可能存在带有一个类型参数的函数模板叫做average，average<typename iterator_traits<lnputIter1>::value_type>将存在二义性，解决方法是建立一个函数对象
```cpp
template<typename FPType>
struct Average : public binary_function<FPType, FPType, FPType> {
FPType operator()(FPType val1. FPType val2) const
{
    return average(val 1 , val2);
}
template<typename InputIter1, typename InputIter2>
void writeAverages(lnputIter1 begin1, InputIter1 end1,
    InputIter2 begin2, ostream& s)
{
    transform(begin1, end1, begin2,
        ostream_iterator<typename iterator_traits
            <lnputIter1>::value_type>(s, "\n"),
        Average<typename iterator_traits
            <InputIter1>::value_type>());
}
```

## 47 避免产生只写代码
* 代码的读比写更经常，所以写代码时需要考虑可读性，比如有一个vector<int>，要删掉最后一个不小于y的元素之后的小于x的元素
```cpp
v.erase(remove_if(
    find_if(v.rbegin(), v.rend(), bind2nd(greater_equal<int>(), y)).base(),
    v.end(),
    bind2nd(less<int>(), x)),
    v.end());
```
* 上述代码比较难以阅读，需要一层层拆分才能看懂
```cpp
find_if(v.rbegin(), v.rend(), bind2nd(greater_equal<int>(), y)
// 从后往前找到不小于y的数。假设找不到满足的元素，返回尾后迭代器v.rend()
v.rend().base()
// 相当于v.begin()
v.erase(std::remove_if(v.begin(), v.end(), std::bind2nd(std::less<int>(), x))
```
* 上述代码一般被称为只写（write-only）代码，很容易写但很难读和理解，代码是否只写依赖于读它的人。要让上述代码更好理解，应该分解成几个块
```cpp
typedef vector<int>::iterator VecIntIter;
VecIntIter rangeBegin = find_if(v.rbegin(), v.rend(),
    bind2nd(greater_equal<int>(), y)).base();
v.erase(remove_if(rangeBegin, v.end(), bind2nd(less<int>(), x)), v.end());
```

## 48 总是#include适当的头文件
* 容器几乎都在同名的头文件中，例外的是<set>声明了set和multiset，<map>声明了map和multimap
* 所有的算法都在<algorithm>和<numeric>中声明
* 特殊的迭代器，包括istream_iterators和istreambuf_iterators，在<iterator>中声明
* 标准仿函数（比如less<T>）和仿函数适配器（如not1、bind2nd）在<functional>中声明

## 49 学习破解有关STL的编译器诊断信息
* 定义一个特定大小的vector是合法的
```cpp
vector<int> v(10);
```
* 于是想到用同样的方法定义string
```cpp
string s(10);
```
* 这是错误的，报错如下
`errorC2664:'std::basic_string<char,std::char_traits<char>,std::allocator<char>>::basic_string(std::initialzer_list<_Elem>,const std::allocator<char> &)' : cannot convert argument 1 from 'int' to 'const std::basic_string<char,std::char_traits<char>,std::allocator<char>> &'`
* string不是一个类，实际上是这个的typedef
```cpp
basic_string<char, char_traits<char>, allocator<char>>
```
* 通常诊断信息会明确指出basic_string在std中
```cpp
std::basic_string<char, std::char_traits<char>, std::allocator<char>>
```
* 如果把上述完整写法用string替换，就很容易识别错误信息了
`errorC2664:'string::string(const class std::allocator<char> &)':' : cannot convert argument 1 from 'int' to 'const std::basic_string<char,std::char_traits<char>,std::allocator<char>> &'`
## 50 让你自己熟悉有关STL的网站
* [cppreference](https://en.cppreference.com/w/%E9%A6%96%E9%A1%B5)
* [cplusplus.com](http://www.cplusplus.com/reference/)
* [STLport](http://www.stlport.org/)
* [Boost](https://www.boost.org/)
