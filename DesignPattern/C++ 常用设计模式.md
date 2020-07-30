# [C++ 常用设计模式](https://www.cnblogs.com/schips/p/12306851.html)

## 背景

设计模式是来源于工业实践的重要开发经验,它实际上是面向对象的数据结构,掌握设计模式是掌握面向对象设计的根本要求。

> 原文：[《C++ 常用设计模式》](https://www.cnblogs.com/chengjundu/p/8473564.html) (已经根据比较好的学习顺序进行了排序)

## 1、工厂模式(Factory)

在工厂模式中，我们在**创建对象时不会对客户端暴露创建逻辑**，并且是通过使用一个共同的接口来指向新创建的对象。工厂模式作为一种创建模式，一般在创建复杂对象时，考虑使用；在创建简单对象时，建议直接new完成一个实例对象的创建。

### 1.1、简单工厂模式

主要特点是需要**在工厂类中做判断，从而创造相应的产品，当增加新产品时，需要修改工厂类**。使用简单工厂模式，我们只需要知道具体的产品型号就可以创建一个产品。

缺点：**工厂类集中了所有产品类的创建逻辑**，如果**产品量较大，会使得工厂类变的非常臃肿**。

```cpp
/*
关键代码：创建过程在工厂类中完成。
*/
//定义产品类型信息
typedef enum{
    Tank_Type_56,
    Tank_Type_96,
    Tank_Type_Num
}Tank_Type;

//抽象产品类
class Tank{}
//具体的产品类
class Tank56 : public Tank{};
//具体的产品类
class Tank96 : public Tank{};

//工厂类
class TankFactory{
public:
    //根据产品信息创建具体的产品类实例，返回一个抽象产品类
    Tank* createTank(Tank_Type type){
        switch(type){
        case Tank_Type_56:
            return new Tank56();
        case Tank_Type_96:
            return new Tank96();
        default:
            return nullptr;
        }
    }
};


int main()
{
    TankFactory* factory = new TankFactory();
    Tank* tank56 = factory->createTank(Tank_Type::Tank_Type_56);

    delete tank56;
    tank56 = nullptr;
    delete factory;
    factory = nullptr;

    return 0;
}
```

### 1.2、工厂方法模式

**定义一个创建对象的接口，其子类去具体现实这个接口以完成具体的创建工作**。如果需要增加新的产品类，只需要扩展一个相应的工厂类即可。

缺点：产品类数据较多时，**需要实现大量的工厂类**，这无疑增加了代码量。

```cpp
/*关键代码：创建过程在其子类执行。*/

//产品抽象类
class Tank{};
//抽象工厂类，提供一个创建接口
class TankFactory{
public:
    //提供创建产品实例的接口，返回抽象产品类
    virtual Tank* createTank() = 0;
};
//具体的产品类
class Tank56 : public Tank{};
class Tank96 : public Tank{};

//具体的创建工厂类，使用抽象工厂类提供的接口，去创建具体的产品实例
class Tank56Factory : public TankFactory{
public:
    Tank* createTank() override{
        return new Tank56();
    }
};
class Tank96Factory : public TankFactory{
public:
    Tank* createTank() override{
        return new Tank96();
    }
};

int main()
{
    TankFactory* factory56 = new Tank56Factory();
    Tank* tank56 = factory56->createTank();
    
    delete tank56;
    tank56 = nullptr;
    delete factory56;
    factory56 = nullptr;

    return 0;
}
```

### 1.3、抽象工厂模式

**抽象工厂模式提供创建一系列相关或相互依赖对象的接口，而无需指定它们具体的类**。

当存在多个产品系列，而客户端只使用一个系列的产品时，可以考虑使用抽象工厂模式。

缺点：当增加一个新系列的产品时，不仅需要现实具体的产品类，还需要增加一个新的创建接口，**扩展相对困难**。

```cpp
/*
* 关键代码：在一个工厂里聚合多个同类产品。
* 以下代码以白色衣服和黑色衣服为例，白色衣服为一个产品系列，黑色衣服为一个产品系列。白色上衣搭配白色裤子，   黑色上衣搭配黑色裤字。每个系列的衣服由一个对应的工厂创建，这样一个工厂创建的衣服能保证衣服为同一个系列。
*/

//抽象上衣类
class Coat{
public:
    virtual const string& color() = 0;
};

//黑色上衣类
class BlackCoat : public Coat{};

//白色上衣类
class WhiteCoat : public Coat{}; 

//抽象裤子类
class Pants
{
public:
    virtual const string& color() = 0;
};

//黑色裤子类
class BlackPants : public Pants{}; 

//白色裤子类
class WhitePants : public Pants{}; 

//抽象工厂类，提供衣服创建接口
class Factory
{
public:
    //上衣创建接口，返回抽象上衣类
    virtual Coat* createCoat() = 0;
    //裤子创建接口，返回抽象裤子类
    virtual Pants* createPants() = 0;
};

//创建白色衣服的工厂类，具体实现创建白色上衣和白色裤子的接口
class WhiteFactory : public Factory{
        // return new WhiteCoat();
        // return new WhitePants();
};

//创建黑色衣服的工厂类，具体实现创建黑色上衣和白色裤子的接口
class BlackFactory : public Factory{
        // return new BlackCoat();
        // return new BlackPants();
};
```

## 2、单例模式(Singleton)

单例模式顾名思义，保证类仅有一个实例化对象，且提供一个可以访问它的全局接口。实现单例模式必须注意一下几点：

- 单例类仅有一个实例化对象。
- 单例类必须自己提供一个实例化对象。
- 单例类必须提供一个可以访问唯一实例化对象的接口。

单例模式分为懒汉和饿汉两种实现方式。

### 2.1、懒汉单例模式

**懒汉：故名思义，不到万不得已就不会去实例化类**，也就是说在第一次用到类实例的时候才会去实例化一个对象。在访问量较小，甚至可能不会去访问的情况下，采用懒汉实现，这是以时间换空间。

#### 2.1.1、非线程安全的懒汉单例模式

```cpp
//懒汉式，线程非安全版本，需要delete，用智能指针
class Singleton{
private:
    Singleton(){}                                    //构造函数私有
    Singleton(const Singleton&) = delete;            //明确拒绝
    Singleton& operator=(const Singleton&) = delete; //明确拒绝
public:
    static Singleton* getInstance();
    static Singleton& getInstance();//C++11，返回一个reference指向local static对象

    static Singleton* m_instance;// 静态(static)成员
};
Singleton* Singleton::m_instance=nullptr;

Singleton* Singleton::getInstance() {
    if (m_instance == nullptr) {
        m_instance = new Singleton();
    }
    return m_instance;
}
```

#### 2.1.2、线程安全的懒汉单例模式

```cpp
//懒汉，线程安全版本，但锁的代价过高
Singleton* Singleton::getInstance() {
    Lock lock;
    if (m_instance == nullptr) {
        m_instance = new Singleton();
    }
    return m_instance;
}
```

#### 2.1.3、双检查锁，线程非安全版本，但由于内存读写reorder不安全

```cpp
//std::mutex sm_mutex;
//双检查锁，线程非安全版本，但由于内存读写reorder不安全
Singleton* Singleton::getInstance() {

    if(m_instance==nullptr){
        Lock lock;//std::lock_guard<std::mutex> guard(sm_mutex);
        if (m_instance == nullptr) {
            m_instance = new Singleton();
        }
    }
    return m_instance;
}
```



#### 2.1.4、返回一个reference指向local static对象

**C++11**之前，这种单例模式实现方式多线程可能存在不确定性：任何一种non-const static对象，不论它是local或non-local,在多线程环境下“等待某事发生”都会有麻烦。

解决的方法：在程序的**单线程启动阶段手工调用所有reference-returning函数**。这种实现方式的好处是不需要去delete它。

**C++11**，若当变量在初始化时，并发同时进入声明语句，并发线程将会阻塞等待初始化结束。**所以具有线程安全性。**

```cpp
class Singleton
{
public:
    static Singleton& getInstance();
private:
    Singleton(){}
    Singleton(const Singleton&) = delete;  //明确拒绝
    Singleton& operator=(const Singleton&) = delete; //明确拒绝
};

//C++11，若当变量在初始化时，并发同时进入声明语句，并发线程将会阻塞等待初始化结束。所以具有线程安全性。
Singleton& Singleton::getInstance()
{
    static Singleton singleton;
    return singleton;
}
```

#### 2.1.5 volatile方式

```cpp
//C++ 11版本之后的跨平台实现 (volatile)
std::atomic<Singleton*> Singleton::m_instance;
std::mutex Singleton::m_mutex;

Singleton* Singleton::getInstance() {
    Singleton* tmp = m_instance.load(std::memory_order_relaxed);
    std::atomic_thread_fence(std::memory_order_acquire);//获取内存fence
    if (tmp == nullptr) {
        std::lock_guard<std::mutex> lock(m_mutex);
        tmp = m_instance.load(std::memory_order_relaxed);
        if (tmp == nullptr) {
            tmp = new Singleton;
            std::atomic_thread_fence(std::memory_order_release);//释放内存fence
            m_instance.store(tmp, std::memory_order_relaxed);
        }
    }
    return tmp;
}
```



### 2.2、饿汉单例模式

**饿汉：饿了肯定要饥不择食。所以在单例类定义的时候就进行实例化**。

> 在**访问量比较大**，或者可能访问的**线程比较多**时，**采用饿汉实现**，可以实现更好的性能。这是**以空间换时间。**

```cpp
//饿汉式：线程安全，注意一定要在合适的地方去delete它
class Singleton
{
public:
    static Singleton* getInstance();
private:
    Singleton(){}                                    //构造函数私有
    Singleton(const Singleton&) = delete;            //明确拒绝
    Singleton& operator=(const Singleton&) = delete; //明确拒绝

    static Singleton* m_pSingleton;
};

Singleton* Singleton::m_pSingleton = new Singleton();

Singleton* Singleton::getInstance()
{
    return m_pSingleton;
}
```

## 3、外观模式(Facade)

外观模式：为子系统中的一组接口定义一个一致的界面；**外观模式提供一个高层的接口**，这个接口使得这一子系统更加容易被使用；对于复杂的系统，系统为客户端提供一个简单的接口，把负责的实现过程封装起来，客户端不需要连接系统内部的细节。

以下情形建议考虑外观模式：

- 设计初期阶段，应有意识的**将不同层分离，层与层之间建立外观模式**。
- 开发阶段，子系统越来越复杂，使用外观模式提供一个简单的调用接口。
- 一个系统可能已经非常难易维护和扩展，但又包含了非常重要的功能，可以为其开发一个外观类，使得新系统可以方便的与其交互。

优点：

- 实现了子系统与客户端之间的松耦合关系。
- **客户端屏蔽了子系统组件，减少了客户端所需要处理的对象数据，使得子系统使用起来更方便容易。**
- 更好的划分了设计层次，对于后期维护更加的容易。

```cpp
/*
* 关键代码：客户与系统之间加一个外观层，外观层处理系统的调用关系、依赖关系等。
*以下实例以电脑的启动过程为例，客户端只关心电脑开机的、关机的过程，并不需要了解电脑内部子系统的启动过程。
*/
#include <iostream>
using namespace std;

//抽象控件类，提供接口
class Control{
public:
    virtual void start() = 0;
    virtual void shutdown() = 0;
};

//子控件， 主机
class Host : public Control{};

//子控件， 显示屏
class LCDDisplay : public Control{};

//子控件， 外部设备
class Peripheral : public Control{};
class Computer{
public:
    void start()
    {
        m_host.start();
        m_display.start();
        m_peripheral.start();
        cout << "Computer start" << endl;
    }
    void shutdown()
    {
        m_host.shutdown();
        m_display.shutdown();
        m_peripheral.shutdown();
        cout << "Computer shutdown" << endl;
    }
private:
    Host   m_host;
    LCDDisplay m_display;
    Peripheral   m_peripheral;
};

int main(){
    Computer computer;
    computer.start();
    //do something
    computer.shutdown();
    return 0;
}
```

## 4、模板模式(Template)

模板模式：定义一个操作中的算法的骨架，而将一些步骤延迟到子类中。**模板方法使得子类可以不改变一个算法的结构即可重定义该算法的某些特定步骤**。

当多个类有相同的方法，并且逻辑相同，只是细节上有差异时，可以考虑使用模板模式。具体的实现上可以将相同的核心算法设计为模板方法，具体的实现细节有子类实现。

**缺点:每一个不同的实现都需要一个子类来实现，导致类的个数增加，使得系统更加庞大**。

以生产电脑为例，电脑生产的过程都是一样的，只是一些装配的器件可能不同而已。

```cpp
/*
* 关键代码：在抽象类实现通用接口，细节变化在子类实现。
*/
#include <iostream>
#include <unique_ptr>
using namespace std;

class Computer{
public:
    void product(){
        installCpu();
        installRam();
        installGraphicsCard();
    }
protected:
    virtual void installCpu() = 0;
    void installRam(){cout << "Computer install 16G Ram" << endl;}
    virtual void installGraphicsCard() = 0;

};

class ComputerA : public Computer{
protected:
    void installCpu() override{
        cout << "ComputerA install Inter Core i5" << endl;
    }

    void installGraphicsCard() override{
        cout << "ComputerA install Gtx940 GraphicsCard" << endl;
    }
};

class ComputerB : public Computer{
protected:
    void installCpu() override{
        cout << "ComputerB install Inter Core i7" << endl;
    }
    void installGraphicsCard() override{
        cout << "ComputerB install Gtx960 GraphicsCard" << endl;
    }
};

int main(){
    unique_ptr<ComputerB> ptest(new ComputerB());
    ptest->product();
    return 0;
}
```

## 5、组合模式(Composite)

组合模式：将对象组合成树形结构以表示“部分-整体”的层次结构，**组合模式使得客户端对单个对象和组合对象的使用一致**。

既然讲到以树形结构表示“部分-整体”，那可以将组合模式想象成一根大树，**将大树分成树枝和树叶两部分**，**树枝上可以再长树枝，也可以长树叶**。

以下情况可以考虑使用组合模式：

- 希望表示对象的部分-整体层次结构。
- 希望客户端忽略组合对象与单个对象的不同，客户端将统一的使用组合结构中的所有对象。

```cpp
class Component{//接口
public:
    virtual void process() = 0;
    virtual ~Component(){}
};

//树节点
class Composite : public Component{//继承接口Component
    string name;
    list<Component*> elements;//组合，
public:
    Composite(const string & s) : name(s) {}
    void add(Component* element) {
        elements.push_back(element);
    }
    void remove(Component* element){
        elements.remove(element);
        // if((*iter).get()->name() == strName)
    }
    
    void process(){
        //1. process current node
        //2. process leaf nodes
        for (auto &e : elements)
            e->process(); //多态调用
    }
};

//叶子节点
class Leaf : public Component{
    string name;
public:
    Leaf(string s) : name(s) {}
    void process(){
        //process current node
    }
};


void Invoke(Component & c){
    //...
    c.process();
    //...
}


int main()
{

    Composite root("root");
    Composite treeNode1("treeNode1");
    Composite treeNode2("treeNode2");
    Composite treeNode3("treeNode3");
    Composite treeNode4("treeNode4");
    Leaf leat1("left1");
    Leaf leat2("left2");
    
    root.add(&treeNode1);
    treeNode1.add(&treeNode2);
    treeNode2.add(&leaf1);
    
    root.add(&treeNode3);
    treeNode3.add(&treeNode4);
    treeNode4.add(&leaf2);
    
    process(root);
    process(leaf2);
    process(treeNode3);
  
}
```

## 6、代理模式

**代理模式**：为其它对象提供一种代理以控制这个对象的访问。在某些情况下，一个对象不适合或者不能直接引用另一个对象，而代理对象可以在客户端和目标对象之间起到中介作用。

优点：

- 职责清晰。**真实的角色只负责实现实际的业务逻辑**，不用关心其它非本职责的事务，通过后期的代理完成具体的任务。这样代码会简洁清晰。
- **代理对象可以在客户端和目标对象之间起到中介的作用，这样就保护了目标对象**。
- 扩展性好。

```cpp
/*
* 关键代码：一个是真正的你要访问的对象(目标类)，一个是代理对象,真正对象与代理对象实现同一个接口,先访问代理*         类再访问真正要访问的对象。
*/
class ISubject{
public:
    virtual void process();
};//抽象类

class RealSubject: public ISubject{
public:
    virtual void process(){}
};

//Proxy的设计
class SubjectProxy: public ISubject{
public:
    virtual void process(){}
};

// ClientApp
class ClientApp{
    ISubject* subject;
public:
    ClientApp(){
        subject=new SubjectProxy();
       // subject=new RealSubject();
    }
    void DoTask(){
        subject->process();
    }
};
```

## 7、观察者模式(Observer)

**观察者模式**：定义对象间的一种一对多的依赖关系，**当一个对象的状态发生改变时，所有依赖于它的对象都要得到通知并自动更新**。

观察者模式从根本上讲必须包含两个角色：观察者和被观察对象。

> - 被观察对象自身应该包含一个容器来存放观察者对象，当被观察者自身发生改变时通知容器内所有的观察者对象自动更新。
> - **观察者对象可以注册到被观察者的中**，完成注册后可以检测被观察者的变化，接收被观察者的通知。
> - **当然观察者也可以被注销掉**，停止对被观察者的监控。
>

```cpp
/*
* 关键代码：在目标类中增加一个ArrayList来存放观察者们。
*/
#include <iostream>
#include <list>
#include <memory>

using namespace std;
class View;

//被观察者抽象类   数据模型
class DataModel{
public:
    virtual ~DataModel(){}
    virtual void addView(View* view) = 0;
    virtual void removeView(View* view) = 0;
    virtual void notify() = 0;   //通知函数
};

//观察者抽象类   视图
class View{
public:
    virtual ~View(){ cout << "~View()" << endl; }
    virtual void update() = 0;
    virtual void setViewName(const string& name) = 0;
    virtual const string& name() = 0;
};

//具体的被观察类， 整数模型
class IntDataModel:public DataModel{
public:
    ~IntDataModel(){
        m_pViewList.clear();
    }

    virtual void addView(View* view) override{
        shared_ptr<View> temp(view);
        auto iter = find(m_pViewList.begin(), m_pViewList.end(), temp);
        if(iter == m_pViewList.end()){
            m_pViewList.push_front(temp);
        }
        else{
            cout << "View already exists" << endl;
        }
    }

    void removeView(View* view) override{
        auto iter = m_pViewList.begin();
        for(; iter != m_pViewList.end(); iter++){
            if((*iter).get() == view){
                m_pViewList.erase(iter);
                cout << "remove view" << endl;
                return;
            }
        }
    }

    virtual void notify() override{
        auto iter = m_pViewList.begin();
        for(; iter != m_pViewList.end(); iter++){
            (*iter).get()->update();
        }
    }

private:
    list<shared_ptr<View>> m_pViewList; 
};

//具体的观察者类    表视图
class TableView : public View{
public:
    TableView() : m_name("unknow"){}
    TableView(const string& name) : m_name(name){}
    ~TableView(){ cout << "~TableView(): " << m_name.data() << endl; }

    void setViewName(const string& name){
        m_name = name;
    }

    const string& name(){
        return m_name;
    }

    void update() override{
        cout << m_name.data() << " update" << endl;
    }

private:
    string m_name;
};

int main()
{
    /*
    * 这里需要补充说明的是在此示例代码中，View一旦被注册到DataModel类之后，DataModel解析时会自动解析掉     * 内部容器中存储的View对象，因此注册后的View对象不需要在手动去delete，再去delete View对象会出错。
    */
    
    View* v1 = new TableView("TableView1");
    View* v2 = new TableView("TableView2");
    View* v3 = new TableView("TableView3");
    View* v4 = new TableView("TableView4");

    IntDataModel* model = new IntDataModel;
    model->addView(v1);
    model->addView(v2);
    model->addView(v3);
    model->addView(v4);

    model->notify();

    cout << "-------------\n" << endl;

    model->removeView(v1);

    model->notify();

    delete model;
    model = nullptr;

    return 0;
}
```

## 8、策略模式(Strategy)

**策略模式是指定义一系列的算法，把它们单独封装起来，并且使它们可以互相替换**，使得算法可以独立于使用它的客户端而变化，也是说这些算法所完成的功能类型是一样的，对外接口也是一样的，只是不同的策略为引起环境角色环境角色表现出不同的行为。

**相比于使用大量的if...else，使用策略模式可以降低复杂度，使得代码更容易维护。**

**缺点：可能需要定义大量的策略类，并且这些策略类都要提供给客户端。**

[环境角色] 持有一个策略类的引用，最终给客户端调用。

### 8.1、传统的策略模式实现

```cpp
/*
* 关键代码：实现同一个接口。
* 以下代码实例中，以游戏角色不同的攻击方式为不同的策略，游戏角色即为执行不同策略的环境角色。
*/

#include <iostream>

using namespace std;

//抽象策略类，提供一个接口
class Hurt{
public:
    virtual void blood() = 0;
};

//具体的策略实现类，Adc持续普通攻击
class AdcHurt : public Hurt{
public:
    void blood() override{cout << "Adc hurt, Blood loss" << endl;}
};

//具体的策略实现类， Apc技能攻击
class ApcHurt : public Hurt{
public:
    void blood() override{cout << "Apc Hurt, Blood loss" << endl;}
};

//环境角色类， 游戏角色战士，传入一个策略类指针参数。
class Soldier{
public:
    Soldier(Hurt* hurt):m_pHurt(hurt){}
    //在不同的策略下，该游戏角色表现出不同的攻击
    void attack(){
        m_pHurt->blood();
    }
private:
    Hurt* m_pHurt;
};

//定义策略标签
typedef enum
{
    Hurt_Type_Adc,
    Hurt_Type_Apc,
    Hurt_Type_Num
}HurtType;

//环境角色类， 游戏角色法师，传入一个策略标签参数。
class Mage{
public:
    Mage(HurtType type){
        switch(type){
        case Hurt_Type_Adc:
            m_pHurt = new AdcHurt();
            break;
        case Hurt_Type_Apc:
            m_pHurt = new ApcHurt();
            break;
        default:
            break;
        }
    }
    ~Mage(){
        delete m_pHurt;
        m_pHurt = nullptr;
        cout << "~Mage()" << endl;
    }

    void attack(){m_pHurt->blood();}
private:
    Hurt* m_pHurt;
};

//环境角色类， 游戏角色弓箭手，实现模板传递策略。
template<typename T>
class Archer{
public:
    void attack(){m_hurt.blood();}
private:
    T m_hurt;
};

int main()
{
    Archer<ApcHurt>* arc = new Archer<ApcHurt>;
    arc->attack();

    delete arc;
    arc = nullptr;
    
    return 0;
}
```

### 8.2、使用函数指针实现策略模式

```cpp
#include <iostream>
#include <functional> 

void adcHurt(){
    std::cout << "Adc Hurt" << std::endl;
}

void apcHurt(){
    std::cout << "Apc Hurt" << std::endl;
}

//环境角色类， 使用传统的函数指针
class Soldier{
public:
    typedef void (*Function)();
    Soldier(Function fun): m_fun(fun){}
    void attack(){m_fun();}
private:
    Function m_fun;
};

//环境角色类， 使用std::function<>
class Mage{
public:
    typedef std::function<void()> Function;

    Mage(Function fun): m_fun(fun){}
    void attack(){m_fun();}
private:
    Function m_fun;
};

int main(){
    Soldier* soldier = new Soldier(apcHurt);
    soldier->attack();
    delete soldier;
    soldier = nullptr;
    return 0;
}
```

## 9、建造者模式(Builder)

建造者模式：将复杂对象的构建和其表示分离，使得相同的构建过程可以产生不同的表示。

以下情形可以考虑使用建造者模式：

- 对象的创建复杂，但是其各个部分的子对象创建算法一定。
- 需求变化大，构造复杂对象的子对象经常变化，但将其组合在一起的算法相对稳定。

建造者模式的优点：

- 将对象的创建和表示分离，客户端不需要了解具体的构建细节。
- 增加新的产品对象时，只需要增加其具体的建造类即可，不需要修改原来的代码，扩展方便。

产品之间差异性大，内部变化较大、较复杂时不建议使用建造者模式。

```cpp
/*
*关键代码：建造者类：创建和提供实例； Director类：管理建造出来的实例的依赖关系。
*/

#include <iostream>
#include <string>

using namespace std;

//具体的产品类
class Order
{
public:
    void setFood(const string& food)
    {
        m_strFood = food;
    }

    const string& food()
    {
        cout << m_strFood.data() << endl;
        return m_strFood;
    }
    
    void setDrink(const string& drink)
    {
        m_strDrink = drink;
    }

    const string& drink()
    {
        cout << m_strDrink << endl;
        return m_strDrink;
    }

private:
    string m_strFood;
    string m_strDrink;
};

//抽象建造类，提供建造接口。
class OrderBuilder
{
public:
    virtual ~OrderBuilder()
    {
        cout << "~OrderBuilder()" << endl;
    }
    virtual void setOrderFood() = 0;
    virtual void setOrderDrink() = 0;
    virtual Order* getOrder() = 0;
};

//具体的建造类
class VegetarianOrderBuilder : public OrderBuilder 
{
public:
    VegetarianOrderBuilder()
    {
        m_pOrder = new Order;
    }

    ~VegetarianOrderBuilder()
    {
        cout << "~VegetarianOrderBuilder()" << endl;
        delete m_pOrder;
        m_pOrder = nullptr;
    }

    void setOrderFood() override
    {
        m_pOrder->setFood("vegetable salad");
    }

    void setOrderDrink() override
    {
        m_pOrder->setDrink("water");
    }

    Order* getOrder() override
    {
        return m_pOrder;
    }

private:
    Order* m_pOrder;
};

//具体的建造类
class MeatOrderBuilder : public OrderBuilder
{
public:
    MeatOrderBuilder()
    {
        m_pOrder = new Order;
    }
    ~MeatOrderBuilder()
    {
        cout << "~MeatOrderBuilder()" << endl;
        delete m_pOrder;
        m_pOrder = nullptr;
    }

    void setOrderFood() override
    {
        m_pOrder->setFood("beef");
    }

    void setOrderDrink() override
    {
        m_pOrder->setDrink("beer");
    }

    Order* getOrder() override
    {
        return m_pOrder;
    }

private:
    Order* m_pOrder;
};

//Director类，负责管理实例创建的依赖关系，指挥构建者类创建实例
class Director
{
public:
    Director(OrderBuilder* builder) : m_pOrderBuilder(builder)
    {
    }
    void construct()
    {
        m_pOrderBuilder->setOrderFood();
        m_pOrderBuilder->setOrderDrink();
    }

private:
    OrderBuilder* m_pOrderBuilder;
};


int main()
{
//  MeatOrderBuilder* mBuilder = new MeatOrderBuilder;
    OrderBuilder* mBuilder = new MeatOrderBuilder;  //注意抽象构建类必须有虚析构函数，解析时才会                                                      调用子类的析构函数
    Director* director = new Director(mBuilder);
    director->construct();
Order* order = mBuilder->getOrder();
order->food();
order->drink();

delete director;
director = nullptr;

delete mBuilder;
mBuilder = nullptr;

return 0;
}
```

## 10、适配器模式(Adapter)

适配器模式可以将一个类的接口转换成客户端希望的另一个接口，使得原来由于接口不兼容而不能在一起工作的那些类可以在一起工作。通俗的讲就是当我们已经有了一些类，而这些类不能满足新的需求，此时就可以考虑是否能将现有的类适配成可以满足新需求的类。适配器类需要继承或依赖已有的类，实现想要的目标接口。

缺点：过多地使用适配器，会让系统非常零乱，不易整体进行把握。比如，明明看到调用的是 A 接口，其实内部被适配成了 B 接口的实现，一个系统如果太多出现这种情况，无异于一场灾难。因此如果不是很有必要，可以不使用适配器，而是直接对系统进行重构。

### 9.1、使用复合实现适配器模式

```cpp
/*
* 关键代码：适配器继承或依赖已有的对象，实现想要的目标接口。
* 以下示例中，假设我们之前有了一个双端队列，新的需求要求使用栈和队列来完成。
  双端队列可以在头尾删减或增加元素。而栈是一种先进后出的数据结构，添加数据时添加到栈的顶部，删除数据时先删   除栈顶部的数据。因此我们完全可以将一个现有的双端队列适配成一个栈。
*/

//双端队列， 被适配类
class Deque
{
public:
    void push_back(int x)
    {
        cout << "Deque push_back:" << x << endl;
    }
    void push_front(int x)
    {
        cout << "Deque push_front:" << x << endl;
    }
    void pop_back()
    {
        cout << "Deque pop_back" << endl;
    }
    void pop_front()
    {
        cout << "Deque pop_front" << endl;
    }
};

//顺序类，抽象目标类
class Sequence  
{
public:
    virtual void push(int x) = 0;
    virtual void pop() = 0;
};

//栈,后进先出, 适配类
class Stack:public Sequence   
{
public:
    //将元素添加到堆栈的顶部。
    void push(int x) override
    {
        m_deque.push_front(x);
    }
    //从堆栈中删除顶部元素
    void pop() override
    {
        m_deque.pop_front();
    }
private:
    Deque m_deque;
};

//队列，先进先出，适配类
class Queue:public Sequence  
{
public:
    //将元素添加到队列尾部
    void push(int x) override
    {
        m_deque.push_back(x);
    }
    //从队列中删除顶部元素
    void pop() override
    {
        m_deque.pop_front();
    }
private:
    Deque m_deque;
};
```

### 9.2、使用继承实现适配器模式

```cpp
//双端队列，被适配类
class Deque  
{
public:
    void push_back(int x)
    {
        cout << "Deque push_back:" << x << endl;
    }
    void push_front(int x)
    {
        cout << "Deque push_front:" << x << endl;
    }
    void pop_back()
    {
        cout << "Deque pop_back" << endl;
    }
    void pop_front()
    {
        cout << "Deque pop_front" << endl;
    }
};

//顺序类，抽象目标类
class Sequence  
{
public:
    virtual void push(int x) = 0;
    virtual void pop() = 0;
};

//栈,后进先出, 适配类
class Stack:public Sequence, private Deque   
{
public:
    void push(int x)
    {
        push_front(x);
    }
    void pop()
    {
        pop_front();
    }
};

//队列，先进先出，适配类
class Queue:public Sequence, private Deque 
{
public:
    void push(int x)
    {
        push_back(x);
    }
    void pop()
    {
        pop_front();
    }
};
```

## 11、桥接模式(Bridge)

桥接模式：将抽象部分与实现部分分离，使它们都可以独立变换。

以下情形考虑使用桥接模式：

- 当一个对象有多个变化因素的时候，考虑依赖于抽象的实现，而不是具体的实现。
- 当多个变化因素在多个对象间共享时，考虑将这部分变化的部分抽象出来再聚合/合成进来。
- 当一个对象的多个变化因素可以动态变化的时候。

优点：

- 将实现抽离出来，再实现抽象，使得对象的具体实现依赖于抽象，满足了依赖倒转原则。
- 更好的可扩展性。
- 可动态的切换实现。桥接模式实现了抽象和实现的分离，在实现桥接模式时，就可以实现动态的选择具体的实现。

```cpp
/*
* 关键代码：将现实独立出来，抽象类依赖现实类。
* 以下示例中，将各类App、各类手机独立开来，实现各种App和各种手机的自由桥接。
*/
#include <iostream>

using namespace std;

//抽象App类，提供接口
class App
{
public:
    virtual ~App(){ cout << "~App()" << endl; }
    virtual void run() = 0;
};

//具体的App实现类
class GameApp:public App
{
public:
    void run()
    {
        cout << "GameApp Running" << endl;
    }
};

//具体的App实现类
class TranslateApp:public App
{
public:
    void run()
    {
        cout << "TranslateApp Running" << endl;
    }
};

//抽象手机类，提供接口
class MobilePhone
{
public:
    virtual ~MobilePhone(){ cout << "~MobilePhone()" << endl;}
    virtual void appRun(App* app) = 0;  //实现App与手机的桥接
};

//具体的手机实现类
class XiaoMi:public MobilePhone
{
public:
    void appRun(App* app)
    {
        cout << "XiaoMi: ";
        app->run();
    }
};

//具体的手机实现类
class HuaWei:public MobilePhone
{
public:
    void appRun(App* app)
    {
        cout << "HuaWei: ";
        app->run();
    }
};

int main()
{
    App* gameApp = new GameApp;
    App* translateApp = new TranslateApp;
    MobilePhone* mi = new XiaoMi;
    MobilePhone* hua = new HuaWei;
    mi->appRun(gameApp);
    mi->appRun(translateApp);
    hua->appRun(gameApp);
    hua->appRun(translateApp);

    delete hua;
    hua = nullptr;
    delete mi;
    mi = nullptr;
    delete gameApp;
    gameApp = nullptr;
    delete translateApp;
    translateApp = nullptr;

    return 0;
}
```

## 12、装饰模式(Decorator)

装饰模式：动态地给一个对象添加一些额外的功能，它是通过创建一个包装对象，也就是装饰来包裹真实的对象。新增加功能来说，装饰器模式比生产子类更加灵活。

以下情形考虑使用装饰模式：

- 需要扩展一个类的功能，或给一个类添加附加职责。
- 需要动态的给一个对象添加功能，这些功能可以再动态的撤销。
- 需要增加由一些基本功能的排列组合而产生的非常大量的功能，从而使继承关系变的不现实。
- 当不能采用生成子类的方法进行扩充时。一种情况是，可能有大量独立的扩展，为支持每一种组合将产生大量的子类，使得子类数目呈爆炸性增长。另一种情况可能是因为类定义被隐藏，或类定义不能用于生成子类。

```cpp
/*
* 关键代码：1、Component 类充当抽象角色，不应该具体实现。 2、修饰类引用和继承 Component 类，具体扩展类重写父类方法。
*/
#include <iostream>

using namespace std;

//抽象构件（Component）角色：给出一个抽象接口，以规范准备接收附加责任的对象。
class Component
{
public:
    virtual ~Component(){}

    virtual void configuration() = 0;
};

//具体构件（Concrete Component）角色：定义一个将要接收附加责任的类。
class Car : public Component
{
public:
    void configuration() override
    {
        cout << "A Car" << endl;
    }
};

//装饰（Decorator）角色：持有一个构件（Component）对象的实例，并实现一个与抽象构件接口一致的接口。
class DecorateCar : public Component
{
public:
    DecorateCar(Component* car) : m_pCar(car){}

    void configuration() override
    {
        m_pCar->configuration();
    }

private:
    Component* m_pCar;
};

//具体装饰（Concrete Decorator）角色：负责给构件对象添加上附加的责任。
class DecorateLED : public DecorateCar
{
public:
    DecorateLED(Component* car) : DecorateCar(car){}

    void configuration() override
    {
        DecorateCar::configuration();
        addLED();
    }

private:
    void addLED()
    {
        cout << "Install LED" << endl;
    }

};

//具体装饰（Concrete Decorator）角色：负责给构件对象添加上附加的责任。
class DecoratePC : public DecorateCar
{
public:
    DecoratePC(Component* car) : DecorateCar(car){}

    void configuration() override
    {
        DecorateCar::configuration();
        addPC();
    }

private:
    void addPC()
    {
        cout << "Install PC" << endl;
    }
};

//具体装饰（Concrete Decorator）角色：负责给构件对象添加上附加的责任。
class DecorateEPB : public DecorateCar
{
public:
    DecorateEPB(Component* car) : DecorateCar(car){}

    void configuration() override
    {
        DecorateCar::configuration();
        addEPB();
    }

private:
    void addEPB()
    {
        cout << "Install Electrical Park Brake" << endl;
    }
};

int main()
{
    Car* car = new Car;
    DecorateLED* ledCar = new DecorateLED(car);
    DecoratePC* pcCar = new DecoratePC(ledCar);
    DecorateEPB* epbCar = new DecorateEPB(pcCar);

    epbCar->configuration();

    delete epbCar;
    epbCar = nullptr;

    delete pcCar;
    pcCar = nullptr;

    delete ledCar;
    ledCar = nullptr;

    delete car;
    car = nullptr;

    return 0;
}
```

## 13、中介者模式(Mediator)

中介者模式：用一个中介对象来封装一系列的对象交互，中介者使各对象不需要显示地相互引用，从而使其耦合松散，而且可以独立地改变它们之前的交互。

如果对象与对象之前存在大量的关联关系，若一个对象改变，常常需要跟踪与之关联的对象，并做出相应的处理，这样势必会造成系统变得复杂，遇到这种情形可以考虑使用中介者模式。当多个对象存在关联关系时，为它们设计一个中介对象，当一个对象改变时，只需要通知它的中介对象，再由它的中介对象通知每个与它相关的对象。

```cpp
/*
* 关键代码：将相关对象的通信封装到一个类中单独处理。
*/
#include <iostream>

using namespace std;

class Mediator;

//抽象同事类。
class Businessman
{
public:
    Businessman(){}
    Businessman(Mediator* mediator) : m_pMediator(mediator){}

    virtual ~Businessman(){}

    virtual void setMediator(Mediator* m)
    {
        m_pMediator = m;
    }

    virtual void sendMessage(const string& msg) = 0;
    virtual void getMessage(const string& msg) = 0;

protected:
    Mediator* m_pMediator;
};

//抽象中介者类。
class  Mediator
{
public:
    virtual ~Mediator(){}
    virtual void setBuyer(Businessman* buyer) = 0;
    virtual void setSeller(Businessman* seller) = 0;
    virtual void send(const string& msg, Businessman* man) = 0;
};

//具体同事类
class Buyer : public Businessman
{
public:
    Buyer() : Businessman(){}
    Buyer(Mediator* mediator) : Businessman(mediator){}

    void sendMessage(const string& msg) override
    {
        m_pMediator->send(msg, this);
    }

    void getMessage(const string& msg)
    {
        cout << "Buyer recv: " << msg.data() << endl;
    }
};

//具体同事类
class Seller : public Businessman
{
public:
    Seller() : Businessman(){}
    Seller(Mediator* mediator) : Businessman(mediator){}

    void sendMessage(const string& msg) override
    {
        m_pMediator->send(msg, this);
    }

    void getMessage(const string& msg)
    {
        cout << "Seller recv: " << msg.data() << endl;
    }
};

//具体中介者类
class HouseMediator : public Mediator
{
public:
    void setBuyer(Businessman* buyer) override
    {
        m_pBuyer = buyer;
    }

    void setSeller(Businessman* seller) override
    {
        m_pSeller = seller;
    }

    void send(const string& msg, Businessman* man) override
    {
        if(man == m_pBuyer)
        {
            m_pSeller->getMessage(msg);
        }
        else if(man == m_pSeller)
        {
            m_pBuyer->getMessage(msg);
        }
    }

private:
    Businessman* m_pBuyer;
    Businessman* m_pSeller;
};

int main()
{
    HouseMediator* hMediator = new HouseMediator;
    Buyer* buyer = new Buyer(hMediator);
    Seller* seller = new Seller(hMediator);

    hMediator->setBuyer(buyer);
    hMediator->setSeller(seller);

    buyer->sendMessage("Sell not to sell?");
    seller->sendMessage("Of course selling!");

    delete buyer;
    buyer = nullptr;

    delete seller;
    seller = nullptr;

    delete hMediator;
    hMediator = nullptr;


    return 0;
}
```

## 14、备忘录模式(Memento)

备忘录模式：在不破坏封装性的前提下，捕获一个对象的内部状态，并在该对象之外保存这个状态。这样以后就可以将该对象恢复到原来保存的状态。

备忘录模式中需要定义的角色类：

1. Originator(发起人)：负责创建一个备忘录Memento，用以记录当前时刻自身的内部状态，并可使用备忘录恢复内部状态。Originator可以根据需要决定Memento存储自己的哪些内部状态。
2. Memento(备忘录)：负责存储Originator对象的内部状态，并可以防止Originator以外的其他对象访问备忘录。备忘录有两个接口：Caretaker只能看到备忘录的窄接口，他只能将备忘录传递给其他对象。Originator却可看到备忘录的宽接口，允许它访问返回到先前状态所需要的所有数据。
3. Caretaker(管理者):负责备忘录Memento，不能对Memento的内容进行访问或者操作。

```cpp
/*
* 关键代码：Memento类、Originator类、Caretaker类；Originator类不与Memento类耦合，而是与Caretaker类耦合。
*/

include <iostream>

using namespace std;

//需要保存的信息
typedef struct  
{
    int grade;
    string arm;
    string corps;
}GameValue;

//Memento类
class Memento   
{
public:
    Memento(){}
    Memento(GameValue value):m_gameValue(value){}
    GameValue getValue()
    {
        return m_gameValue;
    }
private:
    GameValue m_gameValue;
};

//Originator类
class Game   
{
public:
    Game(GameValue value):m_gameValue(value)
    {}
    void addGrade()  //等级增加
    {
        m_gameValue.grade++;
    }
    void replaceArm(string arm)  //更换武器
    {
        m_gameValue.arm = arm;
    }
    void replaceCorps(string corps)  //更换工会
    {
        m_gameValue.corps = corps;
    }
    Memento saveValue()    //保存当前信息
    {
        Memento memento(m_gameValue);
        return memento;
    }
    void load(Memento memento) //载入信息
    {
        m_gameValue = memento.getValue();
    }
    void showValue()
    {
        cout << "Grade: " << m_gameValue.grade << endl;
        cout << "Arm  : " << m_gameValue.arm.data() << endl;
        cout << "Corps: " << m_gameValue.corps.data() << endl;
    }
private:
    GameValue m_gameValue;
};

//Caretaker类
class Caretake 
{
public:
    void save(Memento memento)  //保存信息
    {
        m_memento = memento;
    }
    Memento load()            //读已保存的信息
    {
        return m_memento;
    }
private:
    Memento m_memento;
};

int main()
{
    GameValue v1 = {0, "Ak", "3K"};
    Game game(v1);    //初始值
    game.addGrade();
    game.showValue();
    cout << "----------" << endl;
    Caretake care;
    care.save(game.saveValue());  //保存当前值
    game.addGrade();          //修改当前值
    game.replaceArm("M16");
    game.replaceCorps("123");
    game.showValue();
    cout << "----------" << endl;
    game.load(care.load());   //恢复初始值
    game.showValue();
    return 0;
}
```

## 15、原型模式(Prototype)

原型模式：用原型实例指定创建对象的种类，并且通过拷贝这些原型创建新的对象。通俗的讲就是当需要创建一个新的实例化对象时，我们刚好有一个实例化对象，但是已经存在的实例化对象又不能直接使用。这种情况下拷贝一个现有的实例化对象来用，可能会更方便。

以下情形可以考虑使用原型模式：

- 当new一个对象，非常繁琐复杂时，可以使用原型模式来进行复制一个对象。比如创建对象时，构造函数的参数很多，而自己又不完全的知道每个参数的意义，就可以使用原型模式来创建一个新的对象，不必去理会创建的过程。
- 当需要new一个新的对象，这个对象和现有的对象区别不大，我们就可以直接复制一个已有的对象，然后稍加修改。
- 当需要一个对象副本时，比如需要提供对象的数据，同时又需要避免外部对数据对象进行修改，那就拷贝一个对象副本供外部使用。

```cpp
/*
* 关键代码：拷贝，return new className(*this);
*/
#include <iostream>

using namespace std;

//提供一个抽象克隆基类。
class Clone
{
public:
    virtual Clone* clone() = 0;
    virtual void show() = 0;
};

//具体的实现类
class Sheep:public Clone
{
public:
    Sheep(int id, string name):Clone(),
                               m_id(id),m_name(name)
    {
        cout << "Sheep() id address:" << &m_id << endl;
        cout << "Sheep() name address:" << &m_name << endl;
    }
    ~Sheep()
    {
    }
    //关键代码拷贝构造函数
    Sheep(const Sheep& obj)
    {
        this->m_id = obj.m_id;
        this->m_name = obj.m_name;
        cout << "Sheep(const Sheep& obj) id address:" << &m_id << endl;
        cout << "Sheep(const Sheep& obj) name address:" << &m_name << endl;
    }
    //关键代码克隆函数，返回return new Sheep(*this)
    Clone* clone()
    {
        return new Sheep(*this);
    }
    void show()
    {
        cout << "id  :" << m_id << endl;
        cout << "name:" << m_name.data() << endl;
    }
private:
    int m_id;
    string m_name;
};

int main()
{
    Clone* s1 = new Sheep(1, "abs");
    s1->show();
    Clone* s2 = s1->clone();
    s2->show();
    
    delete s1;
    s1 = nullptr;
    delete s2;
    s2 = nullptr;
    return 0;
}
```

## 16、享元模式(Flyweight)

享元模式：运用共享技术有效地支持大量细粒度的对象。在有大量对象时，把其中共同的部分抽象出来，如果有相同的业务请求，直接返回内存中已有的对象，避免重新创建。

以下情况可以考虑使用享元模式：

- 系统中有大量的对象，这些对象消耗大量的内存，且这些对象的状态可以被外部化。

对于享元模式，需要将对象的信息分为两个部分：内部状态和外部状态。内部状态是指被共享出来的信息，储存在享元对象内部且不随环境变化而改变；外部状态是不可以共享的，它随环境改变而改变，是由客户端控制的。

```cpp
/*
* 关键代码：将内部状态作为标识，进行共享。
*/
#include <iostream>
#include <map>
#include <memory>

using namespace std;

//抽象享元类，提供享元类外部接口。
class AbstractConsumer
{
public:
    virtual ~AbstractConsumer(){}
    virtual void setArticle(const string&) = 0;
    virtual const string& article() = 0;
};

//具体的享元类
class Consumer : public AbstractConsumer
{
public:
    Consumer(const string& strName) : m_user(strName){}
    ~Consumer()
    {
        cout << " ~Consumer()" << endl;
    }

    void setArticle(const string& info) override
    {
        m_article = info;
    }

    const string& article() override
    {
        return m_article;
    }

private:
    string m_user;
    string m_article;
};

//享元工厂类
class Trusteeship
{
public:
    ~Trusteeship()
    {
         m_consumerMap.clear();
    }

    void hosting(const string& user, const string& article)
    {
        if(m_consumerMap.count(user))
        {
            cout << "A customer named " << user.data() << " already exists" << endl;
            Consumer* consumer = m_consumerMap.at(user).get();
            consumer->setArticle(article);
        }
        else
        {
            shared_ptr<Consumer> consumer(new Consumer(user));
            consumer.get()->setArticle(article);
            m_consumerMap.insert(pair<string, shared_ptr<Consumer>>(user, consumer));
        }
    }

    void display()
    {
        map<string, shared_ptr<Consumer>>::iterator iter = m_consumerMap.begin();
        for(; iter != m_consumerMap.end(); iter++)
        {
            cout << iter->first.data() << " : "<< iter->second.get()->article().data() << endl;
        }
    }

private:
    map<string, shared_ptr<Consumer>> m_consumerMap;
};


int main()
{
    Trusteeship* ts = new Trusteeship;
    ts->hosting("zhangsan", "computer");
    ts->hosting("lisi", "phone");
    ts->hosting("wangwu", "watch");

    ts->display();

    ts->hosting("zhangsan", "TT");
    ts->hosting("lisi", "TT");
    ts->hosting("wangwu", "TT");

    ts->display();

    delete ts;
    ts = nullptr;

    return 0;
}
```

## 17、职责链模式(Chain of Resp.)

职责链模式：使多个对象都有机会处理请求，从而避免请求的发送者和接收者之前的耦合关系，将这些对象连成一条链，并沿着这条链传递请求，直到有一个对象处理它为止。

职责链上的处理者负责处理请求，客户只需要将请求发送到职责链上即可，无需关心请求的处理细节和请求的传递，所有职责链将请求的发送者和请求的处理者解耦了。

```cpp
/*
* 关键代码：Handler内指明其上级，handleRequest()里判断是否合适，不合适则传递给上级。
*/
#include <iostream>

using namespace std;

enum RequestLevel
{
    Level_One = 0,
    Level_Two,
    Level_Three,
    Level_Num
};

//抽象处理者（Handler）角色，提供职责链的统一接口。
class Leader
{
public:
    Leader(Leader* leader):m_leader(leader){}
    virtual ~Leader(){}
    virtual void handleRequest(RequestLevel level) = 0;
protected:
    Leader* m_leader;
};

//具体处理者（Concrete Handler）角色
class Monitor:public Leader   //链扣1
{
public:
    Monitor(Leader* leader):Leader(leader){}
    void handleRequest(RequestLevel level)
    {
        if(level < Level_Two)
        {
            cout << "Mointor handle request : " << level << endl;
        }
        else
        {
            m_leader->handleRequest(level);
        }
    }
};

//具体处理者（Concrete Handler）角色
class Captain:public Leader    //链扣2
{
public:
    Captain(Leader* leader):Leader(leader){}
    void handleRequest(RequestLevel level)
    {
        if(level < Level_Three)
        {
            cout << "Captain handle request : " << level << endl;
        }
        else
        {
            m_leader->handleRequest(level);
        }
    }
};

//具体处理者（Concrete Handler）角色
class General:public Leader   //链扣3
{
public:
    General(Leader* leader):Leader(leader){}
    void handleRequest(RequestLevel level)
    {
        cout << "General handle request : " << level << endl;
    }
};

int main()
{
    Leader* general = new General(nullptr);
    Leader* captain = new Captain(general);
    Leader* monitor = new Monitor(captain);
    monitor->handleRequest(Level_One);

    delete monitor;
    monitor = nullptr;
    delete captain;
    captain = nullptr;
    delete general;
    general = nullptr;
    return 0;
}
```

标签: [手册](https://www.cnblogs.com/schips/tag/手册/), [c/c++](https://www.cnblogs.com/schips/tag/c%2Fc%2B%2B/)