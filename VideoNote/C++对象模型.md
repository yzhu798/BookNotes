# 对象模型Object Model

class和class之间的关系

## 十六、复合&继承关系下的构造和析构 

（1）`Inheritance`下的构造和析构 ，从内存的角度去分析，**子类包含父类的成分和特性**，即变量和函数

（2）`composition`下的构造和析构

（3）`Inheritance`+`composition`下的构造和析构

```cpp
//继承中的     构造函数与析构调用顺序
//构造顺序： 父类->子类  由内到外
//析构顺序： 子类->父类  由外到内

//继承+组合模式  构造函数与析构调用顺序
Derived::Derived(...):Base(),Component() { ... };
Derived::~Derived(...){ ...  ~Component(), ~Base() };
```

## 十七、关于vptr和vtbl 

> 当类中存在虚函数的时候（不管有几个虚函数），对象的内存占用都会**多出4bytes**，这个空间存放了一个**指向虚表（Virtual table：vtbl）的指针（Virtual Pointer：vptr）**。虚表里放的都是**函数指针**。

![image-20200420151900705](https://gitee.com/yzhu798/bolgImage/raw/master/C++对象模型.assets/image-20200420151900705.png)

**多态的条件**

1.函数由指针调用。

2.指针向上转型（up cast），父类指针指向子类对象。

3.调用的是虚函数。

## 十八、关于this 

（1）通过一个**对象来调用一个函数**，那么对象的地址就是this pointer。传给函数的隐藏的第一个参数。

![image-20200420153452293](https://gitee.com/yzhu798/bolgImage/raw/master/C++对象模型.assets/image-20200420153452293.png)

## 十九、关于Dynamic Binding

(1)`const`成员函数，不修改成员变量；

```cpp
double real () const { return re; }
```

(2)Dynamic Binding

静态绑定

```cpp
class B b;
class A a = (A)b;
a.vfunc1();//静态绑定，对象a调用的是A的虚函数
```

动态绑定

```cpp
//三个条件：a，指针；b，虚函数；c，向上转型
A* pa = new B;//new出来的对象作向上转型为A ，pa就是this pointer，调用的是A的虚函数。
pa ->vfunc1();
pa = &b; //向上转型
pa ->vfunc1(); //动态绑定；
```

## 二十、关于New,Delete 

- 使用`new`表达式时，实际执行了三步操作：

  - `new`表达式调用名为`operator new`（或`operator new[]`）的标准库函数。该函数**分配**一块足够大、原始、未命名的**内存空间**以便存储特定类型的对象（或对象数组）。

  - 编译器调用对应的**构造函数**构造这些对象并**初始化**。

  - 对象被分配了空间并构造完成，**转换指针类型**，返回**指向该对象的指针**。

    ```cpp
    Foo *p = new Foo;
    {
        void *mem = operator new(sizeof(Foo));
    	p = static_cast<Foo*>(mem);
    	p->Foo::Foo();
    }
    ```

  使用`delete`表达式时，实际执行了两步操作：

  - 对指针所指向的对象（或对象数组）执行对应的**析构函数**。
  - 编译器调用名为`operator delete`（或`operator delete[]`）的标准库函数**释放内存空间**。

  配套使用`new` 和`delete`以及`new [ ]` 和`delete[ ]` 。

  ```cpp
  delete p;
  {
  	p->~Foo();
  	operator delete(Foo);   
  }
  ```

## 二十一、Operator new，operator delete

（1）全局的重载

```cpp
重载::operator new 和::operator delete 
重载::operator new [ ] 和 ::operator delete[ ]
```

（2）在class里重载

```cpp
重载::operator new (size_t) 和::operator delete() 
重载::operator new [ ](size_t) 和 ::operator delete[] ()
```

## 二十二、示例 

## 二十三、重载new(),delete()$示例 

## 二十四、Basic_String使用new(extra)扩充申请量  

（1）new有四种：操作符new，operator new ，new[ ]（即array new）和 placement new 