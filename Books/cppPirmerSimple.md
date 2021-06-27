# note-of-Cpp-primer

## 二、基本数据类型

1. `int` 最小是16位的，**不同编译器可能不同**，16位或32位。

2. `uint`和`int`互转时，**求值结果`uint`**。`float`转成`int`也会丢小数部分精度。(详细原理参考：类型提升)

    **尽量不混用有符号和无符号变量进行计算**。

    ```cpp
    unsigned int u = 10; 
    int i = -42; 
    cout<< u+i;//4294967264
    ```

3. 字面值如`U`、`L`时；**求值结果适配最小可容纳类型**（`（unsigned） int`, `（unsigned）long` ,`（unsigned）long long`）。

4. 初始化值的四种方法：**{窄化}不允许**

    ```cpp
    int i{0}; 
    long double i = 3.14; 
    int j = {i}; //error 窄化，不允许。
    int j = i; // ok,但不好，应static<int>(i)
    int j (i); // ok,但不好，应static<int>(i)
    int j (3.14); 
    int j {3}; 
    ```

5. `extern` 关键字：**声明和定义**

    ```cpp
    extern int i ;   //声明i
    extern int i = 5;//定义i，则extern无用
    ```

6. **引用`&`定义时赋初始值**，是绑定关系（像常量），指针是变量不需要。指针前面如果加上`*const`变成常量指针的话，也无法再改变了。

7. **`const` 变量默认只在单个文件中共享**，要在多文件共享加`extern`或定义在头文件后包含。

8. **引用与被引用需类型一致（或可类型提升）**。

9. `top-level const`修饰指针和 `low-level const`修饰指针所指向内容。

    ```cpp
    //top ：指针是常量，指针不能再指向其他变量
    //low ：指针所指向内存的数据不可改，可当作所指内存为常量。 
    int i = 0;
    int *const p1 = &i; // we can't change the value of p1; const is top-level
    const int ci = 42; // we cannot change ci; const is top-level
    const int *p2 = &ci; // we can change p2; const is low-level
    const int *const p3 = p2; // right most const is top-level, left-most is not
    const int &r = ci; // const in reference type is always low-level 
    
    //const 和*应该是从右向左读
        int     const      int    *        const
        int   constant   pointer to   constant
        <———————————————————————————————
        constant pointer to constant int
    ```

10. `constexpr` 关键字：编译器验证是否是`const`表达式，`constexpr point`  是`top-level`（指针是常量）。

11. `typedef` 的一个**坑**：（**使用C++11的`using`**）

    ```cpp
    typedef char *pstring; 
    const pstring str; //str是指向char的常量指针
    const char * pstr; //pstr指向常量
    ```

12. `auto` 会**自动忽略顶层`const` 。**

    ```cpp
    const int ci = i; auto f = ci;  //const int值可给int 赋值
    const auto f = ci; //f,ci类型一致
    ```

13. `decltype` 关键字，**推测`expr`的类型。**

    ```cpp
    const int ci = 0 ;
    decltype(ci) x = 0; //x的类型是const int
    //特例：
    int *p = &i ;
    decltype(*p); //c则c是int&类型
    ```

14. 预处理器代码`#include`， `#define`

## 三、字符串、向量和数组


1. **头文件不该使用`using`声明**。

2. **拷贝初始化和直接初始化**：

    ```cpp
    string s5 = "hiya"; //拷贝初始化
    string s6("hiyta");  //直接初始化
    string s7 = s5;     //拷贝赋值
    ```

3. 在**执行`cin>>str`操作时，开头的空格会被忽略**。

4. **`getline`读取含`\n`**。

5. `string.size()`类型是`unsigned`**确保能容纳**。

6. `string` 的`+`操作时，两边**至少有一个`string`对象**。

7. `vector`初始化的几种方式： 

    ```cpp
    vector<string> a = {"1", "2", "3"};  //统一初始化
    vector<int> ivec(10, -1);//个数，初始值
    vector<string> svec(10) ;//10个空的 
    vector<string> vec1(vec2);
    vector vec1 = vec2;
    ```

8. **迭代器的循环体，都不要向迭代器所属的容器中添加元素**。（可能导致迭代器失效）

9. char数组的`size`，`len(str)+1`， 编译器会报错了，**尽量不用数组，使用vector、string**

10. ```cpp
    int str[10];
    int * ptrs[10]; //10个指针  
    int (*Parray)[10] = &arr ;// 一个指针，指向一个数组     
    int (&arrRef)[10] = arr; // 一个引用，引用一个数组   
    int *(&arry)[10] = ptrs; //arry是一个数组的引用，数组包含10个指针
    
    int ia[] = {0,1,2,3,4}; 
    auto ia2(ia); // 事实上是 auto ia2(&ia[0]),ia2是指针,指向int
    
    int *beg = begin(ia) //指向ia首元素的指针
    vector<int> ivec(begin(int_arr), end(int_arr));
    ```

## 四、表达式


1. 仅使用`unsigned`整型变量进行**位运算**。

2. > 隐形转换
   >
   > - **数组名转换成指针**（指向数组首元素的）。
   > - **常量整数值`0`或字面值`nullptr`能转成任意指针类型**；
   > - 指向任意非常量的**指针能转换成`void*`；不安全，少用转换。**
   > - 指向任意**对象的指针能转换成`const void*`。**
   > - **算术类型或指针类型都能转换成布尔类型**。
   > - 指向**非常量类型的指针**能转换成指向**相应的常量类型的指针**。

4. 尽量避免强制类型转换。

   > - `dynamic_cast`支持运行时类型识别，（**父类转子类（指针/引用）**）。
   > - 显性类型转换，**不含底层`const`，都能`static_cast`。**
   > - **`const_cast`去除底层`const`，常用于函数重载。**
   > - **`reinterpret_cast`太危险了**，通常为运算对象（就是指鹿为马）。

## 五、语句


1. 使用**空语句加注释**

    ```cpp
    while(cin >> s && s!='a'){
        ;//空语句
    }
    ```

2. **不省略`case`后的`break`，以及`default`。**

4. **`switch` 含要添加定义时候加`{ }`。**


5. **`do while` 语句可以先执行循环体**，在有些情况可能会比较好用。

6. 异常处理代码：

    ```cpp
    #include <stdexcept>
    try{
        if(a == b){
            throw runtime_error("is not same");
        }
    }catch(runtime_error err){
        cout<<err.what();
    }
    //若throw出了一个异常，那么catch会按照函数调用关系依次向上寻找catch，未找到catch，则会转到terminate的标准库函数终止。
    //runtime_error
    //logic _error
    //overflow _error
    ```

## 六、函数


1. `static`变量，全局存储区，仅其作用域可见，若 `static int function（）；`，仅本文件可见。可用于多文件多函数的定义。

2. **函数非基础类型传引用**，不修改值情况下，**尽量传常量引用（1.值不变且传递快；2.常对象参数可用）**。

    ```cpp
    int find_char(string &s, char c);      //错误定义，调用常量的str出错。
    int find_char(const string &s, char c);//正确定义
    find_char("hello world", "o");         //调用
    ```

3. **函数中数组传递的三种方式**。

    ```cpp
    void myfunc(const int*);
    void myfunc(const int[]);
    void myfunc(const int[10]);
    int i = 0; 
    int j[2] = {0, 1};
    myfunc(&i);
    myfunc(j);
    
    func(int (&arr)[10]); //arr是具有十个整数的整形数组的引用，（1.引用，指向数组，数组存的10个int类型）
    func(int &arr[10])  ; //arr是引用的数组,类似int *aa[10];
    ```

4. `main`里面的`(int argc, char *argv[])`，程序本身的名字是`argv[0]`,最后一个参数一定是`argv[argc] = 0`;

5. 可变参数的函数：**所有实参类型相同：传递`initializer_list`这样一个标准库**。

      ```cpp
      void error_msg(initializer_list<string> msg); //类型相同
      error_msg({"123", str1, str2})//花括号不能去掉
      //实参类型不同：可变参数模板
      void foo(int, ...);//C语言，不好
      ```

7. **函数返回左值，只不过比较奇怪**。

    ```cpp
    char &get_val(){
        return str[idx]
    }
    get_val = 'A';
    ```

7. **函数返回值列表**。

    ```cpp
    vector<string> process(){return {"123", "4565"}}
    ```

8. **返回数组指针：`C++`无法直接返回数组指针，只能用`using`。**

    ```cpp
    typedef int arrT[10];
    using arrT = int[10];
    arrT* func(int i);  //数组指针
    int (*func(int i))[10];//等价
    auto func(int i)->int(*)[10];//尾置返回类型->符号 
    ```

9. 同名屏蔽

   ```cpp
   void print(string s);
   void func(){
   	void print(int i);//声明,意味着只能调研int类型的print了
   	print("123");//无法找到,print(int)屏蔽了。
   }
   ```

10. **`inline`函数：适用于优化规模较小、流程直接、调用频繁的函数，`constexpr`函数被隐式`inline`，通常定义在头文件中。**

11. **`constexpr`函数的返回值可以不是一个常量，**通常定义在头文件中。

12. 调试帮助：`__LINE__`，`__func__`

13. 函数重载的匹配过程：

    > - 实参形参**类型相同**、**数组类型和指针类型之间的转换**、添加或者删除**顶层`const`**。
    > - **`const`转换实现的匹配**。
    > - 通过**类型提升**实现的匹配。
    > - **算数类型转换或者指针转换**实现的匹配。
    > - **类类型转换实现的匹配**

14. 函数指针：std::function、Lambda。

    ```cpp
    bool lengthCompare(const string &, const string &);
    // pf points to a function returning bool that takes two const string references
    bool (*pf)(const string &, const string &); // uninitialized
    pf = lengthCompare; // pf now points to the function named lengthCompare
    pf = &lengthCompare; // equivalent assignment: address-of operator is optional
    bool b1 = pf("hello", "goodbye");       // calls lengthCompare
    bool b2 = (*pf)("hello", "goodbye");    // equivalent call
    bool b3 = lengthCompare("hello", "goodbye");    // equivalent call
    ```

15. 重载函数的指针：

    ```cpp
    void ff(int*);
    void ff(unsigned int);
    void (*pf1)(unsigned int) = ff; // pf1 points to ff(unsigned)
    ```

16. **关键字`decltype`作用于函数时，返回的是函数类型，而不是函数指针类型。**

17. 尾置返回类型的函数，一个函数wrapper：

    ```cpp
    int& func(int& i );
    float func(int& f);
    
    template<typename T>
    auto func(T& val) -> decltype(func(val))//C++11,14以后直接可以auto返回
    {
        return func(val);
    }
    ```

18. 函数后面添加`const`关键字，表示**该成员函数不会改变成员中的变量,this是常量指针。**

    ```cpp
    float getPi() const{ 	return 3.14;	}
    string isbn() const{    return this->bookNo; } //this是常量指针,不能改变返回值，也不能改变this里面的值
    ```

## 七、抽象数据类型（类）

1. `friend` 友元，用于友元类或者友元函数**访问私有成员，单向，不传递**。

2. **`mutable` 关键字：突破`const`的限制**。

   ```cpp
   mutable int m_iTimes;
   void ClxTest::Output() const{	m_iTimes++;	}	//突破const的限制。
   ```

3. 前向声明：只声明，不定义，**一般用于声明对象为指针`A*`或者引用`&`的函数**,`class A;`。

4. 函数在查找参数过程，**先查找函数作用域的参数，再查找类成员的参数**。

   ```cpp
   int height;// 1
   void Screen::dummy_func(pos ht){
       cursor = width * height;//1
       cursor = width * this->height; //2
       cursor = width * Screen::height;//2
   }
   ```

6. 类的**委托构造函数**

    ```cpp
    class Sales_data{
    public:
       Sales_data(int a, string b):bookNo(a), describe(b){}
       //委托给上面那个构造函数
       Sales_data():Sales_data(1, "123"){}
    }
    ```

7. `explicit` 关键字：**禁止隐式转换，只对单实参的构造函数有效**，但可用`static_cast`显示转换。

    ```cpp
    string null_book = "9-999-99999-9";
    Sales_data item1 (null_book);   // ok: direct initialization
    Sales_data item2 = null_book;  // error: cannot use copy form of initialization with explicit constructor
    item.combine(Sales_data(null_book)); // ok: the argument is an explicitly constructed Sales_data object
    item.combine(static_cast<Sales_data>(cin)); // ok: static_cast can use an explicit constructor
    ```

8. 字面值常量类：**数据成员都是字面值类型**，类必须有一个`constexpr`的构造函数。

9. 类的`static`成员：

    > 1. 由于**静态成员无法对象绑定，故不能声明为`const`**（const成员函数：不会修改该函数所属对象），也**无法用`this`指针**。
    > 2. 用户代码可以使用**作用域运算符访问静态成员，也可以通过类对象、引用或指针访问**。**类的成员函数可访问静态成员**。
    > 3. 在**类外**部定义静态成员时，**不能重复`static`关键字**，其只能用于类内部的声明语句。
    > 4. 通常情况下，**不应该在类内部初始化静态成员**。而必须在**类外部定义并初始化每个静态成员**。
    > 5. **类内初始值初始值必须是常量表达式`constexpr`。**
    > 6. 静态数据成员的类型，**可为其所属的类类型（单例模式）**。
    > 7. 可以使用静态成员作为函数的默认实参。
    > 8. 建议**把静态数据成员的定义与其他非内联函数的定义放在同一个源文件中**，这样可以确保对象**只被定义一次**。

## 八、IO


1. `unitbuf` 操作符：可以在每次输出操作后立即刷新缓冲区,将缓冲区的数据输出出来，例如 `cout << unitbuf`
2. `tie`方法：将一个`ios`对象和另一个绑定起来，例如 `cin.tie(&ofs)`，就可以每在命令行中输入一个字符，就打印/写入到文件里面。
3. 默认情况下，打开一个`ofstream`的时候，会把里面原来的文件内容全部丢弃掉，贼坑！！！！解决方法是加一个app模式，例如

      ```cpp
      ofstream app("filename", ofstream::app)//隐含为输出模式 或者
      ofstream app("filename", ofstream::app|ofstream::out) 这里的app指的是append
      ```

4. `stringstream`对象，其实就相当于一个`string`的缓冲区。用法：

    ```cpp
    ostringstream badNums;
    if(!valid(nums)){
        badNums << nums; //写入
    }
    cerr << badNums.str() << endl;
    ```

## 九、顺序容器


1. 因为std标准库里面的容器效率比较低，所以在有替代品的时候，不推荐使用任何一种容器。

2. 
    ```cpp
    forward_list 单向链表                只能单向访问，任何位置插入都很快
    vector       可变大小数组            随机访问快，尾部插入元素快，中间插入删除元素慢
    deque        双端队列                随机访问快，头尾插入删除快，中间（可能）慢
    list         双向链表                只能双向顺序访问，任何位置插入都很快
    array        固定大小数组            都很快，但是不能添加删除元素
    string       与vector类似         
    ```

3. 迭代器除了`.begin()` .end()以外还有`.cbegin()`, `.rbegin()`, `.crbegin()`,C指的是`const`，r指的是反向迭代器

4. array数组类型与普通数组不同的地方在于，**array数组类型允许类型赋值**，如：`c = {a, b, c}; c2 = c;`

5. 顺序容器的`assign`成员方法：实现顺序容器的拷贝（array除外，不允许拷贝）：

    ```cpp
    list<string> names;
    vector<const char*> oldstyle
    names = oldstyle // 错误，因为类型不同
    names.assign(oldstyle.begin(), oldstyle.end());//正确，使用迭代器
    names.assign(10, "Hiya");//替换成10个"Hiya"
    ```

6. `swap`交换地址，若`swap`前指向`svec1[3]`的迭代器，后将会指向`svec2[3]`。

    这种做法会让`swap`很快，**但`array`是真的交换数值的**。

7. 范围insert：

    ```cpp
    vector<string> v = {"quasi", "simba", "frollo", "scar"}
    slist.insert(slist.begin(), v.end() - 2, v.end());//将v的最后两个元素添加到slist的开始位置
    svec.insert(svec.end(), 10, "Anna");
    slist.insert(slist.end(),{"these", "words"});
    ```

8. `vector`数组在申请内存的时候，会提前申请大于所需的空间，作为备用。因为如果每`insert`一次就申请内存的话，将会使效率非常低。

9. `capacity`和`size`的区别：`capacity`现有内存空间最多可容纳元素数量,`size`是现有元素数量。

10. 容器的适配器（`stack`，`quene` 和 `priority_quene`）

    ```cpp
    deque<int> deq
    stack<int> stk(deq);
    stack<string, vector<string>> str_stk(svec);
    
    while(!skt.empty()){
        int value = stk.top();
        stk.pop();//还有push以及emplace操作
    }
    //quene也是类似的操作。
    ```

11. 注意：**`vector<bool>`不是一个容器，尽量避免使用**，**是按位存储**。
    
12. 对于`vector<vector<T>>`在使用for的时候，**一定要加`&`引号**，例如下面：
    ```cpp
    vector<vector<int>> v = {{1,0}, {-1,0}, {0,-1}, {0,1}};
    for(auto a : v){}
    for(auto& a : v){}
    //说明下，在for中大量对象创建时候，需要考虑构造与析构问题。可以考虑定义在for外
    ```
    

## 十、泛型算法


1. 泛型算法其实只是我们平时使用容器的类似于`find`/`equal`等方法的泛称，由标准库提供。

2. **只读时，推荐采用`cbegin`和`cend`**而非`begin`和`end`。

3. `accumulate`第三个元素的`+`运算符，故`accumulate(v.cbegin(), v.cend(), "")`不对，**因`""`是`const string`， 未重载`+`**。

4. 一些**写入的泛型算法只修改容器内的值**，但**不申请新的空间**。

    ```cpp
    vector<int> vec;//0==size
    fill(vec.begin(), 10, 0);//其中vec实际是空的，这时候fill就会报错
    ```

5. `back_inserter`插入迭代器，可以通过向此迭代器赋值来**向容器插入元素**

    ```cpp
    vector<int> vec;
    auto it = back_inserter(vec);
    *it = 43;//此时就向vec中插入一个元素，vec的大小为1
    //或者
    fill_n(back_inserter(vec), 10, 0);//就可以向vec中添加10个元素
    ```

6. **自定义排序**：谓词

    ```cpp
    sort(words.begin(), words.end(), isShorter); 
    bool isShorter(const string &s1, const string &s2){return s1.size() < s2.size()}
    ```

7. `lambda` 表达式：**认为是未命名的内联函数**。

    ```cpp
    auto f = []{return 43;};
    f();
    stable_sort(words.begin(), words.end(), [](const string &a, const string &b){return a.size() < b.size()});
    //其中捕获列表[]指的是那些明确使用的局部变量，例如
    int sz;
    f = [sz](const string &a){return sz > a.size()};
    //其中这个捕获的值是在创建的时候捕获，而不是调用的时候捕获，所以应该尽量减少捕获的值，防止在创建到调用这段时间内变量发生变化。
    ```

8. 隐式捕获：

    ```c++
    f = [=, &os](){}//os是引用捕获方式，其他为值捕获方式
    ```

9. `lambda`表达式的返回类型，需要尾置：

    ```cpp
    []() -> int {}//C++ 14以前
    ```

10. `lambda`的好处：**本身是可调用对象，可以作为参数存在**，如果想要让函数也有这样的功能，可以使用`bind`方法：

    ```cpp
    bool check_size(const int, const string);
    bind(check_size, 1, "str");
    
    //此时就可以作为参数调用了，例如 
    find_if(word.begin(), words,end(), bind(check_size, 1, "str"))
    ```

11. **`placeholders` 用于占位，`ref`用于生成一个可以拷贝的引用对象**。

      ```cpp
      istream_iterator<Sales_item> item_iter(cin);//处理类的输入输出
      ```

12. 所有**泛型算法的`_if`版本都是可以接受一个谓词**作为判断条件的。

## 十一、关联容器（主要是map，set）

1. `map.upper_bound(k)`返回**第一个`key > k`的元素的迭代器**，`count(t)`返回关键字等于k的元素的数量

2. **因为`map`、`multimap`是有序的**，**查找元素可用`lower_bound`和`upper_bound`或者`equal_range`进行搜索**，。

    ```cpp
    for(auto beg = authors.lower_bound(searchItem), end = authors.upper_bound(searchItem);beg!=end; beg++)
    for(auto pos = authors.equal_range(searchItem);  pos.first!=pos.second  ; pos.first++) //`equal_range`
    ```

3. **无序关联容器的本质是使用哈希函数和关键字来进行。**

    > **存储组织上实际是一组桶，每个桶保存0个或者多个元素。用哈希函数将元素映射到桶。**
    >
    > 管理桶的方法：`bucket_count()`正在使用桶的个数等。**自定义类类型作为关键字**，需要自定义`hash`如：
    >
    > ```cpp
    > size_t hasher(const Sales_data &sd){ return hash<string>()(sd.isbn());}
    > bool eqOp(const Sales_data &lhs, const Sales_data &rhs){ return lhs.isbn()==rhs.isbn();}
    > ```

## 十二、动态内存、智能指针

1. **静态内存**：局部static变量、类static数据成员、定义在函数之外的变量，编译器自动创建和销毁，在使用之前分配，在程序结束后销毁。

   **栈内存**：保存函数内的非static变量，编译器自动创建和销毁，程序块运行的时候才存在。

   **堆内存**：动态分配内存。

2. `shared_ptr`类：允许多个指针指向同一个对象除了初始化以外，使用方法和普通指针一样：
   
    ```cpp
    shared_ptr<string> p1; 
    *p1 = "hi"; 
    swap(p ,q)//交换pq的指针。推荐使用make_shared来进行shared_ptr的创建，相对比较安全（能够防止例如将同一块内存绑定到多个shared_ptr上面：
shared_ptr<int> p3 = make_shared<int>(42); //安全
   ```
   
3. `shared_ptr` 在最后一个对象销毁前都不会释放内存，若把`shared_ptr`放在容器中重拍后，一部分元素不需要，要`erase`元素。
4. 使用`auto`进行变量的初始化（已被弃用）：

    ```cpp
    auto p1 = new auto(obj)//这简直太秀了。。。不管不顾的。。，
    auto //会转义所有权，sort后元素都成为nullptr
    ```

5. 定位new

    ```cpp
    (placemant new) int *p2 = new(nothrow) int  //给new传递一个参数，如果内存耗尽，则不抛出异常，返回一个空指针。
    ```

6. 关于该不该使用智能指针的问题：

  - C++primer中提到：“**坚持只使用智能指针，就可以避免所有的内存泄漏问题**，所以应该提倡使用智能指针”。

  - 但是在网易的分享/培训当中却说道：不要使用共享指针。其主要原因是：共享指针的乱用导致：被shared_ptr的资源实际上并没有共享，这样就会使代码出现资源泄漏和一些bug，而且因为有可能会出现其他程序员通过赋值给另一个共享指针而修改了这一段资源，这样的bug就会很难查出来。另一个原因时shared_ptr并不一定是线程安全的，所以要小心。同时有时候会忘记使用make_share来创建shared_ptr，会导致性能下降以及安全问题。同时经常会出现使用delete把智能指针删除的情况。
    另外对于游戏来说不用智能指针更好，因为引用计数本身会带来额外的开销，而且内存分配东一块西一块很不好管理，cache也不友好，最好是对象都放到列表里面，都用数组下标访问，这种方式既容易管理又容易统计还可以把完全不一样的数据结构做出功能上的抽象，比如参考bgfx对于图形API的封装，DX9/DX11/DX12/OpenGL/Vulkan 全都可以用一套API包装起来，texture这样的复杂结构反正也只需要用到一个索引访问----by 果哥

7. 防止野指针：养成在变量离开作用域之前就释放掉或者**在`delete`后将`nullptr`赋给指针**,就表示已经释放掉了但是可能还要用。

8. 智能指针不初始化的话,会被初始化成空指针,我们也可以自己初始化:

    ```cpp
    shared_ptr<int> p2(new int(42));
    shared_ptr<int> p1 = new int(1023);//错误，必须使用直接初始化，此时是explicit，我们不能把内置指针隐式转换成智能指针
    ```

9. **不要混合使用智能指针和普通指针**，会出现很多问题，例如：

    ```cpp
    void process(shared_ptr<int> ptr){}
    int *x(new int(1024))
    process(x)//错误：x是普通指针
    process(shared_ptr<int>(x))//合法的，但是内存会被释放
    int j = *x //未定义的，x是空指针
    ```

10. **不要使用智能指针的**get方法返回的**内置指针初始化另一个指针**，或者给智能指针复制。

11. 若程序崩溃发生异常的时`new`出来的对象没有`delete`，则内存永远不会释放。

12. 自定义删除器：

     ```cpp
     shared_ptr<connection> p(&c, end_connection); //p被销毁的时候，会调用end_connection可以做扫尾工作。
     ```

13. `unique_ptr`是不能拷贝和复制的，但是可以通过`release`和`reset`将指针的所有权从一个非`const`的`unique_ptr`转移给另一个。

     `unique.release`会切断`unique_ptr`和原来对象间的关系。

     `reset`是让`unique_ptr`重新指向给定的指针。

     ```cpp
      p2.release()//错误，p2不会释放内存而且我们也丢失了指针
      auto p = p2.release()//正确，但要自己 delete(p)
      p3.reset(p2.release())//p2内存交给p3
     ```

14. `weak_ptr`局外人：一种`shared_ptr`，**不增加计数**。在使用`weak_ptr`时要**先用`lock`判断对象是否存在**：

     ```cpp
      if(shared_ptr<np> np = wp.lock()){} //是否存在
     ```

15. 对动态数组的初始化：

     ```cpp
     int *p = new int[10]()//10个都是0的int
     int *p = new int[0]; //p是一个类似于尾后指针(end())的非空指针。
     ```

16. 智能指针支持动态数组，但是**只有`unique_ptr`支持直接下标访问**，`shared_ptr`想要访问的话必须提供自己的删除器，并且通过`get`来修改数组。

17. `new`和`delete`将**对象构造/析构**和**内存申请/释放**结合在了一起（某道面试题），因此`allocator`可以将两个分开来。：

     ```cpp
     allocator<string> alloc;//可以分配string的allocator对象
     auto const p = alloc.allocate(n);//可以分配n个未初始化的string
    
     alloc.construct(p++);//p为空字符串
     alloc.destroy(--p);//销毁，这里其实应该有一个while的
    
     auto q = uninitialized_copy(vi.begin(), vi.end(), p);//拷贝数据
     alloc.deallocate(p,n)//释放内存
     ```

## 十三、（对象）拷贝控制

1. 对于拷贝构造函数**是`explicit`的构造函数**来说，使用**拷贝初始化**还是**直接初始化**是**有很大不同**的：

    因为vector的接受单一大小参数的构造函数是`explicit`的，所以会出现错误，**必须显式进行调用**
    
    ```cpp
    vector<int> v1(10); //正确，直接初始化
    vector<int> v2 = 10;//错误，接受大小参数的构造函数是explicit的
    void f(vector<int>);//f的参数进行拷贝初始化
    f(10);              //错误。不能用一个explicit的构造函数拷贝一个实参
    f(vector<int>(10)); //正确：从一个int直接构造一个临时vector
    ```

2. **析构函数**：先执行**函数体**，再**按成员初始化的逆序销毁**。若需要**析构函数**，一般也需要一个自定义**拷贝赋值运算符**和**拷贝构造函数**。
3. **阻止拷贝**：虽然声明了他们，但是不能用任何方式使用他们：

    ```cpp
    noCopy() = default;//使用合成的默认构造函数
    noCopy(const noCopy&) = delete //阻止拷贝
    noCopy &operator=(const noCopy&) = delete; // 阻止赋值
    ~noCopy() = default;// 使用合成的析构函数
    //析构函数不能是delete的，因为如果析构是delete的，那对象无法销毁了
    ```

4. 对于**成员变量有`const`的，或者有引用`&`成员的**，**编译器无法对类进行默认构造函数**。
5. 关于`std::swap` 和 using namespace std; `swap()`的区别：
如果使用前者的话，则每次调用一定会使用std版本的swap，而如果使用后面这种写法，就可以在类内定义swap，然后调用的时候会先检查类内部是否有swap，如果没有的话才调用std版本的swap，这个小技巧还是要知道的，**使用`std::move`可以避免潜在的名字冲突**。
6. `move`标准库函数：**调用移动构造函数**，相当于一个`static_cast`，但是同时可以将原来的`str`移走，需要使用`str::string t(str::move(r));`如果单纯使用`string&&r = std::move(r)`的话，并不能移走
**move可以获得绑定到左值上的右值引用**
7. **右值引用**：必须绑定到右值的引用， 即只能**绑定到一个即将销毁的对象**

    ```cpp
    int i = 42;//正确
    int &r = i;//正确， r引用i
    int &&rr = i;//错误，不能将一个右值引用绑定到左值上面，这里面i是一个左值
    int &r2 = i*42;//错误 i*42是一个右值
    const int &r3 = i * 42;// 正确，我们可以将一个const的引用绑定到一个右值上面
    int &&rr2 = i*42;// 正确，将rr2绑定到乘法结果上。因为i*42是一个右值
    
    //本质其实是因为i*42 是一个右值，i是左值
    int &&rr1 = 42;//正确，字面常量是右值
    int &&rr2 = rr1//错误，表达式rr1是左值
    ```

8. `noexcept`关键字：告诉我们的函数将不会抛出任何异常：

    ```cpp
    StrVec::StrVec(StrVec &&s) noexcept:element(s.element){}
    //不抛出异常的移动构造函数和移动赋值运算符必须标记为noexcept
    ```

9. 定义了一个**移动构造函数**或者**移动赋值运算符**的类必须**也定义自己的拷贝操作**，不然有些成员会被默认的定义成`delete`的。而如果定义了拷贝构造函数**未定义移动构造函数**的时候，使用了`move`方法， **仍执行拷贝构造**`Foo z(std::move(x))`执行拷贝构造
10. 对于同时存在拷贝构造函数和移动构造函数的时候，编译器对两者的选择是看是否是右值或者是左值的，右值是移动，左值是拷贝，例如

    ```cpp
    StrVec v1, v2;
    v1 = v2 //v2是一个左值，使用拷贝赋值
    StrVec getVec(istream &)//返回一个右值，此时拷贝和移动都是可行的，但是因为调用拷贝赋值需要进行一次到const的转换，而StrVec&是精确匹配的，所以会用移动赋值
    v2 = getVec(cin)//是一个右值，选择移动赋值
    ```

11. 调用`make_move_iterator`可以讲一个普通的迭代器转换成一个**移动迭代器**，这个**迭代器可以返回右值**：

     ```cpp
     auto first = alloc.allocate(newcapacity);
     auto last = uninitialized_copy(make_move_iterator(begin()), make_move_iterator(end()), first);
     ```

12. **引用限定符&**（reference qualifier）放在**函数的参数列表后面**，**表示this可以指向一个右值还是左值**。

    ```cpp
    Foo &operator=(const Foo&)& //只能向可修改的左值赋值
    Foo &operator=(const Foo&)&&//只能向右值赋值
    //如果和const一起使用的话，const必须在前：
    Foo someMem()const &;
    //同时如果重载的话，必须所有的版本都必须加上引用限定符或者不加
    Foo sorted() &&
    Foo sorted() const//错误，因为上面有引用限定符，所以这里必须也有
    ```

## 十四、重载和类型转换

1. 是否将重载运算符定义为成员函数（或者普通函数）

赋值（=）， 下标(\[]), 调用（()）和成员访问箭头必须是成员函数

复合赋值运算符一般来说应该是成员，但并非必须

改变对象状态的运算符或者与给定类型密切相关的运算符，如递增，递减，解引用运算符，通常是成员

具有对称性的运算符可能转换任意一端的运算对象，如相等性，加减乘除，关系运算符，普通的非成员函数

2. 一个类如果有下标运算符重载的时候operator\[](),一般会有两个版本，一个是const版本，另一个是非const版本，这样当我们给对象赋值的时候使用非常量的，当我们只是作为常量返回的时候就不能赋值
3. 如何区分递增++，递减--的前置还是后置：后置版本提供一个值为0的实参（虽然实际上这个实参没什么用，只是用来区分前置和后置的）

    ```cpp
    StrBlobStr operator++(int)//后置运算符 a++
    StrBlobStr& operator++()//前置运算符 ++a
    ```

4. 重载解引用运算符(\*)和箭头运算符（->）的时候,解引用运算符可以返回任意我们想要的数据，例如operator\*(){return 42;},但是箭头运算符则必须指向类对象的指针或者是一个重载了->的累的对象，除此以外都会发生错误
5. 函数调用运算符的重载：即可以通过对类的调用来完成一些操作：

    ```cpp
    struct absInt{
        int operator()(int val)const{
            return val < 0? -val:val;
        }
    }
    int i = -42;
    absInt absObj;
    int ui = absObj(i);
    其作用类似于lambda表达式
    ```

6. 标准库function类型：实际上是一个模板：

    ```cpp
    function<int(int, int)> f1 = add;//int(int, int)是一个函数类型，接受两个int，返回一个int,add是一个函数指针
    function<int(int, int)> f2 = divide();//divide是类似上一条的对象类的对象
    function<int(int, int)> f3 = [](int i, int j){return i*j};//lambda
    cout<< f1(4, 2)<<endl;
    map<string, function<int(int, int)>> binops;然后就可以添加任意形式的可调用对象了
    例如binops["+"] = std::add<int>();
    ```

7. 重载函数无法放到map里面，可以的做法是将函数指针放进去，例如：

    ```cpp
    int (*fp)(int, int) = add;
    binops.insert({"+", fp});
    ```

8. 一般不会编写隐式的类型转换运算符，而可以编写显式的类型转换运算符，并且应该尽量避免有二义性或者容易引起误解的类型转换。
        例如同时编写一个 operator int()const 和 operator double() const的时候，调用long double时就会出现二义性

## 十五、面向对象程序设计

1. 养成写override 关键字的习惯，指明该函数是从基类中改写的。
2. 动态绑定：根据调用的实参类型选择到底是使用父类的成员函数还是子类的成员函数：

    ```cpp
    class Buik_quote:public Quote{    }
    double print_total(const Ouote &item){}//动态绑定
    print_total(basic)//调用父类
    print_total(bulk)//调用子类
    ```

3. 派生类和基类的存储空间是不连续（可能需要考虑到cache存储）
4. 如果一些类不想让其他类继承，可以使用final关键字：

    ```cpp
    class NoDerived final {}//这个类不能被继承
    ```

5. 从派生类到基类的转换：自动类型转换只对指针和引用类型有效，同时忽略派生类独有的对象。
   基类向派生类不存在隐式类型转换
和任何其他成员一样，派生类向基类的类型转换也可能因为由于访问受限而变得不可行。
6. 多态性：具有继承关系的多个类型成为多态类型。当我们使用基类的引用或者指针调用基类中定义的一个函数时，我们并不知道该函数真正作用的对象是什么类型。
7. 使用作用域运算符可以实现强迫虚函数执行某个特定版本的功能：

    ```cpp
    double undiscounted = baseP->Quote::net_price()
    ```

8. 纯虚函数：即表示当前这个函数时无意义的，拥有纯虚函数的类被称为“抽象基类”，是无法定义对象的，只能用来继承

    ```cpp
    class Disc_quote : public Quote{
        double net_price() const = 0;
    }
    Disc_quote discounted;//错误
    ```

9. 派生访问说明符：控制派生类用户对于基类成员的访问权限：

    ```cpp
    class Base{
        public:
            void pub_mem();
        protected:
            int prot_mem;
        private:
            char priv_mem;
    }
    class Pub_derv:public Base{
        int f(){return prot_mem}//正确，派生类可以访问protect
        char g(){return priv_mem;}//错误，private不可访问
    }
    class Priv_derv:private Base{
        int f(){return prot_mem}//正确，派生类可以访问protect
        char g(){return priv_mem;}//错误，private不可访问
    }
    ```

但是在对象对于基类成员的访问时就会出现不同，即所有父类的成员都是private或者public的：

```cpp
    Pub_derv pub_d1;
    Pirv_derv priv_d2;
    pub_d1.pub_mem();//正确，因为是public
    priv_d2.pub_mem();//错误，因为是private的
    类型转换也同样适用
```

10. 友元friend关系是不能继承的，不能因为父类是friend，所有的派生类都是friend。
11. 改变个别成员的可访问性：有时候我们需要改变个别成员的可访问性，可以使用using关键字来解决：

    ```cpp
    class Derived : private Base{
        public:
            using Base::size;
        protect:
            using Base::n;
    }
    ```

通过这样的写法，使用Derived的用户可以直接访问Base的size成员，即使是private继承的。同时Derived的派生类可以访问n这个成员。

12. 同成员访问说明符一样，struct和class在继承的时候也是按照默认的权限继承的，例如所有的class默认都是private继承，所有的struct默认都是public继承。
13. 因为名字查找永远优先于类型检查，所以当名字相同的时候会出现下面的问题：

    ```cpp
    struct Base{
        int memfcn();
    }
    struct Derived:Base{
        int memfcn(int);
    }
    Derived d;
    d.memfcn()//错误，因为参数列表为空的memfcn被隐藏掉了。正确的用法是d.Base::memfcn()
    ```

14. 虚析构函数：

    ```cpp
    class Quote{
        virtual ~Quote() = default;
    }
    ```

如果一个类定义了虚析构函数，那么即使它通过=dafault的方式使用了默认的版本，编译器也不会为这个类定义默认的移动操作。

15. 基类和派生类的构造-析构顺序，假设A继承自B，B继承自C：
        构造顺序：A调用B的构造函数，B调用C的构造函数
                 C执行构造函数，B执行构造函数，最后A执行构造函数
        析构顺序：A销毁自己的成员，调用B的析构函数，B销毁自己的成员，调用C的析构函数
16. 派生类可以使用从父类继承来的构造函数：

    ```cpp
    class Bulk_quote : public Disc_quote{
        public:
        using Disc_quote:: Disc_quote//继承Disc_quote的构造函数
    }
    ```

其实类似于构造函数里面成员变量初始化的 Bulk_quote():Disc_quote(){}

17. 如果想要在vector既存放父类又存放派生类，如果使用vector<Base>，那么派生类的部分将会被扔掉，如果使用vector<child>那么父类无法存放。正确的做法应该是存放父类的指针：vector<shared_ptr<Base>>

## 十六、模板与泛型编程

1.
        template<typename T, typename U, class M>//必须使用class 或者typename，建议使用typename
        int compare(const T &v1, const T &v2){} 

2. 非类型模板参数：

    ```cpp
    template<unsigned N, unsigned M>
    int compare(const char (&p)[N], const char (&p2)[M])
    当我们调用compare("hi", "mom")的时候
    会生成int compare(const char (&p)[3], const char (&p2)[4])，考虑到末尾有一个空字符
    ```

3. 也可以是inline的,但是要在模板参数列表之后，返回类型之前：

    ```cpp
    template<typename T> inline T min(){}
    ```

4. 编写泛型代码的要求：模板中的函数参数是const的引用；函数体中的条件判断仅使用“<”或者"less"比较运算，这样可以少一种类型的支持，让代码运行更快，尽量减少对实参类型的要求
5. 当编译器遇到一个模板定义的时候，他并不生成代码，只有出现一个实例的时候才生成代码。而为了生成一个实例化版本，编译器需要掌握函数模板或者类模板成员函数的定义，所以
        模板的头文件通常既包括声明也包括定义
6. 模板类：

    ```cpp
    template <typename T> class Blob{
        Blob();
        Blob(std::initializer_list<T> il);
    }
    Blob<int> ia = {0, 1, 2, 3, 4};
    ```

7. 在模板类的内部的作用域内，我们可以直接使用模板名而不必指定模板实参
8. 在模板类中声明友元：

    ```cpp
    template <typename> class BlobPtr;
    template <typename> class Blob;
    template <typename T> bool operator==(const Blob<T>&, const Blob<T>&);
    template <typename T> class Blob{
        friend class BlobPtr<T>;
        friend bool operator==<T>(const Blob<T>&, const Blob<T>&);
    }
    
    这样友元的声明用Blob的模板形参作为他们自己的模板实参，friend的关系就被限定在相同类型实例化的Blob和BlobPtr相等运算符之间
    如果使用不同的模板参数，例如：friend bool operator==<typename X>();则所有的实例都将成为友元
    或者将模板类型参数声明为友元，例如friend T;
    ```

9. 模板的类型别名，除了传统的类型别名以外，还可以使用：

    ```cpp
    template<typename T> using partNo = pair<T, unsigned>;
    partNo<string> books; books是一个pair<string, unsigned>
    ```

10. 当我们希望通知编译器一个名字表示类型时，必须使用关键字typename而不是class：

    ```cpp
    return typename T::value_type();
    ```

11. 普通类和模板类的模板成员：

     ```cpp
     class DebugDelete{
         template<typename T> void operator()(T *p)const{delete p;}
     }
     double *p = new double; DebugDelete d;
     d(p);//通过类来进行delete
     template <typename T> class Blob{
         template<typename It>Blob(It b, It e);
     }
     ```

12. 显式实例化：

     ```cpp
     extern template class Blob<string> //实例化声明
     template int compare(const int&, const int&) //实例化定义
     这样可以避免在多个文件中实例化相同的模板，避免不必要的开销。
     对于每一个实例化的声明，在程序中某个位置必须有其显式的实例化定义
     ```

13. 顶层（top）const在模板中的实参转换通常会被忽略：

     ```cpp
     template <typename T> T fobj(T, T);
     template <typename T> T fref(const T&, const T&);
     int a[10], b[42];
     fobj(a, b);//调用f(int*, int*);
     fref(a, b);//错误，数据类型不匹配
     ```

14. 显示模板参数：例如

     ```cpp
     template<typename T1, typename T2, typename T3>
                     T1 sum(T2, T3);
     使用的时候：auto val = sum<long long>(i, j);//这里的long long指的是T1
     三个参数必须从左向右匹配：如果是这样写的：
                     template<typename T1, typename T2, typename T3>
                     T3 sum2(T2, T1);
     则必须制定所有三个模板，因为无法只使用一个的时候无法确定到底是T1还是T3，所以一定要按照顺序来
                     用的时候只能：auto val = sum2<long long, int, long>(i, j);
                 其中 long long指的是T1，int指的是T2， long是T3
     ```

15. 组合使用类型转换模板remove_reference、尾置返回、decltype，我们可以在函数中返回元素值的拷贝：

     ```cpp
     template <typename It>
     auto fcn2(It beg, It end) -> typename remove_reference<decltype(*beg)>::type{return *beg;}
     ```

16. 函数指针实参推断：

     ```cpp
     template <typename T> int compare(const T&, const T&);
     int (*pf1)(const int&, const int&) = compare;//pf1指向实例int compare(const int&, const int&);
     void func(int(*)(const int&, const int&));
     void func(int(*)(const string&, const string&));
     func(compare<int>);;//显式的指出实例化哪一个compare
     ```

17. 引用折叠：在正常绑定规则之外有两个例外规则：

     ```cpp
     (1). template <typename T>void f3(T&&);
     当我们将一个左值（如i）传递给函数的右值引用参数，且此右值引用指向模板类型参数（如T&&)时，编译器推断模板类型参数为实参的左值引用类型，因此当我们调用f3(i)的时候，编译器推断T的类型为int&而不是int
     (2). 如果简介创建一个引用的引用，（不是直接使用&&，而是通过类型（1）的这种转换创建的，类型别名或者模板参数），则这些引用形成了折叠，在所有情况下，引用会折叠成一个普通的左值引用类型。
     X& &， X& &&和X&& X都折叠成X&，X&& &&折叠成X&&
     这就会引起非常多的问题，例如调用f3(42);因为是右值，所以实际上里面T为int，而调用f3(i);时，T是int&，就可能改变i的值。所以正确的做法是重载一下：
     template <typename T>void f(T&&);
     template <typename T>void f(const T&);
     ```

18. 通过引用折叠实现标准库move函数;

     ```cpp
     template <typename T>
     typename remove_reference<T>::type&& move(T&& t){
         return static_cast<typename remove_reference<T>::type&&>(t);
     }
     请自己分析一下调用 std::move(string("bye"))和std::move(s);在move里面的步骤//具体答案在p611
     这种方法可以同时适配左值和右值，根据具体情况来具体指示到底是左值版本还是右值版本。
     ```

19. forward关键字：可以保持原始实参的类型，保存实参类型的所有细节：

     ```cpp
     template <typename F, typename T1, typename T2>
     void flip(F f, T1 &&t1, T2 &&t2){
         f(std::forward<T2>(t2), std::forward<T1>(t1));
     }
     
     此时如果我们调用flip(g, i, 42);i将以int&类型传递给g，42将以int&&类型传递给g
     ```

20. 当重载和模板同时发生的时候，编译器会选择非模板的版本，因此当两个模板都可以精确匹配某一次调用的时候，为了保证精确调用某一个模板，建议声明一个非模板版本（当然我觉得这里尽量不要出现这种情况，，，这样代码的可读性也太emmmm）
21. 可变参数模板：

     ```cpp
     template <typename T, typename... Args>
     void foo(const T &t, const Args&, ...rest);
     当使用foo(i, s, 42, d);的时候，编译器会生成void foo(const int&, const string&, const int&, const double&);的版本
     也可以使用sizeof...(Args)（表示类型参数的数目） 和sizeof...(rest)（表示函数参数的数目） 
     ```

22. 可变参数函数在调用的时候通常是递归的，例如上面那个，会首先调用foo(i,s,42,d),然后调用(i,42,d),以此类推
23. emplace_back实际上是一个可变参数模板，然后内部进行了转发参数的操作：

     ```cpp
     template <class...Args> inline
     void StrVec::emplace_back(Args&&... args){
         chk_n_alloc();
         alloc.construct(first_free++, std::forward<Args>(args)...);
     }
     实际上调用emplace_back(10, 'c')的时候会扩展出std::forward<int>(10), std::forward<char>(c)
     ```

24. 函数和类模板的特例化：

    ```cpp
    template<typename T>int compare(const T&, const T&);
    template <> int compare(const char* const &p1, const char* const &p2);
    ```

## 十七、标准库特殊设施

1. tuple类型：类似于pair的模板，但是一个tuple可以有任意数量的成员。其实tuple的作用有些类似于class和struct

    ```cpp
    tuple<string, vector<double>, int, list<int>> someVal("constants", {3.14, 2.71}, 42, {0, 1, 2, 3});
    tuple的初始化要么使用直接初始化方法，要么使用make_tuple方法生成
    获取tuple的成员：auto book = get<0>(someVal);
    如果不知道tuple准确的类型，可以使用decltype，tuple_element和tuple_size：
    size_t sz = tuple_size<decltype(someVal)>::value;//返回4
    tuple_element<1, decltype(someVal)>::type cnt = get<1>(someVal);//cnt是一个vector
    ```

2. bitset类型：是一个能够处理最长整型类型大小的位集合：

    ```cpp
    bitset<32> bitvec(1U);//低位为1，其他为0，编号从0开始的二进制位是低位。31为高位
    用string初始化bitset的话，正好和string的下标相反：
    bitset<32> bitvec4("1100");2,3位为1，剩余两位为0；
    ```

3. 正则表达式（regex类）：

    ```cpp
    regex r("[[:alpha:]]*" + "[^c]ei" + "[[:alpha:]]*");
    smatch results;
    if(regex_search(test_str, results, r)){cout<<results.str()<<endl;}
    因为正则表达式是在运行的时候编译的，所以效率非常的慢，应该尽可能少的使用正则表达式
    ```

4. 正则表达式迭代器：

    ```cpp
    for(sregex_iterator it(file.begin(), file.end(), r), end_it; it != end_it; ++it){
        cout<<it->str<<endl;//输出所有匹配的单词
        it->prefix().length()//前缀大小
        it->suffix().str().substr(0, 40);//后缀的一部分
    }
    ```

5. 和rand不一样的随机数引擎类：

    ```cpp
    default_random_engine e;
    e.seed(time(0));//可以不设置
    for (size_t i = 0; i < 10; i++){
        cout << e() <<endl;
    }
    ```

6. 随机数分布类：

      ```cpp
      uniform_int_distribution<unsigned> u(0, 9);
      default_random_engine e;
      for(size_t i = 0; i < 10; ++i){
          cout << u(e) <<
      }
      ```
      这些类应该全部定义成static的，这样才能保证每次调用返回的结果不一样。

7. IO库：改变输入输入格式的状态

    ```cpp
    cout<< boolalpha << true << " " << false <<endl;
    ```

会输出 true false 而不是 1 0，这时候要用cout<< noboolalpha才行。类似的还有hex，oct，dec等。以及在cout上面的cout.precision(12); 

8. 底层的未格式化IO操作（注意，这些底层的操作容易出错）：

    ```cpp
    cout.put(ch)//单字节操作，可以保留空白字符的
    peek返回下一个字符的副本，但是不会从输入流中把这个字符删除掉
    unget使输入流向后移动，从而最后读取的值又回到流中//不过这些操作的返回都是int类型的，方便判断是否是文件结尾
    cin.getline等操作是多字节IO操作，可以比较快速的输入
    ```

## 十八、用于大型程序的工具  

1. 栈展开（stack unwinding）：在抛出一个异常的时候，程序会暂停当前函数的执行并且找对应的catch，如果找不到，就继续检查外层的catch。在展开的过程当中，因为调用链上的语句会提前退出，所以调用链上的局部对象可能会销毁掉。同时如果一些需要手动释放的资源在释放之前发生了异常，那么这些资源将不会释放，我们写代码的时候需要注意
2. 异常的重新抛出：当一块catch语句语法解决某个异常的时候，可以将这个异常抛出给上一层函数处理：

    ```cpp
    catch(my_error &eObj){
        eObj.status = errCOdes::servereErr;
        throw;抛给上一层
    }
    ```

3. 捕获所有异常：

    ```cpp
    catch(...){    }
    ```

4. 处理构造函数初始值抛出的异常

    ```cpp
    template <typename T>
    Blob<T>::Blob(std::initializer_list<T> il)
    try:
    data(std::make_shared<std::vector<T>> (il)){
    }
    catch(const std::bad_alloc &e){
        handle_out_of_memory(e);
    }
    ```

5. noexcept关键字：
    告诉程序这个地方不会抛出异常（即使抛出异常也会终止整个程序）

    ```cpp
    void recoup(int) noexcept;
void recoup(int) throw();和上面是等价的。
    ```

    判断一个函数是不是会抛出异常的：
    
```cpp
    noexcept(recoup(i));//返回true
    void f() noexcept(noexcept(g()));//让f和g的异常说明一致
```

    noexcept说明符会影响函数指针的使用，例如加入noexcept的函数指针和没有加入noexcept的函数指针不能相等
6. 定义命名空间：

    ```cpp
    namespace cplusplus_primer{
        class tempClass{
        };
}//是可以没有分号的
    ```
    
    注意不要随便写namespace xxx,会导致名字混乱
7. 模板特例化：模板特例化必须定义在原始模板所属的命名空间中

    ```cpp
    namespace std{
        template<> struct hash<Sales_data>;
    }
    ```

    模板特例化以后，然后就可以在命名空间外部定义它了

    ```cpp
    template<> struct std::hast<Sales_data>{}
    ```

8. 嵌套的命名空间：

    ```cpp
    namespace cplusplus{
        namespace QueryLib{
    
        }
        QueryLib::XXX//前面需加内部命名空间名
    }
    cplusplus::QueryLib::XXX
    ```

9. 内联命名空间：

    ```cpp
    namespace cplusplus{
        inline namespace FifthEd{//必须出现在命名空间第一次定义的地方
            XXX;
        }
    }
    cplusplus::XXX;//可以不写内部内联的命名空间名
    ```

10. 未命名的命名空间：
    未命名的空间只在一个文件中起作用，不同文件中的为命名空间互相不冲突。所以一个文件中只能有一个未命名空间，且他本身类似于静态的作用

    ```cpp
    int i;
    namespace{
        int i;
    }
    i = 10;//错误，因为i的定义即出现在全局作用域中，又出现在未嵌套的未命名的命名空间中
    namespace local{
        namespace{
            int i;
        }
    }
    local::i = 42//正确，定义在嵌套的未命名的命名空间中的i与全局作用域的i不同
    ```

11. 代替using namespace，使用类型空间的别名：

     ```cpp
     namespace primer = cplusplus_primer;
     primer::XXX;
     ```

12. 友元声明和实参相关的查找：

     ```cpp
     namespace A{
         class C{
             friend void f2();//没有形参，很可能找不到
             friend void f(const C&);//根据实参相关的查找规则可以被找到
         }
     }
     A::C obj;
     f(obj);//通过在A::C中的友元声明和obj实参找到f
     f2();//找不到，没有形参
     ```

13. 多继承,多继承的类包含所有基类的成员，但是基类的构造函数形参不能相同：

     ```cpp
     class Bear:public ZooAnimal{
         Bear(const string&);
     }
     class Endangered{
         Endangered(const string&);
     }
     class Panda:public Bear, public Endangered{
         因为两个基类构造函数形参相同，所以必须重新定义构造函数
     }
     ```

14. 多继承的二义性，当两个基类有相同的成员名字，将会引发二义性，建议定义一个全新的版本
15. 虚继承：对某个类作出声明，承诺愿意共享他的基类，其中共享的基类称为虚基类

    ```cpp
    class Raccoon : public virtual ZooAnimal{}
    class Bear    : virtual public ZooAnimal{}
    class Panda   : Public Bear, public Raccoon, public Endangered{}
    ```

在普通情况下，如果不是虚继承的话，Panda将会有两份ZooAnimal，现在虚继承下，就只会有一份ZooAnimal了
这个时候当我们创建一个Panda对象的时候，首先构造虚基类ZooAnimal，接下来构造Bear，然后构造Raccoon，然后构造第三个直接基类Endangered，最后构造Panda。因为虚基类总是先于非虚基类构造，和他们继承的顺序无关。多个虚基类同时存在的时候，则按照顺序来

举例来说：假如类A和类B各自从类X派生（非虚继承且假设类X包含一些数据成员），且类C同时多继承自类A和B，那么C的对象就会拥有两套X的实例数据（可分别独立访问，一般要用适当的消歧义限定符）。但是如果类A与B各自虚继承了类X，那么C的对象就只包含一套类X的实例数据。

## 十九、特殊工具和技术  

1. new和delete的实现步骤：

    ```cpp
    string *sp = new string("a value");
    1. new表达式调用一个名为operator new（或者operator new[]）的标准库函数
    2. 标准库函数分配一块足够大的、原始的、未命名的内存空间以便存储特定类型的对象（或者对象的数组）
    3. 编译器运行相应的构造函数以构造这些对象，并为其传入初始值
    4. 对象被分配了空间并且构造完成，返回一个指向该对象的指针。
    delete sp;
    1. 对sp所致的对象或者数组执行对应的析构函数
    2. 编译器调用名为operator delete（或者delete[]） 的标准库释放内存空间 
    ```

2. 自定义自己的new和delete：必须使用noexcept保证他不会出错，返回类型必须是void\*，第一个形参必须是size_t类型且不能包括默认实参

    ```cpp
    void *operator new(size_tw size){
        if(void*mem==malloc(size)){
            return mem;
        }
        else{   }
    }
    ```

3. 显示调用析构函数：

      ```cpp
      string *sp = new string("a value");
      sp->~string();
      跟destroy(类似)
      ```

4. 运行时类型识别（RTTI功能）由两个运算符实现：

    ```cpp
    typeid运算符，用于返回表达式的类型
    dynamic_cast运算符,用于将基类的指针或者引用安全的转换成派生类的指针或引用
    假如Base类至少含有一个虚函数，Derived是Base的public派生类，如果有一个指向Base的指针bp，则我们可以在运行时将它转换成指向Derived的指针：
    if(Derived *dp = dynamic_cast<Derived*> (bp)){}
    if(typeid(*bp) == typeid(*dp)){}//表达具体是什么类型，typeid作用于对象，所以我们用*bp
    ```
5. 给枚举指定类型：
        
    ```cpp
    enum intValues:unsigned long long{
        charTyp = 255;.......
    }
    以及可以像类一样进行提前声明
enum class IntValues{}//限定作用于的枚举类型，在括号外如果没有对象的话就不可访问
   ```
   
6. 枚举类型的形参匹配：
        
    ```cpp
    enum Tokens{INLINE = 128, VIRTUAL = 129};
    void ff(Tokens);
    void ff(int);
    int main(){
        Tokens curTok = INLINE;
        ff(128);   //匹配int
        ff(INLINE);//匹配ff(Tokens)实参匹配！！
        ff(curTok);//匹配ff(Tokens)
    }
    ```
7. 类成员指针：使用->\*和.\*来进行访问
8. 与function关键字类似的：mem_fn标准库功能，可以自动推断可调用对象的类型;
        
   
    find_if(svec.begin(), svec.end(), mem_fn(&string::empty));
   
9. union:一种节省空间的类，其中可以有多个数据成员，但是在任意时刻只能有一个数据成员有值，给他某个成员赋值以后，其他的成员会变成未定义的，union不能含有引用类型的成员。因为union不能作为基类或者继承自其他类，所以他也没有虚函数。

而匿名union则是一个未命名的union，并且在右花括号和分号之间没有任何声明，一旦我们定义了一个匿名union，编译器就自动的为该union创建一个未命名的对象,在匿名union定义所在的作用域内，该union里面的成员都是可以直接访问的。

```cpp
    union{
        char cval;
        int ival;
        double dval;
    };
    cval = 'c';
    ival = 42;
```

10. 使用类来管理union：
当union包含内置类型的成员时，我们可以使用普通的赋值语句改变union保存的值，但是当含有特殊类类型成员的union就没那么简单了，我们如果需要将union的值改为类类型成员对应的值，就需要构造或者析构该类类型的成员

同时如果是包含内置类型的成员，编译器将按照成员的次序依次生成默认构造函数或者拷贝控制成员。

因为有些复杂，所以通常将union放在类里面而不是类放在union里面

```cpp
    class Token{
    public:
            //因为union含有一个string这样的类成员，所以Token必须定义拷贝控制成员
            Token() : tok(INT), ival{0}{

            }
            Token(const Token &t) : tok(t.tok){
                    copyUnion(t);
            }
            Token &operator=(const Token&);
            //如果类内有一个string这样的类成员，我们必须销毁他
            ~Token(){
                    if(tok==STR)
                        sval.~string();
            }
            //下面的赋值运算符负责设置union的不同成员
            Token &operator=(const std::string&);
            Token &operator=(char);
            Token &operator=(int);
            Token &operator=(double);
    private:
            enum{INT, CHAR, DBL, STR} tok;//判别式，用来辨认union存储的值
            union{
                    char cval;
                    int ival;
                    double dval;
                    std::string sval;
            };//每个Token对象含有一个该未命名Union类型的未命名成员
            void copyUnion(const Token&);
    };

    类的赋值运算符将负责设置tok并且为union的相应成员赋值，和析构函数一样，这些运算符在为union赋新值前必须首先销毁string：

    Token &Token::operator=(int i){
            //如果union的当前值是string，那么我们必须先调用string的析构函数销毁这个string，然后才能赋新值
            if(tok == STR){
                    sval.~string();
            }
            ival = i;
            tok = INT;
            return *this;
    }
```

11. 局部类无法访问其外层作用域的局部变量等
12. 位域：类可以将非静态的数据成员定义成位域(bit field)一个位域中含有一定数量的二进制位，位域的类型必须是整形或者枚举类型

    ```cpp
      typedef unsigned int Bit;
    class File{
            Bit mode : 2;
            Bit modified : 1;
            Bit prot_owner : 3;
            Bit prot_group : 3;
            Bit prot_world : 3;
    public:
            enum modes{READ = 01, WRITE = 02, EXECUTE = 03};
            ......  
    }
    ```
13. volatile 限定符：和const是一个性质的，不过具体含义和机器系统有关
14. 链接提示：extern"C":需要用到调用其他语言编写的函数的时候：

    ```cpp
    例如可能出现在C++头文件<cstring>中的链接提示;
    extern "C" size_t strlen(const char*);
    extern "C"{
            int strcmp(const char*, const char*);
    }
    ```
15. 导出C++到其他语言：

    ```cpp
    extern "C" double calc(double dparm){....}//将会为该函数生成适合于C的代码
    ```



