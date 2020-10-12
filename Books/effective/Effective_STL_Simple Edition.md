> 转自：[c437yuyang/EffectiveSTL](https://github.com/c437yuyang/EffectiveSTL)

###  前言:
> 本书讨论的STL是指:标准容器,`iostream`库的一部分，函数对象和算法
> (其他一些书里面通常是指:`algorithm`,`container`,`iterator`三大部分)
>
> - 不包含标准容器适配器(`stack`、`deque`、`priority_queue`，因为无迭代器支持)，数组(属C++语言)
> - 也不包含标准C++库的扩展，比如散列容器、单链表和多种非标准函数对象等。

###  术语:
> - 标准序列容器:`vector`,`string`,`deque`,`list`
> - 标准关联容器:`set`,`multiset`,`map`,`multimap`
> - 迭代器:输入、输出、前向、双向、随机
> - 仿函数类:重载了`operator()`的类  使用仿函数对象的地方大部分可以用真函数替代
> - `bind1st`和`bind2nd`称为绑定器

# 一、容器


###  1:仔细选择容器
>  - **连续内存容器**:vector,string,deque，**插入、删除**操作会使**迭代器失效** （移动导致）
>- **基于Node的容器**:`list`,**插入、删除不会使迭代器失效**

###  2:小心对"容器无关代码"的幻想
>  STL是建立在泛化的基础上的。
>
>  - 数组泛化为容器，参数泛化所包含对象的类型。
>  - 函数泛化为算法，参数泛化所用的迭代器类型。
>  - 指针泛化为迭代器，参数泛化所指向的对象的类型。
>
>  不同容器是不同的，优点和缺点大不相同，不被设计成可互换的，不要去对它们做包装
>
>  - **序列容器**支持`push_front`、`push_back`，但关联容器不支持
>  - **关联容器提供logN**复杂度的`lower_bound`、`upper_bound`和`equal_range`，**（N叉树）**
>  - **尽量用typedef来代替**冗长的`container<class>` 以及`container<class>::iterator`代码,使用typedef的好处还有，**换另一种容器方便**(以及更换`allocator`等其他`template`参数的时候)
>  - 若**不想对用户暴露**所使用容器的类型，则把容器进行封装，把容器类型定义在`private`域，**只提供相应的接口给用户**


###  3:使容器里对象的拷贝操作轻量而且正确
>  - STL里的容器，所有的操作，都是**基于拷贝**的，插入，读取，删除(导致移动)
>  - **分割问题**表明派生类对象拷贝进入基类容器的时候会被砍掉派生类部分
>  - 解决办法是是建立**智能指针的容器**，拷贝指针很快(条款7)

###  4:用empty()代替size()==0
>  事实上empty的典型实现是一个返回size是否返回0的内联函数，对**所有的标准容器**`empty()`总是**常数时间(**因为只检查有没有)`size()`**不一定是常数时间**(可能要遍历所有如list)

###  5:用区间成员函数代替单元素操作

> **所有`insert`函数,都有对应的`emplace`函数。**

>  - 给容器完全新的数据集时，应使用assign()，所有**标准序列容器**都有效
>  - **区间是通过迭代器指定的copy**都可以由**区间成员函数**代替,如insert
>  - **连续内存序列容器**：使用区间版本insert相对于单元素insert的好处:
>
>  > 1. 减少函数调用开销
>  > 2. 单元素版，把区间插入到头部的时候，**反复移动元素**，区间版本计算一次，移动一次
>  > 3. 单元素版本每次insert可能**多次内存分配**，区间版本不会。
>  > 4. 对于list来说，**反复prev和next赋值也是开销**
>  > 5. 代码更少，程序含义更加明确,利于后期的维护
>  >
>
>  - **所有标准容器都支持的区间成员函数**
>
>    > 1. 区间构造 `container::container(begin,end)`
>    > 2. 区间插入 `insert(position,begin,end)`,关联容器:`insert(begin,end)`,用hash函数判断位置
>    > 3. 区间删除 `erase`
>    > 4. 区间赋值 `assign`
>
>  ```cpp
>  //使v1的内容和v2的后半段相同
>  vector<int> vec1 = {1,2,3,4};
>  vector<int> vec2 = { 5,6,7,8,9 };
>  //1.assign
>  vec1.assign(vec2.begin() + vec2.size() / 2, vec2.end());
>  //2.insert
>  vec1.clear();
>  vec1.insert(vec1.end(), vec2.begin() + vec2.size() / 2, vec2.end());
>  //3.copy(效率低，内部存在循环)
>  vec1.clear();
>  copy(vec2.begin() + vec2.size() / 2, vec2.end(), back_inserter(vec1)); 
>  //back_inserter创建一个vec1的插入迭代器，防止vec1本身没有分配内存，本质就是调用push_back
>  //4.手动单成员赋值
>  vec1.clear();
>  for (auto it=vec2.begin()+vec2.size()/2;it!=vec2.end();++it)
>  	vec1.push_back(*it);
>  ```

###  6: C++里的解析方式(函数声明)

> **不要在参数内递临时构建对象**再来传入，而是先构建，再传入。
>
> **C++11，{}初始化可以避免**

>  ```cpp
>  ifstream dataFile("ints.dat");
>  list<int> data(istream_iterator<int>(dataFile), istream_iterator<int>());//函数
>  //list<int> 是类型，声明名为data的函数
>  //不要在参数内递临时构建对象再来传入，而是先构建，再传入
>  ifstream dataFile("ints.dat");
>  istream_iterator<int> dataBegin(dataFile);
>  istream_iterator<int> dataEnd;//不能加括号，否则又是函数声明了
>  list<int> data(dataBegin, dataEnd);
>  ```

###  7: 容器里是指针的时候记得delete
>  当一个指针的容器被销毁时，会销毁每个元素，**但new的对象不会调用delete**。**装`std::shared_ptr`**。
>
>  ```cpp
>  void f(){
>      vector<shared_ptr<A>> v;
>      for (int i = 0; i < 10; ++i) 
>          v.push_back(shared_ptr<A>(new A));
>      ...
>  } // 这里没有泄漏
>  ```

###  8: 容器里不能装auto指针
>  拷贝`auto_ptr`所有权转移，值被置为NULL，所以禁止使用。
>
>  **`std::auto_ptr`已经移除，替代品是`std::unique_ptr`。**
>
>  **容器里装值和智能指针 `shared_ptr<A>`;**

###  9:不同类型容器的delete
>  ```cpp
>  // 连续内存容器（vector、deque、string）最好的删除方法是erase-remove惯用法
>  c.erase(remove(c.begin(), c.end(), 1963), c.end()); 
>  
>  // list的成员函数直接remove更高效
>  c.remove(1963);
>  // 关联容器没有remove，方法是调用erase，只花费对数时间（序列容器为线性时间）
>  c.erase(1963);
>  // 删除bool f(int x)返回值为true的对象
>  c.erase(remove_if(c.begin(), c.end(), f), c.end());//序列容器 迭代器有效
>  c.remove_if(f); // c为list 更高效
>  ```
>
>  - 再增加问题，每次删除后要把一条消息写到日志文件中，**对关联容器很简单，只要加个打印**
>
>
>  ```cpp
>  ofstream logFile;
>  AssocContainer<int> c;
>  for (AssocContainer<int>::iterator it = c.begin(); it !=c.end();) {
>    if (f(*it)) {
>        logFile << "Erasing " << *it <<'\n'; //add log
>        c.erase(it++); // 删除元素,关联容器，使用it++保持迭代器有效
>        it = c.erase(i);//删除元素,序列容器，使用`erase`的返回值保持迭代器有效
>    }
>    else ++it;
>  }
>  ```

### 10 注意分配器的协定和约束

> 如果你想要写自定义分配器，必须
>
> - 把分配器做成一个模板，**模板参数T代表要分配内存的对象类型**。
> - **提供`pointer`和`reference`的`typedef**`，让`pointer`是`T*`，`reference`是`T&`。
> - 不要给你的分配器对象状态，**分配器不能有非静态的数据成员**
> - **传给分配器的`allocate`成员函数需要分配的对象个数**而不是字节数，**函数返回`T*`指针**（通过`pointer` `typedef`），即使还没有T对象被构造
> - **提供标准容器依赖的内嵌`rebind`模板**

###  11:Allocator的使用

###  12:STL容器的线程安全性
> 1.STL里的容器保证:
>
> > (1)多个读取者**同时读取是安全的**(不能有写入这时候)
> > (2)对不同容器同时写是安全的(感觉是废话)
>
> 2.要实现线程安全,需要做到:
>
> > (1)每次调用容器的成员函数期间
> > (2)容器返回迭代器的有效生存期内
> > (3)容器的算法执行期间
>
> **构造函数里获得互斥量并在析构函数里释放**，lock写容器，RAII方式。
>
> **`std::mutex`**

# 二、 vector和string

###  13:尽量使用vector和string来代替动态数组
>  1.new进行分配的话，要确保(**安全性**)
>
>  - (1)有`delete`
>  - (2)`delete`是正确的形式
>  - (3)只`delete`一次(所以通常`delete`接把指针=`nullptr`)
>
>  2.容器还有很多算法
>  3.`vector`和`string`也都是兼容C风格的遗留代码的，因为是**基于数组实现**的
>
>  ```cpp
>  void printArr(const int *p, int size)
>  vector<int> v = { 1,2,3,4,5 };
>  if(!v.empty())
>     printArr(&v[0], v.size()); //因为vector的存储在内存中也是连续的，判断!v.empty
>  //printArr(v.begin(), v.size()); //错误，iterator和指针不一样,&*v.begin()可以，不推荐
>  ```
>
>  4.唯一可能的问题是,`string`**采用了引用计数**,可以用`vector<char>`来代替

###  14:使用reverse来避免不必要的重新分配
>  > resize把size改为n，n<size则尾部元素被销毁，否则重新分配，默认构造的新元素会添加到容器尾部
>  >
>  > reserve把capacity改为n，注意，**reserve只是改动空间大小，而不能直接用下标赋值**
>  >
>  > > 若n<capacity，
>  > >
>  > > - 对vector这个调用什么都不做，
>  > > - 对string把容量减少为size和n中大的数，但size不变
>
>  ```cpp
>  size();// 容器已存元素个数,size大于下标，可以用下标赋值
>  capacity(); //容器不重新分配的情况下，最多可以存元素个数
>  resize(Container::size_type n); //把size 设置成n,小于n则删除元素，大于则构造元素
>  reserve(Container::size_type n);//把capacity的大小改为n，reverse也不能变小
>  if (s.size() < s.capacity()) {
>      s.push_back('x'); // 不会使容器的迭代器失效
>  }
>  ```

### 15:小心string实现的多样性

> string对象大小可能是1到至少7倍char*指针大小，存在差别的原因，须知string可能存的数据和保存的位置
>
> 实际上每个string实现都容纳了下面的信息，不同的string实现以不同的方式把这些信息放在一起
>
> > - 字符串的**大小**
> > - **容纳字符串字符的内存容量**
> > - 这个**字符串的值**，即构成这个字符串的字符。
> > - 一个string可能容纳它的**配置器的拷贝**
> > - 依赖引用计数的string实现包含了这个**值的引用计数**

###  16:vector和string传递给传统的C API
>  1. **vector存储连续**，故**可用`&v[0]`获取首元素指针**，与数组相同操作，建议`if(!v.empty())`避免越界
>
>  ```cpp
>  void doSomething(const int* pints, size_t numInts);
>  set<int> intSet; //想将set的数据当参数给doSomething
>  ...
>  vector<int> v(intSet.begin(), intSet.end());//vector中转下
>  if (!v.empty()) //vector和数组内存分布的兼容
>   doSomething(&v[0], v.size());
>  ```
>
>  2. **迭代器不是指针**，不能用`.begin()`代替`&v[0]`,但是可以`&*v.begin()`，不过`&v[0]`更明确
>
>  3. **`string`不保证连续**，应该使用`.c_str()`,`string`为`empty()`时，`.c_str()`返回`nullptr`，而`string`结尾不保证有`null`
>
>  4. **`vector`传给C风格的指针操作后，可以修改内容，但是不能增加或删除**，因为不会同时改变vecotr其他成员比如`size`和`capacity`
>
>  5. `string`要实现从C风格初始化的话，必须先`vector<char>`,在把迭代器给`string(x.begin(),x.end())`这样来实现事实上这个方法适用于所有的容器
>
>  6. ```cpp
>      size_t fillString(char *pArray, size_t arraySize); //char *pArray => string
>      vector<char> vc(maxNumChars);
>      size_t charsWritten = fillString(&vc[0], vc.size());
>      string s(vc.begin(), vc.begin()+charsWritten);
>      ```
>    ```
>  
>    ```
>
>    ```
>  
>    ```
>
>  ```
>  
>  ```
>
>  ```
>  
>  ```

###  17:使用交换技巧来修正过剩容量

> **C++11加入了`shrink_to_fit`函数**

```cpp
vector<Contestant> v;
string s;
... // 使用v和s
    
vector<Contestant>(contestants).swap(contestants);
    
vector<Contestant>().swap(v); // 清除v并最小化容量
string().swap(s); // 清除s并最小化容量
```

###  18:避免使用vector\<bool>

>  **使用`std::bitset`或`deque<bool>`**
>
>  容器必要条件，c是一个T类型对象的容器，且c支持`operator[]`，则满足`T *p = &c[0]`;
>
>  1.`vector<bool>`不是STL容器，里面装的是bit，不满足于 `bool *p = &v[0]`
>  2.`deque<bool>`是STL容器，它保存真正的`bool`值，但是内存不连续，也不能用C风格
>  3.`bitset`不是STL容器，大小（元素数量）在编译期固定，不支持插入和删除元素，就像`vector<bool>`

# 三、 关联容器

###  19:相等和等价的区别

> **等价**基于**有序区间中**对象值的**相对位置**，通常基于`operator<`，判断等价: `!(a<b)&&!(b<a)`
>
> **相等**基于`operator==`，在类中重载时，即使两个类不完全相同也可以相等
>
> 使用的时**优先使用成员函数的版本**，才能保持统一
> **关联容器之所有使用等价而不是相等**，原因在于**关联容器本身还要做排序**

>  ```cpp
>  set<string, CIStringCompare> ciss; // case-insensitive string set
>  ciss.insert("Persephone"); // 已添加到set中    等价
>  ciss.insert("persephone"); // 未添加到set中    等价
>  //set的find成员函数 true   operator<
>  if (ciss.find("persephone") != ciss.end())... // true
>  //算法的find函数 false     operator==
>  if (find(ciss.begin(), ciss.end(), "persephone") != ciss.end())... // false
>  ```

###  20:为元素是指针的关联容器指定比较类型
>  ```cpp
>  struct DereferenceLess {
>      template <typename PtrType>
>      bool operator()(PtrType pT1, PtrType pT2) const
>      {
>          return *pT1 < *pT2;
>      }
>  };
>  
>  set<string*, DereferenceLess> ssp; // 行为就像set<string*, StringPtrLess>
>  ```

###  21:关联容器（需排序）指定比较器时，相等时返回false

> 除非**比较函数总是为相等的值返回false**，否则将会打破所有的标准关联型容器，
>
> 就是不能用`<=`来进行比较，必须强`＜`才行，`less_equal`意思就是`operator<=`。
>
> **C++17移除了`std::binary_function`**

>  ```cpp
>  // 对item20的StringPtrLess中的operator()结果取反实现StringPtrGreater
>  struct StringPtrGreater : public binary_function<const string*, const string*, bool> {
>      bool operator()(const string *ps1, const string *ps2) const
>      {
>          return *ps2 < *ps1;
>      }
>  };
>  // return !(*ps1 < *ps2);这里取反返回的是>=而不是>，因此这是一个无效的比较函数
>  // 需要改为 return *ps2 < *ps1;
>  ```

###  22:set和map的键（影响排序的值）不允许被改变

> **C++17，可用`extract`修改关联容器的`key`**
>
> 安全地改变`set`、`multiset`、`map`或`multimap`里的元素，5步骤
>
> ```cpp
> typedef set<Employee, IDNumberLess> EmpIDSet;
> EmpIDSet se;
> Employee selectedID;
> ...
> EmpIDSet::iterator i = se.find(selectedID); // 第一步：找到要改变的元素
> if (i!=se.end())
> {
>  Employee e(*i); // 第二步：拷贝元素
>  se.erase(i++); // 第三步：删除元素，自增保持迭代器有效
>  e.setTitle("Corporate Deity"); // 第四步：修改副本
>  se.insert(i, e); // 第五步：插入新值，提示它的位置和原先元素的一样
> }
> ```

###  23:考虑用有序vector替代关联容器
>  - 若查找速度优先，可用**散列容器**
>  - 标准关联容器的实现是使用**平衡二叉树**(对插入、删除和查找都进行了混合优化)，每个元素都会有其父指针及兄弟指针，因此最少一个元素12个字节，
>  - 而`vector`则不用(结构变大带来的问题是**页面和高速缓存方面的**)，但是得自己忍受排序的价值，排序的时候又要全部拷贝
>    但是排序完成后，查找速度更快，所以当你需要的结构**插入和删除比较少的时候**，使用**有序`vector`**是不错的选择。
>  - 只有**有序的容器**才能使用`binary_search`、`lower_bound`、`equal_range`等查找算法，所以只要`vector`有序也可以用
>  - 具体的实现则是`vector`里面存储pair结构，自行实现排序的算法(针对`pair`)
>  - `vector`的插入和删除很昂贵，所以只有当数据结构使用的时候查找不和删除和插入混合时，使用`vector`代替关联容器才有意义。

```cpp
typedef pair<string, int> Data;
class DataCompare { // 用于比较的类
public:
    bool operator()(const Data& lhs, const Data& rhs) const // 用于排序的比较函数
    {
        return keyLess(lhs.first, rhs.first);
    }
// 用于查找的比较函数（形式1）
    bool operator()(const Data& Ihs, const Data::first_type& k) const 
    {
        return keyLess(lhs.first, k);
    }
// 用于查找的比较函数（形式2）
    bool operator()(const Data::first_type& k, const Data& rhs) const 
    {
        return keyLess(k, rhs.first);
    }
private:
// 真正的比较函数
    bool keyLess(const Data::first_type& k1, const Data::first_type& k2) const 
    {
        return k1 < k2;
    }
};
//vector里面存储pair结构
vector<Data> vd; // 代替map<string, int>
... // 建立阶段
sort(vd.begin(), vd.end(), DataCompare()); // 结束建立阶段，模拟multimap时用stable_sort
string s; // 用于查找的值的对象
... // 开始查找阶段

// 通过binary_search查找 
if (binary_search(vd.begin(), vd.end(), s, DataCompare()))

// 通过lower_bound查找
vector<Data>::iterator i = lower_bound(vd.begin(), vd.end(), s, DataCompare()); 

if (i != vd.end() && !DataCompare()(s, *i))
pair<vector<Data>::iterator, vector<Data>::iterator> range =
    equal_range(vd.begin(), vd.end(), s, DataCompare()); // 通过equal_range查找
if (range.first != range.second)...
... // 结束查找阶段，开始重组阶段
    
sort(vd.begin(), vd.end(), DataCompare()); // 开始新的查找阶段...
```

###  24:map容器的下标[]操作和insert操作
> 结论:
>
> **添加用`insert`，更新用`[]`**,**用`emplace`代替`insert`**。
>
> > **`emplace`可传参构造**。
>
> ```cpp
> //-------------------------------添加
> map<int, Widget> m;
> m[1] = 1.50;
> // 等价于
> typedef map<int, Widget> IntWidgetMap;
> pair<IntWidgetMap::iterator, bool> result 
> = m.insert(IntWidgetMap::value_type(1, Widget())); //Widget() 默认构造，临时对象
> result.first->second = 1.50;//赋值
> 
> //多出，默认构造，析构，赋值等3个操作
> m.insert(IntWidgetMap::value_type(1, 1.50)); //值构造
> 
> //-------------------------------更新
> m[k] = v; // 使用operator[] ,没有构造和析构pair和Widget
> m.insert(IntWidgetMap::value_type(k, v)).first->second = v; // 使用insert
> ```

> **添加和更新都适用的函数**，efficientAddOrUpdate
>
> ```cpp
> template<typename MapType, typename KeyArgType, typename ValueArgtype>
> typename MapType::iterator efficientAddOrUpdate
>     (MapType& m, const KeyArgType& k, const ValueArgtype& v)
> {
>     typename MapType::iterator Ib = m.lower_bound(k);//查找
>     if(Ib != m.end() && !(m.key_comp()(k, Ib->first)))
>     {
>         Ib->second = v;
>         return Ib;
>     }
>     else{
>         typedef typename MapType::value_type MVT;
>         return m.insert(Ib, MVT(k, v)); 
>         // 把pair(k, v)添加到m并返回指向新map元素的迭代器
>     }
> }
> 
> efficientAddOrUpdate(m, 10, 1.5);
> // 如果m已经包含键是10的元素，推断出ValueArgType是double
> // 函数体直接把1.5作为double赋给与10相关的那个Widget
> ```

###  25:熟悉散列容器，unordered_set
>  - 可改写hash函数，比较是**通过相等**而不是等价，因为不需要排序了
>  - 但是仍然**不是按照插入顺序来排序**的，如果需要这种需求的话，改写一个vector实现。
>  - **C++11**引入`unordered_set`、`unordered_multiset`、`unordered_map`和`unordered_multimap`。

# 四、 迭代器

> **C++11直接用的`auto`获取迭代器类型`std::cbegin`、`std::cend`**

| 迭代器分类                  | 属性                                                         | 可用表达式                                                   |
| --------------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| 所有                        | 拷贝构造、拷贝赋值、析构、自增                               | X b(a); b = a; ++a , a++                                     |
| 输入                        | ==、!= ，单次解引用为右值                                    | a == b,a != b ，\*a  ，a->m                                  |
| 输出                        | 解引用为左值                                                 | \*a = t，\*a++ = t                                           |
| 前向<br/>（base输入、输出） | 默认构造，反复取引用                                         | X a;    X() ，{ b=a; *a++; *b; }                             |
| 双向<br/>（base前向）       | 递减                                                         | --a，a--，*a--                                               |
| 随机访问<br/>（base双向）   | 算术运算符+和-<br/>支持迭代器之间的不相等比较（<，>，<=和> =）<br/>带偏移解引用运算符（[]） | a ± n<br/>a < b<br/>a > b<br/>a <= b<br/>a >= b<br/>a += n<br/>a -= n<br/>a[n] |

###  26:尽量使用iterator而不是const_iterator或者reverse_iterator

>  - **`const_iterator`无法隐式转换到`iterator`**，`const`的可以操作非`const`的。
>  - 涉及`insert`和`erase`的一些版本要求`iterator`，而不能用`const_iterator`或`reverse_iterator`，故**不混用`const_iterator`和`iterator**`。


###  27:使用distance和advance实现去掉const_iterator的const
>  1.无法靠`const_cast`去掉`const_iterator`的`const`,本质`const_iterator`和`iterator`是完全不同类型，不只是`const`修饰符。
>  2.但`vector`和`string`内部就是`typedef const T* vector<T>::iterator`,故可直接转型，亲测VS不行
>
>  **C++11 提供范围`erase`得到迭代器**
>
>  ```cpp
>  typedef deque<int> IntDeque;
>  typedef IntDeque::iterator Iter;
>  typedef IntDeque::const_iterator ConstIter;
>  IntDeque d;
>  ConstIter citer;
>  Iter iter;
>  //传递给distance的两个参数类型必须相同,ConstIter
>  advance(iter, distance<ConstIter>(iter, citer));
>  
>  //C++11 提供范围erase得到迭代器。  
>  template <typename Container, typename ConstIterator>
>  typename Container::iterator remove_constness(Container& c, ConstIterator it)
>  {
>    return c.erase(it, it);
>  }
>  ```

###  28:通过reverse_iterator转换成Iterator
>  ```cpp
>  vector<int> v;
>  v.reserve(5);
>  for(int i = 0；i < 5; ++ i) 
>      v.push_back(i);
>  vecot<int>::reverse_iterator ri = find(v.rbegin(), v.rend(), 3);
>  //1    2    3      4     5
>  //         ri  ri.base()
>  //insert，ri指向3时，ri.base()指向4，
>  //若要删除ri，则删除的是ri.base()的往左一个元素
>  //v.erase(--ri.base());//无法编译，修改了base指针结果
>  v.erase((++ri).base());
>  ```

###  29:若只需单字符输入时，考虑istreambuf_iterator

> `istream_iterators`依靠的`operator>>`函数进行的是格式化输入，每次调用时要做大量工作。
>
> 它们必须建立和销毁岗哨（sentry）对象（为每个`operator>>`调用进行建立和清除活动的特殊的`iostream`对象），检查可能影响行为的流标志（比如`skipws`），进行全面的读取错误检查，如果遇到问题必须检查流的异常掩码决定是否抛出异常。若进行格式化输入，这些活动很重要。

>  ```cpp
>  ifstream inputFile("interestingData.txt");
>  inputFile.unset(ios::skipws); // 关闭inputFile的忽略空格标志
>  string fileData((istream_iterator<char>(inputFile)), istream_iterator<char>());
>  
>  ifstream inputFile("interestingData.txt");
>  string fileData((istreambuf_iterator<char>(inputFile)), istreambuf_iterator<char>());
>  ```

# 五、算法

###  30:确保容器的容量足够大小
>  1.`back_inserter`等`inserter`的使用可以确保不会出现未定义的操作.
>
>  ```cpp
>  int f(int x); // 此函数从x产生一些新值
>  vector<int> values;
>  vector<int> results;
>  //错误，*results.end()无对象
>  transform(values.begin(), values.end(), results.end(), f);//error
>  transform(values.begin(), values.end(), back_inserter(results), f);//正确
>  //覆盖现有容器元素，
>  if (results.size() < values.size()) 
>      results.resize(values.size());
>  transform(values.begin(), values.end(), results.begin(), f);
>  //或者
>  results.clear();
>  results.reserve(values.size());
>  transform(values.begin(), values.end(), back_inserter(results), transmogrify);
>  ```
>
>  2.插入`vector`或`string`时，**预先调用`reserve`，**移动元素的开销，但**避免了多次重新分配容器问题**
>
>  ```cpp
>  vector<int> values;
>  vector<int> results;
>  results.reserve(results.size() + values.size());//避免了重新分配容器
>  transform(values.begin(), values.end(), back_inserter(results), f);
>  ```
>
>  3.任何时候，容器需增加`size`的时，要用`inserter`，`deque`和`list`才能`front_insert`

 ###  31:选择合适的sort
>  - 需随机访问迭代器的排序:   `sort`、`stable_sort`、`partial_sort`（有序）和`nth_element`（无序），只能用于`vector`、`string`、`deque`和`array`。
>
>  - 需双向迭代器，因此可在任何标准序列迭代器上使用`partition`和`stable_partition`
>
>  - 要对`list`中进行`partial_sort`或`nth_element`，必须间接完成，3种方法。
>
>    > - 把元素拷贝到一个支持随机访问迭代器的容器中再对其用算法，
>    > - 建立`list::iterator`容器，对容器使用算法，然后通过迭代器访问list元素，
>    > - 使用有序的迭代器容器的信息来迭代地把list的元素接合到想让它们所处的位置
>
>    **性能:** `partion`>`stable_partition`>`nth_element`>`partial_sort`>`sort`>`stable_sort`

###  32:std:remove一定要和成员的erase联合使用(item 9)
>  - `std::remove`无法删除元素，未改变元素顺序，采取未删除元素向前覆盖被删除（`remove`不知道具体容器，故需成员的`erase`，因此用了remove一定要erase）。
>  - `list.remove`是整合了`erase`的，**成员函数版完成了删除操作**。
>  - 关联容器叫做`erase`
>  - 类似的还有`remove_if`和`unique`

###  33:避免在装有指针的容器使用remove类似的算法
>  1.remove会向前移动元素，就会有覆盖，覆盖了指针的话根本就没法`delete`了，就内存泄露了
>  2.解决方法是，先用一个`for_each`，`delete`掉相应的内存，或者使用`Partition`
>  其实就是正常解决方法
>  3.如果用的是**智能指针**的话，就可以直接`remove_erase`了


###  35:为STL库实现忽略大小写的比较
>  ```cpp
>  bool ciCharLess(char c1, char c2) // 函数对象
>  {
>      tolower(static_cast<unsigned char>(c1)) < 
>          tolower(static_cast<unsigned char>(c2));
>  }
>  bool ciStringCompare(const string& s1, const string& s2)
>  {
>      return lexicographical_compare(s1.begin(), s1.end(),
>          s2.begin(), s2.end(), ciCharLess);//函数对象,ciCharLess
>  }
>  ```

###  36:copy_if在VS里面已经实现了

> **C++11有了`std::copy_if`**

# 六、 仿函数、仿函数类、函数等

**用`lambda`替代仿函数**


###  38:把仿函数类设计为用于值传递
>  1.默认**仿函数对象**的传递都是基于值传递的。尽量设计可以拷贝的对象(**不要有多态性质**)

###  39:用纯函数做判断式
>  1.判断式就是返回`bool`的函数
>  2.**纯函数**是"返回值**只依赖于函数参数**"的函数,比如`f(x,y)`只依赖于`x`,`y`而不依赖于其他变量比如自己的成员变量,
>  也就是说每一次不同调用，只要参数相同结果就相同
>  3.**判断式类**是它的`operator()`是一个判断式，任何`STL`需要判断式的地方都可以传递判断式对象。
>  4.因为仿函数是值传递的，**因每次拷贝后为不同的对象，故一定要纯函数做判断式**，不然可能会出错。
>  这样一来，通常`operator()`都需要加上`const`使之成为`const`成员函数
>
>  ```cpp
>  class BadPredicate : public unary_function<Widget, bool> {
>  public:
>      BadPredicate(): timesCalled(0) {}
>      bool operator()(const Widget&){  //判断式，非纯函数
>          return ++timesCalled == 3;
>      }
>  private:
>      size_t timesCalled;
>  };
>  
>  vector<Widget> vw;
>  ...
>  vw.erase(remove_if(vw.begin(), vw.end(), BadPredicate()), vw.end());//错误
>  ```
>
>  `find_if`先接收了一个`timesCalled`为`0`的`BadPredicate`对象，`find_if`调用三次后对象返回`true`，然后才调用`remove_copy_if`，**传`p`的另一个拷贝**作为判断式，`p`的`timesCalled`成员仍为`0`。

###  40:使仿函数类可适配

>  ```cpp
>  class Widget {};
>  bool isInteresting(const Widget*pw){
>  	return true;
>  }
>  std::list<Widget*> widgetPtrs;
>  ```
>
>  查找第一个感兴趣的指针
>
>  ```cpp
>  auto it = std::find_if(widgetPtrs.begin(), widgetPtrs.end(), isInteresting);
>  ```
>
>  查找第一个**不感兴趣**的指针
>
>  ```cpp
>  it = std::find_if(widgetPtrs.begin(),widgetPtrs.end(),std::not1(
>      std::ptr_fun(isInteresting)));//要这样才行,std::ptr_fun包一下
>  ```
>
>  若要直接使用**模板继承**,帮你实现`typedefs`
>
>  ```cpp
>  struct IsInterestringFunctor :public std::unary_function<Widget*, bool>{
>  	bool operator()(Widget*pw) const { return true; }
>  };
>  struct IsInterestringFunctor2:public std::unary_function<Widget, bool>{//无const引用
>  	bool operator()(const Widget&w) const { return true; } 
>      //就是这样用的(非指针类型都去掉了const和引用)，没有为什么。
>  };
>  ```

###  41:ptr_fun,mem_fun,mem_fun_ref

> **C++17，移除`ptr_fun`，`mem_fun`，`mem_fun_ref`。**
>
> 早期：
>
> > - 使用`ptr_fun`传递非成员函数
> > - 使用`mem_fun_ref` 传递对象
> > - 使用`mem_fun` 传递指针，不能传参
> >
> > ```cpp
> > for_each(v1.begin(), v1.end(), test); //#1 非成员函数
> > for_each(v1.begin(), v1.end(), ptr_fun(test)); //#1 非成员函数,加ptr_fun也可，定义typedef
> > for_each(v1.begin(), v1.end(), mem_fun_ref(&Widget::printInfo)); 
> > for_each(pv1.begin(), pv1.end(), mem_fun(&Widget::printInfo)); //不能传参
> > ```
> 
>  **用`lambda`表达式**
> 
>  ```cpp
>  for_each(v1.begin(), v1.end(), [](Widget &w) {w.printInfo(); });
>  for_each(pv1.begin(), pv1.end(), [](Widget *pw) {pw->printInfo(); });
>  ```
> 
> **std::mem_fn**
>
> ```cpp
> struct Foo {
>     void display_greeting() {}
>     void display_number(int i) {std::cout << i << '\n';}
>     int data = 7;
> };
> 
> Foo f;
> auto greet = std::mem_fn(&Foo::display_greeting); //无参
> 
> greet(f);
> auto print_num = std::mem_fn(&Foo::display_number);//有参
> print_num(f, 42);
> 
> auto access_data = std::mem_fn(&Foo::data);       //成员变量
> std::cout << "data: " << access_data(f) << '\n';
> ```
>
> **std::function**
>
> ```cpp
> struct Foo {
>     Foo(int num) : num_(num) {}
>     void print_add(int i) const { std::cout << num_+i << '\n'; }
>     int num_;
> };
>  
> void print_num(int i){std::cout << i << '\n';}
> struct PrintNum {
>     void operator()(int i) const{std::cout << i << '\n';}
> };
>  
> int main()
> {
>     // store a free function
>     std::function<void(int)> f_display = print_num;
>     f_display(-9);
>  
>     // store a lambda
>     std::function<void()> f_display_42 = []() { print_num(42); };
>     f_display_42();
>  
>     // store the result of a call to std::bind
>     std::function<void()> f_display_31337 = std::bind(print_num, 31337);
>     f_display_31337();
>  
>     // store a call to a member function
>     std::function<void(const Foo&, int)> f_add_display = &Foo::print_add;
>     const Foo foo(314159);
>     f_add_display(foo, 1);
>     f_add_display(314159, 1);
>  
>     // store a call to a data member accessor
>     std::function<int(Foo const&)> f_num = &Foo::num_;
>     std::cout << "num_: " << f_num(foo) << '\n';
>  
>     // store a call to a member function and object
>     using std::placeholders::_1;
>     std::function<void(int)> f_add_display2 = std::bind( &Foo::print_add, foo, _1 );
>     f_add_display2(2);
>  
>     // store a call to a member function and object ptr
>     std::function<void(int)> f_add_display3 = std::bind( &Foo::print_add, &foo, _1 );
>     f_add_display3(3);
>  
>     // store a call to a function object
>     std::function<void(int)> f_display_obj = PrintNum();
>     f_display_obj(18);
> }
> ```

###  42:不要改变`less<T>`的行为(调用`operator<`)
> ```cpp
> //所以最好的办法是直接用一个新的比较器类型,而不是去改变less的行为
> struct MaxSpeedCompare:public std::binary_function<Widget,Widget,bool>{
> 	bool operator()(const Widget& _Left, const Widget& _Right) const{
> 		return _Left.maxSpeed() < _Right.maxSpeed();
> 	}
> };
> std::multiset<Widget, MaxSpeedCompare> set; //现在这样用即可。
> ```
>
> **仿函数类,可用`lambda`表达式**

# 七、使用STL编程


###  43：尽量用算法代替手写循环
>  - 使用算法原因：高效、正确、可维护
>  - `transform` 针对一个区间的元素进行某种操作，转换到另一个区间

###  44:尽量用成员函数替代同名的std::algorithm
>  - 成员函数版本更快(比如`set`，调用`.find` 对数时间`,std::find()`线性时间)
>
>  - 与容器结合的更好
>
>  - 判断元素相同，成员函数使用的是等价而不是相等。
>
>    > 这一差别对`map`和`multimap`尤其明显，它们的成员函数只关心`key`，如`count`成员函数只统计`key`值匹配的元素数，而count算法的寻找基于相等的全部组成部分
>
>  - `list`的成员函数的行为和`std::`版本通常都不一样
>
>  - **算法被特化为`list`时**（`remove`、`remove_if`、`unique`、`sort`、`merge`和`reverse`）都**要拷贝对象**，而`list`成员函数仅操作节点的指针。
>
>  - 算法和成员函数的**算法复杂度是相同。**
>
>  - `list`成员函数的行为和同名算法的行为经常不相同，比如`list`的`remove`、`remove_if`和`unique`**成员函数真正删除了元素**，不需要再调用`erase`。**`sort`算法不能用于`list`**，**但`list`有`sort`成员函数。**

###  45:`find`,`count`,`binary_search`,`lower_bound`,`upper_bound`,`equal_range`的区别
>  1.无序区间用`find`和`count`线性时间，有序对数时间；.`count`和`find`用相等搜索，其他的用的等价。
>  2.`find`和`count`检查是否存在，`find()`效率高，仅找第一个，但用`count`给关联容器比调用`equal_range`然后`distance`更快也更简单。
>  3.`find()`返回迭代器可以继续操作，`count`不能。
>  5.`binary_search`返回`bool`,只检查是否存在。`equal_range`返回一个区间，为找到的元素的区间，迭代器相同未找到。`equal_range`只用等价。
>
>  ```cpp
>  vector<Widget>::iterator i = lower_bound(vw.begin(), vw.end(), w);
>  if (i != vw.end() && *i == w) //真实相等，未找到情况下，返回可插入的位置
>  ```
>
>  **`equal_range`和`distance`**
>
>  ```cpp
>  VWIterPair p = equal_range(vw.begin(), vw.end(), w);
>  if (p.first != p.second) ... // 找到了
>  auto count =distance(p.first, p.second);
>  ```
>
>  **删除所有比`ageLimit`老的`timestamp`**
>
>  ```cpp
>  Timestamp ageLimit;
>  ...
>  vt.erase(vt.begin(), lower_bound(vt.begin(), vt.end(), ageLimit));// 不含ageLimit
>  ```
>
>  **要删除所有至少和ageLimit一样老的timestamp**
>
>  ```cpp
>  vt.erase(vt.begin(), upper_bound(vt.begin(), vt.end(), ageLimit));
>  ```



|                   目标                    |  无序区间算法  |           有序区间算法            | set或map的成员函数 | multiset或multimap的成员函数 |
| :---------------------------------------: | :------------: | :-------------------------------: | :----------------: | :--------------------------: |
|                值是否存在                 |     `find`     |          `binary_search`          |      `count`       |            `find`            |
| 值是否存在，第一个等于`Value`的对象在哪里 |     `find`     |           `equal_range`           |       `find`       |    `find`或`lower_bound`     |
|     第一个不在`Value`之前的对象在哪里     |   `find_if`    |           `lower_bound`           |   `lower_bound`    |        `lower_bound`         |
|      第一个在`Value`之后的对象在哪里      |   `find_if`    |           `upper_bound`           |   `upper_bound`    |        `upper_bound`         |
|           有多少对象等于`Value`           |    `count`     | `equal_range`再`distance` `count` |      `count`       |           `count`            |
|        等于`Value`的所有对象在哪里        | `find`（迭代） |           `equal_range`           |   `equal_range`    |        `equal_range`         |

###  46:考虑用函数对象替代函数作为算法的参数

>  1.函数对象的`operator()`可以`inline`，因此调用没有开销，**函数指针作为参数会抑制内联**。
>  2.直接传递函数的话，是先传指针，再通过指针去找的函数，更慢。

###  47:避免写太过复杂的算法代码，尽量分步写的简单一些，考虑到后期的维护

> 代码的读比写更经常，所以写代码时需要考虑可读性
>
> ```cpp
> typedef vector<int>::iterator VecIntIter;
> VecIntIter rangeBegin = find_if(v.rbegin(), v.rend(),bind2nd(greater_equal<int>(), y)).base();//find_if
> v.erase(remove_if(rangeBegin, v.end(), bind2nd(less<int>(), x)), v.end());//erase and remove_if
> ```

###  48:总是#include适当的头文件

> 尽量手动去包含所需要的头文件，有利于移植性。
>
> * 容器几乎都在同名的头文件中，例外的是`<set>`声明了`set`和`multiset`，`<map>`声明了`map`和`multimap`
> * 所有的算法都在`<algorithm>`和`<numeric>`中声明
> * 特殊的迭代器，包括`istream_iterators`和`istreambuf_iterators`，在`<iterator>`中声明
> * 标准仿函数（比如`less<T>`）和仿函数适配器（如`not1`、`bind2nd`）在`<functional>`中声明

###  49:STL库的错误信息可能非常多，但是要学会整理，替换，最后就很简单了

```cpp
string s(10);//error
errorC2664:'std::basic_string<char,std::char_traits<char>,std::allocator<char>>::basic_string(std::initialzer_list<_Elem>,const std::allocator<char> &)' : cannot convert argument 1 from 'int' to 'const std::basic_string<char,std::char_traits<char>,std::allocator<char>> &'
```

###  50:在const成员函数内部，类的所有非静态成员变量都会成为const的

* [cppreference](https://en.cppreference.com/w/%E9%A6%96%E9%A1%B5)
* [cplusplus.com](http://www.cplusplus.com/reference/)
* [STLport](http://www.stlport.org/)
* [Boost](https://www.boost.org/)