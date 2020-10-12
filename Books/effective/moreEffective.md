## 前言
最近在看《More Effective C++》这个书，自己 C++ 基础还是不行，有的地方看的有点懵，最后还是坚持看完了，做做笔记，**简短的**记录一下有哪些改善编程与设计的有效方法。

推荐还是可以买一本原书的，书中例子比较丰富，更容易理解一些。

## 一、基础议题
### 1. 仔细区别指针（pointer）和引用（reference）
指针可以指向 `null`，引用不允许指向 `null`。
指针可以重新被赋值(=赋值)，引用(=初始化)，所以引用初始化时必须有初值。

### 2. 使用 C++ 显性的类型转换操作符
使用新式转型符比较容易辨析，无论对于人还是工具而言。

- `static_cast` 基本拥有和 C 旧式转型相同的效果和威力，以及相同的限制。
- `const_cast` 用于改变表达式中的 **常量性（constness）**或 **易变性（volatileness）**。
- `dynamic_cast` 用来执行继承体系中 “安全的向下转型或跨系转型动作”。，父-》子
- `reinterpret_cast` 用来将一种类型重新解释为另一种类型，二进制层面。

### 3. 绝对不要以多态方式处理数组
子类通常都比父类更大，所以进行指针算术时会发生不可预计的后果，数组对象几乎总是会涉及指针的算术。所以数组和多态不要混用。

### 4. 非必要不提供默认构造函数（default constructor）
默认构造函数即没有参数的构造函数。添加无意义的默认构造函数显得画蛇添足，也会影响效率。

### 二、操作符
### 5. 对定制的 “类型转换函数” 保持警觉
隐式类型转换操作符，是一个拥有奇怪名称的成员函数：关键字 `operator` 之后加上一个类型名称。如：

```cpp
class Rational {
public:
    // ...
    operator double() const; // 将 Rational 转换为 double
};
```

然而它们的出现可能导致错误（非预期）的函数被调用。为了避免发生这种情况，**解决办法是以功能对等的另一个函数取代类型转换操作符**，例如使用一个名为 `asDouble` 的函数取代它。

单自变量 `constructor` 是指能够以单一自变量成功调用的构造函数。我们可以将构造方法声明为 `explicit` 避免发生隐式转换带来的非预期错误。
### 6. 区分 `++`，`--` 操作符的前置形式和后置形式
重载函数是以其参数类型区分彼此的，然而 `++` 或 `--` 操作符前置式和后置式都没有参数。为了填平这个语法上的漏洞，只好让后置式有一个 `int` 自变量，并且在在它被调用时，编译器默默地为该 `int` 指定一个 0 值。

前置递增运算符通常如下：

```cpp
Date& operator ++ () {
    // increment
    return *this; // 后取出
}
```

后置递增运算符的返回值类型不同，且有一个输入参数，通常如下：

```cpp
const Date operator ++ (int) {
    Date copy (*this); // 先取出
    // increment
    return copy; // 返回先前取出的值
}
```

返回 `const` 类型是为了禁止 Date++++ 这样的错误操作。

单从效率来看，前置式比后置式要好。

### 7. 千万不要重载 `&&`、`||` 和 `,` 操作符
对于短路与（&&）、短路非（||）和逗号运算符（,）不管你多么努力，都无法令其行为像它们应有的那样，所以千万不要重载它们。

### 8. 了解不同意义的 new 和 delete
如果希望对象产生于 heap，请使用 new operator。它不但分配内存还会调用一个构造函数进行初始化。
`string *p = new string("Hello");`

如果只是打算分配内存，请使用 operator new。它不会调用任何构造函数，你也可以写一个自己的 operator new。
`void *p = operator new(sizeof(string));`

如果你打算在指定的内存位置构造对象（需要先分配），请使用 `placement new`。这常用于 `shared memory` 或 `memory-mapped I/O`。

```cpp
#include <new>
void * operator new(size_t, void *location) {
    return location;
}
```

`delete` 会先调用析构函数，再执行 `operator delete` 释放内存。
`delete[]` 会为数组中的每个元素调用析构函数，再执行 `operator delete[]` 释放内存。

## 三、异常
### 9. 利用 destructor 避免泄漏资源
坚持一个原则，将资源封装在对象内，这样即使发生异常，局部对象在自动销毁的时候也可以调用其析构函数释放资源，避免泄漏。

### 10. 在 constructor 内阻止资源泄漏
**注意**： C++ 只会析构**已构造完成**的对象。也就是说在构造函数中发生异常的话，析构函数是不会执行的。

最好是使用 `auto_ptr` 对象来取代 pointer class members。

### 11. 禁止异常流出 destructor 之外
有两个好处：

- 第一是它可以避免 `terminate` 函数在异常传播过程的栈展开机制中被调用，你的程序将被立即结束。
- 第二是它可以协助确保 `destructor` 完成其应该完成的所有事情。

### 12. 了解 “抛出一个异常” 与 “传递一个参数” 或 “调用一个虚函数” 之间的差异
“抛出一个异常”，异常对象总是会被复制一次，如果以 `by value` 方式捕捉，则会发生两次复制。

“传递一个参数”，如果是 `by reference` 方式则不会发生复制，如果是 `by value` 方式则发生复制。（简单的说就是抛出异常会比传递参数多发生一次复制）

抛出的异常不会发生隐式转型（即 int 类型不会默默转为 `double` 类型而被捕获）。只有两种转换可以发生，一种是继承关系中的向上转型，另一种是 “有型指针” 转为 “无型指针”。`const void*` 指针可捕捉任何指针类型的异常。

异常的捕捉遵循 “最先吻合” 策略，即找到第一个匹配者执行。调用虚函数则是 “最佳吻合” 策略，即执行的是与对象类型最吻合的函数。

### 13. 以 by reference 方式捕捉 exception
上面已经说到了，这样可以减少一次复制。

### 14. 明智的运用 exception specification
**exception specification** 即明确指出一个函数可以抛出什么样的异常，例如：

```c++
void fun() throw(int); // 只抛出类型为 int 的异常
```

然而它是一把双刃剑，虽然可以用来规范异常的运用，但也可能带来 `unexpected` 的异常。

### 15. 了解异常处理的成本
你必须知道，异常的支持会导致程序变大，执行效率也比较慢。

## 四、效率
### 16. 谨记 80-20 法则
**80-20 法则**说：一个程序 80% 的资源用在 20% 的代码上。即软件整体性能几乎总是由其构成代码的一小部分决定。

我们可以借助分析器得知程序不同区段花费时间的多少，然后**专注于特别耗时的地方**加以改善。

### 17. 考虑使用 lazy evaluation（缓式评估）
**lazy evaluation**是一种拖延战术，即当运算结果真正要被用到时才进行运算，通常这样我们可以省掉部分运算消耗。

例如矩阵计算中我们场次只会使用到其中的部分结果。

### 18. 分期摊还预期的计算成本
**超急评估（over-eager-evaluation）**是指在被要求前先把事情做下去。

通常有两种策略：一种是**缓存（cache）**，另一种是**预先取出（prefetching）**。它们都是使用空间换取时间的策略。

### 19. 了解临时对象的来源
临时对象可能很耗成本，所以我们应该**尽量消除**它们。

任何时候只要看到对象以 **by value**（值传递） 方式传递，或是以一个 **reference-to-const** 参数方式传递，还有函数直接返回一个对象，这些都极可能产生一个临时对象。

### 20. 协助完成 “返回值优化（Return Value Optimization）”
如果函数一定得以 by-value 方式返回对象，我们就无法消除临时对象的创建和销毁。

但是，如果我们像下面这么做，**编译器可能** 会帮我们将临时对象优化掉，使它们不存在：

```c++
inline const Rational operator* (const Rational& lhs, const Rational& rhs) {
    return Rational(lhs.numerator() * rhs.numerator(),
                    lhs.denominator() * rhs.denominator());
}
```

### 21. 利用重载避免隐式类型转换
通过重载函数指定不同的参数类型，这样可避免传入不同参数类型时，发生隐式转换的过程。

但是增加一大堆重载函数也不是一件好事，除非你认为使用重载函数后程序整体效率可以得到较大的改善。

### 22. 考虑使用复合形式操作符取代独身形式
所谓复合形式即 `+=`，`-=` 等等。例如 `x += y`;
所谓独身形式即 `+`，`-` 等等。例如 `x = x + y`;

一般而言，**复合形式** 效率更高，因为它不会产生临时对象，而独身形式通常需要返回一个新对象，需要负担一个临时对象的构造和析构成本。

### 23. 考虑使用其它程序库
两个程序库可能提供相同的机能，但却有着不同的性能表现。

就比如 `<iostream>` 和 `<stdio.h>`，前者使用更直接方便，后者 I/O 性能更好。所以当我们找到程序的瓶颈时，也可以考虑是否存在另一个功能相近但在效率上有较高提升的程序库。

### 24. 了解虚函数、多继承、虚基类、运行时期类型辨别（RTTI）的成本
**虚函数（virtual function）**意味着编译器会帮你的类维护一个 virtual tables 和 virtua table pointers。常简写为 vtbls 和 vptrs。

但你存在大量拥有虚函数的类，或是每一个类中有大量的虚函数，你可能就会发现，`vtbls` 占用不少内存。

**多继承** 会让事情变得更加复杂，往往还会导致虚基类（virtual base class）的需求，而针对 base class 形成特殊的 `vtbls` 进一步增大负担。

**运行时期类型辨别（runtime type identification）** 依存于类的 `vtbl` 实现，它的空间成本是在 class vtbl 内增加一个条目，再加上每个 class 所需的一份 type_info 对象空间。

## 五、技术
### 25. 将 constructor 和 non-member function 虚化
所谓 virtual constructor 并不是真的将 constructor 声明为 `viatual`，而是某种函数，视其输入而产生不同类型的对象。比较常用的一种场景就是从磁盘读取对象信息。

一种特别的 virtual constructor 即 virtual copy constructor，返回一个指向调用者新副本的指针，常以 `copy` 或 `clone` 命名。例如：

```c++
class NLComponent {
public:
    virtual NLComponent * clone() const = 0;
};
class TextBlock: public NLComponent {
public:
    virtual TextBlock * clone() const {
        return new TextBlock(*this);
    }
};
class Graphic: public NLComponent {
public:
    virtual Graphic * clone() const {
        return new Graphic(*this);
    }
};
```

就像 constructor 无法真的被虚化一样，non-member function 也是。当我们也可以让其行为视其参数的动态类型而不同。例如：

```cpp
class NLComponent {
public:
    virtual ostream& print(ostream& s) const = 0;
};
class TextBlock: public NLComponent {
public:
    virtual ostream& print(ostream& s);
};
class Graphic: public NLComponent {
public:
    virtual ostream& print(ostream& s);
};

inline ostream& operator<<(ostream& s, const NLComponent& c) {
    return c.print(s);
}
```

### 26. 限制某个 class 产生的对象数量
有时候我们希望 `class` 产生对象的数量是有限制的，方式就是将构造方法私有化，然后提供一个获取 `class` 对象的方法，在这个方法里我们去控制产生对象的数量。

### 27. 要求（或禁止）对象产生于 heap 中
我们可以让析构函数成为 `private` 来要求对象产生于 heap 中。

将 `operator new` 声明为 `private` 禁止让对象产生与 heap 中。

### 28. 智能指针（smart pointer）
智能指针是看起来，用起来和感觉起来都像是内建指针，但提供更多功能的一种对象。通常包括资源管理、自动的重复写码工作。

### 29. 引用计数（reference counting）
引用计数允许多个等值对象共享同一个实值。

当对象运用了引用计数，一旦不再有任何人使用它，便自动销毁自己，建构出垃圾回收机制的一个简单形式。

同时许多对象有相同的值，将那个值存储多次也是件愚蠢的事，共享一份实值不止节省内存，也使程序速度加快。

### 30. 代理类、替身类（proxy class）
大多情况下，代理类可以完美取代所代表的真正对象，将我们 “与真正对象合作” 转移到 “与替身对象合作”。
代理类可以让我们完成某些很困难的行为，例如多维数组、左右值的区分、压抑隐式转换。

### 31. 让函数根据一个以上的对象类型决定如何虚化
人们把 “虚函数调用动作” 称之为一个 **消息分派（message dispatch）**。某个函数调用如果根据两个参数而虚化，称之为 **双分派（double dispatch）**。

C++、Java 等语言并不直接直接双分派，简单的虚函数实现出来的是单分派（single dispatch），但我们可以通过一些策略实现双分派。

### 32. 在未来时态下发展程序
身为开发人员，需要接受事情总是改变的事实，所以我们应该尽量写出可移植的代码、可应对系统改变的代码。
提供完整的 class，即使某些功能暂时用不到，但当新的需求进来，你不太需要回头去修改那些类。

设计你的接口，使有利于共同操作行为，阻止共同的错误。让类能够轻易的被正确的使用，难以被错误的使用。
尽量使你的代码一般化。

### 33. 将非尾端类设计为抽象类
一般性的法则： 继承体系中的非尾端类应该是抽象类。坚持这个法则，有利于整个软件的可靠度、健壮度、精巧度、扩充度。

### 34. 如何在同一个程序中结合 C 和 C++
确定你的 C++ 和 C 编译器产出兼容的目标文件。
将双方都使用的函数声明为 `extern "C"`。
如果可能，尽量在 C++ 中撰写 `main`。
总是以 `delete` 删除 `new` 返回的内存，总是以 `free` 释放 `malloc` 返回的内存。
将两个语言间的数据结构传递限制于 C 所能了解的形式；C++ `struct` 如果内含非虚函数，倒是不受此限。

### 35. 让自己习惯于标准 C++ 语言
学习 C++ 标准程序库，不仅可以增加你的只是，知道如何包装完整的组件运用于自己的软件上面，也可以使你学习如何更有效的运用 C++ 特性，并对如何设计更好的程序库有所体会。
