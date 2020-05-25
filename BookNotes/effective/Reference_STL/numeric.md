## accumulate
* 累加指定范围内的元素
```cpp
template <class InputIterator, class T>
     T accumulate (InputIterator first, InputIterator last, T init)
{
    while (first!=last) {
        init = init + *first;    // or: init=binary_op(init,*first) for the binary_op version
        ++first;
    }
    return init;
}
// accumulate example
#include <iostream>         // std::cout
#include <functional>     // std::minus
#include <numeric>            // std::accumulate

int myfunction (int x, int y) {return x+2*y;}
struct myclass {
	int operator()(int x, int y) {return x+3*y;}
} myobject;

int main () {
    int init = 100;
    int numbers[] = {10,20,30};

    std::cout << "using default accumulate: ";
    std::cout << std::accumulate(numbers,numbers+3,init);
    std::cout << '\n';

    std::cout << "using functional's minus: ";
    std::cout << std::accumulate (numbers, numbers+3, init, std::minus<int>());
    std::cout << '\n';

    std::cout << "using custom function: ";
    std::cout << std::accumulate (numbers, numbers+3, init, myfunction);
    std::cout << '\n';

    std::cout << "using custom object: ";
    std::cout << std::accumulate (numbers, numbers+3, init, myobject);
    std::cout << '\n';

    return 0;
}
// output
using default accumulate: 160
using functional's minus: 40
using custom function: 220
using custom object: 280
```

## adjacent_difference
* 计算范围中相邻元素的差
```cpp
template <class InputIterator, class OutputIterator>
     OutputIterator adjacent_difference (InputIterator first, InputIterator last,
                                                                             OutputIterator result)
{
    if (first!=last) {
        typename iterator_traits<InputIterator>::value_type val,prev;
        *result = prev = *first;
        while (++first!=last) {
            val = *first;
            *++result = val - prev;    // or: *++result = binary_op(val,prev)
            prev = val;
        }
        ++result;
    }
    return result;
}
// adjacent_difference example
#include <iostream>         // std::cout
#include <functional>     // std::multiplies
#include <numeric>            // std::adjacent_difference

int myop (int x, int y) {return x+y;}

int main () {
    int val[] = {1,2,3,5,9,11,12};
    int result[7];

    std::adjacent_difference (val, val+7, result);
    std::cout << "using default adjacent_difference: ";
    for (int i=0; i<7; i++) std::cout << result[i] << ' ';
    std::cout << '\n';

    std::adjacent_difference (val, val+7, result, std::multiplies<int>());
    std::cout << "using functional operation multiplies: ";
    for (int i=0; i<7; i++) std::cout << result[i] << ' ';
    std::cout << '\n';

    std::adjacent_difference (val, val+7, result, myop);
    std::cout << "using custom function: ";
    for (int i=0; i<7; i++) std::cout << result[i] << ' ';
    std::cout << '\n';
    return 0;
}
// output
using default adjacent_difference: 1 1 1 2 4 2 1
using functional operation multiplies: 1 2 6 15 45 99 132
using custom function: 1 3 5 8 14 20 23
```
## inner_product
* 计算两个范围的内积
```cpp
template <class InputIterator1, class InputIterator2, class T>
     T inner_product (InputIterator1 first1, InputIterator1 last1,
                                        InputIterator2 first2, T init)
{
    while (first1!=last1) {
        init = init + (*first1)*(*first2);
                             // or: init = binary_op1 (init, binary_op2(*first1,*first2));
        ++first1; ++first2;
    }
    return init;
}
// inner_product example
#include <iostream>         // std::cout
#include <functional>     // std::minus, std::divides
#include <numeric>            // std::inner_product

int myaccumulator (int x, int y) {return x-y;}
int myproduct (int x, int y) {return x+y;}

int main () {
    int init = 100;
    int series1[] = {10,20,30};
    int series2[] = {1,2,3};

    std::cout << "using default inner_product: ";
    std::cout << std::inner_product(series1,series1+3,series2,init);
    std::cout << '\n';

    std::cout << "using functional operations: ";
    std::cout << std::inner_product(series1,series1+3,series2,init,
                                                                    std::minus<int>(),std::divides<int>());
    std::cout << '\n';

    std::cout << "using custom functions: ";
    std::cout << std::inner_product(series1,series1+3,series2,init,
                                                                    myaccumulator,myproduct);
    std::cout << '\n';

    return 0;
}
// output
using default inner_product: 240
using functional operations: 70 // 100-10/1-20/2-30/3
using custom functions: 34 // 100-(10+1)-(20+2)-(30+3)
```
## partial_sum
* 计算范围中的元素的部分和
```cpp
template <class InputIterator, class OutputIterator>
     OutputIterator partial_sum (InputIterator first, InputIterator last,
                                                             OutputIterator result)
{
    if (first!=last) {
        typename iterator_traits<InputIterator>::value_type val = *first;
        *result = val;
        while (++first!=last) {
            val = val + *first;     // or: val = binary_op(val,*first)
            *++result = val;
        }
        ++result;
    }
    return result;
}
// partial_sum example
#include <iostream>         // std::cout
#include <functional>     // std::multiplies
#include <numeric>            // std::partial_sum

int myop (int x, int y) {return x+y+1;}

int main () {
    int val[] = {1,2,3,4,5};
    int result[5];

    std::partial_sum (val, val+5, result);
    std::cout << "using default partial_sum: ";
    for (int i=0; i<5; i++) std::cout << result[i] << ' ';
    std::cout << '\n';

    std::partial_sum (val, val+5, result, std::multiplies<int>());
    std::cout << "using functional operation multiplies: ";
    for (int i=0; i<5; i++) std::cout << result[i] << ' ';
    std::cout << '\n';

    std::partial_sum (val, val+5, result, myop);
    std::cout << "using custom function: ";
    for (int i=0; i<5; i++) std::cout << result[i] << ' ';
    std::cout << '\n';
    return 0;
}
// output
using default partial_sum: 1 3 6 10 15
using functional operation multiplies: 1 2 6 24 120
using custom function: 1 4 8 13 19
```
## iota
* 用顺序递增的值赋值指定范围
```cpp
template <class ForwardIterator, class T>
void iota (ForwardIterator first, ForwardIterator last, T val)
{
    while (first!=last) {
        *first = val;
        ++first;
        ++val;
    }
}
// iota example
#include <iostream>         // std::cout
#include <numeric>            // std::iota

int main () {
    int numbers[10];

    std::iota (numbers,numbers+10,100);

    std::cout << "numbers:";
    for (int& i:numbers) std::cout << ' ' << i;
    std::cout << '\n';

    return 0;
}
// output
numbers: 100 101 102 103 104 105 106 107 108 109
```
