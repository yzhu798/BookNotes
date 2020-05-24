�这会导致易碎的且紧耦合的代码，而且很快将会成为维护的噩梦。任何代码如果把这些数据成员设值为无效的或者预料之外的值的组合，都可能搞坏对象以及随后对象的所有使用。

大多数的类要么都是 A，要么都是 B：

* *全 public*: 如果编写的是聚集一组变量而没有在这些变量之间维护不变式的话，所有这些变量都应当为 `public`。
  [依照惯例，应当把这样的类声明为 `struct` 而不是 `class`](#Rc-struct)
* *全 private*: 如果编写的类型维护了某个不变式，则所有的非 `const` 变量都应当是 `private`——应当对它进行封装。

##### 例外

偶尔会出现混合了 A 和 B 的类，通常是为了方便调试。一个被封装的对象可能包含如非 `const` 调试信息的某种东西，它并不是不变式的一部分，因此属于 A 类别——其实它并非对象的值的一部分，也不是这个对象的有意义的可观察状态。这种情况下，A 类别的部分应当按 A 的方式对待（使之 `public`，或罕见情况下当它们只应对派生类可见时，使之为 `protected`），而 B 类别的部分应当仍按 B 的方式对待（`private` 或 `const`）。

##### 强制实施

对包含具有不同访问级别的非 `const` 数据成员的类给出标记。

### <a name="Rh-mi-interface"></a>C.135: 用多继承来表达多个不同的接口

##### 理由

并非所有的类都应当支持全部的接口，而且并非所有的调用方都应当处理所有的操作。
尤其应当把巨大的接口拆解为可以被某个给定派生类所支持的不同的行为"方面"。

##### 示例

    class iostream : public istream, public ostream {   // 充分简化
        // ...
    };

`istream` 提供了输入操作的接口；`ostream` 提供了输出操作的接口。
`iostream` 提供了 `istream` 和 `ostream` 接口的并集，以及在单个流上同时允许二者所需的同步操作。

##### 注解

这是非常常见的继承的用法，因为需要同一个实现提供多个不同接口是很常见的，
而通常把这种接口组织到一个单根层次中都是不容易的或者不自然的。

##### 注解

这样的接口通常都是抽象类。

##### 强制实施

???

### <a name="Rh-mi-implementation"></a>C.136: 用多继承来表达一些实现特性的合并

##### 理由

一些形式的混元（Mixin）带有状态，并通常会有对其状态提供的操作。
如果这些操作是虚的，就必须进行继承，而如果不是虚的，使用继承也可以避免例行代码和转发函数。

##### 示例

      class iostream : public istream, public ostream {   // 充分简化
        // ...
    };

`istream` 提供了输入操作的接口（以及一些数据）；`ostream` 提供了输出操作的接口（以及一些数据）。
`iostream` 提供了 `istream` 和 `ostream` 接口的并集，以及在单个流上同时允许二者所需的同步操作。

##### 注解

这是一种相对少见的用法，因为实现通常都可以被组织到一个单根层次之中。

##### 示例

有时候，"实现特性"更像是"混元"，决定实现的行为，
并向其中注入成员以使该实现提供其所要求的策略。
相关的例子可以参考 `std::enable_shared_from_this`
或者 boost.intrusive 中的各种基类（例如 `list_base_hook` 和 `intrusive_ref_counter`）。

##### 强制实施

???

### <a name="Rh-vbase"></a>C.137: 用 `virtual` 基类以避免过于通用的基类

##### 理由

允许共享的数据和接口的分离。
避免将所有共享数据都被放到一个终极基类之中。

##### 示例

    struct Interface {
        virtual void f();
        virtual int g();
        // ... 没有数据 ...
    };
    
    class Utility {  // 带有数据
        void utility1();
        virtual void utility2();    // 定制点
    public:
        int x;
        int y;
    };
    
    class Derive1 : public Interface, virtual protected Utility {
        // 覆盖了 Iterface 的函数
        // 可能覆盖 Utility 的虚函数
        // ...
    };
    
    class Derive2 : public Interface, virtual protected Utility {
        // 覆盖了 Iterface 的函数
        // 可能覆盖 Utility 的虚函数
        // ...
    };

如果许多派生类都共享了显著的"实现细节"，弄一个 `Utility` 出来就是有意义的。


##### 注解

很明显，这个例子过于"理论化"，但确实很难找到一个*小型*的现实例子出来。
`Interface` 是一个[接口层次](#Rh-abstract)的根，
而 `Utility` 则是一个[实现层次](#Rh-kind)的根。
[一个稍微更现实的例子](https://www.quora.com/What-are-the-uses-and-advantages-of-virtual-base-class-in-C%2B%2B/answer/Lance-Diduck)，有一些解释。

##### 注解

将层次结构线性化通常是更好的方案。

##### 强制实施

对接口和实现混合的层次进行标记。

### <a name="Rh-using"></a>C.138: 用 `using` 来为派生类和其基类建立重载集合

##### 理由

没有 using 声明的话，派生类的成员函数将会隐藏全部其所继承的重载集合。

##### 示例，不好

    #include <iostream>
    class B {
    public:
        virtual int f(int i) { std::cout << "f(int): "; return i; }
        virtual double f(double d) { std::cout << "f(double): "; return d; }
        virtual ~B() = default;
    };
    class D: public B {
    public:
        int f(int i) override { std::cout << "f(int): "; return i + 1; }
    };
    int main()
    {
        D d;
        std::cout << d.f(2) << '\n';   // 打印 "f(int): 3"
        std::cout << d.f(2.3) << '\n'; // 打印 "f(int): 3"
    }

##### 示例，好

    class D: public B {
    public:
        int f(int i) override { std::cout << "f(int): "; return i + 1; }
        using B::f; // 展露了 f(double)
    };

##### 注解

这个问题对虚的和非虚的成员函数都有影响。

对于可变基类，C++17 引入了一种 using 声明的可变形式：

    template <class... Ts>
    struct Overloader : Ts... {
        using Ts::operator()...; // 展露了每个基类中的 operator()
    };

##### 强制实施

诊断名字隐藏情况

### <a name="Rh-final"></a>C.139: `final` 的运用应当保守

##### 理由

用 `final` 来把类层次封闭很少是由于逻辑上的原因而必须的，并可能破坏掉类层次的可扩展性。

##### 示例，不好

    class Widget { /* ... */ };
    
    // 没有人会想要改进 My_widget（你可能这么觉得）
    class My_widget final : public Widget { /* ... */ };
    
    class My_improved_widget : public My_widget { /* ... */ };  // 错误: 办不到了

##### 注解

并非每个类都要作为基类的。
大多数标准库的类都是这样（比如，`std::vector` 和 `std::string` 都并非为了派生而设计）。
这条规则是关于在准备作为某个类层次的接口的带有虚函数的类中有关 `final` 的使用的。

##### 注解

用 `final` 来把单个的虚函数封印则是易错的，因为在定义和覆盖一组函数时，`final` 是很容易被忽视的。
幸运的是，编译器能够捕捉到这种错误：你无法在派生类中重新声明/重新打开一个 `final` 成员。

##### 注解

有关 `final` 带来的性能提升的断言是需要证实的。
非常常见的是，这种断言都是基于推测或者在其他语言上的经验而来的。

有一些例子中的 `final` 对于逻辑和性能因素来说可能都是重要的。
一个例子是编译器和语言分析工具中的性能关键的 AST 层次结构。
其中并非每年都会添加新的派生类，而且只有程序库的实现者会做这种事。
不过，误用（或者至少曾经的误用）的情况远远比这常见。

##### 强制实施

标记出 `final` 的所有使用。


### <a name="Rh-virtual-default-arg"></a>C.140: 不要在虚函数和其覆盖函数上提供不同的默认参数

##### 理由

这会造成混乱：覆盖函数是不会继承默认参数的。

##### 示例，不好

    class Base {
    public:
        virtual int multiply(int value, int factor = 2) = 0;
        virtual ~Base() = default;
    };
    
    class Derived : public Base {
    public:
        int multiply(int value, int factor = 10) override;
    };
    
    Derived d;
    Base& b = d;
    
    b.multiply(10);  // 这两次调用将会调用相同的函数，但分别
    d.multiply(10);  // 使用不同的默认实参，因此获得不同的结果

##### 强制实施

对虚函数默认参数中，如果其在基类和派生类的声明中是不同的，则对其进行标记。

## C.hier-access: 对类层次中的对象进行访问

### <a name="Rh-poly"></a>C.145: 通过指针和引用来访问多态对象

##### 理由

如果类带有虚函数的话，你（通常）并不知道你所使用的函数是由哪个类提供的。

##### 示例

    struct B { int a; virtual int f(); virtual ~B() = default };
    struct D : B { int b; int f() override; };
    
    void use(B b)
    {
        D d;
        B b2 = d;   // 切片
        B b3 = b;
    }
    
    void use2()
    {
        D d;
        use(d);   // 切片
    }

两个 `d` 都发生了切片。

##### 例外

你可以安全地访问处于自身定义的作用域之内的具名多态对象，这并不会将之切片。

    void use3()
    {
        D d;
        d.f();   // OK
    }

##### 另见

[多态类应当抑制复制操作](#Rc-copy-virtual)

##### 强制实施

对所有切片都进行标记。

### <a name="Rh-dynamic_cast"></a>C.146: 当无法避免在类层次上进行导航时应使用 `dynamic_cast`

##### 理由

`dynamic_cast` 是运行时检查。

##### 示例

    struct B {   // 接口
        virtual void f();
        virtual void g();
        virtual ~B();
    };
    
    struct D : B {   // 更宽的接口
        void f() override;
        virtual void h();
    };
    
    void user(B* pb)
    {
        if (D* pd = dynamic_cast<D*>(pb)) {
            // ... 使用 D 的接口 ...
        }
        else {
            // ... 通过 B 的接口做事 ...
        }
    }

使用其他的强制转换可能会违反类型安全，导致程序中所访问的某个真实类型为 `X` 的变量，被当做具有某个无关的类型 `Z` 而进行访问：

    void user2(B* pb)   // 不好
    {
        D* pd = static_cast<D*>(pb);    // I know that pb really points to a D; trust me
        // ... 使用 D 的接口 ...
    }
    
    void user3(B* pb)    // 不安全
    {
        if (some_condition) {
            D* pd = static_cast<D*>(pb);   // I know that pb really points to a D; trust me
            // ... 使用 D 的接口 ...
        }
        else {
            // ... 通过 B 的接口做事 ...
        }
    }
    
    void f()
    {
        B b;
        user(&b);   // OK
        user2(&b);  // 糟糕的错误
        user3(&b);  // OK，*如果*程序员已经正确检查了 some_condition 的话
    }

##### 注解

和其他强制转换一样，`dynamic_cast` 被过度使用了。
[优先使用虚函数而不是强制转换](#Rh-use-virtual)。
如果可行（无须运行时决议）并且相当便利的话，优先使用[静态多态](#???)而不是
继承层次的导航。

##### 注解

一些人会在 `typeid` 更合适的时候使用 `dynamic_cast`；
`dynamic_cast` 是一个通用的"是一个"操作，用以发现对象上的最佳接口，
而 `typeid` 是"报告对象的精确类型"操作，用以发现对象的真实类型。
后者本质上就是更简单的操作，因而应当更快一些。
后者（`typeid`）是可以在需要时进行手工模仿的（比如说，当工作在（因为某种原因）禁止使用 RTTI 的系统上），
而前者（`dynamic_cast`）要正确地实现则要困难得多。

考虑：

    struct B {
        const char* name {"B"};
        // 若 pb1->id() == pb2->id() 则 *pb1 与 *pb2 类型相同
        virtual const char* id() const { return name; }
        // ...
    };
    
    struct D : B {
        const char* name {"D"};
        const char* id() const override { return name; }
        // ...
    };
    
    void use()
    {
        B* pb1 = new B;
        B* pb2 = new D;
    
        cout << pb1->id(); // "B"
        cout << pb2->id(); // "D"


        if (pb1->id() == "D") {         // 貌似没问题
            D* pd = static_cast<D*>(pb1);
            // ...
        }
        // ...
    }

`pb2->id == "D"` 的结果实际上是由实现定义的。
我们加上这个是为了警告自造的 RTTI 中的危险之处。
这个代码可能可以多年正常工作，但只在不会对字符字面量进行唯一化的新机器，新编译器，或者新的连接器上失败。

当实现你自己的 RTTI 时，请当心这一点。

##### 例外

如果你所用的实现提供的 `dynamic_cast` 确实很慢的话，你可能只得使用一种替代方案了。
不过，所有无法静态决议的替代方案都设计显式的强制转换（通常为 `static_cast`），而且易于出错。
你基本上都将会创建自己的特殊目的 `dynamic_cast`。
因此，首先应当确定你的 `dynamic_cast` 确实和你想的一样慢（其实有相当多的并不被支持的流言），
而且你使用的 `dynamic_cast` 确实是性能十分关键的。

我们的观点是，当前的实现中的 `dynamic_cast` 并非很慢。
比如说，在合适的条件下，是可以以[快速常量时间](http://www.stroustrup.com/fast_dynamic_casting.pdf)来实施 `dynamic_cast` 的。
但是，兼容性问题使得作出改变很难，虽然大家都同意对优化付出的努力是值得的。

在很罕见的情况下，如果你以及测量出 `dynamic_cast` 的开销确实有影响，你也有其他的方式来静态地保证向下转换的成功（比如说小心地应用 CRTP 时），而且其中并不涉及虚继承的话，可以考虑战术性地使用 `static_cast` 并带上显著的代码注释和免责声明，概述这个段落，而且由于类型系统无法验证其正确性而在维护中需要人工的关切。即便是这样，以我们的经验来说，这种"我知道我在干什么"的情况仍然是一种已知的 BUG 来源。

##### 例外

考虑：

    template<typename B>
    class Dx : B {
        // ...
    };

##### 强制实施

* 对所有用 `static_cast` 来进行向下转换进行标记，其中也包括实施 `static_cast` 的 C 风格的强制转换。
* 本条规则属于[类型安全性剖面配置](#Pro-type-downcast)。

### <a name="Rh-ref-cast"></a>C.147: 当查找所需类的失败被当做一种错误时，应当对引用类型使用 `dynamic_cast`

##### 理由

对引用进行的强制转换，所表达的意图是最终会获得一个有效对象，因此这种强制转换必须成功。如果无法成功的话，`dynamic_cast` 将会抛出异常。

##### 示例

    ???

##### 强制实施

???

### <a name="Rh-ptr-cast"></a>C.148: 当查找所需类的失败被当做一种有效的可能情况时，应当对指针类型使用 `dynamic_cast`

##### 理由

`dynamic_cast` 转换允许测试指针是否指向某个其类层次中包含给定类的多态对象。由于其找不到类时仅会返回空值，因而可以在运行时予以测试。这允许编写的代码可以基于其结果而选择不同的代码路径。

与此相对，[C.147](#Rh-ptr-cast) 中失败即是错误，而且不能用于条件执行。

##### 示例

下面的例子的 `Shape_owner` 的 `add` 函数获取构造的 `Shape` 对象的所有权。这些对象也根据其几何特性被存储到了不同视图中。
这个例子中的 `Shape` 并不继承于 `Geometric_attributes`，而是其各个子类继承。

    void add(Shape * const item)
    {
      // 总是获得其所有权
      owned_shapes.emplace_back(item);
    
      // 检查 Geometric_attribute 并将该形状加入到（零个/一个/某些/全部）视图中
    
      if (auto even = dynamic_cast<Even_sided*>(item))
      {
        view_of_evens.emplace_back(even);
      }
    
      if (auto trisym = dynamic_cast<Trilaterally_symmetrical*>(item))
      {
        view_of_trisyms.emplace_back(trisym);
      }
    }

##### 注解

找不到所需的类时 `dynamic_cast` 将返回空值，而解引用空指针将导致未定义的行为。
因此 `dynamic_cast` 的结果应当总是当做可能包含空值并进行测试。

##### 强制实施

* 【复杂】 除非在指针类型 `dynamic_cast` 的结果上进行了空值测试，否则就对该指针的解引用给出警告。

### <a name="Rh-smart"></a>C.149: 用 `unique_ptr` 或 `shared_ptr` 来避免忘记对以 `new` 所创建的对象进行 `delete` 的情况

##### 理由

避免资源泄漏。

##### 示例

    void use(int i)
    {
        auto p = new int {7};           // 不好: 用 new 来初始化局部指针
        auto q = make_unique<int>(9);   // ok: 保证了为 9 所分配的内存会被回收
        if (0 < i) return;              // 可能会返回并泄漏
        delete p;                       // 太晚了
    }

##### 强制实施

* 标记用 `new` 的结果来对裸指针所进行的初始化。
* 标记对局部变量的 `delete`。

### <a name="Rh-make_unique"></a>C.150: 用 `make_unique()` 来构建由 `unique_ptr` 所拥有的对象

##### 理由

`make_unique` 为构造提供了更精炼的语句。
它也保证了复杂表达式中的异常安全性。

##### 示例

    unique_ptr<Foo> p {new Foo{7}};    // OK: 不过有重复
    
    auto q = make_unique<Foo>(7);      // 有改善: 并未重复 Foo
    
    // 非异常安全: 编译器可能如下交错进行各参数的计算:
    //
    // 1. 为 Foo 分配内存
    // 2. 构造 Foo
    // 3. 调用 bar
    // 4. 构造 unique_ptr<Foo>
    //
    // 如果 bar 抛出了异常，Foo 就不会被销毁，而为其所分配的内存则会泄漏。
    f(unique_ptr<Foo>(new Foo()), bar());
    
    // 异常安全: 各函数调用无法互相交错。
    f(make_unique<Foo>(), bar());

##### 强制实施

* 标记模板特化列表 `<Foo>` 的重复使用。
* 标记声明为 `unique_ptr<Foo>` 的变量。

### <a name="Rh-make_shared"></a>C.151: 用 `make_shared()` 来构建由 `shared_ptr` 所拥有的对象

##### 理由

`make_shared` 为构造提供了更精炼的语句。
它也提供了一个机会，通过把 `shared_ptr` 的使用计数和对象相邻放置，来消除为引用计数进行独立的内存分配操作。

##### 示例

    void test() {
        // OK: 但出现重复；而且为这个 Bar 和 shared_ptr 的使用计数分别进行了分配
        shared_ptr<Bar> p {new Bar{7}};
    
        auto q = make_shared<Bar>(7);   // 有改善: 并未重复 Bar；只有一个对象
    }

##### 强制实施

* 标记模板特化列表 `<Bar>` 的重复使用。
* 标记声明为 `shared_ptr<Bar>` 的变量。

### <a name="Rh-array"></a>C.152: 禁止把指向派生类对象的数组的指针赋值给指向基类的指针

##### 理由

对所得到的基类指针进行下标操作，将导致无效的对象访问并且可能造成内存损坏。

##### 示例

    struct B { int x; };
    struct D : B { int y; };
    
    void use(B*);
    
    D a[] = {{1, 2}, {3, 4}, {5, 6}};
    B* p = a;     // 不好: a 衰变为 &a[0]，并被转换为 B*
    p[1].x = 7;   // 覆盖了 D[0].y
    
    use(a);       // 不好: a 衰变为 &a[0]，并被转换为 B*

##### 强制实施

* 对任何的数组衰变和基类向派生类转换之间的组合进行标记。
* 应当将数组作为 `span` 而不是指针来进行传递，而且不能让数组的名字在放入 `span` 之前经手派生类向基类的转换。


### <a name="Rh-use-virtual"></a>C.153: 优先采用虚函数而不是强制转换

##### 理由

虚函数调用安全，而强制转换易错。
虚函数调用达到全派生函数，而强制转换可能达到某个中间类
而得到错误的结果（尤其是当类层次在维护中被修改之后）。

##### 示例

    ???

##### 强制实施

参见 [C.146](#Rh-dynamic_cast) 和 ???

## <a name="SS-overload"></a>C.over: 重载和运算符重载

可以对普通函数，模板函数，以及运算符进行重载。
不能重载函数对象。

重载规则概览：

* [C.160: 定义运算符应当主要用于模仿传统用法](#Ro-conventional)
* [C.161: 对于对称的运算符应当采用非成员函数](#Ro-symmetric)
* [C.162: 重载的操作之间应当大体上是等价的](#Ro-equivalent)
* [C.163: 应当仅对大体上等价的操作进行重载](#Ro-equivalent-2)
* [C.164: 避免隐式转换运算符](#Ro-conversion)
* [C.165: 为定制点采用 `using`](#Ro-custom)
* [C.166: 一元 `&` 的重载只能作为某个智能指针或引用系统的一部分而提供](#Ro-address-of)
* [C.167: 应当为带有传统含义的操作提供运算符](#Ro-overload)
* [C.168: 应当在操作数所在的命名空间中定义重载运算符](#Ro-namespace)
* [C.170: 当想要重载 lambda 时，应当使用泛型 lambda](#Ro-lambda)

### <a name="Ro-conventional"></a>C.160: 定义运算符应当主要用于模仿传统用法

##### 理由

最小化意外情况。

##### 示例

    class X {
    public:
        // ...
        X& operator=(const X&); // 定义赋值的成员函数
        friend bool operator==(const X&, const X&); // == 需要访问其内部表示
                                                    // 执行 a = b 之后将有 a == b
        // ...
    };

这里维持了传统的语义：[副本之间相等](#SS-copy)。

##### 示例，不好

    X operator+(X a, X b) { return a.v - b.v; }   // 不好: 让 + 执行减法

##### 注解

非成员运算符应当要么是友元，要么定义于[其操作数所在的命名空间中](#Ro-namespace)。
[二元运算符应当等价地对待其两个操作数](#Ro-symmetric)。

##### 强制实施

也许是不可能的。

### <a name="Ro-symmetric"></a>C.161: 对于对称的运算符应当采用非成员函数

##### 理由

如果使用的是成员函数的话，就需要两个才行。
如果为（比如）`==` 采用的不是非成员函数的话，`a == b` 和 `b == a` 就会存在微妙的差别。

##### 示例

    bool operator==(Point a, Point b) { return a.x == b.x && a.y == b.y; }

##### 强制实施

标记成员运算符函数。

### <a name="Ro-equivalent"></a>C.162: 重载的操作之间应当大体上是等价的

##### 理由

让逻辑上互相等价的操作对不同的参数类型使用不同的名字会带来混乱，导致在函数名字中编码类型信息，并妨碍泛型编程。

##### 示例

考虑：

    void print(int a);
    void print(int a, int base);
    void print(const string&);

这三个函数都对其参数进行（适当的）打印。相对而言：

    void print_int(int a);
    void print_based(int a, int base);
    void print_string(const string&);

这三个函数也都对其参数进行（适当的）打印。在名字上附加仅仅增添了啰嗦程度，而且妨碍了泛型代码。

##### 强制实施

???

### <a name="Ro-equivalent-2"></a>C.163: 应当仅对大体上等价的操作进行重载

##### 理由

让逻辑上不同的函数使用相同的名字会带来混乱，并导致在泛型编程时发生错误。

##### 示例

考虑：

    void open_gate(Gate& g);   // 把车库出口通道的障碍移除
    void fopen(const char* name, const char* mode);   // 打开文件

这两个操作本质上就是不同的（而且没有关联），因此让它们的名字相区别是正确的。相对而言：

    void open(Gate& g);   // 把车库出口通道的障碍移除
    void open(const char* name, const char* mode ="r");   // 打开文件

这两个操作仍旧本质不同（且没有关联），但它们的名字缩减成了（共同的）最小词，并带来了发生混乱的机会。
幸运的是，类型系统能够识别出许多这种错误。

##### 注解

对于一般性的和流行的名字，比如 `open`、`move`、`+` 和 `==` 等等，应当特别小心。

##### 强制实施

???

### <a name="Ro-conversion"></a>C.164: 避免隐式转换运算符

##### 理由

隐式类型转换可以很基础（比如 `double` 向 `int`），但经常会导致意外（比如 `String` 到 C 风格的字符串）。

##### 注解

优先采用明确命名的转换，除非给出了一条很重要的需求。
我们所谓"很重要的需求"的意思是，它在其应用领域中是基本的（比如整数向复数的转换），
并且经常需要用到。不要仅仅为了微小的便利就（通过转换运算符或非 `explicit` 构造函数）
引入隐式的类型转换。

##### 示例

    struct S1 {
        string s;
        // ...
        operator char*() { return s.data(); } // 不好，可能会引起以外
    }
    
    struct S2 {
        string s;
        // ...
        explicit operator char*() { return s.data(); }
    };
    
    void f(S1 s1, S2 s2)
    {
        char* x1 = s1;     // 可以，但在许多情况下会引起意外
        char* x2 = s2;     // 错误，这通常是一件好事
        char* x3 = static_cast<char*>(s2); // 我们可以明确（在你的头脑里）
    }

可能在任意的难以发现的上下文里发生令人惊讶且可能具有破坏性的隐式转换，例如，

    S1 ff();
    
    char* g()
    {
        return ff();
    }

由`ff（）返回的字符串在返回它的指针之前被销毁。    

##### 强制实施

标记所有的转换运算符。

### <a name="Ro-custom"></a>C.165: 为定制点采用 `using`

##### 理由

以便找到在一个独立的命名空间中所定义的函数对象或函数，它们对一个一般性函数进行了"定制"。

##### 示例

考虑 `swap`。这是一个通用的（标准库）函数，其定义可以在任何类型上工作。
不过，通常需要为特定的类型定义专门的 `swap()`。
例如说，通用的 `swap()` 会对互相交换的两个 `vector` 的元素进行复制，而一个好的专门实现则完全不会复制任何元素。

    namespace N {
        My_type X { /* ... */ };
        void swap(X&, X&);   // 针对 N::X 所优化的 swap
        // ...
    }
    
    void f1(N::X& a, N::X& b)
    {
        std::swap(a, b);   // 可能不是我们所要的: 调用 std::swap()
    }

`f1()` 中的 `std::swap()` 严格做了我们所要求的工作：它调用了命名空间 `std` 中的 `swap()`。
不幸的是，这可能并非我们想要的。
怎样让其考虑 `N::X` 呢？

    void f2(N::X& a, N::X& b)
    {
        swap(a, b);   // 调用了 N::swap
    }

但这也不是我们在泛型代码中所要的。
通常如果专门的函数存在的话我们就想用它，否则的话我们则需要通用的函数。
这是通过在函数的查找中包含通用函数而达成的：

    void f3(N::X& a, N::X& b)
    {
        using std::swap;  // 使得 std::swap 可用
        swap(a, b);        // 如果存在 N::swap 则调用之，否则为 std::swap
    }

##### 强制实施

不大可能，除非是如 `swap` 这样的已知的定制点。
问题在于无限定和带限定的名字查找都有其作用。

### <a name="Ro-address-of"></a>C.166: 一元 `&` 的重载只能作为某个智能指针或引用系统的一部分而提供

##### 理由

`&` 运算符在 C++ 中很基本。
C++ 语义中的很多部分都假定了其默认的含义。

##### 示例

    class Ptr { // 一种智能指针
        Ptr(X* pp) :p(pp) { /* 检查 */ }
        X* operator->() { /* 检查 */ return p; }
        X operator[](int i);
        X operator*();
    private:
        T* p;
    };
    
    class X {
        Ptr operator&() { return Ptr{this}; }
        // ...
    };

##### 注解

如果你要"折腾"运算符 `&` 的话，请保证其定义和 `->`，`[]`，`*` 和 `.` 在结果类型上具有匹配的含义。
注意，运算符 `.` 现在是无法重载的，因此不可能做出一个完美的系统。
我们打算修正这一点： <http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2015/n4477.pdf>。
注意 `std::addressof()` 总会产生一个内建指针。

##### 强制实施

比较微妙。如果用户定义了 `&` 而并未同时为其结果类型定义 `->`，则进行警告。

### <a name="Ro-overload"></a>C.167: 应当为带有传统含义的操作提供运算符

##### 理由

可读性。约定。可重用性。支持泛型代码。

##### 示例

    void cout_my_class(const My_class& c) // 含糊，并非传统约定，非泛型
    {
        std::cout << /* 此处为类成员 */;
    }
    
    std::ostream& operator<<(std::ostream& os, const my_class& c) // OK
    {
        return os << /* 此处为类成员 */;
    }

对于其自身而言，`cout_my_class` 也许是没问题的，但它无法用在（或组合到）依赖于 `<<` 的输出用法约定代码之中：

    My_class var { /* ... */ };
    // ...
    cout << "var = " << var << '\n';

##### 注解

大多数运算符都有很强烈和活跃的含义约定用法，比如

* 比较：`==`，`!=`，`<`，`<=`，`>`，以及 `>=`
* 算术操作：`+`，`-`，`*`，`/`，以及 `%`
* 访问操作：`.`，`->`，一元 `*`，以及 `[]`
* 赋值：`=`

请勿定义违反约定的用法，请勿为它们发明自己的名字。

##### 强制实施

比较棘手。需要洞悉其语义。

### <a name="Ro-namespace"></a>C.168: 应当在操作数所在的命名空间中定义重载运算符

##### 理由

可读性。
通过 ADL 寻找运算符的能力。
避免不同命名空间中的定义之间不一致。

##### 示例

    struct S { };
    bool operator==(S, S);   // OK: 和 S 在相同的命名空间，甚至紧跟着 S
    S s;
    
    bool x = (s == s);

如果有默认的 == 的话，这正是默认的 == 所做的。

##### 示例

    namespace N {
        struct S { };
        bool operator==(S, S);   // OK: 和 S 在相同的命名空间，甚至紧跟着 S
    }
    
    N::S s;
    
    bool x = (s == s);  // 通过 ADL 找到了 N::operator==()

##### 示例，不好

    struct S { };
    S s;
    
    namespace N {
        S::operator!(S a) { return true; }
        S not_s = !s;
    }
    
    namespace M {
        S::operator!(S a) { return false; }
        S not_s = !s;
    }

这里的 `!s` 的含义在 `N` 和 `M` 中是不同的。
这可能是最易混淆的一点。
移除 `namespace M` 的定义之后，混乱就被一种犯错的机会所代替。

##### 注解

当为定义于不同命名空间的两个类型定义一个二元运算符时，无法遵循这条规则。
例如：

    Vec::Vector operator*(const Vec::Vector&, const Mat::Matrix&);

也许最好避免这种情形。

##### 参见

这是这条规则的一种特殊情况：[辅助函数应当定义于它们的类相同的命名空间之中](#Rc-helper)。

##### 强制实施

* 对并非处于其操作数的命名空间中的运算符的定义进行标记。

### <a name="Ro-lambda"></a>C.170: 当想要重载 lambda 时，应当使用泛型 lambda

##### 理由

无法通过定义两个带有相同名字的不同 lambda 来进行重载。

##### 示例

    void f(int);
    void f(double);
    auto f = [](char);   // 错误: 无法重载变量和函数
    
    auto g = [](int) { /* ... */ };
    auto g = [](double) { /* ... */ };   // 错误: 无法重载变量
    
    auto h = [](auto) { /* ... */ };   // OK

##### 强制实施

编译期会查明对 lambda 进行重载的企图。

## <a name="SS-union"></a>C.union: 联合体

`union` 是一种 `struct`，其所有成员都开始于相同的地址，因而它同时只能持有一个成员。
`union` 并不会跟踪其所存储的是哪个成员，因此必须由程序员来保证其正确；
这本质上就是易错的，但有一些弥补的办法。

由一个 `union` 加上一个用于指出其当前持有哪个成员的指示符构成的类型被称为*带标记联合体（tagged union）*，*区分性联合体（discriminated union）*，或者*变异体（variant）*。

联合体规则概览：

* [C.180: 采用 `union` 用以节省内存](#Ru-union)
* [C.181: 避免"裸" `union`](#Ru-naked)
* [C.182: 利用匿名 `union` 实现带标记联合体](#Ru-anonymous)
* [C.183: 不要将 `union` 用于类型双关](#Ru-pun)
* ???

### <a name="Ru-union"></a>C.180: 采用 `union` 用以节省内存

##### 理由

`union` 可以让一块内存在不同的时间用于不同类型的数据。
因此，当我们有几个不可能同时使用的对象时，可以用它来节省内存。

##### 示例

    union Value {
        int x;
        double d;
    };
    
    Value v = { 123 };  // 现在 v 持有一个 int
    cout << v.x << '\n';    // 写下 123
    v.d = 987.654;  // 现在 v 持有一个 double
    cout << v.d << '\n';    // 写下 987.654

但请你留意这个警告：[避免"裸"`union`](#Ru-naked)。

##### 示例

    // 短字符串优化
    
    constexpr size_t buffer_size = 16; // 比指针的大小稍大
    
    class Immutable_string {
    public:
        Immutable_string(const char* str) :
            size(strlen(str))
        {
            if (size < buffer_size)
                strcpy_s(string_buffer, buffer_size, str);
            else {
                string_ptr = new char[size + 1];
                strcpy_s(string_ptr, size + 1, str);
            }
        }
    
        ~Immutable_string()
        {
            if (size >= buffer_size)
                delete string_ptr;
        }
    
        const char* get_str() const
        {
            return (size < buffer_size) ? string_buffer : string_ptr;
        }
    
    private:
        // 当字符串足够短时，可以以其自己保存字符串
        // 而不是指向字符串的指针。
        union {
            char* string_ptr;
            char string_buffer[buffer_size];
        };
    
        const size_t size;
    };

##### 强制实施

???

### <a name="Ru-naked"></a>C.181: 避免"裸" `union`

##### 理由

*裸联合体（naked union）*是没有相关的指示其持有哪个成员（如果有）的指示器的联合体，
程序员必须保持对它的跟踪。
裸联合体是类型错误的一种来源。

##### Example, bad

    union Value {
        int x;
        double d;
    };
    
    Value v;
    v.d = 987.654;  // v 持有一个 double

至此为止还都不错，但我们可能会轻易地误用这个 `union`：

    cout << v.x << '\n';    // 不好，未定义的行为：v 持有一个 double，但我们将之当做一个 int 来读取

注意这个类型错误的发生并没有任何明确的强制转换。
当我们测试程序时，其所打印的最后一个值是 `1683627180`，这是 `987.654` 的位模式的整数值。
我们这里遇到的是一个"不可见"的类型错误，它刚好给出的结果轻易可能被看作是无辜清白的。

对于"不可见"来说，下面的代码不会产生任何输出：

    v.x = 123;
    cout << v.d << '\n';    // 不好：未定义的行为

##### 替代方案

将它们和一个类型字段一起包装到类之中。

可以使用 C++17 的 `variant` 类型（在 `<variant>` 中可以找到）：

    variant<int, double> v;
    v = 123;        // v 持有一个 int
    int x = get<int>(v);
    v = 123.456;    // v 持有一个 double
    w = get<double>(v);

##### 强制实施

???

### <a name="Ru-anonymous"></a>C.182: 利用匿名 `union` 实现带标记联合体

##### 理由

设计良好的带标记联合体是类型安全的。
*匿名*联合体可以简化这种带有 (tag, union) 对的类的定义。

##### 示例

这个例子基本上是从 TC++PL4 pp216-218 借鉴而来的。
你可以查看原文以获得其介绍。

这段代码多少有点复杂。
处理一个带有用户定义的赋值和析构函数的类型是比较麻烦的。
在标准中包含 `variant` 的原因之一就是避免程序员不得不编写这样的代码。

    class Value { // 以一个联合体来表现两个候选表示
    private:
        enum class Tag { number, text };
        Tag type; // 区分
    
        union { // 表示（注意这是匿名联合体）
            int i;
            string s; // string 带有默认构造函数，复制操作，和析构函数
        };
    public:
        struct Bad_entry { }; // 用作异常
    
        ~Value();
        Value& operator=(const Value&);   // 因为 string 变体的缘故而需要这个
        Value(const Value&);
        // ...
        int number() const;
        string text() const;
    
        void set_number(int n);
        void set_text(const string&);
        // ...
    };
    
    int Value::number() const
    {
        if (type != Tag::number) throw Bad_entry{};
        return i;
    }
    
    string Value::text() const
    {
        if (type != Tag::text) throw Bad_entry{};
        return s;
    }
    
    void Value::set_number(int n)
    {
        if (type == Tag::text) {
            s.~string();      // 显式销毁 string
            type = Tag::number;
        }
        i = n;
    }
    
    void Value::set_text(const string& ss)
    {
        if (type == Tag::text)
            s = ss;
        else {
            new(&s) string{ss};   // 放置式 new: 显式构造 string
            type = Tag::text;
        }
    }
    
    Value& Value::operator=(const Value& e)   // 因为 string 变体的缘故而需要这个
    {
        if (type == Tag::text && e.type == Tag::text) {
            s = e.s;    // 常规的 string 赋值
            return *this;
        }
    
        if (type == Tag::text) s.~string(); // 显式销毁
    
        switch (e.type) {
        case Tag::number:
            i = e.i;
            break;
        case Tag::text:
            new(&s) string(e.s);   // 放置式 new: 显式构造
        }
    
        type = e.type;
        return *this;
    }
    
    Value::~Value()
    {
        if (type == Tag::text) s.~string(); // 显式销毁
    }

##### 强制实施

???

### <a name="Ru-pun"></a>C.183: 不要将 `union` 用于类型双关

##### 理由

读取一个 `union` 曾写入的成员不同类型的成员是未定义的行为。
这种双关是不可见的，或者至少比具名强制转换更难于找出。
使用 `union` 的类型双关是一种错误来源。

##### 示例，不好

    union Pun {
        int x;
        unsigned char c[sizeof(int)];
    };

`Pun` 的想法是要能够查看 `int` 的字符表示。

    void bad(Pun& u)
    {
        u.x = 'x';
        cout << u.c[0] << '\n';     // 未定义行为
    }

如果你想要查看 `int` 的字节的话，应使用（具名）强制转换：

    void if_you_must_pun(int& x)
    {
        auto p = reinterpret_cast<unsigned char*>(&x);
        cout << p[0] << '\n';     // OK；好多了
        // ...
    }

对对象的声明类型不同的 `reinterpret_cast` 结果进行访问是有定义的行为（虽然并不建议使用 `reinterpret_cast`），
但至少我们可以发觉发生了某种微妙的事情。

##### 注解

不幸的是，`union` 经常被用于类型双关。
我们认为"它有时候能够按预期工作"并不是一种强力的理由。

C++17 引入了一个独立类型 `std::byte` 以支持在原始对象表示上进行的操作。在这些操作中应当使用这个类型代替 `unsigned char` 或 `char`。

##### 强制实施

???



# <a name="S-enum"></a>Enum: 枚举

枚举用于定义整数值的集合，并用于为这种值集定义类型。有两种类型的枚举，
"普通"的 `enum` 和 `class enum`。

枚举规则概览：

* [Enum.1: 优先采用枚举而不是宏](#Renum-macro)
* [Enum.2: 采用枚举来表示相关的具名常量的集合](#Renum-set)
* [Enum.3: 优先采用 `enum class` 而不是"普通"`enum`](#Renum-class)
* [Enum.4: 针对安全和简单的用法来为枚举定义操作](#Renum-oper)
* [Enum.5: 请勿为枚举符采用 `ALL_CAPS` 命名方式](#Renum-caps)
* [Enum.6: 避免使用无名枚举](#Renum-unnamed)
* [Enum.7: 仅在必要时才为枚举指定其底层类型](#Renum-underlying)
* [Enum.8: 仅在必要时才指定枚举符的值](#Renum-value)

### <a name="Renum-macro"></a>Enum.1: 优先采用 `enum` 而不是宏

##### 理由

宏不遵守作用域和类型规则。而且，宏的名字在预处理中就被移除，因而通常不会出现在如调试器这样的工具中。

##### 示例

首先是一些不好的老代码：

    // webcolors.h (第三方头文件)
    #define RED   0xFF0000
    #define GREEN 0x00FF00
    #define BLUE  0x0000FF
    
    // productinfo.h
    // 以下则基于颜色定义了产品的子类型
    #define RED    0
    #define PURPLE 1
    #define BLUE   2
    
    int webby = BLUE;   // webby == 2; 可能不是我们所想要的

代之以 `enum`：

    enum class Web_color { red = 0xFF0000, green = 0x00FF00, blue = 0x0000FF };
    enum class Product_info { red = 0, purple = 1, blue = 2 };
    
    int webby = blue;   // 错误: 应当明确
    Web_color webby = Web_color::blue;

我们用 `enum class` 来避免名字冲突。

##### 强制实施

标记定义整数值的宏。


### <a name="Renum-set"></a>Enum.2: 采用枚举来表示相关的具名常量的集合

##### 理由

枚举展示其枚举符之间是有关联的，且可以用作具名类型。



##### 示例

    enum class Web_color { red = 0xFF0000, green = 0x00FF00, blue = 0x0000FF };


##### 注解

对枚举的 `switch` 是很常见的，编译器可以对不平常的 `case` 标签进行警告。例如：

    enum class Product_info { red = 0, purple = 1, blue = 2 };
    
    void print(Product_info inf)
    {
        switch (inf) {
        case Product_info::red: cout << "red"; break;
        case Product_info::purple: cout << "purple"; break;
        }
    }

这种漏掉一个的 `switch` 语句通常是添加枚举符并缺少测试的结果。

##### 强制实施

* 当 `switch` 语句的 `case` 标签并未覆盖枚举的全部枚举符时，对其进行标记。
* 当 `switch` 语句的 `case` 覆盖了枚举的几个枚举符，但没有 `default` 时，对其进行标记。


### <a name="Renum-class"></a>Enum.3: 优先采用 `class enum` 而不是"普通"`enum`

##### 理由

最小化意外情况：传统的 `enum` 太容易转换为 `int` 了。

##### 示例

    void Print_color(int color);
    
    enum Web_color { red = 0xFF0000, green = 0x00FF00, blue = 0x0000FF };
    enum Product_info { Red = 0, Purple = 1, Blue = 2 };
    
    Web_color webby = Web_color::blue;
    
    // 显然至少有一个调用是有问题的。
    Print_color(webby);
    Print_color(Product_info::Blue);

代之以 `enum class`：

    void Print_color(int color);
    
    enum class Web_color { red = 0xFF0000, green = 0x00FF00, blue = 0x0000FF };
    enum class Product_info { red = 0, purple = 1, blue = 2 };
    
    Web_color webby = Web_color::blue;
    Print_color(webby);  // 错误: 无法转换 Web_color 为 int。
    Print_color(Product_info::Red);  // 错误: 无法转换 Product_info 为 int。

##### 强制实施

【简单】 对所有非 `class enum` 定义进行警告。

### <a name="Renum-oper"></a>Enum.4: 针对安全和简单的用法来为枚举定义操作

##### 理由

便于使用并避免犯错。

##### 示例

    enum Day { mon, tue, wed, thu, fri, sat, sun };
    
    Day& operator++(Day& d)
    {
        return d = (d == Day::sun) ? Day::mon : static_cast<Day>(static_cast<int>(d)+1);
    }
    
    Day today = Day::sat;
    Day tomorrow = ++today;

这里使用 `static_cast` 有点不好，但

    Day& operator++(Day& d)
    {
        return d = (d == Day::sun) ? Day::mon : Day(++d);    // 错误
    }

是无限递归，而且不用强制转换而使用一个针对所有情况的 `switch` 太冗长了。


##### 强制实施

对重复出现的强制转换回枚举的表达式进行标记。


### <a name="Renum-caps"></a>Enum.5: 请勿为枚举符采用 `ALL_CAPS` 命名方式

##### 理由

避免和宏之间发生冲突

##### 示例，不好

     // webcolors.h （第三方头文件）
    #define RED   0xFF0000
    #define GREEN 0x00FF00
    #define BLUE  0x0000FF
    
    // productinfo.h
    // 以下则基于颜色定义了产品的子类型
    
    enum class Product_info { RED, PURPLE, BLUE };   // 语法错误

##### 强制实施

标记 ALL_CAPS 风格的枚举符。

### <a name="Renum-unnamed"></a>Enum.6: 避免使用无名枚举

##### 理由

如果无法对枚举命名的话，它的值之间就是没有关联的。

##### 示例，不好

    enum { red = 0xFF0000, scale = 4, is_signed = 1 };

这种代码在出现指定整数常量的其他方便方式之前并不少见。

##### 替代方案

代之以使用 `constexpr` 值。例如：

    constexpr int red = 0xFF0000;
    constexpr short scale = 4;
    constexpr bool is_signed = true;

##### 强制实施

对无名枚举进行标记。


### <a name="Renum-underlying"></a>Enum.7: 仅在必要时才为枚举指定其底层类型

##### 理由

缺省情况的读写都是最简单的。
`int` 是缺省的整数类型。
`int` 是和 C 的 `enum` 相兼容的。

##### 示例

    enum class Direction : char { n, s, e, w,
                                  ne, nw, se, sw };  // 底层类型可以节省空间
    
    enum class Web_color : int32_t { red   = 0xFF0000,
                                     green = 0x00FF00,
                                     blue  = 0x0000FF };  // 底层类型是多余的

##### 注解

枚举前向声明中是有必要指定底层类型的：

    enum Flags : char;
    
    void f(Flags);
    
    // ....
    
    enum flags : char { /* ... */ };


##### 强制实施

????


### <a name="Renum-value"></a>Enum.8: 仅在必要时才指定枚举符的值

##### 理由

这是最简单的。
避免了枚举符值发生重复。
缺省情况会提供一组连续的值，并有利于 `switch` 语句的实现。

##### 示例

    enum class Col1 { red, yellow, blue };
    enum class Col2 { red = 1, yellow = 2, blue = 2 }; // 打错字
    enum class Month { jan = 1, feb, mar, apr, may, jun,
                       jul, august, sep, oct, nov, dec }; // 传统是从 1 开始
    enum class Base_flag { dec = 1, oct = dec << 1, hex = dec << 2 }; // 位的集合

为了和传统的值相匹配（比如 `Month`），以及当连续的值不合要求
（比如像 `Base_flag` 一样分配不同的位），是需要指定值的。

##### 强制实施

* 标记重复的枚举值
* 对明确指定的全部连续的枚举符的值进行标记。


# <a name="S-resource"></a>R: 资源管理

本章节中包含于资源相关的各项规则。
资源，就是任何必须进行获取，并（显式或隐式）进行释放的东西，比如内存、文件句柄、Socket 和锁等等。
其必须进行释放的原因在于它们是短缺的，因而即便是采用延迟释放也是有害的。
基本的目标是要确保不会泄漏任何资源，而且不会持有不在需要的任何资源。
负责释放某个资源的实体被称作是其所有者。

少数情况下，发生泄漏是可接受的甚至是理想的：
如果所编写的程序只是基于输入来产生输出，而其所需的内存正比于输入的大小，那么最理想的（性能和开发便利性）策略有时候恰是不要删除任何东西。
如果有足够的内存来处理最大输入的话，让其泄漏即可，但如果并非如此，应当保证给出一条恰当的错误消息。
这里，我们将忽略这样的情况。

* 资源管理规则概览：

  * [R.1: 利用资源句柄和 RAII（资源获取即初始化）来自动管理资源](#Rr-raii)
  * [R.2: 接口中的原生指针（仅）代表个体对象](#Rr-use-ptr)
  * [R.3: 原生指针（`T*`）没有所有权](#Rr-ptr)
  * [R.4: 原生引用（`T&`）没有所有权](#Rr-ref)
  * [R.5: 优先采用有作用域的对象，避免不必要的堆分配](#Rr-scoped)
  * [R.6: 避免非 `const` 的全局变量](#Rr-global)

* 分配和回收规则概览：

  * [R.10: 避免 `malloc()` 和 `free()`](#Rr-mallocfree)
  * [R.11: 避免显式调用 `new` 和 `delete`](#Rr-newdelete)
  * [R.12: 显式资源分配的结果应当立即交给一个管理对象](#Rr-immediate-alloc)
  * [R.13: 单个表达式语句中至多进行一次显式资源分配](#Rr-single-alloc)
  * [R.14: 避免使用 `[]` 形参，优先使用 `span`](#Rr-ap)
  * [R.15: 总是同时重载相匹配的分配、回收函数对](#Rr-pair)

* <a name="Rr-summary-smartptrs"></a>智能指针规则概览：

  * [R.20: 用 `unique_ptr` 或 `shared_ptr` 表示所有权](#Rr-owner)
  * [R.21: 优先采用 `unique_ptr` 而不是 `shared_ptr`，除非需要共享所有权](#Rr-unique)
  * [R.22: 使用 `make_shared()` 创建 `shared_ptr`](#Rr-make_shared)
  * [R.23: 使用 `make_unique()` 创建 `unique_ptr`](#Rr-make_unique)
  * [R.24: 使用 `std::weak_ptr` 来打断 `shared_ptr` 的循环引用](#Rr-weak_ptr)
  * [R.30: 以智能指针为参数，仅用于明确表达生存期语义](#Rr-smartptrparam)
  * [R.31: 非 `std` 的智能指针，应当遵循 `std` 的行为模式](#Rr-smart)
  * [R.32: `unique_ptr<widget>` 参数用以表达函数假定获得 `widget` 的所有权](#Rr-uniqueptrparam)
  * [R.33: `unique_ptr<widget>&` 参数用以表达函数对该 `widget` 重新置位](#Rr-reseat)
  * [R.34: `shared_ptr<widget>` 参数用以表达函数是所有者的一份子](#Rr-sharedptrparam-owner)
  * [R.35: `shared_ptr<widget>&` 参数用以表达函数可能会对共享的指针重新置位](#Rr-sharedptrparam)
  * [R.36: `const shared_ptr<widget>&` 参数用以表达它可能将保留一个对对象的引用 ???](#Rr-sharedptrparam-const)
  * [R.37: 不要把来自某个智能指针别名的指针或引用传递出去](#Rr-smartptrget)

### <a name="Rr-raii"></a>R.1: 利用资源句柄和 RAII（资源获取即初始化）来自动管理资源

##### 理由

避免资源泄漏和人工资源管理的复杂性。
C++ 语言确保的构造函数/析构函数对称性，反映了资源的获取/释放函数对（比如 `fopen`/`fclose`，`lock`/`unlock`，以及 `new`/`delete` 等）的对称性本质。
每当需要处理某个需要成对儿的获取/释放函数调用的资源时，应当将资源封装到保证这种配对调用的对象之中——在构造函数中获取资源，并在其析构函数中释放它。

##### 示例，不好

考虑：

    void send(X* x, cstring_span destination)
    {
        auto port = open_port(destination);
        my_mutex.lock();
        // ...
        send(port, x);
        // ...
        my_mutex.unlock();
        close_port(port);
        delete x;
    }

这段代码中，你必须记得在所有路径中调用 `unlock`、`close_port` 和 `delete`，并且每个都恰好调用一次。
而且，一旦上面标有 `...` 的任何代码抛出了异常，`x` 就会泄漏，而 `my_mutex` 则保持锁定。

##### 示例

考虑：

    void send(unique_ptr<X> x, cstring_span destination)  // x 拥有这个 X
    {
        Port port{destination};            // port 拥有这个 PortHandle
        lock_guard<mutex> guard{my_mutex}; // guard 拥有这个锁
        // ...
        send(port, x);
        // ...
    } // 自动解锁 my_mutex 并删除 x 中的指针

现在所有的资源清理都是自动进行的，不管是否发生了异常，所有路径中都会执行一次。额外的好处是，该函数现在明确声称它将接过指针的所有权。

`Port` 又是什么呢？是一个封装资源的便利句柄：

    class Port {
        PortHandle port;
    public:
        Port(cstring_span destination) : port{open_port(destination)} { }
        ~Port() { close_port(port); }
        operator PortHandle() { return port; }
    
        // port 句柄通常是不能克隆的，因此根据需要关闭了复制和赋值
        Port(const Port&) = delete;
        Port& operator=(const Port&) = delete;
    };

##### 注解

一旦发现一个"表现不良"的资源并未以带有析构函数的类来表示，就用一个类来包装它，或者使用 [`finally`](#Re-finally)。

**参见**: [RAII](#Rr-raii)

### <a name="Rr-use-ptr"></a>R.2: 接口中的原生指针（仅）代表个体对象

##### 理由

最好用某个容器类型（比如 `vector`，拥有数据），或者用 `span`（不拥有数据）来表示数组。
这些容器和视图都带有足够的信息来进行范围检查。

##### 示例，不好

    void f(int* p, int n)   // n 为 p[] 中的元素数量
    {
        // ...
        p[2] = 7;   // 不好: 对原生指针采用下标
        // ...
    }

编译期不会读注释，而如果不读其他代码的话你也无法指导 `p` 是否真的指向了 `n` 个元素。
应当代之以 `span`。

##### 示例

    void g(int* p, int fmt)   // 用格式 fmt 打印 *p
    {
        // ... 只使用 *p 和 p[0] ...
    }

##### 例外

C 风格的字符串是以单个指向以零结尾的字符序列的指针来传递的。
为了表明对这种约定的依赖，应当使用 `zstring` 而不是 `char*`。

##### 注解

当前许多的单元素指针的用法其实都应当用引用。
不过，如果 `nullptr` 是可能的值的话，引用就不是合理的替代方案了。

##### 强制实施

* 对并非来自容器、视图或迭代器的指针进行的指针算术（包括 `++`）进行标记。
  这条规则对比较老的代码库实施时，可能会产生巨量的误报。
* 对把数组名被传递为单纯的指针进行标记。

### <a name="Rr-ptr"></a>R.3: 原生指针（`T*`）没有所有权

##### 理由

对此（C++ 标准中和大多数代码中都）没有异议，大多数原生指针都是无所有权的。
我们希望将有所有权的指针标示出来，以使得可以可靠和高效地删除由有所有权指针所指向的对象。

##### 示例

    void f()
    {
        int* p1 = new int{7};           // 不好: 原生指针拥有了所有权
        auto p2 = make_unique<int>(7);  // OK: int 被一个唯一指针所拥有
        // ...
    }

`unique_ptr` 保证对它的对象进行删除（即便是发生异常时也如此），以此保护不发生泄漏。而 `T*` 做不到这点。

##### 示例

    template<typename T>
    class X {
    public:
        T* p;   // 不好: 不清楚 p 是不是带有所有权
        T* q;   // 不好: 不清楚 q 是不是带有所有权
        // ...
    };

可以通过明确所有权来修正这个问题：

    template<typename T>
    class X2 {
    public:
        owner<T*> p;  // OK: p 具有所有权
        T* q;         // OK: q 没有所有权
        // ...
    };

##### 例外

最主要的例外就是遗留代码，尤其是那些必须维持可以用 C 编译或者通过 ABI 来建立 C 和 C 风格的 C++ 之间的接口的代码。
亿万行的代码都违反本条规则而让 `T*` 具有所有权的现实是无法被忽略的。
我们由衷希望看到程序变换工具把这些 20 岁以上的"遗留"代码转换成光鲜的现代代码，
我们鼓励这种工具的开发、部署和使用，
我们希望这里的各项指导方针能够有助于这种工具的开发，
而且我们也在这一领域的研发工作中持续作出贡献。
但是，这是需要时间的："遗留代码"的产生比我们能翻新的老代码还要快，因此这将会花费许多年的时间。

这些代码是无法被全部重写的（即便假定有良好的代码转换软件），尤其不会很快发生。
这个问题是不能简单通过把所有有所有权的指针都转换成 `unique_ptr` 和 `shared_ptr` 来（大规模）解决的，
这部分是因为我们确实需要在基础的资源句柄的实现中一起使用有所有权的"原生指针"和简单的指针。
例如，常见的 `vector` 实现中都有一个有所有权的指针和两个没有所有权的指针。
许多 ABI（以及基本上全部的面向 C 的接口代码）都使用 `T*`，其中不少都是有所有权的。
一些接口是无法简单地用 `owner` 来标记的，因为它们需要维持可以作为 C 来编译
，（这可能是少见的恰当的使用宏的场合，它仅在 C++ 模式中扩展为 `owner`）。

##### 注解

`owner<T*>` 并没有超出 `T*` 的默认语义。使用它可以不改动任何使用方代码，也不会影响 ABI。
它只不过是一项针对程序员和分析工具的提示。
比如说，当 `owner<T*>` 是某个类的成员时，这个类最好提供一个析构函数来 `delete` 它。

##### 示例，不好

返回（原生）指针的做法向调用方暴露了在生存期管理上的不确定性；就是说，谁应该删除其所指向的对象呢？

    Gadget* make_gadget(int n)
    {
        auto p = new Gadget{n};
        // ...
        return p;
    }
    
    void caller(int n)
    {
        auto p = make_gadget(n);   // 要记得 delete p
        // ...
        delete p;
    }

除了遭受[资源泄漏](#???)的问题外，这也带来了一组假性的分配和回收操作，而这其实是不必要的。如果 Gadget 可以廉价地从函数转移出来（就是说，它很小，或者具有高效的移动操作）的话，直接"按值"返回即可（参见[输出返回值](#Rf-out)）：

    Gadget make_gadget(int n)
    {
        Gadget g{n};
        // ...
        return g;
    }

##### 注解

这条规则适用于工厂函数。

##### 注解

如果指针语义是必须的（比如说，因为返回类型需要指代类层次中的基类（或接口）），则可以返回"智能指针"。

##### 强制实施

* 【简单】 对在并非 `owner<T>` 的原生指针上进行的 `delete` 给出警告。
* 【中等】 对一个 `owner<T>` 指针，当并非每个代码路径中都要么进行 `reset` 要么明确 `delete`，则给出警告。
* 【简单】 当 `new` 的返回值被赋值给原生指针时，给出警告。
* 【简单】 当函数所返回的对象是在函数中所分配的，并且它具有移动构造函数时，给出警告。
  建议代之以按值返回。

### <a name="Rr-ref"></a>R.4: 原生引用（`T&`）没有所有权

##### 理由

对此（C++ 标准中和大多数代码中都）没有异议，大多数原生引用都是无所有权的。
我们希望将所有者都标示出来，以使得可以可靠和高效地删除由有所有权指针所指向的对象。

##### 示例

    void f()
    {
        int& r = *new int{7};  // 不好: 原生的具有所有权的引用
        // ...
        delete &r;             // 不好: 违反了有关删除原生指针的规则
    }

**参见**: [原生指针的规则](#Rr-ptr)

##### 强制实施

参见[原生指针的规则](#Rr-ptr)

### <a name="Rr-scoped"></a>R.5: 优先采用有作用域的对象，避免不必要的堆分配

##### 理由

有作用域的对象是局部对象、全局对象，或者成员。
它们也意味着在其所在作用域或者所在对象之外无须花费单独的分配和回收成本。
有作用域对象的成员自身也是有作用域的，有作用域对象的构造函数和析构函数负责管理其成员的生存期。

##### 示例

下面的例子效率不佳（因为无必要的分配和回收），在 `...` 中抛出异常和返回也是脆弱的（导致发生泄漏），而且也比较啰嗦：

    void f(int n)
    {
        auto p = new Gadget{n};
        // ...
        delete p;
    }

可以用局部变量来代替它：

    void f(int n)
    {
        Gadget g{n};
        // ...
    }

##### 强制实施

* 【中级】 如果分配了某个对象，又在函数内的所有路径中都进行了回收，则给出警告。建议它应当被代之以一个局部的 `auto` 栈对象。
* 【简单】 当局部的 `Unique_pointer` 或 `Shared_pointer` 在其生存期结束前未被移动、复制、赋值或 `reset`，则给出警告。

### <a name="Rr-global"></a>R.6: 避免非 `const` 的全局变量

##### 理由

全局变量是可以在任何地方访问的，因此它们可能会导致在貌似无关的对象之间出现预期之外的依赖关系。
这是一种可观的错误来源。

**警告**: 全局对象的初始化并不是完全有序的。
当使用全局对象时，应当用常量为之初始化。
还要注意，即便对于 `const` 对象，也可能发生未定义的初始化顺序。

##### 例外

全局对象通常优于单例。

##### 例外

不可变（`const`）的全局对象并不会引入那些我们试图通过禁止全局对象来避免的问题。

##### 强制实施

(??? NM: 显然可以对非 `const` 的静态对象给出警告……要不要这样做呢？)

## <a name="SS-alloc"></a>R.alloc: 分配与回收

### <a name="Rr-mallocfree"></a>R.10: 避免 `malloc()` 和 `free()`

##### 理由

`malloc()` 和 `free()` 并不支持构造和销毁，而且无法同 `new` 和 `delete` 之间进行混用。

##### 示例

    class Record {
        int id;
        string name;
        // ...
    };
    
    void use()
    {
        // p1 可能是 nullptr
        // *p1 并未初始化；尤其是，
        // 其中的 string 也还不是一个 string，而是一片和 string 大小相同的字节而已
        Record* p1 = static_cast<Record*>(malloc(sizeof(Record)));
    
        auto p2 = new Record;
    
        // 如果没有抛出异常的话，*p2 就经过了默认初始化
        auto p3 = new(nothrow) Record;
        // p3 可能为 nullptr；否则 *p3 就经过了默认初始化
    
        // ...
    
        delete p1;    // 错误: 不能 delete 由 malloc() 分配的对象
        free(p2);    // 错误: 不能 free() 由 new 分配的对象
    }

在某些实现中，`delete` 和 `free()` 可能可以工作，或者也可能引发运行时的错误。

##### 例外

有些应用程序或者代码段是不能接受异常的。
它们的最佳例子就是那些姓名攸关的硬实时代码。
但要注意的是，许多对异常的禁止其实是基于某种（不良的）迷信，
或者来源于对没有进行系统性的资源管理的老式代码库的关注（很不幸，但这经常是必须的）。
在这些情况下，应当考虑使用 `nothrow` 版本的 `new`。

##### 强制实施

对 `malloc` 和 `free` 的使用进行标记。

### <a name="Rr-newdelete"></a>R.11: 避免显式调用 `new` 和 `delete`

##### 理由

由 `new` 所返回的指针应当属于一个资源句柄（它将调用 `delete`）。
若由 `new` 所返回的指针被赋值给普通的裸指针，那么这个对象就可能会泄漏。

##### 注解

大型程序中，裸露的 `delete`（即出现于应用程序代码中，而不是专门进行资源管理的代码中）
很可能是一个 BUG：如果已经有了 N 个 `delete` 的话，怎么确定我们需要的不是 N+1 或者 N-1 个呢？
这种 BUG 可能会潜伏起来：它可能只会在维护过程中暴露出来。
如果出现了裸露的 `new`，那就可能在别的什么地方需要一个裸露的 `delete`，因而可能也是一个 BUG。

##### 强制实施

【简单】 对任何 `new` 和 `delete` 的显式使用都给出警告。建议代之以 `make_unique`。

### <a name="Rr-immediate-alloc"></a>R.12: 显式资源分配的结果应当立即交给一个管理对象

##### 理由

如果不这样做的话，当发生异常或者返回时就可能造成泄露。

##### 示例，不好

    void f(const string& name)
    {
        FILE* f = fopen(name, "r");            // 打开文件
        vector<char> buf(1024);
        auto _ = finally([f] { fclose(f); });  // 记得要关闭文件
        // ...
    }

`buf` 的分配可能会失败，并导致文件句柄的泄漏。

##### 示例

    void f(const string& name)
    {
        ifstream f{name};   // 打开文件
        vector<char> buf(1024);
        // ...
    }

对文件句柄（在 `ifstream` 中）的使用是简单、高效而且安全的。

##### 强制实施

* 将用于初始化指针的显式分配标记出来。（问题：我们能识别出多少直接资源分配呢？）

### <a name="Rr-single-alloc"></a>R.13: 单个表达式语句中至多进行一次显式资源分配

##### 理由

如果在一条语句中进行两次显式资源分配的话就可能发生资源泄漏，这是因为许多的子表达式（包括函数参数）的求值顺序都是未指明的。

##### 示例

    void fun(shared_ptr<Widget> sp1, shared_ptr<Widget> sp2);

可以这样调用 `fun`：

    // 不好：可能会泄漏
    fun(shared_ptr<Widget>(new Widget(a, b)), shared_ptr<Widget>(new Widget(c, d)));

这是异常不安全的，因为编译器可能会把两个用以创建函数的两个参数的表达式重新排序。
特别是，编译器是可以交错执行这两个表达式的：
它可能会首先为两个对象都（通过调用 `operator new`）进行内存分配，然后再试图调用二者的 `Widget` 构造函数。
一旦其中一个构造函数调用抛出了异常，那么另一个对象的内存可能永远不会被释放了！

这个微妙的问题其实有一种简单的解决方案：永远不要在一条表达式语句中进行多于一次的显式资源分配。
例如：

    shared_ptr<Widget> sp1(new Widget(a, b)); // 好多了，但不太干净
    fun(sp1, new Widget(c, d));

最佳的方案是使用返回所有者对象的工厂函数，而完全避免显式的分配：

    fun(make_shared<Widget>(a, b), make_shared<Widget>(c, d)); // 最佳

如果还没有，请自己编写一个工厂包装。

##### 强制实施

* 将包含多次显式资源分配的表达式标记出来。（问题：我们能识别出多少直接资源分配呢？）

### <a name="Rr-ap"></a>R.14: 避免使用 `[]` 形参，优先使用 `span`

##### 理由

数组会退化为指针，因而丢失其大小信息，并留下了发生范围错误的机会。
使用 `span` 来保留大小信息。

##### 示例

    void f(int[]);          // 不建议的做法
    
    void f(int*);           // 对多个对象不建议的做法
                            // （指针应当指向单个对象，不要进行下标运算）
    
    void f(gsl::span<int>); // 好，建议的做法

##### 强制实施

标记出 `[]` 参数。代之以使用 `span`。

### <a name="Rr-pair"></a>R.15: 总是同时重载相匹配的分配、回收函数对

##### 理由

不然的话就出现不匹配的操作，并导致混乱。

##### 示例

    class X {
        // ...
        void* operator new(size_t s);
        void operator delete(void*);
        // ...
    };

##### 注解

如果想要无法进行回收的内存的话，可以将回收操作 `=delete`。
请勿留下它而不进行声明。

##### 强制实施

标记出不完全的操作对。

## <a name="SS-smart"></a>R.smart: 智能指针

### <a name="Rr-owner"></a>R.20: 用 `unique_ptr` 或 `shared_ptr` 表示所有权

##### 理由

它们可以避免资源泄漏。

##### 示例

考虑：

    void f()
    {
        X x;
        X* p1 { new X };              // 参见 ???
        unique_ptr<T> p2 { new X };   // 唯一所有权；参见 ???
        shared_ptr<T> p3 { new X };   // 共享所有权；参见 ???
        auto p4 = make_unique<X>();   // 唯一所有权，比显式使用"new"要好
        auto p5 = make_shared<X>();   // 共享所有权，比显式使用"new"要好
    }

这里（只有）初始化 `p1` 的对象将会泄漏。

##### 强制实施

【简单】 如果 `new` 的返回值或者指针类型的函数调用返回值被赋值给了原生指针，就给出警告。

### <a name="Rr-unique"></a>R.21: 优先采用 `unique_ptr` 而不是 `shared_ptr`，除非需要共享所有权

##### 理由

`unique_ptr` 概念上要更简单且更可预测（知道它何时会销毁），而且更快（不需要暗中维护引用计数）。

##### 示例，不好

这里并不需要维护一个引用计数。

    void f()
    {
        shared_ptr<Base> base = make_shared<Derived>();
        // 局部范围中使用 base，并未进行复制——引用计数不会超过 1
    } // 销毁 base

##### 示例

这样更加高效：

    void f()
    {
        unique_ptr<Base> base = make_unique<Derived>();
        // 局部范围中使用 base
    } // 销毁 base

##### 强制实施

【简单】 如果函数所使用的 `Shared_pointer` 的对象是函数之内所分配的，而且既不会将这个 `Shared_pointer` 返回，也不会将其传递给其他接受 `Shared_pointer&` 的函数的话，就给出警告。建议代之以 `unique_ptr`。

### <a name="Rr-make_shared"></a>R.22: 使用 `make_shared()` 创建 `shared_ptr`

##### 理由

如果先创建对象再将其传给 `shared_ptr` 的构造函数的话，（最大可能是）比用 `make_shared()` 时多进行一次分配（以及随后的回收），这是因为引用计数只能和对象分开进行分配。

##### 示例

考虑：

    shared_ptr<X> p1 { new X{2} }; // 不好
    auto p = make_shared<X>(2);    // 好

`make_shared()` 版本仅提到一次 `X`，因而它通常比显式的 `new` 方式要更简短（而且更快）。

##### 强制实施

【简单】 如果 `shared_ptr` 从 `new` 的结果而不是 `make_shared` 进行构造，就给出警告。

### <a name="Rr-make_unique"></a>R.23: 使用 `make_unique()` 创建 `unique_ptr`

##### 理由

编码便利以及和 `shared_ptr` 保持一致。

##### 注解

`make_unique()` 是 C++14 中的，不过已经广泛可用（而且也很容易编写）。

##### 强制实施

【简单】 如果 `unique_ptr` 从 `new` 的结果而不是 `make_unique` 进行构造，就给出警告。

### <a name="Rr-weak_ptr"></a>R.24: 使用 `std::weak_ptr` 来打断 `shared_ptr` 的循环引用

##### 理由

`shared_ptr` 是基于引用计数的，而带有循环的结构中的引用计数不可能变为零，因此我们需要一种机制
来打破带有循环的结构。

##### 示例

    #include <memory>
    
    class bar;
    
    class foo
    {
    public:
      explicit foo(const std::shared_ptr<bar>& forward_reference)
        : forward_reference_(forward_reference)
      { }
    private:
      std::shared_ptr<bar> forward_reference_;
    };
    
    class bar
    {
    public:
      explicit bar(const std::weak_ptr<foo>& back_reference)
        : back_reference_(back_reference)
      { }
      void do_something()
      {
        if (auto shared_back_reference = back_reference_.lock()) {
          // 使用 *shared_back_reference
        }
      }
    private:
      std::weak_ptr<foo> back_reference_;
    };

##### 注解

??? (HS: 许多人都说要"打破循环引用"，不过我觉得"临时性共享所有权"可能更关键。)
???(BS: 打破循环引用是必须达成的目标；临时性共享所有权则是达成的方式。
你也可以仅仅使用另一个 `shared_ptr` 来得到"临时性共享所有权"。)

##### 强制实施

??? 可能无法做到。如果能够静态地检测出循环引用的话，我们就不需要 `weak_ptr` 了。

### <a name="Rr-smartptrparam"></a>R.30: 以智能指针为参数，仅用于明确表达生存期语义

##### 理由

如果函数仅仅需要 `widget` 自身的话，接受一个 `widget` 的智能指针就是错误的。
它应当可以接受任何 `widget` 对象，而不仅是那些由特定种类的智能指针管理其生存期的对象才对。
不会操纵对象生存期的函数应当接受原生的指针或引用。

##### 示例，不好

    // 被调用方
    void f(shared_ptr<widget>& w)
    {
        // ...
        use(*w); // w 的唯一使用 -- 完全没有用到其生存期
        // ...
    };
    
    // 调用方
    shared_ptr<widget> my_widget = /* ... */;
    f(my_widget);
    
    widget stack_widget;
    f(stack_widget); // 错误

##### 示例，好

    // 被调用方
    void f(widget& w)
    {
        // ...
        use(w);
        // ...
    };
    
    // 调用方
    shared_ptr<widget> my_widget = /* ... */;
    f(*my_widget);
    
    widget stack_widget;
    f(stack_widget); // ok -- 现在没问题了

##### 强制实施

* 【简单】 如果函数接受可以复制的智能指针类型（重载了 `operator->` 或者 `operator*`）的参数，但仅调用了它的 `operator*`，`operator->` 或者 `get()` 的话，就给出警告。
  建议代之以 `T*` 或 `T&`。
* 如果智能指针类型（即重载了 `operator->` 或 `operator*` 的类型）的参数是可以复制/移动的，但并未从函数体中复制或移动出去，同时它未被改动，而且也未被传递给可能做这些事的其他函数的话，就对其进行标记。这些意味着它的所有权语义并未被用到。
  建议代之以 `T*` 或 `T&`。

### <a name="Rr-smart"></a>R.31: 非 `std` 的智能指针，应当遵循 `std` 的行为模式

##### 理由

下面段落中的规则同样适用于第三方和自定义的其他种类的智能指针，而且对于诊断引发导致了性能和正确性问题的一般性的智能指针错误来说也是非常有帮助的。
你将会期望你所使用的所有智能指针都遵循这些规则。

任何重载了一元 `*` 和 `->` 的类型（无论主模板还是特化）都被当成是智能指针：

* 如果它可以复制，则将其当做一种具有引用计数的 `Shared_ptr`。
* 如果它不能复制，则将其当做一种唯一的 `Unique_ptr`。

##### 示例

    // 使用 Boost 的 intrusive_ptr
    #include <boost/intrusive_ptr.hpp>
    void f(boost::intrusive_ptr<widget> p)  // 根据 'sharedptrparam' 规则是错误的
    {
        p->foo();
    }
    
    // 使用 Microsoft 的 CComPtr
    #include <atlbase.h>
    void f(CComPtr<widget> p)               // 根据 'sharedptrparam' 规则是错误的
    {
        p->foo();
    }

上面两段根据 [`sharedptrparam` 指导方针](#Rr-smartptrparam)来说都是错误的：
`p` 是一个 `Shared_pointer`，但其共享性质完全没有被用到，而对其进行按值传递则是一种暗含的劣化；
这两个函数应当仅当它们需要参与 `widget` 的生存期管理时才接受智能指针。否则当可以为 `nullptr` 时它们就应当接受 `widget*`，否则，理想情况下，函数应当接受 `widget&`。
这些智能指针都符合 `Shared_pointer` 的概念，因此这些强制实施指导方针的规则可以直接应用，并使得这种一般性的劣化情况暴露出来。

### <a name="Rr-uniqueptrparam"></a>R.32: `unique_ptr<widget>` 参数用以表达函数假定获得 `widget` 的所有权

##### 理由

以这种方式使用 `unique_ptr` 同时说明并强制施加了函数调用时的所有权转移。

##### 示例

    void sink(unique_ptr<widget>); // 获得这个 widget 的所有权
    
    void uses(widget*);            // 仅仅使用了这个 widget

##### 示例，不好

    void thinko(const unique_ptr<widget>&); // 通常不是你想要的

##### 强制实施

* 【简单】 如果函数以左值引用接受 `Unique_pointer<T>` 参数，但并未在至少一个代码路径中向其赋值或者对其调用 `reset()`，则给出警告。建议代之以接受 `T*` 或 `T&`。
* 【简单】〔基础〕 如果函数以 `const` 引用接受 `Unique_pointer<T>` 参数，则给出警告。建议代之以接受 `const T*` 或 `const T&`。

### <a name="Rr-reseat"></a>R.33: `unique_ptr<widget>&` 参数用以表达函数对该 `widget` 重新置位

##### 示例

以这种方式使用 `unique_ptr` 同时说明并强制施加了函数调用时的重新置位语义。

##### 注解

所谓"重新置位（Reseat）"的含义是"让指针或智能指针指代某个不同的对象"。

##### 示例

    void reseat(unique_ptr<widget>&); // "将要"或"可能"重新置位指针

##### 示例，不好

    void thinko(const unique_ptr<widget>&); // 通常不是你想要的

##### 强制实施

* 【简单】 如果函数以左值引用接受 `Unique_pointer<T>` 参数，但并未在至少一个代码路径中向其赋值或者对其调用 `reset()`，则给出警告。建议代之以接受 `T*` 或 `T&`。
* 【简单】〔基础〕 如果函数以 `const` 引用接受 `Unique_pointer<T>` 参数，则给出警告。建议代之以接受 `const T*` 或 `const T&`。

### <a name="Rr-sharedptrparam-owner"></a>R.34: `shared_ptr<widget>` 参数用以表达函数是所有者的一份子

##### 理由

这样做明确了函数的所有权共享语义。

##### 示例，好

    void share(shared_ptr<widget>);            // 共享——"将会"保持一个引用计数
    
    void may_share(const shared_ptr<widget>&); // "可能"保持一个引用计数
    
    void reseat(shared_ptr<widget>&);          // "可能"重新置位指针

##### 强制实施

* 【简单】 如果函数以左值引用接受 `Shared_pointer<T>` 参数，但并未在至少一个代码路径中向其赋值或者对其调用 `reset()`，则给出警告。建议代之以接受 `T*` 或 `T&`。
* 【简单】〔基础〕 如果函数按值或者以 `const` 引用接受 `Shared_pointer<T>` 参数，但并未在至少一个代码路径中将其复制或移动给另一个 `Shared_pointer`，则给出警告。建议代之以接受 `T*` 或 `T&`。
* 【简单】〔基础〕 如果函数以右值引用接受 `Shared_pointer<T>` 参数，则给出警告。建议代之以按值传递。

### <a name="Rr-sharedptrparam"></a>R.35: `shared_ptr<widget>&` 参数用以表达函数可能会对共享的指针重新置位

##### 理由

这样做明确了函数的重新置位语义。

##### 注解

所谓"重新置位（Reseat）"的含义是"让引用或智能指针指代某个不同的对象"。

##### 示例，好

    void share(shared_ptr<widget>);            // 共享——"将会"保持一个引用计数
    
    void reseat(shared_ptr<widget>&);          // "可能"重新置位指针
    
    void may_share(const shared_ptr<widget>&); // "可能"保持一个引用计数

##### 强制实施

* 【简单】 如果函数以左值引用接受 `Shared_pointer<T>` 参数，但并未在至少一个代码路径中向其赋值或者对其调用 `reset()`，则给出警告。建议代之以接受 `T*` 或 `T&`。
* 【简单】〔基础〕 如果函数按值或者以 `const` 引用接受 `Shared_pointer<T>` 参数，但并未在至少一个代码路径中将其复制或移动给另一个 `Shared_pointer`，则给出警告。建议代之以接受 `T*` 或 `T&`。
* 【简单】〔基础〕 如果函数以右值引用接受 `Shared_pointer<T>` 参数，则给出警告。建议代之以按值传递。

### <a name="Rr-sharedptrparam-const"></a>R.36: `const shared_ptr<widget>&` 参数用以表达它可能将保留一个对对象的引用 ???

##### 理由

这样做明确了函数的 ??? 语义。

##### 示例，好

    void share(shared_ptr<widget>);            // 共享——"将会"保持一个引用计数
    
    void reseat(shared_ptr<widget>&);          // "可能"重新置位指针
    
    void may_share(const shared_ptr<widget>&); // "可能"保持一个引用计数

##### 强制实施

* 【简单】 如果函数以左值引用接受 `Shared_pointer<T>` 参数，但并未在至少一个代码路径中向其赋值或者对其调用 `reset()`，则给出警告。建议代之以接受 `T*` 或 `T&`。
* 【简单】〔基础〕 如果函数按值或者以 `const` 引用接受 `Shared_pointer<T>` 参数，但并未在至少一个代码路径中将其复制或移动给另一个 `Shared_pointer`，则给出警告。建议代之以接受 `T*` 或 `T&`。
* 【简单】〔基础〕 如果函数以右值引用接受 `Shared_pointer<T>` 参数，则给出警告。建议代之以按值传递。

### <a name="Rr-smartptrget"></a>R.37: 不要把来自某个智能指针别名的指针或引用传递出去

##### 理由

违反这条规则，是导致引用计数的丢失和出现悬挂指针的首要原因。
函数应当优先向其调用链中传递原生指针和引用。
在调用树的顶层，原生指针或引用是从用以保持对象存活的智能指针中获得的。
我们需要确保这个智能指针不会在调用树的下面被疏忽地进行重置或者重新赋值。

##### 注解

为了做到这点，有时候需要获得智能指针的一个局部副本，它可以确保在函数及其调用树的执行期间维持对象存活。

##### 示例

考虑下面的代码：

    // 全局（静态或堆）对象，或者有别名的局部对象 ...
    shared_ptr<widget> g_p = ...;
    
    void f(widget& w)
    {
        g();
        use(w);  // A
    }
    
    void g()
    {
        g_p = ...; // 噢，如果这就是这个 widget 的最后一个 shared_ptr 的话，这会销毁这个 widget
    }

下面的代码是不应该通过代码评审的：

    void my_code()
    {
        // 不好: 传递的是从非局部的智能指针中获得的指针或引用
        //       而这可能会在 f 或其调用的函数中的某处被不经意地重置掉
        f(*g_p);
    
        // 不好: 原因相同，只不过将其作为"this"指针传递
        g_p->func();
    }

修正很简单——获取该指针的一个局部副本，为调用树"保持一个引用计数"：

    void my_code()
    {
        // 很廉价: 一次增量就搞定了整个函数以及下面的所有调用树
        auto pin = g_p;
    
        // 好: 传递的是从局部的无别名智能指针中获得的指针或引用
        f(*pin);
    
        // 好: 原因相同
        pin->func();
    }

##### 强制实施

* 【简单】 如果从非局部或局部但潜在具有别名的智能指针变量（`Unique_pointer` 或 `Shared_pointer`）中所获取的指针或引用，被用于进行函数调用，则给出警告。如果智能指针是一个 `Shared_pointer`，则建议代之以获取该智能指针的一个局部副本并从中获取指针或引用。

# <a name="S-expr"></a>ES: 表达式和语句

表达式和语句是用以表达动作和计算的最底层也是最直接的方式。局部作用域中的声明也是语句。

有关命名、注释和缩进的规则，参见 [NL: 命名与代码布局](#S-naming)。

一般规则：

* [ES.1: 优先采用标准库而不是其他的库或者"手工自制代码"](#Res-lib)
* [ES.2: 优先采用适当的抽象而不是直接使用语言功能特性](#Res-abstr)

声明的规则：

* [ES.5: 保持作用域尽量小](#Res-scope)
* [ES.6: 在 for 语句的初始化式和条件中声明名字以限制其作用域](#Res-cond)
* [ES.7: 保持常用的和局部的名字尽量简短，而让非常用的和非局部的名字较长](#Res-name-length)
* [ES.8: 避免使用看起来相似的名字](#Res-name-similar)
* [ES.9: 避免 `ALL_CAPS` 式的名字](#Res-not-CAPS)
* [ES.10: 每条声明中（仅）声明一个名字](#Res-name-one)
* [ES.11: 使用 `auto` 来避免类型名字的多余重复](#Res-auto)
* [ES.12: 不要在嵌套作用域中重用名字](#Res-reuse)
* [ES.20: 坚持为对象进行初始化](#Res-always)
* [ES.21: 不要在确实需要使用变量（或常量）之前就引入它](#Res-introduce)
* [ES.22: 要等到获得了用以初始化变量的值之后才声明变量](#Res-init)
* [ES.23: 优先使用 `{}` 初始化式语法](#Res-list)
* [ES.24: 用 `unique_ptr<T>` 来保存指针](#Res-unique)
* [ES.25: 应当将对象声明为 `const` 或 `constexpr`，除非后面需要修改其值](#Res-const)
* [ES.26: 不要用一个变量来达成两个不相关的目的](#Res-recycle)
* [ES.27: 使用 `std::array` 或 `stack_array` 作为栈上的数组](#Res-stack)
* [ES.28: 为复杂的初始化（尤其是 `const` 变量）使用 lambda](#Res-lambda-init)
* [ES.30: 不要用宏来操纵程序文本](#Res-macros)
* [ES.31: 不要用宏来作为常量或"函数"](#Res-macros2)
* [ES.32: 对所有的宏名采用 `ALL_CAPS` 命名方式](#Res-ALL_CAPS)
* [ES.33: 如果必须使用宏的话，请为之提供唯一的名字](#Res-MACROS)
* [ES.34: 不要定义（C 风格的）变参函数](#Res-ellipses)

表达式的规则：

* [ES.40: 避免复杂的表达式](#Res-complicated)
* [ES.41: 对运算符优先级不保准时应使用括号](#Res-parens)
* [ES.42: 保持单纯直接的指针使用方式](#Res-ptr)
* [ES.43: 避免带有未定义的求值顺序的表达式](#Res-order)
* [ES.44: 不要对函数参数求值顺序有依赖](#Res-order-fct)
* [ES.45: 避免"魔法常量"，采用符号化常量](#Res-magic)
* [ES.46: 避免窄化转换](#Res-narrowing)
* [ES.47: 使用 `nullptr` 而不是 `0` 或 `NULL`](#Res-nullptr)
* [ES.48: 避免强制转换](#Res-casts)
* [ES.49: 当必须使用强制转换时，使用具名的强制转换](#Res-casts-named)
* [ES.50: 不要强制掉 `const`](#Res-casts-const)
* [ES.55: 避免发生对范围检查的需要](#Res-range-checking)
* [ES.56: 仅在确实需要明确移动某个对象到别的作用域时才使用 `std::move()`](#Res-move)
* [ES.60: 避免在资源管理函数之外使用 `new` 和 `delete`](#Res-new)
* [ES.61: 用 `delete[]` 删除数组，用 `delete` 删除非数组对象](#Res-del)
* [ES.62: 不要在不同的数组之间进行指针比较](#Res-arr2)
* [ES.63: 不要产生切片](#Res-slice)
* [ES.64: 使用 `T{e}` 写法来进行构造](#Res-construct)
* [ES.65: 不要解引用无效指针](#Res-deref)

语句的规则：

* [ES.70: 面临选择时，优先采用 `switch` 语句而不是 `if` 语句](#Res-switch-if)
* [ES.71: 面临选择时，优先采用范围式 `for` 语句而不是普通 `for` 语句](#Res-for-range)
* [ES.72: 当存在显然的循环变量时，优先采用 `for` 语句而不是 `while` 语句](#Res-for-while)
* [ES.73: 当没有显然的循环变量时，优先采用 `while` 语句而不是 `for` 语句](#Res-while-for)
* [ES.74: 优先在 `for` 语句的初始化部分中声明循环变量](#Res-for-init)
* [ES.75: 避免使用 `do` 语句](#Res-do)
* [ES.76: 避免 `goto`](#Res-goto)
* [ES.77: 尽量减少循环中使用的 `break` 和 `continue`](#Res-continue)
* [ES.78: 不要依靠 `switch` 语句中的隐含直落行为](#Res-break)
* [ES.79: `default`（仅）用于处理一般情况](#Res-default)
* [ES.84: 不要试图声明没有名字的局部变量](#Res-noname)
* [ES.85: 让空语句显著可见](#Res-empty)
* [ES.86: 避免在原生的 `for` 循环中修改循环控制变量](#Res-loop-counter)
* [ES.87: 请勿在条件上添加多余的 `==` 或 `!=`](#Res-if)

算术规则：

* [ES.100: 不要进行有符号和无符号混合运算](#Res-mix)
* [ES.101: 使用无符号类型进行位操作](#Res-unsigned)
* [ES.102: 使用有符号类型进行算术运算](#Res-signed)
* [ES.103: 避免上溢出](#Res-overflow)
* [ES.104: 避免下溢出](#Res-underflow)
* [ES.105: 避免除零](#Res-zero)
* [ES.106: 不要试图用 `unsigned` 来防止负数值](#Res-nonnegative)
* [ES.107: 不要对下标使用 `unsigned`，优先使用 `gsl::index`](#Res-subscripts)

### <a name="Res-lib"></a>ES.1: 优先采用标准库而不是其他的库或者"手工自制代码"

##### 理由

使用程序库的代码要远比直接使用语言功能特性的代码易于编写，更为简短，更倾向于更高的抽象层次，而且程序库代码假定已经经过测试。
ISO C++ 标准库是最广为了解而且经过最好测试的程序库之一。
它是任何 C++ 实现的一部分。

##### 示例

    auto sum = accumulate(begin(a), end(a), 0.0);   // 好

`accumulate` 的范围版本就更好了：

    auto sum = accumulate(v, 0.0); // 更好

但请不要手工编写众所周知的算法：

    int max = v.size();   // 不好：啰嗦，目的不清晰
    double sum = 0.0;
    for (int i = 0; i < max; ++i)
        sum = sum + v[i];

##### 例外

标准库的很大部分都依赖于动态分配（自由存储）。这些部分，尤其是容器但并不包括算法，并不适用于某些硬实时和嵌入式的应用的。在这些情况下，请考虑提供或使用类似的设施，比如说某个标准库风格的采用池分配器的容器实现。

##### 强制实施

并不容易。??? 寻找混乱的循环，嵌套的循环，长函数，函数调用的缺失，缺乏使用非内建类型。圈复杂度？

### <a name="Res-abstr"></a>ES.2: 优先采用适当的抽象而不是直接使用语言功能特性

##### 理由

"适当的抽象"（比如程序库或者类），更加贴近应用的概念而不是语言概念，将带来更简洁的代码，而且更容易进行测试。

##### 示例

    vector<string> read1(istream& is)   // 好
    {
        vector<string> res;
        for (string s; is >> s;)
            res.push_back(s);
        return res;
    }

与之近乎等价的更传统且更低层的代码，更长、更混乱、更难编写正确，而且很可能更慢：

    char** read2(istream& is, int maxelem, int maxstring, int* nread)   // 不好：啰嗦而且不完整
    {
        auto res = new char*[maxelem];
        int elemcount = 0;
        while (is && elemcount < maxelem) {
            auto s = new char[maxstring];
            is.read(s, maxstring);
            res[elemcount++] = s;
        }
        nread = &elemcount;
        return res;
    }

一旦添加了溢出检查和错误处理，代码就变得相当混乱了，而且还有个要记得 `delete` 其所返回的指针以及数组所包含的 C 风格字符串的问题。

##### 强制实施

并不容易。??? 寻找混乱的循环，嵌套的循环，长函数，函数调用的缺失，缺乏使用非内建类型。圈复杂度？

## ES.dcl: 声明

声明也是语句。一条声明向一个作用域中引入一个名字，并可能导致对一个具名对象进行构造。

### <a name="Res-scope"></a>ES.5: 保持作用域尽量小

##### 理由

可读性。最小化资源持有率。避免值的意外误用。

**其他形式**: 不要把名字在不必要的大作用域中进行声明。

##### 示例

    void use()
    {
        int i;    // 不好: 循环之后并不需要访问 i
        for (i = 0; i < 20; ++i) { /* ... */ }
        // 此处不存在对 i 的有意使用
        for (int i = 0; i < 20; ++i) { /* ... */ }  // 好: i 局部于 for 循环
    
        if (auto pc = dynamic_cast<Circle*>(ps)) {  // 好: pc 局部于 if 语句
            // ... 处理 Circle ...
        }
        else {
            // ... 处理错误 ...
        }
    }

##### 示例，不好

    void use(const string& name)
    {
        string fn = name + ".txt";
        ifstream is {fn};
        Record r;
        is >> r;
        // ... 200 行代码，都不存在使用 fn 或 is 的意图 ...
    }

大多数测量都会称这个函数太长了，但其关键点是 `fn` 所使用的资源和 `is` 所持有的文件句柄
所持有的时间比其所需长太多了，而且可能在函数后面意外地出现对 `is` 和 `fn` 的使用。
这种情况下，把读取操作重构出来可能是一个好主意：

    Record load_record(const string& name)
    {
        string fn = name + ".txt";
        ifstream is {fn};
        Record r;
        is >> r;
        return r;
    }
    
    void use(const string& name)
    {
        Record r = load_record(name);
        // ... 200 行代码 ...
    }

##### 强制实施

* 对声明于循环之外，且在循环之后不再使用的循环变量进行标记。
* 当诸如文件句柄和锁这样的昂贵资源超过 N 行未被使用时进行标记（对某个适当的 N）。

### <a name="Res-cond"></a>ES.6: 在 for 语句的初始化式和条件中声明名字以限制其作用域

##### 理由

可读性。最小化资源持有率。

##### 理由

    void use()
    {
        for (string s; cin >> s;)
            v.push_back(s);
    
        for (int i = 0; i < 20; ++i) {   // 好: i 局部于 for 循环
            // ...
        }
    
        if (auto pc = dynamic_cast<Circle*>(ps)) {   // 好: pc 局部于 if 语句
            // ... 处理 Circle ...
        }
        else {
            // ... 处理错误 ...
        }
    }

##### 强制实施

* 对声明于循环之前，且在循环之后不再使用的循环变量进行标记。
* 【困难】 对声明与循环之前，且在循环之后用于某个无关用途的循环变量进行标记。

##### C++17 和 C++30 示例

注：C++17 和 C++20 还增加了 `if`、`switch` 和范围式 `for` 的初始化式语句。以下代码要求支持 C++17 和 C++20。

    map<int, string> mymap;
    
    if (auto result = mymap.insert(value); result.second) {
        // 本代码块中，插入成功且 result 有效
        use(result.first);  // ok
        // ...
    } // result 在此处销毁

##### C++17 和 C++20 强制实施（当使用 C++17 或 C++20 编译器时）

* 选择/循环变量，若其在选择或循环体之前声明而在其之后不再使用，则对其进行标记
* 【困难】 选择/循环变量，若其在选择或循环体之前声明而在其之后用于某个无关目的，则对其进行标记



### <a name="Res-name-length"></a>ES.7: 保持常用的和局部的名字尽量简短，而让非常用的和非局部的名字较长

##### 理由

可读性。减低在无关的非局部名字之间发生冲突的机会。

##### 示例

符合管理的简短的局部名字能够增强可读性：

    template<typename T>    // 好
    void print(ostream& os, const vector<T>& v)
    {
        for (gsl::index i = 0; i < v.size(); ++i)
            os << v[i] << '\n';
    }

索引根据惯例称为 `i`，而这个泛型函数中不存在有关这个 vector 的含义的提示，因此和其他名字一样， `v` 也没问题。与之相比，

    template<typename Element_type>   // 不好: 啰嗦，难于阅读
    void print(ostream& target_stream, const vector<Element_type>& current_vector)
    {
        for (gsl::index current_element_index = 0;
             current_element_index < current_vector.size();
             ++current_element_index
        )
        target_stream << current_vector[current_element_index] << '\n';
    }

当然，这是一个讽刺，但我们还见过更糟糕的。

##### 示例

不合惯例而简短的非局部名字则会搞乱代码：

    void use1(const string& s)
    {
        // ...
        tt(s);   // 不好: tt() 是什么？
        // ...
    }

更好的做法是，为非局部实体提供可读的名字：

    void use1(const string& s)
    {
        // ...
        trim_tail(s);   // 好多了
        // ...
    }

这样的话，有可能读者指导 `trim_tail` 是什么意思，而且读者可以找一下它并回忆起来。

##### 示例，不好

大型函数的参数的名字实际上可当作是非局部的，因此应当有意义：

    void complicated_algorithm(vector<Record>& vr, const vector<int>& vi, map<string, int>& out)
    // 根据 vi 中的索引，从 vr 中读取事件（并标记所用的 Record），
    // 向 out 中放置（名字，索引）对
    {
        // ... 500 行的代码，使用 vr，vi，和 out ...
    }

我们建议保持函数短小，但这条规则并不受到普遍坚持，而命名应当能反映这一点。

##### 强制实施

检查局部和非局部的名字的长度。同时考虑函数的长度。

### <a name="Res-name-similar"></a>ES.8: 避免使用看起来相似的名字

##### 理由

代码清晰性和可读性。太过相似的名字会拖慢理解过程并增加出错的可能性。

##### 示例，不好

    if (readable(i1 + l1 + ol + o1 + o0 + ol + o1 + I0 + l0)) surprise();

##### 示例，不好

不要在同一个作用域中用和类型相同的名字声明一个非类型实体。这样做将消除为区分它们所需的关键字 `struct` 或 `enum` 等。这同样消除了一种错误来源，因为 `struct X` 在对 `X` 的查找失败时会隐含地声明新的 `X`。

    struct foo { int n; };
    struct foo foo();       // 不好, foo 在作用域中已经是一个类型了
    struct foo x = foo();   // 需要进行区分

##### 例外

很古老的头文件中可能会用在相同作用域中用同一个名字同时声明非类型实体和类型。

##### 强制实施

* 用一个已知的易混淆字母和数字组合的列表来对名字进行检查。
* 当变量、函数或枚举符的声明隐藏了在相同作用域中所声明的类或枚举时，给出警告。

### <a name="Res-not-CAPS"></a>ES.9: 避免 `ALL_CAPS` 式的名字

##### 理由

这样的名字通常是用于宏的。因此，`ALL_CAPS` 这样的名字可能遭受意外的宏替换。

##### 示例

    // 某个头文件中：
    #define NE !=
    
    // 某个别的头文件中的别处：
    enum Coord { N, NE, NW, S, SE, SW, E, W };
    
    // 某个糟糕程序员的 .cpp 中的第三处：
    switch (direction) {
    case N:
        // ...
    case NE:
        // ...
    // ...
    }

##### 注解

不要仅仅因为常量曾经是宏，就使用 `ALL_CAPS` 作为常量。

##### 强制实施

对所有的 ALL CAPS 进行标记。对于老代码，则接受宏名字的 ALL CAPS 而标记所有的非 ALL-CAPS 的宏名字。

### <a name="Res-name-one"></a>ES.10: 每条声明中（仅）声明一个名字

##### 理由

每行一条声明的做法增加可读性并可避免与 C++ 的文法
相关的错误。这样做也为更具描述性的行尾注释留下了
空间。

##### 示例，不好

    char *p, c, a[7], *pp[7], **aa[10];   // 讨厌！

##### 例外

函数声明中可以包含多个函数参数声明。

##### 例外

结构化绑定（C++17）就是专门设计用于引入多个变量的：

    auto [iter, inserted] = m.insert_or_assign(k, val);
    if (inserted) { /* 已插入新条目 */ }

##### 示例

    template <class InputIterator, class Predicate>
    bool any_of(InputIterator first, InputIterator last, Predicate pred);

用 concept 则更佳：

    bool any_of(InputIterator first, InputIterator last, Predicate pred);

##### 示例

    double scalbn(double x, int n);   // OK: x * pow(FLT_RADIX, n); FLT_RADIX 通常为 2

或者：

    double scalbn(    // 有改善: x * pow(FLT_RADIX, n); FLT_RADIX 通常为 2
        double x,     // 基数
        int n         // 指数
    );

或者：

    // 有改善: base * pow(FLT_RADIX, exponent); FLT_RADIX 通常为 2
    double scalbn(double base, int exponent);

##### 示例

    int a = 7, b = 9, c, d = 10, e = 3;

在较长的声明符列表中，很容易忽视某个未能初始化的变量。

##### 强制实施

对具有多个声明符的变量或常量的声明式（比如 `int* p, q;`）进行标记。

### <a name="Res-auto"></a>ES.11: 使用 `auto` 来避免类型名字的多余重复

##### 理由

* 单纯的重复既麻烦又易错。
* 当使用 `auto` 时，所声明的实体的名字是处于声明的固定位置的，这增加了可读性。
* 模板函数声明的返回类型可能是某个成员类型。

##### 示例

考虑：

    auto p = v.begin();   // vector<int>::iterator
    auto h = t.future();
    auto q = make_unique<int[]>(s);
    auto f = [](int x){ return x + 10; };

以上都避免了写下冗长又难记的类型，它们是编译器已知的，但程序员则可能会搞错。

##### 示例

    template<class T>
    auto Container<T>::first() -> Iterator;   // Container<T>::Iterator

##### 例外

当使用初始化式列表，而所需要的确切类型是已知的，同时某个初始化式可能需要转换时，应当避免使用 `auto`。

##### 示例

    auto lst = { 1, 2, 3 };   // lst 是一个 initializer_list
    auto x{1};   // x 是一个 int（C++17；在 C++11 中则为 initializer_list）

##### 注解

如果可以使用概念的话，我们可以（而且应该）更加明确说明所推断的类型：

    // ...
    ForwardIterator p = algo(x, y, z);

##### 示例（C++17）

    auto [ quotient, remainder ] = div(123456, 73);   // 展开 div_t 结果中的成员

##### 强制实施

对声明中多余的类型名字进行标记。

### <a name="Res-reuse"></a>ES.12: 不要在嵌套作用域中重用名字

##### 理由

这样很容易把所用的是哪个变量搞混。
会造成维护问题。

##### 示例，不好

    int d = 0;
    // ...
    if (cond) {
        // ...
        d = 9;
        // ...
    }
    else {
        // ...
        int d = 7;
        // ...
        d = value_to_be_returned;
        // ...
    }
    
    return d;

这是个大型的 `if` 语句，很容易忽视在内部作用域中所引入的新的 `d`。
这是一种已知的 BUG 来源。
这种在内部作用域中的名字重用有时候被称为"遮蔽"。

##### 注解

当函数变得过大和过于复杂时，遮蔽是一种主要的问题。

##### 示例

语言不允许在函数的最外层块中遮蔽函数参数：

    void f(int x)
    {
        int x = 4;  // 错误：重用函数参数的名字
    
        if (x) {
            int x = 7;  // 允许，但不好
            // ...
        }
    }

##### 示例，不好

把成员名重用为局部变量也会造成问题：

    struct S {
        int m;
        void f(int x);
    };
    
    void S::f(int x)
    {
        m = 7;    // 对成员赋值
        if (x) {
            int m = 9;
            // ...
            m = 99; // 对局部变量赋值
            // ...
        }
    }

##### 例外

我们经常在派生类中重用基类中的函数名：

    struct B {
        void f(int);
    };
    
    struct D : B {
        void f(double);
        using B::f;
    };

这样做是易错的。
例如，要是忘了 using 声明式的话，`d.f(1)` 的调用就不会找到 `int` 版本的 `f`。

??? 我们需要为类层次中的遮蔽/隐藏给出专门的规则吗？

##### 强制实施

* 对嵌套局部作用域中的名字重用进行标记。
* 对成员函数中将成员名重用为局部变量进行标记。
* 对把全局名字重用为局部变量或成员的名字进行标记。
* 对在派生类中重用（除函数名之外的）基类成员名进行标记。

### <a name="Res-always"></a>ES.20: 坚持为对象进行初始化

##### 理由

避免发生"设值前使用"的错误及其所关联的未定义行为。
避免由复杂的初始化的理解所带来的问题。
简化重构。

##### 示例

    void use(int arg)
    {
        int i;   // 不好: 未初始化的变量
        // ...
        i = 7;   // 初始化 i
    }

错了，`i = 7` 并不是 `i` 的初始化；它是向其赋值。而且 `i` 也可能在 `...` 的部分中被读取。更好的做法是：

    void use(int arg)   // OK
    {
        int i = 7;   // OK: 初始化
        string s;    // OK: 默认初始化
        // ...
    }

##### 注释

我们有意让*总是进行初始化*规则比*对象在使用前必须设值*的语言规则更强。
后者是较为宽松的规则，虽然能够识别出技术上的 BUG，不过：

* 它会导致较不可读的代码，
* 它鼓励人们在比所需的更大的作用域中声明名字，
* 它会导致较难于阅读的代码，
* 它会因为鼓励复杂的代码而导致出现逻辑 BUG，
* 它会妨碍进行重构。

而*总是进行初始化*规则则是以提升可维护性为目标的一条风格规则，同样也是保护避免出现"设值前使用"错误的规则。

##### 示例

这个例子经常被当成是用来展示需要更宽松的初始化规则的例子。

    widget i;    // "widget" 是一个初始化操作昂贵的类型，可能是一种大型 POD
    widget j;
    
    if (cond) {  // 不好: i 和 j 进行了"延迟"初始化
        i = f1();
        j = f2();
    }
    else {
        i = f3();
        j = f4();
    }

这段代码是无法简单重写为用初始化式来对 `i` 和 `j` 进行初始化的。
注意，对于带有默认构造函数的类型来说，试图延后初始化只会导致变为一次默认初始化之后跟着一次赋值的做法。
这种例子的一种更加流行的理由是"效率"，不过可以检查出是否出现"设置前使用"错误的编译器，同样可以消除任何多余的双重初始化。

假定 `i` 和 `j` 之间存在某种逻辑关联，则这种关联可能应当在函数中予以表达：

    pair<widget, widget> make_related_widgets(bool x)
    {
        return (x) ? {f1(), f2()} : {f3(), f4() };
    }
    
    auto [i, j] = make_related_widgets(cond);    // C++17

如果除此之外 `make_related_widgets` 函数是多余的，
可以使用 lambda [ES.28](#Res-lambda-init) 来消除之：

    auto [i, j] = [x]{ return (x) ? pair{f1(), f2()} : pair{f3(), f4()} }();    // C++17

用一个值代表 `uninitialized` 只是一种问题的症状，而不是一种解决方案：

    widget i = uninit;  // 不好
    widget j = uninit;
    
    // ...
    use(i);         // 可能发生设值前使用
    // ...
    
    if (cond) {     // 不好: i 和 j 进行了"延迟"初始化
        i = f1();
        j = f2();
    }
    else {
        i = f3();
        j = f4();
    }

这样的话编译器甚至无法再简单地检测出"设值前使用"。而且我们也在 widget 的状态空间中引入了复杂性：哪些操作对 `uninit` 的 widget 是有效的，哪些不是？

##### 注解

几十年来，精明的程序员中都流行进行复杂的初始化。
这样做也是一种错误和复杂性的主要来源。
而许多这样的错误都是在最初实现之后的多年之后的维护过程中所引入的。

##### 示例

本条规则涵盖成员变量。

    class X {
    public:
        X(int i, int ci) : m2{i}, cm2{ci} {}
        // ...
    
    private:
        int m1 = 7;
        int m2;
        int m3;
    
        const int cm1 = 7;
        const int cm2;
        const int cm3;
    };

编译器能够标记 `cm3` 为未初始化（因其为 `const`），但它无法发觉 `m3` 缺少初始化。
通常来说，以很少不恰当的成员初始化来消除错误，要比缺乏初始化更有价值，
而且优化器是可以消除冗余的初始化的（比如紧跟在赋值之前的初始化）。

##### 例外

当声明一个即将从输入进行初始化的对象时，其初始化就可能导致发生双重初始化。
不过，应当注意这也可能造成输入之后留下未初始化的数据——而这已经是一种错误和安全攻击的重大来源：

    constexpr int max = 8 * 1024;
    int buf[max];         // OK, 但是可疑: 未初始化
    f.read(buf, max);

由于数组和 `std::array` 的有所限制的初始化规则，它们提供了对于需要这种例外的大多数有意义的例子。

某些情况下，这个数组进行初始化的成本可能是显著的。
但是，这样的例子确实倾向于留下可访问到的未初始化变量，因而应当严肃对待它们。

    constexpr int max = 8 * 1024;
    int buf[max] = {};   // 某些情况下更好
    f.read(buf, max);

如果可行的话，应当用某个已知不会溢出的库函数。例如：

    string s;   // s 默认初始化为 ""
    cin >> s;   // s 进行扩充以持有字符串

不要把用于输入操作的简单变量作为本条规则的例外：

    int i;   // 不好
    // ...
    cin >> i;

在并不罕见的情况下，当输入目标和输入操作分开（其实不应该）时，就带来了发生"设值前使用"的可能性。

    int i2 = 0;   // 更好，假设 0 是 i2 可接受的值
    // ...
    cin >> i2;

优秀的优化器应当能够识别输入操作并消除这种多余的操作。


##### 注解

有时候，可以用 lambda 作为初始化式以避免未初始化变量：

    error_code ec;
    Value v = [&] {
        auto p = get_value();   // get_value() 返回 pair<error_code, Value>
        ec = p.first;
        return p.second;
    }();

还可以是：

    Value v = [] {
        auto p = get_value();   // get_value() 返回 pair<error_code, Value>
        if (p.first) throw Bad_value{p.first};
        return p.second;
    }();

**参见**: [ES.28](#Res-lambda-init)

##### 强制实施

* 标记出每个未初始化的变量。
  不要对具有默认构造函数的自定义类型的变量进行标记。
* 检查未初始化的缓冲区是否在声明后*立即*进行了写入。
  将未初始化变量作为一个非 `const` 的引用参数进行传递可以被假定为向变量进行的写入。

### <a name="Res-introduce"></a>ES.21: 不要在确实需要使用变量（或常量）之前就引入它

##### 理由

可读性。限制变量可以被使用的范围。

##### 示例

    int x = 7;
    // ... 这里没有对 x 的使用 ...
    ++x;

##### 强制实施

对离其首次使用很远的声明进行标记。

### <a name="Res-init"></a>ES.22: 要等到获得了用以初始化变量的值之后才声明变量

##### 理由

可读性。限制变量可以被使用的范围。避免"设值前使用"的风险。初始化通常比赋值更加高效。

##### 示例，不好

    string s;
    // ... 此处没有 s 的使用 ...
    s = "what a waste";

##### 示例，不好

    SomeLargeType var;   // 难看的骆驼风格命名
    
    if (cond)   // 某个不简单的条件
        Set(&var);
    else if (cond2 || !cond3) {
        var = Set2(3.14);
    }
    else {
        var = 0;
        for (auto& e : something)
            var += e;
    }
    
    // 使用 var; 可以仅通过控制流而静态地保证这并不会过早进行

如果 `SomeLargeType` 的默认初始化并非过于昂贵的话就没什么问题。
不过，程序员可能十分想知道是否所有的穿过这个条件迷宫的路径都已经被覆盖到了。
如果没有的话，就存在一个"设值前使用"的 BUG。这是维护工作的一个陷阱。

对于具有必要复杂性的初始化式，也包括 `const` 变量的初始化式，应当考虑使用 lambda 来表达它；参见 [ES.28](#Res-lambda-init)。

##### 强制实施

* 如果具有默认初始化的声明在其首次被读取前就进行赋值，则对其进行标记。
* 对于任何在未初始化变量之后且在其使用之前进行的复杂计算进行标记。

### <a name="Res-list"></a>ES.23: 优先使用 `{}` 初始化式语法

##### 理由

优先使用 `{}`。`{}` 初始化的规则比其他形式的初始化更简单，更通用，更少歧义，而且更安全。

仅当你确定不存在窄化转换时才可使用 `=`。对于内建算术类型，`=` 仅和 `auto` 一起使用。

避免 `()` 初始化，它会导致解析中的歧义。

##### 示例

    int x {f(99)};
    int y = x;
    vector<int> v = {1, 2, 3, 4, 5, 6};

##### 例外

对于容器来说，存在用 `{...}` 给出元素列表而用 `(...)` 给出大小的传统做法：

    vector<int> v1(10);    // vector 有 10 个具有默认值 0 的元素
    vector<int> v2{10};    // vector 有 1 个值为 10 的元素
    
    vector<int> v3(1, 2);  // vector 有 1 个值为 2 的元素
    vector<int> v4{1, 2};  // vector 有 2 个值为 1 和 2 的元素

##### 注解

`{}` 初始化式不允许进行窄化转换（这点通常都很不错），并允许使用显式构造函数（这没有问题，我们的意图就是初始化一个新变量）。

##### 示例

    int x {7.9};   // 错误: 发生窄化
    int y = 7.9;   // OK: y 变为 7. 希望编译器给出了警告消息
    int z = gsl::narrow_cast<int>(7.9);  // OK: 这个正是你想要的

##### 注解

`{}` 初始化可以用于几乎所有的初始化；而其他的初始化则不行：

    auto p = new vector<int> {1, 2, 3, 4, 5};   // 初始化 vector
    D::D(int a, int b) :m{a, b} {   // 成员初始化式（比如说 m 可能是 pair）
        // ...
    };
    X var {};   // 初始化 var 为空
    struct S {
        int m {7};   // 成员的默认初始化
        // ...
    };

由于这个原因，以 `{}` 进行初始化通常被称为"统一初始化"，
（但很可惜存在少数不符合规则的例外）。

##### 注解

对以 `auto` 声明的变量用单个值进行的初始化，比如 `{v}`，直到 C++17 之前都还具有令人意外的含义。
C++17 的规则多少会少些意外：

    auto x1 {7};        // x1 是一个值为 7 的 int
    auto x2 = {7};      // x2 是一个具有一个元素 7 的 initializer_list<int>
    
    auto x11 {7, 8};    // 错误: 两个初始化式
    auto x22 = {7, 8};  // x22 是一个具有元素 7 和 8 的 initializer_list<int>

如果确实需要一个 `initializer_list<T>` 的话，可以使用 `={...}`：

    auto fib10 = {1, 1, 2, 3, 5, 8, 13, 21, 34, 55};   // fib10 是一个列表

##### 注解

`={}` 进行的是复制初始化，而 `{}` 则进行直接初始化。
与在复制初始化和直接初始化自身之间存在的区别类似的是，这里也可能带来一些意外情况。
`{}` 可以接受 `explicit` 构造函数；而 `={}` 则不能。例如：

    struct Z { explicit Z() {} };
    
    Z z1{};     // OK: 直接初始化，使用的是 explicit 构造函数
    Z z2 = {};  // 错误: 复制初始化，不能使用 explicit 构造函数

除非特别要求禁止使用显式构造函数，否则都应当使用普通的 `{}` 初始化。

##### 示例

    template<typename T>
    void f()
    {
        T x1(1);    // T 以 1 进行初始化
        T x0();     // 不好: 函数声明（一般都是一个错误）
    
        T y1 {1};   // T 以 1 进行初始化
        T y0 {};    // 默认初始化 T
        // ...
    }

**参见**: [讨论](#???)

##### 强制实施

* 当使用 `=` 初始化算术类型并发生窄化转换时予以标记。
* 当使用 `()` 初始化语法但实际上是声明式时予以标记。（许多编译器已经可就此给出警告。）

### <a name="Res-unique"></a>ES.24: 用 `unique_ptr<T>` 来保存指针

##### 理由

使用 `std::unique_ptr` 是避免泄漏的最简单方法。它是可靠的，它
利用类型系统完成验证所有权安全性的大部分工作，它
增加可读性，而且它没有或近乎没有运行时成本。

##### 示例

    void use(bool leak)
    {
        auto p1 = make_unique<int>(7);   // OK
        int* p2 = new int{7};            // 不好: 可能泄漏
        // ... 未对 p2 赋值 ...
        if (leak) return;
        // ... 未对 p2 赋值 ...
        vector<int> v(7);
        v.at(7) = 0;                    // 抛出异常
        // ...
    }

当 `leak == true` 时，`p2` 所指向的对象就会泄漏，而 `p1` 所指向的对象则不会。
当 `at()` 抛出异常时也是同样的情况。

##### 强制实施

寻找作为 `new`，`malloc()`，或者可能返回这类指针的函数的目标的原生指针。

### <a name="Res-const"></a>ES.25: 应当将对象声明为 `const` 或 `constexpr`，除非后面需要修改其值

##### 理由

这样的话你就不会误改掉这个值。而且这种方式可能给编译器的优化带来机会。

##### 示例

    void f(int n)
    {
        const int bufmax = 2 * n + 2;  // 好: 无法意外改掉 bufmax
        int xmax = n;                  // 可疑: xmax 是不是会改掉？
        // ...
    }

##### 强制实施

查看变量是不是真的被改动过，若并非如此就进行标记。
不幸的是，也许不可能检测出某个非 `const` 是不是
*有意*要改动，还是仅仅是没被改动而已。

### <a name="Res-recycle"></a>ES.26: 不要用一个变量来达成两个不相关的目的

##### 理由

可读性和安全性。

##### 示例，不好

    void use()
    {
        int i;
        for (i = 0; i < 20; ++i) { /* ... */ }
        for (i = 0; i < 200; ++i) { /* ... */ } // 不好: i 重复使用了
    }

+##### 注解

你可能想把一个缓冲区当做暂存器来重复使用以作为一种优化措施，但即便如此也请尽可能限定该变量的作用域，还要当心不要导致由于遗留在重用的缓冲区中的数据而引发的 BUG，这是安全性 BUG 的一种常见来源。

    void write_to_file() {
        std::string buffer;             // 以避免每次循环重复中的重新分配
        for (auto& o : objects)
        {
            // 第一部分工作。
            generate_first_string(buffer, o);
            write_to_file(buffer);
    
            // 第二部分工作。
            generate_second_string(buffer, o);
            write_to_file(buffer);
    
            // 等等...
        }
    }

##### 强制实施

标记被重复使用的变量。

### <a name="Res-stack"></a>ES.27: 使用 `std::array` 或 `stack_array` 作为栈上的数组

##### 理由

它们是可读的，而且不会隐式转换为指针。
它们不会和内建数组的非标准扩展相混淆。

##### 示例，不好

    const int n = 7;
    int m = 9;
    
    void f()
    {
        int a1[n];
        int a2[m];   // 错误: 并非 ISO C++
        // ...
    }

##### 注解

`a1` 的定义是合法的 C++ 而且一直都是。
存在大量的这类代码。
不过它是易错的，尤其当它的界并非局部时更是如此。
而且它也是一种"流行"的错误来源（缓冲区溢出，数组衰退而成的指针，等等）。
而 `a2` 的定义符合 C 但不符合 C++，而且被认为存在安全性风险。

##### 示例

    const int n = 7;
    int m = 9;
    
    void f()
    {
        array<int, n> a1;
        stack_array<int> a2(m);
        // ...
    }

##### 强制实施

* 对具有非常量界的数组（C 风格的 VLA）作出标记。
* 对具有非局部的常量界的数组作出标记。

### <a name="Res-lambda-init"></a>ES.28: 为复杂的初始化（尤其是 `const` 变量）使用 lambda

##### 理由

它可以很好地封装局部的初始化，包括对仅为初始化所需的临时变量进行清理，而且避免了创建不必要的非局部而且无法重用的函数。它对于应当为 `const` 的变量也可以工作，不过必须先进行一些初始化。

##### 示例，不好

    widget x;   // 应当为 const, 不过:
    for (auto i = 2; i <= N; ++i) {          // 这是由 x 的
        x += some_obj.do_something_with(i);  // 初始化所需的
    }                                        // 一段任意长的代码
    // 自此开始，x 应当为 const，不过我们无法在这种风格的代码中做到这点

##### 示例，好

    const widget x = [&]{
        widget val;                                // 假定 widget 具有默认构造函数
        for (auto i = 2; i <= N; ++i) {            // 这是由 x 的
            val += some_obj.do_something_with(i);  // 初始化所需的
        }                                          // 一段任意长的代码
        return val;
    }();

##### 示例

    string var = [&]{
        if (!in) return "";   // 默认值
        string s;
        for (char c : in >> c)
            s += toupper(c);
        return s;
    }(); // 注意这个 ()

如果可能的话，应当将条件缩减成一个后续的简单集合（比如一个 `enum`），并避免把选择和初始化相互混合起来。

##### 强制实施

很难。最多是某种启发式方案。查找跟随某个未初始化变量之后的循环中向其赋值。

### <a name="Res-macros"></a>ES.30: 不要用宏来操纵程序文本

##### 理由

宏是 BUG 的一个主要来源。
宏不遵守常规的作用域和类型规则。
宏保证会让人读到的东西和编译器见到的东西不一样。
宏使得工具的建造复杂化。

##### 示例，不好

    #define Case break; case   /* 不好 */

这个貌似无害的宏会把某个大写的 `C` 替换为小写的 `c` 导致一个严重的控制流错误。

##### 注解

这条规则并不禁止在 `#ifdef` 等部分中使用用于"配置控制"的宏。

将来，模块可能会消除配置控制中对宏的需求。

##### 注解

此规则也意味着不鼓励使用 `#` 进行字符串化和使用 `##` 进行连接。
照例，宏有一些"无害"的用途，但即使这些也会给工具带来麻烦，
例如自动完成器、静态分析器和调试器。
通常，使用花式宏的欲望是过于复杂的设计的标志。
另外，`＃` 和 `##` 促进了宏的定义和使用：

    #define CAT(a, b) a ## b
    #define STRINGIFY(a) #a
    
    void f(int x, int y)
    {
        string CAT(x, y) = "asdf";   // 不好: 工具难以处理（也很丑陋）
        string sx2 = STRINGIFY(x);
        // ...
    }

有使用宏进行低级字符串操作的变通方法。例如：

    string s = "asdf" "lkjh";   // 普通的字符串文字连接
    
    enum E { a, b };
    
    template<int x>
    constexpr const char* stringify()
    {
        switch (x) {
        case a: return "a";
        case b: return "b";
        }
    }
    
    void f(int x, int y)
    {
        string sx = stringify<x>();
        // ...
    }

这不像定义宏那样方便，但是易于使用、零开销，并且是类型化的和作用域化的。

将来，静态反射可能会消除对程序文本操作的预处理器的最终需求。

##### 强制实施

见到并非仅用于源代码控制（比如 `#ifdef`）的宏时应当大声尖叫。

### <a name="Res-macros2"></a>ES.31: 不要用宏来作为常量或"函数"

##### 理由

宏是 BUG 的一个主要来源。
宏不遵守常规的作用域和类型规则。
宏不遵守常规的参数传递规则。
宏保证会让人读到的东西和编译器见到的东西不一样。
宏使得工具的建造复杂化。

##### 示例，不好

    #define PI 3.14
    #define SQUARE(a, b) (a * b)

即便我们并未在 `SQUARE` 中留下这个众所周知的 BUG，也存在多种表现好得多的替代方式；比如：

    constexpr double pi = 3.14;
    template<typename T> T square(T a, T b) { return a * b; }

##### 强制实施

见到并非仅用于源代码控制（比如 `#ifdef`）的宏时应当大声尖叫。

### <a name="Res-ALL_CAPS"></a>ES.32: 对所有的宏名采用 `ALL_CAPS` 命名方式

##### 理由

遵循约定。可读性。区分宏。

##### 示例

    #define forever for (;;)   /* 非常不好 */
    
    #define FOREVER for (;;)   /* 仍然很邪恶，但至少对人来说是可见的 */

##### 强制实施

见到小写的宏时应当大声尖叫。

### <a name="Res-MACROS"></a>ES.33: 如果必须使用宏的话，请为之提供唯一的名字

##### 理由

宏并不遵守作用域规则。

##### 示例

    #define MYCHAR        /* 不好，最终将会和别人的 MYCHAR 相冲突 */
    
    #define ZCORP_CHAR    /* 还是不好，但冲突的机会较小 */

##### 注解

如果可能就应当避免使用宏：[ES.30](#Res-macros)，[ES.31](#Res-macros2)，以及 [ES.32](#Res-ALL_CAPS)。
然而，存在亿万行的代码中包含宏，以及一种使用并过度使用宏的长期传统。
如果你被迫使用宏的话，请使用长名字，而且应当带有唯一前缀（比如你的组织机构的名字）以减少冲突的可能性。

##### 强制实施

对较短的宏名给出警告。

### <a name="Res-ellipses"></a> ES.34: 不要定义（C 风格的）变参函数

##### 理由

它并非类型安全。
而且需要杂乱的满是强制转换和宏的代码才能正确工作。

##### 示例

    #include <cstdarg>
    
    // "severity" 后面跟着以零终结的 char* 列表；将 C 风格字符串写入 cerr
    void error(int severity ...)
    {
        va_list ap;             // 一个持有参数的神奇类型
        va_start(ap, severity); // 参数启动："severity" 是 error() 的第一个参数
    
        for (;;) {
            // 将下一个变量看作 char*；没有检查：经过伪装的强制转换
            char* p = va_arg(ap, char*);
            if (!p) break;
            cerr << p << ' ';
        }
    
        va_end(ap);             // 参数清理（不能忘了这个）
    
        cerr << '\n';
        if (severity) exit(severity);
    }
    
    void use()
    {
        error(7, "this", "is", "an", "error", nullptr);
        error(7); // 崩溃
        error(7, "this", "is", "an", "error");  // 崩溃
        const char* is = "is";
        string an = "an";
        error(7, "this", "is", an, "error"); // 崩溃
    }

**替代方案**: 重载。模板。变参模板。

    #include <iostream>
    
    void error(int severity)
    {
        std::cerr << '\n';
        std::exit(severity);
    }
    
    template <typename T, typename... Ts>
    constexpr void error(int severity, T head, Ts... tail)
    {
        std::cerr << head;
        error(severity, tail...);
    }
    
    void use()
    {
        error(7); // 不会崩溃！
        error(5, "this", "is", "not", "an", "error"); // 不会崩溃！
    
        std::string an = "an";
        error(7, "this", "is", "not", an, "error"); // 不会崩溃！
    
        error(5, "oh", "no", nullptr); // 编译器报错！不需要 nullptr。
    }


##### 注解

这基本上就是 `printf` 的实现方式。

##### 强制实施

* 对 C 风格的变参函数的定义作出标记。
* 对 `#include <cstdarg>` 和 `#include <stdarg.h>` 作出标记。


## ES.expr: 表达式

表达式对值进行操作。

### <a name="Res-complicated"></a>ES.40: 避免复杂的表达式

##### 理由

复杂的表达式是易错的。

##### 示例

    // 不好: 在子表达式中藏有赋值
    while ((c = getc()) != -1)
    
    // 不好: 在一个子表达式中对两个非局部变量进行了赋值
    while ((cin >> c1, cin >> c2), c1 == c2)
    
    // 有改善，但可能仍然过于复杂
    for (char c1, c2; cin >> c1 >> c2 && c1 == c2;)
    
    // OK: 若 i 和 j 并非别名
    int x = ++i + ++j;
    
    // OK: 若 i != j 且 i != k
    v[i] = v[j] + v[k];
    
    // 不好: 子表达式中"隐藏"了多个赋值
    x = a + (b = f()) + (c = g()) * 7;
    
    // 不好: 依赖于经常被误解的优先级规则
    x = a & b + c * d && e ^ f == 7;
    
    // 不好: 未定义行为
    x = x++ + x++ + ++x;

这些表达式中有几个是无条件不好的（比如说依赖于未定义行为）。其他的只不过过于复杂和不常见，即便是优秀的程序员匆忙中也可能会误解或者忽略其中的某个问题。

##### 注解

C++17 收紧了有关求值顺序的规则
（除了赋值中从右向左，以及函数实参求值顺序未指明外均为从左向右，[参见 ES.43](#Res-order)），
但这并不影响复杂表达式很容易引起混乱的事实。

##### 注解

程序员应当了解并运用表达式的基本规则。

##### 示例

    x = k * y + z;             // OK
    
    auto t1 = k * y;           // 不好: 不必要的啰嗦
    x = t1 + z;
    
    if (0 <= x && x < max)   // OK
    
    auto t1 = 0 <= x;        // 不好: 不必要的啰嗦
    auto t2 = x < max;
    if (t1 && t2)            // ...

##### 强制实施

很麻烦。多复杂的表达式才能被当成复杂的？把计算写成每条语句一个操作同样是让人混乱的。需要考虑的有：

* 副作用：（对于某种非局部性定义，）多个非局部变量上发生副作用是值得怀疑的，尤其是当这些副作用是在不同的子表达式中时
* 向别名变量的写入
* 超过 N 个运算符（N 应当为多少？）
* 依赖于微妙的优先级规则
* 使用了未定义行为（我们是否应当识别所有的未定义行为？）
* 实现定义的行为？
* ???

### <a name="Res-parens"></a>ES.41: 对运算符优先级不保准时应使用括号

##### 理由

避免错误。可读性。不是每个人都能记住运算符表格。

##### 示例

    const unsigned int flag = 2;
    unsigned int a = flag;
    
    if (a & flag != 0)  // 不好: 含义为 a&(flag != 0)

注意：我们建议程序员了解算术运算和逻辑运算的优先级表，但应当考虑当按位逻辑运算和其他运算符混合使用时需要采用括号。

    if ((a & flag) != 0)  // OK: 按预期工作

##### 注解

你应当了解足够的知识以避免在这样的情况下需要括号：

    if (a < 0 || a <= max) {
        // ...
    }

##### 强制实施

* 当按位逻辑运算符合其他运算符组合时进行标记。
* 当赋值运算符不是最左边的运算符时进行标记。
* ???

### <a name="Res-ptr"></a>ES.42: 保持单纯直接的指针使用方式

##### 理由

复杂的指针操作是一种重大的错误来源。

##### 注解

代之以使用 `gsl::span`。
指针[只应当指代单个对象](#Ri-array)。
指针算术是脆弱而易错的，是许多许多糟糕的 BUG 和安全漏洞的来源。
`span` 是一种用于访问数组对象的带有边界检查的安全类型。
以常量为下标来访问已知边界的数组，编译器可以进行验证。

##### 示例，不好

    void f(int* p, int count)
    {
        if (count < 2) return;
    
        int* q = p + 1;    // 不好
    
        ptrdiff_t d;
        int n;
        d = (p - &n);      // OK
        d = (q - p);       // OK
    
        int n = *p++;      // 不好
    
        if (count < 6) return;
    
        p[4] = 1;          // 不好
    
        p[count - 1] = 2;  // 不好
    
        use(&p[0], 3);     // 不好
    }

##### 示例，好

    void f(span<int> a) // 好多了：函数声明中使用了 span
    {
        if (a.size() < 2) return;
    
        int n = a[0];      // OK
    
        span<int> q = a.subspan(1); // OK
    
        if (a.size() < 6) return;
    
        a[4] = 1;          // OK
    
        a[a.size() - 1] = 2;  // OK
    
        use(a.data(), 3);  // OK
    }

##### 注解

用变量做下标，对于工具和人类来说都是很难将其验证为安全的。
`span` 是一种用于访问数组对象的带有运行时边界检查的安全类型。
`at()` 是可以保证单次访问进行边界检查的另一种替代方案。
如果需要用迭代器来访问数组的话，应使用构造于数组之上的 `span` 所提供的迭代器。

##### 示例，不好

    void f(array<int, 10> a, int pos)
    {
        a[pos / 2] = 1; // 不好
        a[pos - 1] = 2; // 不好
        a[-1] = 3;    // 不好（但易于被工具查出） - 没有替代方案，请勿这样做
        a[10] = 4;    // 不好（但易于被工具查出） - 没有替代方案，请勿这样做
    }

##### 示例，好

使用 `span`：

    void f1(span<int, 10> a, int pos) // A1: 将参数类型改为使用 span
    {
        a[pos / 2] = 1; // OK
        a[pos - 1] = 2; // OK
    }
    
    void f2(array<int, 10> arr, int pos) // A2: 增加局部的 span 并使用之
    {
        span<int> a = {arr.data(), pos};
        a[pos / 2] = 1; // OK
        a[pos - 1] = 2; // OK
    }

使用 `at()`：

    void f3(array<int, 10> a, int pos) // 替代方案 B: 用 at() 进行访问
    {
        at(a, pos / 2) = 1; // OK
        at(a, pos - 1) = 2; // OK
    }

##### 示例，不好

    void f()
    {
        int arr[COUNT];
        for (int i = 0; i < COUNT; ++i)
            arr[i] = i; // 不好，不能使用非常量索引
    }

##### 示例，好

使用 `span`：

    void f1()
    {
        int arr[COUNT];
        span<int> av = arr;
        for (int i = 0; i < COUNT; ++i)
            av[i] = i;
    }

使用 `span` 和基于范围的 `for`：

    void f1a()
    {
         int arr[COUNT];
         span<int, COUNT> av = arr;
         int i = 0;
         for (auto& e : av)
             e = i++;
    }

使用 `at()` 进行访问：

    void f2()
    {
        int arr[COUNT];
        int i = 0;
        for (int i = 0; i < COUNT; ++i)
            at(arr, i) = i;
    }

使用基于范围的 `for`：

    void f3()
    {
         int arr[COUNT];
         for (auto& e : arr)
             e = i++;
    }

##### 注解

工具可以提供重写能力，以将涉及动态索引表达式的数组访问替换为使用 `at()` 进行访问：

    static int a[10];
    
    void f(int i, int j)
    {
        a[i + j] = 12;      // 不好，可以重写为 ...
        at(a, i + j) = 12;  // OK - 带有边界检查
    }

##### 示例

把数组转变为指针（语言基本上总会这样做），移除了进行检查的机会，因此应当予以避免

    void g(int* p);
    
    void f()
    {
        int a[5];
        g(a);        // 不好：是要传递一个数组吗？
        g(&a[0]);    // OK：传递单个对象
    }

如果要传递数组的话，应该这样：

    void g(int* p, size_t length);  // 老的（危险）代码
    
    void g1(span<int> av); // 好多了：改动了 g()。
    
    void f()
    {
        int a[5];
        span<int> av = a;
    
        g(av.data(), av.size());   // OK, 如果没有其他选择的话
        g1(a);                     // OK - 这里没有衰变，而是使用了隐式的 span 构造函数
    }

##### 强制实施

* 对任何在指针类型的表达式上进行的产生指针类型的值的算术运算进行标记。
* 对任何数组类型的表达式或变量（无论是静态数组还是 `std::array`）上进行索引的表达式，若其索引不是值为从 `0` 到数组上界之内的编译期常量表达式，则进行标记。
* 对任何可能依赖于从数组类型向指针类型的隐式转换的表达式进行标记。

本条规则属于[边界安全性剖面配置](#SS-bounds)。


### <a name="Res-order"></a>ES.43: 避免带有未定义的求值顺序的表达式

##### 理由

你没办法搞清楚这种代码会做什么。可移植性。
即便它做到了对你可能有意义的事情，它可能在别的编译器（比如你的编译器的下个版本）或者不同的优化设置中作出不同的事情。

##### 注解

C++17 收紧了有关求值顺序的规则：
除了赋值中从右向左，以及函数实参求值顺序未指明外均为从左向右。

不过，要记住你的代码可能是由 C++17 之前的编译器进行编译的（比如通过复制粘贴），请勿自作聪明。

##### 示例

    v[i] = ++i;   //  其结果是未定义的

一条不错经验法则是，你不应当在一个表达式中两次读取你所写入的值。

##### 强制实施

可以由优秀的分析器检测出来。

### <a name="Res-order-fct"></a>ES.44: 不要对函数参数求值顺序有依赖

##### 理由

因为这种顺序是未定义的。

##### 注解

C++17 收紧了有关求值顺序的规则，但函数实参求值顺序仍然是未指明的。

##### 示例

    int i = 0;
    f(++i, ++i);

这个调用很可能是 `f(0, 1)` 或 `f(1, 0)`，但你不知道是哪个。
技术上讲，其行为是未定义的。
在 C++17 中这段代码没有未定义行为，但仍未指定是哪个实参被首先求值。

##### 示例

重载运算符可能导致求值顺序问题：

    f1()->m(f2());          // m(f1(), f2())
    cout << f1() << f2();   // operator<<(operator<<(cout, f1()), f2())

在 C++17 中，这些例子将按预期工作（自左向右），而赋值则按自右向左求值（`=` 正是自右向左绑定的）

    f1() = f2();    // C++14 中为未定义行为；C++17 中 f2() 在 f1() 之前求值

##### 强制实施

可以由优秀的分析器检测出来。

### <a name="Res-magic"></a>ES.45: 避免"魔法常量"，采用符号化常量

##### 理由

表达式中内嵌的无名的常量很容易被忽略，而且经常难于理解：

##### 示例

    for (int m = 1; m <= 12; ++m)   // 请勿如此: 魔法常量 12
        cout << month[m] << '\n';

不是所有人都知道一年中有 12 个月份，号码是 1 到 12。更好的做法是：

    // 月份索引值为 1..12
    constexpr int first_month = 1;
    constexpr int last_month = 12;
    
    for (int m = first_month; m <= last_month; ++m)   // 好多了
        cout << month[m] << '\n';

更好的做法是，不要暴露常量：

    for (auto m : month)
        cout << m << '\n';

##### 强制实施

标记代码中的字面量。让 `0`，`1`，`nullptr`，`\n'`，`""`，以及某个确认列表中的其他字面量通过检查。

### <a name="Res-narrowing"></a>ES.46: 避免丢失数据（窄化、截断）的算术转换

##### 理由

窄化转换会销毁信息，通常是不期望发生的。

##### 示例，不好

关键的例子就是基本的窄化：

    double d = 7.9;
    int i = d;    // 不好: 窄化: i 变为了 7
    i = (int) d;  // 不好: 我们打算声称这样的做法仍然不够明确
    
    void f(int x, long y, double d)
    {
        char c1 = x;   // 不好: 窄化
        char c2 = y;   // 不好: 窄化
        char c3 = d;   // 不好: 窄化
    }

##### 注解

指导方针支持库提供了一个 `narrow_cast` 操作，用以指名发生窄化是可接受的，以及一个 `narrow`（"窄化判定"）当窄化将会损失信息时将会抛出一个异常：

    i = narrow_cast<int>(d);   // OK (明确需要): 窄化: i 变为了 7
    i = narrow<int>(d);        // OK: 抛出 narrowing_error

其中还包含了一些含有损失的算术强制转换，比如从负的浮点类型到无符号整型类型的强制转换：

    double d = -7.9;
    unsigned u = 0;
    
    u = d;                          // 不好
    u = narrow_cast<unsigned>(d);   // OK (明确需要): u 变为了 4294967289
    u = narrow<unsigned>(d);        // OK: 抛出 narrowing_error

##### 强制实施

优良的分析器可以检测到所有的窄化转换。不过，对所有的窄化转换都进行标记将带来大量的误报。建议的做法是：

* 标记出所有的浮点向整数转换（可能只有 `float`->`char` 和 `double`->`int`。这里有问题！需要数据支持）。
* 标记出所有的 `long`->`char`（我怀疑 `int`->`char` 非常常见。这里有问题！需要数据支持）。
* 在函数参数上发生的窄化转换特别值得怀疑。

### <a name="Res-nullptr"></a>ES.47: 使用 `nullptr` 而不是 `0` 或 `NULL`

##### 理由

可读性。最小化意外：`nullptr` 不可能和 `int` 混淆。
`nullptr` 还有一个严格定义的（非常严格）类型，且因此
可以在类型推断可能在 `NULL` 或 `0` 上犯错的场合中仍能
正常工作。

##### 示例

考虑：

    void f(int);
    void f(char*);
    f(0);         // 调用 f(int)
    f(nullptr);   // 调用 f(char*)

##### 强制实施

对用作指针的 `0` 和 `NULL` 进行标记。可以用简单的程序变换来达成这种变换。

### <a name="Res-casts"></a>ES.48: 避免强制转换

##### 理由

强制转换是众所周知的错误来源。它们使得一些优化措施变得不可靠。

##### 示例，不好

    double d = 2;
    auto p = (long*)&d;
    auto q = (long long*)&d;
    cout << d << ' ' << *p << ' ' << *q << '\n';

你觉得这段代码会打印出什么呢？结果最好由实现定义。我得到的是

    2 0 4611686018427387904

加上这些

    *q = 666;
    cout << d << ' ' << *p << ' ' << *q << '\n';

得到的是

    3.29048e-321 666 666

奇怪吗？我很庆幸程序没有崩溃掉。

##### 注解

写下强制转换的程序员通常认为他们知道所做的是什么事情，
或者写出强制转换能让程序"更易读"。
而实际上，他们这样经常会禁止掉使用值的一些一般规则。
如果存在正确的函数的话，重载决议和模板实例化通常都能挑选出正确的函数。
如果没有的话，则可能本应如此，而不应该进行某种局部的修补（强制转换）。

##### 注解

强制转换在系统编程语言中是必要的。例如，否则我们怎么
才能把设备寄存器的地址放入一个指针呢？然而，强制转换
却被严重过度使用了，而且也是一种主要的错误来源。

##### 注解

当你觉得需要进行大量强制转换时，可能存在一个基本的设计问题。

##### 例外

【强制转换为 `(void)`】是由标准认可的关闭 `[[nodiscard]]` 警告的方法。当调用带有 `[[nodiscard]]` 返回的函数而你又有意要丢弃其返回值时，应当首先深入思考这是不是确实是个好主意（通常，这个函数或者使用了 `[[nodiscard]]` 的返回类型的作者，当初确实是有充分理由的），但要是你仍然觉得这样做是合适的，而且你的代码评审者也同意的话，就可以写出 `(void)` 来关闭这个警告。

##### 替代方案

强制转换被广泛（误）用了。现代 C++ 已经提供了一些规则和语言构造，消除了许多语境中对强制转换的需求，比如

* 使用模板
* 使用 `std::variant`
* 借助良好定义的，安全的，指针类型之间的隐式转换

##### 强制实施

* 强制消除除了带有 `[[nodiscard]]` 返回的函数上之外的 C 风格的强制转换。
* 当存在许多函数风格的强制转换时给出警告（显而易见的问题是如何量化"许多"）。
* [类型剖面配置](#Pro-type-reinterpretcast)禁用了 `reinterpret_cast`。
* 对指针类型之间的[同一强制转换](#Pro-type-identitycast)给出警告，这之中的源类型和目标类型相同(#Pro-type-identitycast)。
* 当指针强制转换可以为[隐式转换](#Pro-type-implicitpointercast)时给出警告。

### <a name="Res-casts-named"></a>ES.49: 当必须使用强制转换时，使用具名的强制转换

##### 理由

可读性。避免错误。
具名的强制转换比 C 风格或函数风格的强制转换更加特殊，允许编译器捕捉到某些错误。

具名的强制转换包括：

* `static_cast`
* `const_cast`
* `reinterpret_cast`
* `dynamic_cast`
* `std::move`         // `move(x)` 是指代 `x` 的右值引用
* `std::forward`      // `forward<T>(x)` 是指代 `x` 的左值或右值引用（取决于 `T`）
* `gsl::narrow_cast`  // `narrow_cast<T>(x)` 就是 `static_cast<T>(x)`
* `gsl::narrow`       // `narrow<T>(x)` 在当 `static_cast<T>(x) == x` 时即为 `static_cast<T>(x)` 否则会抛出 `narrowing_error`

##### 示例

    class B { /* ... */ };
    class D { /* ... */ };
    
    template<typename D> D* upcast(B* pb)
    {
        D* pd0 = pb;                        // 错误：不存在从 B* 向 D* 的隐式转换
        D* pd1 = (D*)pb;                    // 合法，但干了什么？
        D* pd2 = static_cast<D*>(pb);       // 错误：D 并非派生于 B
        D* pd3 = reinterpret_cast<D*>(pb);  // OK：你自己负责！
        D* pd4 = dynamic_cast<D*>(pb);      // OK：返回 nullptr
        // ...
    }

这个例子是从真实世界的 BUG 合成的，其中 `D` 曾经派生于 `B`，但某个人重构了继承层次。
C 风格的强制转换很危险，因为它可以进行任何种类的转换，使我们丧失了今后受保护不犯错的机会。

##### 注解

当在类型之间进行没有信息丢失的转换时（比如从 `float` 到
`double` 或者从 `int64` 到 `int32`），可以代之以使用花括号初始化。

    double d {some_float};
    int64_t i {some_int32};

这样做明确了有意进行类型转换，而且同样避免了
发生可能导致精度损失的结果的类型转换。（比如说，
试图用这种风格来从 `double` 初始化 `float` 会导致
编译错误。）

##### 注解

`reinterpret_cast` 可以很基础，但其基础用法（如将机器地址转化为指针）并不是类型安全的：

    auto p = reinterpret_cast<Device_register>(0x800);  // 天生危险


##### 强制实施

* 对 C 风格和函数式的强制转换进行标记。
* [类型剖面配置](#Pro-type-reinterpretcast)禁用了 `reinterpret_cast`。
* [类型剖面配置](#Pro-type-arithmeticcast)对于在算术类型之间使用 `static_cast` 时给出警告。

### <a name="Res-casts-const"></a>ES.50: 不要强制掉 `const`

##### 理由

这是在 `const` 上说谎。
若变量确实声明为 `const`，则"强制掉 `const`"是未定义的行为。

##### 示例，不好

    void f(const int& x)
    {
        const_cast<int&>(x) = 42;   // 不好
    }
    
    static int i = 0;
    static const int j = 0;
    
    f(i); // 暗藏的副作用
    f(j); // 未定义的行为

##### 示例

有时候，你可能倾向于借助 `const_cast` 来避免代码重复，比如两个访问函数仅在是否 `const` 上有区别而实现相似的情况。例如：

    class Bar;
    
    class Foo {
    public:
        // 不好，逻辑重复
        Bar& get_bar() {
            /* 获取 my_bar 的非 const 引用前后的复杂逻辑 */
        }
    
        const Bar& get_bar() const {
            /* 获取 my_bar 的 const 引用前后的相同的复杂逻辑 */
        }
    private:
        Bar my_bar;
    };

应当改为共享实现。通常可以直接让非 `const` 函数来调用 `const` 函数。不过当逻辑复杂的时候这可能会导致下面这样的模式，仍然需要借助于 `const_cast`：

    class Foo {
    public:
        // 不大好，非 const 函数调用 const 版本但借助于 const_cast
        Bar& get_bar() {
            return const_cast<Bar&>(static_cast<const Foo&>(*this).get_bar());
        }
        const Bar& get_bar() const {
            /* 获取 my_bar 的 const 引用前后的复杂逻辑 */
        }
    private:
        Bar my_bar;
    };

虽然这个模式如果恰当应用的话是安全的（因为调用方必然以一个非 `const` 对象来开始），但这并不理想，因为其安全性无法作为检查工具的规则而自动强制实施。

换种方式，可以优先将公共代码放入一个公共辅助函数中，并将之作为模板以使其推断 `const`。这完全不会用到 `const_cast`：

    class Foo {
    public:                         // 好
              Bar& get_bar()       { return get_bar_impl(*this); }
        const Bar& get_bar() const { return get_bar_impl(*this); }
    private:
        Bar my_bar;
    
        template<class T>           // 好，推断出 T 是 const 还是非 const
        static auto get_bar_impl(T& t) -> decltype(t.get_bar())
            { /* 获取 my_bar 的可能为 const 的引用前后的复杂逻辑 */ }
    };

##### 例外

当调用 `const` 不正确的函数时，你可能需要强制掉 `const`。
应当优先将这种函数包装到内联的 `const` 正确的包装函数中，以将强制转换封装到一处中。

##### 示例

有时候，"强制掉 `const`"是为了允许对本来无法改动的对象中的某种临时性的信息进行更新操作。
其例子包括进行缓存，备忘，以及预先计算等。
这样的例子，通常可以通过使用 `mutable` 或者通过一层间接进行处理，而同使用 `const_cast` 一样甚或比之更好。

考虑为昂贵操作将之前所计算的结果保留下来：

    int compute(int x); // 为 x 计算一个值；假设这是昂贵的
    
    class Cache {   // 为 int->int 操作实现一种高速缓存的某个类型
    public:
        pair<bool, int> find(int x) const;   // 有针对 x 的值吗？
        void set(int x, int v);             // 使 y 成为针对 x 的值
        // ...
    private:
        // ...
    };
    
    class X {
    public:
        int get_val(int x)
        {
            auto p = cache.find(x);
            if (p.first) return p.second;
            int val = compute(x);
            cache.set(x, val); // 插入针对 x 的值
            return val;
        }
        // ...
    private:
        Cache cache;
    };

这里的 `get_val()` 逻辑上是个常量，因此我们想使其成为 `const` 成员。
为此我们仍然需要改动 `cache`，因此人们有时候会求助于 `const_cast`：

    class X {   // 基于强制转换的可疑的方案
    public:
        int get_val(int x) const
        {
            auto p = cache.find(x);
            if (p.first) return p.second;
            int val = compute(x);
            const_cast<Cache&>(cache).set(x, val);   // 很难看
            return val;
        }
        // ...
    private:
        Cache cache;
    };

幸运的是，有一种更好的方案：
将 `cache` 称为即便对于 `const` 对象来说也是可改变的：

    class X {   // 更好的方案
    public:
        int get_val(int x) const
        {
            auto p = cache.find(x);
            if (p.first) return p.second;
            int val = compute(x);
            cache.set(x, val);
            return val;
        }
        // ...
    private:
        mutable Cache cache;
    };

另一种替代方案是存储指向 `cache` 的指针：

    class X {   // OK，但有点麻烦的方案
    public:
        int get_val(int x) const
        {
            auto p = cache->find(x);
            if (p.first) return p.second;
            int val = compute(x);
            cache->set(x, val);
            return val;
        }
        // ...
    private:
        unique_ptr<Cache> cache;
    };

这个方案最灵活，但需要显式进行 `*cache` 的构造和销毁
（最可能发生于 `X` 的构造函数和析构函数中）。

无论采用哪种形式，在多线程代码中都需要保护对 `cache` 的数据竞争，可能需要使用一个 `std::mutex`。

##### 强制实施

* 标记 `const_cast`。
* 本条规则属于[类型安全性剖面配置](#Pro-type-constcast)。

### <a name="Res-range-checking"></a>ES.55: 避免发生对范围检查的需要

##### 理由

无法溢出的构造时不会溢出的（而且通常运行得更快）：

##### 示例

    for (auto& x : v)      // 打印 v 的所有元素
        cout << x << '\n';
    
    auto p = find(v, x);   // 在 v 中寻找 x

##### 强制实施

查找显式的范围检查，并启发式地给出替代方案建议。

### <a name="Res-move"></a>ES.56: 仅在确实需要明确移动某个对象到别的作用域时才使用 `std::move()`

##### 理由

我们用移动而不是复制，以避免发生重复并提升性能。

一次移动通常会遗留一个空对象（[C.64](#Rc-move-semantic)），这可能令人意外甚至很危险，因此我们试图避免从左值进行移动（它们可能随后会被访问到）。

##### 注解

当来源是右值（比如 `return` 的值或者函数的结果）时就会隐式地进行移动，因此请不要在这些情况下明确写下 `move` 而无意义地使代码复杂化。可以代之以编写简短的返回值的函数，这样的话无论是函数的返回还是调用方的返回值接收，都会很自然地得到优化。

一般来说，遵循本文档中的指导方针（包括不要让变量的作用域无必要地变大，编写返回值的简短函数，返回局部变量等），有助于消除大多数对显式使用 `std::move` 的需要。

显式的 `move` 需要用于把某个对象明确移动到另一个作用域，尤其是将其传递给某个"接收器"函数，以及移动操作自身（移动构造函数，移动赋值运算符）和交换（`swap`）操作的实现之中。

##### 示例，不好

    void sink(X&& x);   // sink 接收 x 的所有权
    
    void user()
    {
        X x;
        // 错误: 无法将作者绑定到右值引用
        sink(x);
        // OK: sink 接收了 x 的内容，x 随即必须假定为空
        sink(std::move(x));
    
        // ...
    
        // 可能是个错误
        use(x);
    }

通常来说，`std::move()` 都用做某个 `&&` 形参的实参。
而这点之后，应当假定对象已经被移走（参见 [C.64](#Rc-move-semantic)），而直到首次向它设置某个新值之前，请勿再次读取它的状态。

    void f() {
        string s1 = "supercalifragilisticexpialidocious";
    
        string s2 = s1;             // ok, 接收了一个副本
        assert(s1 == "supercalifragilisticexpialidocious");  // ok
    
        // 不好, 如果你打算保留 s1 的值的话
        string s3 = move(s1);
    
        // 不好, assert 很可能会失败, s1 很可能被改动了
        assert(s1 == "supercalifragilisticexpialidocious");
    }

##### 示例

    void sink(unique_ptr<widget> p);  // 将 p 的所有权传递给 sink()
    
    void f() {
        auto w = make_unique<widget>();
        // ...
        sink(std::move(w));               // ok, 交给 sink()
        // ...
        sink(w);    // 错误: unique_ptr 经过严格设计，你无法复制它
    }

##### 注解

`std::move()` 经过伪装的向 `&&` 的强制转换；其自身并不会移动任何东西，但会把具名的对象标记为可被移动的候选者。
语言中已经了解了对象可以被移动的一般情况，尤其是从函数返回时，因此请不要用多余的 `std::move()` 使代码复杂化。

绝不要仅仅因为听说过"这样更加高效"就使用 `std::move()`。
通常来说，请不要相信那些没有支持数据的有关"效率"的断言。(???).
通常来说，请不要无理由地使代码复杂化。(??)
绝不要在 const 对象上 `std::move()`，它只会暗中将其转变成一个副本（参见 [Meyers15](#Meyers15) 的条款 23)。

##### 示例，不好

    vector<int> make_vector() {
        vector<int> result;
        // ... 加载 result 的数据
        return std::move(result);       // 不好; 直接写 "return result;" 即可
    }

绝不要写 `return move(local_variable);`，这是因为语言已经知道这个变量是移动的候选了。
在这段代码中用 `move` 并不会带来帮助，而且可能实际上是有害的，因为它创建了局部变量的一个额外引用别名，而在某些编译器中这回对 RVO（返回值优化）造成影响。


##### 示例，不好

    vector<int> v = std::move(make_vector());   // 不好; 这个 std::move 完全是多余的

绝不在返回值上使用 `move`，如 `x = move(f());`，其中的 `f` 按值返回。
语言已经知道返回值是临时对象而且可以被移动。

##### 示例

    void mover(X&& x) {
        call_something(std::move(x));         // ok
        call_something(std::forward<X>(x));   // 不好, 请勿对右值引用 std::forward
        call_something(x);                    // 可疑  为什么不用std:: move?
    }
    
    template<class T>
    void forwarder(T&& t) {
        call_something(std::move(t));         // 不好, 请勿对转发引用 std::move
        call_something(std::forward<T>(t));   // ok
        call_something(t);                    // 可疑, 为什么不用 std::forward?
    }

##### 强制实施

* 对于 `std::move(x)` 的使用，当 `x` 是右值，或者语言已经将其当做右值，这包括 `return std::move(local_variable);` 以及在按值返回的函数上的 `std::move(f())`，进行标记
* 当没有接受 `const S&` 的函数重载来处理左值时，对接受 `S&&` 参数的函数进行标记。
* 当将经过 `std::move` 的实参传递给某个形参时进行标记，除非形参的类型为右值引用 `X&&`，或者类型是只能移动的而该形参为按值传递。
* 当对转发引用（`T&&` 其中 `T` 为模板参数类型）使用 `std::move` 时进行标记。应当代之以使用 `std::forward`。
* 当对并非非 const 右值引用的变量使用 `std::move` 时进行标记。（这是前一条规则的更一般的情况，以覆盖非转发的情况。）
* 当对右值引用（`X&&` 其中 `X` 为独立类型）使用 `std::forward` 时进行标记。应当代之以使用 `std::move`。
* 当对并非转发引用使用 `std::forward` 时进行标记。（这是前一条规则的更一般的情况，以覆盖非移动的情况。）
* 如果对象潜在地被移动走之后的下一个操作是 `const` 操作的话，则进行标记；首先应当交错进行一个非 `const` 操作，最好是赋值，以首先对对象的值进行重置。

### <a name="Res-new"></a>ES.60: 避免在资源管理函数之外使用 `new` 和 `delete`

##### 理由

应用程序代码中的直接资源管理既易错又麻烦。

##### 注解

通常也被称为"禁止裸 `new`！"规则。

##### 示例，不好

    void f(int n)
    {
        auto p = new X[n];   // n 个默认构造的 X
        // ...
        delete[] p;
    }

`...` 部分中的代码可能导致 `delete` 永远不会发生。

**参见**: [R: 资源管理](#S-resource)

##### 强制实施

对裸的 `new` 和裸的 `delete` 进行标记。

### <a name="Res-del"></a>ES.61: 用 `delete[]` 删除数组，用 `delete` 删除非数组对象

##### 理由

这正是语言的要求，而且所犯的错误将导致资源释放的错误以及内存破坏。

##### 示例，不好

    void f(int n)
    {
        auto p = new X[n];   // n 个默认初始化的 X
        // ...
        delete p;   // 错误: 仅仅删除了对象 p，而并未删除数组 p[]
    }

##### 注解

这个例子不仅像上前一个例子一样违反了[禁止裸 `new` 规则](#Res-new)，它还有更多的问题。

##### 强制实施

* 如果 `new` 和 `delete` 在同一个作用域中的话，就可以标记出现错误。
* 如果 `new` 和 `delete` 出现在构造函数/析构函数对之中的话，就可以标记出现错误。

### <a name="Res-arr2"></a>ES.62: 不要在不同的数组之间进行指针比较

##### 理由

这样做的结果是未定义的。

##### 示例，不好

    void f()
    {
        int a1[7];
        int a2[9];
        if (&a1[5] < &a2[7]) {}       // 不好: 未定义
        if (0 < &a1[5] - &a2[7]) {}   // 不好: 未定义
    }

##### 注解

这个例子中有许多问题。

##### 强制实施

???

### <a name="Res-slice"></a>ES.63: 不要产生切片

##### 理由

切片——亦即使用赋值或初始化而只对对象的一部分进行复制——通常会导致错误，
这是因为对象总是被当成是一个整体。
在罕见的进行蓄意的切片的代码中，其代码会让人意外。

##### 示例

    class Shape { /* ... */ };
    class Circle : public Shape { /* ... */ Point c; int r; };
    
    Circle c {{0, 0}, 42};
    Shape s {c};    // 仅复制构造了 Circle 中的 Shape 部分
    s = c;          // 仅复制赋值了 Circle 中的 Shape 部分
    
    void assign(const Shape& src, Shape& dest) {
        dest = src;
    }
    Circle c2 {{1, 1}, 43};
    assign(c, c2);   // 噢，传递的并不是整个状态
    assert(c == c2); // 如果提供复制操作，就也得提供比较操作，
                     //   但这里很可能返回 false

这样的结果是无意义的，因为不会把中心和半径从 `c` 复制给 `s`。
针对这个的第一条防线是[将基类 `Shape` 定义为不允许这样做](#Rc-copy-virtual)。

##### 替代方案

如果确实需要切片的话，应当为之定义一个明确的操作。
这会避免读者产生混乱。
例如：

    class Smiley : public Circle {
        public:
        Circle copy_circle();
        // ...
    };
    
    Smiley sm { /* ... */ };
    Circle c1 {sm};  // 理想情况下由 Circle 的定义所禁止
    Circle c2 {sm.copy_circle()};

##### 强制实施

针对切片给出警告。

### <a name="Res-construct"></a>ES.64: 使用 `T{e}` 写法来进行构造

##### 理由

对象构造语法 `T{e}` 明确了所需进行的构造。
对象构造语法 `T{e}` 不允许发生窄化。
`T{e}` 是唯一安全且通用的由表达式 `e` 构造一个 `T` 类型的值的表达式。
强制转换的写法 `T(e)` 和 `(T)e` 既不安全也不通用。

##### 示例

对于内建类型，构造的写法保护了不发生窄化和重解释

    void use(char ch, int i, double d, char* p, long long lng)
    {
        int x1 = int{ch};     // OK，但多余
        int x2 = int{d};      // 错误：double->int 窄化；如果需要的话应使用强制转换
        int x3 = int{p};      // 错误：指针->int；如果确实需要的话应使用 reinterpret_cast
        int x4 = int{lng};    // 错误：long long->int 窄化；如果需要的话应使用强制转换
    
        int y1 = int(ch);     // OK，但多余
        int y2 = int(d);      // 不好：double->int 窄化；如果需要的话应使用强制转换
        int y3 = int(p);      // 不好：指针->int；如果确实需要的话应使用 reinterpret_cast
        int y4 = int(lng);    // 不好：long long->int 窄化；如果需要的话应使用强制转换
    
        int z1 = (int)ch;     // OK，但多余
        int z2 = (int)d;      // 不好：double->int 窄化；如果需要的话应使用强制转换
        int z3 = (int)p;      // 不好：指针->int；如果确实需要的话应使用 reinterpret_cast
        int z4 = (int)lng;    // 不好：long long->int 窄化；如果需要的话应使用强制转换
    }

整数和指针之间的转换，在使用 `T(e)` 和 `(T)e` 时是由实现定义的，
而且在不同整数和指针大小的平台之间不可移植。

##### 注解

[避免强制转换](#Res-casts)（显式类型转换），如果必须要做的话[优先采用具名强制转换](#Res-casts-named)。

##### 注解

当没有歧义时，可以不写 `T{e}` 中的 `T`。

    complex<double> f(complex<double>);
    
    auto z = f({2*pi,1});

##### 注解

对象构造语法是最通用的[初始化式语法](#Res-list)。

##### 例外

`std::vector` 和其他的容器是在 `{}` 作为对象构造语法之前定义的。
考虑：

    vector<string> vs {10};                           // 十个空字符串
    vector<int> vi1 {1, 2, 3, 4, 5, 6, 7, 8, 9, 10};  // 十个元素 1..10
    vector<int> vi2 {10};                             // 一个值为 10 的元素

如何得到包含十个默认初始化的 `int` 的 `vector`？

    vector<int> v3(10); // 十个值为 0 的元素

使用 `()` 而不是 `{}` 作为元素数量是一种约定（源于 1980 年代早期），很难改变，
但仍然是一个设计错误：对于元素类型与元素数量可能发生混淆的容器，必须解决
其中的歧义。
约定的方案是将 `{10}` 解释为单个元素的列表，而用 `(10)` 来指定大小。

不应当在新代码中重复这个错误。
可以定义一个类型来表示元素的数量：

    struct Count { int n; };
    
    template<typename T>
    class Vector {
    public:
        Vector(Count n);                     // n 个默认构造的元素
        Vector(initializer_list<T> init);    // init.size() 个元素
        // ...
    };
    
    Vector<int> v1{10};
    Vector<int> v2{Count{10}};
    Vector<Count> v3{Count{10}};    // 这里仍有一个很小的问题

剩下的主要问题就是为 `Count` 找个合适的名字了。

##### 强制实施

标记 C 风格的 `(T)e` 和函数式风格的 `T(e)` 强制转换。


### <a name="Res-deref"></a>ES.65: 不要解引用无效指针

##### 理由

解引用如 `nullptr` 这样的无效指针是未定义的行为，通常会导致程序立刻崩溃，
产生错误结果，或者内存被损坏。

##### 注解

本条规则是显然并且广为知晓的语言规则，但其可能难于遵守。
这需要良好的编码风格，程序库支持，以及静态分析来消除违反情况而不耗费大量开销。
这正是[C++'s resource- and type-safety model](#Stroustrup15)中所讨论的主要部分。

**参见**：

* 使用 [RAII](#Rr-raii) 以避免生存期问题。
* 使用 [unique_ptr](#Rf-unique_ptr) 以避免生存期问题。
* 使用 [shared_ptr](#Rf-shared_ptr) 以避免生存期问题。
* 当不可能出现 `nullptr` 时应使用[引用](#Rf-ptr-ref)。
* 使用 [not_null](#Rf-not_null) 以尽早捕捉到预期外的 `nullptr`。
* 使用[边界剖面配置](#SS-bounds)以避免范围错误。


##### 示例

    void f()
    {
        int x = 0;
        int* p = &x;
    
        if (condition()) {
            int y = 0;
            p = &y;
        } // p 失效
    
        *p = 42;            // 不好，若走了上面的分支则 p 无效
    }

为解决这个问题，要么应当扩展指针打算指代的这个对象的生存期，要么应当缩短指针的生存期（将解引用移动到所指代的对象生存期结束之前进行）。

    void f1()
    {
        int x = 0;
        int* p = &x;
    
        int y = 0;
        if (condition()) {
            p = &y;
        }
    
        *p = 42;            // OK，p 可指向 x 或 y，而它们都仍在作用域中
    }

不幸的是，大多数无效指针问题都更加难于定位且更加难于解决。

##### 示例

    void f(int* p)
    {
        int x = *p; // 不好：如何确定 p 是否有效？
    }

有大量的这种代码存在。
它们大多数都能工作（经过了大量的测试），但其各自是很难确定 `p` 是否可能为 `nullptr` 的。
后果就是，这同样是错误的一大来源。
有许多方案试图解决这个潜在问题：

    void f1(int* p) // 处理 nullptr
    {
        if (!p) {
            // 处理 nullptr（分配，返回，抛出，使 p 指向什么，等等
        }
        int x = *p;
    }

测试 `nullptr` 的做法有两个潜在的问题：

* 当遇到 `nullptr` 时应当做什么并不总是明确的
* 其测试可能多余并且相对比较昂贵
* 这个测试是为了保护某种违例还是所需逻辑的一部分并不明显

<!-- comment needed for code block after list -->
    void f2(int* p) // 声称 p 不应当为 nullptr
    {
        Assert(p);
        int x = *p;
    }

这样，仅当打开了断言检查时才会有所耗费，而且会向编译器/分析器提供有用的信息。
当 C++ 出现契约的直接支持后，还可以做的更好：

    void f3(int* p) // 声称 p 不应当为 nullptr
        [[expects: p]]
    {
        int x = *p;
    }

或者，还可以使用 `gsl::not_null` 来保证 `p` 不为 `nullptr`。

    void f(not_null<int*> p)
    {
        int x = *p;
    }

这些只是关于 `nullptr` 的处理办法。
要知道还有其他出现无效指针的方式。

##### 示例

    void f(int* p)  // 老代码，没使用 owner
    {
        delete p;
    }
    
    void g()        // 老代码：使用了裸 new
    {
        auto q = new int{7};
        f(q);
        int x = *q; // 不好：解引用了无效指针
    }

##### 示例

    void f()
    {
        vector<int> v(10);
        int* p = &v[5];
        v.push_back(99); // 可能重新分配 v 中的元素
        int x = *p; // 不好：解引用了潜在的无效指针
    }

##### 强制实施

本条规则属于[生存期安全性剖面配置](#SS-lifetime)

* 当对指向已经超出作用域的对象的指针进行解引用时进行标记
* 当对可能已经通过赋值 `nullptr` 而无效的指针进行解引用时进行标记
* 当对可能已经因 `delete` 而无效的指针进行解引用时进行标记
* 当对指向已经失效的容器元素的的指针进行解引用时进行标记


## ES.stmt: 语句

语句控制了控制的流向（除了函数调用和异常抛出，它们是表达式）。

### <a name="Res-switch-if"></a>ES.70: 面临选择时，优先采用 `switch` 语句而不是 `if` 语句

##### 理由

* 可读性。
* 效率：`switch` 与常量进行比较，且通常比一个 `if`-`then`-`else` 链中的一系列测试获得更好的优化。
* `switch` 可以启用某种启发式的一致性检查。例如，是否某个 `enum` 的所有值都被覆盖了？如果没有的话，是否存在 `default`？

##### 示例

    void use(int n)
    {
        switch (n) {   // 好
        case 0:
            // ...
            break;
        case 7:
            // ...
            break;
        default:
            // ...
            break;
        }
    }

要好于：

    void use2(int n)
    {
        if (n == 0)   // 不好：以 if-then-else 链和一组常量进行比较
            // ...
        else if (n == 7)
            // ...
    }

##### 强制实施

对以 `if`-`then`-`else` 链条（仅）和常量进行比较的情况进行标记。

### <a name="Res-for-range"></a>ES.71: 面临选择时，优先采用范围式 `for` 语句而不是普通 `for` 语句

##### 理由

可读性。避免错误。效率。

##### 示例

    for (gsl::index i = 0; i < v.size(); ++i)   // 不好
            cout << v[i] << '\n';
    
    for (auto p = v.begin(); p != v.end(); ++p)   // 不好
        cout << *p << '\n';
    
    for (auto& x : v)    // OK
        cout << x << '\n';
    
    for (gsl::index i = 1; i < v.size(); ++i) // 接触了两个元素：无法作为范围式的 for
        cout << v[i] + v[i - 1] << '\n';
    
    for (gsl::index i = 0; i < v.size(); ++i) // 可能具有副作用：无法作为范围式的 for
        cout << f(v, &v[i]) << '\n';
    
    for (gsl::index i = 0; i < v.size(); ++i) { // 循环体中混入了循环变量：无法作为范围式 for
        if (i % 2 == 0)
            continue;   // 跳过偶数元素
        else
            cout << v[i] << '\n';
    }

人类或优良的静态分析器可以确定，其实在 `f(v, &v[i])` 中的 `v` 的上并不真的存在副作用，因此这个循环可以被重写。

在循环体中"混入循环变量"的情况通常是最好进行避免的。

##### 注解

不要在范围式 `for` 循环中使用昂贵的循环变量副本：

    for (string s : vs) // ...

这将会对 `vs` 中的每个元素复制给 `s`。这样好一点：

    for (string& s : vs) // ...

更好的做法是，当循环变量不会被修改或复制时：

    for (const string& s : vs) // ...

##### 强制实施

查看循环，如果一个传统的循环仅会查看序列中的各个元素，而且其对这些元素所做的事中没有发生副作用，则将该循环重写为范围式的 `for` 循环。

### <a name="Res-for-while"></a>ES.72: 当存在显然的循环变量时，优先采用 `for` 语句而不是 `while` 语句

##### 理由

可读性：循环的全部逻辑都"直观可见"。循环变量的作用域是有限的。

##### 示例

    for (gsl::index i = 0; i < vec.size(); i++) {
        // 干活
    }

##### 示例，不好

    int i = 0;
    while (i < vec.size()) {
        // 干活
        i++;
    }

##### 强制实施

???

### <a name="Res-while-for"></a>ES.73: 当没有显然的循环变量时，优先采用 `while` 语句而不是 `for` 语句

##### 理由

可读性。

##### 示例

    int events = 0;
    for (; wait_for_event(); ++events) {  // 不好，含糊
        // ...
    }

这个"事件循环"会误导人，计数器 `events` 跟循环条件（`wait_for_event()`）并没有任何关系。
更好的做法是

    int events = 0;
    while (wait_for_event()) {      // 更好
        ++events;
        // ...
    }

##### 强制实施

对和 `for` 的条件不相关的 `for` 初始化式和 `for` 增量部分进行标记。

### <a name="Res-for-init"></a>ES.74: 优先在 `for` 语句的初始化部分中声明循环变量

##### 理由

限制循环变量的可见性到循环的作用域之内。
避免在循环之后将循环变量用于其他目的。

##### 示例

    for (int i = 0; i < 100; ++i) {   // 好: 变量 i 仅在循环内部可见
        // ...
    }

##### 示例，请勿如此

    int j;                            // 不好: j 在循环之外可见
    for (j = 0; j < 100; ++j) {
        // ...
    }
    // j 在这里仍然可见但并不需要这样

**参见**: [不要用一个变量来达成两个不相关的目的](#Res-recycle)。

##### 示例

    for (string s; cin >> s; ) {
        cout << s << '\n';
    }

##### 强制实施

如果在 `for` 语句中所修改的变量是在循环外面所声明的，且在循环之外并未用到，则给出警告。

**讨论**: 把循环变量的作用域限制在循环体中同样会极大地帮助优化器。识别出这个归纳变量仅在循环体中
可以访问，能够开启诸如代码外提、强度削弱、循环不变式代码移动等各种优化手段。

### <a name="Res-do"></a>ES.75: 避免使用 `do` 语句

##### 理由

可读性，避免错误。
其终止条件处于尾部（而这可能会被忽略），且其条件不会在第一时间进行检查。

##### 示例

    int x;
    do {
        cin >> x;
        // ...
    } while (x < 0);

##### 注解

确实有一些天才的例子中，`do` 语句是更简洁的方案，但有问题的更多。

##### 强制实施

标记 `do` 语句。

### <a name="Res-goto"></a>ES.76: 避免 `goto`

##### 理由

可读性，避免错误。存在对于人类更好的控制结构；`goto` 是用于机器生成的代码的。

##### 例外

跳出嵌套循环。
这种情况下应当总是向前跳出。

    for (int i = 0; i < imax; ++i)
        for (int j = 0; j < jmax; ++j) {
            if (a[i][j] > elem_max) goto finished;
            // ...
        }
    finished:
    // ...

##### 示例，不好

有相当数量的代码采用 C 风格的 goto-exit 惯用法：

    void f()
    {
        // ...
            goto exit;
        // ...
            goto exit;
        // ...
    exit:
        // ... 公共的清理代码 ...
    }

这是对析构函数的一种专门模仿。
应当将资源声明为带有清理的析构函数的包装类。
如果你由于某种原因无法用析构函数来处理所使用的各个变量的清理工作，
请考虑用 `gsl::finally()` 作为 `goto exit` 的一种简洁且更加可靠的替代方案。

##### 强制实施

* 标记 `goto`。更好的做法是标记出除了从嵌套内层循环中跳出到紧跟一组嵌套循环之后的语句的 `goto` 以外的所有 `goto`。

### <a name="Res-continue"></a>ES.77: 尽量减少循环中使用的 `break` 和 `continue`

##### 理由

在不平凡的循环体中，容易忽略掉 `break` 或 `continue`。

循环中的 `break` 和 `switch` 语句中的 `break` 的含义有很大的区别，
（而且循环中可以有 `switch` 语句，`switch` 的 `case` 中也可以有循环）。

##### 示例

    ???

##### 替代方案

通常，需要 `break` 的循环都是作为一个函数（算法）的良好候选者，其 `break` 将会变为 `return`。

    ???

通常，使用 `continue` 的循环都可以等价且同样简洁地用 `if` 语句来表达。

    ???

##### 注解

如果你确实要打断一个循环，使用 `break` 通常比使用诸如[修改循环变量](#Res-loop-counter)或 [`goto`](#Res-goto) 等其他方案更好：


##### 强制实施

???

### <a name="Res-break"></a>ES.78: 不要依靠 `switch` 语句中的隐含直落行为

##### 理由

总是以 `break` 来结束非空的 `case`。意外地遗漏 `break` 是一种相当常见的 BUG。
蓄意的控制直落（fall through）是维护的噩梦，应该罕见并被明确标示出来。

##### 示例

    switch (eventType) {
    case Information:
        update_status_bar();
        break;
    case Warning:
        write_event_log();
        // 不好 - 隐式的控制直落
    case Error:
        display_error_window();
        break;
    }

单个语句带有多个 `case` 标签是可以的：

    switch (x) {
    case 'a':
    case 'b':
    case 'f':
        do_something(x);
        break;
    }

在 `case` 标签中使用返回语句也是可以的：

    switch (x) {
    case 'a':
        return 1;
    case 'b':
        return 2;
    case 'c':
        return 3;
    }

##### 例外

在罕见的直落被视为合适行为的情况中。应当明确标示，并使用 `[[fallthrough]]` 标注：

    switch (eventType) {
    case Information:
        update_status_bar();
        break;
    case Warning:
        write_event_log();
        [[fallthrough]];
    case Error:
        display_error_window();
        break;
    }

##### 注解

##### 强制实施

对所有从非空的 `case` 隐式发生的直落进行标记。


### <a name="Res-default"></a>ES.79: `default`（仅）用于处理一般情况

##### 理由

代码清晰性。
提升错误检测的机会。

##### 示例

    enum E { a, b, c , d };
    
    void f1(E x)
    {
        switch (x) {
        case a:
            do_something();
            break;
        case b:
            do_something_else();
            break;
        default:
            take_the_default_action();
            break;
        }
    }

此处很明显有一种默认的动作，而情况 `a` 和 `b` 则是特殊情况。

##### 示例

不过当不存在默认动作而只想处理特殊情况时怎么办呢？
这种情况下，应当使用空的 `default`，否则没办法知道你确实处理了所有情况：

    void f2(E x)
    {
        switch (x) {
        case a:
            do_something();
            break;
        case b:
            do_something_else();
            break;
        default:
            // 其他情况无需动作
            break;
        }
    }

如果没有 `default` 的话，维护者以及编译器可能会合理地假定你有意处理了所有情况：

    void f2(E x)
    {
        switch (x) {
        case a:
            do_something();
            break;
        case b:
        case c:
            do_something_else();
            break;
        }
    }

你是忘记了情况 `d` 还是故意遗漏了它？
当有人向枚举中添加一种情况，而又未能对每个针对这些枚举符的 `switch` 中添加时，
容易出现这种遗忘 `case` 的情况。

##### 强制实施

针对某个枚举的 `switch` 语句，若其未能处理其所有枚举符且没有 `default`，则对其进行标记。
这样做对于某些代码库可能会产生大量误报；此时，可以仅标记那些处理了大多数情况而不是所有情况的 `switch` 语句
（这正是第一个 C++ 编译器曾经的策略）。

### <a name="Res-noname"></a>ES.84: 不要试图声明没有名字的局部变量

##### 理由

没有这种东西。
我们眼里看起来像是个无名变量的东西，对于编译器来说是一条由一个将会立刻离开作用域的临时对象所组成的语句。

##### 示例，不好

    void f()
    {
        lock<mutex>{mx};   // 不好
        // ...
    }

这里声明了一个无名的 `lock` 对象，它将在分号处立刻离开作用域。
这并不是一种少见的错误。
特别是，这个特别的例子会导致很难发觉的竞争条件。

##### 注解

无名函数实参是没问题的。

##### 强制实施

标记出仅有临时对象的语句。

### <a name="Res-empty"></a>ES.85: 让空语句显著可见

##### 理由

可读性。

##### 示例

    for (i = 0; i < max; ++i);   // 不好: 空语句很容易被忽略
    v[i] = f(v[i]);
    
    for (auto x : v) {           // 好多了
        // 空
    }
    v[i] = f(v[i]);

##### 强制实施

对并非块语句且不包含注释的空语句进行标记。

### <a name="Res-loop-counter"></a>ES.86: 避免在原生的 `for` 循环中修改循环控制变量

##### 理由

循环控制的第一行应当允许对循环中所发生的事情进行正确的推理。同时在循环的重复表达式和循环体之中修改循环计数器，是发生意外和 BUG 的一种经常性来源。

##### 示例

    for (int i = 0; i < 10; ++i) {
        // 未改动 i -- ok
    }
    
    for (int i = 0; i < 10; ++i) {
        //
        if (/* 某种情况 */) ++i; // 不好
        //
    }
    
    bool skip = false;
    for (int i = 0; i < 10; ++i) {
        if (skip) { skip = false; continue; }
        //
        if (/* 某种情况 */) skip = true;  // 有改善: 为两个概念使用了两个变量。
        //
    }

##### 强制实施

如果变量在循环控制的重复表达式和循环体中都潜在地进行更新（存在非 `const` 使用），则进行标记。


### <a name="Res-if"></a>ES.87: 请勿在条件上添加多余的 `==` 或 `!=`

##### 理由

这样可避免啰嗦，并消除了发生某些错误的机会。
有助于使代码风格保持一致性和协调性。

##### 示例

根据定义，`if` 语句，`while` 语句，以及 `for` 语句中的条件，选择 `true` 或 `false` 的取值。
数值与 `0` 相比较，指针值与 `nullptr` 相比较。

    // 这些都表示"当 `p` 不是 `nullptr` 时"
    if (p) { ... }            // 好
    if (p != 0) { ... }       // `!=0` 是多余的；不好：不要对指针用 0
    if (p != nullptr) { ... } // `!=nullptr` 是多余的，不建议如此

通常，`if (p)` 可解读为"如果 `p` 有效"，这正是程序员意图的直接表达，
而 `if (p != nullptr)` 则只是一种啰嗦的变通写法。

##### 示例

这条规则对于把声明式用作条件时尤其有用

    if (auto pc = dynamic_cast<Circle>(ps)) { ... } // 执行是按照 ps 指向某种 Circle 来进行的，好
    
    if (auto pc = dynamic_cast<Circle>(ps); pc != nullptr) { ... } // 不建议如此

##### 示例

要注意，条件中会实施向 `bool` 的隐式转换。
例如：

    for (string s; cin >> s; ) v.push_back(s);

这里会执行 `istream` 的 `operator bool()`。

##### 注解

明确地将整数和 `0` 进行比较通常并非是多余的。
因为（与指针和布尔值相反），整数通常都具有超过两个的有效值。
此外 `0`（零）还经常会用于代表成功。
因此，最好明确地进行比较。

    void f(int i)
    {
        if (i)            // 可疑
        // ...
        if (i == success) // 可能更好
        // ...
    }

一定要记住整数可以有超过两个值。

##### 示例，不好

众所周知，

    if(strcmp(p1, p2)) { ... }   // 这两个 C 风格的字符串相等吗？（错误！）

是一种常见的新手错误。
如果使用 C 风格的字符串，那么就必须好好了解 `<cstring>` 中的函数。
即便冗余地写为

    if(strcmp(p1, p2) != 0) { ... }   // 这两个 C 风格的字符串相等吗？（错误！）

也不会有效果。

##### 注解

表达相反的条件的最简单的方式就是使用一次取反：

    // 这些都表示"当 `p` 为 `nullptr` 时"
    if (!p) { ... }           // 好
    if (p == 0) { ... }       // `==0` 是多余的；不好：不要对指针用 `0`
    if (p == nullptr) { ... } // `==nullptr` 是多余的，不建议如此

##### 强制实施

容易，仅需检查条件中多余的 `!=` 和 `==` 的使用即可。



## <a name="SS-numbers"></a>算术

### <a name="Res-mix"></a>ES.100: 不要进行有符号和无符号混合运算

##### 理由

避免错误的结果。

##### 示例

    int x = -3;
    unsigned int y = 7;
    
    cout << x - y << '\n';  // 无符号结果，可能是 4294967286
    cout << x + y << '\n';  // 无符号结果：4
    cout << x * y << '\n';  // 无符号结果，可能是 4294967275

在更实际的例子中，这种问题更难于被发现。

##### 注解

不幸的是，C++ 使用有符号整数作为数组下标，而标准库使用无符号整数作为容器下标。
这妨碍了一致性。使用 `gsl::index` 来作为下标类型；[参见 ES.107](#Res-subscripts)。

##### 强制实施

* 编译器已知这种情况，有些时候会给出警告。
* （避免噪声）有符号/无符号的混合比较，若其一个实参是 `sizeof` 或调用容器的 `.size()` 而另一个是 `ptrdiff_t`，则不要进行标记。


### <a name="Res-unsigned"></a>ES.101: 使用无符号类型进行位操作

##### 理由

无符号类型支持位操作而没有符号位的意外。

##### 示例

    unsigned char x = 0b1010'1010;
    unsigned char y = ~x;   // y == 0b0101'0101;

##### 注解

无符号类型对于模算术也很有用。
不过，如果你想要进行模算术时，
应当按需添加代码注释以注明所依赖的回绕行为，因为这样的
代码会让许多程序员感觉意外。

##### 强制实施

* 一般来说基本不可能，因为标准库也使用了无符号下标。
???

### <a name="Res-signed"></a>ES.102: 使用有符号类型进行算术运算

##### 理由

因为大多数算术都假定是有符号的；
当 `y > x` 时，`x - y` 都会产生负数，除了罕见的情况下你确实需要模算术。

##### 示例

当你不期望时，无符号算术会产生奇怪的结果。
这在混合有符号和无符号算术时有其如此。

    template<typename T, typename T2>
    T subtract(T x, T2 y)
    {
        return x - y;
    }
    
    void test()
    {
        int s = 5;
        unsigned int us = 5;
        cout << subtract(s, 7) << '\n';       // -2
        cout << subtract(us, 7u) << '\n';     // 4294967294
        cout << subtract(s, 7u) << '\n';      // -2
        cout << subtract(us, 7) << '\n';      // 4294967294
        cout << subtract(s, us + 2) << '\n';  // -2
        cout << subtract(us, s + 2) << '\n';  // 4294967294
    }

我们这次非常明确发生了什么。
但要是你见到 `us - (s + 2)` 或者 `s += 2; ...; us - s` 时，你确实能够预计到打印的结果将是 `4294967294` 吗？

##### 例外

如果你确实需要模算术的话就使用无符号类型——
根据需要为其添加代码注释以说明其依赖溢出行为，因为这样的
代码会让许多程序员感觉意外。

##### 示例

标准库使用无符号类型作为下标。
内建数组则用有符号类型作为下标。
这不可避免地带来了意外（以及 BUG）。

    int a[10];
    for (int i = 0; i < 10; ++i) a[i] = i;
    vector<int> v(10);
    // 比较有符号和无符号数；有些编译器会警告，但我们不能警告
    for (gsl::index i = 0; i < v.size(); ++i) v[i] = i;
    
    int a2[-2];         // 错误：负的大小
    
    // OK，但 int 的数值（4294967294）过大，应当会造成一个异常
    vector<int> v2(-2);

使用 `gsl::index` 作为下标类型；[参见 ES.107](#Res-subscripts)。

##### 强制实施

* 对混合有符号和无符号算术进行标记。
* 对将无符号算术的结果作为有符号数赋值或打印进行标记。
* 对负数字面量（比如 `-2`）用作容器下标进行标记。
* （避免噪声）有符号/无符号的混合比较，若其一个实参是 `sizeof` 或调用容器的 `.size()` 而另一个是 `ptrdiff_t`，则不要进行标记。


### <a name="Res-overflow"></a>ES.103: 避免上溢出

##### 理由

上溢出通常会让数值算法变得没有意义。
将值增加超过其最大值将导致内存损坏和未定义的行为。

##### 示例，不好

    int a[10];
    a[10] = 7;   // 不好
    
    int n = 0;
    while (n++ < 10)
        a[n - 1] = 9; // 不好（两次）

##### 示例，不好

    int n = numeric_limits<int>::max();
    int m = n + 1;   // 不好

##### 示例，不好

    int area(int h, int w) { return h * w; }
    
    auto a = area(10'000'000, 100'000'000);   // 不好

##### 例外

如果你确实需要模算术的话就使用无符号类型。

**替代方案**: 对于可以负担一些开销的关键应用，可以使用带有范围检查的整数和/或浮点类型。

##### 强制实施

???

### <a name="Res-underflow"></a>ES.104: 避免下溢出

##### 理由

将值减小超过其最小值将导致内存损坏和未定义的行为。

##### 示例，不好

    int a[10];
    a[-2] = 7;   // 不好
    
    int n = 101;
    while (n--)
        a[n - 1] = 9;   // 不好（两次）

##### 例外

如果你确实需要模算术的话就使用无符号类型。

##### 强制实施

???

### <a name="Res-zero"></a>ES.105: 避免除零

##### 理由

其结果是未定义的，很可能导致程序崩溃。

##### 注解

这同样适用于 `%`。

##### 示例，不好

    double divide(int a, int b) {
        // 不好, 应当进行检查（比如一条前条件）
        return a / b;
    }

##### 示例，好

    double divide(int a, int b) {
        // 好, 通过前条件进行处置（并当 C++ 支持契约后可以进行替换）
        Expects(b != 0);
        return a / b;
    }
    
    double divide(int a, int b) {
        // 好, 通过检查进行处置
        return b ? a / b : quiet_NaN<double>();
    }

**替代方案**: 对于可以负担一些开销的关键应用，可以使用带有范围检查的整数和/或浮点类型。

##### 强制实施

* 对以可能为零的整型值的除法进行标记。


### <a name="Res-nonnegative"></a>ES.106: 不要试图用 `unsigned` 来防止负数值

##### 理由

选用 `unsigned` 意味着对于包括模算术在内的整数的常规行为的许多改动，
它将抑制掉与溢出有关的警告，
并打开了与混合符号相关的错误的大门。
使用 `unsigned` 并不会真正消除负数值的可能性。

##### 示例

    unsigned int u1 = -2;   // 合法：u1 的值为 4294967294
    int i1 = -2;
    unsigned int u2 = i1;   // 合法：u2 的值为 4294967294
    int i2 = u2;            // 合法：i2 的值为 -2

真实代码中很难找出这样的（完全合法的）语法构造的问题，而它们是许多真实世界错误的来源。
考虑：

    unsigned area(unsigned height, unsigned width) { return height*width; } // [参见](#Ri-expects)
    // ...
    int height;
    cin >> height;
    auto a = area(height, 2);   // 当输入为 -2 时 a 为 4294967292

记住把 `-1` 赋值给 `unsigned int` 会变成最大的 `unsigned int`。
而且，由于无符号算术是模算术，其乘法并不会溢出，而是会发生回绕。

##### 示例

    unsigned max = 100000;    // "不小心写错了"，应该写 10'000
    unsigned short x = 100;
    while (x < max) x += 100; // 无限循环

要是 `x` 是个有符号的 `short` 的话，我们就会得到有关溢出的未定义行为的警告了。

##### 替代方案

* 使用有符号整数并检查 `x >= 0`
* 使用某个正整数类型
* 使用某个整数子值域类型
* `Assert(-1 < x)`

例如

    struct Positive {
        int val;
        Positive(int x) :val{x} { Assert(0 < x); }
        operator int() { return val; }
    };
    
    int f(Positive arg) { return arg; }
    
    int r1 = f(2);
    int r2 = f(-2);  // 抛出异常

##### 注解

???

##### 强制实施

参见 ES.100 的强制实施。


### <a name="Res-subscripts"></a>ES.107: 不要对下标使用 `unsigned`，优先使用 `gsl::index`

##### 理由

避免有符号和无符号混乱。
允许更好的优化。
允许更好的错误检测。
避免 `auto` 和 `int` 有关的陷阱。

##### 示例，不好

    vector<int> vec = /*...*/;
    
    for (int i = 0; i < vec.size(); i += 2)                    // 可能不够大
        cout << vec[i] << '\n';
    for (unsigned i = 0; i < vec.size(); i += 2)               // 有风险的回绕
        cout << vec[i] << '\n';
    for (auto i = 0; i < vec.size(); i += 2)                   // 可能不够大
        cout << vec[i] << '\n';
    for (vector<int>::size_type i = 0; i < vec.size(); i += 2) // 啰嗦
        cout << vec[i] << '\n';
    for (auto i = vec.size()-1; i >= 0; i -= 2)                // BUG
        cout << vec[i] << '\n';
    for (int i = vec.size()-1; i >= 0; i -= 2)                 // 可能不够大
        cout << vec[i] << '\n';

##### 示例，好

    vector<int> vec = /*...*/;
    
    for (gsl::index i = 0; i < vec.size(); i += 2)             // ok
        cout << vec[i] << '\n';
    for (gsl::index i = vec.size()-1; i >= 0; i -= 2)          // ok
        cout << vec[i] << '\n';

##### 注解

内建数组使用有符号的下标。
标准库容器使用无符号的下标。
因此没有完美的兼容解决方案（除非将来某一天，标准库容器改为使用有符号下标了）。
鉴于无符号和混合符号方面的已知问题，最好坚持使用（有符号）并且足够大的整数，而 `gsl::index` 保证了这点。

##### 示例

    template<typename T>
    struct My_container {
    public:
        // ...
        T& operator[](gsl::index i);    // 不是 unsigned
        // ...
    };

##### 示例

    ??? 演示改进后的代码生成和潜在可进行的错误检查 ???

##### 替代方案

可以代之以

* 使用算法
* 使用基于范围的 `for`
* 使用迭代器或指针

##### 强制实施

* 非常麻烦，因为标准库容器已经搞错了。
* （避免噪声）有符号/无符号的混合比较，若其一个实参是 `sizeof` 或调用容器的 `.size()` 而另一个是 `ptrdiff_t`，则不要进行标记。




# <a name="S-performance"></a>Per: 性能

??? 这一节应该放在主指南中吗 ???

本章节所包含的规则对于需要高性能和低延迟的人们很有价值。
就是说，这些规则是有关如何在可预测的短时段中尽可能使用更短时间和更少资源来完成任务的。
本章节中的规则要比（绝大）多数应用程序所需要的规则更多限制并更有侵入性。
请勿在一般的代码中盲目地尝试遵循这些规则：要达成低延迟的目标是需要进行一些额外工作的。

性能规则概览：

* [Per.1: 请勿进行无理由的优化](#Rper-reason)
* [Per.2: 请勿进行不成熟的优化](#Rper-Knuth)
* [Per.3: 请勿对非性能关键的代码进行优化](#Rper-critical)
* [Per.4: 不能假定复杂代码一定比简单代码更快](#Rper-simple)
* [Per.5: 不能假定低级代码一定比高级代码更快](#Rper-low)
* [Per.6: 请勿不进行测量就作出性能评断](#Rper-measure)
* [Per.7: 设计应当允许优化](#Rper-efficiency)
* [Per.10: 依赖静态类型系统](#Rper-type)
* [Per.11: 把计算从运行时转移到编译期](#Rper-Comp)
* [Per.12: 消除多余的别名](#Rper-alias)
* [Per.13: 消除多余的间接](#Rper-indirect)
* [Per.14: 最小化分配和回收的次数](#Rper-alloc)
* [Per.15: 请勿在关键逻辑分支中进行分配](#Rper-alloc0)
* [Per.16: 使用紧凑的数据结构](#Rper-compact)
* [Per.17: 在时间关键的结构中应当先声明最常用的成员](#Rper-struct)
* [Per.18: 空间即时间](#Rper-space)
* [Per.19: 进行可预测的内存访问](#Rper-access)
* [Per.30: 避免在关键路径中进行上下文切换](#Rper-context)

### <a name="Rper-reason"></a>Per.1: 请勿进行无理由的优化

##### 理由

如果没有必要优化的话，这样做的结果就是更多的错误和更高的维护成本。

##### 注解

一些人作出优化只是出于习惯或者因为感觉这很有趣。

???

### <a name="Rper-Knuth"></a>Per.2: 请勿进行不成熟的优化

##### 理由

经过精心优化的代码通常比未优化的代码更大而且更难修改。

???

### <a name="Rper-critical"></a>Per.3: 请勿对非性能关键的代码进行优化

##### 理由

对程序中并非性能关键的部分进行的优化，对于系统性能是没有效果的。

##### 注解

如果你的程序要耗费大量时间来等待 Web 或人的操作的话，对内存中的计算进行优化可能是没什么用处的。

换个角度来说：如果你的程序花费处理时间的 4% 来
计算 A 而花费 40% 的时间来计算 B，那对 A 的 50% 的改进
其影响只能和 B 的 5% 的改进相比。（如果你甚至不知道
A 或 B 到底花费了多少时间，参见 <a href="#Rper-reason">Per.1</a> 和 <a
href="#Rper-Knuth">Per.2</a>。）

### <a name="Rper-simple"></a>Per.4: 不能假定复杂代码一定比简单代码更快

##### 理由

简单的代码可能会非常快。优化器在简单代码上有时候会发生奇迹。

##### 示例，好

    // 清晰表达意图，快速执行
    
    vector<uint8_t> v(100000);
    
    for (auto& c : v)
        c = ~c;

##### 示例，不好

    // 试图更快，但通常会更慢
    
    vector<uint8_t> v(100000);
    
    for (size_t i = 0; i < v.size(); i += sizeof(uint64_t))
    {
        uint64_t& quad_word = *reinterpret_cast<uint64_t*>(&v[i]);
        quad_word = ~quad_word;
    }

##### 注解

???

???

### <a name="Rper-low"></a>Per.5: 不能假定低级代码一定比高级代码更快

##### 理由

低级代码有时候会妨碍优化。优化器在高级代码上有时候会发生奇迹。

##### 注解

???

???

### <a name="Rper-measure"></a>Per.6: 请勿不进行测量就作出性能评断

##### 理由

性能领域充斥各种错误认识和伪习俗。
现代的硬件和优化器并不遵循这些幼稚的假设；即便是专家也会经常感觉意外。

##### 注解

要进行高质量的性能测量是很难的，而且需要采用专门的工具。

##### 注解

有些使用了 Unix 的 `time` 或者标准库的 `<chrono>` 的简单的微基准测量，有助于打破大多数明显的错误认识。
如果确实无法精确地测量完整系统的话，至少也要尝试对一些关键操作和算法进行测量。
性能剖析可以帮你发现系统的哪些部分是性能关键的。
你很可能会感觉意外。

???

### <a name="Rper-efficiency"></a>Per.7: 设计应当允许优化

##### 理由

因为我们经常需要对最初的设计进行优化。
因为忽略后续改进的可能性的设计是很难修改的。

##### 示例

来自 C（以及 C++）的标准：

    void qsort (void* base, size_t num, size_t size, int (*compar)(const void*, const void*));

什么情况会需要对内存进行排序呢？
时即上，我们需要对元素序列进行排序，通常它们存储于容器之中。
对 `qsort` 的调用抛弃了许多有用的信息（比如元素的类型），强制用户对其已知的信息
进行重复（比如元素的大小），并强制用户编写额外的代码（比如用于比较 `double` 的函数）。
这蕴含了程序员工作量的增加，易错，并剥夺了编译器为优化所需的信息。

    double data[100];
    // ... 填充 a ...
    
    // 对从地址 data 开始的 100 块 sizeof(double) 大小
    // 的内存，用由 compare_doubles 所定义的顺序进行排序
    qsort(data, 100, sizeof(double), compare_doubles);

从接口设计的观点来看，`qsort` 抛弃了有用的信息。

这样做可以更好（C++98）：

    template<typename Iter>
        void sort(Iter b, Iter e);  // sort [b:e)
    
    sort(data, data + 100);

这里，我们利用了编译器关于数组大小，元素类型，以及如何对 `double` 进行比较的知识。

而以 C++11 加上[概念](#SS-concepts)的话，我还可以做得更好：

    // Sortable 指定了 c 必须是一个
    // 可以用 < 进行比较的元素的随机访问序列
    void sort(Sortable& c);
    
    sort(c);

其中的关键在于传递充分的信息以便能够选择一个好的实现。
这里给出的几个 `sort` 接口仍然有一个缺憾：
它们隐含地依赖于元素类型定义了小于（`<`）运算符。
为使接口完整，我们需要另一个接受比较准则的版本：

    // 用 p 比较 c 的元素
    void sort(Sortable& c, Predicate<Value_type<Sortable>> p);

`sort` 的标准库规范提供了这两个版本，
但其语义是以英文而不是使用概念的代码来表达的。

##### 注解

不成熟的优化被称为[一切罪恶之源](#Rper-Knuth)，但这并不是轻视性能的理由。
考虑如何使设计可以为改进而进行修正绝不是不成熟的，而性能改进则是一种常见的改进要求。
我们的目标是建立一组习惯，使缺省情况就能得到高效，可维护，且可优化的代码。
特别是，当你编写并非一次性实现细节的函数时，应当考虑

* 信息传递：
优先采用能够为后续的实现改进带来充分信息的简洁[接口](#S-interfaces)。
要注意信息会通过我们所提供的接口来流入和流出一个实现。
* 紧凑的数据：默认情况[使用紧凑的数据](#Rper-compact)，比如 `std::vector`，并[系统化地进行访问](#Rper-access)。
如果你觉得需要一种有链接的结构的话，应尝试构造接口使这个结构不会被用户所看到。
* 函数的参数传递和返回：
对可改变和不可变的数据加以区分。
不要把资源管理的负担强加给用户。
不要把假性的间接强加给用户。
对通过接口传递信息采用[符合惯例的方式](#Rf-conventional)；
不合惯例的，以及"优化过的"数据传递方式可能会严重影响后续的重新实现。
* 抽象：
不要过度泛化；视图提供每一种可能用法（和误用），并把每个设计决策都（通过编译时或运行时间接）
推迟到后面处理的设计，通常是复杂的，膨胀的，难于理解的混乱体。
应当从具体的例子进行泛化，泛化时要保持性能。
不要仅仅基于关于未来需求的推测而进行泛化。
理想情况是零开销泛化。
* 程序库：
使用带有良好接口的程序库。
当没有可用的程序库时，就构建自己的，并模仿一个好程序库的接口风格。
[标准库](#S-stdlib)是寻找模仿的一个好的第一来源。
* 隔离：
通过将你所选择的接口提供给你的代码来将其和杂乱和老旧风格的代码之间进行隔离。
这有时候称为为有用或必须但杂乱的代码"提供包装"。
不要让不良设计"渗入"你的代码中。

##### 示例

考虑：

    template <class ForwardIterator, class T>
    bool binary_search(ForwardIterator first, ForwardIterator last, const T& val);

`binary_search(begin(c), end(c), 7)` 能够得出 `7` 是否在 `c` 之中。
不过，它无法得出 `7` 在何处，或者是否有多于一个 `7`。

有时候仅把最小数量的信息传递回来（如这里的 `true` 或 `false`）是足够的，但一个好的接口会
向调用方传递其所需的信息。因此，标准库还提供了

    template <class ForwardIterator, class T>
    ForwardIterator lower_bound(ForwardIterator first, ForwardIterator last, const T& val);

`lower_bound` 返回第一个匹配元素（如果有）的迭代器，否则返回第一个大于 `val` 的元素的迭代器，找不到这样的元素时，返回 `last`。

不过 `lower_bound` 还是无法为所有用法返回足够的信息，因此标准库还提供了

    template <class ForwardIterator, class T>
    pair<ForwardIterator, ForwardIterator>
    equal_range(ForwardIterator first, ForwardIterator last, const T& val);

`equal_range` 返回迭代器的 `pair`，指定匹配的第一个和最后一个之后的元素。

    auto r = equal_range(begin(c), end(c), 7);
    for (auto p = r.first; p != r.second; ++p)
        cout << *p << '\n';

显然，这三个接口都是以相同的基本代码实现的。
它们不过是将基本的二叉搜索算法表现给用户的三种方式，
包含从最简单（"让简单的事情简单！"）
到返回完整但不总是必要的信息（"不要隐藏有用的信息"）。
自然，构造这样一组接口是需要经验和领域知识的。

##### 注解

接口的构建不要仅匹配你能想到的第一种实现和第一种用例。
一旦第一个初始实现完成后，应当进行复审；一旦它被部署出去，就很难对错误进行补救了。

##### 注解

对效率的需求并不意味着对[底层代码](#Rper-low)的需求。
高层代码并不意味着缓慢或膨胀。

##### 注解

事物都是有成本的。
不要对成本过于偏执（当代计算机真的非常快），
但需要对你所使用的东西的成本的数量级有大致的概念。
例如，应当对
一次内存访问，
一次函数调用，
一次字符串比较，
一次系统调用，
一次磁盘访问，
以及一个通过网络的消息的成本有大致的概念。

##### 注解

如果你只能想到一种实现的话，可能你并没有某种能够设计一个稳定接口的东西。
有可能它只不过是实现细节——不是每段代码都需要一个稳定接口——停下来想一想。
一个有用的问题是
"如果这个操作需要用多个线程实现的话，需要什么样的接口呢？向量化？"

##### 注解

这条规则并不抵触[不要进行不成熟的优化](#Rper-Knuth)规则。
它对其进行了补充，鼓励程序员在有必要的时候，使后续的——适当并且成熟的——优化能够进行。

##### 强制实施

很麻烦。
也许查找 `void*` 函数参数能够找到妨碍后续优化的接口的例子。

### <a name="Rper-type"></a>Per.10: 依赖静态类型系统

##### 理由

类型违规，弱类型（比如 `void*`），以及低级代码（比如把序列当作独立字节进行操作）等会让优化器的工作变得困难很多。简单的代码通常比手工打造的复杂代码能够更好地优化。

???

### <a name="Rper-Comp"></a>Per.11: 把计算从运行时转移到编译期

##### 理由

减少代码大小和运行时间。
通过使用常量来避免数据竞争。
编译时捕获错误（并因而消除错误处理代码）。

##### 示例

    double square(double d) { return d*d; }
    static double s2 = square(2);    // 旧式代码：动态初始化
    
    constexpr double ntimes(double d, int n)   // 假定 0 <= n
    {
            double m = 1;
            while (n--) m *= d;
            return m;
    }
    constexpr double s3 {ntimes(2, 3)};  // 现代代码：编译期初始化

像 `s2` 的初始化这样的代码并不少见，尤其是比 `square()` 更复杂一些的初始化更是如此。
不过，与 `s3` 的初始化相比，它有两个问题：

* 我们得忍受运行时的一次函数调用的开销
* `s2` 在初始化开始前可能就被某个别的线程访问了。

注意：常量是不可能发生数据竞争的。

##### 示例

考虑一种流行的提供包装类的技术，在包装类自身之中存储小型对象，而把大型对象保存到堆上。

    constexpr int on_stack_max = 20;
    
    template<typename T>
    struct Scoped {     // 在 Scoped 中存储一个 T
            // ...
        T obj;
    };
    
    template<typename T>
    struct On_heap {    // 在自由存储中存储一个 T
            // ...
            T* objp;
    };
    
    template<typename T>
    using Handle = typename std::conditional<(sizeof(T) <= on_stack_max),
                        Scoped<T>,      // 第一种候选
                        On_heap<T>      // 第二种候选
                   >::type;
    
    void f()
    {
        Handle<double> v1;                   // double 在栈中
        Handle<std::array<double, 200>> v2;  // array 保存到自由存储里
        // ...
    }

假定 `Scoped` 和 `On_heap` 均提供了兼容的用户接口。
这里我们在编译时计算出了最优的类型。
对于选择所要调用的最优函数，也有类似的技术。

##### 注解

理想情况是，{不}试图在编译期执行所有的代码。
显然，大多数的运算都依赖于输入，因而它们没办法挪到编译期进行，
而除了这种逻辑限制外，实际情况是，复杂的编译期运算会严重增加编译时间，
并使调试变得复杂。
甚至编译期运算也可能使得代码变慢。
这种情况确实罕见，但当把一种通用运算分解为一组优化的子运算时，可能会导致指令高速缓存的效率变差。

##### 强制实施

* 找出可以（但尚不）是 constexpr 的简单函数。
* 找出调用时其全部实参均为常量表达式的函数。
* 找出可以为 constexpr 的宏。

### <a name="Rper-alias"></a>Per.12: 消除多余的别名

???

### <a name="Rper-indirect"></a>Per.13: 消除多余的间接

???

### <a name="Rper-alloc"></a>Per.14: 最小化分配和回收的次数

???

### <a name="Rper-alloc0"></a>Per.15: 请勿在关键逻辑分支中进行分配

???

### <a name="Rper-compact"></a>Per.16: 使用紧凑的数据结构

##### 理由

性能通常都是由内存访问次数所决定的。

???

### <a name="Rper-struct"></a>Per.17: 在时间关键的结构中应当先声明最常用的成员

???

### <a name="Rper-space"></a>Per.18: 空间即时间

##### 理由

性能通常都是由内存访问次数所决定的。

???

### <a name="Rper-access"></a>Per.19: 进行可预测的内存访问

##### 理由

性能对于 Cache 的性能非常敏感，而 Cache 算法则更喜欢对相邻数据进行的（通常是线性的）简单访问行为。

##### 示例

    int matrix[rows][cols];
    
    // 不好
    for (int c = 0; c < cols; ++c)
        for (int r = 0; r < rows; ++r)
            sum += matrix[r][c];
    
    // 好
    for (int r = 0; r < rows; ++r)
        for (int c = 0; c < cols; ++c)
            sum += matrix[r][c];

### <a name="Rper-context"></a>Per.30: 避免在关键路径中进行上下文切换

???

# <a name="S-concurrency"></a>CP: 并发与并行

我们经常想要我们的计算机同时运行许多任务（或者至少表现为同时运行它们）。
这有许多不同的原因（例如，需要仅用一个处理器等待许多事件，同时处理许多数据流，或者利用大量的硬件设施）
因此也由许多不同的用以表现并发和并行的基本设施。
我们这里将说明使用 ISO 标准 C++ 中用以表现基本并发和并行的设施的原则和规则。

线程是对并发和并行编程支持的机器级别的基础。
使用线程允许互不相关地运行程序的多个部分，
同时共享相同的内存。并发编程很麻烦，
因为在线程之间保护共享的数据说起来容易做起来难。
使现存的单线程代码可以并发执行，
可以通过策略性地添加 `std::async` 或 `std::thread` 这样简单做到，
也可能需要进行完全的重写，这依赖于原始代码是否是以线程友好
的方式编写的。

本文档中的并发/并行规则的设计有三个
目标：

* 有助于编写可以被修改为能够在线程环境中使用
  的代码。
* 展示使用标准库所提供的线程原语的简洁，
  安全的方式。
* 对当并发和并行无法提供所需要的性能增益时应当如何做
  提供指导。

同样重要的一点，是要注意 C++ 的并发仍然是未完成的工作。
C++11 引入了许多核心并发原语，C++14 和 C++17 对它们进行了改进，
而且在使 C++ 编写并发程序更加简单的方面
仍有许多关注。我们预计这里的一些与程序库相关
的指导方针会随着时间有大量的改动。

这一部分需要大量的工作（显然如此）。
请注意我们的规则是从相对非专家们入手的。
真正的专家们还请稍等；
我们欢迎贡献者，
但请为正在努力使他们的并发程序正确且高效的大多数程序员着想。

并发和并行规则概览：

* [CP.1: 假定你的代码将作为多线程程序的一部分而运行](#Rconc-multi)
* [CP.2: 避免数据竞争](#Rconc-races)
* [CP.3: 最小化可写数据的明确共享](#Rconc-data)
* [CP.4: 以任务而不是线程的角度思考](#Rconc-task)
* [CP.8: 不要为同步而使用 `volatile`](#Rconc-volatile)
* [CP.9: 只要可行，就使用工具对并发代码进行验证](#Rconc-tools)

**参见**：

* [CP.con: 并发](#SScp-con)
* [CP.par: 并行](#SScp-par)
* [CP.mess: 消息传递](#SScp-mess)
* [CP.vec: 向量化](#SScp-vec)
* [CP.free: 无锁编程](#SScp-free)
* [CP.etc: 其他并发规则](#SScp-etc)

### <a name="Rconc-multi"></a>CP.1: 假定你的代码将作为多线程程序的一部分而运行

##### 理由

很难说现在或者未来什么时候会不会需要使用并发。
代码是会被重用的。
程序的其他使用了线程的部分可能会使用某个未使用线程的程序库。
请注意这条规则对于程序库代码来说最紧迫，而对独立的应用程序来说则最不紧迫。
不过，久而久之，代码片段可能出现在意想不到的地方。

##### 示例，不好

    double cached_computation(double x)
    {
        // 不好：这两个静态变量导致多线程的使用情况中的数据竞争
        static double cached_x = 0.0;
        static double cached_result = COMPUTATION_OF_ZERO;
        double result;
    
        if (cached_x == x)
            return cached_result;
        result = computation(x);
        cached_x = x;
        cached_result = result;
        return result;
    }

虽然 `cached_computation` 在单线程环境中可以正确工作，但在多线程环境中，其两个 `static` 变量将导致数据竞争进而发生未定义的行为。

有多种方法可以让这个例子在多线程环境中变得安全：

* 将并发事务委派给调用方处理。
* 将 `static` 变量标为 `thread_local`（这可能让缓存变得不那么有效）。
* 实现并发控制逻辑，例如，用一个 `static` 锁来保护这两个 `static` 变量（这可能会降低性能）。
* 让调用方提供用于缓存的内存，由此同时把内存分配和并发事务委派给了调用方。
* 拒绝在多线程环境中进行构建和/或运行。
* 提供两个实现，一个用在单线程环境中，另一个用在多线程环境中。

##### 例外

永远不会在多线程环境中执行的代码。

要小心的是：有许多例子，"认为"永远不会在多线程程序中执行的代码
却真的在多线程程序中执行了，通常是在多年以后。
一般来说，这种程序将导致进行痛苦的移除数据竞争的工作。
因此，确实有意不在多线程环境中执行的代码，应当清晰地进行标注，而且理想情况下应当利用编译或运行时的强制机制来提早检测到这种使用情况。

### <a name="Rconc-races"></a>CP.2: 避免数据竞争

##### 理由

不这样的话，则任何东西都不保证能工作，而且可能出现微妙的错误。

##### 注解

简而言之，当两个线程并发（未同步）地访问同一个对象，且至少一方为写入方（实施某个非 `const` 操作）时，就会出现数据竞争。
有关如何正确使用同步来消除数据竞争的更多信息，请求教于一本有关并发的优秀书籍。

##### 示例，不好

有大量存在数据竞争的例子，其中的一些现在这个时候就运行在
产品软件之中。一个非常简单的例子是：

    int get_id() {
      static int id = 1;
      return id++;
    }

这里的增量操作就是数据竞争的一个例子。这可能以许多方式导致发生错误，
包括：

* 线程 A 加载 `id` 的值，OS 上下文切换使 A 离开
  一段时间，其中有其他线程创建了上百个 ID。线程 A
  再次允许执行，而 `id` 被写回到那个位置，其值为 A
  所读取的 `id` 值加一。
* 线程 A 和 B 同时加载 `id` 并进行增量。它们都将获得
  相同的 ID。

局部静态变量是数据竞争的一种常见来源。

##### 示例，不好

    void f(fstream&  fs, regex pattern)
    {
        array<double, max> buf;
        int sz = read_vec(fs, buf, max);            // 从 fs 读取到 buf 中
        gsl::span<double> s {buf};
        // ...
        auto h1 = async([&]{ sort(std::execution::par, s); });     // 产生一个进行排序的任务
        // ...
        auto h2 = async([&]{ return find_all(buf, sz, pattern); });   // 产生一个查找匹配的任务
        // ...
    }

这里，在 `buf` 的元素上有一个（很讨厌的）数据竞争（`sort` 既会读取也会写入）。
所有的数据竞争都很讨厌。
我们这里设法在栈上的数据上造成了数据竞争。
不是所有数据竞争都像这个这样容易找出来的。

##### 示例，不好

    // 未用锁进行控制的代码
    
    unsigned val;
    
    if (val < 5) {
        // ... 其他线程可能在这里改动 val ...
        switch (val) {
        case 0: // ...
        case 1: // ...
        case 2: // ...
        case 3: // ...
        case 4: // ...
        }
    }

这里，若编译器不知道 `val` 可能改变，它最可能将这个 `switch` 实现为一个有五个项目的跳转表。
然后，超出 `[0..4]` 范围的 `val` 值将会导致跳转到程序中的任何可能位置的地址，从那个位置继续执行。
真的，当你遇到数据竞争时什么都可能发生。
实际上可能更为糟糕：通过查看生成的代码，是可以确定这个走偏的跳转针对给定的值将会跳转到哪里的；
这就是一种安全风险。

##### 强制实施

有些事是可能做到的，至少要做一些事。
有一些商用和开源的工具试图处理这种问题，
但要注意的是任何工具解决方案都有其成本和盲点。
静态工具通常会有许多漏报，而动态工具则通常有显著的成本。
我们希望有更好的工具。
使用多个工具可以找到比单个工具更多的问题。

还有其他方式可以缓解发生数据竞争的机会：

* 避免全局数据
* 避免 `static` 变量
* 更多地使用栈上的值类型（且不要过多地把指针到处传递）
* 更多地使用不可变数据（字面量，`constexpr`，以及 `const`）

### <a name="Rconc-data"></a>CP.3: 最小化可写数据的明确共享

##### 理由

如果不共享可写数据的话，就不会发生数据竞争。
越少进行共享，你就越少有机会忘掉对访问进行同步（而发生数据竞争）。
越少进行共享，你就越少有机会需要等待锁定（并改进性能）。

##### 示例

    bool validate(const vector<Reading>&);
    Graph<Temp_node> temperature_gradiants(const vector<Reading>&);
    Image altitude_map(const vector<Reading>&);
    // ...
    
    void process_readings(const vector<Reading>& surface_readings)
    {
        auto h1 = async([&] { if (!validate(surface_readings)) throw Invalid_data{}; });
        auto h2 = async([&] { return temperature_gradiants(surface_readings); });
        auto h3 = async([&] { return altitude_map(surface_readings); });
        // ...
        h1.get();
        auto v2 = h2.get();
        auto v3 = h3.get();
        // ...
    }

没有这些 `const` 的话，我们就必须为潜在的数据竞争而为在 `surface_readings` 上的所有异步函数调用进行复审。
使 `surface_readings` （对于这个函数）为 `const` 允许我们仅在函数体代码中进行推理。

##### 注解

不可变数据可以安全并高效地共享。
无须对其进行锁定：不可能在常量上发生数据竞争。
另请参见[CP.mess: 消息传递](#SScp-mess)和[CP.31: 优先采用按值传递](#Rconc-data-by-value)。

##### 强制实施

???


### <a name="Rconc-task"></a>CP.4: 以任务而不是线程的角度思考

##### 理由

`thread` 是一种实现概念，一种针对机器的思考方式。
任务则是一种应用概念，有时候你可以使任务和其他任务并发执行。
应用概念更容易进行推理。

##### 理由

    void some_fun() {
        std::string msg, msg2;
        std::thread publisher([&] { msg = "Hello"; });       // 不好: 表达性不足
                                                             //       且更易错
        auto pubtask = std::async([&] { msg2 = "Hello"; });  // OK
        // ...
        publisher.join();
    }

##### 注解

除了 `async()` 之外，标准库中的设施都是底层的，面向机器的，线程和锁层次的设施。
这是一种必须的基础，但我们不得不尝试提升抽象的层次：为提供生产率，可靠性，以及性能。
这对于采用更高层次的，更加面向应用的程序库（如果可能就建立在标准库设施上）的强有力的理由。

##### 强制实施

???

### <a name="Rconc-volatile"></a>CP.8: 不要为同步而使用 `volatile`

##### 理由

和其他语言不同，C++ 中的 `volatile` 并不提供原子性，不会在线程之间进行同步，
而且不会防止指令重排（无论编译器还是硬件）。
它和并发完全没有关系。

##### 示例，不好

    int free_slots = max_slots; // 当前的对象内存的来源
    
    Pool* use()
    {
        if (int n = free_slots--) return &pool[n];
    }

这里有一个问题：
这在单线程程序中是完全正确的代码，但若有两个线程执行，
并且在 `free_slots` 上发生竞争条件时，两个线程就可能拿到相同的值和 `free_slots`。
这（显然）是不好的数据竞争，受过其他语言训练的人可能会试图这样修正：

    volatile int free_slots = max_slots; // 当前的对象内存的来源
    
    Pool* use()
    {
        if (int n = free_slots--) return &pool[n];
    }

这并没有同步效果：数据竞争仍然存在！

C++ 对此的机制是 `atomic` 类型：

    atomic<int> free_slots = max_slots; // 当前的对象内存的来源
    
    Pool* use()
    {
        if (int n = free_slots--) return &pool[n];
    }

现在的 `--` 操作是原子性的，
而不是可能被另一个线程介入其独立操作之间的读-增量-写序列。

##### 替代方案

在某些其他语言中曾经使用 `volatile` 的地方使用 `atomic` 类型。
为更加复杂的例子使用 `mutex`。

##### 另见

[`volatile` 的（罕见）恰当用法](#Rconc-volatile2)

### <a name="Rconc-tools"></a>CP.9: 只要可行，就使用工具对并发代码进行验证

经验表明，让并发代码正确是特别难的，
而编译期检查、运行时检查和测试在找出并发错误方面，
并没有如同在顺序代码中找出错误时那么有效。
一些微妙的并发错误可能会造成显著的不良后果，包括内存破坏和死锁等。

##### 示例

    ???

##### 注解

线程安全性是一种有挑战性的任务，有经验的程序员通常可以做得更好一些：缓解这些风险的一种重要策略是运用工具。
现存有不少这样的工具，既有商用的也有开源的，既有研究性的也有产品级的。
遗憾的是，人们的需求和约束条件之间有巨大的差别，使我们无法给出特定的建议，
但我们可以提一些：

* 静态强制实施工具：[clang](http://clang.llvm.org/docs/ThreadSafetyAnalysis.html)
和一些老版本的 [GCC](https://gcc.gnu.org/wiki/ThreadSafetyAnnotation)
都提供了一些针对线程安全性性质的静态代码标注。
统一地采用这项技术可以将许多种类的线程安全性错误变为编译时的错误。
这些代码标注一般都是局部的（将某个特定成员变量标记为又一个特定的互斥体进行防护），
且一般都易于学习使用。但与许多的静态工具一样，它经常会造成漏报；
这些情况应当被发现但却被其忽略了。

* 动态强制实施工具：Clang 的 [Thread Sanitizer](http://clang.llvm.org/docs/ThreadSanitizer.html)（即 TSAN）
是动态工具的一个强有力的例子：它改变你的程序的构建和执行，向其中添加对内存访问的簿记工作，
精确地识别你的二进制程序的一次特定执行中发生的数据竞争。
使用它的代价在内存上（多数情况为五到十倍），也拖慢 CPU（二到二十倍）。
像这样的动态工具，在集成测试，金丝雀推送，以及在多个线程上操作的单元测试上实施是最好的。
它是与工作负载相关的：一旦 TSAN 识别了一个问题，那它就是实际发生的数据竞争，
但它只能识别出一次特定的执行之中发生的竞争。

##### 强制实施

对于特定的应用，应用的构建者来选择哪些支持工具是有价值的。

## <a name="SScp-con"></a>CP.con: 并发

这个部分所关注的是相对比较专门的通过共享数据进行多线程通信的用法。

* 有关并行算法，参见[并行](#SScp-par)。
* 有关不使用明确共享的任务间，通信参见[消息传递](#SScp-mess)。
* 有关向量并行代码，参见[向量化](#SScp-vec)。
* 有关无锁编程，参见[无锁](#SScp-free)。

并发规则概览：

* [CP.20: 使用 RAII，绝不使用普通的 `lock()`/`unlock()`](#Rconc-raii)
* [CP.21: 用 `std::lock()` 或 `std::scoped_lock` 来获得多个 `mutex`](#Rconc-lock)
* [CP.22: 绝不在持有锁的时候调用未知的代码（比如回调）](#Rconc-unknown)
* [CP.23: 把联结的 `thread` 看作是有作用域的容器](#Rconc-join)
* [CP.24: 把 `thread` 看作是全局的容器](#Rconc-detach)
* [CP.25: 优先采用 `gsl::joining_thread` 而不是 `std::thread`](#Rconc-joining_thread)
* [CP.26: 不要 `detach()` 线程](#Rconc-detached_thread)
* [CP.31: 少量数据在线程之间按值传递，而不是通过引用或指针传递](#Rconc-data-by-value)
* [CP.32: 用 `shared_ptr` 在无关的 `thread` 之间共享所有权](#Rconc-shared)
* [CP.40: 最小化上下文切换](#Rconc-switch)
* [CP.41: 最小化线程的创建和销毁](#Rconc-create)
* [CP.42: 不要无条件地 `wait`](#Rconc-wait)
* [CP.43: 最小化临界区的时间耗费](#Rconc-time)
* [CP.44: 记得为 `lock_guard` 和 `unique_lock` 命名](#Rconc-name)
* [CP.50: `mutex` 要和其所护卫的数据一起定义。一旦可能就使用 `synchronized_value<T>`](#Rconc-mutex)
* ??? 何时使用 spinlock
* ??? 何时使用 `try_lock()`
* ??? 何时应优先使用 `lock_guard` 而不是 `unique_lock`
* ??? 时序多工
* ??? 何时/如何使用 `new thread`

### <a name="Rconc-raii"></a>CP.20: 使用 RAII，绝不使用普通的 `lock()`/`unlock()`

##### 理由

避免源于未释放的锁定的令人讨厌的错误。

##### 示例，不好

    mutex mtx;
    
    void do_stuff()
    {
        mtx.lock();
        // ... 做一些事 ...
        mtx.unlock();
    }

或早或晚都会有人忘记 `mtx.unlock()`，在 `... 做一些事 ...` 中放入一个 `return`，抛出异常，或者别的什么。

    mutex mtx;
    
    void do_stuff()
    {
        unique_lock<mutex> lck {mtx};
        // ... 做一些事 ...
    }

##### 强制实施

标记对成员 `lock()` 和 `unlock()` 的调用。 ???


### <a name="Rconc-lock"></a>CP.21: 用 `std::lock()` 或 `std::scoped_lock` 来获得多个 `mutex`

##### 理由

避免在多个 `mutex` 上造成死锁。

##### 示例

下面将导致死锁：

    // 线程 1
    lock_guard<mutex> lck1(m1);
    lock_guard<mutex> lck2(m2);
    
    // 线程 2
    lock_guard<mutex> lck2(m2);
    lock_guard<mutex> lck1(m1);

代之以使用 `lock()`：

    // 线程 1
    lock(m1, m2);
    lock_guard<mutex> lck1(m1, defer_lock);
    lock_guard<mutex> lck2(m2, defer_lock);
    
    // 线程 2
    lock(m2, m1);
    lock_guard<mutex> lck2(m2, defer_lock);
    lock_guard<mutex> lck1(m1, defer_lock);

或者（这样更佳，但仅为 C++17）：

    // 线程 1
    scoped_lock<mutex, mutex> lck1(m1, m2);
    
    // 线程 2
    scoped_lock<mutex, mutex> lck2(m2, m1);

这样，`thread1` 和 `thread2` 的作者们仍然未在 `mutex` 的顺序上达成一致，但顺序不再是问题了。

##### 注解

在实际代码中，`mutex` 的命名很少便于程序员记得某种有意的关系和有意的获取顺序。
在实际代码中，`mutex` 并不总是便于在连续代码行中依次获取的。

在 C++17 中，可以编写普通的

    lock_guard lck1(m1, adopt_lock);

而让 `mutex` 类型被推断出来。

##### 强制实施

检测多个 `mutex` 的获取。
这一般来说是无法确定的，但找出常见的简单例子（比如上面这个）则比较容易。


### <a name="Rconc-unknown"></a>CP.22: 绝不在持有锁的时候调用未知的代码（比如回调）

##### 理由

如果不了解代码做了什么，就有死锁的风险。

##### 示例

    void do_this(Foo* p)
    {
        lock_guard<mutex> lck {my_mutex};
        // ... 做一些事 ...
        p->act(my_data);
        // ...
    }

如果不知道 `Foo::act` 会干什么（可能它是一个虚函数，调用某个还未编写的某个派生类成员），
它可能会（递归地）调用 `do_this` 因而在 `my_mutex` 上造成死锁。
可能它会在某个别的 `mutex` 上锁定而无法在适当的时间内返回，对任何调用了 `do_this` 的代码造成延迟。

##### 示例

"调用未知代码"问题的一个常见例子是调用了试图在相同对象上进行锁定访问的函数。
这种问题通常可以用 `recursive_mutex` 来解决。例如：

    recursive_mutex my_mutex;
    
    template<typename Action>
    void do_something(Action f)
    {
        unique_lock<recursive_mutex> lck {my_mutex};
        // ... 做一些事 ...
        f(this);    // f 将会对 *this 做一些事
        // ...
    }

如果如同其很可能做的那样，`f()` 调用了 `*this` 的某个操作的话，我们就必须保证在调用之前对象的不变式是满足的。

##### 强制实施

* 当持有非递归的 `mutex` 时调用虚函数则进行标记。
* 当持有非递归的 `mutex` 时调用回调则进行标记。


### <a name="Rconc-join"></a>CP.23: 把联结的 `thread` 看作是有作用域的容器

##### 理由

为了维护指针安全性并避免泄漏，需要考虑 `thread` 所使用的指针。
如果 `thread` 联结了，我们可以安全地把指向这个 `thread` 所在作用域及其外围作用域中的对象的指针传递给它。

##### 示例

    void f(int* p)
    {
        // ...
        *p = 99;
        // ...
    }
    int glob = 33;
    
    void some_fct(int* p)
    {
        int x = 77;
        joining_thread t0(f, &x);           // OK
        joining_thread t1(f, p);            // OK
        joining_thread t2(f, &glob);        // OK
        auto q = make_unique<int>(99);
        joining_thread t3(f, q.get());      // OK
        // ...
    }

`gsl::joining_thread` 是一种 `std::thread`，其析构函数进行联结且不可被 `detached()`。
这里的"OK"表明对象能够在 `thread` 可以使用指向它的指针时一直处于作用域（"存活"）。
`thread` 运行的并发性并不会影响这里的生存期或所有权问题；
这些 `thread` 可以仅仅被看成是从 `some_fct` 中调用的函数对象。

##### 强制实施

确保 `joining_thread` 不会 `detach()`。
之后，可以实施（针对局部对象的）常规的生存期和所有权强制实施方案。

### <a name="Rconc-detach"></a>CP.24: 把 `thread` 看作是全局的容器

##### 理由

为了维护指针安全性并避免泄漏，需要考虑 `thread` 所使用的指针。
如果 `thread` 脱离了，我们只可以安全地把指向静态和自由存储的对象的指针传递给它。

##### 示例

    void f(int* p)
    {
        // ...
        *p = 99;
        // ...
    }
    
    int glob = 33;
    
    void some_fct(int* p)
    {
        int x = 77;
        std::thread t0(f, &x);           // 不好
        std::thread t1(f, p);            // 不好
        std::thread t2(f, &glob);        // OK
        auto q = make_unique<int>(99);
        std::thread t3(f, q.get());      // 不好
        // ...
        t0.detach();
        t1.detach();
        t2.detach();
        t3.detach();
        // ...
    }

这里的"OK"表明对象能够在 `thread` 可以使用指向它的指针时一直处于作用域（"存活"）。
"bad"则表示 `thread` 可能在对象销毁之后使用指向它的指针。
`thread` 运行的并发性并不会影响这里的生存期或所有权问题；
这些 `thread` 可以仅仅被看成是从 `some_fct` 中调用的函数对象。

##### 注解

即便具有静态存储期的对象，在脱离的线程中的使用也会造成问题：
若是这个线程持续到程序终止，则它的运行可能与具有静态存储期的对象的销毁过程之间并发地发生，
而这样对这些对象的访问就可能发生竞争。

##### 注解

如果你[不 `detach()`](#Rconc-detached_thread) 并[使用 `gsl::joining_tread`](#Rconc-joining_thread) 的话，本条规则是多余的。
不过，将代码转化为遵循这些指导方针可能很困难，而对于第三方库来说更是不可能的。
这些情况下，这条规则对于生存期安全性和类型安全性来说就是必要的了。


一般来说是无法确定是否对某个 `thread` 执行了 `detach()` 的，但简单的常见情况则易于检测出来。
如果无法证明某个 `thread` 并没有 `detach()` 的话，我们只能假定它确实脱离了，且它的存活将超过其构造时所处于的作用域；
之后，可以实施（针对全局对象的）常规的生存期和所有权强制实施方案。

##### 强制实施

当试图将局部变量传递给可能 `detach()` 的线程时进行标记。

### <a name="Rconc-joining_thread"></a>CP.25: 优先采用 `gsl::joining_thread` 而不是 `std::thread`

##### 理由

`joining_thread` 是一种在其作用域结尾处进行联结的线程。
脱离的线程很难进行监管。
确保脱离的线程（和潜在脱离的线程）中没有错误则更加困哪。

##### 示例，不好

    void f() { std::cout << "Hello "; }
    
    struct F {
        void operator()() { std::cout << "parallel world "; }
    };
    
    int main()
    {
        std::thread t1{f};      // f() 在独立线程中执行
        std::thread t2{F()};    // F()() 在独立线程中执行
    }  // 请找出问题

##### 示例

    void f() { std::cout << "Hello "; }
    
    struct F {
        void operator()() { std::cout << "parallel world "; }
    };
    
    int main()
    {
        std::thread t1{f};      // f() 在独立线程中执行
        std::thread t2{F()};    // F()() 在独立线程中执行
    
        t1.join();
        t2.join();
    }  // 剩下一个糟糕的 BUG


##### 示例，不好

决定是要 `join()` 还是 `detach()` 的代码可能很复杂，甚至是可能由线程中所调用的函数所决定，也可能由创建线程的函数所调用的函数来决定：

    void tricky(thread* t, int n)
    {
        // ...
        if (is_odd(n))
            t->detach();
        // ...
    }
    
    void use(int n)
    {
        thread t { tricky, this, n };
        // ...
        // ... 这里应不应该联结？ ...
    }

这极大地使生存期分析复杂化了，而且在并不非常罕见的情况下甚至使得生存期分析变得不可能。
这意味着，我们无法在线程中安全地涉指 `use()` 的局部对象，或者从 `use()` 中安全地涉指线程中的局部对象。

##### 注解

使"不死线程"成为全局的，将其放入外围作用域中，或者放入自由存储中，而不要 `detach()` 它们。
[不要 `detach`](#Rconc-detached_thread)。

##### 注解

因为老代码和第三方库也会使用 `std::thread`，本条规则可能很难引入。

##### 理由

标记 `std::thread` 的使用：

* 建议使用 `gsl::joining_thread`.
* 建议当其脱离时使其["外放所有权"](#Rconc-detached_thread)到某个外围作用域中。
* 如果不明确线程是联结还是脱离，则严正警告。

### <a name="Rconc-detached_thread"></a>CP.26: 不要 `detach()` 线程

##### 理由

通常，需要存活超出线程创建的作用域的情况是来源于 `thread` 的任务所决定的，
但用 `detach` 来实现这点将造成更加难于对脱离的线程进行监控和通信。
特别是，要确保线程按预期完成或者按预期的时间存活变得更加困难（虽然不是不可能）。

##### 示例

    void heartbeat();
    
    void use()
    {
        std::thread t(heartbeat);             // 不联结；打算持续运行 heartbeat
        t.detach();
        // ...
    }

这是一种合理的线程用法，一般会使用 `detach()`。
但这里有些问题。
我们怎么监控脱离的线程以查看它是否存活呢？
心跳里边可能会出错，而在需要心跳的系统中，心跳丢失可能是非常严重的问题。
因此，我们需要与心跳线程进行通信
（例如，通过某个消息流，或者使用 `condition_variable` 的通知事件）。

一种替代的，而且通常更好的方案是，通过将其放入某个其创建点（或激活点）之外的作用域来控制其生存期。
例如：

    void heartbeat();
    
    gsl::joining_thread t(heartbeat);             // 打算持续运行 heartbeat

这个心跳，（除非出错或者硬件故障等情况）将在程序运行时一直运行。

有时候，我们需要将创建点和所有权点分离开：

    void heartbeat();
    
    unique_ptr<gsl::joining_thread> tick_tock {nullptr};
    
    void use()
    {
        // 打算在 tick_tock 存活期间持续运行 heartbeat
        tick_tock = make_unique(gsl::joining_thread, heartbeat);
        // ...
    }

##### 强制实施

标记 `detach()`。


### <a name="Rconc-data-by-value"></a>CP.31: 少量数据在线程之间按值传递，而不是通过引用或指针传递

##### 理由

少量数据的复制，其复制和访问要比使用某种锁定机制进行共享更廉价。
复制天然会带来唯一所有权（简化代码），并消除数据竞争的可能性。

##### 注解

对"少量"进行精确的定义是不可能的。

##### 示例

    string modify1(string);
    void modify2(string&);
    
    void fct(string& s)
    {
        auto res = async(modify1, s);
        async(modify2, s);
    }

`modify1` 的调用涉及两个 `string` 值的复制；而 `modify2` 的调用则不会。
另一方面，`modify1` 的实现和我们为单线程代码所编写的完全一样，
而 `modify2` 的实现则需要某种形式的锁定以避免数据竞争。
如果字符串很短（比如 10 个字符），对 `modify1` 的调用将会出乎意料地快；
基本上所有的代价都在 `thread` 的切换上。如果字符串很长（比如 1,000,000 个字符），对其两次复制
可能并不是一个好主意。

注意这个论点和 `async` 并没有任何关系。它同等地适用于任何对采用消息传递
还是共享内存的考虑之上。

##### 强制实施

???


### <a name="Rconc-shared"></a>CP.32: 用 `shared_ptr` 在无关的 `thread` 之间共享所有权

##### 理由

如果线程之间是无关的（就是说，互相不知道是否在相同作用域中，或者一个的生存期在另一个之内），
而它们需要共享需要删除的自由存储内存，`shared_ptr`（或者等价物）就是唯一的
可以保证正确删除的安全方式。

##### 示例

    ???

##### 注解

* 可以共享静态对象（比如全局对象），因为它并不像是需要某个线程来负责其删除那样被谁所拥有。
* 自由存储上不会被删除的对象可以进行共享。
* 由一个线程所拥有的对象可以安全地共享给另一个线程，只要第二个线程存活不会超过这个拥有者线程即可。

##### 强制实施

???


### <a name="Rconc-switch"></a>CP.40: 最小化上下文切换

##### 理由

上下文切换是昂贵的。

##### 示例

    ???

##### 强制实施

???


### <a name="Rconc-create"></a>CP.41: 最小化线程的创建和销毁

##### 理由

线程创建是昂贵的。

##### 示例

    void worker(Message m)
    {
        // 处理
    }
    
    void master(istream& is)
    {
        for (Message m; is >> m; )
            run_list.push_back(new thread(worker, m));
    }

这会为每个消息产生一个线程，而 `run_list` 则假定在它们完成后对这些任务进行销毁。

我们可以用一组预先创建的工作线程来处理这些消息：

    Sync_queue<Message> work;
    
    void master(istream& is)
    {
        for (Message m; is >> m; )
            work.put(m);
    }
    
    void worker()
    {
        for (Message m; m = work.get(); ) {
            // 处理
        }
    }
    
    void workers()  // 设立工作线程（这里是 4 个工作线程）
    {
        joining_thread w1 {worker};
        joining_thread w2 {worker};
        joining_thread w3 {worker};
        joining_thread w4 {worker};
    }

##### 注解

如果你的系统有一个好的线程池的话，就请使用它。
如果你的系统有一个好的消息队列的话，就请使用它。

##### 强制实施

???


### <a name="Rconc-wait"></a>CP.42: 不要无条件地 `wait`

##### 理由

没有条件的 `wait` 可能会丢失唤醒，或者唤醒时只会发现无事可做。

##### 示例，不好

    std::condition_variable cv;
    std::mutex mx;
    
    void thread1()
    {
        while (true) {
            // 做一些工作 ...
            std::unique_lock<std::mutex> lock(mx);
            cv.notify_one();    // 唤醒另一个线程
        }
    }
    
    void thread2()
    {
        while (true) {
            std::unique_lock<std::mutex> lock(mx);
            cv.wait(lock);    // 可能会永远阻塞
            // 做一些工作 ...
        }
    }

这里，如果某个其他 `thread` 消费了 `thread1` 的通知的话，`thread2` 将会永远等待下去。

##### 示例

    template<typename T>
    class Sync_queue {
    public:
        void put(const T& val);
        void put(T&& val);
        void get(T& val);
    private:
        mutex mtx;
        condition_variable cond;    // 这用于控制访问
        list<T> q;
    };
    
    template<typename T>
    void Sync_queue<T>::put(const T& val)
    {
        lock_guard<mutex> lck(mtx);
        q.push_back(val);
        cond.notify_one();
    }
    
    template<typename T>
    void Sync_queue<T>::get(T& val)
    {
        unique_lock<mutex> lck(mtx);
        cond.wait(lck, [this]{ return !q.empty(); });    // 防止假性唤醒
        val = q.front();
        q.pop_front();
    }

这样当执行 `get()` 的线程被唤醒时，如果队列为空（比如因为别的线程在之前已经 `get()` 过了），
它将立刻回到睡眠中继续等待。

##### 强制实施

对所有没有条件的 `wait` 进行标记。


### <a name="Rconc-time"></a>CP.43: 最小化临界区的时间耗费

##### 理由

获取 `mutex` 时耗费的时间越短，其他 `thread` 不得不等待的机会就会越少，
而 `thread` 的挂起和恢复是昂贵的。

##### 示例

    void do_something() // 不好
    {
        unique_lock<mutex> lck(my_lock);
        do0();  // 预备：不需要锁定
        do1();  // 事务：需要锁定
        do2();  // 清理：不需要锁定
    }

这里我们持有的锁定比所需的要长：
我们不应该在必须锁定之前就获取锁定，而且应当在开始清理之前将其释放掉。
可以将其重写为：

    void do_something() // 不好
    {
        do0();  // 预备：不需要锁定
        my_lock.lock();
        do1();  // 事务：需要锁定
        my_lock.unlock();
        do2();  // 清理：不需要锁定
    }

但这样损害了安全性并且违反了[使用 RAII](#Rconc-raii) 规则。
我们可以为临界区添加语句块：

    void do_something() // OK
    {
        do0();  // 预备：不需要锁定
        {
            unique_lock<mutex> lck(my_lock);
            do1();  // 事务：需要锁定
        }
        do2();  // 清理：不需要锁定
    }

##### 强制实施

一般来说是不可能的。
对"裸" `lock()` 和 `unlock()` 进行标记。


### <a name="Rconc-name"></a>CP.44: 记得为 `lock_guard` 和 `unique_lock` 命名

##### 理由

无名的局部对象时临时对象，会立刻离开作用域。

##### 示例

    unique_lock<mutex>(m1);
    lock_guard<mutex> {m2};
    lock(m1, m2);

这个貌似足够有效，但其实并非如此。

##### 强制实施

标记所有的无名 `lock_guard` 和 `unique_lock`。



### <a name="Rconc-mutex"></a>CP.50: `mutex` 要和其所保护的数据一起定义，只要可能就使用 `synchronized_value<T>`

##### 理由

对于读者来说，数据应该且如何被保护应当是显而易见的。这可以减少锁定错误的互斥体，或者互斥体未能被锁定的机会。

使用 `synchronized_value<T>` 保证了数据都带有互斥体，并且当访问数据时锁定正确的互斥体。
参见向某个未来的 TS 或 C++ 标准的修订版添加 `synchronized_value` [WG21 提案](http://wg21.link/p0290)。

##### 示例

    struct Record {
        std::mutex m;   // 访问其他成员之前应当获取这个 mutex
        // ...
    };
    
    class MyClass {
        struct DataRecord {
           // ...
        };
        synchronized_value<DataRecord> data; // 用互斥体保护数据
    };

##### 强制实施

??? 可能吗？


## <a name="SScp-par"></a>CP.par: 并行

这里的"并行"代表的是对许多数据项（或多或少）同时地（"并行进行"）实施某项任务。

并行规则概览：

* ???
* ???
* 适当的时候，优先采用标准库的并行算法
* 使用为并行设计的算法，而不是不必要地依赖于线性求值的算法



## <a name="SScp-mess"></a>CP.mess: 消息传递

标准库的设施是相当底层的，关注于使用 `thread`，`mutex`，`atomic` 等类型的贴近硬件的关键编程。
大多数人都不应该在这种层次上工作：它是易错的，而且开发很慢。
如果可能的话，应当使用高层次的设施：消息程序库，并行算法，以及向量化。
这一部分关注如何传递消息，以使程序员不必进行显式的同步。

消息传递规则概览：

* [CP.60: 使用 `future` 从并发任务返回值](#Rconc-future)
* [CP.61: 使用 `async()` 来产生并发任务](#Rconc-async)
* 消息队列
* 消息程序库

???? 是否应该有某个"使用 X 而不是 `std::async`"，其中 X 是某种更好说明的线程池？

??? 在未来的趋势下（甚至是现存的程序库），`std::async` 是否还是值得使用的并行设施？当有人想要对比如 `std::accumulate`（带上额外的累加前条件），或者归并排序进行并行化时，指导方针应该给出什么样的建议呢？


### <a name="Rconc-future"></a>CP.60: 使用 `future` 从并发任务返回值

##### 理由

`future` 为异步任务保持了常规函数调用的返回语义。
它没有显式的锁定，而且正确的（值）返回和错误的（异常）返回都能被简单处理。

##### 示例

    ???

##### 注解

???

##### 强制实施

???

### <a name="Rconc-async"></a>CP.61: 使用 `async()` 来产生并发任务

##### 理由

`future` 为异步任务保持了常规函数调用的返回语义。
它没有显式的锁定，而且正确的（值）返回和错误的（异常）返回都能被简单处理。

##### 示例

    ???

##### 注解

不幸的是，`async()` 并不完善。
比如说，它并不保证用线程池来最小化线程的创建。
实际上，大多数当前的 `async()` 实现都不这样做。
不过，`async()` 很简单而且逻辑正确，因此在某种更好的东西到来之前，
而且除非你确实需要为大量异步任务进行优化，都可以容忍 `async()`。

##### 强制实施

???


## <a name="SScp-vec"></a>CP.vec: 向量化

向量化是一种在不引入显式同步时并发地执行多个任务的技术。
它会并行地把某个操作实施与某个数据结构（向量，数组，等等）的各个元素之上。
向量化引人关注的性质在于它通常不需要对程序进行非局部的改动。
不过，向量化只对简单的数据结构和特别针对它所构造的算法能够得到最好的工作效果。

向量化规则概览：

* ???
* ???

## <a name="SScp-free"></a>CP.free: 无锁编程

使用 `mutex` 和 `condition_variable` 进行同步相对来说很昂贵。
而且可能导致死锁。
为了性能并消除死锁的可能性，有时候我们不得不使用麻烦的底层"无锁"设施，
它们依赖于对内存短暂地获得互斥（"原子性"）访问。
无锁编程也被用于实现如 `thread` 和 `mutex` 这样的高层并发机制。

无锁编程规则概览：

* [CP.100: 除非绝对必要，请勿使用无锁编程](#Rconc-lockfree)
* [CP.101: 不要信任你的硬件-编译器组合](#Rconc-distrust)
* [CP.102: 仔细研究文献](#Rconc-literature)
* 如何/何时使用原子
* 避免饥饿
* 使用无锁数据结构而不是手工构造的专门的无锁访问
* [CP.110: 不要为初始化编写你自己的双检查锁定](#Rconc-double)
* [CP.111: 当确实需要双检查锁定时应当采用惯用的模式](#Rconc-double-pattern)
* 如何/何时进行比较并交换（CAS）


### <a name="Rconc-lockfree"></a>CP.100: 除非绝对必要，请勿使用无锁编程

##### 理由

无锁编程容易出错，要求专家级的语言特性、机器架构和数据结构知识。

##### 示例，不好

    extern atomic<Link*> head;        // 共享的链表表头
    
    Link* nh = new Link(data, nullptr);    // 为进行插入制作一个连接
    Link* h = head.load();                 // 读取链表中的共享表头
    
    do {
        if (h->data <= data) break;        // 这样的话，就插入到别处
        nh->next = h;                      // 下一个元素是之前的表头
    } while (!head.compare_exchange_weak(h, nh));    // 将 nh 写入 head 或 h

请找出这里的 BUG。
通过测试找到它是非常困难的。
请阅读有关 ABA 问题的材料。

##### 例外

[原子变量](#???)可以简单并安全地使用，只要你所用的是顺序一致性内存模型（`memory_order_seq_cst`），而这是默认情况。

##### 注解

高层的并发机制，诸如 `thread` 和 `mutex`，是使用无锁编程来实现的。

**替代方案**: 使用由他人实现并作为程序库一部分的无锁数据结构。


### <a name="Rconc-distrust"></a>CP.101: 不要信任你的硬件-编译器组合

##### 理由

无锁编程所使用的底层硬件接口，是属于最难正确实现的，
而且属于最可能会发生最微妙的兼容性问题的领域。
如果你为了性能而进行无锁编程的话，你应当进行回归检查。

##### 注解

指令重排（静态和动态的）会让我们很难有效在这个层次上进行思考（尤其当你使用宽松的内存模型的时候）。
经验，（半）形式化的模型以及模型检查可以提供帮助。
测试——通常需要极端程度——是基础。
"不要飞得太靠近太阳。"

##### 强制实施

准备强有力的规则，使得当硬件，操作系统，编译器，和程序库发生任何改变都能重复测试以进行覆盖。


### <a name="Rconc-literature"></a>CP.102: 仔细研究文献

##### 理由

除了原子和少数标准使用模式之外，无锁编程真的是只有专家才懂的议题。
在发布无锁代码给其他人使用之前，应当先成为一名专家。

##### 参考文献

* Anthony Williams: C++ concurrency in action. Manning Publications.
* Boehm, Adve, You Don't Know Jack About Shared Variables or Memory Models , Communications of the ACM, Feb 2012.
* Boehm, "Threads Basics", HPL TR 2009-259.
* Adve, Boehm, "Memory Models: A Case for Rethinking Parallel Languages and Hardware", Communications of the ACM, August 2010.
* Boehm, Adve, "Foundations of the C++ Concurrency Memory Model", PLDI 08.
* Mark Batty, Scott Owens, Susmit Sarkar, Peter Sewell, and Tjark Weber, "Mathematizing C++ Concurrency", POPL 2011.
* Damian Dechev, Peter Pirkelbauer, and Bjarne Stroustrup: Understanding and Effectively Preventing the ABA Problem in Descriptor-based Lock-free Designs. 13th IEEE Computer Society ISORC 2010 Symposium. May 2010.
* Damian Dechev and Bjarne Stroustrup: Scalable Non-blocking Concurrent Objects for Mission Critical Code. ACM OOPSLA'09. October 2009
* Damian Dechev, Peter Pirkelbauer, Nicolas Rouquette, and Bjarne Stroustrup: Semantically Enhanced Containers for Concurrent Real-Time Systems. Proc. 16th Annual IEEE International Conference and Workshop on the Engineering of Computer Based Systems (IEEE ECBS). April 2009.


### <a name="Rconc-double"></a>CP.110: 不要为初始化编写你自己的双检查锁定

##### 理由

从 C++11 开始，静态局部变量是以线程安全的方式初始化的。当和 RAII 模式结合时，静态局部变量可以取代为初始化自己编写双检查锁定的需求。`std::call_once` 也可以达成相同的目的。请使用 C++11 的静态局部变量或者 `std::call_once` 来代替自己为初始化编写的双检查锁定。

##### 示例

使用 `std::call_once` 的例子。

    void f()
    {
        static std::once_flag my_once_flag;
        std::call_once(my_once_flag, []()
        {
            // 这个只做一次
        });
        // ...
    }

使用 C++11 的线程安全静态局部变量的例子。

    void f()
    {
        // 假定编译器遵循 C++11
        static My_class my_object; // 构造函数仅调用一次
        // ...
    }
    
    class My_class
    {
    public:
        My_class()
        {
            // 这个只做一次
        }
    };

##### 强制实施

??? 是否可能检测出这种惯用法？


### <a name="Rconc-double-pattern"></a>CP.111: 当确实需要双检查锁定时应当采用惯用的模式

##### 理由

双检查锁定是很容易被搞乱的。如果确实需要编写自己的双检查锁定，而不顾规则 [CP.110: 不要为初始化编写你自己的双检查锁定](#Rconc-double)和规则 [CP.100: 除非绝对必要，请勿使用无锁编程](#Rconc-lockfree)，那么应当采用惯用的模式。

使用双检查锁定模式而不违反[CP.110: 不要为初始化编写你自己的双检查锁定](#Rconc-double)的情形，出现于当某个非线程安全的动作既困难也罕见，并且存在某个快速且线程安全的测试可以用于保证该动作并不需要实施的情况，但反过来的情况则无法保证。

##### 示例，不好

使用 `volatile` 并不能使得第一个检查线程安全，另见[CP.200: `volatile` 仅用于和非 C++ 内存进行通信](#Rconc-volatile2)

    mutex action_mutex;
    volatile bool action_needed;
    
    if (action_needed) {
        std::lock_guard<std::mutex> lock(action_mutex);
        if (action_needed) {
            take_action();
            action_needed = false;
        }
    }

##### 示例，好

    mutex action_mutex;
    atomic<bool> action_needed;
    
    if (action_needed) {
        std::lock_guard<std::mutex> lock(action_mutex);
        if (action_needed) {
            take_action();
            action_needed = false;
        }
    }

这对于正确调校的内存顺序会带来好处，其中的获取加载要比顺序一致性加载更加高效

    mutex action_mutex;
    atomic<bool> action_needed;
    
    if (action_needed.load(memory_order_acquire)) {
        lock_guard<std::mutex> lock(action_mutex);
        if (action_needed.load(memory_order_relaxed)) {
            take_action();
            action_needed.store(false, memory_order_release);
        }
    }

##### 强制实施

??? 是否可能检测出这种惯用法？


## <a name="SScp-etc"></a>CP.etc: 其他并发规则

这些规则不适于简单的分类：

* [CP.200: `volatile` 仅用于和非 C++ 内存进行通信](#Rconc-volatile2)
* [CP.201: ??? 信号](#Rconc-signal)

### <a name="Rconc-volatile2"></a>CP.200: `volatile` 仅用于和非 C++ 内存进行通信

##### 理由

`volatile` 用于涉指那些与"非 C++"代码之间共享的对象，或者不遵循 C++ 内存模型的硬件。

##### 示例

    const volatile long clock;

这说明了一个被某个时钟电路不断更新的寄存器。
`clock` 为 `volatile` 是因为其值将会在没有使用它的 C++ 程序的任何动作下被改变。
例如，两次读取 `clock` 经常会产生两个不同的值，因此优化器最好不要将下面代码中的第二个读取操作优化掉：

    long t1 = clock;
    // ... 这里没有对 clock 的使用 ...
    long t2 = clock;

`clock` 为 `const` 是因为程序不应当试图写入 `clock`。

##### 注解

除非你是在编写直接操作硬件的最底层代码，否则应当把 `volatile` 当做是某种深奥的功能特性并最好避免使用。

##### 示例

通常 C++ 代码接受的 `volatile` 内存是由别处所拥有的（硬件或其他语言）：

    int volatile* vi = get_hardware_memory_location();
        // 注意：我们获得了指向别人的内存的指针
        // volatile 说明"请特别尊重地对待"

有时候 C++ 代码会分配 `volatile` 内存，并通过故意地暴露一个指针来将其共享给"别人"（硬件或其他语言）：

    static volatile long vl;
    please_use_this(&vl);   // 暴露对这个的一个引用给"别人"（不是 C++）

##### 示例，不好

`volatile` 局部变量几乎都是错误的——既然是短暂的，它们如何才能共享给其他语言或硬件呢？
因为相同的理由，这几乎同样强有力地适用于成员变量。

    void f() {
        volatile int i = 0; // 不好，volatile 局部变量
        // etc.
    }
    
    class My_type {
        volatile int i = 0; // 可以的，volatile 成员变量
        // etc.
    };

##### 注解

于其他一些语言不通，C++ 中的 `volatile` [和同步没有任何关系](#Rconc-volatile)。

##### 强制实施

* 对 `volatile T` 的局部成员变量进行标记；几乎肯定你应当用 `atomic<T>` 进行代替。
* ???

### <a name="Rconc-signal"></a>CP.201: ??? 信号

???UNIX 信号处理??? 也许值得提及异步信号安全有多么微弱，以及如何同信号处理器进行通信（也许最好应当"完全避免"）


# <a name="S-errors"></a>E: 错误处理

错误处理涉及：

* 检测某个错误
* 将有关错误的信息传递给某个处理代码
* 维持程序的某个有效状态
* 避免资源泄漏

不可能做到从所有的错误中恢复。如果从某个错误进行恢复是不可能的话，以明确定义的方式迅速"脱离"则是很重要的。错误处理的策略必须简单，否则就会成为更糟糕错误的来源。未经测试和很少被执行的错误处理代码自身也是许多 BUG 的来源。

以下规则是设计用以帮助避免几种错误的：

* 类型违规（比如对 `union` 和强制转换的误用）
* 资源泄漏（包括内存泄漏）
* 边界错误
* 生存期错误（比如在对象以被 `delete` 后进行访问）
* 复杂性错误（可能由于过于复杂的想法表达而导致的逻辑错误）
* 接口错误（比如通过接口传递了预期外的值）

错误处理规则概览：

* [E.1: 在设计中尽早开发错误处理策略](#Re-design)
* [E.2: 通过抛出异常来明示函数无法完成其所赋予的任务](#Re-throw)
* [E.3: 仅使用异常来进行错误处理](#Re-errors)
* [E.4: 围绕不变式来设计错误处理策略](#Re-design-invariants)
* [E.5: 让构造函数建立不变式，若其无法做到则抛出异常](#Re-invariant)
* [E.6: 使用 RAII 来避免泄漏](#Re-raii)
* [E.7: 明示前条件](#Re-precondition)
* [E.8: 明示后条件](#Re-postcondition)

* [E.12: 当函数不可能或不能接受以 `throw` 来退出时，使用 `noexcept`](#Re-noexcept)
* [E.13: 不要在作为某个对象的直接所有者时抛出异常](#Re-never-throw)
* [E.14: 应当使用为目的所设计的自定义类型（而不是内建类型）作为异常](#Re-exception-types)
* [E.15: 按引用捕获类型层次中的异常](#Re-exception-ref)
* [E.16: 析构函数，回收函数，以及 `swap` 决不能失败](#Re-never-fail)
* [E.17: 不要试图在每个函数中捕获每个异常](#Re-not-always)
* [E.18: 最小化对 `try`/`catch` 的显式使用](#Re-catch)
* [E.19: 当没有合适的资源包装时，使用 `final_action` 对象来表达清理动作](#Re-finally)

* [E.25: 当不能抛出异常时，模拟 RAII 来进行资源管理](#Re-no-throw-raii)
* [E.26: 当不能抛出异常时，考虑采取快速失败](#Re-no-throw-crash)
* [E.27: 当不能抛出异常时，系统化地使用错误代码](#Re-no-throw-codes)
* [E.28: 避免基于全局状态（比如 `errno`）的错误处理](#Re-no-throw)

* [E.30: 请勿使用异常说明](#Re-specifications)
* [E.31: 恰当地对 `catch` 子句排序](#Re_catch)

### <a name="Re-design"></a>E.1: 在设计中尽早开发错误处理策略

##### 理由

在一个系统中改造翻新一种一致且完整的处理错误和资源泄漏的策略是很难的。

### <a name="Re-throw"></a>E.2: 通过抛出异常来明示函数无法完成其所赋予的任务

##### 理由

让错误处理有系统性，强健，而且避免重复。

##### 示例

    struct Foo {
        vector<Thing> v;
        File_handle f;
        string s;
    };
    
    void use()
    {
        Foo bar {{Thing{1}, Thing{2}, Thing{monkey}}, {"my_file", "r"}, "Here we go!"};
        // ...
    }

这里，`vector` 和 `string` 的构造函数可能无法为其元素分配足够的内存，`vector` 的构造函数可能无法复制其初始化式列表中的 `Thing`，而 `File_handle` 可能无法打开所需的文件。
这些情况中，它们都会抛出异常来让 `use()` 的调用者来处理。
如果 `use()` 可以处理这种构造 `bar` 的故障的话，它可以用 `try`/`catch` 来控制。
这些情况下，`Foo` 的构造函数都会在把控制传递给试图创建 `Foo` 的任何代码之前恰当地销毁以及构造的内存。
注意它并不存在可以包含错误代码的返回值。

`File_handle` 的构造函数可以这样定义：

    File_handle::File_handle(const string& name, const string& mode)
        :f{fopen(name.c_str(), mode.c_str())}
    {
        if (!f)
            throw runtime_error{"File_handle: could not open " + name + " as " + mode"}
    }

##### 注解

人们通常说异常应当用于表明意外的事件和故障。
不过这里面存在一点循环，"什么是意外的？"
例如：

* 无法满足的前条件
* 无法构造对象的构造函数（无法建立类的[不变式](#Rc-struct)）
* 越界错误（比如 `v[v.size()] = 7`）
* 无法获得资源（比如网络未连接）

相较而言，终止某个普通的循环则不是意外的。
除非这个循环本应当是无限循环，否则其终止就是正常且符合预期的。

##### 注解

不要用 `throw` 仅仅作为从函数中返回值的另一种方式。

##### 例外

某些系统，比如在执行开始之前就需要保证以（通常很短的）常量最大时间来执行动作的硬实时系统，这样的系统只有当存在工具可以支持精确地预测从一次 `throw` 中恢复的最大时间时才能使用异常。

**参见**: [RAII](#Re-raii)

**参见**: [讨论](#Sd-noexcept)

##### 注解

在你决定你无法负担或者不喜欢基于异常的错误处理之前，请看一看[替代方案](#Re-no-throw-raii)；
它们各自都有自己的复杂性和问题。
同样地，只要可能的话，就应该进行测量之后再发表有关效率的言论。

### <a name="Re-errors"></a>E.3: 仅使用异常来进行错误处理

##### 理由

以保持错误处理和"常规代码"互相分离。
C++ 实现都倾向于基于假定异常的稀有而进行优化。

##### 示例，请勿如此

    // 请勿如此: 异常并未用于错误处理
    int find_index(vector<string>& vec, const string& x)
    {
        try {
            for (gsl::index i = 0; i < vec.size(); ++i)
                if (vec[i] == x) throw i;  // 找到了 x
        } catch (int i) {
            return i;
        }
        return -1;   // 未找到
    }

这种代码要比显然的替代方式更加复杂，而且极可能运行慢得多。
在 `vector` 中寻找一个值是没什么意外情况的。

##### 强制实施

可能应该是启发式措施。
查找从 `catch` 子句"漏掉"的异常值。

### <a name="Re-design-invariants"></a>E.4: 围绕不变式来设计错误处理策略

##### 理由

要使用一个对象，它必须处于某个（正式或非正式通过不变式所定义的）有效的状态，而要从错误中恢复，每个还未销毁的对象也必须处于有效的状态。

##### 注解

[不变式](#Rc-struct)是对象的成员的逻辑条件，构造函数必须进行建立，且为公开的成员函数所假定。

##### 强制实施

???

### <a name="Re-invariant"></a>E.5: 让构造函数建立不变式，若其无法做到则抛出异常

##### 理由

遗留仍未建立不变式的对象将会带来麻烦。
不是任何成员函数都可以对其进行调用。

##### 示例

    class Vector {  // 非常简单的 double 向量
        // 当 elem != nullptr 时 elem 指向 sz 个 double
    public:
        Vector() : elem{nullptr}, sz{0}{}
        Vector(int s) : elem{new double[s]}, sz{s} { /* 元素的初始化 */ }
        ~Vector() { delete [] elem; }
        double& operator[](int s) { return elem[s]; }
        // ...
    private:
        owner<double*> elem;
        int sz;
    };

类不变式——这里以代码注释说明——是由构造函数建立的。
当 `new` 无法分配所需的内存时将抛出异常。
各运算符，尤其是下标运算符，都是依赖于这个不变式的。

**参见**: [当构造函数无法构造有效对象时，应当抛出异常](#Rc-throw)

##### 强制实施

对带有 `private` 状态但没有（公开，受保护或私有的）构造函数的类进行标记。

### <a name="Re-raii"></a>E.6: 使用 RAII 来避免泄漏

##### 理由

资源泄漏通常是不可接受的。
手工的资源释放很易出错。
RAII（Resource Acquisition Is Initialization，资源获取即初始化）是最简单，最系统化的避免泄漏方案。

##### 示例

    void f1(int i)   // 不好: 可能会泄漏
    {
        int* p = new int[12];
        // ...
        if (i < 17) throw Bad{"in f()", i};
        // ...
    }

我们可以在抛出异常前小心地释放资源：

    void f2(int i)   // 笨拙且易错: 显式的释放
    {
        int* p = new int[12];
        // ...
        if (i < 17) {
            delete[] p;
            throw Bad{"in f()", i};
        }
        // ...
    }

这样很啰嗦。在更大型的可能带有多个 `throw` 的代码中，显式的释放将变得重复且易错。

    void f3(int i)   // OK: 通过资源包装来进行资源管理（请见下文）
    {
        auto p = make_unique<int[]>(12);
        // ...
        if (i < 17) throw Bad{"in f()", i};
        // ...
    }

注意即使 `throw` 是在所调用的函数中暗中发生，这也能正常工作：

    void f4(int i)   // OK: 通过资源包装来进行资源管理（请见下文）
    {
        auto p = make_unique<int[]>(12);
        // ...
        helper(i);   // 可能抛出异常
        // ...
    }

除非你确实需要指针语义，否则还是应当使用局部的资源对象：

    void f5(int i)   // OK: 通过局部对象来进行资源管理
    {
        vector<int> v(12);
        // ...
        helper(i);   // 可能抛出异常
        // ...
    }

这即简单又安全，而且通常更加高效。

##### 注解

当没有合适的资源包装，且定义一个适当的 RAII 对象/包装由于某种原因不可行时，
万不得已，可以使用 [`final_action` 对象](#Re-finally)来表达清理动作。

##### 注解

但是当我们所编写的程序不能使用异常时应当怎么办呢？
首先应当质疑这项假设；到处都有许多反异常的错误认识。
据我们所知，只有少量正当理由：

* 我们所在的系统太小，支持异常将会吃掉我们的 2K 内存的大部分。
* 我们所在的是硬实时系统，而且我们没有工具能保证异常会在所需时间内处理掉。
* 我们所在的系统中有成吨的遗留代码以难于理解的方式大量地使用指针
  （尤其是没有可识别的所有权策略），因此异常可能会造成泄露。
* 我们的 C++ 异常机制的实现不合理地糟糕
  （很慢，很耗内存，对于动态链接库无法正确工作，等等）。
  请向你的实现的供应商提出意见；如果没有用户提出意见，就不会出现改进。
* 如果我们质疑经理的古老智慧的话会被炒鱿鱼。

以上原因中只有第一条才是基础问题，因此一旦可能的话，还是要用异常来实现 RAII，或者设计你的 RAII 对象永不失败。
当无法使用异常时，可以模拟 RAII。
就是说，系统化地在对象构造之后检查其有效性，并且仍然在析构函数中释放所有的资源。
一种策略是为每个资源包装添加一个 `valid()` 操作：

    void f()
    {
        vector<string> vs(100);   // 非 std::vector: 添加了 valid()
        if (!vs.valid()) {
            // 处理错误或退出
        }
    
        ifstream fs("foo");   // 非 std::ifstream: 添加了 valid()
        if (!fs.valid()) {
            // 处理错误或退出
        }
    
        // ...
    } // 析构函数如常进行清理

显然这样做增加了代码大小，不允许隐式的"异常"（`valid()` 检查）传播，而且 `valid()` 检查可能被忘掉。
优先采用异常。

**参见**: [`noexcept` 的用法](#Se-noexcept)

##### 强制实施

???

### <a name="Re-precondition"></a>E.7: 明示前条件

##### 理由

避免接口错误。

**参见**: [前条件规则](#Ri-pre)

### <a name="Re-postcondition"></a>E.8: 明示后条件

##### 理由

避免接口错误。

**参见**: [后条件规则](#Ri-post)

### <a name="Re-noexcept"></a>E.12: 当函数不可能或不能接受以 `throw` 来退出时，使用 `noexcept`

##### 理由

使错误处理系统化，强健，且高效。

##### 示例

    double compute(double d) noexcept
    {
        return log(sqrt(d <= 0 ? 1 : d));
    }

这里，我们已知 `compute` 不会抛出异常，因为它仅由不会抛出异常的操作所组成。
通过将 `compute` 声明为 `noexcept`，让编译器和人类阅读者获得信息，使其更容易理解和操作 `compute`。

##### 注解

许多标准库函数都是 `noexcept` 的，这包括所有从 C 标准库中"继承"来的标准库函数。

##### 示例

    vector<double> munge(const vector<double>& v) noexcept
    {
        vector<double> v2(v.size());
        // ... 做一些事 ...
    }

这里的 `noexcept` 表明我不希望或无法处理无法构造局部的 `vector` 对象的情形。
也就是说，我认为内存耗尽是一种严重的设计错误（类比于硬件故障），因此我希望当其发生时让程序崩溃。

##### 注解

请勿使用传统的[异常说明](#Re-specifications)。

##### 参见

[讨论](#Sd-noexcept)。

### <a name="Re-never-throw"></a>E.13: 不要在作为某个对象的直接所有者时抛出异常

##### 理由

这可能导致一次泄漏。

##### 示例

    void leak(int x)   // 请勿如此: 可能泄漏
    {
        auto p = new int{7};
        if (x < 0) throw Get_me_out_of_here{};  // 可能泄漏 *p
        // ...
        delete p;   // 可能不会执行到这里
    }

避免这种问题的一种方法是坚持使用资源包装：

    void no_leak(int x)
    {
        auto p = make_unique<int>(7);
        if (x < 0) throw Get_me_out_of_here{};  // 将按需删除 *p
        // ...
        // 无须 delete p
    }

另一种（通常更好）的方案是使用一个局部变量来消除指针的显式使用：

    void no_leak_simplified(int x)
    {
        vector<int> v(7);
        // ...
    }

##### 注解

如果有需要清理的一些局部"东西"，但并未表示为带用析构函数的对象，则这样的清理
也必须在 `throw` 之前完成。
有时候，[`finally()](#Re-finally) 可以把这种不系统的清理变得更加可管理一些。

### <a name="Re-exception-types"></a>E.14: 应当使用为目的所设计的自定义类型（而不是内建类型）作为异常

##### 理由

自定义类型不大可能会和其他人的异常造成冲突。

##### 示例

    void my_code()
    {
        // ...
        throw Moonphase_error{};
        // ...
    }
    
    void your_code()
    {
        try {
            // ...
            my_code();
            // ...
        }
        catch(const Bufferpool_exhausted&) {
            // ...
        }
    }

##### 示例，请勿如此

    void my_code()     // 请勿如此
    {
        // ...
        throw 7;       // 7 的意思是"月亮在第四象限"
        // ...
    }
    
    void your_code()   // 请勿如此
    {
        try {
            // ...
            my_code();
            // ...
        }
        catch(int i) {  // i == 7 的意思是"输入缓冲区太小"
            // ...
        }
    }

##### 注解

标准库中派生于 `exception` 的类应当仅被当做基类，或者用于仅需要"通用"处理的异常。和内建类型相似，它们的使用可能会与其他人的使用相冲突。

##### 示例，请勿如此

    void my_code()   // 请勿如此
    {
        // ...
        throw runtime_error{"moon in the 4th quarter"};
        // ...
    }
    
    void your_code()   // 请勿如此
    {
        try {
            // ...
            my_code();
            // ...
        }
        catch(const runtime_error&) {   // runtime_error 的含义是"输入缓冲区太小"
            // ...
        }
    }

**参见**: [讨论](#Sd-???)

##### 强制实施

识别内建类型的 `throw` 和 `catch`。可能要对使用标准库 `exception` 类型的 `throw` 和 `catch` 进行警告。当然，派生于 `std::exception` 类型层次的异常是没问题的。

### <a name="Re-exception-ref"></a>E.15: 按引用捕获类型层次中的异常

##### 理由

避免发生切片。

##### 示例

    void f()
    {
        try {
            // ...
        }
        catch (exception e) {   // 请勿如此: 可能造成切片
            // ...
        }
    }

可以代之以引用：

    catch (exception& e) { /* ... */ }

或者（通常更好的）`const` 引用：

    catch (const exception& e) { /* ... */ }

大多数处理器并不会改动异常，一般情况下我们都会[建议使用 `const`](#Res-const)。

##### 注解

重新抛出已捕获的异常应当使用 `throw;` 而非 `throw e;`。使用 `throw e;` 将会抛出 `e` 的一个新副本（并切片成静态类型 `std::exception`），而并非重新抛出原来的 `std::runtime_error` 类型的异常。（但请关注[请勿试图在每个函数中捕获所有的异常](#Re-not-always)，以及[尽可能减少 `try`/`catch` 的显式使用](#Re-catch)。)

##### 强制实施

当异常类型属于某个类型层次时，对按值捕获异常进行标记（这可能需要全程序分析才能完美达成这点）。

### <a name="Re-never-fail"></a>E.16: 析构函数，回收函数，以及 `swap` 决不能失败

##### 理由

如果析构函数，`swap`，或者内存回收函数会失败，就是说如果它通过异常而退出，或者根本不会实施其所需的动作，我们不知道应当如何编写可靠的程序。

##### 示例，请勿如此

    class Connection {
        // ...
    public:
        ~Connection()   // 请勿如此: 非常糟糕的析构函数
        {
            if (cannot_disconnect()) throw I_give_up{information};
            // ...
        }
    };

##### 注解

许多人都曾试图编写违反这条规则的可靠代码，比如当网络连接"拒绝关闭"的情形。
尽我们所知，没人曾找到一个做到这点的通用方案。
虽然偶尔对于非常特殊的例子，你可以通过设置某个状态以便进行将来的清理的方式绕过它。
比如说，我们可能将一个不打算关闭的 socket 放入一个"故障 socket"列表之中，
让其被某个定期的系统状态清理所检查处理。
我们见过的每个这种例子都是易错的，专门的，而且通常有 BUG。

##### 注解

标准库假定析构函数，回收函数（比如 `operator delete`），和 `swap` 都不会抛出异常。当它们这样做时，基本的标准库不变式将会被打破。

##### 注解

回收函数，包括 `operator delete`，必须为 `noexcept`。`swap` 函数必须为 `noexcept`。
大多数的析构函数都是缺省隐含为 `noexcept` 的。
而且，[应该使移动操作为 `noexcept`](#Rc-move-noexcept)。

##### 强制实施

识别会 `throw` 的析构函数，回收操作，和 `swap`。
识别不为 `noexcept` 的这类操作。

**参见**: [讨论](#Sd-never-fail)

### <a name="Re-not-always"></a>E.17: 不要试图在每个函数中捕获每个异常

##### 理由

如果函数无法对异常进行有意义的恢复动作，其捕获这个异常就导致复杂性和浪费。
要让异常传播直到遇到一个可以处理它的函数。
要用 [RAII](#Re-raii) 来处理栈退解路径上�