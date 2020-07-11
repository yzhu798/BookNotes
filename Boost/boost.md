
# [boost库笔记](https://www.cnblogs.com/chendeqiang/p/12941541.html)

# 简介 

Boost库是一个功能强大、构造精巧、跨平台、开源并且完全免费的C++程序库。

Boost涵盖字符串与文本处理、容器、迭代器、算法、图像处理、模板元编程、并发编程等许多领域

> 使用Boost，将大大增强C++的功能和表现力。

Boost库大部分组件不需要编译，直接包含头文件即可，主要是因为编译器不支持模板分离编译。

```cpp
#include <boost/logic/tribool.hpp>          //使用tribool库
using namespace boost; 
```

包括`date_time`、`regex`、`program_options`、test、thread、python等必须编译成库用。非库编译的替代品（xpressive可替代regex、signals2可替代signals）。

# 时间与日期 

## timer 计时器(outed) 

timer是一个很小的库，提供简易的度量时间和进度显示功能，可以用于性能测试等需要计时的任务，对于大多数的情况它足够用。

timer对象一旦被声明，它的**构造函数就启动了计时工作**，之后就可以随时用`elapsed()`函数简单地测量自对象创建后所流逝的时间。（废弃）

```cpp
#include <boost/timer.hpp> //timer的头文件
#include<iostream>
using namespace std;
using namespace boost;     //打开boost名字空间

int main()
{
    timer t;

    //声明一个计时器对象，开始计时
    cout << "max timespan:" //可度量的最大时间，以小时为单位
         << t.elapsed_max()/ 3600
         << "h" << endl;

    cout << "min timespan:" //可度量的最小时间，以秒为单位
         << t.elapsed_min()
         << "s" << endl;

    cout << "now time elapsed:" //输出已经流逝的时间
         << t.elapsed()
         << "s" << endl;
}
```

```cpp
max timespan:5.1241e+09h
min timespan:1e-06s
now time elapsed:5.2e-05s
```

timer的计时使用了标准库头文件`<ctime>`里的`std::clock()`函数，它返回自进程启动以来的clock数，每秒的clock数则由宏`CLOCKS_PER_SEC`定义。`CLOCKS_PER_SEC`的值因操作系统而不同，在Mac OS X、Linux下是1 000 000，而在Win32下则是1 000，Linux下的精度是微秒，而Win32下的精度是毫秒。

## progress_timer 

progress_timer是个计时器，它继承自timer，**会在析构时自动输出时间**，省去了timer手动调用elapsed()的工作，是一个用于自动计时相当方便的小工具。

构造函数`progress_timer`( std::ostream& os )，它允许将析构时的输出定向到指定的IO流里，默认是`std::cout`。可用其他标准库输出流（`ofstream`、`ostringstream`）替换，或用`cout.rdbuf()`重定向cout的输出。

```cpp
#include <sstream>
#include <boost/timer.hpp> //timer的头文件
#include <boost/progress.hpp>

using namespace boost;     //打开boost名字空间

int main()
{
  {
    boost::progress_timer t;        //第一个计时
    //do something ...
  }                                   //progress_timer在这里析构，自动输出时间
  {
    boost::progress_timer t;        //第二个计时
    //do something ...
  }                                   //progress_timer在这里析构，自动输出时间
  
  stringstream ss;                       //一个字符串流对象
  {
    progress_timer t(ss);                  //要求progress_timer输出到ss中
  }                                      //progress_timer在这里析构，自动输出时间
  cout << ss.str();
}
0.00 s
0.00 s
0.00 s
```

## progress_display 

progress_display可以在控制台上显示程序的执行进度。

```cpp
#include <iostream>
#include <vector>
#include <fstream>
using namespace std;

#include <boost/progress.hpp>
using namespace boost;

int main()
{
  vector<string> v(100);                              //一个字符串向量
  ofstream fs("./test.txt");                          //文件输出流

  //声明一个progress_display对象，基数是v的大小
  progress_display pd(v.size());

  //开始迭代遍历向量，处理字符串，写入文件
  for (auto& x : v)                                   //for+auto循环
  {
    fs << x << endl;
    ++pd;                                           //更新进度显示
  }
}
0%   10   20   30   40   50   60   70   80   90   100%
|----|----|----|----|----|----|----|----|----|----|
***************************************************
```

注意：progress_display可以用作基本的进度显示，但它有个固有的缺陷：**无法把进度显示输出与程序的输出分离**。

## date_time库概述 

`date_time`是一个非常全面且灵活的日期时间库，基于我们日常使用的公历（即格里高利历），可以提供时间相关的各种所需功能，如精确定义的时间点、时间段和时间长度、加减若干天/月/年、日期迭代器等等。`date_time`库还支持无限时间和无效时间这种实际生活中有用的概念，而且可以与C的传统时间结构tm相互转换，提供向下支持。

### 编译date_time库 

date_time库是Boost中少数需要编译的库之一。

直接编译date_time需运行的b2命令如下：

```bash
b2 --with-date_time  [--buildtype=complete] [stdlib=stlport] [stage]
```

编译后会生成形如libboost_date_time-vc80-mt-sgdp-1_51.lib的静态库或者动态库。

如不想使用b2预编译库，可以直接在工程内包含date_time库的实现cpp源文件，使用嵌入工程编译的方式。cpp源代码如下：

```cpp
//dateprebuild.cpp
#define BOOST_DATE_TIME_SOURCE
#include <libs/date_time/src/gregorian/greg_names.hpp>
#include <libs/date_time/src/gregorian/date_generators.cpp>
#include <libs/date_time/src/gregorian/greg_month.cpp>
#include <libs/date_time/src/gregorian/greg_weekday.cpp>
#include <libs/date_time/src/gregorian/gregorian_types.cpp>
```

在工程的其他源代码使用`date_time`库时，需要在包含头文件之前定义宏`BOOST_DATE_TIME_SOURCE`、`BOOST_DATE_TIME_NO_LIB`或者`BOOST_ALL_NO_LIB`。

```cpp
#define BOOST_DATE_TIME_SOURCE
#include <boost/date_time/gregorian/gregorian.hpp>    
```

## 处理日期 

`date_time`库的日期基于格里高利历，支持从1400-01-01到9999-12-31之间的日期计算。它位于名字空间boost:: gregorian，为了使用date_time库的日期功能，需要包含头文件<boost/date_time/gregorian/gregorian.hpp>，即：

```cpp
#define BOOST_DATE_TIME_SOURCE
#include <boost/date_time/gregorian/gregorian.hpp>
using namespace boost::gregorian;
```

### 创建日期对象 

```cpp
#include<iostream>
using namespace std;

#define BOOST_DATE_TIME_SOURCE
#include <boost/date_time/gregorian/gregorian.hpp> 
using namespace boost::gregorian;

int main()
{
  date d1;                                  //一个无效的日期
  date d2(2010, 1, 1);                        //使用数字构造日期
  date d3(2000, Jan, 1);                   //也可以使用英文指定月份
  date d4(d2);                              //date支持拷贝构造
  date d5 = from_string("1999-12-31");
  date d6(from_string("2005/1/1"));
  date d7 = from_undelimited_string("20011118");
  date d8(neg_infin);                  //负无限日期
  date d9(pos_infin);                  //正无限日期
  date d10(not_a_date_time);            //无效日期
  date d11(max_date_time);              //最大可能日期9999-12-31
  date d12(min_date_time);              //最小可能日期1400-01-01

  cout << d2 << endl;
  cout << d3 << endl;
}
2010-Jan-01
2000-Jan-01
```

### 访问日期 

date类的对外接口很像C语言中的tm结构，也可以获取它保存的年、月、日、星期等成分，但date提供了更多的操作。

```cpp
date d(2010,4,1);
assert(d.year()  == 2010);
assert(d.month() == 4);
assert(d.day()   == 1);

date::ymd_type ymd =  d.year_month_day();
assert(ymd.year    == 2010);
assert(ymd.month   == 4);
assert(ymd.day     == 1);
assert(d.day_of_week()  == 4);
assert(d.day_of_year()  == 91);
assert(d.end_of_month() == date(2010,4,30));
assert(date(2010,1,10).week_number() ==1 );
assert(date(2010,1,1).week_number()  ==53 );
assert(date(2008,1,1).week_number()  == 1);
assert(date(pos_infin).is_infinity()  );
assert(date(pos_infin).is_pos_infinity() );
assert(date(neg_infin).is_neg_infinity() );
assert(date(not_a_date_time).is_not_a_date() );
assert(date(not_a_date_time).is_special() );
assert(! date(2010,10,1).is_special() );
```

### 日期的输出 

date对象可以很方便地转换成字符串，它提供了三个自由函数。

- to_simple_string(date d)：转换为YYYY-mmm-DD格式的字符串，其中的mmm为3字符的英文月份名；
- to_iso_string(date d)：转换为YYYYMMDD格式的数字字符串；
- to_iso_extended_string(date d)：转换为YYYY-MM-DD格式的数字字符串。

```cpp
#include<iostream>
using namespace std;

#define BOOST_DATE_TIME_SOURCE
#include <boost/date_time/gregorian/gregorian.hpp> 
using namespace boost::gregorian;

int main()
{
  date d(2008, 11, 20);

  cout << to_simple_string(d) << endl;
  cout << to_iso_string(d) << endl;
  cout << to_iso_extended_string(d) << endl;
  cout << d << endl;

  cin >> d;
  cout << d;
}
2008-Nov-20
20081120
2008-11-20
2008-Nov-20
2010-Jan-02（用户的输入）
2010-Jan-02
```

### 与tm结构的转换 

date支持与C标准库中的tm结构相互转换，转换的规则和函数如下：

- `to_tm(date)`:date转换到tm。tm的时分秒成员（tm_hour, tm_min, tm_sec）均置为0，夏令时标志tm_isdst置为-1（表示未知）。
- `date_from_tm(tm datetm)`:tm转换到date。只使用年、月、日三个成员（tm_year, tm_mon, tm_mday），其他成员均被忽略。

```cpp
date d(2010, 2, 1);
tm t = to_tm(d);
assert(t.tm_hour == 0 && t.tm_min == 0);
assert(t.tm_year == 110 && t.tm_mday == 1);
date d2 = date_from_tm(t);
assert(d == d2);
```

### 日期长度 

日期长度是以天为单位的时长，是度量时间长度的一个标量。它与日期不同，值可以是任意的整数，可正可负。

```cpp
days dd1(10), dd2(-100), dd3(255);
assert( dd1 > dd2 && dd1 < dd3);
assert( dd1 + dd2 == days(-90));
assert((dd1 + dd3).days() == 265);
assert( dd3 / 5 == days(51));

weeks w(3);                                       //3个星期
assert(w.days() == 21);

months m(5);                                      //5个月
years y(2);                                       //2年
months m2 = y + m;                                //2年零5个月
assert(m2.number_of_months() == 29);
assert((y ＊ 2).number_of_years() == 4);
```

### 日期运算 

date支持加减运算，两个date对象的加操作是无意义的（date_time库会以编译错误的方式通知我们）, date主要是与时长概念配合运算。
例如，下面的代码计算了从2000年1月1日到2008年8月8日的天数，并执行其他的日期运算：

```cpp
#include<iostream>
using namespace std;

#define BOOST_DATE_TIME_SOURCE
#include <boost/date_time/gregorian/gregorian.hpp>
using namespace boost::gregorian;

int main()
{
  date d1(2000, 1, 1), d2(2008, 8, 8);
  cout << d2 - d1 << endl;                            //3142天
  assert(d1 + (d2 - d1) == d2);

  d1 += days(10);                                     //2000-1-11
  assert(d1.day() == 11);
  d1 += months(2);                                    //2000-3-11
  assert(d1.month() == 3 && d1.day() == 11);
  d1 -= weeks(1);                                     //200-3-4
  assert(d1.day() == 4);

  d2 -= years(7);                                     //2001-8-8
  assert(d2.year() == d1.year() + 1);
}
```

### 日期区间 

date_time库使用`date_period`类来表示日期区间的概念，它是时间轴上的一个`左闭右开区间`，端点是两个date对象。区间的左值必须小于右值，否则date_period将表示一个无效的日期区间。

批注：一段时间的表示方法。

date_period可以指定区间的两个端点构造，也可以指定**左端点再加上时长构造**，通常后一种方法比较常用，相当于生活中从某天开始的一个周期。

```cpp
date_period dp(date(2010,1,1), days(20));

assert(! dp.is_null());
assert(dp.begin().day() == 1);
assert(dp.last().day() == 20);
assert(dp.end().day() == 21);
assert(dp.length().days() == 20);

date_period dp1(date(2010,1,1), days(20));
date_period dp2(date(2010,2,19), days(10));
cout << dp1;                      //[2010-Jan-01/2010-Jan-20]
assert(dp1 < dp2);
```

### 日期区间运算 

date_period同date、days一样，也支持很多运算。

成员函数`shift()`和`expand()`可以变动区间：

> `shift`()将日期区间平移n天而长度不变，`expand()`将日期区间向两端延伸n天，相当于区间长度加2n天。

```cpp
//构造函数
date_period dp(date(2010,1,1), days(20));

dp.shift(days(3)); 
assert(dp.begin().day() == 4);
assert(dp.length().days() == 20);

dp.expand(days(3));
assert(dp.begin().day() == 1);
assert(dp.length().days() == 26);

//判断某个日期是否在区间内
date_period dp(date(2010,1,1), days(20));       //1-1至1-20
assert(dp.is_after(date(2009,12,1)));
assert(dp.is_before(date(2010,2,1)));
assert(dp.contains(date(2010,1,10)));

date_period dp2(date(2010,1,5), days(10));      //1-5至1-15
assert(dp.contains(dp2));
assert(dp.intersects(dp2));
assert(dp.intersection(dp2) == dp2);

date_period dp3(date(2010,1,21), days(5));      //1-21至1-26
assert(! dp3.intersects(dp2));
assert(dp3.intersection(dp2).is_null());

//并集操作
date_period dp1(date(2010,1,1), days(20));
date_period dp2(date(2010,1,5), days(10));
date_period dp3(date(2010,2,1), days(5));
date_period dp4(date(2010,1,15), days(10));

assert( dp1.contains(dp2) && dp1.merge(dp2) == dp1);
assert(! dp1.intersects(dp3) && dp1.merge(dp3).is_null());
assert( dp1.intersects(dp2) && dp1.merge(dp4).end() == dp4.end());
assert( dp1.span(dp3).end() == dp3.end());
```

### 日期迭代器 

date_time库为日期处理提供了迭代器的概念，可以用简单的递增或者递减操作符连续访问日期，这些迭代器包括day_iterator、week_iterator、month_iterator和year_iterator，它们分别以天、周、月和年为单位增减。

```cpp
date d(2006,11,26);
day_iterator d_iter(d);                       //增减步长默认为1天

assert(d_iter == d);
++d_iter;                                     //递增1天
assert(d_iter == date(2006,11,27));

year_iterator y_iter(＊d_iter, 3);            //增减步长为3年
assert(y_iter == d + days(1));
++y_iter;                                     //递增3年
assert(y_iter->year() == 2009);
```

### 闰年 

类boost::gregorian::gregorian_calendar提供了格里高利历的一些操作函数，基本上它被date类在内部使用，用户通常很少用到。但它也提供了几个有用的静态函数：成员函数`is_leap_year()`可以判断年份是否是闰年，`end_of_month_day()`给定年份和月份，返回该月的最后一天。例如：

```cpp
typedef gregorian_calendar gre_cal;         //typedef以简化代码书写
cout << "Y2010 is "
  << (gre_cal::is_leap_year(2010)? "":"not")
  << " a leap year." << endl;
assert(gre_cal::end_of_month_day(2010, 2) == 28);
```

## 处理时间 

date_time库在格里高利历的基础上提供微秒级别的时间系统，但如果需要，它最高可以达到纳秒级别的精确度。

### 时间长度 

与日期长度`date_duration`类似，date_time库使用`time_duration`度量时间长度。

time_duration支持全序比较操作和输入输出，而且比date_duration要支持更多的算术运算，可以进行加减乘除全四则运算。

### 操作时间长度 

time_duration可以在构造函数指定时分秒和微秒来构造，例如创建一个1小时10分钟30秒1毫秒（1000微秒）的时间长度：
批注：创建一个时间段。

```cpp
//构造函数
time_duration td(1,10,30,1000);

time_duration td(1,60,60,1000＊1000＊ 6 + 1000);

hours h(1);                                       //1小时
minutes m(10);                                    //10分钟
seconds s(30);                                    //30秒钟
millisec ms(1);                                   //1毫秒
time_duration td = h + m + s + ms;                //可以赋值给time_duration
time_duration td2 = hours(2) + seconds(10);       //也可以直接赋值

time_duration td = duration_from_string("1:10:30:001");

//方法
time_duration td(1,10,30,1000);
assert(td.hours() == 1 && td.minutes() == 10 && td.seconds() == 30);
assert(td.total_seconds() == 1＊3600+ 10＊60 + 30);
assert(td.total_milliseconds() == td.total_seconds()＊1000 + 1);
assert(td.fractional_seconds() == 1000);

//可以取负值
hours h(-10);
assert(h.is_negative())
time_duration h2 = h.invert_sign();
assert(! h2.is_negative() && h2.hours() == 10);

//特殊值
time_duration td1(not_a_date_time);
assert(td1.is_special() && td1.is_not_a_date_time());
time_duration td2(neg_infin);
assert(td2.is_negative() && td2.is_neg_infinity());

//支持完整的四则运算
time_duration td1 = hours(1);
time_duration td2 = hours(2) + minutes(30);
assert(td1 < td2);
assert((td1+td2).hours() == 3);
assert((td1-td2).is_negative());
assert(td1 ＊ 5 == td2 ＊ 2);
assert((td1/2).minutes() == td2.minutes());

//字符串表示
time_duration td(1,10,30,1000);
cout << to_simple_string(td) << endl;  //01:10:30.001000
cout << to_iso_string(td) << endl;  //011030.001000
```

### 时间长度的精确度 

`date_time`库默认时间的精确度是**微秒**，纳秒相关的类和函数如nanosec和成员函数nanoseconds()、total_nanoseconds()都不可用，秒以下使用微秒。

当定义了宏`BOOST_DATE_TIME_POSIX_TIME_STD_CONFIG`时，time_duration的一些行为将发生变化，它的时间分辨率将精确到纳秒，构造函数中秒以下纳秒。

### 时间点 

在熟悉了时间长度类time_duration后，理解时间点概念就容易多了，它相当于**一个日期**再加上一个**小于一天的时间长度**。如果时间轴的基本单位是天，那么日期就相当于整数，时间点则是实数，定义了天之间的小数部分。

### 创建时间点对象 

最基本的创建`ptime`的方式是在构造函数中同时指定date和time_duration对象，令ptime等于一个日期加当天的时间偏移量。如果不指定time_duration，则默认为当天的零点。

```cpp
//构造函数
using namespace boost::gregorian; // posix_time名字空间不包含gregorian名字空间，因此需要加上对它的引用
ptime p(date(2010,3,5), hours(1));          //2010年3月5日凌晨1时

ptime p1 = time_from_string("2010-3-5 01:00:00");
ptime p2 = from_iso_string("20100305T010000");

ptime p1 = second_clock::local_time();              //秒精度，本地时间
ptime p2 = microsec_clock::universal_time();        //微秒精度，UTC时间
cout << p1 << endl << p2;

//特殊值
ptime  p1(not_a_date_time);                           //无效时间
assert(p1.is_not_a_date_time());
ptime  p2(pos_infin);                                 //正无限时间
assert(p2.is_special() && p2.is_pos_infinity());
```

### 操作时间点对象 

由于ptime相当于date+time_duration，因此对它的操作可以分解为对这两个组成部分的操作。

```cpp
//成员函数
ptime p(date(2010,3,20), hours(12)+minutes(30)); //2010年3月20日中午12:30

date d = p.date();
time_duration td = p.time_of_day();
assert(d.month() == 3 && d.day() == 20);
assert(td.total_seconds() == 12＊3600 + 30＊60);

//四则运算
ptime p1(date(2010,3,20), hours(12)+minutes(30));
ptime p2 = p1 + hours(3);                            //2010年3月20日15:30

assert(p1 < p2);
assert(p2 - p1 == hours(3));
p2 += months(1);
assert(p2.date().month() == 4);

//转换为字符串
ptime p(date(2010,2,14), hours(20));
cout << to_simple_string(p) << endl; //2010-Feb-14 20:00:00
cout << to_iso_string(p) << endl; //20100214T200000
cout << to_iso_extended_string(p) << endl; //2010-02-14T20:00:00
```

### 与tm、time_t等结构的转换 

使用自由函数to_tm(), ptime可以单向转换到tm结构，转换规则是date和time_duration的组合。

```cpp
ptime p(date(2010,2,14), hours(20));
tm t = to_tm(p);
assert(t.tm_year == 110 && t.tm_hour == 20);
```

没有一个叫做time_from_tm()的函数可以把tm结构转换成ptime，这与date对象的date_from_tm()是不同的!

### 时间区间 

与日期区间date_period对应，date_time库也有时间区间的概念，使用类time_period，使用ptime作为区间的两个端点，同样是左闭右开区间。

```cpp
ptime p(date(2010,1,1), hours(12)) ;              //2010年元旦中午
time_period tp1(p, hours(8));                     //一个8小时的区间
time_period tp2(p + hours(8), hours(1));          //1小时的区间
assert(tp1.end() == tp2.begin() && tp1.is_adjacent(tp2));
assert(! tp1.intersects(tp2));                    //两个区间相邻但不相交

tp1.shift(hours(1));                              //tp1平移1小时
assert(tp1.is_after(p));                          //tp1在中午之后
assert(tp1.intersects(tp2));                      //两个区间现在相交

tp2.expand(hours(10));                            //tp2向两端扩展10个小时
assert(tp2.contains(p) && tp2.contains(tp1));
```

### 时间迭代器 

不同于日期迭代器，时间迭代器只有一个`time_iterator`。它在构造时传入一个起始时间点`ptime`对象和一个步长`time_duration`对象，然后就同日期迭代器一样使用**前置式operator++**、**operator--**来递增或递减时间，解引用操作符返回一个ptime对象。

time_iterator也可以直接与ptime比较，无须再使用解引用操作符。

```cpp
ptime p(date(2010,2,27), hours(10)) ;
for (time_iterator t_iter(p, minutes(10));
      t_iter < p + hours(1); ++ t_iter)
{
  cout << ＊t_iter << endl;
}
```

## date_time库的高级议题 

### 编译配置宏 

宏`BOOST_DATE_TIME_SOURCE`和`BOOST_DATE_TIME_NO_LIB`用来指定date_time库的编译设置，定义了它们将告诉编译器不使用自动链接库的功能，而是使用嵌入源码的方式。Boost中许多需要编译的库也有类似名称的宏定义，请读者务必了解这个宏的用法。

宏DATE_TIME_NO_DEFAULT_CONSTRUCTOR可以禁止编译器创造出date和ptime的缺省（默认）构造函数，强制它们在构造时必须有一个有效的值，可以避免某些疏忽而导致的错误。

宏BOOST_DATE_TIME_OPTIONAL_GREGORIAN_TYPES启用了weeks、months、years等日期区间便捷类型，它们在处理日期时很有用，可以使代码更清晰易懂。但它们有时候也会在日期运算时产生非预期结果，如果不想使用它们，就undef这个宏，从而在程序中总使用days保证代码的正确性。

宏BOOST_DATE_TIME_POSIX_TIME_STD_CONFIG将启用date_time库更高的时间精确度，由微秒变为纳秒，同时纳秒相关的一些函数和类也会启用。缺省情况下它是关闭的，因为纳秒精度通常很依赖于操作系统，而且实际生活中很少用到这么高的精确度。

date_time库编译的对象是格里高利历源码，不使用纳秒来处理日期，因此BOOST_DATE_ TIME_POSIX_TIME_STD_CONFIG宏对库的编译没有任何影响，预编译源码文件dateprebuild.cpp不必为了支持纳秒精度而增加宏定义。

### 格式化时间 

date_time库默认的日期时间格式简单、标准且是英文，但并不是不可以被改变的。date_time库提供了专门的格式化对象`date_facet`、time_facet等来搭配IO流，定制日期时间的表现形式。

这些格式化对象就像是printf()函数，使用一个格式化字符串来定制日期或时间的格式，也同样有大量的格式标志符。由于格式标志符非常多，本书不在这里列出，请参考date_time库的说明文档。

示范格式化的使用、把日期格式化为中文显示的代码如下：

```cpp
date d(2010,3,6);
date_facet＊ dfacet = new date_facet("%Y年%m月%d日");
cout.imbue(locale(cout.getloc(), dfacet));
cout << d << endl;

time_facet ＊tfacet = new time_facet("%Y年%m月%d日%H点%M分%S%F秒");
cout.imbue(locale(cout.getloc(), tfacet));
cout << ptime(d , hours(21) + minutes(50) + millisec(100)) << endl;
2010年03月06日
2010年03月06日　21点50分00.100000秒
```

### 本地时间 

date_time库使用time_zone_base、posix_time_zone、custom_time_zone、local_date_time、c_local_adjustor等类和一个文本格式的时区数据库来解决本地时间中时区和夏令时的问题。

本地时间功能位于名字空间boost::local_time，为了使用本地时间功能，需要包含头文件`<boost/date_time/local_time/local_time.hpp>`，即：

```cpp
#include <boost/date_time/local_time/local_time.hpp>
using namespace boost::local_time;
```

time_zone_base是时区表示的抽象类，通常我们使用一个typedef:time_zone_ptr，它是一个指向time_zone_base的智能指针（参见3.4节的shared_ptr）。

local_date_time是一个含有时区信息的时间对象，它可以由date+time_duration+时区构造，构造时必须指定这个时间是否是夏令时，本地时间在内部以UTC的形式保存，以方便计算。

为了便于时区编程，date_time库附带了一个小型CSV格式的文本数据库date_time_zonespec.csv，位于libs/date_time/data/下，可以自由使用。这个数据库包含了世界上几乎所有国家和地区的时区信息，tz_database类专门管理这个数据库，只要指定时区名，就可以很方便地获得时区信息——一个time_zone_ptr。

假设从北京时区飞往纽约时区，飞行时间为15个小时，示范跨时区的时间转换的代码如下：

```cpp
#define BOOST_DATE_TIME_SOURCE

//包含必要的头文件和声明名字空间
#include <boost/date_time/gregorian/gregorian.hpp>
#include <boost/date_time/posix_time/posix_time.hpp>
#include <boost/date_time/local_time/local_time.hpp>
using namespace boost::posix_time;
using namespace boost::gregorian;
using namespace boost::local_time;

int main()
{
  tz_database tz_db;                              //时区数据库对象
  {
    ptimer t;                                     //ptimer，计算打开数据库的时间

    //假设文本数据库位于当前目录下
    tz_db.load_from_file("./date_time_zonespec.csv");
  }
  cout << endl;

  //使用字符串Asia/Shanghai获得上海时区，即北京时间
  time_zone_ptr shz =  tz_db.time_zone_from_region("Asia/Shanghai");

  //使用字符串America/New_York获得纽约时区
  time_zone_ptr nyz =  tz_db.time_zone_from_region("America/New_York");

  cout << shz->has_dst() << endl;                 //上海时区无夏令时
  cout << shz->std_zone_name() << endl;           //上海时区的名称是CST

  local_date_time dt_bj(date(2008,1,7),           //北京时间2008,1,7
    hours(12),                                    //中午12点shz,                                          //上海时区
    false);                                       //没有夏令时
  cout << dt_bj << endl;
  time_duration flight_time = hours(15);          //飞行15小时
  dt_bj += flight_time;                           //到达的北京时间
  cout << dt_bj << endl;
  local_date_time dt_ny = dt_bj.local_time_in(nyz); //纽约当地时间
  cout << dt_ny;
}
00:00:00.953125
0
2008-Jan-07 12:00:00 CST
2008-Jan-08 03:00:00 CST
2008-Jan-07 14:00:00 EST
```

# 内存管理 

## 智能指针 

智能指针（smart pointer）是C++群体中热门的议题，围绕它，有很多有价值的讨论和结论。它实践了推荐书目[1]中的**代理模式**，`代理`了原始“裸”指针的行为，为它添加了更多更有用的特性。

boost.smart_ptr库提供了六种智能指针，除了`shared_ptr`和`weak_ptr`以外还包括`scoped_ ptr`、`scoped_array`、`shared_array`和`intrusive_ptr`。它们都是很轻量级的对象，速度与原始指针相差无几，都是异常安全的（exception safe），而且对于所指的类型T也仅有一个很小且很合理的要求：**类型T的析构函数不能抛出异常**。

这些智能指针都位于名字空间boost，为了使用`smart_ptr`组件，需要包含头文件`<boost/ smart_ptr.hpp>`，即：

```cpp
#include <boost/smart_ptr.hpp>
using namespace boost;
```

## scoped_ptr 

scoped_ptr是一个很类似auto_ptr的智能指针，它包装了new操作符在堆上分配的动态对象，能够保证动态创建的对象在任何时候都可以被正确地删除。但scoped_ptr的所有权更加严格，**不能转让**，一旦scoped_ptr获取了对象的管理权，你就无法再从它那里取回来。

scoped_ptr拥有一个很好的名字，它向代码的阅读者传递了明确的信息：这个智能指针**只能在本作用域里使用，不希望被转让。**

### 操作函数 

scoped_ptr的构造函数接受一个类型为T＊的指针p，创建出一个scoped_ptr对象，并在内部保存指针参数p。p必须是一个new表达式动态分配的结果，或者是个空指针（nullptr）。当scoped_ptr对象的生命期结束时，析构函数～scoped_ptr()会使用delete操作符自动销毁所保存的指针对象，从而正确地回收资源 [4] 。

**`scoped_ptr`同时把拷贝构造函数和赋值操作符都声明为私有的，禁止对智能指针的复制操作**（原理可参考4.1节noncopyable），保证了被它管理的指针不能被转让所有权。

**成员函数`reset()`的功能**是重置`scoped_ptr`：**它删除原来保存的指针，再保存新的指针值p**。如果p是空指针，那么scoped_ptr将不持有任何指针。一般情况下reset()不应该被调用，因为它违背了scoped_ptr的本意——资源应该一直由scoped_ptr自己自动管理。

scoped_ptr用`operator＊()`和`operator->()`重载了解引用操作符＊和箭头操作符->，以模仿被代理的原始指针的行为，因此可以把scoped_ptr对象如同指针一样使用。如果scoped_ptr保存的是空指针，那么这两个操作的行为未定义。

scoped_ptr不支持比较操作，不能在两个scoped_ptr之间，或者在scoped_ptr和原始指针或空指针之间进行相等或者不相等测试，我们也无法为它编写额外的比较函数，**因为它已经将operator==和operator! =两个操作符重载都声明为私有的**。但scoped_ptr提供了一个可以在bool语境（context）中自动转换成bool值（如if的条件表达式）的功能，用来测试`scoped_ptr`是否持有一个有效的指针（非空）。它可以代替与空指针的比较操作，而且写法更简单。

**成员函数`swap()`可以交换两个scoped_ptr保存的原始指针。它是高效的操作**，被用于实现reset()函数，也可以被boost::swap所利用。

**最后是成员函数`get()`，**它返回scoped_ptr内部保存的原始指针，可以用在某些要求必须是原始指针的场景（如底层的C接口）。但使用时必须小心，这将使原始指针脱离scoped_ptr的控制！**不能对这个指针做delete操作**，否则scoped_ptr析构时会对已经删除的指针再进行删除操作，发生未定义行为。

### 用法 

scoped_ptr的用法很简单：在原本使用指针变量接受new表达式结果的地方改成用scoped_ptr对象，然后去掉哪些多余的try/catch和delete操作就可以了。像这样：`scoped_ptr<string> sp(new string("text"));`

scoped_ptr是一种“智能指针”，因此其行为与普通指针基本相同，可以使用非常熟悉的＊和->操作符：

```cpp
cout << ＊sp << endl;                         //取字符串的内容
cout <<  sp->size() << endl;                  //取字符串的长度
```

但记住：不再需要delete操作，scoped_ptr会自动地帮助我们释放资源。如果我们对scoped_ptr执行delete会得到一个编译错误：因为scoped_ptr是一个行为类似指针的对象，而不是指针，**对一个对象应用delete是不允许的**。

使用scoped_ptr会带来两个好处：一是使代码变得清晰简单，而简单意味着更少的错误；二是它并没有增加多余的操作，安全的同时保证了效率，可以获得与原始指针同样的速度。

```cpp
#include<iostream>
using namespace std;

#include <boost/smart_ptr.hpp>
using namespace boost;

struct posix_file                          //一个示范性质的文件类
{
  posix_file(const char * file_name)      //构造函数打开文件
  {
    cout << "open file:" << file_name << endl;
  }
  ~posix_file()                           //析构函数关闭文件
  {
    cout << "close file" << endl;
  }
};

int main()
{
  scoped_ptr<int> p(new int);              //一个int指针的scoped_ptr

  if (p)                                   //在bool语境中测试指针是否有效
  {
    *p = 100;                           //可以像普通指针一样使用解引用操作符＊
    cout << *p << endl;
  }

  p.reset();                               //reset()置空scoped_ptr，仅仅是演示

  assert(p == 0);                          //p不持有任何指针
  if (!p)                                 //在bool语境中测试，可以用！操作符
  {
    cout << "scoped_ptr == null" << endl;
  }

  //文件类的scoped_ptr,
  //将在离开作用域时自动析构，从而关闭文件释放资源
  scoped_ptr<posix_file> fp(new posix_file("/tmp/a.txt"));
}  //在这里发生scoped_ptr的析构，
//p和fp管理的指针自动被删除
100
scoped_ptr == null
open file:/tmp/a.txt
close file
```

### 与unique_ptr的区别 

`std::unique_ptr`是在C++11标准中定义的新的智能指针，用来取代C++98中的`std::auto_ptr`。根据C++11标准，`unique_ptr`不仅能够代理new创建的单个对象，也能够代理`new[]`创建的数组对象，也就是说它结合了`scoped_ptr`和`scoped_array`两者的能力。

但unique_ptr要比scoped_ptr有更多的功能：可以像原始指针一样进行比较，可以像shared_ptr一样定制删除器，也可以安全地放入标准容器。因此，如果读者使用的**编译器支持C++11标准，那么可以毫不犹豫地使用`unique_ptr`来代替`scoped_ptr`。**

当然，scoped_ptr也有它的优点，“少就是多”永远是一句至理名言，它只专注于做好作用域内的指针管理工作，含义明确，而且不允许转让指针所有权。

### scoped_array 

`scoped_array`没有给程序增加额外的负担，用起来很方便轻巧。它的速度与原始数组同样快，很适合那些习惯于用new操作符在堆上分配内存的程序员。但`scoped_array`的功能很有限，不能动态增长，也没有迭代器支持，不能搭配STL算法，仅有一个纯粹的“裸”数组接口。而且，我们应当尽量避免使用new[]操作符，它比new更可怕，是许多错误的来源。

**除非对性能有非常苛刻的要求，或者编译器不支持标准库（比如某些嵌入式操作系统）**，否则本书不推荐使用`scoped_array`，它只是为了与老式C风格代码兼容而使用的类，它的出现往往意味着你的代码中存在着隐患。

## shared_ptr 

`shared_ptr`是一个最像指针的“智能指针”，是`boost.smart_ptr`库中最有价值、最重要的组成部分，也是最有用的，Boost库的许多组件——甚至还包括其他一些领域的智能指针都使用了shared_ptr，所以它被毫无悬念地收入了C++11标准。

`shared_ptr`与`scoped_ptr`一样包装了new操作符在堆上分配的动态对象，但它实现的**是引用计数型的智能指针，可以被自由地拷贝和赋值**，在任意的地方共享它，当**引用计数为0时才删除**被包装的动态分配的对象。`shared_ptr`也可以安全地放到标准容器中，是在STL容器中存储指针的最标准解法。

### 操作函数 

`shared_ptr`与`scoped_ptr`同样是用于管理new动态分配对象的智能指针，因此功能上有很多相似之处：它们都重载了＊和->操作符以模仿原始指针的行为，提供隐式bool类型转换以判断指针的有效性，`get()`可以得到原始指针，并且没有提供指针算术操作。

```cpp
shared_ptr<int> spi(new int);                      //一个int的shared_ptr
assert(spi);                                       //在bool语境中隐式转换为bool值
＊spi = 253;                                       //使用解引用操作符＊

shared_ptr<string>  sps(new string("smart"));      //一个string的shared_ptr
assert(sps->size() == 5);                          //使用箭头操作符->
```

但shared_ptr的名字表明了它与scoped_ptr的主要不同：**它是可以被安全共享的——shared_ptr是一个“全功能”的类，有着正常的拷贝、赋值语义，也可以进行shared_ptr间的比较，是“最智能”的智能指针。**

shared_ptr有多种形式的`构造函数`，应用于各种可能的情形：

- 无参的`shared_ptr()`创建一个持有空指针的shared_ptr；
- shared_ptr(Y ＊ p)**获得指向类型T的指针p的管理权**，同时引用计数置为1。这个构造函数要求Y类型必须能够转换为T类型；
- shared_ptr(shared_ptr const & r)从另外一个shared_ptr获得指针的管理权，同时引用计数加1，**结果是两个shared_ptr共享一个指针的管理权**；
- shared_ptr(std::auto_ptr & r)从一个`auto_ptr`获得指针的管理权，引用计数置为1，同时`auto_ptr`自动失去管理权；
- **operator=赋值操作符**可以从另外一个shared_ptr或auto_ptr获得指针的管理权，其行为同构造函数；
- shared_ptr(Y ＊ p, D d)行为类似shared_ptr(Y ＊ p)，但使用参数d指定了析构时的**定制删除器**，而不是简单的delete。这部分将在3.4.8节详述。

**shared_ptr的`reset()`函数的行为与scoped_ptr也不尽相同，它的作用是将引用计数减1**，停止对指针的共享，除非引用计数为0，否则不会发生删除操作。带参数的reset()则类似相同形式的构造函数，原指针引用计数减1的同时改为管理另一个指针。

`unique()`在`shared_ptr`是指针的唯一所有者时返回true。

**shared_ptr可以被用于`标准关联容器`（set和map）**。

`shared_ptr`提供了类似的转型函数`static_pointer_cast()`、`const_pointer_cast()`和`dynamic_pointer_cast()`，它们与标准的转型操作符`static_cast`、`const_cast` 和`dynamic_cast`类似，但返回的是转型后的`shared_ptr`。

```cpp
shared_ptr<std::exception> sp1(new bad_exception("error"));
shared_ptr<bad_exception>  sp2 = dynamic_pointer_cast<bad_exception>(sp1);
shared_ptr<std::exception> sp3 = static_pointer_cast<std::exception>(sp2);
assert(sp3 == sp1);
```

此外，`shared_ptr`还支持流输出操作符`operator<<`，输出内部的指针值，方便调试。

### 用法 

`shared_ptr`的智能使其行为最接近原始指针，因此它比`auto_ptr`和`scoped_ptr`的应用范围更广。几乎是100%可以在任何`new`出现的地方接受`new`的动态分配结果，然后被任意使用，从而完全消灭`delete`的使用和内存泄漏，而它的用法与`auto_ptr`和`scoped_ptr`一样的简单。

`shared_ptr`也提供基本的线程安全保证，一个`shared_ptr`可以被多个线程安全读取，但其他的访问形式结果是未定义的。
**例1：**

```cpp
shared_ptr<int> sp(new int(10));           //一个指向整数的shared_ptr
assert(sp.unique());                       //现在shared_ptr是指针的唯一持有者

shared_ptr<int> sp2 = sp;                  //第二个shared_ptr，拷贝构造函数

//两个shared_ptr相等，指向同一个对象，引用计数为2
assert(sp == sp2 && sp.use_count() == 2);

＊sp2 = 100;                               //使用解引用操作符修改被指对象
assert(＊sp == 100);                       //另一个shared_ptr也同时被修改

sp.reset();                                //停止shared_ptr的使用
assert(! sp);                              //sp不再持有任何指针（空指针）
```

**例2：**

```cpp
#include<iostream>
using namespace std;

#include<memory>

class shared                               //一个拥有shared_ptr的类
{
  private:
    shared_ptr<int> p;                       //shared_ptr成员变量
  public:
    shared(shared_ptr<int> p_):p(p_){}       //构造函数初始化shared_ptr
    void print()                             //输出shared_ptr的引用计数和指向的值
    {
        cout << "count:" << p.use_count()
          << "v =" <<*p << endl;
    }
};

void print_func(shared_ptr<int> p)         //使用shared_ptr作为函数参数
{
  //同样输出shared_ptr的引用计数和指向的值
  cout << "count:" << p.use_count()
      << " v=" <<*p << endl;
}
int main()
{
  shared_ptr<int> p(new int(100));
  shared s1(p), s2(p);                     //构造两个自定义类

  s1.print();
  s2.print();

  *p = 20;                                //修改shared_ptr所指的值
  print_func(p);

  s1.print();
}
count:3v =100
count:3v =100
count:4 v=20
count:3v =20
```

### 工厂函数 

`shared_ptr`很好地消除了显式的`delete`调用，如果读者掌握了它的用法，可以肯定delete将会在你的编程字典中彻底消失。

但这还不够，因为shared_ptr的构造还需要new调用，这导致了代码中的某种不对称性。虽然shared_ptr很好地包装了new表达式，但过多的显式new操作符也是个问题，显式delete调用应该使用工厂模式来解决。

因此，shared_ptr提供了一个自由工厂函数（位于boost名字空间）make_shared()，来消除显式的new调用，它的名字模仿了标准库的make_pair().

**make_shared()函数可以接受若干个参数，然后把它们传递给类型T的构造函数，创建一个shared_ptr的对象并返回。**通常make_shared()函数要比直接创建shared_ptr对象的方式快且高效，因为它内部**仅分配一次内存，消除了shared_ptr构造时的开销。**

```cpp
int main()
{
  shared_ptr<string> sp = make_shared<string>("make_shared");   //创建string的共享指针
  shared_ptr<vector<int> > spv = make_shared<vector<int> >(10, 2);     //创建vector的共享指针
  assert(spv->size() == 10);
}
```

### 应用于标准容器 

**有两种方式可以将`shared_ptr`应用于标准容器（或者容器适配器等其他容器）**。

一种用法是将容器作为`shared_ptr`管理的对象，如`shared_ptr<list<T> >`，使容器可以被安全地共享，用法与普通shared_ptr没有区别，我们不再讨论。

另一种用法是将`shared_ptr`作为容器的元素，如`vector<shared_ptr<T> >`，因为shared_ptr支持拷贝语义和比较操作，符合标准容器对元素的要求，所以可以在容器中安全地容纳元素的指针而不是拷贝。

标准容器不能容纳auto_ptr，这是C++标准特别规定的。标准容器也不能容纳scoped_ptr，因为scoped_ptr不能拷贝和赋值。标准容器可以容纳原始指针，但这就丧失了容器的许多好处，因为标准容器无法自动管理类型为指针的元素，必须编写额外的大量代码来保证指针最终被正确删除，这通常很麻烦很难实现。

存储shared_ptr的容器与存储原始指针的容器功能几乎一样，但shared_ptr为程序员做了指针的管理工作，可以任意使用shared_ptr而不用担心资源泄漏。

### 定制删除器 

shared_ptr(Y ＊ p, D d)的第一个参数是要被管理的指针，它的含义与其他构造函数的参数相同。而第二个删除器参数d则告诉shared_ptr在析构时不是使用delete来操作指针p，而要用d来操作，即把delete p换成d(p)。

在这里删除器d可以是一个函数对象，也可以是一个函数指针，只要它能够像函数那样被调用，使得d(p)成立即可。**对删除器的要求是它必须可拷贝，行为必须也像delete那样，不能抛出异常**。

```cpp
void any_func(void＊ p)                        //一个可执行任意功能的函数
{ cout << "some operate" << endl; }

int main()
{
  shared_ptr<void> p((void＊)0, any_func);     //容纳空指针，定制删除器
}                                              //退出作用域时将执行any_func()  
```

### 与std::shared_ptr的区别 

C++11标准中定义了`std::shared_ptr`，功能与`boost::shared_ptr`基本相同，但多了>、<=等操作符的重载，实际的差别很小，基本可以等价互换。

### shared_ptr 

shared_ptr能够存储void＊型的指针，而void＊型指针可以指向任意类型，因此shared_ptr就像是一个泛型的指针容器，拥有容纳任意类型的能力。

但将指针存储为void＊同时也丧失了原来的类型信息，为了在需要的时候正确使用，可以用static_pointer_cast等转型函数重新转为原来的指针。但这涉及运行时**动态类型转换，它会使代码不够安全**，建议最好不要这样使用。

批注：`(*void)`的行为是只包含数据的起始地址，没有大小！只有转换回原来的指针类型，才能被正确获取数据大小！

## shared_array 

`shared_array`能力有限，多数情况下它可以用shared_ptr、[std::vector](std::vector)或者`std::vector<shared_ptr>`来代替，这两个方案具有更好的安全性和更多的灵活性，而所付出的代价几乎可以忽略不计。

## weak_ptr 

`weak_ptr`是为配合`shared_ptr`而引入的一种智能指针，**它更像是`shared_ptr`的一个助手而不是智能指针**，因为它不具有普通指针的行为，没有重载operator＊和->。它的最大作用在于协助`shared_ptr`工作，像旁观者那样观测资源的使用情况。

### 用法 

weak_ptr被设计为与`shared_ptr`共同工作，**可以从一个`shared_ptr`或者另一个`weak_ ptr`对象构造，获得资源的观测权**。但weak_ptr没有共享资源，它的构造不会引起指针引用计数的增加。同样，在`weak_ptr`析构时也不会导致引用计数减少，它只是一个静静的观察者。

使用`weak_ptr`的成员函数`use_count()`可以观测资源的引用计数，**另一个成员函数`expired()`的功能等价于`use_count()==0`，**但更快，表示被观测的资源（也就是`shared_ptr`管理的资源）已经不复存在。

```cpp
shared_ptr<int> sp(new int(10));           //一个shared_ptr
assert(sp.use_count() == 1);

weak_ptr<int> wp(sp);                      //从shared_ptr创建weak_ptr
assert(wp.use_count() == 1);               //weak_ptr不影响引用计数

if (! wp.expired())                        //判断weak_ptr观察的对象是否失效
{
  shared_ptr<int> sp2 = wp.lock();         //获得一个shared_ptr
  ＊sp2 = 100;
  assert(wp.use_count() == 2);
}                                          //退出作用域，sp2自动析构，引用计数减1
assert(wp.use_count() == 1);
sp.reset();                                //shared_ptr失效
assert(wp.expired());
assert(! wp.lock());                       //weak_ptr将获得一个空指针
```

## pool库概述 

如果读者学习过操作系统相关的课程，学习过操作系统的内存管理机制和内存分配算法等知识，那么就可能了解“内存池”的概念。简单来说，内存池预先分配了一块大的内存空间，然后就可以在其中使用某种算法实现高效快速的自定制内存分配。

TODO.

# 实用工具 

## noncopyable 

noncopyable允许程序轻松地实现一个禁止复制的类。

noncopyable位于名字空间boost，为了使用noncopyable组件，需要包含头文件：

```cpp
#include <boost/noncopyable.hpp>                        //或者
#include <boost/utility.hpp>
```

在C++中定义一个类时，如果不明确定义复制构造函数和复制赋值操作符，编译器会为我们自动生成这两个函数的空实现。

## 用法 

noncopyable为实现不可复制的类提供了简单清晰的解决方案：从boost::noncopyable派生即可。

```cpp
#include <boost/utility.hpp>
class do_not_copy: boost:: noncopyable

{...};
```

注意，这里使用默认的私有继承是允许的。我们也可以显式写出private或者public修饰词，但效果是相同的。因此直接这样写少输入了一些代码，也更清晰，并且表明了HAS-A关系（而不是IS-A）。

如果有其他人误写了代码（很可能是没有仔细阅读接口文档），企图复制构造或者赋值do_not_copy，那么将不能通过编译器的审查：

```cpp
do_not_copy d1;
do_not_copy d2(d1);                       //编译出错！
do_not_copy d3;
d3 = d1;                                  //编译出错！
```

使用Mac OS X的Clang编译会报出类似下面的错误提示：

```
main.cpp:15:13: error: call to deleted constructor of ' do_not_copy'
do_not_copy d2(d1);
            ^  ～～
main.cpp:9:7: note: function has been explicitly marked deleted here
class do_not_copy: boost:: noncopyable
```

这条错误信息明确地告诉我们：类使用boost:: noncopyable禁用（delete）了复制构造，无法调用复制构造函数。

## typeof 

C++11标准中重新定义了“古老的新特性”auto，并新增了decltype关键字，可以自动推导表达式的类型，能够极大地减轻书写烦琐的变量类型声明的工作并简化代码。typeof库使用宏模拟了这两个关键字，使C++98也可以使用这一方便的特性。

略。

## optional 

optional库使用“容器”语义，包装了“可能产生无效值”的对象，实现了“未初始化”的概念。

optional位于名字空间boost，为了使用optional，需要包含头文件<boost/optional.hpp>，即：

```cpp
#include <boost/optional.hpp>
using namespace boost;
```

### “无意义”的值 

函数并不是总能返回有效值，很多时候函数可能返回“无意义”的值，这不意味着函数执行失败，而是表明函数正确执行了，但结果却不是有用的值。如果用数学语言来解释，就是返回值位于函数解空间之外(值域之外)。

例如，求一个数的倒数，在实数域内开平方，在字符串中查找子串，它们都可能返回无效的值。有些无效返回的情况可以用抛出异常的方式来通知用户，但有的情况下这样代价很高，或者不允许异常，这时必须要以某种合理的高效的方式通知用户。

表示返回值无意义最常用的做法是增加一个“哨兵”的角色，它位于解空间之外，如NULL、-1、EOF、string::npos、vector::end()等。但这些做法不够通用，而且很多时候不存在解空间之外的“哨兵”。

optional使用“容器”语义，为这种“无效值”的情形提供了一个较好的解决方案。

### 操作函数 

optional的模板类型参数T可以是任何类型，就如同一个标准容器对元素的要求，并不需要T具有默认构造函数，但必须是可复制构造的。

可以有很多方式创建optional对象，例如：

- 无参的optional()或者optional(boost::none)构造一个未初始化optional对象。参数boost::none是一个类似空指针的none_t类型常量，表示未初始化；
- optional(v)构造一个已初始化的optional对象，为复制v的值。如果模板类型为T&，那么optional内部持有对引用的包装；
- optional(condition, v)根据条件condition来构造optional对象，如果条件成立（true）则初始化为v，否则为未初始化；
- 此外optional还支持复制构造和赋值操作，可以从另一个optional对象构造。当想让一个optional对象重新恢复到未初始化状态时，可以向对象赋none值。

optional采用了指针语义来访问内部保存的元素，这使得optional未初始化时的行为就像一个空指针。它重载了operator＊和operator->以实现与指针相同的操作，get()和get_ptr()可以以函数的操作形式获得元素的引用和指针。

成员函数get_value_or(default)是一个特别的访问函数，可以保证返回一个有效的值，如果optional已初始化，那么返回内部的元素，否则返回default。

optional也可以用隐式类型转换进行bool测试（用于条件判断），就像一个对指针的判断。

optional还全面支持比较运算，包括==、! =、<、<、>、>=。与普通指针比较的“浅比较”（仅比较指针值）不同，optional的比较是“深比较”，同时加入了对未初始化情况的判断。

### 用法 

```cpp
#include<iostream>
using namespace std;

#include<vector>
#include<string>
#include <boost/optional.hpp>
using namespace boost;
int main()
{
    optional<int> op0;       //一个未初始化的optional对象
    optional<int> op1(none); //同上，使用none赋予未初始化值

    assert(!op0);
    assert(op0 == op1);
    assert(op1.get_value_or(253) == 253); //获取可选值

    optional<string> ops("test"); //初始化为字符串test
    cout << *ops << endl;        //用解引用操作符获取值
    vector<int> v(10);
    optional<vector<int> &> opv(v); //容纳一个容器的引用
    assert(opv);

    opv->push_back(5); //使用箭头操作符操纵容器
    assert(opv->size() == 11);

    opv = none; //置为未初始化状态
    assert(!opv);
}
test
```

另一个例子：

```cpp
#include <iostream>
using namespace std;

#include <vector>
#include <cmath>
#include <boost/optional.hpp>
using namespace boost;

optional<double> calc(int x) //计算倒数
{
    return optional<double>(x != 0, 1.0 / x); //条件构造函数
}

optional<double> sqrt_op(double x) //计算实数的平方根
{
    return optional<double>(x > 0, sqrt(x)); //条件构造函数
}

int main()
{
    optional<double> d = calc(10);

    if (d) //bool语境测试optional的有效性
    {
        cout << *d << endl;
    }

    d = sqrt_op(-10);
    if (!d) //使用重载的逻辑非操作符
    {
        cout << "no result" << endl;
    }
}
0.1
no result
```

### 工厂函数 

optional提供一个类似make_pair()、make_shared()的工厂函数make_optional()，可以根据参数类型自动推导optional的类型，用来辅助创建optional对象。

但make_optional()无法推导出T引用类型的optional对象，因此如果需要一个optional<T&>的对象，就不能使用make_optional()函数。

```cpp
#include <boost/optional.hpp>
using namespace boost;

int main()
{
  auto x = make_optional(5);             //使用auto关键字自动推导类型
  assert(*x == 5);

  auto y = make_optional<double>((*x > 10), 1.0);
  assert(! y);
}
```

## assign 

许多情况下我们都需要为容器初始化或者赋值，填入大量的数据，比如初始错误代码和错误信息，或者是一些测试用的数据。标准容器仅提供了容纳这些数据的方法，但填充的步骤却是相当地麻烦，必须重复调用insert()或者push_back()等成员函数，这正是boost.assign出现的理由。

assign库重载了赋值操作符operator+=、逗号操作符operator，和括号操作符operator()，可以用难以想象的简洁语法非常方便地对标准容器赋值或者初始化。在需要填入大量初值的地方很有用，本书8.1节介绍的foreach库和其他很多地方都大量使用了assign，可以做进一步的参考。

assign库位于名字空间boost::assign，为了使用assign库，需要包含头文件<boost/assign.hpp>，它包含了大部分assign库的工具，即

```cpp
#include <boost/assign.hpp>
using namespace boost::assign;
```

### 使用操作符+=向容器增加元素 

boost.assign的用法非常简单，由于重载了操作符+=和逗号，可以用简洁到令人震惊的语法完成原来用许多代码才能完成的工作，如果不熟悉C++操作符重载的原理你甚至都不会意识到在简洁语法下的复杂工作。

```cpp
#include <iostream>
using namespace std;

#include <vector>
#include <set>
#include <boost/assign.hpp>

int main()
{
    using namespace boost::assign;

    //很重要，启用assign库的功能
    vector<int> v;            //标准向量容器
    v += 1, 2, 3, 4, 5, 6*6; //用operator+=和，填入数据

    set<string> s; //标准集合容器
    s += "cpp", "java", "c#", "python";

    map<int, string> m; //标准映射容器
    m += make_pair(1, "one"), make_pair(2, "two");
}
```

operator+=很好用，但有一点遗憾，它仅限应用于STL中定义的标准容器（vector、list、set等），对于其他类型的容器（如Boost新容器）则无能为力。

# 字符串与文本处理 

## format 

C++标准库提供了强大的、富有弹性的输入/输出流处理，使用流可以对输出的格式做各种精确的控制，如宽度、精度、进制、填充字符、对齐等，新式流输出操作符<<可以串联起任意数量的参数，非常自由。

但C++输入/输出流也不是完美无瑕的，精确输出的格式控制要写大量的操控函数，而且还会改变流的状态，用完后还需要及时恢复，有时候会显得十分烦琐（2.3.3节就是一个很好的例子）。因此，还是有很多程序员怀念C语言中经典的printf()，虽然它缺乏类型安全检查，还有其他的一些缺点，但它语法简单高效，并且被广泛地接受和使用，影响深远。许多其他的编程语言也都受printf()的影响提供类似的格式化输出机制，比如Python、C#。

boost.format库“扬弃”了printf，实现了类似于printf()的格式化对象，可以把参数格式化到一个字符串，而且是完全类型安全的。

format组件位于名字空间boost，为了使用format组件，需要包含头文件<boost/format.hpp>，即：

```cpp
#include <boost/format.hpp>
using namespace boost;
```

### 简单的例子 

我们先通过一个简单的例子来了解format，看看它与printf()有什么相似和不同：

```cpp
#include <iostream>
using namespace std;

#include <boost/format.hpp>
using namespace boost;

int main()
{
    cout << format("%s:%d+%d=%d\n") % "sum" % 1 % 2 % (1 + 2);

    format fmt("(%1% + %2%) ＊ %2% = %3%\n");
    fmt % 2 % 5;
    fmt % ((2 + 5) *5);
    cout << fmt.str();
}
sum:1+2=3
(2 + 5) ＊ 5 = 35
```

C/C++程序员都应该对这段代码有似曾相识的感觉，format的设计在很大程度上参照了printf()，但用法上却有很大的不同。

程序的第一条语句演示了format的最简单用法，使用format(...)构造了一个format临时（匿名）对象。构造函数的参数是格式化字符串，其语法是我们非常熟悉的标准printf()语法，使用%x来指定参数的格式。

因为要被格式化的参数个数是不确定的，printf()使用了C语言里的可变参数（即参数声明中的省略号），但它是不安全的。format模仿了流操作符<<，重载了二元操作符operator%作为参数输入符，它同样可以串联任意数量的参数，因此，

```cpp
format(...) % a % b %c
```

可以理解成：

```cpp
format(...) << a << b << c
```

操作符%把参数逐个地“喂”给format对象，完成对参数的格式化。

最后，format对象支持流输出，可以直接向输出流cout输出内部保存的已格式化好的字符串。

第一条format语句的等价printf()调用是

```cpp
printf("%s:%d+%d=%d\n", "sum", 1 , 2 , (1+2));
```

操作符%把参数逐个地“喂”给format对象，完成对参数的格式化 [3] 。

最后，format对象支持流输出，可以直接向输出流cout输出内部保存的已格式化好的字符串。

第一条format语句的等价printf()调用是

```cpp
printf("%s:%d+%d=%d\n", "sum", 1 , 2 , (1+2));
```

第二个format对象的等价printf()调用是

```cpp
printf("(%d + %d) ＊ %d = %d\n",2, 5, 5, (2+5)＊5);
```

### 格式化语法 

format基本继承了printf的格式化语法，以便老程序员能够尽快掌握，它仅对printf语法有少量的不兼容，一般情况下我们很难遇到。

每个printf格式化选项以%开始，后面是格式规则，规定了输出的对齐、宽度、精度、字符类型，我们都非常熟悉，例如，

- %05d ： 输出宽度为5的整数，不足位用0填充；
- %-8.3f ： 输出左对齐，总宽度为8，小数位3位的浮点数；
- % 10s ： 输出10位的字符串，不足位用空格填充；
- %05X ： 输出宽度为5的大写十六进制整数，不足位用0填充。

```cpp
format fmt("%05d\n%-8.3f\n% 10s\n%05X\n");
cout << fmt %62 % 2.236 % "123456789" % 48;
```

参考[更多print格式](https://www.cnblogs.com/chendeqiang/p/12861588.html)

### format的性能 

printf()不进行类型检查，直接向stdout输出，因此它的速度非常快，而format较printf()做了很多安全检查的工作，因此性能略差，速度上要慢很多。

如果很在意format的性能，那么可以先建立const format对象，然后复制这个对象进行格式化操作，这样比直接使用format对象能够提高速度，像这样：

```cpp
const format fmt("%10d %020.8f %010X %10.5e\n");   //常量对象
cout << format(fmt) %62 % 2.236 % 255 % 0.618;     //复制使用
```

# 容器与数据结构 

## variant 

variant与any有些类似，是一种可变类型，是对C/C++中union概念的增强和扩展。普通的union只能持有POD（普通数据类型），而不能持有如string、vector等复杂类型，variant则没有这个限制。

variant位于名字空间boost，为了使用variant组件，需要包含头文件<boost/variant.hpp>，即：

```cpp
#include <boost/variant.hpp>
using namespace boost;
```

### 访问元素 

variant的操作要比any方便，能够直接访问元素的值，例如：

```cpp
variant<int, float, string> v;                //可容纳int, float和string
v = "123";                                    //v持有一个string对象 
```

但为了实现泛型编程，variant也提供一个外界自由函数来访问内部元素。variant没有提供variant_cast的函数，而是使用boost库中的一个泛型函数get()。这是因为variant的设计出发点与any不同，它的目的是存储多个数据的联合，而不是任意类型的容器。

用于访问variant元素的get()函数声明如下：

```cpp
template<typename U, ...>   U ＊ get(variant ＊ operand);
template<typename U, ...>   const U ＊ get(const variant ＊ operand);
template<typename U, ...>   U & get(variant & operand);
template<typename U, ...>   const U & get(const variant & operand);
```

它的声明与any_cast()类似，用法也基本相同。

如果variant当前的值不是get()想取的类型，那么会抛出boost::bad_get异常，它是std:: exception的子类，但没有使用boost.exception库（参见4.9节）进行包装。如果是返回指针的get()形式，那么会返回一个空指针。

### 用法 

使用variant，我们必须要在模板参数中指定它所能容纳的类型。类型的数量默认最多是20个（一个相当惊人的数字）。例如，声明一个能容纳int、double和string的variant类型，其写法与tuple很像：

```cpp
typedef variant<int, double, string> var_t;

var_t v(1);                                   //v->int
v = 2.13;                                     //v->double
var_t v2("string type");                      //v2->string
v2 = v;                                       //v2->double
```

我们也可以使用自由函数get()来获取variant的值,但get()函数通常不是最方便最有效的访问方法，它与any_cast同样存在着类型不安全的隐患，操作时必须查询variant当前值的类型。

```cpp
#include <iostream>
using namespace std;

#include <string>
#include <boost/variant.hpp>
using namespace boost;

int main()
{
    typedef variant<int, double, string> var_t;
    var_t v;                         //缺省构造，v==0
    assert(v.type() == typeid(int)); //v->int
    assert(v.which() == 0);          //v现在持有第一个类型的元素

    v = "variant demo";                //v->string
    cout << *get<string>(&v) << endl; //使用get()函数取值

    try
    {
        cout << get<double>(v) << endl; //抛出异常
    }
    catch (bad_get &)
    {
        cout << "bad_get" << endl;
    }
}
variant demo
bad_get
```

### 访问器 

variant基于访问者模式提供了模板类static_visitor，它解耦了variant的数据存储和访问操作，把访问操作集中在访问器类，易于增加新的访问操作，使这两者可以彼此独立地变化。

使用static_visitor首先要从static_visitor继承，然后重载operator()，用来访问variant的内部值。访问器必须能够处理variant所可能拥有的所有类型，不能仅处理其中的一部分，否则会引发编译错误：

```cpp
class var_print : public static_visitor<void>   //返回值类型为void
{
  template<typename T>                       //使用模板函数来处理所有元素类型
  void operator()(T &i) const                //通常是const函数
  {
      i *= 2;
      cout << i << endl;                     //输出以验证
  }
};
```

函数对象var_print的operator()是一个模板函数，可以对普通类型执行加倍操作，然后再输出。

假设我们把variant增加一个vector的类型：

```cpp
typedef variant<int, double, vector<int> > var_t;
```

那么访问器的operator()需要增加一个针对vector类型的重载，而原有的处理代码不必做任何变动：

```cpp
template<>
void operator()<vector<int> >(vector<int> &v) const
{
  copy(v.begin(), v.end(), back_inserter(v)) ;
  for (auto& x : v)
    {
      cout << x << ", ";                      //输出以验证
    }
  cout << endl;
}
```

请读者注意针对vector类型的operator()的特化形式，特化的模板参数在第一个括号之后，因为模板参数必须在函数名称之后，而括号操作符函数的全名是operator()。

将访问器对象应用于variant，需要使用函数apply_visitor()。apply_visitor()有多种重载形式，但最常用的形式是接受一个访问器对象和一个variant对象，对variant内的值调用访问器对象的operator()。例如：

```cpp
var_t v(1);                                   //一个variant对象
var_print vp;                                 //一个访问器对象
auto vp = apply_visitor(var_print());         //获得新的函数对象
vp(v);                                        //直接访问variant，不必在调用
                                              //apply_visitor()

var_t v(1);                                   //一个variant对象
apply_visitor(vp, v);                         //应用访问器，翻倍并输出v的值

v = 3.414;                                    //v->double
apply_visitor(vp, v);                         //应用访问器，翻倍并输出v的值

using namespace boost::assign;
v = vector<int>(list_of(1)(2));               //v-> vector<int>
```

# 操作系统相关 

## cpu_timer 

TODO.

# 函数与回调 

## result_of 

result_of是一个很小但很有用的组件，可以帮助程序员确定一个调用表达式的返回类型，主要用于泛型编程和其他Boost库组件，它已被收入C++11标准。

result_of位于名字空间boost，为了使用result_of组件，需要包含头文件<boost/utility/result_of.hpp>，即：

```cpp
#include <boost/utility/result_of.hpp>
using namespace boost;
```

### 用法 

例如这样的一个简单的泛型函数：

```cpp
template<typename T, typename T1>
?? ? call_func(T t, T1 t1)                        //T是个可调用的类型
{ return t(t1); }
```

无论如何，auto都派不上用场，这里不存在任何赋值表达式，只有函数调用式。

这正是result_of发挥威力的机会，它可以正确推导出返回类型，像这样：

```cpp
template<typename T, typename T1>
typename result_of<T(T1)>::type call_func(T t, T1 t1)
{ return t(t1); }
```

这里必须在result_of<>::type前加上关键字typename，否则编译器会认为type是result_of的成员变量，从而产生找不到声明的编译错误。

仍然使用刚才定义的函数指针，使用result_of的完整程序如下：

```cpp
#include <boost/utility/result_of.hpp>
using namespace boost;

template<typename T, typename T1>
typename result_of<T(T1)>::type call_func(T t, T1 t1)
{ return t(t1); }

int main()
{
  typedef double (＊Func)(double d);
  Func func = sqrt;
  auto x =  call_func(func, 5.0);       //赋值表达式，可以用auto
  cout << typeid(x).name();
}
```

## ref 

基本就是C++11中的引用类型。

TODO.

## bind 

bind是对C++标准库中函数适配器bind1st/bind2nd的泛化和增强，可以适配任意的可调用对象，包括函数指针、函数引用、成员函数指针和函数对象。bind远远地超越了STL中的函数绑定器（bind1st/bind2nd），可以绑定最多9个函数参数，而且对被绑定对象的要求很低，可以在没有result_type内部类型定义的情况下完成对函数对象的绑定。bind库很好地增强了标准库的功能，已经被收入C++11标准。

bind位于名字空间boost，为了使用bind组件，需要包含头文件<boost/bind.hpp>，即：

```cpp
#include <boost/bind.hpp>
using namespace boost;
```

### 绑定普通函数 

bind可以绑定普通函数，包括函数、函数指针，假设我们有如下的函数定义：

```cpp
int f(int a, int b)                          //二元函数
{ return a + b; }

int g(int a, int b, int c)                   //三元函数
{ return a + b ＊ c; }

typedef int (＊f_type)(int, int);            //函数指针定义
typedef int (＊g_type)(int, int, int);       //函数指针定义

//使用时搭配占位符更香!
bind(f, _1,  9)(x) ;                          //f(x, 9)，相当于bind2nd(f,9)
bind(f, _1, _2)(x, y) ;                       //f(x, y)
bind(f, _2, _1)(x, y) ;                       //f(y, x)
bind(f, _1, _1)(x, y) ;                       //f(x, x), y参数被忽略
bind(g, _1, 8, _2)(x, y) ;                    //g(x, 8, y)
bind(g, _3, _2, _2)(x, y, z) ;                //g(z, y, y), x参数被忽略

//也可以绑定函数指针
f_type pf = f;
g_type pg = g;
int x = 1, y = 2, z = 3;
cout << bind(pf, _1, 9)(x) << endl;             //(＊pf)(x,9)
cout << bind(pg, _3, _2, _2)(x, y, z) << endl; //(＊pg)(z, y, y)
```

### 绑定成员函数 

bind也可以绑定类的成员函数。

类的成员函数不同于普通函数，因为成员函数指针不能直接调用operator()，它必须被绑定到一个对象或者指针，然后才能得到this指针进而调用成员函数。因此bind需要“牺牲”一个占位符的位置，要求用户提供一个类的实例、引用或者指针，通过对象作为第一个参数来调用成员函数，即：

```cpp
bind( &X::func, x, _1, _2, ...)
```

这意味着使用成员函数时只能最多绑定8个参数,其中x是对象，func是成员函数。

bind能够绑定成员函数，这是个非常有用的功能，它可以替代标准库中令人迷惑的mem_fun和mem_fun_ref绑定器，用来配合标准算法操作容器中的对象。
下面的代码使用bind搭配标准算法for_each用来调用容器中所有对象的print()函数：

```cpp
#include <boost/bind.hpp>
using namespace boost;

struct point                          //一个二维点的类
{
  int x, y;
  point(int a = 0, int b = 0):x(a), y(b){}
  void print()
  {   cout << "(" << x << ", " << y << ")\n";    }
};

int main()
{
    vector<point> v(10);
    for_each(v.begin(), v.end(), bind(&point::print, _1));
}
```

bind同样支持绑定虚拟成员函数，用法与非虚函数相同，虚函数的行为将由实际调用发生时的实例来决定。

### 绑定成员变量 

bind的另一个对类的操作是它可以绑定public成员变量，就像是一个选择器，用法与绑定成员函数类似，只需要把成员变量名像一个成员函数一样去使用。
仍然以point类为例子，假设我们已经在vector中存储了大量的point对象，而我们想要得到它们的x坐标值，那么bind可以这样使用：

```cpp
vector<point> v(10);
vector<int> v2(10);

transform(v.begin(), v.end(), v2.begin(), bind(&point::x, _1));

BOOST_FOREACH(int x, v2)                     //foreach循环输出值
  cout << x << ", ";
```

代码中的bind(&point::x, _1)取出point对象的成员变量x, transform算法调用bind表达式操作容器v，逐个把成员变量填入到v2中。

### 绑定函数对象 

```cpp
struct F
{
    int operator()(int a, int b) { return a – b; }
    bool operator()(long a, long b) { return a == b; }
};

F f;
int x = 100;
bind<int>(f, _1, _1)(x);        // f(x, x)
```

可能某些编译器不支持上述的bind语法，可以用下列方式代替：

```cpp
boost::bind(boost::type<int>(), f, _1, _1)(x);
```

默认情况下，bind拥有的是函数对象的副本，但是也可以使用boost::ref和boost::cref来传入函数对象的引用，尤其是当该function object是non-copyable或者expensive to copy。

### 使用ref库 

bind采用拷贝的方式存储绑定对象和参数，这意味着绑定表达式中的每个变量都会有一份拷贝，如果函数对象或值参数很大、拷贝代价很高，或者无法拷贝，那么bind的使用就会受到限制。

因此bind库可以搭配ref库使用，ref库包装了对象的引用，可以让bind存储对象引用的拷贝，从而降低了拷贝的代价。但这也带来了一个隐患，因为有时候bind的调用可能会延后很久，程序员必须保证bind被调用时引用是有效的。如果调用时引用的变量或者函数对象被销毁了，那么会发生未定义行为。

```cpp
int x = 10;
cout << bind(g, _1, cref(x), ref(x))(10) << endl;

f af;                                                      //一个函数对象
cout << bind<int>(ref(af), _1, _2)(10, 20) << endl;
```

## function 

function是一个函数对象的“容器”，概念上像是C/C++中函数指针类型的泛化，是一种“智能函数指针”。它以对象的形式封装了原始的函数指针或函数对象，能够容纳任意符合函数签名的可调用对象。因此，它可以被用于回调机制，暂时保管函数或函数对象，在之后需要的时机再调用，使回调机制拥有更多的弹性。它已经被收入C++11标准。

function可以配合bind使用，存储bind表达式的结果，使bind可以被多次调用。

function位于名字空间boost，为了使用function组件，需要包含头文件<boost/function.hpp>，即：

```cpp
#include <boost/function.hpp>
using namespace boost;
```

### function的声明 

```cpp
function< int(int,int,int) > func2;
```

### 操作函数 

function的构造函数可以接受任意符合模板中声明的函数类型的可调用对象，如函数指针和函数对象，也可以是另一个function对象的引用，之后在内部存储一份它的拷贝。

无参的构造函数或者传入空指针构造将创建一个空的function对象，不持有任何可调用物，调用空的function对象将抛出bad_function_call异常，因此在使用function前最好检测一下它的有效性。可以用empty()测试function是否为空，或者用重载操作符operator！来测试。function对象也可以在一个bool语境中直接测试它是否为空，它是类型安全的。

function的其余成员函数功能如下：

- lear()可以直接将function对象置空，它与使用operator=赋值0具有同样的效果；
- 模板成员函数target()可以返回function对象内部持有的可调用物Functor的指针，如果function为空则返回空指针nullptr；
- contains()可以检测function是否持有一个Functor对象；
- 最后，function提供了operator()，它把传入的参数转交给内部保存的可调用物，完成真正的函数调用。

### 比较操作 

function重载了比较操作符operator==和operator! =，可以与被包装的函数或函数对象进行比较。如果function存储的是函数指针，那么比较相当于

```cpp
function.target<Functor>() == func_pointer
```

例如：

```cpp
function<int(int, int)> func(f);
assert( func == f );
```

如果function存储的是函数对象，那么要求函数对象必须重载了operator==，是可比较的。

两个function对象不能使用和！=直接比较，这是特意的。因为function存在bool的隐式转换，function定义了两个function对象的operator但没有实现，企图比较两个function对象会导致编译错误。

### 用法 

function就像是一个函数的容器，也可以把function想象成一个泛化的函数指针，只要符合它声明中的函数类型，任何普通函数、成员函数、函数对象都可以存储在function对象中，然后在任何需要的时候被调用。

function这种能够容纳任意可调用对象的能力是非常重要的，在编写泛型代码的时候尤其有用，它使我们可以接受任意的函数或函数对象，增加程序的灵活性。

与原始的函数指针相比，function对象的体积要稍微大一点（3个指针的大小），速度要稍微慢一点（10%左右的性能差距），但这与它带给程序的巨大好处相比是无足轻重的。

```cpp
#include<iostream>
using namespace std;

#include <boost/function.hpp>
using namespace boost;

int f(int a, int b)                        //声明一个二元函数
{
  return a + b;
}

int main()
{
  boost::function<int(int, int)>func;          //无参构造一个function对象
  assert(!func);                          //此时function不持有任何对象

  func = f;                                //func存储了函数f
  if (func)                                //function可以转换为bool值
  {
    cout << func(10, 20);                //调用function的operator()
  }

  func = 0;                                //function清空，相当于clear()
  assert(func.empty());
}
30
```

只要函数签名式一致，function也可以存储成员函数和函数对象，或者是bind表达式的结果。假设我们有如下的一个类demo_class，它既有普通成员函数，又重载了operator()：

```cpp
struct demo_class
{
  int add(int a, int b)                  //加法操作
  {       return a + b;  }
  int operator()(int x) const            //重载operator()
  {       return x＊x; }
};
```

存储成员函数时可以直接在function声明的函数签名式中指定类的类型，然后用bind绑定成员函数：

```cpp
function<int(demo_class&, int,int)>func1;

func1 = bind(&demo_class::add, _1, _2, _3);

demo_class sc;
cout << func1(sc, 10, 20);
```

也可以在函数类型中仅写出成员函数的签名，在bind时直接绑定类的实例：

```cpp
function<int(int,int)>func2;

func2 = bind(&demo_class::add, &sc, _1, _2);
cout << func2(10, 20);
```

### 用于回调 

function可以容纳任意符合函数签名式的可调用物，因此它非常适合代替函数指针，存储用于回调的函数，而且它的强大功能会使代码更灵活、富有弹性。

作为示范，我们定义一个demo_class类，它使用function代替函数指针作为内部类型保存回调函数，存储形式为void(int)的可调用物：

```cpp
class demo_class
{
private:
  typedef function<void(int)> func_t;             //function类型定义
  func_t func;                                    //function对象
  int n;                                          //内部成员变量
public:
  demo_class(int i):n(i){}

//demo_class使用模板函数accept()接受回调函数。之所以使用模板函数，是因为这种形式更加灵活，用户可以在不知道也不关心内部存储形式的情况下传递任何可调用对象，包括函数指针和函数对象。例如：
  template<typename CallBack>
  void accept(CallBack f)                          //存储回调函数
  {   func = f;   }

//demo_class的成员函数run()用于调用回调函数：  
  void run()
  {   func(n);    }
};                                                 //demo_class类定义结束

//接下来我们定义一个用于回调的函数，它将输入翻番：
void call_back_func(int i)
{
  cout << "call_back_func:";
  cout << i ＊ 2 << endl;
}
  
//demo_class的回调可以这样使用：
int main()
{
  demo_class dc(10);
  dc.accept(call_back_func);          //接受回调函数
  dc.run();                           //调用回调函数，输出“call_back_func:20”
}
```

使用普通的C函数进行回调并不能体现function的好处，我们来编写一个带状态的函数对象，并使用ref库传递引用：

```cpp
class call_back_obj
{
private:
  int x;                                  //内部状态
public:
  call_back_obj(int i):x(i){}
  void operator()(int i)
  {
      cout << "call_back_obj:";
      cout << i ＊ x++ << endl;           //先做乘法，然后递增
  }
};

int main()
{
  demo_class dc(10);
  call_back_obj cbo(2);
  dc.accept(ref(cbo));                    //使用ref库
  dc.run();                               //输出：call_back_obj:20
  dc.run();                               //输出：call_back_obj:30
}
```

function还可以搭配bind库，把bind表达式作为回调函数，可以接受类成员函数，或者把不符合函数签名式的函数bind为可接受的形式。下面我们定义一个回调函数工厂类，它有两个回调函数：

```cpp
class call_back_factory
{
public:
  void call_back_func1(int i)                //一个参数
  {
      cout << "call_back_factory1:";
      cout << i ＊ 2 << endl;
  }
  void call_back_func2(int i, int j)         //两个参数
  {
      cout << "call_back_factory2:";
      cout << i ＊j ＊ 2 << endl;
  }
};

//function搭配bind的用法如下：
int main()
{
  demo_class dc(10);
  call_back_factory cbf;

  dc.accept(bind(&call_back_factory::call_back_func1, cbf, _1));
  dc.run();                               //输出：call_back_factory1:20

  dc.accept(bind(&call_back_factory::call_back_func2, cbf, _1, 5));
  dc.run();                               //输出：call_back_factory2:100
}
```

通过以上的示例代码，我们可以看到function用于回调的好处，它无需改变回调的接口就可以解耦客户代码，使客户代码不必绑死在一种回调形式上，进而可以持续演化，而function始终能够保证与客户代码正确沟通。

## signals2 

signals2基于Boost的另一个库signals，实现了线程安全的观察者模式。在signals2库中，观察者模式被称为信号/插槽（signals and slots），它是一种函数回调机制，一个信号关联了多个插槽，当信号发出时，所有关联它的插槽都会被调用。

许多成熟的软件系统都用到了这种信号/插槽机制（另一个常用的名称是事件处理机制：event/event handler），它可以很好地解耦一组互相协作的类，有的语言甚至直接内建了对它的支持（如C#）, signals2以库的形式为C++增加了这个重要的功能。

signals2库位于名字空间boost::sianals2，为了使用signals2组件，需要包含头文件<boost/signals2.hpp>，即：

```cpp
#include <boost/signals2.hpp>
using namespace boost::signals2;
```

signal的模板参数列表相当长，总共有7个参数，这里仅列出了最重要的前4个，而且除了第一个是必须的外，其他的都可以使用默认值：

- 第一个模板参数Signature的含义与function的一模一样，也是一个函数类型签名，表示可被signal调用的函数（插槽、事件处理handler）。例如：
  `signal<void(int, double)>`
- 第二个模板参数Combiner是一个函数对象，它被称为“合并器”，用来组合所有插槽的调用结果，默认是optional_last_value，它使用optional库（4.3节）返回最后一个被调用的插槽的返回值；
- 第三个模板参数Group是插槽编组的类型，缺省使用int来标记组号，也可以改为std::string等类型，但通常没有必要；
- 第四个模板参数GroupCompare与Group配合使用，用来确定编组的排序准则，默认是升序（std::less），因此要求Group必须定义了operator<。
  signal继承自signal_base，而signal_base又继承自noncopyable（4.1节），因此signal是不可拷贝的，如果把signal作为自定义类的成员变量，那么自定义类也将是不可拷贝的，除非使用shared_ptr来包装它。

signal继承自signal_base，而signal_base又继承自noncopyable，因此signal是不可拷贝的，如果把signal作为自定义类的成员变量，那么自定义类也将是不可拷贝的，除非使用shared_ptr来包装它。

### 操作函数 

signal最重要的操作函数是插槽管理connect()函数，它把插槽连接到信号上，相当于为信号（事件）增加了一个处理的handler。

signal最重要的操作函数是插槽管理connect()插槽可以是任意的可调用对象，包括函数指针、函数对象，以及它们的bind表达式和function对象，signal内部使用function作为容器来保存这些可调用对象。连接时可以指定组号也可以不指定组号，当信号发生时将依据组号的排序准则依次调用插槽函数。

如果连接成功，connect()将返回一个connection对象，表示了信号与插槽之间的连接关系，它是一个轻量级的对象，可以处理两者间的连接，如断开、重连接或者测试连接状态。

成员函数disconnect()可以断开插槽与信号的连接，它有两种形式：传递组号将断开该组的所有插槽，传递一个插槽对象将仅断开该插槽。函数disconnect_all_slots()可以一次性断开信号的所有插槽连接。

当前信号所连接的插槽数量可以用num_slots()获得，成员函数empty()相当于num_slots()==0，但它的执行效率比num_slots()高。disconnect_all_slots()的后果就是令empty()返回true。

signal提供operator()，可以接受最多9个参数。当operator()被外界调用时意味着产生一个信号（事件），从而导致信号所关联的所有插槽被调用。插槽调用的结果使用合并器处理后返回，默认情况下是一个optional对象。

成员函数combiner()和set_combiner()分别用于获取和设置合并器对象，通过signal的构造函数也可以在创建的时候就传入一个合并器的实例。但通常我们可以直接使用缺省构造函数创建模板参数列表中指定的合并器对象，除非你想改用其他的合并方式。

当signal析构时，将自动断开所有插槽连接，相当于调用disconnect_all_slots()。

### 插槽的连接与调用 

signal就像是一个增强的function对象，它可以容纳（使用connect()连接）多个符合模板参数中函数签名类型的函数（插槽），形成一个插槽链表，然后在信号发生时一起调用。

例如，我们有如下两个无参的函数，它们可以被用做插槽：

```cpp
void slots1()
    { cout << "slot1 called" << endl; }
    void slots2()
    { cout << "slot2 called" << endl; }
```

除了类名字不同，signal的声明语法与function几乎一模一样：

```cppp
signal<void()>sig;              //指定插槽类型void()，其他模板参数使用缺省值
```

然后我们就可以使用connect()来连接插槽，最后用operator()来产生信号：

```cpp
int main()
{
  signal<void()>sig;             //一个信号对象

  sig.connect(&slots1);           //连接插槽1
  sig.connect(&slots2);           //连接插槽2
  sig();                          //调用operator()，产生信号（事件），触发插槽调用
}
```

在连接插槽时我们省略了connect()的第二个参数connect_position，它的缺省值是at_back，表示插槽将插入到信号插槽链表的尾部，因此slots2将在slots1之后被调用。程序的运行结果是：

```cpp
slot1 called
slot2 called
```

如果在连接slots2的时候不使用缺省参数，而是明确地传入at_front位置标志，即：

```cpp
sig.connect(&slots2, at_front);
```

那么slots2将在slots1之前被调用。

**使用组号**

connect()函数的另一个重载形式可以在连接时指定插槽所在的组号，缺省情况下组号是int类型。组号不一定要从0开始连续编号，它可以是任意的数值，离散的、负值都允许。

如果在连接的时候指定组号，那么每个编组的插槽将是又一个插槽链表，形成一个略微有些复杂的二维链表，它们的顺序规则如下：

- 各编组的调用顺序由组号从小到大决定（也可以在signal的第四个模板参数改变排序函数对象）；
- 每个编组的插槽链表内部的插入顺序用at_back和at_front指定；
- 未被编组的插槽如果位置标志是at_front，将在所有的编组之前调用；
- 未被编组的插槽如果位置标志是at_back，将在所有的编组之后调用。

我们使用一个新的函数对象slots来演示一下signal的编组，它是一个模板类：

```cpp
template<int N>
struct slots                                  //模板类，可以生成一系列的插槽
{
  void operator()()
  {   cout << "slot"<< N <<" called" << endl;   }
};
```

signal的连接代码如下：

```cpp
sig.connect(slots<1>(), at_back);             //最后被调用
sig.connect(slots<100>(), at_front);          //第一个被调用

sig.connect(5, slots<51>(), at_back);         //组号5，该组最后一个
sig.connect(5, slots<55>(), at_front);        //组号5，该组第一个

sig.connect(3, slots<30>(), at_front);        //组号3，该组第一个
sig.connect(3, slots<33>(), at_back);         //组号3，该组最后一个

sig.connect(10, slots<10>());                 //组号10，该组仅有一个
```

当执行sig()后，插槽的调用结果如下：

```cpp
slot100 called
slot30  called
slot33  called
slot55  called
slot51  called
slot10  called
slot1   called
```

### 信号的返回值 

signal如function一样，不仅可以把输入参数转发给所有插槽，也可以传回插槽的返回值。默认情况下signal使用合并器optional_last_value，它将使用optional对象返回最后被调用的插槽的返回值。

我们修改一下之前定义的slots模板类，为它的operator()增加参数和返回值：

```cpp
template<int N>
struct slots
{
  int operator()(int x)
  {
      cout << "slot"<< N <<" called" << endl;
      return x *N;
  }
};
```

signal的声明对应也需要修改：

```cpp
signal<int(int)> sig;
```

然后我们向信号连接三个插槽：

```cpp
sig.connect(slots<10>());
sig.connect(slots<20>());
sig.connect(slots<50>());
```

signal的operator()调用这时需要传入一个整数参数，这个参数会被signal存储一个拷贝，然后转发给各个插槽。最后signal将返回插槽链表末尾slots<50>()的计算结果，它是一个optional对象，必须用解引用操作符＊来获得值，即：

```cpp
cout << *sig(2);                          //输出100
```

### 合并器 

缺省的合并器optional_last_value并没有太多的意义，它通常用在我们不关心插槽返回值或者返回值是void的时候。但大多数时候，插槽的返回值都是有意义的，需要以某种方式处理多个插槽的返回值。

signal允许用户自定义合并器来处理插槽的返回值，把多个插槽的返回值合并为一个结果返回给用户。合并器应该是一个函数对象（不是函数或者函数指针），具有类似如下的形式：

```cpp
template<typename T>
class combiner    //自定义合并器
{
public:
  typedef T result_type;                      //返回值类型定义
  template<typename InputIterator>
    result_type operator()(InputIterator, InputIterator) const;
};
```

combiner类的调用操作符operator()的返回值类型可以是任意类型，完全由用户指定，不一定必须是optional或者是插槽的返回值类型。operator()的模板参数InputIterator是插槽链表的返回值迭代器，可以使用它来遍历所有插槽的返回值，进行所需的处理。

作为示范，我们编写一个自定义的合并器，它使用pair返回所有插槽的返回值之和以及其中的最大值：

```cpp
template<typename T>
class combiner
{
  T v;                                        //计算总和的初始值
  public:
    typedef std::pair<T, T> result_type;
    combiner(T t = T()):v(t){}                  //构造函数

    template<typename InputIterator>
    result_type operator()(InputIterator begin, InputIterator end) const
    {
        if (begin == end)                       //如果返回值链表为空，则返回0
        {   return result_type();      }

        vector<T> vec(begin, end);              //使用容器保存插槽调用结果

        T sum = std::accumulate(vec.begin(), vec.end(), v);
        T max = *std::max_element(vec.begin(), vec.end());
        return result_type(sum, max);
    }
};
```

使用自定义合并器的时候我们需要改写signal的声明，在模板参数列表中增加第二个模板参数——合并器类型：

```cpp
signal<int(int), combiner<int> > sig;
```

在这里我们没有向构造函数传递合并器的实例，因为signal的构造函数会缺省构造出一个实例，相当于：

```cpp
signal<int(int), combiner<int> > sig(combiner<int>());
```

插槽的连接和调用如下：

```cpp
sig.connect(slots<10>());
sig.connect(slots<20>());
sig.connect(slots<30>(), at_front);           //最大值，第一个调用

auto x = sig(2);                              //用auto获得信号的返回值
cout << x.first << ", " << x.second;          //输出120,60
```

当信号被调用时，signal会自动把解引用操作转换为插槽调用，将调用给定的合并器的operator()逐个处理插槽的返回值，并最终返回合并器operator()的结果，因此，上面的代码将输出“120,60”。

如果我们不使用signal的缺省构造函数，而是在构造signal时传入一个合并器的实例，那么signal将使用这个合并器（的拷贝）处理返回值。例如，下面的代码使用了一个有初值的合并器对象，累加值从100开始：

```cpp
signal<int(int), combiner<int> > sig(combiner<int>(100));
...
cout << x.first << ", " << x.second;         //输出220,60
```

### 应用于观察者模式 

本节我们将使用signals2开发一个完整的观察者模式示例程序，用来演示信号/插槽的用法。这个程序将

模拟一个日常生活情景：客人按门铃，门铃响，护士开门，婴儿哭闹。

首先我们要实现门铃类ring，它是本程序中的核心类，拥有一个signal对象，当按门铃时就会发出信号。

```cpp
class ring
{
public:
  typedef signal<void()> signal_t;        //内部类型定义
  typedef signal_t::slot_type slot_t;

  connection connect(const slot_t& s)     //连接插槽
  {   return alarm.connect(s);   }
  void press()                            //按门铃动作
  {
      cout << "Ring alarm..." << endl;
      alarm();                            //调用signal，发出信号，引发插槽调用
  }
private:
  signal_t alarm;                         //信号对象
};
```

我们决定采用随机数来让护士和婴儿的行为具有不确定性，这样程序会更有趣些。随机数的产生使用random库，为了方便使用我们把随机数发生器定义为全局变量：

```cpp
typedef variate_generator<rand48, uniform_smallint<> > bool_rand;
bool_rand g_rand(rand48(time(0)), uniform_smallint<>(0,100));
```

然后我们实现护士类nurse，它有一个action()函数，根据随机数决定是惊醒开门还是继续睡觉。注意：它的模板参数，使用了char const＊作为护士的名字，因此实例化时字符串必须被声明成extern:

```cpp
extern char const  nurse1[] = "Mary";
extern char const  nurse2[] = "Kate";

template<char const ＊name>
class nurse                                   //护士类
{
private:
  bool_rand &rand;                            //随机数发生器
public:
  nurse():rand(g_rand){}                      //构造函数

  void action()
  {
      cout << name;
      if (rand() > 30)                        //70%的几率惊醒
      {   cout << " wakeup and open door." << endl; }
      else                                    //30%几率继续睡觉
      {   cout << " is sleeping..." << endl; }
  }
};
```

接下来是婴儿类，它与护士类的实现差不多：

```cpp
extern char const  baby1[] = "Tom";
extern char const  baby2[] = "Jerry";

template<char const ＊name>

class baby
{
private:
  bool_rand &rand;
public:
  baby():rand(g_rand){}

  void action()
  {
      cout << "Baby " << name;
      if (rand() > 50)
      {   cout << " wakeup and crying loudly..." << endl;  }
      else
      {   cout << " is sleeping sweetly..." << endl;   }
  }
};
```

最后我们还需要一个客人类，它的唯一动作就是按门铃触发press事件：

```cpp
class guest
{
public:
  void press(ring &r)
  {
      cout << "A guest press the ring." << endl;
      r.press();
  }
};
```

程序的主要功能都完成了，现在可以把它们组合起来：

```cpp
int main()
{
  //声明门铃、护士、婴儿、客人等类的实例
  ring r;                             //门铃
  nurse<nurse1> n1;                   //护士1
  nurse<nurse2> n2;                   //护士2
  baby<baby1> b1;                     //婴儿1
  baby<baby2> b2;                     //婴儿2
  guest g;                            //访客

  //把护士、婴儿与门铃连接起来
  r.connect(bind(&nurse<nurse1>::action, n1));
  r.connect(bind(&nurse<nurse2>::action, n2));
  r.connect(bind(&baby<baby1>::action, b1));
  r.connect(bind(&baby<baby2>::action, b2));

  //客人按动门铃，触发一系列的事件
  g.press(r);
}
```

程序的运行结果可能是这样（比较有趣）：

```cpp
A guest press the ring.
Ring alarm...
Mary is sleeping...
Kate is sleeping...
Baby Tom wakeup and crying loudly...
Baby Jerry is sleeping sweetly...
```

参考链接：
《boost程序库完全开发指南》

作者： 多弗朗强哥

出处：https://www.cnblogs.com/chendeqiang/p/12941541.html

版权：本文采用「[署名-非商业性使用-相同方式共享 4.0 国际](https://creativecommons.org/licenses/by-nc-sa/4.0/)」知识共享许可协议进行许可。



分类: [C&C++](https://www.cnblogs.com/chendeqiang/category/1761487.html), [字典](https://www.cnblogs.com/chendeqiang/category/1762083.html)



Sponsor

- PayPal
- AliPay
- WeChat

1

0

支持成功

[« ](https://www.cnblogs.com/chendeqiang/p/12920146.html)上一篇： [开发经验总结](https://www.cnblogs.com/chendeqiang/p/12920146.html)
[» ](https://www.cnblogs.com/chendeqiang/p/12941964.html)下一篇： [Windows下 gcc/g++的安装与配置](https://www.cnblogs.com/chendeqiang/p/12941964.html)

posted @ 2020-05-23 11:08 [多弗朗强哥](https://www.cnblogs.com/chendeqiang/) 阅读(13) 评论(0) [编辑](https://i.cnblogs.com/EditPosts.aspx?postid=12941541) [收藏](javascript:void(0))





发表评论



编辑预览



 [退出](javascript:void(0);) [订阅评论](javascript:void(0);)

[Ctrl+Enter快捷键提交]

right © 2020 多弗朗强哥
Powered by .NET Core on Kubernetes & Theme [Silence v2.0.2](https://github.com/esofar/cnblogs-theme-silence)