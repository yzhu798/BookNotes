�清理操作。

##### 示例，请勿如此

    void f()   // 不好
    {
        try {
            // ...
        }
        catch (...) {
            // 不做任何事
            throw;   // 传播异常
        }
    }

##### 强制实施

* 标记嵌套的 `try` 块。
* 对带有过高的 `try` 块/函数比率的源代码文件进行标记。 (??? 问题：定义"过高")

### <a name="Re-catch"></a>E.18: 最小化对 `try`/`catch` 的显式使用

##### 理由

`try`/`catch` 很啰嗦，而且非平凡的使用是易错的。
`try`/`catch` 可以作为对非系统化和/或低级的资源管理或错误处理的一个信号。

##### 示例，不好

    void f(zstring s)
    {
        Gadget* p;
        try {
            p = new Gadget(s);
            // ...
            delete p;
        }
        catch (Gadget_construction_failure) {
            delete p;
            throw;
        }
    }

这段代码很混乱。
可能在 `try` 块中的裸指针上发生泄漏。
不是所有的异常都被处理了。
`delete` 一个构造失败的对象几乎肯定是一个错误。
更好的是：

    void f2(zstring s)
    {
        Gadget g {s};
    }

##### 替代方案

* 合适的资源包装以及 [RAII](#Re-raii)
* [`finally`](#Re-finally)

##### 强制实施

??? 很难，需要启发式方法

### <a name="Re-finally"></a>E.19: 当没有合适的资源包装时，使用 `final_action` 对象来表达清理动作

##### 理由

`finally` 要比 `try`/`catch` 更不啰嗦且难于搞错。

##### 示例

    void f(int n)
    {
        void* p = malloc(n);
        auto _ = finally([p] { free(p); });
        // ...
    }

##### 注解

`finally` 没有 `try`/`catch` 那样混乱，但它仍然比较专门化。
优先采用[适当的资源管理对象](#Re-raii)。
万不得已考虑使用 `finally`。

##### 注解

相对于老式的 [`goto exit;` 技巧](#Re-no-throw-codes)来说，使用 `finally` 是处理并非系统化的资源管理中的清理工作的
更加系统化并且相当简洁的方案。

##### 强制实施

启发式措施：检测 `goto exit;`。

### <a name="Re-no-throw-raii"></a>E.25: 当不能抛出异常时，模拟 RAII 来进行资源管理

##### 理由

即便没有异常，[RAII](#Re-raii) 通常仍然是最佳且最系统化的处理资源的方式。

##### 注解

使用异常进行错误处理，是 C++ 中唯一完整且系统化的处理非局部错误的方式。
特别是，非侵入地对对象构造的失败进行报告需要使用异常。
以无法被忽略的方式报告错误需要使用异常。
如果无法使用异常，应当尽你所能模拟它们的使用。

大量对异常的惧怕是被误导的。
当用在并非充斥指针和复杂控制结构的代码中的违例情形时，
异常处理几乎总是（在时间和空间上）可以负担，且几乎总会导致更好的代码。
当然这假定存在一个优良的异常处理机制实现，而这并非在所有系统上都存在。
存在另一些情况并不适用于上述问题，但由于其他原因而无法使用异常。
一个例子是一些硬实时系统：必须在固定的时间之内完成操作并得到错误或正确的响应。
在没有适当的时间评估工具的条件下，很难对异常作出保证。
这样的系统（比如飞控系统）通常也会禁止使用动态（堆）内存。

因此，对于错误处理的主要指导方针还是"使用异常和 [RAII](#Re-raii)。"
本节所处理的情况是，要么你没有高效的异常实现，
或者要面对大量的老式代码
（比如说，大量的指针，不明确定义的所有权，以及大量的不系统化的基于错误代码检查的错误处理）
而且向其引入简单且系统化的错误处理的做法不可行。

在宣称不能使用异常或者抱怨它们成本过高之前，应当考虑一下使用[错误代码](#Re-no-throw-codes)的例子。
请考虑使用错误代码的成本和复杂性。
如果你担心性能的话，请进行测量。

##### 示例

假定你想要编写

    void func(zstring arg)
    {
        Gadget g {arg};
        // ...
    }

当这个 `g` 并未正确构造时，`func` 将以一个异常退出。
当无法抛出异常时，我们可以通过向 `Gadget` 添加 `valid()` 成员函数来模拟 RAII 风格的资源包装：

    error_indicator func(zstring arg)
    {
        Gadget g {arg};
        if (!g.valid()) return gadget_construction_error;
        // ...
        return 0;   // 零代表"正常"
    }

显然问题现在变成了调用者必须记得测试其返回值。

**参见**: [讨论](#Sd-???)

##### 强制实施

（仅）对于这种想法的特定版本是可能的：比如检查资源包装的构造后进行系统化的 `valid()` 测试。

### <a name="Re-no-throw-crash"></a>E.26: 当不能抛出异常时，考虑采取快速失败

##### 理由

如果你无法做好错误恢复的话，至少你可以在发生更多后续的损害之前拜托出来。

**参见**：[模拟 RAII](#Re-no-throw-raii)

##### 注解

当你无法系统化地进行错误处理时，考虑以"程序崩溃"作为对任何无法局部处理的错误的回应。
就是说，如果你无法在检测到错误的函数的上下文中处理它，则调用 `abort()`，`quick_exit()`，
或者相似的某个将触发某种系统重启的函数。

在具有大量进程和/或大量计算机的系统中，你总要预计到并处理这些关键程序崩溃，
比如说源于硬件故障而引发。
这种情况下，"程序崩溃"只不过把错误处理留给了系统的下一个层次。

##### 示例

    void f(int n)
    {
        // ...
        p = static_cast<X*>(malloc(n * sizeof(X)));
        if (!p) abort();     // 当内存耗尽时 abort
        // ...
    }

大多数程序都无法得体地处理内存耗尽。这大略上等价于

    void f(int n)
    {
        // ...
        p = new X[n];    // 当内存耗尽时抛出异常（默认情况会调用 terminate）
        // ...
    }

通常，在退出之前将"崩溃"的原因记录日志是个好主意。

##### 强制实施

很难对付

### <a name="Re-no-throw-codes"></a>E.27: 当不能抛出异常时，系统化地使用错误代码

##### 理由

系统化地使用任何错误处理策略都能最小化忘记处理错误的机会。

**参见**：[模拟 RAII](#Re-no-throw-raii)

##### 注解

需要处理几个问题：

* 如何从函数向外传递错误指示？
* 如何在错误退出函数之前释放所有资源？
* 使用什么来作为错误指示？

通常，返回错误指示意味着返回两个值：其结果以及一个错误指示。
错误指示可以是对象的一部分，例如对象可以带有 `valid()` 指示，
也可以返回一对值。

##### 示例

    Gadget make_gadget(int n)
    {
        // ...
    }
    
    void user()
    {
        Gadget g = make_gadget(17);
        if (!g.valid()) {
                // 错误处理
        }
        // ...
    }

这种方案符合[模拟 RAII 资源管理](#Re-no-throw-raii)。
`valid()` 函数可以返回一个 `error_indicator`（比如说 `error_indicator` 枚举的某个成员）。

##### 示例

要是我们无法或者不想改动 `Gadget` 类型呢？
这种情况下，我们只能返回一对值。
例如：

    std::pair<Gadget, error_indicator> make_gadget(int n)
    {
        // ...
    }
    
    void user()
    {
        auto r = make_gadget(17);
        if (!r.second) {
                // 错误处理
        }
        Gadget& g = r.first;
        // ...
    }

可见，`std::pair` 是一种可能的返回类型。
某些人则更喜欢专门的类型。
例如：

    Gval make_gadget(int n)
    {
        // ...
    }
    
    void user()
    {
        auto r = make_gadget(17);
        if (!r.err) {
                // 错误处理
        }
        Gadget& g = r.val;
        // ...
    }

倾向于专门返回类型的一种原因是为其成员提供命名，而不是使用多少有些隐秘的 `first` 和 `second`,
而且可以避免与 `std::pair` 的其他使用相混淆。

##### 示例

通常，在错误退出之前必须进行清理。
这样做是很混乱的：

    std::pair<int, error_indicator> user()
    {
        Gadget g1 = make_gadget(17);
        if (!g1.valid()) {
                return {0, g1_error};
        }
    
        Gadget g2 = make_gadget(17);
        if (!g2.valid()) {
                cleanup(g1);
                return {0, g2_error};
        }
    
        // ...
    
        if (all_foobar(g1, g2)) {
            cleanup(g1);
            cleanup(g2);
            return {0, foobar_error};
        // ...
    
        cleanup(g1);
        cleanup(g2);
        return {res, 0};
    }

模拟 RAII 可能不那么简单，尤其是在带有多个资源和多种可能错误的函数之中。
一种较为常见的技巧是把清理都集中到函数末尾以避免重复（注意这里本不必为 `g2` 增加一层作用域，但是编译 `goto` 版本却需要它）：

    std::pair<int, error_indicator> user()
    {
        error_indicator err = 0;
    
        Gadget g1 = make_gadget(17);
        if (!g1.valid()) {
                err = g1_error;
                goto exit;
        }
    
        {
        Gadget g2 = make_gadget(17);
        if (!g2.valid()) {
                err = g2_error;
                goto exit;
        }
    
        if (all_foobar(g1, g2)) {
            err = foobar_error;
            goto exit;
        }
        // ...
        }
    
    exit:
      if (g1.valid()) cleanup(g1);
      if (g1.valid()) cleanup(g2);
      return {res,err};
    }

函数越大，这种技巧就越有吸引力。
`finally` 可以[略微缓解这个问题](#Re-finally)。
而且，程序变得越大，系统化地采用一中基于错误指示的错误处理策略就越加困难。

我们[优先采用基于异常的错误处理](#Re-throw)，并建议[保持函数短小](#Rf-single)。

**参见**: [讨论](#Sd-???)

**参见**: [返回多个值](#Rf-out-multi)

##### 强制实施

很难对付。

### <a name="Re-no-throw"></a>E.28: 避免基于全局状态（比如 `errno`）的错误处理

##### 理由

全局状态难于管理，且易于忘记检查。
你上次检查 `printf()` 的返回值是什么时候了？

**参见**：[模拟 RAII](#Re-no-throw-raii)

##### 示例，不好

    int last_err;
    
    void f(int n)
    {
        // ...
        p = static_cast<X*>(malloc(n * sizeof(X)));
        if (!p) last_err = -1;     // 当内存耗尽时发生的错误
        // ...
    }

##### 注解

C 风格的错误处理就是基于全局变量 `errno` 的，因此基本上不可能完全避免这种风格。

##### 强制实施

很难对付。


### <a name="Re-specifications"></a>E.30: 请勿使用异常说明

##### 理由

异常说明使得错误处理变得脆弱，隐含一些运行时开销，并且已经从 C++ 标准中被删除了。

##### 示例

    int use(int arg)
        throw(X, Y)
    {
        // ...
        auto x = f(arg);
        // ...
    }

当 `f()` 抛出了不同于 `X` 和 `Y` 的异常时将会执行未预期异常处理器，其默认将终止程序。
这没什么问题，但假定我们检查过着并不会发生而 `f` 则被改写为抛出某个新异常 `Z`，
这样将导致程序崩溃，除非我们改写 `use()`（并重新测试所有东西）。
障碍在于 `f()` 可能在某个我们无法控制的程序库中，而对于新的异常 `use()`
没办法对其做任何事，或者对其完全不感兴趣。
我们可以改写 `use()` 使其传递 `Z` 出去，但这样的话 `use()` 的调用方可能也需要被改写。
如此事态将很快变得无法掌控。
或者，我们可以在 `use()` 中添加一个 `try`-`catch` 以将 `Z` 映射为某种可以接受的异常。
这种方法也会很快变得无法掌控。
注意，对异常集合的改动通常都发生在系统的最底层
（比如说，当改换了网络库或者某种中间件时），因此改变将沿着冗长的调用链"冒泡上浮"。
在大型代码库中，这将意味着直到最后一个使用方也被改写之前，没人可以更新某个库到新版本。
如果 `use()` 是某个库的一部分，则也许不可能对其进行更新，因为其改动可能影响到未知的客户代码。

而让异常继续传递直到其到达某个潜在可以处理它的函数的策略，已经在多年的实践中得到了证明。

##### 注解

静态强制检查异常说明并不会带来任何好处。
相关例子请参见 [Stroustrup94](#Stroustrup94)。

##### 注解

当不会抛出异常时，请使用 [`noexcept`](#Re-noexcept) 或其等价的 `throw()`。

##### 强制实施

标记每个异常说明。

### <a name="Re_catch"></a>E.31: 恰当地对 `catch` 子句排序

##### 理由

`catch` 子句是以其出现顺序依次求值的，而其中一个可能会隐藏掉另一个。

##### 示例

    void f()
    {
        // ...
        try {
                // ...
        }
        catch (Base& b) { /* ... */ }
        catch (Derived& d) { /* ... */ }
        catch (...) { /* ... */ }
        catch (std::exception& e){ /* ... */ }
    }

若 `Derived` 派生自 `Base` 则 `Derived` 的处理器永远不会被执行。
"捕获任何东西"的处理器保证 `std::exception` 的处理器永远不会被执行。

##### 强制实施

标记出所有的"隐藏处理器"。

# <a name="S-const"></a>Con: 常量与不可变性

常量是不会出现竞争条件的。
当大量的对象不会改变它们的值时，对程序进行推理将变得更容易。
承诺"不改动"作为参数所传递对象的接口，极大地提升了可读性。

常量规则概览：

* [Con.1: 缺省情况下，对象应当是不可变的](#Rconst-immutable)
* [Con.2: 缺省情况下，成员函数应当为 `const`](#Rconst-fct)
* [Con.3: 缺省情况下，应当传递指向 `const` 对象的指针或引用](#Rconst-ref)
* [Con.4: 构造之后不再改变其值的对象应当以 `const` 来定义](#Rconst-const)
* [Con.5: 以 `constexpr` 来定义可以在编译期计算的值](#Rconst-constexpr)

### <a name="Rconst-immutable"></a>Con.1: 缺省情况下，对象应当是不可变的

##### 理由

不可变对象更易于进行推理，应仅当需要改动对象的值时，才使之为非 `const` 对象。
避免出现意外造成的或者很难发觉的值的改变。

##### 示例

    for (const int i : c) cout << i << '\n';    // 仅进行读取: const
    
    for (int i : c) cout << i << '\n';          // 不好: 仅进行读取

##### 例外

函数参数时很少改动的，但也很少被声明为 `const`。
为了避免造成混淆和大量的误报，不要对函数参数参数实施这条规则。。

    void f(const char* const p); // 迂腐
    void g(const int i);        // 迂腐

注意，函数参数时局部变量，其改动也是局部的。

##### 强制实施

* 标记未发生改动的非 `const` 变量（排除参数以避免误报）

### <a name="Rconst-fct"></a>Con.2: 缺省情况下，成员函数应当为 `const`

##### 理由

除非成员函数会改变对象的可观察状态，否则它应当标记为 `const`。
这样做更精确地描述了设计意图，具有更佳的可读性，编译器可以识别更多的错误，而且有时能够带来更多的优化机会。

##### 示例，不好

    class Point {
        int x, y;
    public:
        int getx() { return x; }    // 不好，应当为 const，它并不改变对象的状态
        // ...
    };
    
    void f(const Point& pt) {
        int x = pt.getx();          // 错误，无法通过编译，因为 getx 并未标记为 const
    }

##### 注解

传递非 `const` 的指针或引用并非天生就是不好的，
但应当只有在所调用的函数预计会修改这个对象时才这样做。
代码的读者必须假定接受"普通的" `T*` 或 `T&` 的函数都将会修改其所指代的对象。
如果它现在不会，那它可能以后会，且无需强制要求重新编译。

##### 注解

有些代码和程序库提供的函数是声明为 `T*`，
但这些函数并不会修改这个 `T`。
这对于进行代码现代化转换的人们来说是个问题。
你可以：

* 如果倾向于长期解决方案的话，将程序库更新为 `const` 正确的；
* "强制掉 `const`"（[最好避免这样做](#Res-casts-const)）；
* 提供包装函数。

例如：

    void f(int* p);   // 老代码：f() 并不会修改 `*p`
    void f(const int* p) { f(const_cast<int*>(p)); } // 包装函数

注意，这种包装函数的方案是一种补丁，只能在无法修改 `f()` 的声明时才使用它，
比如当它属于某个你无法修改的程序库时。

##### 注解

`const` 成员函数可以改动 `mutable` 对象的值，或者通过某个指针成员改动对象的值。
一种常见用法是来维护一个缓存以避免重复进行复杂的运算。
例如，这里的 `Date` 缓存（记住）了其字符串表示，以简化其重复使用：

    class Date {
    public:
        // ...
        const string& string_ref() const
        {
            if (string_val == "") compute_string_rep();
            return string_val;
        }
        // ...
    private:
        void compute_string_rep() const;    // 计算字符串表示并将其存入 string_val
        mutable string string_val;
        // ...
    };

另一种说法是 `const` 特性不会传递。
通过 `const` 成员函数改动 `mutable` 成员的值和通过非 `const` 指针来访问的对象的值
是有可能的。
由类负责确保这样的改动仅当根据其语义（不变式）对于其用户有意义时
才会发生。

**参见**：[PImpl](#Ri-pimpl)

##### 强制实施

* 如果未标记为 `const` 的成员函数并未对任何成员变量实施非 `const` 操作的话，对其进行标记。

### <a name="Rconst-ref"></a>Con.3: 缺省情况下，应当传递指向 `const` 对象的指针或引用

##### 理由

避免所调用的函数意外地改变了这个值。
如果被调用的函数不会改动状态的话，对程序的推理将变得容易得多。

##### 示例

    void f(char* p);        // f 会不会修改 *p?（假定它会修改）
    void g(const char* p);  // g 不会修改 *p

##### 注解

传递指向非 `const` 对象的指针或引用并不是天生就有问题的，
不过只有当所调用的函数本就有意改动对象时才能这样做。

##### 注解

[请勿强制掉 `const`](#Res-casts-const)。

##### 强制实施

* 如果函数并未修改以指向非 `const` 的指针或引用传递的对象，则对其进行标记。
* 如果函数（利用强制转换）修改了以指向 `const` 的指针或引用传递的对象，则对其进行标记。

### <a name="Rconst-const"></a>Con.4: 构造之后不再改变其值的对象应当以 `const` 来定义

##### 理由

避免意外地改变对象的值。

##### 示例

    void f()
    {
        int x = 7;
        const int y = 9;
    
        for (;;) {
            // ...
        }
        // ...
    }

既然 `x` 并非 `const`，我们就必须假定它可能在循环中的某处会被修改。

##### 强制实施

* 标记并未被修改的非 `const` 变量。

### <a name="Rconst-constexpr"></a>Con.5: 以 `constexpr` 来定义可以在编译期计算的值

##### 理由

更好的性能，更好的编译期检查，受保证的编译期求值，竞争条件可能性为零。

##### 示例

    double x = f(2);            // 可能在运行时求值
    const double y = f(2);      // 可能在运行时求值
    constexpr double z = f(2);  // 除非 f(2) 可在编译期求值，否则会报错

##### 注解

参见 F.4。

##### 强制实施

* 对带有常量表达式初始化式的 `const` 定义进行标记。

# <a name="S-templates"></a>T: 模板和泛型编程

泛型编程是使用以类型、值和算法进行参数化的类型和算法所进行的编程。
在 C++ 中，泛型编程是以"模板"语言机制来提供支持的。

泛型函数的参数是以针对参数类型及其所涉及的值的规定的集合来刻画的。
在 C++ 中，这些规定是以被称为概念（concept）的编译期谓词来表达的。

模板还可用于进行元编程；亦即由编译期代码所组成的程序。

泛型编程的中心是"概念"；亦即以编译时谓词表现的对于模板参数的要求。
"概念"是在一份 ISO 技术规范中定义的：[concepts](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2015/n4553.pdf)。
而一组标准库概念的草案则可以在另一份 ISO TS 中找到：[ranges](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2016/n4569.pdf)。
GCC 6.1 及其后版本支持概念。
因此，我们在例子中将概念注释掉了；就是说我们仅把它们当成形式化的注释。
如果你使用 GCC 6.1 或更新版本，那么你就可以取消它们的注释。

模板使用的规则概览：

* [T.1: 用模板来提升代码的抽象层次](#Rt-raise)
* [T.2: 用模板来表达适用于许多参数类型的算法](#Rt-algo)
* [T.3: 用模板来表达容器和范围](#Rt-cont)
* [T.4: 用模板来表达对语法树的操作](#Rt-expr)
* [T.5: 结合泛型和面向对象技术来增强它们的能力，而不是它们的成本](#Rt-generic-oo)

概念使用的规则概览：

* [T.10: 为所有模板实参指明概念](#Rt-concepts)
* [T.11: 尽可能采用标准概念](#Rt-std-concepts)
* [T.12: 对于局部变量，优先采用概念名而不是 `auto`](#Rt-auto)
* [T.13: 对于简单的单类型参数概念，优先采用简写形式](#Rt-shorthand)
* ???

概念定义的规则概览：

* [T.20: 避免没有有意义的语义的"概念"](#Rt-low)
* [T.21: 为概念提出一组完整的操作要求](#Rt-complete)
* [T.22: 为概念指明公理](#Rt-axiom)
* [T.23: 通过添加新的使用模式，从更一般情形的概念中区分出提炼后的概念](#Rt-refine)
* [T.24: 用标签类或特征类来区分仅在语义上存在差别的概念](#Rt-tag)
* [T.25: 避免互补性的约束](#Rt-not)
* [T.26: 优先采用使用模式而不是简单的语法来定义概念](#Rt-use)
* [T.30: 节制地采用概念求反（`!C<T>`）来表示微小差异](#Rt-not)
* [T.31: 节制地采用概念析取（disjunction）（`C1<T> || C2<T>`）来表示备选项](#Rt-or)
* ???

模板接口的规则概览：

* [T.40: 使用函数对象向算法传递操作](#Rt-fo)
* [T.41: 在模板的概念上仅提出基本的性质要求](#Rt-essential)
* [T.42: 使用模板别名来简化写法并隐藏实现细节](#Rt-alias)
* [T.43: 优先使用 `using` 而不是 `typedef` 来定义别名](#Rt-using)
* [T.44: （如果可行）使用函数模板来对类模板的参数类型进行推断](#Rt-deduce)
* [T.46: 要求模板参数至少为 `Regular` 或者 `SemiRegular`](#Rt-regular)
* [T.47: 避免用常用名字命名高度可见的无约束模板](#Rt-visible)
* [T.48: 如果你的编译器不支持概念的话，可以用 `enable_if` 来模拟](#Rt-concept-def)
* [T.49: 尽可能避免类型擦除](#Rt-erasure)

模板定义的规则概览：

* [T.60: 最小化模板的上下文依赖性](#Rt-depend)
* [T.61: 请勿对成员进行过度参数化（恐怖）](#Rt-scary)
* [T.62: 将无依赖的类模板成员置于一个非模板基类之中](#Rt-nondependent)
* [T.64: 用特化来提供类模板的其他实现](#Rt-specialization)
* [T.65: 用标签分派来提供函数的其他实现](#Rt-tag-dispatch)
* [T.67: 用特化来提供不规则类型的其他实现](#Rt-specialization2)
* [T.68: 在模板中用 `{}` 而不是 `()` 以避免歧义](#Rt-cast)
* [T.69: 在模板中，请勿进行未限定的非成员函数调用，除非有意将之作为定制点](#Rt-customization)

模板和类型层次的规则概览：

* [T.80: 请勿不成熟地对类层次进行模板化](#Rt-hier)
* [T.81: 请勿混合类层次和数组](#Rt-array) // ??? 放在"类型层次"部分
* [T.82: 当不想要虚函数时，可以将类层次线性化](#Rt-linear)
* [T.83: 请勿声明虚的成员函数模板](#Rt-virtual)
* [T.84: 使用非模板的核心实现来提供 ABI 稳定的接口](#Rt-abi)
* [T.??: ????](#Rt-???)

变参模板的规则概览：

* [T.100: 当需要可以接受可变数量的多种类型参数的函数时，使用变参模板](#Rt-variadic)
* [T.101: ??? 如何向变参模板传递参数 ???](#Rt-variadic-pass)
* [T.102: ??? 如何处理变参模板的参数 ???](#Rt-variadic-process)
* [T.103: 请勿对同质参数列表使用变参模板](#Rt-variadic-not)
* [T.??: ????](#Rt-???)

元编程的规则概览：

* [T.120: 仅当确实需要时才使用模板元编程](#Rt-metameta)
* [T.121: 模板元编程主要用于模拟概念机制](#Rt-emulate)
* [T.122: 用模板（通常为模板别名）来在编译期进行类型运算](#Rt-tmp)
* [T.123: 用 `constexpr` 函数来在编译期进行值运算](#Rt-fct)
* [T.124: 优先使用标准库的模板元编程设施](#Rt-std-tmp)
* [T.125: 当需要标准库之外的模板元编程设施时，使用某个现存程序库](#Rt-lib)
* [T.??: ????](#Rt-???)

其他模板规则概览：

* [T.140: 对所有的可能会被重用的操作命名](#Rt-name)
* [T.141: 当仅在一个地方需要一个简单的函数对象时，使用无名的 lambda](#Rt-lambda)
* [T.142: 使用模板变量以简化写法](#Rt-var)
* [T.143: 请勿编写并非有意非泛型的代码](#Rt-nongeneric)
* [T.144: 请勿特化函数模板](#Rt-specialize-function)
* [T.150: 用 `static_assert` 来检查类是否与概念相符](#Rt-check-class)
* [T.??: ????](#Rt-???)

## <a name="SS-GP"></a>T.gp: 泛型编程

泛型编程是利用以类型、值和算法进行了参数化的类型和算法所进行的编程。

### <a name="Rt-raise"></a>T.1: 用模板来提升代码的抽象层次

##### 理由

通用性。重用。效率。鼓励用户类型的定义一致性。

##### 示例，不好

概念上说，以下要求是错误的，因为我们所要求的 `T` 不止是如"可以增量"或"可以进行加法"这样的非常低级的概念：

    template<typename T>
        // requires Incrementable<T>
    T sum1(vector<T>& v, T s)
    {
        for (auto x : v) s += x;
        return s;
    }
    
    template<typename T>
        // requires Simple_number<T>
    T sum2(vector<T>& v, T s)
    {
        for (auto x : v) s = s + x;
        return s;
    }

由于假设 `Incrementable` 并不支持 `+`，而 `Simple_number` 也不支持 `+=`，我们对 `sum1` 和 `sum2` 的实现者过度约束了。
而且，这种情况下也错过了一次通用化的机会。

##### 示例

    template<typename T>
        // requires Arithmetic<T>
    T sum(vector<T>& v, T s)
    {
        for (auto x : v) s += x;
        return s;
    }

通过假定 `Arithmetic` 同时要求 `+` 和 `+=`，我们约束 `sum` 的使用者来提供完整的算术类型。
这并非是最小的要求，但它为算法的实现者更多所需的自由度，并确保了任何 `Arithmetic` 类型
都可被用于各种各样的算法。

为达成额外的通用性和可重用性，我们还可以用更通用的 `Container` 或 `Range` 概念而不是仅投入到 `vector` 一个容器上。

##### 注解

如果我们所定义的模板提出的要求，完全是由一个算法的一个实现所要求的操作
（比如说仅要求 `+=` 而不同时要求 `=` 和 `+`），而没有别的要求，就会对维护者提出过度的约束。
我们的目标是最小化模板参数的要求，但一个实现的绝对最小要求很少能作为有意义的概念。

##### 注解

可以用模板来表现几乎任何东西（它是图灵完备的），但（利用模板进行）泛型编程的目标在于，
使操作和算法对于一组带有相似语义性质的类型有效地进行通用化。

##### 注解

代码注释中的 `requires` 是 `concept` 的用法。
"概念"是在一份 ISO 技术规范中定义的：[concepts](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2015/n4553.pdf)。
GCC 6.1 及其后版本支持概念。
因此，我们在例子中将概念注释掉了；就是说我们仅把它们当成形式化的注释。
如果你使用 GCC 6.1 或更新版本，那么你就可以取消它们的注释。

##### 强制实施

* 对带有"过于简单"的要求（诸如不用概念而直接使用特定的运算符）的算法进行标记。
* 不要对"过于简单"的概念本身进行标记；它们可能只不过是更有用的概念的构造块。

### <a name="Rt-algo"></a>T.2: 用模板来表达适用于许多参数类型的算法

##### 理由

通用性。最小化源代码数量。互操作性。重用。

##### 示例

这正是 STL 的基础所在。一个 `find` 算法可以很轻易地同任何输入范围一起工作：

    template<typename Iter, typename Val>
        // requires Input_iterator<Iter>
        //       && Equality_comparable<Value_type<Iter>, Val>
    Iter find(Iter b, Iter e, Val v)
    {
        // ...
    }

##### 注解

除非确实需要多于一种模板参数类型，否则请不要使用模板。
请勿过度抽象。

##### 强制实施

??? 很难办，可能需要人来进行

### <a name="Rt-cont"></a>T.3: 用模板来表达容器和范围

##### 理由

容器需要一种元素类型，而将之表示为一个模板参数则是通用的，可重用的，且类型安全的做法。
这样也避免了采用脆弱的或者低效的变通手段。约定：这正是 STL 的做法。

##### 示例

    template<typename T>
        // requires Regular<T>
    class Vector {
        // ...
        T* elem;   // 指向 sz 个 T
        int sz;
    };
    
    Vector<double> v(10);
    v[7] = 9.9;

##### 示例，不好

    class Container {
        // ...
        void* elem;   // 指向 sz 个某种类型的元素
        int sz;
    };
    
    Container c(10, sizeof(double));
    ((double*) c.elem)[7] = 9.9;

这样做无法直接表现出程序员的意图，而且向类型系统和优化器隐藏了程序的结构。

把 `void*` 隐藏在宏之中只会掩盖问题，并引入新的发生混乱的机会。

**例外**: 当需要 ABI 稳定的接口时，也许你不得不提供一个基本实现，在基于它来呈现（类型安全的）目标。
参见[稳定的基类](#Rt-abi)。

##### 强制实施

* 对出现于低级实现代码之外的 `void*` 和强制转换进行标记。

### <a name="Rt-expr"></a>T.4: 用模板来表达对语法树的操作

##### 理由

 ???

##### 示例

    ???

**例外**: ???

### <a name="Rt-generic-oo"></a>T.5: 结合泛型和面向对象技术来增强它们的能力，而不是它们的成本

##### 理由

泛型和面向对象技术是互补的。

##### 示例

静态能够帮助动态：使用静态多态来实现动态多态的接口：

    class Command {
        // 纯虚函数
    };
    
    // 实现
    template</*...*/>
    class ConcreteCommand : public Command {
        // 实现虚函数
    };

##### 示例

动态有助于静态：提供通用的，便利的，静态绑定的接口，但内部进行动态派发，这样就可以提供统一的对象布局。
这样的例子包括如 `std::shared_ptr` 的删除器的类型擦除。（不过[请勿过度使用类型擦除](#Rt-erasure)。）

##### 注解

类模板之中，非虚函数仅会在被使用时才会实例化——而虚函数则每次都会实例化。
这会导致代码大小膨胀，而且可能因为实例化从不需要的功能而导致对通用类型的过度约束。
请避免这样做，虽然标准的刻面类犯过这种错误。

##### 参见

* ref ???
* ref ???
* ref ???

##### 强制实施

参见参考条目以获得更具体的规则。

## <a name="SS-concepts"></a>T.concepts: 概念规则

概念是一种用于为模板参数指定要求的设施。
它是一项 [ISO 技术规范](#Ref-conceptsTS)，但当前仅由 GCC 所支持。
然而，在考虑泛型编程，以及未来的 C++ 程序库（无论标准的还是其他的）的基础时，
概念都是关键性的。

本部分假定有概念支持。

概念使用的规则概览：

* [T.10: 为所有模板实参指明概念](#Rt-concepts)
* [T.11: 尽可能采用标准概念](#Rt-std-concepts)
* [T.12: 优先采用概念名而不是 `auto`](#Rt-auto)
* [T.13: 对于简单的单类型参数概念，优先采用简写形式](#Rt-shorthand)
* ???

概念定义的规则概览：

* [T.20: 避免没有有意义的语义的"概念"](#Rt-low)
* [T.21: 为概念提出一组完整的操作要求](#Rt-complete)
* [T.22: 为概念指明公理](#Rt-axiom)
* [T.23: 通过添加新的使用模式，从更一般情形的概念中区分出提炼后的概念](#Rt-refine)
* [T.24: 用标签类或特征类来区分仅在语义上存在差别的概念](#Rt-tag)
* [T.25: 避免互补性的约束](#Rt-not)
* [T.26: 优先采用使用模式而不是简单的语法来定义概念](#Rt-use)
* ???

## <a name="SS-concept-use"></a>T.con-use: 概念的使用

### <a name="Rt-concepts"></a>T.10: 为所有模板实参指明概念

##### 理由

正确性和可读性。
针对模板参数所假定的含义（包括语法和语义），是模板接口的基础部分。
使用概念能够显著改善模板的文档和错误处理。
为模板参数指定概念是一种强力的设计工具。

##### 示例

    template<typename Iter, typename Val>
    //    requires Input_iterator<Iter>
    //             && Equality_comparable<Value_type<Iter>, Val>
    Iter find(Iter b, Iter e, Val v)
    {
        // ...
    }

也可以等价地用更为简洁的方式：

    template<Input_iterator Iter, typename Val>
    //    requires Equality_comparable<Value_type<Iter>, Val>
    Iter find(Iter b, Iter e, Val v)
    {
        // ...
    }

##### 注解

"概念"是在一份 ISO 技术规范中定义的：[concepts](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2015/n4553.pdf)。
而一组标准库概念的草案则可以在另一份 ISO TS 中找到：[ranges](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2016/n4569.pdf)。
GCC 6.1 及其后版本支持概念。
因此，我们在例子中将概念注释掉了；就是说我们仅把它们当成形式化的注释。
如果你使用 GCC 6.1 或更新版本，那么你就可以取消它们的注释。

    template<typename Iter, typename Val>
        requires Input_iterator<Iter>
               && Equality_comparable<Value_type<Iter>, Val>
    Iter find(Iter b, Iter e, Val v)
    {
        // ...
    }

##### 注解

普通的 `typename`（或 `auto`）是受最少约束的概念。
应当仅在只能假定"这是一个类型"的罕见情况中使用它们。
通常，这只会在当我们（用模板元编程代码）操作纯粹的表达式树，并推迟进行类型检查时才会需要。

**参考**: TC++PL4, Palo Alto TR, Sutton

##### 强制实施

对没有概念的模板类型参数进行标记。

### <a name="Rt-std-concepts"></a>T.11: 尽可能采用标准概念

##### 理由

"标准"概念（即由 [GSL](#S-GSL) 和 [Ranges TS](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2016/n4569.pdf)，以及希望近期 ISO 标准自身所提供的）概念，
避免了我们思考自己的概念，它们比我们匆忙中能够想出来的要好得多，而且还提升了互操作性。

##### 注解

如果你不是要创建一个新的泛型程序库的话，大多数所需的概念都已经在标准库中定义过了。

##### 示例（采用 TS 版本的概念）

    template<typename T>
        // 请勿定义这个: GSL 中已经有 Sortable
    concept Ordered_container = Sequence<T> && Random_access<Iterator<T>> && Ordered<Value_type<T>>;
    
    void sort(Ordered_container& s);

这个 `Ordered_container` 貌似相当合理，但它和 GSL（以及 Range TS）中的 `Sortable` 概念非常相似。
它是更好？更正确？它真的精确地反映了标准对于 `sort` 的要求吗？
直接使用 `Sortable` 则更好而且更简单：

    void sort(Sortable& s);   // 更好

##### 注解

在我们推进一个包含概念的 ISO 标准的过程中，"标准"概念的集合是不断演进的。

##### 注解

设计一个有用的概念是很有挑战性的。

##### 强制实施

很难。

* 查找无约束的参数，使用"非常规"或非标准的概念的模板，以及使用"自造的"又没有公理的概念的目标。
* 开发一种概念识别工具（例如，参考[一种早期实验](http://www.stroustrup.com/sle2010_webversion.pdf)）。

### <a name="Rt-auto"></a>T.12: 对于局部变量，优先采用概念名而不是 `auto`

##### 理由

`auto` 是最弱的概念。概念的名字会比仅用 `auto` 传达出更多的意义。

##### 示例（采用 TS 版本的概念）

    vector<string> v{ "abc", "xyz" };
    auto& x = v.front();     // 不好
    String& s = v.front();   // 好（String 是 GSL 的一个概念）

##### 强制实施

* ???

### <a name="Rt-shorthand"></a>T.13: 对于简单的单类型参数概念，优先采用简写形式

##### 理由

可读性。直接表达意图。

##### 示例（采用 TS 版本的概念）

这样来表达"`T` 是一种 `Sortable`"：

    template<typename T>       // 正确但很啰嗦："参数的类型
    //    requires Sortable<T>   // 为 T，这是某个 Sortable
    void sort(T&);             // 类型的名字"
    
    template<Sortable T>       // 有改善（假定支持概念）："参数的类型
    void sort(T&);             // 为 Sortable 的类型 T"
    
    void sort(Sortable&);      // 最佳方式（假定支持概念）："参数为 Sortable"

越简练的版本越符合我们的说话方式。注意许多模板不在需要使用 `template` 关键字了。

##### 注解

"概念"是在一份 ISO 技术规范中定义的：[concepts](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2015/n4553.pdf)。
而一组标准库概念的草案则可以在另一份 ISO TS 中找到：[ranges](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2016/n4569.pdf)。
GCC 6.1 及其后版本支持概念。
因此，我们在例子中将概念注释掉了；就是说我们仅把它们当成形式化的注释。
如果你使用支持概念的编译器（比如 GCC 6.1 或更新版本），那么你就可以删掉 `//`。

##### 强制实施

* 当人们从 `<typename T>` and `<class T>` 写法进行转换时，使用简短形式是不可行的。
* 之后，如果声明中首先引入了一个 `typename`，之后又用简单的单类型参数概念对其进行约束的话，就对其进行标记。

## <a name="SS-concepts-def"></a>T.concepts.def: 概念定义规则

定义恰当的概念并不简单。
概念是用于表现应用领域中的基本概念的（正如其名"概念"）。
只是把用在某个具体的类或算法的参数上的一组语法约束聚在一起，并不是概念所设计的用法，
也无法获得这个机制的全部好处。

显然，定义概念对于那些可以使用某个实现（比如 GCC 6.1 或更新版本）的代码是最有用的，
不过定义概念本身就是一种有益的设计技巧，有助于发现概念上的错误并清理实现中的各种概念。

### <a name="Rt-low"></a>T.20: 避免没有有意义的语义的"概念"

##### 理由

概念是用于表现语义的观念的，比如"数"，元素的"范围"，以及"全序的"等等。
简单的约束，比如"带有 `+` 运算符"和"带有 `>` 运算符"，是无法独立进行有意义的运用的，
而仅应当被用作有意义的概念的构造块，而不是在用户代码中使用。

##### 示例，不好（采用 TS 版本的概念）

    template<typename T>
    concept Addable = has_plus<T>;    // 不好，不充分
    
    template<Addable N> auto algo(const N& a, const N& b) // 使用两个数值
    {
        // ...
        return a + b;
    }
    
    int x = 7;
    int y = 9;
    auto z = algo(x, y);   // z = 16
    
    string xx = "7";
    string yy = "9";
    auto zz = algo(xx, yy);   // zz = "79"

也许拼接是有意进行的。不过更可能的是一种意外。而对减法进行同等的定义则会导致可接受类型的集合的非常不同。
这个 `Addable` 违反了加法应当可交换的数学法则：`a+b == b+a`。

##### 注解

给出有意义的语义的能力，在于定义真正的概念的特征，而不是仅给出语法约束。

##### 示例（使用 TS 概念语法）

    template<typename T>
    // 假定数值的运算符 +、-、* 和 / 都遵循常规的数学法则
    concept Number = has_plus<T>
                     && has_minus<T>
                     && has_multiply<T>
                     && has_divide<T>;
    
    template<Number N> auto algo(const N& a, const N& b)
    {
        // ...
        return a + b;
    }
    
    int x = 7;
    int y = 9;
    auto z = algo(x, y);   // z = 16
    
    string xx = "7";
    string yy = "9";
    auto zz = algo(xx, yy);   // 错误：string 不是 Number

##### 注解

带有多个操作的概念要远比单个操作的概念更少和类型发生意外匹配的机会。

##### 强制实施

* 对在其他 `concept` 的定义之外使用的但操作 `concept` 进行标记。
* 对表现为模拟单操作 `concept` 的 `enable_if` 的使用进行标记。


### <a name="Rt-complete"></a>T.21: 为概念提出一组完整的操作要求

##### 理由

易于理解。
提升互操作性。
帮助实现者和维护者。

##### 注解

这是对一般性规则[必须让概念有语义上的意义](#Rt-low)的一个专门的变体。

##### 示例，不好（采用 TS 版本的概念）

    template<typename T> concept Subtractable = requires(T a, T, b) { a-b; };

这个是没有语义作用的。
你至少还需要 `+` 来让 `-` 有意义和有用处。

完整集合的例子有

* `Arithmetic`: `+`, `-`, `*`, `/`, `+=`, `-=`, `*=`, `/=`
* `Comparable`: `<`, `>`, `<=`, `>=`, `==`, `!=`

##### 注解

无论我们是否使用了概念的直接语言支持，本条规则都适用。
这是一种一般性的设计规则，即便对于非模板也同样适用：

    class Minimal {
        // ...
    };
    
    bool operator==(const Minimal&, const Minimal&);
    bool operator<(const Minimal&, const Minimal&);
    
    Minimal operator+(const Minimal&, const Minimal&);
    // 没有其他运算符
    
    void f(const Minimal& x, const Minimal& y)
    {
        if (!(x == y)) { /* ... */ }    // OK
        if (x != y) { /* ... */ }       // 意外！错误
    
        while (!(x < y)) { /* ... */ }  // OK
        while (x >= y) { /* ... */ }    // 意外！错误
    
        x = x + y;          // OK
        x += y;             // 意外！错误
    }

这是最小化的设计，但会使用户遇到意外或受到限制。
可能它还会比较低效。

这条规则支持这样的观点：概念应当反映（数学上）协调的一组操作。

##### 示例

    class Convenient {
        // ...
    };
    
    bool operator==(const Convenient&, const Convenient&);
    bool operator<(const Convenient&, const Convenient&);
    // ... 其他比较运算符 ...
    
    Minimal operator+(const Convenient&, const Convenient&);
    // .. 其他算术运算符 ...
    
    void f(const Convenient& x, const Convenient& y)
    {
        if (!(x == y)) { /* ... */ }    // OK
        if (x != y) { /* ... */ }       // OK
    
        while (!(x < y)) { /* ... */ }  // OK
        while (x >= y) { /* ... */ }    // OK
    
        x = x + y;     // OK
        x += y;        // OK
    }

定义所有的运算符也许很麻烦，但并不困难。
理想情况下，语言应当默认提供比较运算符以支持这条规则。

##### 强制实施

* 如果类所支持的运算符是运算符集合的"奇异"子集，比如有 `==` 但没有 `!=` 或者有 `+` 但没有 `-`，就对其进行标记。
  确实，`std::string` 也是"奇异"的，但要修改它太晚了。


### <a name="Rt-axiom"></a>T.22: 为概念指明公理

##### 理由

有意义或有用的概念都有语义上的含义。
以非正式、半正式或正式的方式表达这些语义可以使概念对于读者更加可理解，而且对其进行表达的工作也能发现一些概念上的错误。
对语义的说明是一种强大的设计工具。

##### 示例（采用 TS 版本的概念）

    template<typename T>
        // 假定数值的运算符 +、-、* 和 / 都遵循常规的数学法则
        // axiom(T a, T b) { a + b == b + a; a - a == 0; a * (b + c) == a * b + a * c; /*...*/ }
        concept Number = requires(T a, T b) {
            {a + b} -> T;   // a + b 的结果可以转换为 T
            {a - b} -> T;
            {a * b} -> T;
            {a / b} -> T;
        }

##### 注解

这是一种数学意义上的公理：可以假定成立而无需证明。
通常公理是无法证明的，而即便可以证明，通常也超出了编译器的能力。
一条公理可能并非是通用的，不过模板作者可以假定其对于其所实际使用的所有输入均成立（类似于一种前条件）。

##### 注解

这种上下文中的公理都是布尔表达式。
参见 [Palo Alto TR](#S-references) 中的例子。
当前，C++ 并不支持公理（即便 ISO Concepts TS 也不支持），我们不得不长期用代码注释来给出它们。
语言支持一旦出现，公理前面的 `//` 就可以删掉了。

##### 注解

GSL 中的概念都具有恰当定义的语义；请参见 Palo Alto TR 和 Ranges TS。

##### 例外（采用 TS 版本的概念）

一个处于开发之中的新"概念"的早期版本，经常会仅仅定义了一组简单的约束而并没有恰当指定的语义。
为其寻找正确的语义可能是很费功夫和时间的。
不过不完整的约束集合仍然是很有用的：

    // 对于一般二叉树的平衡器
    template<typename Node> concept bool Balancer = requires(Node* p) {
        add_fixup(p);
        touch(p);
        detach(p);
    }

这样 `Balancer` 就必须为树的 `Node` 至少提供三个操作，
但我们还是无法指定详细的语义，因为一种新种类的平衡树可能需要更多的操作，
而在设计的早期阶段，很难确定把对于所有节点的精确的一般语义所确定下来。

不完整的或者没有恰当指定的语义的"概念"仍然是很有用的。
比如说，它可能允许在初始的试验之中进行某些检查。
不过，请不要把它当作是稳定的。
它的每个新的用例都可能导致不完整概念的改善。

##### 强制实施

* 在概念定义的代码注释中寻找单词"axiom"。

### <a name="Rt-refine"></a>T.23: 通过添加新的使用模式，从更一般情形的概念中区分出提炼后的概念

##### 理由

否则编译器是无法自动对它们进行区分的。

##### 示例（采用 TS 版本的概念）

    template<typename I>
    concept bool Input_iter = requires(I iter) { ++iter; };
    
    template<typename I>
    concept bool Fwd_iter = Input_iter<I> && requires(I iter) { iter++; }

编译器可以基于所要求的操作的集合（这里为前缀 `++`）来确定提炼关系。
这样做减少了这些类型的实现者的负担，
因为他们不再需要任何特殊的声明来"打入概念内部"了。
如果两个概念具有完全相同的要求的话，它们在逻辑上就是等价的（不存在提炼）。

##### 强制实施

* 对与已经出现的另一个概念具有完全相同的要求的概念进行标记（它们中不存在更精炼的概念）。
  为对它们进行区分，参见 [T.24](#Rt-tag)。

### <a name="Rt-tag"></a>T.24: 用标签类或特征类来区分仅在语义上存在差别的概念

##### 理由

要求相同的语法但具有不同语义的两个概念之间会造成歧义，除非程序员对它们进行区分。

##### 示例（采用 TS 版本的概念）

    template<typename I>    // 提供随机访问的迭代器
    concept bool RA_iter = ...;
    
    template<typename I>    // 提供对连续数据的随机访问的迭代器
    concept bool Contiguous_iter =
        RA_iter<I> && is_contiguous<I>::value;  // 使用 is_contiguous 特征

程序员（在程序库中）必须适当地定义（特征） `is_contiguous`。

把标签类包装到概念中可以得到这个方案的更简单的表达方式：

    template<typename I> concept Contiguous = is_contiguous<I>::value;
    
    template<typename I>
    concept bool Contiguous_iter = RA_iter<I> && Contiguous<I>;

程序员（在程序库中）必须适当地定义（特征） `is_contiguous`。

##### 注解

特征可以是特征类或者类型特征。
它们可以是用户定义的，或者标准库中的。
优先采用标准库中的特征。

##### 强制实施

* 编译器会将对相同的概念的有歧义的使用标记出来。
* 对相同的概念定义进行标记。

### <a name="Rt-not"></a>T.25: 避免互补性的约束

##### 理由

清晰性。可维护性。
用否定来表达的具有互补要求的函数是很脆弱的。

##### 示例（采用 TS 版本的概念）

最初，人们总会试图定义带有互补要求的函数：

    template<typename T>
        requires !C<T>    // 不好
    void f();
    
    template<typename T>
        requires C<T>
    void f();

这样会好得多：

    template<typename T>   // 通用模板
        void f();
    
    template<typename T>   // 用概念进行特化
        requires C<T>
    void f();

仅当 `C<T>` 无法满足时，编译器将会选择无约束的模板。
如果你并不想（或者无法）定义无约束版本的
`f()` 的话，你可以删掉它。

    template<typename T>
    void f() = delete;

编译器将会选取这个重载并给出一个适当的错误。

##### 注解

很不幸，互补约束在 `enable_if` 代码中很常见：

    template<typename T>
    enable_if<!C<T>, void>   // 不好
    f();
    
    template<typename T>
    enable_if<C<T>, void>
    f();


##### 注解

有时候会（不正确地）把对单个要求的互补约束当做是可以接受的。
不过，对于两个或更多的要求来说，所需要的定义的数量是按指数增长的（2，4，8，16，……）：

    C1<T> && C2<T>
    !C1<T> && C2<T>
    C1<T> && !C2<T>
    !C1<T> && !C2<T>

这样，犯错的机会也会倍增。

##### 强制实施

* 对带有 `C<T>` 和 `!C<T>` 约束的函数对进行标记。

### <a name="Rt-use"></a>T.26: 优先采用使用模式而不是简单的语法来定义概念

##### 理由

其定义更可读，而且更直接地对应于用户需要编写的代码。
其中同时兼顾了类型转换。你再不需要记住所有的类型特征的名字。

##### 示例（采用 TS 版本的概念）

你可能打算这样来定义概念 `Equality`：

    template<typename T> concept Equality = has_equal<T> && has_not_equal<T>;

显然，直接使用标准的 `EqualityComparable` 要更好而且更容易，
但是——只是一个例子——如果你不得不定义这样的概念的话，应当这样：

    template<typename T> concept Equality = requires(T a, T b) {
        bool == { a == b }
        bool == { a != b }
        // axiom { !(a == b) == (a != b) }
        // axiom { a = b; => a == b }  // => 的意思是"意味着"
    }

而不是定义两个无意义的概念 `has_equal` 和 `has_not_equal` 仅用于帮助 `Equality` 的定义。
"无意义"的意思是我们无法独立地指定 `has_equal` 的语义。

##### 强制实施

???

## <a name="SS-temp-interface"></a>模板接口

这些年以来，使用模板的编程一直忍受着模板的接口及其实现之间的
微弱区分性。
在引入概念之前，这种区分是没有直接的语言支持的。
不过，模板的接口是一个关键概念——是用户和实现者之间的一种契约——而且应当进行周密的设计。

### <a name="Rt-fo"></a>T.40: 使用函数对象向算法传递操作

##### 理由

函数对象比"普通"的函数指针能够向接口传递更多的信息。
一般来说，传递函数对象比传递函数指针能带来更好的性能。

##### 示例（采用 TS 版本的概念）

    bool greater(double x, double y) { return x > y; }
    sort(v, greater);                                    // 函数指针：可能较慢
    sort(v, [](double x, double y) { return x > y; });   // 函数对象
    sort(v, std::greater<>);                             // 函数对象
    
    bool greater_than_7(double x) { return x > 7; }
    auto x = find_if(v, greater_than_7);                 // 函数指针：不灵活
    auto y = find_if(v, [](double x) { return x > 7; }); // 函数对象：携带所需数据
    auto z = find_if(v, Greater_than<double>(7));        // 函数对象：携带所需数据

当然，也可以使用 `auto` 或（当可用时）概念来使这些函数通用化。例如：

    auto y1 = find_if(v, [](Ordered x) { return x > 7; }); // 要求一种有序类型
    auto z1 = find_if(v, [](auto x) { return x > 7; });    // 期望类型带有 >

##### 注解

Lambda 会生成函数对象。

##### 注解

性能表现依赖于编译器和优化器技术。

##### 强制实施

* 标记以函数指针作为模板参数。
* 标记将函数指针作为模板的参数进行传递（存在误报风险）。


### <a name="Rt-essential"></a>T.41: 在模板的概念上仅提出基本的性质要求

##### 理由

保持接口的简单和稳定。

##### 示例（采用 TS 版本的概念）

考虑一个带有（过度简化的）简单调试支持的 `sort`：

    void sort(Sortable& s)  // 对序列 s 进行排序
    {
        if (debug) cerr << "enter sort( " << s <<  ")\n";
        // ...
        if (debug) cerr << "exit sort( " << s <<  ")\n";
    }

是否该把它重写为：

    template<Sortable S>
        requires Streamable<S>
    void sort(S& s)  // 对序列 s 进行排序
    {
        if (debug) cerr << "enter sort( " << s <<  ")\n";
        // ...
        if (debug) cerr << "exit sort( " << s <<  ")\n";
    }

毕竟，`Sortable` 里面并没有要求任何 `iostream` 支持。
而另一方面，在排序的基本概念中也没有任何东西是有关于调试的。

##### 注解

如果我们要求把用到的所有操作都在要求部分中列出的话，接口就会变得不稳定：
每当我们改动了调试设施，使用情况数据收集，测试支持，错误报告等等，
模板的定义就都得进行改动，而模板的所有使用方都不得不进行重新编译。
这样做很是累赘，而且在某些环境下是不可行的。

与之相反，如果我们在实现中使用了某个并未被概念检查提供保证的操作的话，
我们可能会遇到一个延后的编译时错误。

通过不对模板参数的那些不被认为是基本的性质进行概念检查，
我们把其检查推迟到了进行实例化的时候。
我们认为这是值得做出的折衷。

注意，使用非局部的，非待决的名字（比如 `debug` 和 `cerr`），同样会引入可能导致"神秘的"错误的某些上下文依赖。

##### 注解

可能很难确定一个类型的哪些性质是基本的，而哪些不是。

##### 强制实施

???

### <a name="Rt-alias"></a>T.42: 使用模板别名来简化写法并隐藏实现细节

##### 理由

提升可读性。
隐藏实现。
注意，模板别名取代了许多用于计算类型的特征。
它们也可以用于封装一个特征。

##### 示例

    template<typename T, size_t N>
    class Matrix {
        // ...
        using Iterator = typename std::vector<T>::iterator;
        // ...
    };

这免除了 `Matrix` 的用户必须了解其元素是存储于 `vector` 之中，而且也免除了用户重复书写 `typename std::vector<T>::`。

##### 示例

    template<typename T>
    void user(T& c)
    {
        // ...
        typename container_traits<T>::value_type x; // 不好，啰嗦
        // ...
    }
    
    template<typename T>
    using Value_type = typename container_traits<T>::value_type;


这免除了 `Value_type` 的用户必须了解用于实现 `value_type` 的技术。

    template<typename T>
    void user2(T& c)
    {
        // ...
        Value_type<T> x;
        // ...
    }

##### 注解

一种简洁的常用说法是："包装特征！"

##### 强制实施

* 将 `using` 声明之外用于消除歧义的 `typename` 进行标记。
* ???

### <a name="Rt-using"></a>T.43: 优先使用 `using` 而不是 `typedef` 来定义别名

##### 理由

提升可读性：使用 `using` 时，新名字在前面，而不是被嵌在声明中的什么地方。
通用性：`using` 可用于模板别名，而 `typedef` 无法轻易作为模板。
一致性：`using` 在语法上和 `auto` 相似。

##### 示例

    typedef int (*PFI)(int);   // OK, 但很别扭
    
    using PFI2 = int (*)(int);   // OK, 更好
    
    template<typename T>
    typedef int (*PFT)(T);      // 错误
    
    template<typename T>
    using PFT2 = int (*)(T);   // OK

##### 强制实施

* 标记 `typedef` 的使用。不过这样会出现大量的"命中" :-(

### <a name="Rt-deduce"></a>T.44: （如果可行）使用函数模板来对类模板的参数类型进行推断

##### 理由

显式写出模板参数类型既麻烦又无必要。

##### 示例

    tuple<int, string, double> t1 = {1, "Hamlet", 3.14};   // 明确类型
    auto t2 = make_tuple(1, "Ophelia"s, 3.14);         // 更好；推断类型

注意这里用 `s` 后缀来确保字符串是 `std::string` 而不是 C 风格字符串。

##### 注解

既然你可以轻易写出 `make_T` 函数，编译器也可以。以后 `make_T` 函数将会变得多余。

##### 例外

有时候没办法对模板参数进行推断，而有时候你则想要明确指定参数：

    vector<double> v = { 1, 2, 3, 7.9, 15.99 };
    list<Record*> lst;

##### 注解

注意，C++17 强允许模板参数直接从构造函数参数进行推断，而使这条规则变得多余：
[构造函数的模板形参推断(Rev. 3)](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2016/p0091r1.html)。
例如：

    tuple t1 = {1, "Hamlet"s, 3.14}; // 推断为：tuple<int, string, double>

##### 强制实施

当显式指定的类型与所使用的类型精确匹配时进行标记。

### <a name="Rt-regular"></a>T.46: 要求模板参数至少为 `Regular` 或者 `SemiRegular`

##### 理由

可读性。
避免意外和错误。
大多数用法都支持这样做。

##### 示例

    class X {
    public:
        explicit X(int);
        X(const X&);            // 复制
        X operator=(const X&);
        X(X&&) noexcept;                 // 移动
        X& operator=(X&&) noexcept;
        ~X();
        // ... 没有别的构造函数了 ...
    
        // ...
    };
    
    X x {1};    // 没问题
    X y = x;      // 没问题
    std::vector<X> v(10); // 错误: 没有默认构造函数

##### 注解

`SemiRegular` 要求可以默认构造。

##### 强制实施

* 对并非至少为 `SemiRegular` 的类型进行标记。

### <a name="Rt-visible"></a>T.47: 避免用常用名字命名高度可见的无约束模板

##### 示例

无约束的模板参数和任何东西都能完全匹配，因此这样的模板相对于需要少量转换的更特定类型来说可能更优先。
而当使用 ADL 时，这一点将会更加麻烦和危险。
而常用的名字则会让这种问题更易于出现。

##### 示例

    namespace Bad {
        struct S { int m; };
        template<typename T1, typename T2>
        bool operator==(T1, T2) { cout << "Bad\n"; return true; }
    }
    
    namespace T0 {
        bool operator==(int, Bad::S) { cout << "T0\n"; return true; }  // 与 int 比较
    
        void test()
        {
            Bad::S bad{ 1 };
            vector<int> v(10);
            bool b = 1 == bad;
            bool b2 = v.size() == bad;
        }
    }

这将会打印出 `T0` 和 `Bad`。

这里 `Bad` 中的 `==` 有意设计为造成问题，不过你是否在真实代码中发现过这个问题呢？
问题在于 `v.size()` 返回的是 `unsigned` 整数，因此需要进行转换才能调用局部的 `==`；
而 `Bad` 中的 `=` 则不需要任何转换。
实际的类型，比如标准库的迭代器，也可以被弄成类似的反社会倾向。

##### 注解

如果在相同的命名空间中定义了一个无约束模板类型，
这个无约束模板是可以通过 ADL 所找到的（如例子中所发生的一样）。
就是说，它是高度可见的。

##### 注解

这条规则应当是没有必要的，不过委员会在是否将无约束模板从 ADL 中排除上无法达成一致。

不幸的是，这会导致大量的误报；标准库也大量地违反了这条规则，它将许多无约束模板和类型都放入了单一的命名空间 `std` 之中。


##### 强制实施

如果定义模板的命名空间中同样定义了具体的类型，就对其进行标记（可能在我们有概念支持之前都是不可行的）。


### <a name="Rt-concept-def"></a>T.48: 如果你的编译器不支持概念的话，可以用 `enable_if` 来模拟

##### 理由

因为这是我们没有直接的概念支持时能够做到的最好的方式。
`enable_if` 可被用于有条件地定义函数，以及用于在一组函数中进行选择。

##### 示例

    template <typename T>
    enable_if_t<is_integral_v<T>>
    f(T v)
    {
        // ...
    }
    
    // Equivalent to:
    template <Integral T>
    void f(T v)
    {
        // ...
    }

##### 注解

请当心[互补约束](# T.25)。
当用 `enable_if` 来模拟概念重载时，有时候会迫使我们使用这种易错的设计技巧。

##### 强制实施

???

### <a name="Rt-erasure"></a>T.49: 尽可能避免类型擦除

##### 理由

类型擦除通过在一个独立的编译边界之后隐藏类型信息而招致一层额外的间接。

##### 示例

    ???

**例外**: 有时候类型擦除是合适的，比如 `std::function`。

##### 强制实施

???


##### 注解


## <a name="SS-temp-def"></a>T.def: 模板定义

模板的定义式（无论是类还是函数）都可能包含任意的代码，因而只有相当范围的 C++ 编程技术才能覆盖这个主题。
不过，这个部分所关注的是特定于模板的实现的问题。
特别地说，这里关注的是模板定义式对其上下文之间的依赖。

### <a name="Rt-depend"></a>T.60: 最小化模板的上下文依赖性

##### 理由

便于理解。
最小化源于意外依赖的错误。
便于创建工具。

##### 示例

    template<typename C>
    void sort(C& c)
    {
        std::sort(begin(c), end(c)); // 必要并且有意义的依赖
    }
    
    template<typename Iter>
    Iter algo(Iter first, Iter last) {
        for (; first != last; ++first) {
            auto x = sqrt(*first); // 潜在的意外依赖：哪个 sqrt()？
            helper(first, x);      // 潜在的意外依赖：
                                   // helper 是基于 first 和 x 所选择的
            TT var = 7;            // 潜在的意外依赖：哪个 TT？
        }
    }

##### 注解

通常模板是放在头文件中的，因此相对于 `.cpp` 文件之中的函数来说，其上下文依赖更加会受到 `#include` 的顺序依赖的威胁。

##### 注解

让模板仅对其参数进行操作是一种把依赖减到最少的方式，但通常这是很难做到的。
比如说，一个算法通常会使用其他的算法，并且调用一些并非仅在参数上做动作的一些操作。
而且别再让我们从宏开始干活！

**参见**：[T.69](#Rt-customization)。

##### 强制实施

??? 很麻烦

### <a name="Rt-scary"></a>T.61: 请勿对成员进行过度参数化（恐怖）

##### 理由

不依赖于模板参数的成员，除非给定某个特定的模板参数，否则也是无法使用的。
这样会限制其使用，而且通常会增加代码大小。

##### 示例，不好

    template<typename T, typename A = std::allocator{}>
        // requires Regular<T> && Allocator<A>
    class List {
    public:
        struct Link {   // 并未依赖于 A
            T elem;
            T* pre;
            T* suc;
        };
    
        using iterator = Link*;
    
        iterator first() const { return head; }
    
        // ...
    private:
        Link* head;
    };
    
    List<int> lst1;
    List<int, My_allocator> lst2;

这看起来没什么问题，但现在 `Link` 形成依赖于分配器（尽管它不使用分配器）。 这迫使冗余的实例化在某些现实场景中可能造成出奇的高的成本。
通常，解决方案是使用自己的最小模板参数集使嵌套类非局部化。

    template<typename T>
    struct Link {
        T elem;
        T* pre;
        T* suc;
    };
    
    template<typename T, typename A = std::allocator{}>
        // requires Regular<T> && Allocator<A>
    class List2 {
    public:
        using iterator = Link<T>*;
    
        iterator first() const { return head; }
    
        // ...
    private:
        List* head;
    };
    
    List<int> lst1;
    List<int, My_allocator> lst2;

人们发现 `Link` 不再隐藏在列表中很可怕，所以我们命名这个技术为 [SCARY]（http://www.open-std.org/jtc1/sc22/WG21/docs/papers/2009/n2911.pdf）。
引自该学术论文："首字母缩略词 SCARY 描述了看似错误的赋值和初始化（受冲突的通用参数的约束），
但实际上使用了正确的实现（由于最小化的依赖而不受冲突的约束）。"

##### 强制实施

* 对并未依赖于全部模板参数的成员类型进行标记。
* 对并未依赖于全部模板参数的成员函数进行标记。

### <a name="Rt-nondependent"></a>T.62: 将无依赖的类模板成员置于一个非模板基类之中

##### 理由

可以在使用基类成员时不需要指定模板参数，也不需要模板实例化。

##### 示例

    template<typename T>
    class Foo {
    public:
        enum { v1, v2 };
        // ...
    };

???

    struct Foo_base {
        enum { v1, v2 };
        // ...
    };
    
    template<typename T>
    class Foo : public Foo_base {
    public:
        // ...
    };

##### 注解

这条规则的更一般化的版本是，
"如果模板类的成员依赖于 M 个模板参数中的 N 个，就将它置于只有 N 个参数的基类之中。"
当 N == 1 时，可以如同 [T.61](#Rt-scary) 一样在一个基类和其外围作用域中的一个类之间进行选择。

??? 常量的情况如何？类的静态成员呢？

##### 强制实施

* 标记 ???

### <a name="Rt-specialization"></a>T.64: 用特化来提供类模板的其他实现

##### 理由

模板定义了通用接口。
特化是一种为这个接口提供替代实现的强大机制。

##### 示例

    ??? 字符串的特化 (==)
    
    ??? 表示特化？

##### 注解

???

##### 强制实施

???

### <a name="Rt-tag-dispatch"></a>T.65: 用标签分派来提供函数的其他实现

##### 理由

* 模板定义了通用接口。
* 标签派发允许我们基于参数类型的特定性质选择不同的实现。
* 性能。

##### 示例

这是 `std::copy` 的一个简化版本（忽略了非连续序列的可能性）

    struct pod_tag {};
    struct non_pod_tag {};
    
    template<class T> struct copy_trait { using tag = non_pod_tag; };   // T 不是"朴素数据"
    
    template<> struct copy_trait<int> { using tag = pod_tag; };         // int 是"朴素数据"
    
    template<class Iter>
    Out copy_helper(Iter first, Iter last, Iter out, pod_tag)
    {
        // 使用 memmove
    }
    
    template<class Iter>
    Out copy_helper(Iter first, Iter last, Iter out, non_pod_tag)
    {
        // 使用调用复制构造函数的循环
    }
    
    template<class Itert>
    Out copy(Iter first, Iter last, Iter out)
    {
        return copy_helper(first, last, out, typename copy_trait<Iter>::tag{})
    }
    
    void use(vector<int>& vi, vector<int>& vi2, vector<string>& vs, vector<string>& vs2)
    {
        copy(vi.begin(), vi.end(), vi2.begin()); // 使用 memmove
        copy(vs.begin(), vs.end(), vs2.begin()); // 使用调用复制构造函数的循环
    }

这是一种进行编译时算法选择的通用且有力的技巧。

##### 注解

当可以广泛使用 `concept` 之后，这样的替代实现就可以直接进行区分了：

    template<class Iter>
        requires Pod<Value_type<iter>>
    Out copy_helper(In, first, In last, Out out)
    {
        // 使用 memmove
    }
    
    template<class Iter>
    Out copy_helper(In, first, In last, Out out)
    {
        // 使用调用复制构造函数的循环
    }

##### 强制实施

???


### <a name="Rt-specialization2"></a>T.67: 用特化来提供不规则类型的其他实现

##### 理由

 ???

##### 示例

    ???

##### 强制实施

???

### <a name="Rt-cast"></a>T.68: 在模板中用 `{}` 而不是 `()` 以避免歧义

##### 理由

`()` 会带来文法歧义。

##### 示例

    template<typename T, typename U>
    void f(T t, U u)
    {
        T v1(x);    // v1 是函数还是变量？
        T v2 {x};   // 变量
        auto x = T(u);  // 构造还是强制转换？
    }
    
    f(1, "asdf"); // 不好：从 const char* 强制转换为 int

##### 强制实施

* 标记 `()` 初始化式。
* 标记函数风格的强制转换。


### <a name="Rt-customization"></a>T.69: 在模板中，请勿进行未限定的非成员函数调用，除非有意将之作为定制点

##### 理由

* 仅提供预计之内的灵活性。
* 避免源于意外的环境改变的威胁。

##### 示例

主要有三种方法使调用代码对模板进行定制化。

    template<class T>
        // 调用成员函数
    void test1(T t)
    {
        t.f();    // 要求 T 提供 f()
    }
    
    template<class T>
    void test2(T t)
        // 不带限定地调用非成员函数
    {
        f(t);  // 要求 f(/*T*/) 在调用方的作用域或者 T 的命名空间中可用
    }
    
    template<class T>
    void test3(T t)
        // 调用一个"特征"
    {
        test_traits<T>::f(t); // 要求定制化 test_traits<>
                              // 以获得非默认的函数和类型
    }

特征通常是用以计算一个类型的类型别名，
用以计算一个值的 `constexpr` 函数，
或者针对用户的类型进行特化的传统的特征模板。

##### 注解

当你打算为依赖于某个模板类型参数的值 `t` 调用自己的辅助函数 `helper(t)` 时，
请将函数放入一个 `::detail` 命名空间中，并把调用限定为 `detail::helper(t);`。
无限定的调用将成为一个定制点，它将会调用处于 `t` 的类型所在命名空间中的任何 `helper` 函数；
这可能会导致诸如[意外地调用了无约束函数模板](#Rt-unconstrained-adl)这样的问题。


##### 强制实施

* 在模板中，如果非成员函数的无限定调用传递了具有依赖类型的变量，而在该模板的命名空间中存在相同名字的非成员函数，则对其进行标记。


## <a name="SS-temp-hier"></a>T.temp-hier: 模板和类型层次规则：

模板是 C++ 为泛型编程提供支持的基石，而类层次则是 C++ 为面向对象编程提供
支持的基石。
这两种语言机制可以有效地组合起来，但必须避免一些设计上的陷阱。

### <a name="Rt-hier"></a>T.80: 请勿不成熟地对类层次进行模板化

##### 理由

使一个带有许多函数，尤其是有许多虚函数的类层次进行模板化，会导致代码膨胀。

##### 示例，不好

    template<typename T>
    struct Container {         // 这是一个接口
        virtual T* get(int i);
        virtual T* first();
        virtual T* next();
        virtual void sort();
    };
    
    template<typename T>
    class Vector : public Container<T> {
    public:
        // ...
    };
    
    Vector<int> vi;
    Vector<string> vs;

这可能是一个比较笨拙的把 `sort` 定义为容器的成员函数的做法，不过这样做并不鲜见，而且它是一个展示不应当做的事情的好例子。

在这之中，编译器不知道 `Vector<int>::sort()` 是不是会被调用，因此它必须为之生成代码。
`Vector<string>::sort()` 也与此相似。
除非这两个函数被调用，否则这就是代码爆炸。
不难想象当一个带有几十个成员函数和几十个派生类的类层次被大量实例化时会怎么样。

##### 注解

许多情况下都可以通过不为基类进行参数化而提供一个稳定的接口；
参见["稳定的基类"](#Rt-abi)和 [OO 与 GP](#Rt-generic-oo)。

##### 强制实施

* 对依赖于模板参数的虚函数进行标记。 ??? 误报

### <a name="Rt-array"></a>T.81: 请勿混合类层次和数组

##### 理由

派生类的数组可以隐式地"衰退"称指向基类的指针，并带来潜在的灾难性后果。

##### 示例

假定 `Apple` 和 `Pear` 是两种 `Fruit`。

    void maul(Fruit* p)
    {
        *p = Pear{};     // 把一个 Pear 放入 *p
        p[1] = Pear{};   // 把一个 Pear 放入 p[1]
    }
    
    Apple aa [] = { an_apple, another_apple };   // aa 包含的是 Apple （显然！）
    
    maul(aa);
    Apple& a0 = &aa[0];   // 是 Pear 吗？
    Apple& a1 = &aa[1];   // 是 Pear 吗？

`aa[0]` 可能会变为 `Pear`（并且没进行过强制转换！）。
当 `sizeof(Apple) != sizeof(Pear)` 时，对 `aa[1]` 的访问就是并未跟数组中的对象的适当起始位置进行对齐的。
这里出现了类型违例，以及很可能出现的内存损坏。
决不要写这样的代码。

注意，`maul()` 违反了 [`T*` 应指向独立对象的规则](#Rf-ptr)。

**替代方案**: 使用适当的（模板化）容器：

    void maul2(Fruit* p)
    {
        *p = Pear{};   // 把一个 Pear 放入 *p
    }
    
    vector<Apple> va = { an_apple, another_apple };   // va 包含的是 Apple （显然！）
    
    maul2(va);       // 错误: 无法把 vector<Apple> 转换为 Fruit*
    maul2(&va[0]);   // 这是你明确要做的
    
    Apple& a0 = &va[0];   // 是 Pear 吗？

注意，`maul2()` 中的赋值违反了[避免发生切片的规则](#Res-slice)。

##### 强制实施

* 对这种恐怖的东西进行检测！

### <a name="Rt-linear"></a>T.82: 当不想要虚函数时，可以将类层次线性化

##### 理由

 ???

##### 示例

    ???

##### 强制实施

???

### <a name="Rt-virtual"></a>T.83: 请勿声明虚的成员函数模板

##### 理由

C++ 是不支持这样做的。
如果支持的话，就只能等到连接时才能生成 VTBL 了。
而且一般来说，各个实现还要搞定动态连接的问题。

##### 示例，请勿如此

    class Shape {
        // ...
        template<class T>
        virtual bool intersect(T* p);   // 错误：模板不能为虚
    };

##### 注解

我们保留这条规则是因为人们总是问这个问题。

##### 替代方案

双派发或访问器模式，或者计算出所要调用的函数。

##### 强制实施

编译器会处理这个问题。

### <a name="Rt-abi"></a>T.84: 使用非模板的核心实现来提供 ABI 稳定的接口

##### 理由

提升代码的稳定性。
避免代码爆炸。

##### 示例

这个应当是基类：

    struct Link_base {   // 稳定
        Link_base* suc;
        Link_base* pre;
    };
    
    template<typename T>   // 模板化的包装带来了类型安全性
    struct Link : Link_base {
        T val;
    };
    
    struct List_base {
        Link_base* first;   // 第一个元素（如果有）
        int sz;             // 元素数量
        void add_front(Link_base* p);
        // ...
    };
    
    template<typename T>
    class List : List_base {
    public:
        void put_front(const T& e) { add_front(new Link<T>{e}); }   // 隐式强制转换为 Link_base
        T& front() { static_cast<Link<T>*>(first).val; }   // 显式强制转换回 Link<T>
        // ...
    };
    
    List<int> li;
    List<string> ls;

这样的话就只有一份用于对 `List` 的元素进行入链和解链的操作的代码了。
而类 `Link` 和 `List` 除了进行类型操作之外什么也没做。

除了使用一个独立的"base"类型外，另一种常用的技巧是对 `void` 或 `void*` 进行特化，并让针对 `T` 的通用模板成为在从或向 `void` 的核心实现进行强制转换的一层类型安全封装。

**替代方案**: 使用一个 [PImpl](#Ri-pimpl) 实现。

##### 强制实施

???

## <a name="SS-variadic"></a>T.var: 变参模板规则

???

### <a name="Rt-variadic"></a>T.100: 当需要可以接受可变数量的多种类型参数的函数时，使用变参模板

##### 理由

变参模板是做到这点的最通用的机制，而且既高效又类型安全。请不要使用 C 的变参。

##### 示例

    ??? printf

##### 强制实施

* 对用户代码中 `va_arg` 的使用进行标记。

### <a name="Rt-variadic-pass"></a>T.101: ??? 如何向变参模板传递参数 ???

##### 理由

 ???

##### 示例

    ??? 当心仅能移动参数和引用参数

##### 强制实施

???

### <a name="Rt-variadic-process"></a>T.102: 如何处理变参模板的参数

##### 理由

 ???

##### 示例

    ??? 转发参数，类型检查，引用

##### 强制实施

???

### <a name="Rt-variadic-not"></a>T.103: 请勿对同质参数列表使用变参模板

##### 理由

存在更加正规的给出同质序列的方式，比如使用 `initializer_list`。

##### 示例

    ???

##### 强制实施

???

## <a name="SS-meta"></a>T.meta: 模板元编程（TMP）

模板提供了一种编译期编程的通用机制。

元编程，是其中至少一项输入或者一项输出是类型的编程。
模板提供了编译期的图灵完备（除了内存消耗外）的鸭子类型系统。
其语法和所需技巧都相当可怕。

### <a name="Rt-metameta"></a>T.120: 仅当确实需要时才使用模板元编程

##### 理由

模板元编程很难做对，它会拖慢编译速度，而且通常很难维护。
不过，现实世界有些例子中模板元编程提供了比其他专家级的汇编代码替代方案还要更好的性能。
而且，也存在现实世界的例子用模板元编程做到比运行时代码更好地表达基本设计意图的情况。
例如，当确实需要在编译期进行 AST 操作时，（比如说对矩阵操作进行可选的折叠），C++ 中可能没有其他的实现方式。

##### 示例，不好

    ???

##### 示例，不好

    enable_if

请使用概念来替代它。不过请参见[如何在没有语言支持时模拟概念](#Rt-emulate)。

##### 示例

    ??? 好例子

**替代方案**: 如果结果是一个值而不是类型，请使用 [`constexpr` 函数](#Rt-fct)。

##### 注解

当你觉得需要把模板元编程代码隐藏到宏之中时，你可能已经跑得太远了。

### <a name="Rt-emulate"></a>T.121: 模板元编程主要用于模拟概念机制

##### 理由

在概念可以广泛使用之前，我们需要用 TMP 来模拟它。
对概念给出要求的用例（比如基于概念进行重载）是 TMP 的最常见（而且最简单）的用法。

##### 示例

    template<typename Iter>
        /*requires*/ enable_if<random_access_iterator<Iter>, void>
    advance(Iter p, int n) { p += n; }
    
    template<typename Iter>
        /*requires*/ enable_if<forward_iterator<Iter>, void>
    advance(Iter p, int n) { assert(n >= 0); while (n--) ++p;}

##### 注解

这种代码使用概念时将更加简单：

    void advance(RandomAccessIterator p, int n) { p += n; }
    
    void advance(ForwardIterator p, int n) { assert(n >= 0); while (n--) ++p;}

##### 强制实施

???

### <a name="Rt-tmp"></a>T.122: 用模板（通常为模板别名）来在编译期进行类型运算

##### 理由

模板元编程是在编译期进行类型生成的受到直接支持和部分正规化的唯一方式。

##### 注解

"特征（Trait）"技术基本上在计算类型方面被模板别名所代替，而在计算值方面则被 `constexpr` 函数所代替。

##### 示例

    ??? 大型对象 / 小型对象的优化

##### 强制实施

???

### <a name="Rt-fct"></a>T.123: 用 `constexpr` 函数来在编译期进行值运算

##### 理由

函数是用于表达计算一个值的最显然和传统的方式。
通常 `constexpr` 函数都比其他的替代方式具有更少的编译期开销。

##### 注解

"特征（Trait）"技术基本上在计算类型方面被模板别名所代替，而在计算值方面则被 `constexpr` 函数所代替。

##### 示例

    template<typename T>
        // requires Number<T>
    constexpr T pow(T v, int n)   // 幂/指数
    {
        T res = 1;
        while (n--) res *= v;
        return res;
    }
    
    constexpr auto f7 = pow(pi, 7);

##### 强制实施

* 对产生值的模板元程序进行标记。它们应当被替换成 `constexpr` 函数。

### <a name="Rt-std-tmp"></a>T.124: 优先使用标准库的模板元编程设施

##### 理由

标准中所定义的设施，诸如 `conditional`，`enable_if`，以及 `tuple` 等，是可移植的，可以假定为大家所了解。

##### 示例

    ???

##### 强制实施

???

### <a name="Rt-lib"></a>T.125: 当需要标准库之外的模板元编程设施时，使用某个现存程序库

##### 理由

要搞出高级的 TMP 设施是很难的，而使用一个库则可让你进入某个（有希望收到支持的）社区。
只有当确实不得不编写自己的"高级 TMP 支持"时，才应当这样做。

##### 示例

    ???

##### 强制实施

???

## <a name="SS-temp-other"></a>其他模板规则

### <a name="Rt-name"></a>T.140: 对所有的可能会被重用的操作命名

##### 理由

文档，可读性，重用机会。

##### 示例

    struct Rec {
        string name;
        string addr;
        int id;         // 唯一标识符
    };
    
    bool same(const Rec& a, const Rec& b)
    {
        return a.id == b.id;
    }
    
    vector<Rec*> find_id(const string& name);    // 寻找"name"的所有记录
    
    auto x = find_if(vr.begin(), vr.end(),
        [&](Rec& r) {
            if (r.name.size() != n.size()) return false; // 要比较的名字都在 n 里
            for (int i = 0; i < r.name.size(); ++i)
                if (tolower(r.name[i]) != tolower(n[i])) return false;
            return true;
        }
    );

这里蕴含着一个有用的函数（大小写无关的字符串比较），因为它通常会导致 lambda 参数变大。

    bool compare_insensitive(const string& a, const string& b)
    {
        if (a.size() != b.size()) return false;
        for (int i = 0; i < a.size(); ++i) if (tolower(a[i]) != tolower(b[i])) return false;
        return true;
    }
    
    auto x = find_if(vr.begin(), vr.end(),
        [&](Rec& r) { compare_insensitive(r.name, n); }
    );

或者也可以这样（如果你更希望避免对 n 进行隐式的名字绑定的话）：

    auto cmp_to_n = [&n](const string& a) { return compare_insensitive(a, n); };
    
    auto x = find_if(vr.begin(), vr.end(),
        [](const Rec& r) { return cmp_to_n(r.name); }
    );

##### 注解

无论是函数，lambda，还是运算符。

##### 例外

* Lambda 逻辑上仅在局部作用域中使用，比如作为 `for_each` 和类似的控制流算法的参数等。
* Lambda 用作[初始化式](#???)

##### 强制实施

* 【困难】 标记出相似的 lambda。
* ???

### <a name="Rt-lambda"></a>T.141: 当仅在一个地方需要一个简单的函数对象时，使用无名的 lambda

##### 理由

这样能够使代码精简并比其他方式提供更好的局部性。

##### 示例

    auto earlyUsersEnd = std::remove_if(users.begin(), users.end(),
                                        [](const User &a) { return a.id > 100; });


##### 例外

为 lambda 命名是有用的，即便它可能只会一次性使用也是如此。

##### 强制实施

* 寻找相同和几乎相同的 lambda（以便将它们替换为具名的函数或者具名的 lambda）。

### <a name="Rt-var"></a>T.142?: 使用模板变量以简化写法

##### 理由

改善可读性。

##### 示例

    ???

##### 强制实施

???

### <a name="Rt-nongeneric"></a>T.143: 请勿编写并非有意非泛型的代码

##### 理由

一般性。可重用性。请勿无必要地陷入技术细节之中；请使用最广泛可用的设施。

##### 示例

用 `!=` 而不是 `<` 来比较迭代器；`!=` 可以在更多对象上正常工作，因为它并未蕴含有序性。

    for (auto i = first; i < last; ++i) {   // 通用性较差
        // ...
    }
    
    for (auto i = first; i != last; ++i) {   // 好; 通用性较强
        // ...
    }

当然，范围式 `for` 在符合需求的时候当然是更好的选择。

##### 示例

使用能够提供所需功能的最接近基类的类。

    class Base {
    public:
        Bar f();
        Bar g();
    };
    
    class Derived1 : public Base {
    public:
        Bar h();
    };
    
    class Derived2 : public Dase {
    public:
        Bar j();
    };
    
    // 不好，除非确实有特别的原因来将之仅限制为 Derived1 对象
    void my_func(Derived1& param)
    {
        use(param.f());
        use(param.g());
    }
    
    // 好，仅使用 Base 的接口，且保证了这个类型
    void my_func(Base& param)
    {
        use(param.f());
        use(param.g());
    }

##### 强制实施

* 对使用 `<` 而不是 `!=` 的迭代器比较进行标记。
* 当存在 `x.empty()` 或 `x.is_empty()` 时，对 `x.size() == 0` 进行标记。`empty()` 比 `size()` 能够对于更多的容器工作，因为某些容器是不知道自己的大小的，甚至概念上就是大小无界的。
* 如果函数接受指向更加派生的类型的指针或引用，但仅使用了在某个基类中所声明的函数，则对其进行标记。

### <a name="Rt-specialize-function"></a>T.144: 请勿特化函数模板

##### 理由

根据语言规则，函数模板是无法被部分特化的。函数模板可以被完全特化，不过你基本上需要的都是进行重载而不是特化——因为函数模板特化并不参与重载，它们的行为和你想要的可能是不同的。少数情况下，应当通过你可以进行适当特化的类模板来进行真正的特化。

##### 示例

    ???

**例外**: 当确实有特化函数模板的恰当理由时，请只编写一个函数模板，并使它委派给一个类模板，然后对这个类模板进行特化（这提供了编写部分特化的能力）。

##### 强制实施

* 标记出所有的函数模板特化。代之以函数重载。


### <a name="Rt-check-class"></a>T.150: 用 `static_assert` 来检查类是否与概念相符

##### 理由

当你打算使一个类符合某个概念时，应该提早进行验证以减少麻烦。

##### 示例

    class X {
    public:
        X() = delete;
        X(const X&) = default;
        X(X&&) = default;
        X& operator=(const X&) = default;
        // ...
    };

在别的地方，也许是某个实现文件中，可以让编译器来检查 `X` 的所需各项性质：

    static_assert(Default_constructible<X>);    // 错误: X 没有默认构造函数
    static_assert(Copyable<X>);                 // 错误: 忘记定义 X 的移动构造函数了


##### 强制实施

不可行。

# <a name="S-cpl"></a>CPL: C 风格的编程

C 和 C++ 是联系很紧密的两门语言。
它们都是源于 1978 年的"经典 C"语言的，且从此之后就在 ISO 标准委员会中进行演化。
为了让它们保持兼容，我们做过许多努力，但它们各自都并非是对方的子集。

C 规则概览：

* [CPL.1: 优先使用 C++ 而不是 C](#Rcpl-C)
* [CPL.2: 当一定要用 C 时，应使用 C 和 C++ 的公共子集，并将 C 代码以 C++ 来编译](#Rcpl-subset)
* [CPL.3: 当一定要用 C 来作为接口时，应在使用这些接口的调用方代码中使用 C++](#Rcpl-interface)

### <a name="Rcpl-C"></a>CPL.1: 优先使用 C++ 而不是 C

##### 理由

C++ 提供更好的类型检查和更多的语法支持。
它能为高层的编程提供更好的支持，而且通常会产生更快速的代码。

##### 示例

    char ch = 7;
    void* pv = &ch;
    int* pi = pv;   // 非 C++
    *pi = 999;      // 覆盖了 &ch 附近的 sizeof(int) 个字节

针对在 C 中从 `void*` 或向它进行的隐式强制转换的相关规则比较麻烦而且并未强制实施。
特别是，这个例子违反了禁止把类型转换为具有更严格对齐的类型的规则。

##### 强制实施

使用 C++ 编译器。

### <a name="Rcpl-subset"></a>CPL.2: 当一定要用 C 时，应使用 C 和 C++ 的公共子集，并将 C 代码以 C++ 来编译

##### 理由

它们的子集语言，C 和 C++ 编译器都可以编译，而当作为 C++ 编译时，比"纯 C" 进行更好的类型检查。

##### 示例

    int* p1 = malloc(10 * sizeof(int));                      // 非 C++
    int* p2 = static_cast<int*>(malloc(10 * sizeof(int)));   // 非 C, C 风格的 C++
    int* p3 = new int[10];                                   // 非 C
    int* p4 = (int*) malloc(10 * sizeof(int));               // C 和 C++ 均可

##### 强制实施

* 当使用某种将代码作为 C 来编译的构建模式时进行标记。

  * C++ 将会确保代码是合法的 C++ 代码，除非使用了 C 扩展的编译器选项。

### <a name="Rcpl-interface"></a>CPL.3: 当一定要用 C 来作为接口时，应在使用这些接口的代码中使用 C++

##### 理由

C++ 比 C 的表达能力更强，而且为许多种类的编程都提供了更好的支持。

##### 示例

例如，为使用第三方 C 程序库或者 C 系统接口，可以使用 C 和 C++ 的公共子集来定义其底层接口，以获得更好的类型检查。
尽可能将底层接口封装到一个遵循了 C++ 指导方针的接口之中（以获得更好的抽象、内存安全性和资源安全性），并在 C++ 代码中使用这个 C++ 接口。

##### 示例

在 C++ 中可以调用 C：

    // C 中:
    double sqrt(double);
    
    // C++ 中:
    extern "C" double sqrt(double);
    
    sqrt(2);

##### 示例

在 C 中可以调用 C++：

    // C 中:
    X call_f(struct Y*, int);
    
    // C++ 中:
    extern "C" X call_f(Y* p, int i)
    {
        return p->f(i);   // 可能是虚函数调用
    }

##### 强制实施

不需要做什么。

# <a name="S-source"></a>SF: 源文件

区分声明（用作接口）和定义（用作实现）。
用头文件来表达接口并强调逻辑结构。

源文件规则概览：

* [SF.1: 如果你的项目还未采用别的约定的话，应当为代码文件使用后缀 `.cpp`，而对接口文件使用后缀 `.h`](#Rs-file-suffix)
* [SF.2: `.h` 文件不应当包含对象定义或非内联的函数定义](#Rs-inline)
* [SF.3: 对在多个源文件中使用的任何声明，都应使用 `.h` 文件](#Rs-declaration-header)
* [SF.4: 在文件中的其他所有声明之前包含 `.h` 文件](#Rs-include-order)
* [SF.5: `.cpp` 文件必须包含定义了它的接口的一个或多个 `.h` 文件](#Rs-consistency)
* [SF.6: `using namespace` 指令，（仅）可以为迁移而使用，可以为基础程序库使用（比如 `std`），或者在局部作用域中使用](#Rs-using)
* [SF.7: 请勿在头文件中的全局作用域使用 `using namespace` 指令](#Rs-using-directive)
* [SF.8: 为所有的 `.h` 文件使用 `#include` 防卫宏](#Rs-guards)
* [SF.9: 避免源文件的循环依赖](#Rs-cycles)
* [SF.10: 避免依赖于隐含地 `#include` 进来的名字](#Rs-implicit)
* [SF.11: 头文件应当是自包含的](#Rs-contained)

* [SF.20: 用 `namespace` 表示逻辑结构](#Rs-namespace)
* [SF.21: 请勿在头文件中使用无名（匿名）命名空间](#Rs-unnamed)
* [SF.22: 为所有的内部/不导出的实体使用无名（匿名）命名空间](#Rs-unnamed2)

### <a name="Rs-file-suffix"></a>SF.1: 如果你的项目还未采用别的约定的话，应当为代码文件使用后缀 `.cpp`，而对接口文件使用后缀 `.h`

##### 理由

这是一条历史悠久的约定。
不过一致性更加重要，因此如果你的项目用了别的约定的话，应当遵守它。

##### 注解

这项约定反应了一种常见使用模式：
头文件更容易和 C 语言共享使用，并可以作为 C++ 和 C 编译，它们通常用 `.h` 后缀，
并且对于有意要和 C 共用的头文件来说，让所有头文件都使用 `.h` 而不是别的扩展名要更加容易。
另一方面，实现文件则很少会和 C 共用，通常应当和 `.c` 文件相区别，
因此一般最好为所有的 C++ 实现文件用别的扩展名（如 `.cpp`）来命名。

特定的名字 `.h` 和 `.cpp` 并不是必要的（只是作为缺省建议），其他的名字也被广泛采用。
例子包括 `.hh`，`.C`，和 `.cxx` 等。请以类似方式使用这些名字。
本文档中我们把 `.h` 和 `.cpp` 作为头文件和实现文件的简便提法，
虽然实际上的扩展名可能是不同的。

你的 IDE（如果你使用的话）可能对后缀有较强的倾向。

##### 示例

    // foo.h:
    extern int a;   // 声明
    extern void foo();
    
    // foo.cpp:
    int a;   // 定义
    void foo() { ++a; }

`foo.h` 提供了 `foo.cpp` 的接口。最好避免全局变量。

##### 示例，不好

    // foo.h:
    int a;   // 定义
    void foo() { ++a; }

一个程序中两次 `#include <foo.h>` 将导致因为对唯一定义规则的两次违反而出现一个连接错误。

##### 强制实施

* 对不符合约定的文件名进行标记。
* 检查 `.h` 和 `.cpp`（或等价文件）遵循下列各规则。

### <a name="Rs-inline"></a>SF.2: `.h` 文件不应当包含对象定义或非内联的函数定义

##### 理由

对受制于唯一定义规则的实体的包含将导致连接错误。

##### 示例

    // file.h:
    namespace Foo {
        int x = 7;
        int xx() { return x+x; }
    }
    
    // file1.cpp:
    #include <file.h>
    // ... 更多代码 ...
    
     // file2.cpp:
    #include <file.h>
    // ... 更多代码 ...

当连接 `file1.cpp` 和 `file2.cpp` 时将出现两个连接器错误。

**其他形式**: `.h` 文件必须仅包含：

* `#include` 其他的 `.h` 文件（可能包括包含防卫宏）
* 模板
* 类定义
* 函数声明
* `extern` 声明
* `inline` 函数定义
* `constexpr` 定义
* `const` 定义
* `using` 别名定义
* ???

##### 强制实施

根据以上白名单来检查。

### <a name="Rs-declaration-header"></a>SF.3: 对在多个源文件中使用的任何声明，都应使用 `.h` 文件

##### 理由

可维护性。可读性。

##### 示例，不好

    // bar.cpp:
    void bar() { cout << "bar\n"; }
    
    // foo.cpp:
    extern void bar();
    void foo() { bar(); }

`bar` 的维护者在需要改变 `bar` 的类型时，无法找到其全部声明。
`bar` 的使用者不知道他所使用的接口是否完整和正确。顶多会从连接器获得一些（延迟的）错误消息。

##### 强制实施

* 对并未放入 `.h` 而在其他源文件中的实体声明进行标记。

### <a name="Rs-include-order"></a>SF.4: 在文件中的其他所有声明之前包含 `.h` 文件

##### 理由

最小化上下文的依赖并增加可读性。

##### 示例

    #include <vector>
    #include <algorithm>
    #include <string>
    
    // ... 我自己的代码 ...

##### 示例，不好

    #include <vector>
    
    // ... 我自己的代码 ...
    
    #include <algorithm>
    #include <string>

##### 注解

这对于 `.h` 和 `.cpp` 文件都同样适用。

##### 注解

有一种论点是通过在打算保护的代码的*后面*再 `#include` 头文件，以此将代码同头文件中的声明式和宏等之间进行隔离
（如上面例子中标为"不好"之处）。
不过，

* 这只能对单个文件（在单个层次上）工作：如果采用这个技巧的头文件被别的头文件所包含，这个威胁就会再次出现。
* 命名空间（一个"实现命名空间"）可以针对许多的上下文依赖进行保护。
* 完全的保护和灵活性需要模块。

**参见**：

* [工作草案，C++ 的模块扩展](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2016/n4592.pdf)
* [模块，组件化及其迁移](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2016/p0141r0.pdf)

##### 强制实施

容易。

### <a name="Rs-consistency"></a>SF.5: `.cpp` 文件必须包含定义了它的接口的一个或多个 `.h` 文件

##### 理由

这使得编译器可以提早进行一致性检查。

##### 示例，不好

    // foo.h:
    void foo(int);
    int bar(long);
    int foobar(int);
    
    // foo.cpp:
    void foo(int) { /* ... */ }
    int bar(double) { /* ... */ }
    double foobar(int);

这个错误直到调用了 `bar` 或 `foobar` 的程序的连接时才会被发现。

##### 示例

    // foo.h:
    void foo(int);
    int bar(long);
    int foobar(int);
    
    // foo.cpp:
    #include <foo.h>
    
    void foo(int) { /* ... */ }
    int bar(double) { /* ... */ }
    double foobar(int);   // 错误: 错误的返回类型

`foobar` 的返回类型错误在编译 `foo.cpp` 时立即就被发现了。
对 `bar` 的参数类型错误在连接时之前无法被发现，因为可能会有重载发生，但系统性地使用 `.h` 文件能够增加时其被程序员更早发现的可能性。

##### 强制实施

???

### <a name="Rs-using"></a>SF.6: `using namespace` 指令，（仅）可以为迁移而使用，可以为基础程序库使用（比如 `std`），或者在局部作用域中使用

##### 理由

`using namespace` 可能造成名字冲突，因而应当节制使用。
然而，将用户代码中的每个命名空间中的名字都进行限定并不总是能够做到（比如在转换过程中）
而且有时候命名空间非常基础，并且在代码库中广为使用，坚持进行限定将使其既啰嗦又分散注意力。

##### 示例

    #include <string>
    #include <vector>
    #include <iostream>
    #include <memory>
    #include <algorithm>
    
    using namespace std;
    
    // ...

显然地，大量使用了标准库，而且貌似没使用别的程序库，因此要求每一处带有使用 `std::`
会使人分散注意力。

##### 示例

使用 `using namespace std;` 导致程序员可能面临与标准库中的名字造成名字冲突

    #include <cmath>
    using namespace std;
    
    int g(int x)
    {
        int sqrt = 7;
        // ...
        return sqrt(x); // 错误
    }

不过，不大可能导致并非错误的名字解析，
假定使用 `using namespace std` 的人们都了解 `std` 以及这种风险。

##### 注解

`.cpp` 文件也是一种形式的局部作用域。
包含一条 `using namespace X` 的 N 行的 `.cpp` 文件中发生名字冲突的机会，
和包含一条 `using namespace X` 的 N 行的函数，
以及每个都包含一条 `using namespace X` 的总行数为 N 行的 M 个函数，没有多少差别。

##### 注解

[请勿在头文件中使用 `using namespace`](#Rs-using-directive)。

##### 强制实施

对于单个源文件中，对不同命名空间的多个 `using namespace` 指令进行标记。

### <a name="Rs-using-directive"></a>SF.7: 请勿在头文件中的全局作用域使用 `using namespace`

##### 理由

这样做使 `#include` 一方无法有效地进行区分并使用其他方式。这还可能使所 `#include` 的头文件之间出现顺序依赖，它们以不同次序包含时可能具有不同的意义。

##### 示例

    // bad.h
    #include <iostream>
    using namespace std; // bad
    
    // user.cpp
    #include "bad.h"
    
    bool copy(/*... some parameters ...*/);    // some function that happens to be named copy
    
    int main() {
        copy(/*...*/);    // now overloads local ::copy and std::copy, could be ambiguous
    }

##### 注解

一个例外是 `using namespace std::literals;`。若要在头文件中使用
字符串字面量，则必须如此，而且根据[规则](http://eel.is/c++draft/over.literal)——用户必须以
`operator""_x` 来命名他们自己的 UDL——它们并不会与标准库相冲突。

##### 强制实施

标记头文件的全局作用域中的 `using namespace`。

### <a name="Rs-guards"></a>SF.8: 为所有的 `.h` 文件使用 `#include` 防卫宏

##### 理由

避免文件被多次 `#include`。

为避免包含防卫宏的冲突，不要仅使用文件名来命名防卫宏。
确保还要包含一个关键词和好的区分词，比如头文件所属的程序库
或组件的名字。

##### 示例

    // file foobar.h:
    #ifndef LIBRARY_FOOBAR_H
    #define LIBRARY_FOOBAR_H
    // ... 声明 ...
    #endif // LIBRARY_FOOBAR_H

##### 强制实施

标记没有 `#include` 防卫的 `.h` 文件。

##### 注解

一些实现提供了如 `#pragma once` 这样的厂商扩展作为包含防卫宏的替代。
这并非标准且不可移植。它向程序中注入了宿主机器的文件系统的语义，
而且把你锁定到某个特定厂商。
我们的建议是编写 ISO C++：参见[规则 P.2](#Rp-Cplusplus)。

### <a name="Rs-cycles"></a>SF.9: 避免源文件的循环依赖

##### 理由

循环会使理解变得困难，并拖慢编译速度。
它们还会使（当其可用时）向利用语言支持的模块进行转换工作变得复杂。

##### 注解

要消除循环依赖；请勿仅仅用 `#include` 防卫宏来试图打破它们。

##### 示例，不好

    // file1.h:
    #include "file2.h"
    
    // file2.h:
    #include "file3.h"
    
    // file3.h:
    #include "file1.h"

##### 强制实施

对任何循环依赖进行标记。


### <a name="Rs-implicit"></a>SF.10: 避免依赖于隐含地 `#include` 进来的名字

##### 理由

避免意外。
避免当 `#include` 的头文件改变时改变一条 `#include`。
避免意外地变为依赖于所包含的头文件中的实现细节和逻辑上独立的实体。

##### 示例

    #include <iostream>
    using namespace std;
    
    void use()                  // 不好
    {
        string s;
        cin >> s;               // 好
        getline(cin, s);        // 错误：getline() 未定义
        if (s == "surprise") {  // 错误：== 未定义
            // ...
        }
    }

`<iostream>` 暴露了 `std::string` 的定义（"为什么？"是一个有趣的问题），
但其并不必然是通过传递包含整个 `<string>` 头文件而做到这一点的，
这带来了常见的新手问题"为什么 `getline(cin,s);` 不成？"，
甚至偶尔出现的"`string` 无法用 `==` 来比较"。

其解决方案是明确地 `#include <string>`：

    #include <iostream>
    #include <string>
    using namespace std;
    
    void use()
    {
        string s;
        cin >> s;               // 好
        getline(cin, s);        // 好
        if (s == "surprise") {  // 好
            // ...
        }
    }

##### 注解

一些头文件正是用于从一些头文件中合并一组声明。
例如：

    // basic_std_lib.h:
    
    #include <string>
    #include <map>
    #include <iostream>
    #include <random>
    #include <vector>

用户只用一条 `#include` 就可以获得整组的声明了：

    #include "basic_std_lib.h"

本条反对隐式包含的规则并不防止这种特意的聚集包含。

##### 强制实施

强制实施将需要一些有关头文件中哪些是"导出"给用户所用而哪些是用于实现的知识。
在我们能用到模块之前没有真正的好方案。

### <a name="Rs-contained"></a>SF.11: 头文件应当是自包含的

##### 理由

易用性，头文件应当易于使用，且单独包含即可正常工作。
头文件应当对其所提供的功能进行封装。
避免让头文件的使用方来管理它的依赖项。

##### 示例

    #include "helpers.h"
    // helpers.h 依赖于 std::string 并已包含了 <string>

##### 注解

不遵守这条规则将导致头文件的使用方难于诊断所出现的错误。

##### 注解

头文件应当包含其所有依赖项。请小心使用相对路径，各 C++ 实现对于它们的含义是有分歧的。

##### 强制实施

以一项测试来验证头文件自身可通过编译，或者一个仅包含了该头文件的 cpp 文件可通过编译。

### <a name="Rs-namespace"></a>SF.20: 用 `namespace` 表示逻辑结构

##### 理由

 ???

##### 示例

    ???

##### 强制实施

???

### <a name="Rs-unnamed"></a>SF.21: 请勿在头文件中使用无名（匿名）命名空间

##### 理由

在头文件中使用无名命名空间差不多都是一个 BUG。

##### 示例

    ???

##### 强制实施

* 对头文件中所使用的任何匿名命名空间进行标记。

### <a name="Rs-unnamed2"></a>SF.22: 为所有的内部/不导出的实体使用无名（匿名）命名空间

##### 理由

外部实体无法依赖于嵌套的无名命名空间中的实体。
考虑将实现源文件中的所有定义都放入无名命名空间中，除非它定义的是一个"外部/导出"实体。

##### 示例

API 类及其成员不能放在无名命名空间中；而在实现源文件中所定义的任何的"辅助"类或函数则应当放在无名命名空间作用域之中。

    ???

##### 强制实施

* ???

# <a name="S-stdlib"></a>SL: 标准库

如果只使用纯语言本身的话，任何开发任务都会变得很麻烦（无论以何种语言）。
如果使用了某个合适的程序库的话，则任何开发任务都会变得相当简单。

这些年来标准库一直在持续增长。
现在它在标准中的描述已经比语言功能特性的描述更大了。
因此，可能指导方针的库部分的规模最终将会增长等于甚至超过其他的所有部分。

<< ??? 我们需要另一个层次的规则编号 ??? >>

C++ 标准库组件概览：

* [SL.con: 容器](#SS-con)
* [SL.str: 字符串](#SS-string)
* [SL.io: I/O 流（iostream）](#SS-io)
* [SL.regex: 正则表达式](#SS-regex)
* [SL.chrono: 时间](#SS-chrono)
* [SL.C: C 标准库](#SS-clib)

标准库规则概览：

* [SL.1: 尽可能使用程序库](#Rsl-lib)
* [SL.2: 优先使用标准库而不是其他程序库](#Rsl-sl)
* [SL.3: 请勿向命名空间 `std` 中添加非标准实体](#sl-std)
* [SL.4: 以类型安全的方式使用标准库](#sl-safe)
* ???

### <a name="Rsl-lib"></a>SL.1:  尽可能使用程序库

##### 理由

节约时间。避免重复发明轮子。
避免重复他人的工作。
如果其他人的工作有了改进，则可以从中获得好处。
当你进行了改进之后可以帮助其他人。

### <a name="Rsl-sl"></a>SL.2: 优先使用标准库而不是其他程序库

##### 理由

了解标准库的人更多。
相对于你自己的代码或者大多数其他程序库来说，标准库更加倾向于稳定，进行了良好维护，而且广泛可用。


### <a name="sl-std"></a>SL.3: 请勿向命名空间 `std` 中添加非标准实体

##### 理由

向 `std` 中添加东西可能会改变本来是遵循标准的代码的含义。
添加到 `std` 的东西可能会与未来版本的标准产生冲突。

##### 示例

    ???

##### 强制实施

有可能，但很麻烦而且在一些平台上很可能导致一些问题。

### <a name="sl-safe"></a>SL.4: 以类型安全的方式使用标准库

##### 理由

因为，很显然，违反这条规则将导致未定义的行为，内存损坏，以及其他所有种类的糟糕的错误。

##### 注解

本条规则是半哲学性的元规则，需要许多具体规则予以支持。
我们需要将之作为对于更加专门的规则的总括。

更加专门的规则概览：

* [SL.4: 以类型安全的方式使用标准库](#sl-safe)


## <a name="SS-con"></a>SL.con: 容器

???

容器规则概览：

* [SL.con.1: 优先采用 STL 的 `array` 或 `vector` 而不是 C 数组](#Rsl-arrays)
* [SL.con.2: 除非有理由使用别的容器，否则默认情况应优先采用 STL 的 `vector`](#Rsl-vector)
* [SL.con.3: 避免边界错误](#Rsl-bounds)
* [SL.con.4: 请勿对非可平凡复制的实参使用 `memset` 或 `memcpy`](#Rsl-copy)

### <a name="Rsl-arrays"></a>SL.con.1: 优先采用 STL 的 `array` 或 `vector` 而不是 C 数组

##### 理由

C 数组不那么安全，而且相对于 `array` 和 `vector` 也没有什么优势。
对于定长数组，应使用 `std::array`，它传递给函数时并不会退变为指针并丢失其大小信息。
而且，和内建数组一样，栈上分配的 `std::array` 会在栈上保存它的各个元素。
对于变长数组，应使用 `std::vector`，它还可以改变大小并处理内存分配。

##### 示例

    int v[SIZE];                        // 不好
    
    std::array<int, SIZE> w;             // ok

##### 示例

    int* v = new int[initial_size];     // 不好，有所有权的原生指针
    delete[] v;                         // 不好，手工 delete
    
    std::vector<int> w(initial_size);   // ok

##### 注解

在不拥有而引用容器中的元素时使用 `gsl::span`。

##### 注解

在栈上分配的固定大小的数组和把元素都放在自由存储上的 `vector` 之间比较性能是没什么意义的。
你同样也可以在栈上的 `std::array` 和通过指针访问 `malloc()` 的结果之间进行这样的比较。
对于大多数代码来说，即便是栈上分配和自由存储分配之间的差异也没那么重要，但 `vector` 带来的便利和安全性却是重要的。
如果有人编写的代码中这种差异确实重要，那么他显然可以在 `array` 和 `vector` 之间做出选择。

##### 强制实施

* 如果 C 数组的声明所在的函数或类也声明了 STL 的某个容器（这是为了避免在老式的非 STL 代码中的大量警告噪音），则对其进行标记。修正：最少要把 C 数组改成 `std::array`。

### <a name="Rsl-vector"></a>SL.con.2: 除非有理由使用别的容器，否则默认情况应优先采用 STL 的 `vector`

##### 理由

`vector` 和 `array` 是仅有的能够提供以下各项优势的标准容器：

* 最快的通用访问（随机访问，还包括对于向量化友好性）；
* 最快的默认访问模式（从头到尾或从尾到头方式是对预读器友好的）；
* 最少的空间耗费（连续布局中没有每个元素的开销，而且是 cache 友好的）。

通常你都需要对容器进行元素的添加和删除，因此默认应当采用 `vector`；如果并不需要改动容器的大小的话，则应采用 `array`。

即便其他容器貌似更加合适，比如 `map` 的 O(log N) 查找性能，或者 `list` 的中部高效插入，对于几个 KB 以内大小的容器来说，`vector` 仍然经常性能更好。

##### 注解

`string` 不应当用作独立字符的容器。`string` 是文本字符串；如果需要字符的容器的话，应当采用 `vector</*char_type*/>` 或者 `array</*char_type*/>`。

##### 例外

如果你有正当的理由来使用别的容器的话，就请使用它。例如：

* 若 `vector` 满足你的需求，但你并不需要容器大小可变，则应当代之以 `array`。

* 若你需要支持字典式查找的容器并保证 O(K) 或 O(log N) 的查找效率，而且容器将会比较大（超过几个 KB），你需要经常进行插入使得维护有序的 `vector` 的开销不大可行，则请代之以使用 `unordered_map` 或者 `map`。

##### 注解

使用 `()` 初始化来将 `vector` 初始化为具有特定数量的元素。
使用 `{}` 初始化来以一个元素列表来对 `vector` 进行初始化。

    vector<int> v1(20);  // v1 具有 20 个值为 0 的元素（vector<int>{}）
    vector<int> v2 {20}; // v2 具有 1 个值为 20 的元素

[优先采用 `{}` 初始化式语法](#Res-list)。

##### 强制实施

* 如果 `vector` 构造之后大小不会改变（比如因为它是 `const` 或者因为没有对它调用过非 `const` 函数），则对其进行标记。修正：代之以使用 `array`。

### <a name="Rsl-bounds"></a>SL.con.3: 避免边界错误

##### 理由

越过已分配的元素的范围进行读写，通常都会导致糟糕的错误，不正确的结果，程序崩溃，以及安全漏洞。

##### 注解

应用于一组元素的范围的标准库函数，都有（或应当有）接受 `span` 的边界安全重载。
如 `vector` 这样的标准类型，在边界剖面配置下（以某种不兼容的方式，如添加契约）可以被修改为实施边界检查，或者使用 `at()`。

理想情况下，边界内保证应当可以被静态强制实行。
例如：

* 基于范围的 `for` 的循环不会越过其所针对的容器的范围
* `v.begin(),v.end()` 可以很容易确定边界安全性

这种循环和任何的等价的无检查或不安全的循环一样高效。

通常，可以用一个简单的预先检查来消除检查每个索引的需要。
例如

* 对于 `v.begin(),v.begin()+i`，`i` 可以很容易针对 `v.size()` 检查

这种循环比每次都待检查元素访问要快得多。

##### 示例，不好

    void f()
    {
        array<int, 10> a, b;
        memset(a.data(), 0, 10);         // 不好，且包含长度错误（length = 10 * sizeof(int)）
        memcmp(a.data(), b.data(), 10);  // 不好，且包含长度错误（length = 10 * sizeof(int)）
    }

而且，`std::array<>::fill()` 或 `std::fill()`，甚或是空的初始化式，都比 `memset()` 要好。

##### 示例，好

    void f()
    {
        array<int, 10> a, b, c{};       // c 被初始化为零
        a.fill(0);
        fill(b.begin(), b.end(), 0);    // std::fill()
        fill(b, 0);                     // std::fill() + Ranges TS
    
        if ( a == b ) {
          // ...
        }
    }

##### Example

如果代码使用的是未修改的标准库，仍然有一些变通方案来以边界安全的方式使用 `std::array` 和 `std::vector`。代码中可以调用各个类的 `.at()` 成员函数，这将抛出 `std::out_of_range` 异常。或者，代码中可以调用 `at()` 自由函数，这将在边界违例时导致快速失败（或者某个自定义动作）。

    void f(std::vector<int>& v, std::array<int, 12> a, int i)
    {
        v[0] = a[0];        // 不好
        v.at(0) = a[0];     // OK（替代方案 1）
        at(v, 0) = a[0];    // OK（替代方案 2）
    
        v.at(0) = a[i];     // 不好
        v.at(0) = a.at(i);  // OK（替代方案 1）
        v.at(0) = at(a, i); // OK（替代方案 2）
    }

##### 强制实施

* 对于没有边界检查的标准库函数的任何调用都给出诊断消息。
??? 在这里添加一组禁用函数的连接列表

本条规则属于[边界剖面配置](#SS-bounds)。


### <a name="Rsl-copy"></a>SL.con.4: 请勿对非可平凡复制的实参使用 `memset` 或 `memcpy`

##### 理由

这样做会破坏对象语义（例如，其会覆写掉 `vptr`）。

##### 注解

`(w)memset`，`(w)memcpy`，`(w)memmove`，以及 `(w)memcmp` 与此相似。

##### 示例

    struct base {
        virtual void update() = 0;
    };
    
    struct derived : public base {
        void update() override {}
    };


    void f (derived& a, derived& b) // 虚表再见！
    {
        memset(&a, 0, sizeof(derived));
        memcpy(&a, &b, sizeof(derived));
        memcmp(&a, &b, sizeof(derived));
    }

应当代之以定义适当的默认初始化，复制，以及比较函数

    void g(derived& a, derived& b)
    {
        a = {};    // 默认初始化
        b = a;     // 复制
        if (a == b) do_something(a,b);
    }

##### 强制实施

* 对在不可平凡复制的类型使用这些函数进行标记

**TODO 注释**:

* 对于标准库的影响需要和 WG21 之间进行紧密的协调，即便不需要标准化也应当至少保证兼容性。
* 我们正在考虑为标准库（尤其是 C 标准库）中如 `memcmp` 这样的函数指定边界安全的重载，并在 GSL 中提供它们。
* 对于标准中没有进行完全的边界检查的现存函数和如 `vector` 这样的类型来说，我们的目标是在启用了边界剖面配置的代码中调用时，这些功能应当进行边界检查，而从遗留代码中调用时则没有检查，可能需要利用契约来实现（正由几个 WG21 成员进行提案工作）。



## <a name="SS-string"></a>SL.str: 字符串

文本处理是一个大的主题。
`std::string` 无法全部覆盖这些。
这一部分主要尝试澄清 `std::string` 和 `char*`、`zstring`、`string_view` 和 `gsl::string_span` 之间的关系。
有关非 ASCII 字符集和编码的重要问题（比如 `wchar_t`，Unicode，以及 UTF-8 等）将在别处讨论。

**参见**：[正则表达式](#SS-regex)

在这里，我们用"字符序列"或"字符串"来代表（终将）作为文本来读取的字符序列。
We don't consider ???

字符串概览：

* [SL.str.1: 使用 `std::string` 以拥有字符序列](#Rstr-string)
* [SL.str.2: 使用 `std::string_view` 或 `gsl::string_span` 以指代字符序列](#Rstr-view)
* [SL.str.3: 使用 `zstring` 或 `czstring` 以指代 C 风格、以零结尾的字符序列](#Rstr-zstring)
* [SL.str.4: 使用 `char*` 以指代单个字符](#Rstr-char*)
* [SL.str.5: 使用 `std::byte` 以指代并不必须表示字符的字节值](#Rstr-byte)

* [SL.str.10: 当需要实施相关于文化地域的操作时，使用 `std::string`](#Rstr-locale)
* [SL.str.11: 当需要改动字符串时，使用 `gsl::string_span` 而不是 `std::string_view`](#Rstr-span)
* [SL.str.12: 为作为标准库的 `string` 类型的字符串字面量使用后缀 `s`](#Rstr-s)

**参见**：

* [F.24 span](#Rf-range)
* [F.25 zstring](#Rf-zstring)


### <a name="Rstr-string"></a>SL.str.1: 使用 `std::string` 以拥有字符序列

##### 理由

`string` 能够正确处理资源分配，所有权，复制，渐进扩容，并提供许多有用的操作。

##### 示例

    vector<string> read_until(const string& terminator)
    {
        vector<string> res;
        for (string s; cin >> s && s != terminator; ) // 读取一个单词
            res.push_back(s);
        return res;
    }

注意已经为 `string` 提供了 `>>` 和 `!=`（作为有用操作的例子），并且没有显示的内存分配，
回收，或者范围检查（`string` 会处理这些）。

C++17 中，我们可以使用 `string_view` 而不是 `const string*` 作为参数，以允许调用方更大的灵活性：

    vector<string> read_until(string_view terminator)   // C++17
    {
        vector<string> res;
        for (string s; cin >> s && s != terminator; ) // 读取一个单词
            res.push_back(s);
        return res;
    }

`gsl::string_span` 是当前的一种替代方案，为简单的例子提供了 `std::string_view` 的大多数优势：

    vector<string> read_until(string_span terminator)
    {
        vector<string> res;
        for (string s; cin >> s && s != terminator; ) // 读取一个单词
            res.push_back(s);
        return res;
    }

##### 示例，不好

不要使用 C 风格的字符串来进行需要不单纯的内存管理的操作：

    char* cat(const char* s1, const char* s2)   // 当心！
        // return s1 + '.' + s2
    {
        int l1 = strlen(s1);
        int l2 = strlen(s2);
        char* p = (char*)malloc(l1 + l2 + 2);
        strcpy(p, s1, l1);
        p[l1] = '.';
        strcpy(p + l1 + 1, s2, l2);
        p[l1 + l2 + 1] = 0;
        return p;
    }

我们搞对了吗？
调用者能记得要对返回的指针调用 `free()` 吗？
这段代码能通过安全性评审吗？

##### 注解

没有测量就不要假设 `string` 比底层技术慢，要记得并非所有代码都是性能攸关的。
[请勿进行不成熟的优化](#Rper-Knuth)

##### 强制实施

???

### <a name="Rstr-view"></a>SL.str.2: 使用 `std::string_view` 或 `gsl::string_span` 以指代字符序列

##### 理由

`std::string_view` 或 `gsl::string_span` 提供了简易且（潜在）安全的对字符序列的访问，并与序列的
分配和存储方式无关。

##### 示例

    vector<string> read_until(string_span terminator);
    
    void user(zstring p, const string& s, string_span ss)
    {
        auto v1 = read_until(p);
        auto v2 = read_until(s);
        auto v3 = read_until(ss);
        // ...
    }

##### 注解

`std::string_view`（C++17）是只读的。

##### 强制实施

???

### <a name="Rstr-zstring"></a>SL.str.3: 使用 `zstring` 或 `czstring` 以指代 C 风格、以零结尾的字符序列

##### 理由

可读性。
明确意图。
普通的 `char*` 可以是指向单个字符的指针，指向字符数组的指针，指向 C 风格（零结尾）字符串的指针，甚或是指向小整数的指针。
对这些情况加以区分能够避免误解和 BUG。

##### 示例

    void f1(const char* s); // s 可能是个字符串

我们所知的只不过是它可能是 nullptr 或者指向至少一个字符

    void f1(zstring s);     // s 是 C 风格字符串或者 nullptr
    void f1(czstring s);    // s 是 C 风格字符串常量或者 nullptr
    void f1(std::byte* s);  // s 是某个字节的指针（C++17）

##### 注解

除非确实有理由，否则不要把 C 风格的字符串转换为 `string`。

##### 注解

与其他的"普通指针"一样，`zstring` 不能表达所有权。

##### 注解

已经存在了上亿行的 C++ 代码，它们大多使用 `char*` 和 `const char*` 却并不注明其意图。
各种不同的方式都在使用它们，包括以之表示所有权，以及（代替 `void*`）作为通用的内存指针。
很难区分这些用法，因此这条指导方针很难被遵守。
而这是 C 和 C++ 程序中的最主要的 BUG 来源之一，因此一旦可行就遵守这条指导方针是值得的。

##### 强制实施

* 标记在 `char*` 上使用的 `[]`
* 标记在 `char*` 上使用的 `delete`
* 标记在 `char*` 上使用的 `free()`

### <a name="Rstr-char*"></a>SL.str.4: 使用 `char*` 以指代单个字符

##### 示例

现存代码中对 `char*` 的各种不同用法，是一种主要的错误来源。

##### 示例，不好

    char arr[] = {'a', 'b', 'c'};
    
    void print(const char* p)
    {
        cout << p << '\n';
    }
    
    void use()
    {
        print(arr);   // 运行时错误；可能非常糟糕
    }

数组 `arr` 并非 C 风格字符串，因为它不是零结尾的。

##### 替代方案

参见 [`zstring`](#Rstr-zstring)，[`string`](#Rstr-string)，以及 [`string_span`](#Rstr-view)。

##### 强制实施

* 标记在 `char*` 上使用的 `[]`

### <a name="Rstr-byte"></a>SL.str.5: 使用 `std::byte` 以指代并不必须表示字符的字节值

##### 理由

用 `char*` 来表示指向不一定是字符的东西的指针会造成混乱，
并会妨碍有价值的优化。

##### 示例

    ???

##### 注解

C++17

##### 强制实施

???


### <a name="Rstr-locale"></a>SL.str.10: 当需要实施相关于文化地域的操作时，使用 `std::string`

##### 理由

`std::string` 支持标准库的 [`locale` 功能](#Rstr-locale)

##### 示例

    ???

##### 注解

???

##### 强制实施

???

### <a name="Rstr-span"></a>SL.str.11: 当需要改动字符串时，使用 `gsl::string_span` 而不是 `std::string_view`

##### 理由

`std::string_view` 是只读的。

##### 示例

???

##### 注解

???

##### 强制实施

编译器会标记出试图写入 `string_view` 的地方。

### <a name="Rstr-s"></a>SL.str.12: 为作为标准库的 `string` 类型的字符串字面量使用后缀 `s`

##### 理由

直接表达想法能够最小化犯错机会。

##### 示例

    auto pp1 = make_pair("Tokyo", 9.00);         // {C 风格字符串,double} 有意如此？
    pair<string, double> pp2 = {"Tokyo", 9.00};  // 稍微啰嗦
    auto pp3 = make_pair("Tokyo"s, 9.00);        // {std::string,double}    // C++14
    pair pp4 = {"Tokyo"s, 9.00};                 // {std::string,double}    // C++17



##### 强制实施

???


## <a name="SS-io"></a>SL.io: I/O 流（iostream）

`iostream` 是一种类型安全的，可扩展的，带格式的和无格式的流式 I/O 的 I/O 程序库。
它支持多种（且用户可扩展的）缓冲策略以及多种文化地域。
它可以用于进行便利的 I/O，内存读写（字符串流），
以及用户定义的扩展，诸如跨网络的流（asio：尚未标准化）。

I/O 流规则概览：

* [SL.io.1: 仅在必要时才使用字符层面的输入](#Rio-low)
* [SL.io.2: 当进行读取时，总要考虑非法输入](#Rio-validate)
* [SL.io.3: 优先使用 iostream 进行 I/O](#Rio-streams)
* [SL.io.10: 除非你使用了 `printf` 族函数，否则要调用 `ios_base::sync_with_stdio(false)`](#Rio-sync)
* [SL.io.50: 避免使用 `endl`](#Rio-endl)
* [???](#???)

### <a name="Rio-low"></a>SL.io.1: 仅在必要时才使用字符层面的输入

##### 理由

除非你确实仅处理单个的字符，否则使用字符级的输入将导致用户代码实施潜在易错的
且潜在低效的从字符进行标记组合的工作。

##### 示例

    char c;
    char buf[128];
    int i = 0;
    while (cin.get(c) && !isspace(c) && i < 128)
        buf[i++] = c;
    if (i == 128) {
        // ... 处理过长的字符串 ....
    }

更好的做法（简单得多而且可能更快）：

    string s;
    s.reserve(128);
    cin>>s;

而且额能并不需要 `reserve(128)`。

##### 强制实施

???


### <a name="Rio-validate"></a>SL.io.2: 当进行读取时，总要考虑非法输入

##### 理由

错误通常最好尽快处理。
如果输入无效，所有的函数都必须编写为对付不良的数据（而这并不现实）。

##### 示例

    ???

##### 强制实施

???

### <a name="Rio-streams"></a>SL.io.3: 优先使用 iostream 进行 I/O

##### 理由

`iosteam` 安全，灵活，并且可扩展。

##### 示例

    // 写出一个复数：
    complex<double> z{ 3,4 };
    cout << z << '\n';

`complex` 是一个用户定义的类型，而其 I/O 的定义无需改动 `iostream` 库。

##### 示例

    // 读取一系列复数：
    for (complex<double> z; cin>>z)
        v.push_back(z);

##### 例外

??? 性能 ???

##### 讨论：`iostream` vs. `printf()` 家族

人们通常说（并且通常是正确的）`printf` 家族比 `iostream` 由两个优势：
格式化的灵活性和性能。
这需要与 `iostream` 在处理用户定义类型方面的扩展性，针对安全性的违反方面的韧性，
隐含的内存管理，以及 `locale` 处理等优势之间进行权衡。

如果需要 I/O 性能的话，你几乎总能做到比 `printf()` 更好。

`gets()`，使用 `%s` 的 `scanf()`，和使用 `%s` 的 `printf()` 在安全性方面冒风险（容易遭受缓冲区溢出问题而且通常很易错）。
C11 定义了一些"可选扩展"，它们对其实参进行一些额外检查。
如果您的 C 程序库中包含 `gets_s()`、`scanf_s()` 和 `printf_s()`，它们将会是更安全的替代方案，但仍然并非是类型安全的。

##### 强制实施

可选地标记 `<cstdio>` 和 `<stdio.h>`。

### <a name="Rio-sync"></a>SL.io.10: 除非你使用了 `printf` 族函数，否则要调用 `ios_base::sync_with_stdio(false)`

##### 理由

`iostreams` 和 `printf` 风格的 I/O 之间的同步是由代价的。
`cin` 和 `cout` 默认是与 `printf` 相同步的。

##### 示例

    int main()
    {
        ios_base::sync_with_stdio(false);
        // ... 使用 iostreams ...
    }

##### 强制实施

???

### <a name="Rio-endl"></a>SL.io.50: 避免使用 `endl`

##### 理由

`endl` 操纵符大致相当于 `'\n'` 和 `"\n"`；
其最常用的情况只不过会以添加多余的 `flush()` 的方式拖慢程序。
与 `printf` 式输出相比，这种拖慢程度是比较显著的。

##### 示例

    cout << "Hello, World!" << endl;    // 两次输出操作和一次 flush
    cout << "hello, World!\n";          // 一次输出操作且没有 flush

##### 注解

对于 `cin`/`cout`（或同等设备）的交互来说，没什么原因必须进行冲洗；它们是自动进行的。
对于向文件写入来说，也很少需要 `flush`。

##### 注解

除了（偶尔会比较重要的）性能问题外，
从 `"\\n"` 和 `endl` 之间进行选择基本上完全是审美问题。

## <a name="SS-regex"></a>SL.regex: 正则表达式

`<regex>` 是标准 C++ 的正则表达式库。
它支持许多正则表达式的模式约定。

## <a name="SS-chrono"></a>SL.chrono: 时间

`<chrono>`（在命名空间 `std::chrono` 中定义）提供了 `time_point` 和 `duration`，并同时提供了
用于以各种不同单位输出时间的函数。
它还提供了用于注册 `time_point` 的时钟。

## <a name="SS-clib"></a>SL.C: C 标准库

???

C 标准库规则概览：

* [S.C.1: 请勿使用 setjmp/longjmp](#Rclib-jmp)
* [???](#???)
* [???](#???)

### <a name="Rclib-jmp"></a>SL.C.1: 请勿使用 setjmp/longjmp

##### 理由

`longjmp` 会忽略析构函数，由此使得依赖于 RAII 的所有资源管理策略全部失效。

##### 强制实施

标记出现的所有 `longjmp` 和 `setjmp`



# <a name="S-A"></a>A: 架构设计的观念

本部分包括有关高层次的架构性观念和程序库的观念。

架构性规则概览：

* [A.1: 分离稳定的代码和不稳定的代码](#Ra-stable)
* [A.2: 将潜在可复用的部分作为程序库](#Ra-lib)
* [A.4: 程序库之间不能有循环依赖](#Ra-dag)
* [???](#???)
* [???](#???)
* [???](#???)
* [???](#???)
* [???](#???)
* [???](#???)

### <a name="Ra-stable"></a>A.1: 分离稳定的代码和不稳定的代码

对较不稳定的代码进行隔离，有助于其单元测试，接口改进，重构，以及最终弃用。

### <a name="Ra-lib"></a>A.2: 将潜在可复用的部分作为程序库

##### 理由

##### 注解

程序库是一些共同进行维护，文档化，并发布的声明式和定义式的集合体。
程序库可以是一组头文件（"仅有头文件的程序库"），或者一组头文件加上一组目标文件构成。
你可以静态或动态地将程序库连接到程序中，或者你还可以 `#included` 仅头文件的库。


### <a name="Ra-dag"></a>A.4: 程序库之间不能有循环依赖

##### 理由

* 循环依赖导致构建过程变得复杂。
* 循环依赖难于理解，可能会引入不确定性（未定义行为）。

##### 注解

一个程序库可以在它的组件的定义之间包含循环引用。
例如：

    ???

不过，程序库不能对依赖于它的其他程序库产生依赖。


# <a name="S-not"></a>NR: 伪规则和错误的看法

本部分包含一些在不少地方流行的规则和指导方针，但是我们慎重地建议不要采纳它们。
我们完全了解这些规则曾经在某些时间和场合是有意义的，而且我们自己也曾经采用过它们。
不过，在我们所推荐并以各项指导方针所支持的编程风格的情况中，这些"伪规则"是有害的。

即便是今天，仍有一些情况下这些规则是有意义的。
比如说，缺少合适的工具支持会导致异常在硬实时系统中的不适用，
但请不要盲目地信任"通俗智慧"（比如有关"效率"的未经数据支持的观点）；
这种"智慧"可能是基于几十年前的信息，或者是来自于与 C++ 有非常不同性质的语言的经验
（比如 C 或者 Java）。

对于这些伪规则的替代方案的正面观点都在各个规则的"替代方案"部分中给出。

伪规则概览：

* [NR.1: 请勿坚持认为声明都应当放在函数的最上面](#Rnr-top)
* [NR.2: 请勿坚持使函数中只保留一个 `return` 语句](#Rnr-single-return)
* [NR.3: 请勿避免使用异常](#Rnr-no-exceptions)
* [NR.4: 请勿坚持把每个类声明放在其自己的源文件中](#Rnr-lots-of-files)
* [NR.5: 请勿采用两阶段初始化](#Rnr-two-phase-init)
* [NR.6: 请勿把所有清理操作放在函数末尾并使用 `goto exit`](#Rnr-goto-exit)
* [NR.7: 请勿使所有数据成员 `protected`](#Rnr-protected-data)
* ???

### <a name="Rnr-top"></a>NR.1: 请勿坚持认为声明都应当放在函数的最上面

##### 理由

"所有声明都在开头"的规则，是来自不允许在语句之后对变量和常量进行初始化的老编程语言的遗产。
这样做会导致更长的程序，以及更多由于未初始化的或者错误初始化的变量所导致的错误。

##### 示例，不好

    int use(int x)
    {
        int i;
        char c;
        double d;
    
        // ... 做一些事 ...
    
        if (x < i) {
            // ...
            i = f(x, d);
        }
        if (i < x) {
            // ...
            i = g(x, c);
        }
        return i;
    }

未初始化变量和其使用点的距离越长，出现 BUG 的机会就越大。
幸运的是，编译器可以发现许多"设值前使用"的错误。
不幸的是，编译器无法捕捉到所有这样的错误，而且一些 BUG 并不都像这个小例子中的这样容易发现。


##### 替代方案

* [坚持为对象进行初始化](#Res-always)。
* [ES.21: 不要在确实需要使用变量（或常量）之前就引入它](#Res-introduce)。

### <a name="Rnr-single-return"></a>NR.2: 请勿坚持使函数中只保留一个 `return` 语句

##### 理由

单返回规则会导致不必要地复杂的代码，并引入多余的状态变量。
特别是，单返回规则导致更难在函数开头集中进行错误检查。

##### 示例

    template<class T>
    //  requires Number<T>
    string sign(T x)
    {
        if (x < 0)
            return "negative";
        else if (x > 0)
            return "positive";
        return "zero";
    }

为仅使用一个返回语句，我们得做类似这样的事：

    template<class T>
    //  requires Number<T>
    string sign(T x)        // 不好
    {
        string res;
        if (x < 0)
            res = "negative";
        else if (x > 0)
            res = "positive";
        else
            res = "zero";
        return res;
    }

这不仅更长，而且很可能效率更差。
越长越复杂的函数，对其进行变通就越是痛苦。
当然许多简单的函数因为它们本来就简单的逻辑都天然就只有一个 `return`。

##### 示例

    int index(const char* p)
    {
        if (!p) return -1;  // 错误指标：替代方案是 "throw nullptr_error{}"
        // ... 进行查找以找出 p 的索引
        return i;
    }

如果我们采纳这条规则的话，得做类似这样的事：

    int index2(const char* p)
    {
        int i;
        if (!p)
            i = -1;  // 错误指标
        else {
            // ... 进行查找以找出 p 的索引
        }
        return i;
    }

注意我们（故意地）违反了禁止未初始化变量的规则，因为这种风格通常都会导致这样。
而且，这种风格也会倾向于采用 [goto exit](#Rnr-goto-exit) 伪规则。

##### 替代方案

* 保持函数短小简单。
* 随意使用多个 `return` 语句（以及抛出异常）。

### <a name="Rnr-no-exceptions"></a>NR.3: 请勿避免使用异常

##### 理由

一般有四种主要的不用异常的理由：

* 异常是低效的
* 异常会导致泄漏和错误
* 异常的性能无法预测
* 异常处理的运行时支持耗费过多空间

我们没有能够满足所有人的解决这个问题的办法。
无论如何，针对异常的讨论已经持续了四十多年了。
一些语言没有异常就无法使用，而另一些并不支持异常。
这在使用和不使用异常的方面都造成了强大的传统，并导致激烈的争论。

不过，我们可以简要说明，为什么我们认为对于通用编程以及这里的指导方针的情况来说，
异常是最佳的候选方案。
简单的论据，无论支持还是反对，都是缺乏说服力的。
确实存在一些特殊的应用，其中异常就是不合适的。
（例如，硬实时系统，且缺乏可靠的对于异常处理的耗费进行估计的支持）。

我们依次来考虑针对异常的主要反对观点：

* 异常是低效的：
和什么相比？
当进行比较时，请确保处理了同样的错误集合，并且它们都进行了等价的处理。
尤其是，不要对一个见到异常就立刻终止的程序和一个在记录错误日志之前
小心地进行资源清理的程序之间进行比较。
确实，某些系统的异常处理实现很糟糕；有时候，这样的实现迫使我们使用
其他错误处理方案，但这并不是异常的基本问题。
当使用某个有效的论据时——无论什么样的上下文——请小心你能拿出确实提供了所讨论的问题的
内部情况的健全的数据。
* 异常会导致泄漏和错误。
不会。
如果你的程序时一大堆乱糟糟的指针而没有总体的资源管理策略，
那么无论你干什么都会有问题。
如果你的系统是由上百万行这样的代码构成的，
那你可能是无法使用异常的，
不过这是一个有关过度和放纵的使用指针的问题，而不是异常的问题。
我们的观点是，你需要用 RAII 来让基于异常的错误处理变得简单且安全——比其他方案都要更简单和安全。
* 异常的性能无法预测。
如果你是在硬实时系统上，而你必须确保一个任务要在给定的时间内完成，
你需要一些工具来支撑这样的保证。
就我们所知，还没有出现这样的工具（至少对大多数程序员没有）。
* 异常处理的运行时支持耗费过多空间
小型（通常为嵌入式）系统中可能如此。
不过在放弃异常之前，请考虑采用统一的利用错误码的错误处理将耗费的空间有多少，
以及错误未被捕获将造成的损失由多少。

许多（可能是大多数）的和异常有关的问题都源自于需要和杂乱的老代码进行交互的历史性原因。

而支持使用异常的基本论点是：

* 它们把错误返回和普通返回进行了清晰的区分
* 它们无法被忘记或忽略
* 它们可以系统化地使用

请记住

* 异常是用于报告错误的（C++ 中；其他语言可能有异常的不同用法）。
* 异常不是用于可以局部处理的错误的。
* 不要试图在每个函数中捕获每一种异常（这样做是冗长的，臃肿的，而且会导致代码缓慢）。
* 异常不是用于那些当发生无法恢复的错误之后需要立即终止模块或系统的错误的。

##### 示例

    ???

##### 替代方案

* [RAII](#Re-raii)
* 契约/断言：使用 GSL 的 `Expects` 和 `Ensures`（直到对契约的语言支持可以使用）

### <a name="Rnr-lots-of-files"></a>NR.4: 请勿坚持把每个类声明放在其自己的源文件中

##### 理由

将每个类都放进其自己的文件所导致的文件数量难于管理，并会拖慢编译过程。
单个的类很少是一种良好的维护和发布的逻辑单位。

##### 示例

    ???

##### 替代方案

* 使用命名空间来包含逻辑上聚合的类和函数。

### <a name="Rnr-two-phase-init"></a>NR.5: 请勿采用两阶段初始化

##### 理由

将初始化拆分为两步会导致不变式的弱化，
更复杂的代码（必须处理半构造对象），
以及错误（当未能一致地正确处理半构造对象时）。

##### 示例，不好

    class Picture
    {
        int mx;
        int my;
        char * data;
    public:
        Picture(int x, int y)
        {
            mx = x,
            my = y;
            data = nullptr;
        }
    
        ~Picture()
        {
            Cleanup();
        }
    
        bool Init()
        {
            // 不变式检查
            if (mx <= 0 || my <= 0) {
                return false;
            }
            if (data) {
                return false;
            }
            data = (char*) malloc(mx*my*sizeof(int));
            return data != nullptr;
        }
    
        void Cleanup()
        {
            if (data) free(data);
            data = nullptr;
        }
    };
    
    Picture picture(100, 0); // 此时 picture 尚未就绪可用
    // 这里将失败
    if (!picture.Init()) {
        puts("Error, invalid picture");
    }
    // 现在有一个无效的 picture 对象实例。

##### 示例，好

    class Picture
    {
        size_t mx;
        size_t my;
        vector<char> data;
    
        static size_t check_size(size_t s)
        {
            // 不变式检查
            Expects(s > 0);
            return s;
        }
    
    public:
        // 更好的方式是以一个 2D 的 Size 类作为单个形参
        Picture(size_t x, size_t y)
            : mx(check_size(x))
            , my(check_size(y))
            // 现在已知 x 和 y 为有效的大小
            , data(mx * my * sizeof(int)) // 出错时将抛出 std::bad_alloc
        {
            // 图片就绪可用
        }
        // 编译器生成的析构函数会完成工作。（另见 C.21）
    };
    
    Picture picture1(100, 100);
    // picture 已就绪可用……
    
    // y 并非有效大小值，
    // 缺省的契约违规行为将会调用 std::terminate
    Picture picture2(100, 0);
    // 不会抵达这里……

##### 替代方案

* 始终在构造函数中建立类不变式。
* 不要在需要对象之前就定义它。

### <a name="Rnr-goto-exit"></a>NR.6: 请勿把所有清理操作放在函数末尾并使用 `goto exit`

##### 理由

`goto` 是易错的。
这种技巧是进行 RAII 式的资源和错误处理的前异常时代的技巧。

##### 示例，不好

    void do_something(int n)
    {
        if (n < 100) goto exit;
        // ...
        int* p = (int*) malloc(n);
        // ...
        if (some_error) goto_exit;
        // ...
    exit:
        free(p);
    }

请找出其中的 BUG。

##### 替代方案

* 使用异常和 [RAII](#Re-raii)
* 对于非 RAII 资源，使用 [`finally`](#Re-finally)。

### <a name="Rnr-protected-data"></a>NR.7: 请勿使所有数据成员 `protected`

##### 理由

`protected` 数据是一种错误来源。
`protected` 数据可以被各种地方的无界限数量的代码所操纵。
`protected` 数据是在类层次中等价于全局对象的东西。

##### 示例

    ???

##### 替代方案

* [使成员数据 `public` 或者（更好地）`private`](#Rh-protected)。


# <a name="S-references"></a>RF: 参考材料

已经为 C++，尤其是对 C++ 的使用编写过了许多的编码标准、规则和指导方针。
它们中许多都

* 关注的是低级问题，比如标识符的拼写
* 是由 C++ 的新手编写的
* 将"禁止程序员作出不常见行为"作为其首要目标
* 将维持许多编译器的可移植性作为目标（有些已经是 10 年前的了）
* 是为了维持好几十年的代码库而编写的
* 是仅关注单一的应用领域的
* 只会产生反效果
* 被忽略了（程序员为了完成工作不得不忽略它们）

不良的编码标准要比没有编码标准还要差。
不过一组恰当的指导方针比没有标准要好得多："形式即解放。"

我们为什么不能有一种允许所有我们想要的同时又禁止所有我们不期望的东西的语言（"完美的语言"）呢？
本质上说，这是由于可负担的语言（及其工具链）同时也要为那些需求与你不同的人提供服务，并且要为你今后比今天更多的需求提供服务。
而且，你的需求会随时间而改变，而为此你则需要采用一种通用语言。
今日貌似理想的语言在未来可能会变得过于受限了。

编码指导方针可以使语言能够适应于特定的需求。
因此，并不存在适用于每个人的单一编码风格。
我们预计不同的组织会提供更具限制性和更严格的编码风格附加规定。

参考材料部分：

* [RF.rules: 编码规则](#SS-rules)
* [RF.books: 带有编码指导方针的书籍](#SS-books)
* [RF.C++: C++ 编程 (C++11/C++14/C++17)](#SS-Cplusplus)
* [RF.web: 网站](#SS-web)
* [RS.video: 有关"当代 C++"的视频](#SS-vid)
* [RF.man: 手册](#SS-man)
* [RF.core: 核心指导方针相关材料](#SS-core)

## <a name="SS-rules"></a>RF.rules: 编码规则

* [Boost Library Requirements and Guidelines](http://www.boost.org/development/requirements.html).
  ???.
* [Bloomberg: BDE C++ Coding](https://github.com/bloomberg/bde/wiki/CodingStandards.pdf).
  着重强调了代码的组织和布局。
* Facebook: ???
* [GCC Coding Conventions](https://gcc.gnu.org/codingconventions.html).
  C++03 以及（相当）一部分向后兼容。
* [Google C++ Style Guide](https://google.github.io/styleguide/cppguide.html).
  面向 C++03 和（同样）较老的代码库。Google 的专家们现在正展开活跃的合作，以改进这里的各项指导方针，有希望能够合并这些成果，以使它们能够成为他们也同样推荐采纳的一组现代的通用指导方针。
* [JSF++: JOINT STRIKE FIGHTER AIR VEHICLE C++ CODING STANDARDS](http://www.stroustrup.com/JSF-AV-rules.pdf).
  文档编号 2RDU00001 Rev C. December 2005.
  针对飞行控制软件。
  针对硬实时。
  这意味着它需要非常多的限制（"程序如果发生故障就会有人挂掉"）。
  例如，飞机起飞后禁止进行自由存储的分配和回收（禁止内存溢出并禁止发生碎片化）。
  禁止使用异常（因为没有可用工具可以保证异常能够在固定的短时间段内被处理）。
  所使用的程序库必须是已被证明可以用于关键任务应用的。
  它和这个指导方针集合的相似性并不让人惊讶，因为 Bjarne Stroustrup 正是 JSF++ 的作者之一。
  建议采纳，但请注意其非常特定的关注领域。
* [Mozilla Portability Guide](https://developer.mozilla.org/en-US/docs/Mozilla/C%2B%2B_Portability_Guide).
  如其名称所示，它关注于跨许多（老）编译器的兼容性。
  因此，它是很具有限制性的。
* [Geosoft.no: C++ Programming Style Guidelines](http://geosoft.no/development/cppstyle.html).
  ???.
* [Possibility.com: C++ Coding Standard](http://www.possibility.com/Cpp/CppCodingStandard.html).
  ???.
* [SEI CERT: Secure C++ Coding Standard](https://www.securecoding.cert.org/confluence/pages/viewpage.action?pageId=637).
  针对安全关键代码所编写的一组非常好的规则（还带有示例和原理说明）。
  它们的许多规则都广泛适用。
* [High Integrity C++ Coding Standard](http://www.codingstandard.com/).
* [llvm](http://llvm.org/docs/CodingStandards.html).
  有些简略，前 C++11 时代的，而且是（有理由地）针对其应用领域的。
* ???

## <a name="SS-books"></a>RF.books: 带有编码指导方针的书籍

* [Meyers96](#Meyers96) Scott Meyers: *More Effective C++*. Addison-Wesley 1996.
* [Meyers97](#Meyers97) Scott Meyers: *Effective C++, Second Edition*. Addison-Wesley 1997.
* [Meyers01](#Meyers01) Scott Meyers: *Effective STL*. Addison-Wesley 2001.
* [Meyers05](#Meyers05) Scott Meyers: *Effective C++, Third Edition*. Addison-Wesley 2005.
* [Meyers15](#Meyers15) Scott Meyers: *Effective Modern C++*. O'Reilly 2015.
* [SuttAlex05](#SuttAlex05) Sutter and Alexandrescu: *C++ Coding Standards*. Addison-Wesley 2005. 与其说是一组规则，不如说是一组元规则。前 C++11 时代。
* [Stroustrup05](#Stroustrup05) Bjarne Stroustrup: [A rationale for semantically enhanced library languages](http://www.stroustrup.com/SELLrationale.pdf).
  LCSD05. October 2005.
* [Stroustrup14](#Stroustrup05) Stroustrup: [A Tour of C++](http://www.stroustrup.com/Tour.html).
  Addison Wesley 2014.
  每章的结尾都有一个包含一组建议的忠告部分。
* [Stroustrup13](#Stroustrup13) Stroustrup: [The C++ Programming Language (4th Edition)](http://www.stroustrup.com/4th.html).
  Addison Wesley 2013.
  每章的结尾都有一个包含一组建议的忠告部分。
* Stroustrup: [Style Guide](http://www.stroustrup.com/Programming/PPP-style.pdf)
  for [Programming: Principles and Practice using C++](http://www.stroustrup.com/programming.html).
  大多是一些低级的命名和代码布局规则。
  主要作为教学工具。

## <a name="SS-Cplusplus"></a>RF.C++: C++ 编程 (C++11/C++14)

* [TC++PL4](http://www.stroustrup.com/4th.html):
  面向有经验的程序员的，对 C++ 语言和标准库的全面彻底的描述。
* [Tour++](http://www.stroustrup.com/Tour.html):
  面向有经验的程序员的，对 C++ 语言和标准库的简介。
* [Programming: Principles and Practice using C++](http://www.stroustrup.com/programming.html):
  面向初学者和新手们的教材。

## <a name="SS-web"></a>RF.web: 网站

* [isocpp.org](https://isocpp.org)
* [Bjarne Stroustrup 的个人主页](http://www.stroustrup.com)
* [WG21](http://www.open-std.org/jtc1/sc22/wg21/)
* [Boost](http://www.boost.org)<a name="Boost"></a>
* [Adobe open source](http://www.adobe.com/open-source.html)
* [Poco libraries](http://pocoproject.org/)
* Sutter's Mill?
* ???

## <a name="SS-vid"></a>RS.video: 有关"当代 C++"的视频

* Bjarne Stroustrup: [C++11?Style](http://channel9.msdn.com/Events/GoingNative/GoingNative-2012/Keynote-Bjarne-Stroustrup-Cpp11-Style). 2012.
* Bjarne Stroustrup: [The Essence of C++: With Examples in C++84, C++98, C++11, and?C++14](http://channel9.msdn.com/Events/GoingNative/2013/Opening-Keynote-Bjarne-Stroustrup). 2013
* [CppCon '14](https://isocpp.org/blog/2014/11/cppcon-videos-c9) 的全部演讲
* Bjarne Stroustrup: [The essence of C++](https://www.youtube.com/watch?v=86xWVb4XIyE) 在爱丁堡大学。2014
* Bjarne Stroustrup: [The Evolution of C++ Past, Present and Future](https://www.youtube.com/watch?v=_wzc7a3McOs). CppCon 2016 keynote.
* Bjarne Stroustrup: [Make Simple Tasks Simple!](https://www.youtube.com/watch?v=nesCaocNjtQ). CppCon 2014 keynote.
* Bjarne Stroustrup: [Writing Good C++14](https://www.youtube.com/watch?v=1OEu9C51K2A). CppCon 2015 keynote about the Core Guidelines.
* Herb Sutter: [Writing Good C++14... By Default](https://www.youtube.com/watch?v=hEx5DNLWGgA). CppCon 2015 keynote about the Core Guidelines.
* CppCon 15
* ??? C++ Next
* ??? Meting C++
* ??? more ???

## <a name="SS-man"></a>RF.man: 手册

* ISO C++ Standard C++11.
* ISO C++ Standard C++14.
* [ISO C++ Standard C++17](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2016/n4606.pdf). 委员会草案。
* [Palo Alto "Concepts" TR](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2012/n3351.pdf).
* [ISO C++ Concepts TS](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2015/n4553.pdf).
* [WG21 Ranges report](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2016/n4569.pdf). 草案。


## <a name="SS-core"></a>RF.core: 核心指导方针相关材料

这个部分包含一些用于展示核心指导方针及其背后的思想的有用材料：

* [Our documents directory](https://github.com/isocpp/CppCoreGuidelines/tree/master/docs)
* Stroustrup, Sutter, and Dos Reis: [A brief introduction to C++'s model for type- and resource-safety](http://www.stroustrup.com/resource-model.pdf). A paper with lots of examples.
* Sergey Zubkov: [a Core Guidelines talk](https://www.youtube.com/watch?v=DyLwdl_6vmU)
and here are the [slides](http://2017.cppconf.ru/talks/sergey-zubkov). In Russian. 2017.
* Neil MacIntosh: [The Guideline Support Library: One Year Later](https://www.youtube.com/watch?v=_GhNnCuaEjo). CppCon 2016.
* Bjarne Stroustrup: [Writing Good C++14](https://www.youtube.com/watch?v=1OEu9C51K2A). CppCon 2015 keynote.
* Herb Sutter: [Writing Good C++14... By Default](https://www.youtube.com/watch?v=hEx5DNLWGgA). CppCon 2015 keynote.
* Peter Sommerlad: [C++ Core Guidelines - Modernize your C++ Code Base](https://www.youtube.com/watch?v=fQ926v4ZzAM). ACCU 2017.
* Bjarne Stroustrup: [No Littering!](https://www.youtube.com/watch?v=01zI9kV4h8c). Bay Area ACCU 2016.
It gives some idea of the ambition level for the Core uidelines.

CppCon 的展示的幻灯片是可以获得的（其链接，还有上传的视频）。

极大欢迎对于这个列表的贡献。

## <a name="SS-ack"></a>鸣谢

感谢对规则、建议、支持信息和参考材料等等作出了各种贡献的许多人：

* Peter Juhl
* Neil MacIntosh
* Axel Naumann
* Andrew Pardoe
* Gabriel Dos Reis
* Zhuang, Jiangang (Jeff)
* Sergey Zubkov

请查看 github 的贡献者列表。

# <a name="S-profile"></a>Pro: 剖面配置

理想情况，我们应当遵循所有这些指导方针。
这样能够得到最简洁，最规范，最少易错性，而且通常是最快的代码。
不幸的是这通常是不可能的，因为我们不得不让代码适合于大型的代码库并使用一些现存的程序库。
常常是，这样的代码已经编写好几十年了，并且并不遵循这些指导方针。
我们必须以[渐次采纳](#S-modernizing)为目标。

无论采用何种渐次采纳的策略，我们都应当能够首先采用一些相关指导方针的集合来
处理某些问题的集合，遗留其他的以后处理。
当发现某些而不是全部的指导方针对于代码库有关时，以及当在某个专门化的应用领域
采用一组专门化的指导方针的集合时，会出现类似的"相关指导方针"的主意。
我们称这样的相关指导方针的集合为一个"剖面配置"。
我们对这种指导方针集合的目标是其内聚性，它们一起可以有助于我们达成某个特定的目标，如"消除范围错误"
或"静态类型安全性"。
每个剖面配置都被设计用于消除一个类别的错误。
而"随意"实施一些独立的规则，相对于提供确定的改善来说，更像是对代码库的破坏。

"剖面配置"是确定的并且可移植实施的规则（限制）子集，它们是专门设计以达成某项特定保证的。
"确定性"意味着它们仅需要进行局部分析，并且可以在一台计算机中进行实现（虽然并不比如此）。
"可移植实施性"表明它们和语言规则相似，因而程序员们可以期望不同实施工具对于相同的代码给出相同的答案。

编写成在这样的语言剖面配置下仍免于警告的代码，可以认为是遵循这个剖面配置的。
而遵从的代码则可以认为对于该剖面配置的目标安全属性来说是安全的。
遵从的代码不会成为这种性质的错误的根本原因，
虽然程序中可能从其他的代码引入这样的错误。
一个剖面配置还可能会引入一些额外的库类型，以简化遵从性并鼓励编写正确的代码。

剖面配置概览：

* [Pro.type: 类型安全性](#SS-type)
* [Pro.bounds: 边界安全性](#SS-bounds)
* [Pro.lifetime: 生存期安全性](#SS-lifetime)

未来，我们打算定义更多的剖面配置，并向现有剖面配置中添加更多的检查。
候选者有：

* 窄化算术提升和转换（可能会成为一个单独的安全算术剖面配置的一部分）
* 从负浮点数向无符号整型类型进行算术强制转换（同上）
* 经选择的未定义行为：从 Gabriel Dos Reis 为 WG21 研究小组开发的 UB 列表入手
* 经选择的未指明行为：处理可移植性问题。
* `const` 违反：大多数情况已经由编译器完成，但我们可以捕捉不适当的强制转换和 `const` 的不当使用。

剖面配置的开启是由实现所定义的；典型情况下，是在分析工具之中进行的设置。

要抑制对某个剖面配置检查，可以在语言构造上放一个 `suppress` 标注。例如：

    [[suppress(bounds)]] char* raw_find(char* p, int n, char x)    // 在 p[0]..p[n - 1] 中寻找 x
    {
        // ...
    }

这样 `raw_find()` 就可以在内存中到处爬了。
显然，进行抑制应当是非常罕见的。

## <a name="SS-type"></a>Pro.safety: 类型安全性剖面配置

这个剖面配置将能够简化正确使用类型的代码编写，并避免因疏忽产生类型双关。
它是关注于移除各种主要的类型违例的因素（包括对强制转换和联合的不安全使用）而达成这点的。

针对本部分的目的而言，
类型安全性被定义为这样的性质：对变量的使用不会不遵守其所定义的类型的规则。
通过类型 `T` 所访问的内存，不应该是某个实际上包含了无关类型 `U` 的对象的有效内存。
注意，当和[边界安全性](#SS-bounds)、[生存期安全性](#SS-lifetime)组合起来时，安全性才是完整的。

这个剖面配置的实现应当在源代码中识别出下列模式，将之作为不符合并给出诊断信息。

类型安全性剖面配置概览：

* <a name="Pro-type-avoidcasts"></a>Type.1: [避免强制转换](#Res-casts)：
<a name="Pro-type-reinterpretcast">a. </a>请勿使用 `reinterpret_cast`；此为[避免强制转换](#Res-casts)和[优先使用具名的强制转换](#Res-casts-named)的严格的版本。  
<a name="Pro-type-arithmeticcast">b. </a>请勿在算术类型上使用 `static_cast`；此为[避免强制转换](#Res-casts)和[优先使用具名的强制转换](#Res-casts-named)的严格的版本。  
<a name="Pro-type-identitycast">c. </a>当源指针类型和目标类型相同时，请勿进行指针强制转换；此为[避免强制转换](#Res-casts)的严格的版本。  
<a name="Pro-type-implicitpointercast">d. </a>当指针转换可以隐式转换时，请勿使用指针强制转换；此为[避免强制转换](#Res-casts)的严格的版本。  
* <a name="Pro-type-downcast"></a>Type.2: 请勿使用 `static_cast` 进行向下强制转换：
[代之以使用 `dynamic_cast`](#Rh-dynamic_cast)。
* <a name="Pro-type-constcast"></a>Type.3: 请勿使用 `const_cast` 强制掉 `const`（亦即不要这样做）：
[不要强制掉 `const`](#Res-casts-const)。
* <a name="Pro-type-cstylecast"></a>Type.4: 请勿使用  C 风格的强制转换 `(T)expression` 和函数式风格强制转换 `T(expression)`：
优先使用[构造语法](#Res-construct)和[具名的强制转换](#Res-casts-named)。
* <a name="Pro-type-init"></a>Type.5: 请勿在初始化之前使用变量：
[坚持进行初始化](#Res-always)。
* <a name="Pro-type-memberinit"></a>Type.6: 坚持初始化成员变量：
[坚持进行初始化](#Res-always)，
可以采用[默认构造函数](#Rc-default0)或者
[默认成员初始化式](#Rc-in-class-initializer)。
* <a name="Pro-type-unon"></a>Type.7: 避免裸 union：
[代之以使用 `variant`](#Ru-naked)。
* <a name="Pro-type-varargs"></a>Type.8: 避免 varargs：
[不要使用 `va_arg` 参数](#F-varargs)。

##### 影响

在类型安全性剖面配置下，你可以相信每个操作都将在有效的对象上进行。
可能抛出异常以报告无法（在编译时）被静态地检测到的错误。
要注意的是，这种类型安全性仅当我们同样具有[边界安全性](#SS-bounds)和[生存期安全性](#SS-lifetime)时才是完整的。
而没有这些保证的话，一个内存区域可能以与其所存储的单个或多个对象，或对象的一部分无关的方式被访问。


## <a name="SS-bounds"></a>Pro.bounds: 边界安全性剖面配置

这个剖面配置将能简化对于在分配的内存块的边界之中进行操作的编码工作。
它是通过关注于移除边界违例的主要根源——即指针算术和数组索引——而做到这点的。
这个剖面配置的核心功能之一就是限制指针只能指向单个对象而不是数组。

我们将边界安全性定义为这样一种性质：程序不通过一个对象来对分配给这个对象的内存范围之外的内存进行访问。
仅当边界安全性与[类型安全性](#SS-type)和[生存期安全性](#SS-lifetime)组合起来时才是完整的，
它们还会包含其他允许发生边界违例的不安全操作。

边界安全性剖面配置概览：

* <a name="Pro-bounds-arithmetic"></a>Bounds.1: 请勿使用指针算术。请使用 `span` 代替：
[（仅）传递单个对象的指针](#Ri-array)，并[保持指针算术的简单性](#Res-ptr)。
* <a name="Pro-bounds-arrayindex"></a>Bounds.2: 仅使用常量表达式对数组进行索引操作：
[（仅）传递单个对象的指针](#Ri-array)，并[保持指针算术的简单性](#Res-ptr)。
* <a name="Pro-bounds-decay"></a>Bounds.3: 避免数组向指针的衰变：
[（仅）传递单个对象的指针](#Ri-array)，并[保持指针算术的简单性](#Res-ptr)。
* <a name="Pro-bounds-stdlib"></a>Bounds.4: 请勿使用不进行边界检查的标准库函数和类型：
[以类型安全的方式使用标准库](#Rsl-bounds)

##### 影响

边界安全性意味着，当访问对象（尤其是数组）时不会越过对象的内存分配范围。
这消除了一大类的隐伏且难于发现的错误，包括（不）著名的"缓冲区溢出"错误。
这避免了安全漏洞，以及（当越界写入时发生）内存损坏错误的大量来源。
即使越界访问"只是读取操作"，它也可能导致不变式的违反（当所访问的不是预期的类型时）
和"神秘的值"。


## <a name="SS-lifetime"></a>Pro.lifetime: 生存期安全性剖面配置

通过已经不指向任何东西的指针进行访问，是错误的一种主要来源，
而且在许多传统的 C 或 C++ 风格的编程中这很难避免。
例如，指针可能未初始化，值为 `nullptr`，指向越过指针范围，或者指向已删除的对象。

[参见此处的设计说明书的当前版本](https://github.com/isocpp/CppCoreGuidelines/blob/master/docs/Lifetime.pdf)

生存期安全性剖面配置概览：

* <a name="Pro-lifetime-invalid-deref"></a>Lifetime.1: 不要解引用无效指针：
[检测或避免](#Res-deref)。

##### 影响

一旦强制实施了编码风格规则，静态分析，以及程序库支持的组合方案之后，本剖面配置将能

* 消除 C++ 中的恶劣错误的一种主要来源
* 消除潜在安全漏洞的一种主要来源
* 通过消除多余的"偏执"检查而改善性能
* 提升代码正确性的信心
* 通过强制遵循一种关键的 C++ 语言规则而避免未定义的行为


# <a name="S-gsl"></a>GSL: 指导方针支持库

GSL 是一个小型的程序库，其中的设施被设计用于支持本指导方针。
不使用这些设施的话，这些指导方针不得不变得对语言细节过于限制。

核心指导方针支持库是定义在 `gsl` 命名空间中的，其中的名字可能是对标准库和其他著名程序库的名字的别名。通过 `gsl` 命名空间进行的（编译期）间接，使我们可以进行试验，以及对这些支持设施提供局部变体。

GSL 只有头文件，可以在 [GSL: 指导方针支持库](https://github.com/Microsoft/GSL)找到。
支持库中的设施被设计为极为轻量化（零开销），它们相比于传统方案并不会带来任何开销。
当需要时，它们还可以用其他功能"工具化"（比如一些检查）来帮助进行诸如调试等任务。

这里的指导方针假定有一个 `variant` 类型。
最终，请使用[通过表决进入 C++17 的版本](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2016/p0088r3.html)。

GSL 组件概览：

* [GSL.view: 视图](#SS-views)
* [GSL.owner](#SS-ownership)
* [GSL.assert: 断言](#SS-assertions)
* [GSL.util: 工具](#SS-utilities)
* [GSL.concept: 概念](#SS-gsl-concepts)

我们计划提供一个"ISO C++ 标准风格的"半正式的 GSL 规范。

我们依赖于 ISO C++ 标准库，并希望 GSL 的一些部分能够被吸收到标准库之中。

## <a name="SS-views"></a>GSL.view: 视图

这些类型使用户可以区分带有和没有所有权的指针，并区分指向单个对象的指针和指向序列的第一个元素的指针。

"视图"都不是所有者。

引用都不是所有者（参见 [R.4](#Rr-ref)）。注意：有许多机会能让引用存活超过其所指代的对象，如按引用返回局部变量，持有 vector 的某个元素的引用然后进行 `push_back`，绑定到  `std::max(x, y + 1)`，等等。生存期安全性剖面配置的目标就是处理这些事情，但即便如此 `owner<T&>` 也没有意义且不建议使用。

它们的名字基本上遵循 ISO 标准库风格（小写字母和下划线）：

* `T*`      // `T*` 不是所有者，可能为 null；假定为指向单个元素。
* `T&`      // `T&` 不是所有者，不可能为"null 引用"；引用总是绑定到对象上。

"原生指针"写法（如 `int*`）假定为具有其最常见的含义；亦即指向一个对象的指针，但并不拥有它。
所有者应当被转换为资源包装（如 `unique_ptr` 或 `vector<T>`），或标为 `owner<T*>`。

* `owner<T*>`   // `T*`，拥有所指向/指代的对象；可能为 `nullptr`。

`owner` 用于对代码中有所有权的指针进行标记，它们无法更改为使用适当的资源包装。
其原因可能包括：

* 转换的成本。
* 需要为某个 ABI 使用指针。
* 这个指针时某种资源包装的实现的一部分。

`owner<T>` 和 `T` 的某种资源包装的区别在于它仍然需要明确进行 `delete`。

`owner<T>` 假定为指代自由存储（堆）上的某个对象。

当某个东西不应当为 `nullptr` 时，可以这样做：

* `not_null<T>`   // `T` 通常是某个指针类型（例如 `not_null<int*>` 和 `not_null<owner<Foo*>>`），且不能为 `nullptr`。
  `T` 可以是 `==nullptr` 有意义的任何类型。

* `span<T>`       // `[p:p+n)`，构造函数接受 `{p, q}` 和 `{p, n}`；`T` 为指针类型
* `span_p<T>`     // `{p, predicate}` `[p:q)`，其中 `q` 为首个使 `predicate(*p)` 为真的元素
* `string_span`   // `span<char>`
* `cstring_span`  // `span<const char>`

`span<T>` 指代零或更多可改动的 `T`，除非 `T` 为 `const` 类型。

"指针算术"最好在 `span` 之内进行。
指向多个 `char` 但并非 C 风格字符串的 `char*`（比如指向某个输入缓冲区的指针）应当表示为一个 `span`。

* `zstring`    // `char*`，假定为 C 风格字符串；亦即以零结尾的 `char` 的序列或者是 `nullptr`
* `czstring`   // `const char*`，假定为 C 风格字符串；亦即以零结尾的 `const` `char` 的序列或者是 `nullptr`

逻辑上来说，最后两种别名是没有必要的，但我们并不总是依照逻辑的，它们可以在指向单个 `char` 的指针和指向 C 风格字符串的指针之间明确地进行区分。
并未假定为零结尾的字符序列应当是 `char*`，而不是 `zstring`。
法文重音是可选的。

对于不能为 `nullptr` 的 C 风格字符串，应使用 `not_null<zstring>`。 ??? 我们需要为 `not_null<zstring>` 命名吗？还是说它的难看是有用的？

## <a name="SS-ownership"></a>GSL.owner: 所有权指针

* `unique_ptr<T>`     // 唯一所有权：`std::unique_ptr<T>`
* `shared_ptr<T>`     // 共享所有权：`std::shared_ptr<T>`（引用计数指针）
* `stack_array<T>`    // 栈分配数组。元素的数量在构造时确定并固定下来。其元素可改变，除非 `T` 为 `const` 类型。
* `dyn_array<T>`      // ??? 有必要吗 ??? 堆分配数组。元素的数量在构造时确定并固定下来。
  其元素可改变，除非 `T` 为 `const` 类型。基本上这是一个进行分配并拥有其元素的 `span`。

## <a name="SS-assertions"></a>GSL.assert: 断言

* `Expects`     // 前条件断言。当前放置于函数体内。今后应当移动到声明中。
                // `Expects(p)` 当不满足 `p == true` 时会终止程序
                // `Expect` 处于一组选项的控制之下（强制，错误消息，对终止程序的替代）
* `Ensures`     // 后条件断言。当前放置于函数体内。今后应当移动到声明中。

现在这些断言还是宏（天呐！）而且必须（只）被用在函数定义式之内。
等待标准委员会对于契约和断言语法的确定。
参见使用属性语法的[契约提案](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2016/p0380r1.pdf)，
比如说，`Expects(p)` 将变为 `[[expects: p]]`。

## <a name="SS-utilities"></a>GSL.util: 工具

* `finally`        // `finally(f)` 创建一个 `final_action{f}`，其析构函数将执行 `f`
* `narrow_cast`    // `narrow_cast<T>(x)` 就是 `static_cast<T>(x)`
* `narrow`         // `narrow<T>(x)` 在满足 `static_cast<T>(x) == x` 时为 `static_cast<T>(x)`，否则抛出 `narrowing_error`
* `[[implicit]]`   // 放在单参数构造函数上的"记号"，以明确说明它们并非显式构造函数。
* `move_owner`     // `p = move_owner(q)` 含义为 `p = q` 但 ???
* `joining_thread` // RAII 风格版本的进行联结的 `std::thread`
* `index`          // 用于进行所有的容器和数组索引的类型（当前是 `ptrdiff_t` 的别名）

## <a name="SS-gsl-concepts"></a>GSL.concept: 概念

这些概念（类型谓词）借用于
Andrew Sutton 的 Origin 程序库，
Range 提案，
以及 ISO WG21 的 Palo Alto TR。
它们可能会与 ISO C++ 标准中将会提供的概念十分相似。
它们的写法依照 ISO WG21 的 [Concepts TS](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2015/n4553.pdf)。
下列概念中的大多数都定义在 [Ranges TS](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2016/n4569.pdf) 中。

* `Range`
* `String`   // ???
* `Number`   // ???
* `Sortable`
* `Pointer`  // 带有 `*`，`->`，`==`，以及默认构造的类型（默认构造被假定为设值为唯一的"null"值）；参见[智能指针](#SS-gsl-smartptrconcepts)
* `Unique_pointer`  // 符合 `Pointer` 的类型，具有移动（而不是复制）操作，并符合生存期剖面配置中针对 `unique` 所有者类型的准则；参见[智能指针](#SS-gsl-smartptrconcepts)
* `Shared_pointer`   // 符合 `Pointer` 的类型，具有复制操作，并符合生存期剖面配置中针对 `shared` 所有者类型的准则；参见[智能指针](#SS-gsl-smartptrconcepts)
* `EqualityComparable`   // ???我们非得用 CaMelcAse 吗???
* `Convertible`
* `Common`
* `Boolean`
* `Integral`
* `SignedIntegral`
* `SemiRegular` // ??? Copyable?
* `Regular`
* `TotallyOrdered`
* `Function`
* `RegularFunction`
* `Predicate`
* `Relation`
* ...

### <a name="SS-gsl-smartptrconcepts"></a>GSL.ptr: 智能指针概念

参见 [Lifetime paper](https://github.com/isocpp/CppCoreGuidelines/blob/master/docs/Lifetime.pdf)。

# <a name="S-naming"></a>NL: 命名和代码布局规则

维持一致的命名和代码布局是很有用的。
即便不为其他原因，也可以减少"我的代码风格比你的好"这类的纷争。
然而，人们使用许多许多的不同代码风格，并狂热地坚持它们（的优缺点）。
而且，大多数的现实项目都包含来自于许多来源的代码，因而通常不可能把所有的代码都标准化为某个单一的代码风格。
经过许多的用户请求给予指导后，我们给出一组规则，当你没有更好的选择时可以使用它们，但真正的目标在于一致性，而不是任何一组特定的规则。
IDE 和工具可以提供辅助（当然也可能造成妨碍）。

命名和代码布局规则：

* [NL.1: 不要在代码注释中说明可以由代码来清晰表达的东西](#Rl-comments)
* [NL.2: 在代码注释中说明意图](#Rl-comments-intent)
* [NL.3: 保持代码注释简明干脆](#Rl-comments-crisp)
* [NL.4: 保持一种统一的缩进风格](#Rl-indent)
* [NL.5: 避免在名字中编码类型信息](#Rl-name-type)
* [NL.7: 使名字的长度大约正比于其作用域的长度](#Rl-name-length)
* [NL.8: 使用一种统一的命名风格](#Rl-name)
* [NL.9: 将 `ALL_CAPS`（全大写）仅用于宏的名字](#Rl-all-caps)
* [NL.10: 优先采用 `underscore_style`（下划线风格）的名字](#Rl-camel)
* [NL.11: 使字面量可阅读](#Rl-literals)
* [NL.15: 节制地使用空格](#Rl-space)
* [NL.16: 使用一种常规的类成员声明次序](#Rl-order)
* [NL.17: 使用从 K&R 衍生出的代码布局](#Rl-knr)
* [NL.18: 使用 C++ 风格的声明符布局](#Rl-ptr)
* [NL.19: 避免使用容易误读的名字](#Rl-misread)
* [NL.20: 不要把两个语句放在同一行中](#Rl-stmt)
* [NL.21: 每个声明式（仅）声明一个名字](#Rl-dcl)
* [NL.25: 请勿将 `void` 用作参数类型](#Rl-void)
* [NL.26: 采用符合惯例的 `const` 写法](#Rl-const)

这些问题的大部分都是审美问题，程序员都有很强的个人倾向。
IDE 也都会提供某些默认方案和一组替代方案。
这些规则是作为缺省建议的，如果没有别的理由，请采用它们。

我们收到一些意见称命名和代码布局非常个人化和任意性，我们不应该试图为之"立法"。
我们并不是在"立法"（参见前一个段落）。
不过，我们也收到了大量的针对某些命名和代码布局约定的请求，要求当没有外来限制的时候应当采用它们。

更专门和详细的规则更加易于强制实施。

这些规则恐怕会和 [PPP Style Guide](http://www.stroustrup.com/Programming/PPP-style.pdf) 中的建议有很强的相似性，
它是为支持 Stroustrup 的 [Programming: Principles and Practice using C++](http://www.stroustrup.com/programming.html) 而编制的。

### <a name="Rl-comments"></a>NL.1: 不要在代码注释中说明可以由代码来清晰表达的东西

##### 理由

编译器不会读注释。
注释没有代码那么精确。
注释不会像代码那样进行一致地更新。

##### 示例，不好

    auto x = m * v1 + vv;   // 将 m 乘以 v1 并将其结果加上 vv

##### 强制实施

构建一个 AI 程序来解释口语英文文字，看看它所说的是否可以用 C++ 来更好地表达。

### <a name="Rl-comments-intent"></a>NL.2: 在代码注释中说明意图

##### 理由

代码表示的是做了什么，而不是想要做成什么。通常来说意图比实现能够更清晰简明地进行说明。

##### 示例

    void stable_sort(Sortable& c)
        // 对 c 根据由 < 决定的顺序进行排序，保持相等元素（由 == 定义）的
        // 原始相对顺序
    {
        // ... 相当多的不平常的代码行 ...
    }

##### 注解

如果代码注释和代码有冲突，则它们都可能是错的。

### <a name="Rl-comments-crisp"></a>NL.3: 保持代码注释简明干脆

##### 理由

冗长啰嗦会拖慢理解速度，而且到处散布在代码文件里也会让代码难于阅读。

##### 注解

使用明白易懂的英文。
我可以流利使用丹麦语，但大多数程序员不行；我的代码的维护者也不行。
避免使用网络用语，注意你的文法，标点，以及大小写。
目标是专业性，而不是"够酷"。

##### 强制实施

不可能。

### <a name="Rl-indent"></a>NL.4: 保持一种统一的缩进风格

##### 理由

可读性。避免"微妙的错误"。

##### 示例，不好

    int i;
    for (i = 0; i < max; ++i); // 可能出现的 BUG
    if (i == j)
        return i;

##### 注解

总是把 `if (...)`，`for (...)`，以及 `while (...)` 之后的语句进行缩进是一个好主意：

    if (i < 0) error("negative argument");
    
    if (i < 0)
        error("negative argument");

##### 强制实施

使用一种工具。

### <a name="Rl-name-type"></a>NL.5: 避免在名字中编码类型信息

##### 原理

当名字反映类型而不是功能时，它将变得难于为提供其功能而改变其所使用的类型。
而且，当改变变量的类型时，使用它的代码也得修改。
最小化无意进行的转换。

##### 示例，不好

    void print_int(int i);
    void print_string(const char*);
    
    print_int(1);          // 重复，人工进行类型匹配
    print_string("xyzzy"); // 重复，人工进行类型匹配

##### 示例，好

    void print(int i);
    void print(string_view);    // 对任意字符串式的序列都能工作
    
    print(1);              // 简洁，自动类型匹配
    print("xyzzy");        // 简洁，自动类型匹配

##### 注解

带有类型编码的名字要么啰嗦要么难懂。

    printS  // 打印一个 std::string
    prints  // 打印一个 C 风格字符串
    printi  // 打印一个 int

在无类型语言中曾经采用过像匈牙利记法这样的技巧来在名字中编码类型，但在像 C++ 这样的强静态类型语言中，这通常是不必要而且实际上是有害的，因为这些标注会过时（这些累赘和注释类似，而且和它们一样会烂掉），而且它们干扰了语言的恰当用法（应当代之以使用相同的名字和重载决议）。

##### 注解

一些代码风格会使用非常一般性的（而不是特定于类型的）前缀来代表变量的一般用法。

    auto p = new User();
    auto p = make_unique<User>();
    // 注："p" 并非是说"User 类型的原始指针"，
    //     而只是一般性的"这是一次间接访问"
    
    auto cntHits = calc_total_of_hits(/*...*/);
    // 注："cnt" 并非用于编码某个类型，
    //     而只是一般性的"这是某种东西的一个计数"

这样做是没有害处的，且并不属于本条指导方针，因为其并未编码类型信息。

##### 注解

一些代码风格会对成员和局部变量，以及全局变量之间进行区分。

    struct S {
        int m_;
        S(int m) :m_{abs(m)} { }
    };

这样做是没有害处的，且并不属于本条指导方针，因为其并未编码类型信息。

##### 注解

像 C++ 这样，一些代码风格对类型和非类型之间进行区分。
例如，对类型名字首字母大写，而函数和变量名字则不这样做。

    typename<typename T>
    class HashTable {   // 将 string 映射为 T
        // ...
    };
    
    HashTable<int> index;

这样做是没有害处的，且并不属于本条指导方针，因为其并未编码类型信息。

### <a name="Rl-name-length"></a>NL.7: 使名字的长度大约正比于其作用域的长度

**原理**: 作用域越大，搞混的机会和意外的名字冲突的机会就越大。

##### 示例

    double sqrt(double x);   // 返回 x 的平方根；x 必须是非负数
    
    int length(const char* p);  // 返回零结尾的 C 风格字符串的字符数量
    
    int length_of_string(const char zero_terminated_array_of_char[])    // 不好: 啰嗦
    
    int g;      // 不好: 全局变量具有密秘的名字
    
    int open;   // 不好: 全局变量使用短小且常用的名字

为指针使用 `p`，以及为浮点变量使用 `x` 是符合惯例的，在受限的作用域中不会造成混乱。

##### 强制实施

???

### <a name="Rl-name"></a>NL.8: 使用一种统一的命名风格

**原理**: 命名和命名风格的一致性会提高可读性。

##### 注解

命名风格有好多，当你使用多个程序库时，你无法遵循所有它们不同的命名约定。
应当选用一种"自有风格"，但保持"导入"的程序为其原有风格不变。

##### 示例

ISO 标准仅使用小写字母和数字，并用下划线进行词的连接。

* `int`
* `vector`
* `my_map`

避免使用双下划线 `__`。

##### 示例

[Stroustrup](http://www.stroustrup.com/Programming/PPP-style.pdf)：
采用 ISO 标准，但在自己的类型和概念上采用大写字母：

* `int`
* `vector`
* `My_map`

##### 示例

CamelCase：多词标识符的每个词首字母大写：

* `int`
* `vector`
* `MyMap`
* `myMap`

一些命名约定会将首字母大写，而另一些不会。

##### 注解

应当试图在对缩略词的使用和标识符长度上保持一致风格。

    int mtbf {12};
    int mean_time_between_failures {12}; // 你自己决定

##### 强制实施

除使用具有不同命名约定的程序库之外应当是可能做到的。

### <a name="Rl-all-caps"></a>NL.9: 将 `ALL_CAPS`（全大写）仅用于宏的名字

##### 理由

避免在宏和遵循作用域和类型规则的名字之间造成混乱。

##### 示例

    void f()
    {
        const int SIZE{1000};  // 不好，应代之以 'size'
        int v[SIZE];
    }

##### 注解

这条规则适用于非宏的符号常量：

    enum bad { BAD, WORSE, HORRIBLE }; // 不好

##### 强制实施

* 对带有小写字母的宏进行标记
* 对 `ALL_CAPS` 非宏名字进行标记

### <a name="Rl-camel"></a>NL.10: 优先采用 `underscore_style`（下划线风格）的名字

##### 理由

用下划线来分隔名字的各部分就是 C 和 C++ 的原始风格，并被用于 C++ 标准库中。

##### 注解

这条规则仅作为当你有选择权时的缺省方案。
通常你是没有什么选择权的，而只能遵循某个已经设立的风格以维持[一致性](#Rl-name)。
对一致性的需要优先于个人喜好。

这个推荐适用于[当你没有约束条件或者没有更好的想法时](#S-naming)的情况。
经过很多要求给予指导后，添加这个规则。

##### 示例

[Stroustrup](http://www.stroustrup.com/Programming/PPP-style.pdf)：
采用 ISO 标准，但在自己的类型和概念上采用大写字母：

* `int`
* `vector`
* `My_map`

##### 强制实施

不可能。

### <a name="Rl-space"></a>NL.15: 节制地使用空格

##### 理由

太多的空格会让文本更大更分散。

##### 示例，不好

    #include < map >
    
    int main(int argc, char * argv [ ])
    {
        // ...
    }

##### 示例

    #include <map>
    
    int main(int argc, char* argv[])
    {
        // ...
    }

##### 注解

一些 IDE 有其自己的看法，并会添加分散的空格。

这个推荐适用于[当你没有约束条件或者没有更好的想法时](#S-naming)的情况。
经过很多要求给予指导后，添加这个规则。

##### 注解

我们将恰当放置的空白评价为能够明显有助于可读性。但请勿过度。

### <a name="Rl-literals"></a>NL.11: 使字面量可阅读

##### 理由

可读性。

##### 示例

用数字分隔符来避免长串的数字

    auto c = 299'792'458; // m/s2
    auto q2 = 0b0000'1111'0000'0000;
    auto ss_number = 123'456'7890;

##### 示例

需要清晰性时使用字面量后缀

    auto hello = "Hello!"s; // std::string
    auto world = "world";   // C 风格字符串
    auto interval = 100ms;  // 使用 <chrono>

##### 注解

不能在代码中到处当做["魔法常量"](#Res-magic)一样乱用字面量，
但当定义它们时使它们更可读仍是个好主意。
在较长的整数串中很容易出现拼写错误。

##### 强制实施

标记长数字串。麻烦的是"长"的定义；也许应当是 7。

### <a name="Rl-order"></a>NL.16: 使用一种常规的类成员声明次序

##### 理由

一种常规的成员次序会提高可读性。

以如下次序声明类

* 类型：类，枚举，别名（`using`）
* 构造函数，赋值，析构函数
* 函数
* 数据

采用先是 `public`，然后是 `protected`，之后是 `private` 的次序。

这个推荐适用于[当你没有约束条件或者没有更好的想法时](#S-naming)的情况。
经过很多要求给予指导后，添加这个规则。

##### 示例

    class X {
    public:
        // 接口
    protected:
        // 供派生类实现使用的不带检查的函数
    private:
        // 实现细节
    };

##### 示例

有时候，成员的默认顺序，与将公开接口从实现细节中分离出来的需求之间有冲突。
这种情况下，私有类型和函数可以和私有数据放在一起。

    class X {
    public:
        // 接口
    protected:
        // 供派生类实现使用的不带检查的函数
    private:
        // 实现细节（类型，函数和数据）
    };

##### 示例，不好

避免让具有某一种访问（如 `public`）的多个声明块被具有不同访问（如 `private`）的其他声明块分隔开。

    class X {
    public:
        void f();
    public:
        int g();
        // ...
    };

用宏来声明成员组的做法通常会导致违反所有的次序规则。
不过，宏的使用掩盖了其所表达的东西。

##### 强制实施

对背离上述建议次序的代码进行标记。将会有大量的老代码不符合这条规则。

### <a name="Rl-knr"></a>NL.17: 使用从 K&R 衍生出的代码布局

##### 理由

这正是 C 和 C++ 的原始代码布局。它很好地保持了纵向空间。它对不同语言构造（如函数和类）进行了很好的区分。

##### 注解

在 C++ 的语境中，这种风格通常被称为"Stroustrup"。

这个推荐适用于[当你没有约束条件或者没有更好的想法时](#S-naming)的情况。
经过很多要求给予指导后，添加这个规则。

##### 示例

    struct Cable {
        int x;
        // ...
    };
    
    double foo(int x)
    {
        if (0 < x) {
            // ...
        }
    
        switch (x) {
        case 0:
            // ...
            break;
        case amazing:
            // ...
            break;
        default:
            // ...
            break;
        }
    
        if (0 < x)
            ++x;
    
        if (x < 0)
            something();
        else
            something_else();
    
        return some_value;
    }

注意 `if` 和 `(` 之间有一个空格

##### 注解

每个语句，`if` 的分支，以及 `for` 的代码体都使用单独的代码行。

##### 注解

`class` 和 `struct` 的 `{` *并不*在单独的代码行上，但函数的 `{` 在单独的代码行上。

##### 注解

对你自定义的类型的名字进行首字母大写，以将其与标准库类型相区分。

##### 注解

不要对函数名大写。

##### 强制实施

如果想要强制实施的话，请使用某个 IDE 进行格式化。

### <a name="Rl-ptr"></a>NL.18: 使用 C++ 风格的声明符布局

##### 理由

C 风格的布局强调其在表达式中的用法和文法，而 C++ 风格强调的是类型。
对表达式用法的说辞并不适用于引用。

##### 示例

    T& operator[](size_t);   // OK
    T &operator[](size_t);   // 奇怪
    T & operator[](size_t);   // 不确定

##### 注解

这个推荐适用于[当你没有约束条件或者没有更好的想法时](#S-naming)的情况。
经过很多要求给予指导后，添加这个规则。

##### 强制实施

由于历史原因而不可能。


### <a name="Rl-misread"></a>NL.19: 避免使用容易误读的名字

##### 理由

可读性。
并非每个人都有能将字符轻易区分开的屏幕和打印机。
我们很容易搞混拼写相似和略微拼错的单词。

##### 示例

    int oO01lL = 6; // 不好
    
    int splunk = 7;
    int splonk = 8; // 不好：splunk 和 splonk 很容易搞混

##### 强制实施

???

### <a name="Rl-stmt"></a>NL.20: 不要把两个语句放在同一行中

##### 理由

可读性。
当一行里有多个语句时，相当容易忽视某个语句。

##### 示例

    int x = 7; char* p = 29;    // 请勿如此
    int x = 7; f(x);  ++x;      // 请勿如此

##### 强制实施

容易。

### <a name="Rl-dcl"></a>NL.21: 每个声明式（仅）声明一个名字

##### 理由

可读性。
最小化声明符语法造成的混乱。

##### 注解

相关细节，参见 [ES.10](#Res-name-one)、


### <a name="Rl-void"></a>NL.25: 请勿将 `void` 用作参数类型

##### 理由

这很啰嗦，而且仅在考虑 C 兼容性是才有必要。

##### 示例

    void f(void);   // 不好
    
    void g();       // 好多了

##### 注解

即便是 Dennis Ritchie 自己都认为 `void f(void)` 很讨厌。
你可以反驳称，在 C 中当函数原型很少见时禁止这样的代码：

    int f();
    f(1, 2, "weird but valid C89");   // 希望 f() 被定义为 int f(a, b, c) char* c; { /* ... */ }

可能造成很大的问题，但这并不适于 21 世纪和 C++。

### <a name="Rl-const"></a>NL.26: 采用符合惯例的 `const` 写法

##### 理由

更多程序员更加熟悉惯例写法。
大型代码库中的一致性。

##### 示例

    const int x = 7;    // OK
    int const y = 9;    // 不好
    
    const int *const p = nullptr;   // OK, 指向常量 int 的常量指针
    int const *const p = nullptr;   // 不好，指向常量 int 的常量指针

##### 注解

我们知道你可能会说"不好"的例子比标有"OK"的更符合逻辑，
但它们会让更多人搞混，尤其是那些依赖于采用了远为常用，符合惯例的 OK 风格的教学材料的新手们。

一如往常，请记住这些命名和代码布局规则的目标在于一致性，而审美则会有广泛的变化。

这个推荐适用于[当你没有约束条件或者没有更好的想法时](#S-naming)的情况。
经过很多要求给予指导后，添加这个规则。

##### 强制实施

标记用作类型的后缀的 `const`。

# <a name="S-faq"></a>FAQ: 常见问题及其回答

本节中包括了对关于这些指导方针的常见问题的回答。

### <a name="Faq-aims"></a>FAQ.1: 这些指导方针的想要达成什么目标？

请参见<a href="#S-abstract">本页面开头</a>。这是一个开源项目，旨在为采用当今的 C++ 标准（写此文时为 C++14）来编写 C++ 代码而维护的一组现代的权威指导方针。这些指导方针的设计是现代的，尽可能使机器可实施的，并且是为贡献和分支保持开放，以使各种组织机构可以便于将它们整合到其自己组织的编码指导方针之中。

### <a name="Faq-announced"></a>FAQ.2: 这项工作室何时何地首次公开的？

是在 [Bjarne Stroustrup 在他为 CppCon 2015 的开场主旨演讲，"Writing Good C++14"](https://isocpp.org/blog/2015/09/stroustrup-cppcon15-keynote)。另请参见[相应的 isocpp.org 博客条目](https://isocpp.org/blog/2015/09/bjarne-stroustrup-announces-cpp-core-guidelines)，关于类型和内存安全性指导方针的原理请参见 [Herb Sutter 的后续 CppCon 2015 演讲，"Writing Good C++14 ... By Default"](https://isocpp.org/blog/2015/09/sutter-cppcon15-day2plenary)。

### <a name="Faq-maintainers"></a>FAQ.3: 谁是这些指导方针的作者和维护者？

最初的主要作者和维护者是 Bjarne Stroustrup 和 Herb Sutter，而迄今为止的指导方针则是由来自 CERN，Microsoft，Morgan Stanley，以及许多其他组织机构的专家所贡献的。指导方针发布时，其正处于 "0.6" 状态，我们欢迎人们进行贡献。正如 Stroustrup 在其声明中所说："我们需要帮助！"

### <a name="Faq-contribute"></a>FAQ.4: 我如何进行贡献呢？

参见 [CONTRIBUTING.md](https://github.com/isocpp/CppCoreGuidelines/blob/master/CONTRIBUTING.md)。我们感激志愿者的帮助！

### <a name="Faq-maintainer"></a>FAQ.5: 怎样成为一名编辑或维护者？

通过先进行大量贡献并使你的贡献被认可具有一致的质量。参见 [CONTRIBUTING.md](https://github.com/isocpp/CppCoreGuidelines/blob/master/CONTRIBUTING.md)。我们感激志愿者的帮助！

### <a name="Faq-iso"></a>FAQ.6: 这些指导方针被 ISO C++ 标准委员会采纳了吗？它们是否代表委员会的一致意见？

不是这样。这些指导方针不在标准之内。它们是为标准服务的，而当前维护的指导方针是为了更有效地使用当前的标准 C++的。我们的目标是使其与委员会所设计的标准保持同步。

### <a name="Faq-isocpp"></a>FAQ.7: 既然这些指导方针并不是委员会所采纳的，它们为何在 `github.com/isocpp` 之下呢？

因为 `isocpp` 是标准 C++ 基金会；而标准委员会的仓库则处于 [github.com/*cplusplus*](https://github.com/cplusplus) 之下。我们需要一个中立组织来持有版权和许可以明确其并不是由某个人或供应商所控制的。这个自然实体就是基金会，其设立是为了推进使用并持续更新对现代标准 C++ 的理解，以及推进标准委员会的工作。其所遵循的正是与 isocpp.org 为 [C++ FAQ](https://isocpp.org/faq) 所做的相同模式，它是有 Bjarne Stroustrup，Marshall Cline，和 Herb Sutter 所发起的工作，并以相同的方式贡献为了开放项目。

### <a name="Faq-cpp98"></a>FAQ.8: 会有 C++98 版本的指导方针吗？C++11 版本呢？

不会。这些指导方针的目标是更好地使用标准 C++14（和 Concepts 技术规范，当你有一个实现的话），以及假定你有一个现代的遵循标准的编译器时如何进行代码编写的。

### <a name="Faq-language-extensions"></a>FAQ.9: 这些指导方针中会提出新的语言功能吗？

不会。这些指导方针的目标是更好地使用标准 C++14 和 Concepts 技术规范的，它们会自我限定为仅建议使用这些功能。

### <a name="Faq-markdown"></a>FAQ.10: 这些指导方针的书写使用的是哪个版本的 Markdown？

这些编码指导方针使用的是 [CommonMark](http://commonmark.org)，以及 `<a>` HTML 锚定元素。

我们正在考虑以下这些来自 [GitHub Flavored Markdown (GFM)](https://help.github.com/articles/github-flavored-markdown/) 的扩展：

* 有围栏代码块（正在讨论是否统一使用缩进还是围栏代码块）
* 表格（我们虽然还没用到，但很需要它们，这是一种 GFM 扩展）

避免使用其他 HTML 标签和其他扩展。

注意：我们还没对这种风格达成一致。

### <a name="Faq-gsl"></a>FAQ.50: 什么是 GSL（指导方针支持程序库）？

GSL 是在指导方针中所指定的类型和别名的一个小集合。当写下本文时，对它们的说明还过于松散；我们计划添加一个 WG21 风格的接口规范来确保不同实现之间保持一致，并作为一项可能的标准化提案，按常规遵循标准委员会进行采纳、改进、修订或否决。

### <a name="Faq-msgsl"></a>FAQ.51: [github.com/Microsoft/GSL](https://github.com/Microsoft/GSL) 是 GSL 吗？

不是。它只是由 Microsoft 所贡献的第一个实现。我们鼓励其他供应商提供其他的实现，对该实现的分支和贡献也是被鼓励的。书写本文作为一项公开项目的一周中，已经出现了至少一个 GPLv3 的开源实现。我们计划制定一个 WG21 风格的接口规范来确保不同实现之间保持一致。

### <a name="Faq-gsl-implementation"></a>FAQ.52: 为何不在指导方针之中提供一个真正的 GSL 实现呢？

我们不愿去保佑某个特定的实现，因为我们不希望让人们以为只有一个实现，而疏忽大意地扼杀了其他并行的实现。而如果在指导方针中包含一个真正实现的话，无论是谁提供了它都会变得过于有影响力。我们更倾向于采用委员会的更具长期性的方案，即指定其接口而不是实现。但同时我们也需要至少存在一个实现；希望可以有很多。

### <a name="Faq-boost"></a>FAQ.53: 为什么不把 GSL 类型提交给 Boost 呢？

因为我们想要立刻使用它们，也因为我们想要在一旦标准库中出现了满足其需要的类型时立刻将它们撤销掉。

### <a name="Faq-gsl-iso"></a>FAQ.54: ISO C++ 标准委员会采纳了 GSL（指导方针支持程序库）吗？

没有。GSL 的存在只为提供少量标准库中还没有的类型和别名。如果委员会决定了（这些类型或者满足其需要的其他类型的）标准化的版本，就可以将它们从 GSL 中删除了。

### <a name="Faq-gsl-string-view"></a>FAQ.55: 既然你是尽可能使用标准类型，为什么 GSL 的 `string_span` 同 Library Fundamentals 1 Technical Specification 和 C++17 工作文本中的 `string_view` 不同呢？为什么不使用委员会采纳的 `string_view`？

有关 C++ 标准库的视图的分类的统一观点是，"视图（view）"意味着"只读"，而"跨距（span）"意味着"可读写"。只读的 `string_view` 是第一个完成了标准化过程的这种组件，而 `span` 和 `string_span` 现在还在考虑如何标准化。

### <a name="Faq-gsl-owner"></a>FAQ.56: `owner` 和提案的 `observer_ptr` 一样吗？

不一样。`owner` 有所有权，它是一个别名，而且适用于任何间接类型。而 `observer_ptr` 的主要意图则是明确某个*没有*所有权的指针。

### <a name="Faq-gsl-stack-array"></a>FAQ.57: `stack_array` 和标准的 `array` 一样吗？

不一样。`stack_array` 保证在栈上分配。虽然 `std::array` 直接在其自身内部包含存储，但 `array` 对象可以放在包括堆在内的任何地方。

### <a name="Faq-gsl-dyn-array"></a>FAQ.58: `dyn_array` 和 `vector` 或者提案的 `dynarray` 一样吗？

不一样。`dyn_array` 是不可改变大小的，是一种指代堆分配的固定大小数组的一种安全方式。与 `vector` 不同，它是为了取代数组 `new[]` 的。与委员会中提案的 `dynarray` 不同，它并不会参与编译器和语言的魔法，来在当它作为分配于栈上的对象的成员时也在栈上分配；它只不过指代一个"动态的"或基于堆的数组而已。

### <a name="Faq-gsl-expects"></a>FAQ.59: `Expects` 和 `assert` 一样吗？

不一样。它是一种对于契约前条件语言支持的占位符。

### <a name="Faq-gsl-ensures"></a>FAQ.60: `Ensures` 和 `assert` 一样吗？

不一样。它是一种对于契约后条件语言支持的占位符。

# <a name="S-libraries"></a>附录 A: 程序库

这个部分列出了一些推荐的程序库，并且特别推荐了其中的几个。

??? 这个对一般性指南来说合适吗？我觉得不是 ???

# <a name="S-modernizing"></a>附录 B: 代码的现代化转换

理想情况下，我们的所有代码都应当遵循全部的规则。
而实际情况则是，我们不得不对付大量的老代码：

* 那些在我们的指导方针被建立或者被了解到之前所编写的代码
* 那些依据老的或者不同的标准所编写的程序库
* 那些在"不寻常"的约束下编写的代码
* 那些我们还未来得及使其现代化的代码

如果有上百万行的新代码的话，"立刻改掉它们"的想法一般都是不现实的。
因此，我们需要一种方式能够渐进地对代码库进行现代化转换。

将老代码升级为现代风格可能是很让人却步的工作。
老代码通常即混乱（难于理解）又可以（在当前使用范围内）正确工作。
很有可能，原来的程序员并不在场，而且测试用例也不完整。
代码的混乱性显著地增大了为进行任何改动所需要的工作量和引入错误的风险。
通常，混乱的老代码的运行会有不必要的缓慢，因为它需要过期的编译器，并且无法得益于当代硬件的改进。
许多情况下，都需要某种自动进行"现代化转换"的工具支持来进行主要的升级工作。

对代码现代化转换的目的在于简化新功能的添加，简化维护工作，以及增加性能（吞吐量或延迟），和更好地利用当代的硬件能力。
而让代码"看起来更好"或"遵循现代风格"自身并不能成为改动的理由。
每一种改动都蕴含着风险，而是由过时的代码库则会蕴含一些成本（包含丢失机会带来的成本）。
成本的缩减必须超过风险。

如何做呢？

并不存在唯一的代码现代化转换的方案。
如何最好地执行，依赖于代码，更新的进度压力，开发者的背景，以及可用的工具。
下面是一些（非常一般的）想法：

* 理想情况是"对全部代码一起进行升级"。这将在最短的总时间内获得做大的好处。
  在大多数情况下，这也是不可能的。
* 我们可以对代码库以模块为单位进行转换，不过任何影响接口（尤其是 ABI）的规则，如 [使用 `span`](#SS-views)，都无法按模块来达成。
* 我们可以"自底向上"转换代码，并最先应用我们估计在给定的代码库上将会带来最大好处和最少麻烦的那些规则。
* 我们可以从关注接口开始，比如说，保证没有资源的泄漏，没有指针误用等。
  这可能会导致涉及整个代码库的一些改动，不过它们是最可能会带来巨大好处的改动。
  以后，隐藏在这些接口后面的代码可以渐进地进行现代化转换而不会影响其他的代码。

无论你选择哪种方式，都要注意，对指导方针的最高度的遵循性才会带来大多数的好处。
这些指导方针并不是一组无关规则的随机集合，并不能让你随意选取并期望取得成功。

我们衷心希望听到有关它们的使用经验，以及有关工具是如何使用的。
如果有分析工具（即便是代码变换工具）的支持的话，代码现代化转换后可以变得更快，更简单，而且更安全。

# <a name="S-discussion"></a>附录 C: 讨论

这个部分包含了对规则和规则集合的跟进材料。
尤其是，我们列出了更多的原理说明，更长的例子，以及对替代方案的探讨等。

### <a name="Sd-order"></a>讨论: 以成员的声明顺序进行成员变量的定义和初始化

成员变量总是以它们在类定义中的声明顺序进行初始化，因此在构造函数初始化列表中应当以该顺序来书写它们。以别的顺序书写它们只会让代码混淆，因为它并不会以你所见到的顺序来运行，而这会导致难于发现与顺序有关的 BUG。

    class Employee {
        string email, first, last;
    public:
        Employee(const char* firstName, const char* lastName);
        // ...
    };
    
    Employee::Employee(const char* firstName, const char* lastName)
      : first(firstName),
        last(lastName),
        // 不好: first 和 last 还未构造
        email(first + "." + last + "@acme.com")
    {}

在这个例子中，`email` 比 `first` 和 `last` 构造得早，因为它是先声明的。这意味着其构造函数对 `first` 和 `last` 的使用过早了——不只在它们被设为所需的值之前，而完全是在它们被构造之前就使用了。

如果类定义和构造函数体是在不同文件中的话，这种由成员变量声明顺序对构造函数的正确性造成的远距离影响将更难于发现。

**参考**：

[\[Cline99\]](#Cline99) §22.03-11, [\[Dewhurst03\]](#Dewhurst03) §52-53, [\[Koenig97\]](#Koenig97) §4, [\[Lakos96\]](#Lakos96) §10.3.5, [\[Meyers97\]](#Meyers97) §13, [\[Murray93\]](#Murray93) §2.1.3, [\[Sutter00\]](#Sutter00) §47

### <a name="Sd-init"></a>讨论：使用 `=`，`{}`，和 `()` 作为初始化式

???

### <a name="Sd-factory"></a>讨论: 当需要在初始化过程中使用"虚函数行为"时，使用工厂函数

如果你的设计需要从基类的构造函数或析构函数中对 `f` 或者 `g` 这样的函数向派生类进行虚函数派发的话，你其实需要的是其他技巧，比如后构造函数——一种必须由调用者调用以完成初始化过程的成员函数，它可以安全地调用 `f` 和 `g`，这是由于成员函数中的虚函数调用能够正常工作。"参考"部分中列出了一些这样的技巧。以下是一个不完整的可选项列表：

* *推卸责任：* 仅仅给出文档说明，要求用户代码在对象构造之后必须立刻调用后初始化函数。
* *惰性后初始化：* 在第一个调用的成员函数中进行。用基类中的一个布尔标记说明后初始化是否已经执行过。
* *使用虚基类语义：* 语言规则要求由最终派生类的构造函数来决定调用哪个基类构造函数；你可以利用这点。（参见[\[Taligent94\]](#Taligent94)。）
* *使用工厂函数：* 以这种方式，你可以轻易确保进行对后构造函数的调用。

以下是对最后一种选项的一个例子：

    class B {
    public:
        B() {
            /* ... */
            f(); // 不好: C.82：不要在构造函数和析构函数中调用虚函数
            /* ... */
        }
    
        virtual void f() = 0;
    };
    
    class B {
    protected:
        class Token {};
    
    public:
        // 需要公开构造函数以使 make_shared 可以访问它。
        // 通过要求一个 Token 达成受保护访问等级。
        explicit B(Token) { /* ... */ }  // 创建不完全初始化的对象
        virtual void f() = 0;
    
        template<class T>
        static shared_ptr<T> create()    // 创建共享对象的接口
        {
            auto p = make_shared<T>(typename T::Token{});
            p->post_initialize();
            return p;
        }
    
    protected:
        virtual void post_initialize()   // 构造之后立即调用
            { /* ... */ f(); /* ... */ } // 好: 虚函数分派是安全的
    };


    class D : public B {                 // 某个派生类
    protected:
        class Token {};
    
    public:
        // 需要公开构造函数以使 make_shared 可以访问它。
        // 通过要求一个 Token 达成受保护访问等级。
        explicit D(Token) : B( B::Token{} ) {}
        void f() override { /* ... */ };
    
    protected:
        template<class T>
        friend shared_ptr<T> B::create();
    };
    
    shared_ptr<D> p = D::Create<D>();    // 创建一个 D 对象

这种设计需要遵守以下纪律：

* 像 `D` 这样的派生类不能暴露可公开调用的构造函数。否则的话，`D` 的使用者就能够创建 `D` 对象而不调用 `post_initialize` 了。
* 分配被限定为使用 `operator new`。不过，`B` 可以覆盖 `new`（参见 [SuttAlex05](#SuttAlex05) 条款 45 和 46）。
* `D` 必须定义一个带有与 `B` 所选择的相同的参数的构造函数。不过，定义多个重载的 `create` 可以缓和这个问题；而且还可以是这些重载对参数类型进行模板化。

一旦满足了上述要求，这个设计就可以保证对于任意完全构造的 `B` 的派生类对象，都将调用 `post_initialize`。`post_initialize` 不必是虚函数；它可以随意进行虚函数调用。

总之，不存在完美的后构造技巧。最差的方式是完全回避问题而只是让调用方来人工调用后构造函数。即便是最佳方案也需要采用一种不同的对象构造语法（易于进行编译期检查）以及需要派生类的作者的协作（这无法进行编译期进行检查）。

**参考**: [\[Alexandrescu01\]](#Alexandrescu01) §3, [\[Boost\]](#Boost), [\[Dewhurst03\]](#Dewhurst03) §75, [\[Meyers97\]](#Meyers97) §46, [\[Stroustrup00\]](#Stroustrup00) §15.4.3, [\[Taligent94\]](#Taligent94)

### <a name="Sd-dtor"></a>讨论: 基类的析构函数应当要么是 public 和 virtual，要么是 protected 且非 virtual

析构应不应该表现为虚函数？就是说，是否允许通过指向 `base` 类的指针来进行析构呢？如果是的话，`base` 的析构函数为被调用则必须是 public 的，而且必须 virtual，否则调用就会导致未定义行为。否则的话，它应当是 protected 的，这样就只有派生类可以在它们自己的析构函数中调用它，且应当是非 virtual 的，因为它并不需要表现为虚函数的行为。

##### 示例

基类的一般情况是为了具有 public 的派生类，因而调用方代码基本上可以确定要用到某种比如 `shared_ptr<base>` 这样的东西：

    class Base {
    public:
        ~Base();                   // 不好, 非 virtual
        virtual ~Base();           // 好
        // ...
    };
    
    class Derived : public Base { /* ... */ };
    
    {
        unique_ptr<Base> pb = make_unique<Derived>();
        // ...
    } // 只有当 ~Base 是虚函数时 ~pb 才会调用正确的析构函数

少数比如策略类这类的情况下，类被用作基类是为方便起见，而并非是其多态行为。建议将它们的析构函数作为 protected 和非 virtual 函数：

    class My_policy {
    public:
        virtual ~My_policy();      // 不好, public 并且 virtual
    protected:
        ~My_policy();              // 好
        // ...
    };
    
    template<class Policy>
    class customizable : Policy { /* ... */ }; // 注: private 继承

##### 注解

这个简单的指导方针演示了一种微妙的问题，而且反映了继承的现代用法以及面向对象的设计原则。

对于某个基类 `Base`，调用方代码可能通过指向 `Base` 的指针来销毁派生类对象，比如使用一个 `unique_ptr<Base>`。如果 `Base` 的析构函数是 public 的且非 virtual（默认情况），它就可能意外地在实际指向一个派生类对象的指针上进行调用，这种情况下想要进行的删除的行为是未定义的。这种状况层导致一些老编码标准提出通用的要求，让所有基类析构函数都必须 virtual。这中做法杀伤力过大了（虽然这是常见的情况）；其实，规则应当是当且仅当基类析构函数是 public 时才要求它是 virtual 的。

编写一个基类就是在定义一种抽象（参见条款 35 到 37）。注意对于参与这个抽象的每个成员函数来说，你都需要作出以下决定：

* 它是否应当表现为虚函数。
* 它是应当对所有使用 `Base` 指针的调用方公开，还是作为隐藏的内部实现细节。

如条款 39 中所述，对于普通成员函数来说，其选择可以是：允许通过 `Base` 指针对其进行非虚调用（但当它调用了虚函数时可具有虚行为，比如在 NVI 或者模板方法模式中那样），进行虚调用，或者完全不能调用。NVI 模式是一种避免公开虚函数的技巧。

析构可以仅仅被看做是另一种操作，虽然它带有特殊的语义，并且非虚调用要么很危险要么是错误的。因而，对于基类析构函数来说，其选择有：允许通过 `Base` 指针进行虚函数调用，或者完全不能调用；"非虚调用"是不行的。这样的话，基类析构函数当其可以被调用（即为 public）时应当是 virtual 的，否则就为非 virtual。

注意 NVI 模式并不适用于析构函数，因为构造函数和析构函数无法进行深析构调用。（参见条款 39 和 55。）

推论：当编写基类时，一定要明确编写析构函数，因为隐式生成的析构函数是 public 和非 virtual 的。当预置函数体没问题是你当然可以用 `=default`，而仅仅为其指定正确的可见性和虚函数性质即可。

##### 例外

某些组件体系架构（如 COM 和 CORBA）并不适用标准的删除机制，而是为对象的处置设立了不同的方案。请遵循相应的模式和惯用法，并在适当是采纳本条指导方针。

再考虑一下这种罕见情况：

* `B` 既是一个基类，也是可以被实例化的具体类，因而其析构函数必须为 public 以便 `B` 的对象可以创建和销毁。
* 而 `B` 也没有虚函数，且并不打算按多态方式使用，因此虽然其析构函数是 public 它也不必是 virtual 的。

这样的话，虽然析构函数必须为 public，也会有很强的压力来阻止它变为 virtual，因为作为第一个 virtual 函数，若其所添加的功能永远不会被利用的话，它就会损害所有的运行时类型开销。

在这种罕见情况下，可以是析构函数 public 且非 virtual，但要明确说明其派生类对象绝不能当作 `B` 来多态地使用。我们对 `std::unary_function` 正是这样做的。

不过，一般来说应当避免具体的基类（参见条款 35）。例如，`unary_function` 不过是聚合了一组 typedef，它不可能会被有意单独实例化。给它提供 public 的析构函数完全没有任何意义；更好的设计应当是遵循本条款的建议来给它一个 protected 非虚析构函数猜到。

**References**: [\[C++CS\]](#CplusplusCS) Item 50, [\[Cargill92\]](#Cargill92) pp. 77-79, 207? [\[Cline99\]](#Cline99) §21.06, 21.12-13? [\[Henricson97\]](#Henricson97) pp. 110-114? [\[Koenig97\]](#Koenig97) Chapters 4, 11? [\[Meyers97\]](#Meyers97) §14? [\[Stroustrup00\]](#Stroustrup00) §12.4.2? [\[Sutter02\]](#Sutter02) §27? [\[Sutter04\]](#Sutter04) §18

### <a name="Sd-noexcept"></a>讨论: noexcept 的用法

???

### <a name="Sd-never-fail"></a>讨论: 虚构函数，回收函数和 swap 不允许失败

绝不能允许从虚构函数，资源回收函数（如 `operator delete`），或者 `swap` 函数中用 `throw` 来报告错误。如果这些操作可以失败的话，就几乎不可能编写有用的代码了，而且即便真的发生了某种错误，也几乎不可能有进行重试的任何意义。特别是，C++ 标准库是直截了当地禁止使用可能在析构函数中抛出异常的类型的。现在，大多数析构函数缺省就隐含带有 `noexcept` 了。

##### 示例

    class Nefarious {
    public:
        Nefarious()  { /* 可能抛出异常的代码 */ }   // 好
        ~Nefarious() { /* 可能抛出异常的代码 */ }   // 不好, 可能抛出异常
        // ...
    };

1. `Nefarious` 对象很难安全地使用，即便是作为局部变量也是如此：


        void test(string& s)
        {
            Nefarious n;          // 要有麻烦了
            string copy = s;      // 复制 string
        } // 先后销毁 copy 和 n
    
    这里，对 `s` 的复制可能抛出异常，且当其抛出了异常而 `n` 的析构函数也抛出了异常时，程序就会因调用 `std::terminate` 而退出，因为无法同时传播两个异常。

2. 以 `Nefarious` 为成员或者基类的类同样很难安全地使用，因为它们的析构函数必须调用 `Nefarious` 的析构函数，且同样遭受其低劣的行为的毒害：


        class Innocent_bystander {
            Nefarious member;     // 噢，毒害了外围类的析构函数
            // ...
        };
    
        void test(string& s)
        {
            Innocent_bystander i; // 要有更多麻烦了
            string copy2 = s;      // 复制 string
        } // 依次销毁 copy 和 i
    
    这里，当 `copy2` 的构造中抛出了异常时，我们会遇到同样的问题，因为 `i` 的析构函数现在也会抛出异常，且因此会使我们调用 `std::terminate`。

3. 你也无法可靠地创建全局或静态的 `Nefarious` 对象：


        static Nefarious n;       // 噢，无法捕获任何析构函数异常

4. 你无法可靠地创建 `Nefarious` 的数组：


        void test()
        {
            std::array<Nefarious, 10> arr; // 这行代码会导致 std::terminate(!)
        }
    
    当出现可能抛出异常的析构函数时，数组的行为是未定义的，因为根本不可能发明出合理的回退行为。请想象一下：编译器如何才能生成用来构造 `arr` 的代码，如果第四个对象的构造函数抛出了异常，这段代码必须放弃，且在其清理模式中将试图调用已经构造完成的每个对象的析构函数……而这些析构函数中的一个或更多会抛出异常呢？不存在令人满意的答案。

5. 你无法在标准容器中使用 `Nefarious`：


        std::vector<Nefarious> vec(10);   // 这行代码会导致 std::terminate()
    
    标准库禁止其所使用的任何析构函数抛出异常。你无法把 `Nefarious` 对象存储到标准容器中，或者在标准库的任何其他组件上使用它们。

##### 注解

它们是绝不能失败的关键函数，因为在事务性编程中需要它们提供两种关键操作：当处理过程中遇到问题时撤回工作，以及当未发生问题时提交工作。如果没有办法可以用无失败操作来安全地撤回的话，就不可能实现无失败的回滚操作。如果没有办法可以用无失败操作（显然 `swap` 可以，但并不仅限于它）来安全地提交状态的改变的话，就不可能实现无失败的提交操作。

请考虑以下在 C++ 标准中所找到的建议和要求：

> 当在栈展开过程中所调用的析构函数因为异常而退出时，将调用 terminate (15.5.1)。因此析构函数通常应当捕获异常，并防止它们被传播出析构函数。 --[\[C++03\]](#Cplusplus03) §15.2(3)
>
> C++ 标准库中所定义的任何析构函数（也包括用于实例化标准库模板的任何类型的析构函数）的操作都不会抛出异常。 --[\[C++03\]](#Cplusplus03) §17.4.4.8(3)

包括专门重载的 `operator delete` 和 `operator delete[]` 在内的回收函数也属于这一类别，因为一般它们也被用在清理过程，尤其是在异常处理过程中，用以对部分完成的工作进行撤回。
除了析构函数和回收函数之外，一般的错误安全性技术也依赖于永不失败的 `swap` 操作——这种情况下，它们不仅用于实现确保成功的回滚操作，也用于实现确保成功的提交操作。例如，以下是对类型 `T` 的一种惯用的 `operator=` 实现，它在复制构造之后，调用了无失败的 `swap`：

    T& T::operator=(const T& other) {
        auto temp = other;
        swap(temp);
        return *this;
    }

(另见条款 56。 ???)

幸运的是，当进行资源释放时，发生故障的范围肯定会比较小。如果使用异常作为错误报告机制的话，请确保这样的函数会处理其内部的处理中可能会产生的所有异常和其他错误。（对于异常，可以直接把你的析构函数中的所有相关部分都包围到一个 `try/catch(...)` 块中。）这点非常重要，因为析构函数可能会在某种紧要关头被调用，比如当无法分配某种系统资源（如内存、文件、锁、端口、窗口，或者其他系统对象）的时候。

当使用异常作为错误处理机制的时候，请始终明示这种行为，将这些函数声明为 `noexcept`。（参见条款 75。）

**参考**: [\[C++CS\]](#CplusplusCS) Item 51; [\[C++03\]](#Cplusplus03) §15.2(3), §17.4.4.8(3)? [\[Meyers96\]](#Meyers96) §11? [\[Stroustrup00\]](#Stroustrup00) §14.4.7, §E.2-4? [\[Sutter00\]](#Sutter00) §8, §16? [\[Sutter02\]](#Sutter02) §18-19

## <a name="Sd-consistent"></a>统一对复制、移动和销毁操作进行定义

##### 理由

 ???

##### 注解

一旦定义了复制构造函数，就也得定义复制赋值运算符。

##### 注解

一旦定义了移动构造函数，就也得定义移动赋值运算符。

##### 示例

    class X {
        // ...
    public:
        X(const x&) { /* stuff */ }
    
        // 不好: 未同时定义复制赋值运算符
    
        X(x&&) noexcept { /* stuff */ }
    
        // 不好: 未同时定义移动赋值运算符
    };
    
    X x1;
    X x2 = x1; // ok
    x2 = x1;   // 陷阱：要么不能通过编译，要么会做出不好的事

一旦定义了析构函数，就不能再使用编译器所生成的复制或移动操作了；你可能需要定义或者抑制掉移动或复制操作。

    class X {
        HANDLE hnd;
        // ...
    public:
        ~X() { /* 自定义的行为，比如关闭 hnd */ }
        // 可疑: 未提到过复制或移动操作——hnd 会怎么样？
    };
    
    X x1;
    X x2 = x1; // 陷阱：要么不能通过编译，要么会做出不好的事
    x2 = x1;   // 陷阱：要么不能通过编译，要么会做出不好的事

如果定义了复制操作，且有任何基类或成员的诶性定义了移动操作的话，应当同样定义移动操作。

    class X {
        string s; // 定义了更高效的移动操作
        // ... 其他数据成员 ...
    public:
        X(const X&) { /* stuff */ }
        X& operator=(const X&) { /* stuff */ }
    
        //    不好: 并未一同定义移动构造函数和移动赋值
        //   （为何不把那些自定义的"stuff"重复一下呢？）
    };
    
    X test()
    {
        X local;
        // ...
        return local;  // 陷阱：可能会低效甚或产生错误的行为
    }

一旦定义了复制构造函数，复制赋值运算符，或者析构函数中额任何一个，你就可能需要也定义其他的。

##### 注解

如果需要定义这五个函数中的任何一个，这就意味着你需要得到与其预置行为不同的行为——而这五者则是非对称相关的。如下所述：

* 当编写或禁用复制构造函数或复制赋值运算符之一时，很可能需要对另一个同样对待：若其中之一有"特别的"任务，则很可能另一个也应当如此，因为这两个函数应当具有相似的效果。（参见条款 53，其中对这点进行专门的展开说明。）
* 当明确编写复制函数时，很可能需要编写析构函数：若复制构造函数所做的"特别的"任务为分配或复制某中资源（诸如内存、文件、socket等），则需要在析构函数中对其进行回收。
* 当明确编写析构函数时，很可能需要明确编写或禁用复制操作：若不得不编写一个不平凡的析构函数的话，这通常是由于你需要人工释放对象所持有的某个资源。若是如此的话，很可能需要特别小心这些资源的复制，而你就需要关注对象进行复制和赋值的方式，或者完全禁止复制操作。

许多情况下，持有以 RAII 的"拥有者"对象恰当封装了的资源，是能够吧自己编写这些操作的需要消除掉的。（参见条款 13。）

应当优先采用编译器生成（包括 `=default`）的特殊成员；只有它们才被归类为"平凡的"，而且至少有一家主要的标准库供应商针对带有平凡特殊成员的类进行了大量地优化。这可能会成为一种常规实践。

**例外**: 当特殊函数的声明仅为了使其非公开或者为虚函数而没有特殊的语义时，它并不导致需要其他的特殊成员。
少数情况下，带有奇怪类型的成员（诸如引用成员）的类也是例外，因为它们的复制语义很古怪。
在持有引用的类中，你可能需要编写复制构造函数和赋值运算符，但预置的析构函数仍能够做出正确的处理。（需要注意，基本上使用引用成员几乎总是错误的。）

**参考**: [\[C++CS\]](#CplusplusCS) Item 52; [\[Cline99\]](#Cline99) §30.01-14? [\[Koenig97\]](#Koenig97) §4? [\[Stroustrup00\]](#Stroustrup00) §5.5, §10.4? [\[SuttHysl04b\]](#SuttHysl04b)

资源管理规则概览：

* [提供强资源安全性；亦即，绝不让你认为是资源的任何东西发生泄漏](#Cr-safety)
* [绝不在持有未被句柄所拥有的资源时抛出异常](#Cr-never)
* ["原生"的指针或引用不可能是资源句柄](#Cr-raw)
* [绝不让指针的生存期超过其所指向的对象](#Cr-outlive)
* [用模板来表现容器（和其他资源句柄）](#Cr-templates)
* [按值返回容器（依靠移动或复制消除来获得性能）](#Cr-value-return)
* [若类为资源句柄，则它需要构造函数，析构函数，复制以及移动操作](#Cr-handle)
* [若类为容器，则应为其提供一个初始化式列表构造函数](#Cr-list)

### <a name="Cr-safety"></a>讨论：提供强资源安全性；亦即，绝不让你认为是资源的任何东西发生泄漏

##### 理由

避免泄漏。泄漏会导致性能损耗，发生神秘的错误，系统崩溃，以及安全性的违犯。

**其他形式**: 使所有资源都表示为某种可以自我管理生存期的类的对象。

##### 示例

    template<class T>
    class Vector {
    private:
        T* elem;   // 自由存储中的 sz 个元素，由类对象所拥有
        int sz;
        // ...
    };

这个类是一个资源句柄。它管理各个 `T` 对象的生存期。为此，`Vector` 必然要对[一组特殊操作](???)（几个构造函数，析构函数，等等）进行定义或弃置。

##### 示例

    ??? "奇异的"非内存资源 ???

##### 强制实施

防止泄漏的基本技巧是让所有的资源都被某种带有回档析构函数的资源句柄所拥有。检查工具能够查找出"裸 `new`"。给定一组 C 风格的分配函数（如 `fopen()`），检查工具也能够查找出未被资源句柄管理的使用点。一般来说，可以带着怀疑看待"裸指针"，对其进行标记和分析。如果没有人为输入的话，时无法产生资源的完整列表的（"资源"的定义有些过于宽泛），不过可以用一个资源列表来对工具进行"参数化"。

### <a name="Cr-never"></a>讨论：绝不在持有未被句柄所拥有的资源时抛出异常

##### 理由

这会导致泄漏。

##### 示例

    void f(int i)
    {
        FILE* f = fopen("a file", "r");
        ifstream is { "another file" };
        // ...
        if (i == 0) return;
        // ...
        fclose(f);
    }

当 `i == 0` 时 `a file` 的文件句柄就会泄漏。另一方面，`another file` 的 `ifstream` 则将会（在销毁时）正确关闭它的文件。如果你必须显式使用指针而不是带有特定语义的资源句柄的话，可以使用带有自定义删除器的 `unique_ptr` 或 `shared_ptr`：

    void f(int i)
    {
        unique_ptr<FILE, int(*)(FILE*)> f(fopen("a file", "r"), fclose);
        // ...
        if (i == 0) return;
        // ...
    }

这样更好：

    void f(int i)
    {
        ifstream input {"a file"};
        // ...
        if (i == 0) return;
        // ...
    }

##### 强制实施

检查器必须将任何"裸指针"当作可疑处理。
检查器可能必须依赖于人工提供的资源列表进行工作。
上手时，我们知道标准库容器，`string`，以及智能指针。
`span` 和 `string_span` 的使用能够提供巨大的帮助（它们并非资源句柄）。

### <a name="Cr-raw"></a>讨论："原生"的指针或引用不可能是资源句柄

##### 理由

使得能够区分所有者和视图。

##### 注解

这和你如何"拼写"指针是两回事：`T*`，`T&`，`Ptr<T>` 和 `Range<T>` 都不是所有者。

### <a name="Cr-outlive"></a>讨论：绝不让指针的生存期超过其所指向的对象

##### 理由

避免极难找到的错误。这种指针的解引用时未定义行为，能够导致发生对类型系统的违犯。

##### 示例

    string* bad()   // 确实很坏
    {
        vector<string> v = { "This", "will", "cause", "trouble", "!" };
        // 导致指向已经销毁的对象（v）的已经销毁的成员的一个指针被泄漏出去
        return &v[0];
    }
    
    void use()
    {
        string* p = bad();
        vector<int> xx = {7, 8, 9};
        // 未定义行为: x 可能不是字符串 "This"
        string x = *p;
        // 未定义行为: 我们不知道在位置 p 上分配的到底是什么（如果有的话）
        *p = "Evil!";
    }

`v` 中的各个 `string` 都在 `bad()` 退出之时被销毁了， `v` 自身也是如此。其所返回的指针指向自由存储上的未分配内存。（由 `p` 所指向的）这块内存，在执行 `*p` 之时可能已经被重新分配了。此时很可能并不存在可以读取的 `string` 对象，而通过 `p` 进行写入则会轻易损坏某些无关类型的对象。

##### 强制实施

大多数编译器已经能对简单情况进行警告，而且它们带有可以更进一步的信息。将函数所返回的任何指针都当作是可疑的。用容器、资源句柄和视图（例如 `span`，它不是资源句柄）来减少需要检查的情形。上手时，可将带有析构函数的类都当作是资源句柄处理。

### <a name="Cr-templates"></a>讨论：用模板来表现容器（和其他资源句柄）

##### 理由

提供静态类型安全的元素操作。

##### 示例

    template<typename T> class Vector {
        // ...
        T* elem;   // 指向 sz 个 T 类型的元素
        int sz;
    };

### <a name="Cr-value-return"></a>讨论：按值返回容器（依靠移动或复制消除来获得性能）

##### 理由

简化代码并消除一种进行显式内存管理的需要。将对象递交给外围作用域，由此扩展其生存期。

**参见**：[F.20，有关"输出（Out）"值的一般条款](#Rf-out)

##### 示例

    vector<int> get_large_vector()
    {
        return ...;
    }
    
    auto v = get_large_vector(); //  按值返回没有问题，大多数现代编译器都会进行复制消除

##### 例外

见 [F.20](#Rf-out) 中的例外。

##### 强制实施

检查函数所返回额指针和引用，看看它们是否被赋值给资源句柄（如 `unique_ptr`）。

### <a name="Cr-handle"></a>讨论：若类为资源句柄，则它需要构造函数，析构函数，复制以及移动操作

##### 理由

以提供对资源的生存期的完全控制。以提供一组协调的对资源的操作。

##### 示例

    ??? 折腾指针

##### 注解

若所有的成员都为资源句柄，则尽可能要依赖预置的特殊操作。

    template<typename T> struct Named {
        string name;
        T value;
    };

现在 `Named` 带有一个默认构造函数，一个析构函数，以及高效的复制和移动操作，只要 `T` 也提供了它们。

##### 强制实施

一般来说，工具是无法知道类是否是资源句柄的。不过，如果类带有某种[默认操作](#SS-ctor)的话, 它就得拥有全部，而如果类中有成员为资源句柄的话，它也应被当做是资源句柄。

### <a name="Cr-list"></a>讨论：若类为容器，则应为其提供一个初始化式列表构造函数

##### 理由

提供一组初始元素是一种常见情形。

##### 示例

    template<typename T> class Vector {
    public:
        Vector(std::initializer_list<T>);
        // ...
    };
    
    Vector<string> vs { "Nygaard", "Ritchie" };

##### 强制实施

类怎么算作是容器呢？ ???

# <a name="S-tools"></a>附录 D: 支持工具

这个部分列出了直接支持采用 C++ 核心指导方针的一些工具。这个列表并非要穷尽那些有助于编写良好的 C++ 代码的工具。
如果一个工具被专门设计以支持并关联到 C++ 核心指导方针，那它就是包括进来的候选者。

### <a name="St-clangtidy"></a>工具: [Clang-tidy](http://clang.llvm.org/extra/clang-tidy/checks/list.html)

Clang-tidy 有一组专门用于强制实施 C++ 核心指导方针的规则。这些规则的命名模式为 `cppcoreguidelines-*`。

### <a name="St-cppcorecheck"></a>工具: [CppCoreCheck](https://docs.microsoft.com/en-us/visualstudio/code-quality/using-the-cpp-core-guidelines-checkers)

微软编译器的 C++ 代码分析中包含一组专门用于强制实施 C++ 核心指导方针的规则。

# <a name="S-glossary"></a>词汇表

这是在指导方针中用到的一些术语的相对非正式的定义
（基于 [Programming: Principles and Practice using C++](http://www.stroustrup.com/programming.html) 中的词汇表）。

有关 C++ 的许多主题的更多信息，可以在[标准 C++ 基金会的网站](https://isocpp.org) 找到。

* *ABI*: 应用二进制接口，对于特定硬件平台与操作系统的组合的一种规范。与 API 相对。
* *抽象类（abstract class）*: 不能直接用于创建对象的类；通常用于为派生类定义接口。
  当类带有纯虚函数或只有受保护的构造函数时，它就是抽象的。
* *抽象（abstraction）*: 对事物的描述，有选择并有意忽略（隐藏）了细节（如实现细节）；选择性忽略。
* *地址（address）*: 用以在计算机的内存中找到某个对象的值。
* *算法（algorithm）*: 用以解决某个问题的过程或公式；有限的一系列计算步骤以产生一个结果。
* *别名（alias）*: 指代某个对象的替代方式；通常为名字，指针，或者引用。
* *API*: 应用编程接口，一组构成不同软件之间之间的交互的函数。与 ABI 相对。
* *应用程序（application）*: 程序或程序的集合，用户将其看作一个实体。
* *近似（approximation）*: 事物（比如值或者设计），接近于完美的或者理想的（值或设计）。
  通常近似都是在理想情形中进行各种权衡的结果。
* *参数/实参（argument）*: 传递给函数或模板的值，其中以形参来进行访问。
* *数组（array）*: 同质元素序列，通常是数值，例如 `[0:max)`。
* *断言（assertion）*: 插入到程序中的语句，以声称（断言）在程序的这个位置某事物必定为真。
* *基类（base class）*: 用作类层次的基础的类。通常基类带有一个或更多的虚函数。
* *位（bit）*: 计算机中信息的基本单位。一个位的值可以为 0 或 1。
* *bug*: 程序中的错误。
* *字节（byte）*: 大多数计算机中进行寻址的基本单位。通常一个字节有 8 位。
* *类（class）*: 一种用户定义的类型，可以包含数据成员，函数成员，以及成员类型。
* *代码（code）*: 程序或程序的部分；有歧义地同时用于源代码和目标代码。
* *编译器（compiler）*: 一种将源代码变为目标代码的程序。
* *复杂度（complexity）*: 对某个问题构造解决方案的难度，或解决方案自身的一种难于精确定义的记法或度量。
  有时候复杂度只是（简单地）表示对执行某个算法所需操作的数量的估计。
* *计算（computation）*: 执行一些代码，通常接受一些输入并产生一些输出。
* *概念（concept）*: (1) 提法，想法；(2) 一组要求，通常针对模板参数。
* *具体类（concrete class）*: 可以使用常规的构造语法（例如，在栈上构造对象）创建对象的类，并且产生的对象更像是 `int`，它可以进行复制、比较等等操作，
（但与类型层次中的基类相对）。
* *常量（constant）*: （在给定作用域中）不能改变的值；不可变。
* *构造函数（constructor）*: 初始化（"构造"）一个对象的操作。
  通常构造函数会建立起不变式，并且通常会获取对象被使用时所需的资源（并通常将由析构函数所释放）。
* *容器（container）*: 持有一些元素（其他对象）的对象。
* *复制（copy）*: 制造两个对象使其值比较为相等的操作。另见移动。
* *正确性（correctness）*: 如果程序或程序片段符合其说明，则其为正确的。
  不幸的是，说明可能不完整或不一致，或者也可能无法满足用户的合理预期。
  因此为了产生可接受的代码，我们有时候比仅仅遵守形式说明要做更多的事。
* *成本（cost）*: 产生一个程序，或者执行它的耗费（如开发时间，执行时间或空间等）。
  理想情况下，成本应当是复杂度的函数。
* *定制点（customization point）*:
* *数据（data）*: 计算中所用到的值。
* *调试（debugging）*: 寻找并移除程序中的错误的行为；通常远没有测试那样系统化。
* *声明式（declaration）*: 程序中对一个名字及其类型的说明。
* *定义式（definition）*: 实体的声明式，提供了程序使用该实体所需的所有信息。
  简化版定义：分配了内存额声明。
* *派生类（derived class）*: 派生自一个或多个基类的类。
* *设计（design）*: 对软件的某个片段应当如何运作以满足其说明的一个总体描述。
* *析构函数（destructor）*: 当对象销毁（如在作用域结束时）被隐式执行（调用）的操作。它通常进行资源的释放。
* *封装（encapsulation）*: 将某些事物（如实现细节）保护为私有的，不接受未授权的访问。
* *错误（error）*: 对程序行为的合理期望（通常表现为某种需求或者一份用户指南）和程序的实际行为之间的不一致。
* *可执行程序（executable）*: 预备在计算机上运行（执行）的程序。
* *功能蔓延（feature creep）*: 为"预防万一"而向程序添加过量的功能的倾向。
* *文件（file）*: 计算机中的持久信息的容器。
* *浮点数（floating-point number）*: 计算机对实数（如 7.93 和 10.78e–3）的近似。
* *函数（function）*: 命名的代码单元，可以从程序的不同部分执行（调用）；计算的逻辑单元。
* *泛型编程（generic programming）*: 关注于算法的设计和高效实现的一种编程风格。
  泛型算法能够对所有符合其要求的参数类型正确工作。在 C++ 中，泛型编程通常使用模板进行。
* *全局变量（global variable）*: 技术上说，命名空间作用域中的具名对象。
* *句柄（handle）*: 一个类，允许通过一个成员指针或引用来访问另一个对象。另见资源，复制，移动。
* *头文件（header）*: 包含用于在程序的各个部分中共享接口的声明的文件。
* *隐藏（hiding）*: 防止一个信息片段被直接看到或访问的行为。
  例如，嵌套（内部）作用域中的名字会防止外部（外围）作用域中相同的名字被直接使用。
* *理想的（ideal）*: 我们力争达成的事物的完美版本。我们经常不得不进行各种权衡最后获得一个近似。
* *实现（implementation）*: (1) 编写代码并测试的活动；(2) 用以实现一个程序的代码。
* *无限循环（infinite loop）*: 终止条件永不为真的循环。参见重复。
* *无限递归（infinite recursion）*: 无法终止的递归，直到机器耗尽内存无法维持其调用。
  在现实中这种递归不可能是无限的，它会因某种硬件错误而终止。
* *信息隐藏（information hiding）*: 分离接口和实现，以此将用户不感兴趣的实现细节隐藏起来，并提供一种抽象的活动。
* *初始化（initialize）*: 为一个对象给定其第一个（初始）值。
* *输入（input）*: 计算中所使用的值（比如函数参数以及通过键盘所输入的字符）。
* *整数（integer）*: 整数，比如 42 和 –99。
* *接口（interface）*: 一个或一组声明，说明了一个代码片段（比如函数或者类）应当如何进行调用。
* *不变式（invariant）*: 程序中的某些点必然总为真的事物；通常用于描述对象的状态（值的集合），或者循环进入其重复的语句之前的状态。
* *重复（iteration）*: 重复执行代码片段的行为；参见递归。
* *迭代器（iterator）*: 用以标识序列中的一个元素的对象。
* *ISO*: 国际标准化组织。C++ 语言是一项 ISO 标准：ISO/IEC 14882。更多信息请参考 [iso.org](http://iso.org)。
* *程序库（library）*: 类型、函数、类等等的集合，它们实现了一组设施（抽象），预备可能被用作不止一个程序的组成部分。
* *生存期（lifetime）*: 从对象的初始化直到它变为不可用（离开作用域，被删除，或程序终止）的时间。
* *连接器（linker）*: 用以将目标代码文件和程序库合并构成一个可执行程序的程序。
* *字面量（literal）*: 直接指定一个值的写法，比如 12 指定的是整数值"十二"。
* *循环（loop）*: 重复执行的代码片段；在 C++ 中，通常是 `for` 语句或者 `while` 语句。
* *移动（move）*: 将值从一个对象转移到另一个对象，并遗留一个表示"空"的值的操作。另见复制。
* *可变的（mutable）*: 可以改动；不可变、常量和不变量的反义词。
* *对象（object）*: (1) 已经初始化的一块具有已知类型的内存区域，持有该类型的一个值；(2) 一块内存区域。
* *目标代码（object code）*: 编译器的输出，预备作为连接器的输入（连接器以其产生可执行代码）。
* *目标文件（object file）*: 包含目标代码的文件。
* *面向对象编程（object-oriented programming）*: （OOP）一种关注类和类层次的设计和使用的编程风格。
* *操作（operation）*: 能够实施某种活动的事物，比如函数或运算符。
* *输出（output）*: 由计算所产生的值（例如函数的结果，或者在屏幕上写下的一行行字符等）。
* *溢出（overflow）*: 产生无法被其预期目标所存储的值。
* *重载（overload）*: 定义两个函数或运算符，使其具有相同名字但不同的参数（操作数）类型。
* *覆盖（override）*: 在派生类中用声明和基类中的某个虚函数具有相同名字和参数类型的函数，以此使该函数可以通过由基类所定义的接口来进行调用。
* *所有者（owner）*: 负责释放某个资源的对象。
* *范式（paradigm）*: 设计和编程风格的一种多少有些做作的术语；通常会被用于（错误地）暗示有一种范式被其他的都更优秀。
* *形参（parameter）*: 对函数或模板的一个明确输入的声明。当进行调用时，函数可以通过其形参的名字来访问向其所传递的各个实参。
* *指针（pointer）*: (1) 值，用于标识内存中的一个有类型的对象；(2) 持有这种值的变量。
* *后条件（post-condition）*: 当从一个代码片段（如函数或者循环）退出时必须满足的条件。
* *前条件（pre-condition）*: 当进入一个代码片段（如函数或者循环）时必须满足的条件。
* *程序（program）*: 足够完整以便能够在计算机上执行的代码（可能带有关联的数据）。
* *编程（programming）*: 将问题的解决方案表现为代码的工艺。
* *编程语言（programming language）*: 用于表达程序的语言。
* *伪代码（pseudo code）*: 以非正式的写法而非编程语言所编写的对计算的一种描述。
* *纯虚函数（pure virtual function）*: 必须在派生类中予以覆盖的虚函数。
* *RAII*: （"资源获取即初始化，Resource Acquisition Is Initialization"）一种基于作用域进行资源管理的基本技术。
* *范围（range）*: 值的序列，可以以一个开始点和一个结尾点进行描述。例如，`[0:5)` 的意思是值 0，1，2，3，和 4。
* *递归（recursion）*: 函数调用其自身的行为；另见重复。
* *引用（reference）*: (1) 一种值，描述内存中具有类型的值的位置；(2) 持有这种值的变量。
* *正则表达式（regular expression）*: 对字符串的模式的一种表示法。
* *正规*: 类型的行为与像 `int` 这样内建类型相似，且可以用 `==` 进行比较。
特别是，正规类型的对象可以进行复制，且复制的结果是与原对象比较为相等的一个独立对象。另见*半正规类型*。
* *要求（requirement）*: (1) 对程序或程序的一部分的预期行为的描述；(2) 对函数或模板对其参数所作出的假设的描述。
* *资源（resource）*: 获取而得的并随后必须被释放的事物，比如文件句柄，锁，或者内存。另见句柄，所有者。
* *舍入（rounding）*: 将一个值转换为某个较不精确类型的数学上最接近的值。
* *RTTI*: 运行时类型信息（Run-Time Type Information）。 ???
* *作用域（scope）*: 程序文本（源代码）的区域，在其中可以对一个名字进行涉指。
* *半正规*: 类型的行为与像 `int` 这样内建类型大致相似，但可能没有 `==` 运算符。另见*正规类型*。
* *序列（sequence）*: 可以以线性的顺序访问的一组元素。
* *软件（software）*: 代码片段及其关联数据的集合；通常可以和程序互换运用。
* *源代码（source code）*: 由程序员所生产的代码，（原则上）可以被其他程序员阅读。
* *源文件（source file）*: 包含源代码的文件。
* *规范（specification）*: 对代码片段应当做什么的描述。
* *标准（standard）*: 由官方承认的对某事物的定义，比如编程语言。
* *状态（state）*: 一组值。
* *STL*: 标准库中的容器，迭代器，以及算法部分。
* *字符串（string）*: 字符的序列。
* *风格（style）*: 旨在统一语言功能特征的使用的一组编程技巧；有时候以非常限定的方式来仅代表诸如命名和代码展现等的低层次规则。
* *子类型（subtype）*: 派生类型；一个类型具有另一个类型的所有（可能更多）的性质。
* *超类型（supertype）*: 基类型；一个类型具有另一个类型的性质的子集。
* *系统（system）*: (1) 用以在计算机上实施某种任务的一个或一组程序；(2) 对"操作系统"的简称，即计算机的基本执行环境及工具。
* *TS*: [技术规范](https://www.iso.org/deliverables-all.html type=ts)。技术规范所处理的是仍处于技术开发之中的工作，或者是认为这项工作以后可能会被同意采纳为国际标准，但并不会立即处理。技术规范的出版是为了其立即可用，也是为了提供一种获得反馈的方法。其目标是最终能够被转化并重新作为国际标准来出版。
* *模板（template）*: 由一个或多个的类型或（编译时）值进行参数化的类或函数；支持泛型编程的基本 C++ 语言构造。
* *测试（testing）*: 系统化地查找程序中的错误。
* *权衡（trade-off）*: 对多个设计和实现准则进行平衡的结果。
* *截断（truncation）*: 从一个类型转换为另一个无法精确表示被转换的值的类型时发生的信息损失。
* *类型（type）*: 为一个对象定义了一组可能的值和一组操作的事物。
* *未初始化的（uninitialized）*: 对象在初始化之前的（未定义的）状态。
* *单元（unit）*: (1) 为值赋予含义的一种标准度量（例如，距离单位 km）；(2) 较大的整体中的一个可区分的（比如命名的）部分。
* *用例（use case）*: 程序的某个特定（通常简化的）使用，以测试其功能并演示其目的。
* *值（value）*: 根据某个类型所解释的一组内存中的位。
* *变量（variable）*: 给定类型的具名对象；除非未初始化否则包含一个值。
* *虚函数（virtual function）*: 可在派生类中进行覆盖的成员函数。
* *字（word）*: 计算机中内存的基本单元，通常是用以持有一个整数的单元。

# <a name="S-unclassified"></a>To-do: 未分类的规则原型

这是我们的未完成列表。
以下各条目最终将成为规则或者规则的一部分。
或者，我们也会决定不需要做出改动并将条目移除。

* 禁止远距离友元关系
* 应不应该处理物理设计（文件里有什么）和大规模设计（程序库，程序库的组合）？
* 命名空间
* 避免在全局作用域中使用 using 指令（但允许如 std 或其他的"基础"命名空间（如 experimental））
* 命名空间应当有什么粒度？是（如 Sutter/Alexandrescu 所定义的）所有被设计为一同工作或者一同发布的类和函数，还是应该更窄或是更宽？
* 应该用内联命名空间吗（比如 `std::literals::*_literals`）？
* 避免隐式转换
* Const 成员函数应当是线程安全的……aka, 但我并不想真的改掉变量，只是在第一次调用它的时候向它赋一个值……argh
* 始终初始化变量，为成员变量使用初始化列表。
* 无论谁编写了接受或返回 `void*` 的公开接口，都应该上火刑。我曾经好多年都以它作为自己的个人喜好来着。 :)
* 尽可能应用 `const`：成员函数，变量，以及 `const_iterators`
* 使用 `auto`
* `(size)` vs. `{initializers}` vs. `{Extent{size}}`
* 不要过度抽象
* 不要沿着调用栈向下传递指针
* 通过函数底部退出
* 应当提供在多态之间进行选择的指导方针吗？是的。经典的（虚函数，引用语义） vs. Sean Parent 风格（值语义，类型擦除，类似 `std::function`）  vs. CRTP/静态的？也许还需要 vs. 标签派发？
* 我们的指导方针是否应当在构造函数或析构函数中禁止进行虚函数调用？是的。许多人都禁止了，虽然我觉得这是 C++ 的一大优势 ??? -保留意见（D 走向 Java 之路太让我失望了）。有好的例子吗？
* 在 lambda 方面，在算法调用和其他回调场景中什么因素会影响决定使用 lambda 还是（局部？）类？
* 讨论一下 `std::bind`，Stephen T. Lavavej 对它有太多批评，使我开始觉得它是不是真的会在未来消失掉。应该建议以 lambda 代替它吗？
* 怎么处理泄漏的临时变量？ : `p = (s1 + s2).c_str();`
* 指针和迭代器的失效会导致悬挂指针：

        void bad()
        {
            int* p = new int[700];
            int* q = &p[7];
            delete p;
        
            vector<int> v(700);
            int* q2 = &v[7];
            v.resize(900);
        
            // ... 使用 q 和 q2 ...
        }

* LSP
* 私有继承 vs/and 成员
* 避免静态类成员变量（竞争条件，几乎就是全局变量）

* 使用 RAII 锁定保护（`lock_guard`，`unique_lock`，`shared_lock`），绝不直接调用 `mutex.lock` 和 `mutex.unlock`（RAII）
* 优先使用非递归锁（它们通常用作不良情况的变通手段，有开销）
* 联结（join）你的每个线程！（因为如果没被联结或脱离（detach）的话，析构函数会调用 `std::terminate`……有什么好理由来脱离线程吗？） -- ??? 支持库该不该为 `std::thread` 提供一个 RAII 包装呢？
* 当必须同时获取两个或更多的互斥体时，应当使用 `std::lock`（或者别的死锁免除算法？）
* 当使用 `condition_variable` 时，始终用一个互斥体来保护它（在互斥体外面设置原子 bool 的值的做法是错误的！），并对条件变量自身使用同一个互斥体。
* 绝不对 `std::atomic<user-defined-struct>` 使用 `atomic_compare_exchange_strong`（填充位中的区别会造成影响，而在循环中使用 `compare_exchange_weak` 则能够归于稳定的填充位）
* 单独的 `shared_future` 对象不是线程安全的：两个线程不能等待同一个 `shared_future` 对象（它们可以等待指代相同共享状态的 `shared_future` 的副本）
* 单独的 `shared_ptr` 对象不是线程安全的：不同的线程可以调用指代相同共享对象的*不同* `shared_ptr` 的非 `const` 成员函数，但当一个线程访问一个 `shared_ptr` 对象时，另一个线程不能调用相同 `shared_ptr` 对象的非 `const` 成员函数（如果确实需要，考虑代之以 `atomic_shared_ptr`）

* 算术相关规则

# 参考文献

* <a name="Abrahams01"></a>
  \[Abrahams01]:  D. Abrahams. [Exception-Safety in Generic Components](http://www.boost.org/community/exception_safety.html).
* <a name="Alexandrescu01"></a>
  \[Alexandrescu01]:  A. Alexandrescu. Modern C++ Design (Addison-Wesley, 2001).
* <a name="Cplusplus03"></a>
  \[C++03]:           ISO/IEC 14882:2003(E), Programming Languages — C++ (updated ISO and ANSI C++ Standard including the contents of (C++98) plus errata corrections).
* <a name="CplusplusCS"></a>
  \[C++CS]:           ???
* <a name="Cargill92"></a>
  \[Cargill92]:       T. Cargill. C++ Programming Style (Addison-Wesley, 1992).
* <a name="Cline99"></a>
  \[Cline99]:         M. Cline, G. Lomow, and M. Girou. C++ FAQs (2ndEdition) (Addison-Wesley, 1999).
* <a name="Dewhurst03"></a>
  \[Dewhurst03]:      S. Dewhurst. C++ Gotchas (Addison-Wesley, 2003).
* <a name="Henricson97"></a>
  \[Henricson97]:     M. Henricson and E. Nyquist. Industrial Strength C++ (Prentice Hall, 1997).
* <a name="Koenig97"></a>
  \[Koenig97]:        A. Koenig and B. Moo. Ruminations on C++ (Addison-Wesley, 1997).
* <a name="Lakos96"></a>
  \[Lakos96]:         J. Lakos. Large-Scale C++ Software Design (Addison-Wesley, 1996).
* <a name="Meyers96"></a>
  \[Meyers96]:        S. Meyers. More Effective C++ (Addison-Wesley, 1996).
* <a name="Meyers97"></a>
  \[Meyers97]:        S. Meyers. Effective C++ (2nd Edition) (Addison-Wesley, 1997).
* <a name="Meyers15"></a>
  \[Meyers15]:        S. Meyers. Effective Modern C++ (O'Reilly, 2015).
* <a name="Murray93"></a>
  \[Murray93]:        R. Murray. C++ Strategies and Tactics (Addison-Wesley, 1993).
* <a name="Stroustrup94"></a>
  \[Stroustrup94]:    B. Stroustrup. The Design and Evolution of C++ (Addison-Wesley, 1994).
* <a name="Stroustrup00"></a>
  \[Stroustrup00]:    B. Stroustrup. The C++ Programming Language (Special 3rdEdition) (Addison-Wesley, 2000).
* <a name="Stroustrup05"></a>
  \[Stroustrup05]:    B. Stroustrup. [A rationale for semantically enhanced library languages](http://www.stroustrup.com/SELLrationale.pdf).
* <a name="Stroustrup13"></a>
  \[Stroustrup13]:    B. Stroustrup. [The C++ Programming Language (4th Edition)](http://www.stroustrup.com/4th.html). Addison Wesley 2013.
* <a name="Stroustrup14"></a>
  \[Stroustrup14]:    B. Stroustrup. [A Tour of C++](http://www.stroustrup.com/Tour.html).
  Addison Wesley 2014.
* <a name="Stroustrup15></a>
  \[Stroustrup15]:    B. Stroustrup, Herb Sutter, and G. Dos Reis: [A brief introduction to C++'s model for type- and resource-safety](https://github.com/isocpp/CppCoreGuidelines/blob/master/docs/Introduction%20to%20type%20and%20resource%20safety.pdf).
* <a name="SuttHysl04b"></a>
  \[SuttHysl04b]:     H. Sutter and J. Hyslop. [Collecting Shared Objects](https://web.archive.org/web/20120926011837/http://www.drdobbs.com/collecting-shared-objects/184401839) (C/C++ Users Journal, 22(8), August 2004).
* <a name="SuttAlex05"></a>
  \[SuttAlex05]:      H. Sutter and  A. Alexandrescu. C++ Coding Standards. Addison-Wesley 2005.
* <a name="Sutter00"></a>
  \[Sutter00]:        H. Sutter. Exceptional C++ (Addison-Wesley, 2000).
* <a name="Sutter02"></a>
  \[Sutter02]:        H. Sutter. More Exceptional C++ (Addison-Wesley, 2002).
* <a name="Sutter04"></a>
  \[Sutter04]:        H. Sutter. Exceptional C++ Style (Addison-Wesley, 2004).
* <a name="Taligent94"></a>
  \[Taligent94]: Taligent's Guide to Designing Programs (Addison-Wesley, 1994).
