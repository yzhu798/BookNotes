* 结构化绑定
```cpp
struct employee{
    unsigned id;
    std::string name;
    std::string role;
    unsigned salary;
};

int main(){
    std::vector<employee> employees{
        /* Initialized from somewhere */
    };
    
    for (const auto &[id, name, role, salary] : employees){
        std::cout << "Name: " << name
                  << "Role: " << role
                  << "Salary: " << salary << '\n';
    }
}
```
* if、switch内初始化
```cpp
if (auto itr (character_map.find(c)); itr != character_map.end()) {
  // *itr is valid. Do something with it.
} else {
  // itr is the end-iterator. Don't dereference.
}
// itr is not available here at all

switch (char c (getchar()); c) {
  case 'a': move_left(); break;
  case 's': move_back(); break;
  case 'w': move_fwd(); break;
  case 'd': move_right(); break;
  case 'q': quit_game(); break;
  case '0'...'9': select_tool('0' - c); break;
  default:
    std::cout << "invalid input: " << c << '\n';
}
```
* 移除元素
```cpp
vector<int> v{1, 2, 3, 2, 5, 2, 6, 2, 4, 8};
const auto new_end(remove(begin(v), end(v), 2)); 
v.erase(new_end, end(v));

const auto odd([](int i){return i % 2 != 0;});
v.erase(remove_if(begin(v), end(v), odd), end(v));
```
* O(1)时间移除元素（不保持原有顺序）
```cpp
template <typename T>
void quick_remove_at(std::vector<T> &v, std::size_t idx)
{
    if (idx < v.size()) {
        v[idx] = std::move(v.back()); // 把最后一个元素覆盖要删除的元素
        v.pop_back(); // 然后弹出最后一个元素
    }
}
```
* 如果容器已经有序，不打乱顺序地插入元素
```cpp
void insert_sorted(vector<string> &v, const string &word)
{
    const auto insert_pos (lower_bound(begin(v), end(v), word));
    v.insert(insert_pos, word);
}
```
* 用try_emplace为map插入元素
```cpp
#include <iostream>
#include <utility>
#include <string>
#include <map>
int main()
{
    using namespace std::literals;
    std::map<std::string, std::string> m;
 
    m.try_emplace("a", "a"s);
    m.try_emplace("b", "abcd");
    m.try_emplace("c", 10, 'c');
    m.try_emplace("c", "Won't be inserted");
 
    for (const auto &p : m) {
        std::cout << p.first << " => " << p.second << '\n';
    }
}

// output
a => a
b => abcd
c => cccccccccc
```
* 为map插入多个元素
```cpp
std::map<std::string, size_t> m { {"b", 1}, {"c", 2}, {"d", 3} };
auto insert_it (std::end(m));
for (const auto &s : {"z", "y", "x", "w"}) {
    insert_it = m.insert(insert_it, {s, 1});
}

for (const auto & [key, value] : m) {
    std::cout << "\"" << key << "\": " << value << ", ";
}
```
* 修改map的key
```cpp
auto a(m.extract(1)); // 提取key为1的元素（map中的元素被删除）
auto b(m.extract(2));
swap(a.key(), b.key());
m.insert(move(a));
m.insert(move(b));
```
* unordered_map中使用自定义类型
```cpp
#include <iostream>
#include <unordered_map> 

struct coord { // 没有哈希函数，要作为key必须实现哈希函数
    int x;
    int y;
};

bool operator==(const coord &l, const coord &r)
{
    return l.x == r.x && l.y == r.y;
}

namespace std
{
    template <>
    struct hash<coord> // 在std中为自定义类型特化一个哈希函数
    {
        using argument_type = coord;
        using result_type = size_t;
        result_type operator()(const argument_type &c) const
        { // 这个哈希算法并不好，只是一个实现示例
            return static_cast<result_type>(c.x) + static_cast<result_type>(c.y);
        }
    };
}

int main()
{
    std::unordered_map<coord, int> m { 
        { {0, 0}, 1}, 
        { {0, 1}, 2},
        { {2, 1}, 3}
    };
    for (const auto & [key, value] : m) {
        std::cout << "{(" << key.x << ", " << key.y
    			 << "): " << value << "} ";
    }
}

// output
{(0, 0): 1} {(0, 1): 2} {(2, 1): 3}
```
* 打印元素
```cpp
vector<int> v{ 1, 2, 3, 4 };
copy(begin(v), end(v), ostream_iterator<int>{cout, ", "});
```
* set获取输入流
```cpp
set<string> s;
istream_iterator<string> it {cin};
istream_iterator<string> end;
copy(it, end, inserter(s, s.end()));
```
* 并置（concatenation）函数
```cpp
#include <iostream>
#include <functional>

template <typename T, typename ...Ts>
auto concat(T t, Ts ...ts) // 如果每个参数是一个函数，假设是fgh
{
    if constexpr (sizeof...(ts) > 0) {
        return [=](auto ...parameters) {
        	return t(concat(ts...)(parameters...)); // f(g(h(参数包)))
        };
    }
    else {
   		return t;
    }
} 

int main()
{
    auto twice ([] (int i) { return i * 2; });
    auto thrice ([] (int i) { return i * 3; });
    auto combined (
    	concat(twice, thrice, std::plus<int>{}) // twice(thrice(plus))
    );
    std::cout << combined(2, 3); // 30：2 * 3 * (2 + 3)
}
```
* 容器赋值
```cpp
vector<pair<int, string>> v {
        {1, "one"}, {2, "two"}, {3, "three"},
        {4, "four"}, {5, "five"}};
map<int, string> m;
copy_n(begin(v), 3, inserter(m, begin(m))); // 把vector的三个元素拷到m
m.clear();
move(begin(v), end(v), inserter(m, begin(m))); // 把v的所有元素移动到m
```
* 消除字符串开始和结束处的空格
```cpp
string trim_whitespace_surrounding(const string &s)
{
    const char whitespace[] {" \t\n"};
    const size_t first (s.find_first_not_of(whitespace));
    if (string::npos == first) { return {}; }
    const size_t last (s.find_last_not_of(whitespace));
    return s.substr(first, (last - first + 1));
}
```
* string_view可以避免拷贝字符串，而是共享空间，起到只读的作用
```cpp
string_view sv1("abcd", 3); // abc
string_view sv2("abcd");
string s("abcd");
string_view sv3(s);
char cstr[] {'a', 'b', 'c'};
string_view sv4(cstr, sizeof(cstr));
```
* 打印不同格式的数
```cpp
#include <iostream>
#include <iomanip>

using namespace std;

class format_guard {
    decltype(cout.flags()) f{ cout.flags() };
public:
    ~format_guard() { cout.flags(f); }
};

template <typename T>
struct scientific_type {
    T value;
    explicit scientific_type(T val) : value{ val } {}
};

template <typename T>
ostream& operator<<(ostream &os, const scientific_type<T> &w) {
    format_guard _;
    os << scientific << uppercase << showpos;
    return os << w.value;
}

int main()
{
    {
        format_guard _;
        cout << hex << scientific << showbase << uppercase;

        cout << "Numbers with special formatting:\n";
        cout << 0x123abc << '\n';
        cout << 0.123456789 << '\n';
    } // 析构format_guard，清除之前的格式
    cout << "Same numbers, but normal formatting again:\n";
    cout << 0x123abc << '\n';
    cout << 0.123456789 << '\n';
    cout << "Mixed formatting: "
        << 123.0 << " "
        << scientific_type{ 123.0 } << " "
        << 123.456 << '\n';
}
```
* 用std::any打印一个任意类型
```cpp
#include <iostream>
#include <iomanip>
#include <list>
#include <any>
#include <iterator>
#include <string>

using namespace std;

using int_list = list<int>;
void print_anything(const std::any &a)
{
    if (!a.has_value()) {
        cout << "Nothing.\n";
    }
    else if (a.type() == typeid(string)) {
        cout << "It's a string: "
            << quoted(any_cast<const string&>(a)) << '\n';
    }
    else if (a.type() == typeid(int)) {
        cout << "It's an integer: "
            << any_cast<int>(a) << '\n';
    }
    else if (a.type() == typeid(int_list)) {
        const auto &l(any_cast<const int_list&>(a));

        cout << "It's a list: ";
        copy(begin(l), end(l),
            ostream_iterator<int>{cout, ", "});
        cout << '\n';
    }
    else {
        cout << "Can't handle this item.\n";
    }
}

int main()
{
    print_anything({});
    print_anything("abc"s);
    print_anything(123);
    print_anything(int_list{ 1, 2, 3 });
    print_anything(any(in_place_type_t<int_list>{}, { 1, 2, 3 }));
}

// output
Nothing.
It's a string: "abc"
It's an integer: 123
It's a list: 1, 2, 3,
It's a list: 1, 2, 3,
```
* 用std::variant存储不同类型
```cpp
#include <iostream>
#include <variant>
#include <list>
#include <string>
#include <algorithm>

using namespace std;

class cat {
    string name;
public:
    cat(string n) : name{ n } {}
    void meow() const {
        cout << name << " says Meow!\n";
    }
};

class dog {
    string name;
public:
    dog(string n) : name{ n } {}
    void woof() const {
        cout << name << " says Woof!\n";
    }
};

using animal = variant<dog, cat>;

template <typename T>
bool is_type(const animal &a) {
    return holds_alternative<T>(a);
}

struct animal_voice
{
    void operator()(const dog &d) const { d.woof(); }
    void operator()(const cat &c) const { c.meow(); }
};

int main()
{
    list<animal> l{ cat{"Tuba"}, dog{"Balou"}, cat{"Bobby"} };
    for (const animal &a : l) {
        switch (a.index()) {
        case 0:
            get<dog>(a).woof();
            break;
        case 1:
            get<cat>(a).meow();
            break;
        }
    }
    cout << "-----\n";
    for (const animal &a : l) {
        if (const auto d(get_if<dog>(&a)); d) {
            d->woof();
        }
        else if (const auto c(get_if<cat>(&a)); c) {
            c->meow();
        }
    }
    cout << "-----\n";
    for (const animal &a : l) {
        visit(animal_voice{}, a);
    }
    cout << "-----\n";
    cout << "There are "
        << count_if(begin(l), end(l), is_type<cat>)
        << " cats and "
        << count_if(begin(l), end(l), is_type<dog>)
        << " dogs in the list.\n";
}

// output
Tuba says Meow!
Balou says Woof!
Bobby says Meow!
-----
Tuba says Meow!
Balou says Woof!
Bobby says Meow!
-----
Tuba says Meow!
Balou says Woof!
Bobby says Meow!
-----
There are 2 cats and 1 dogs in the list.
```
* 并行计算五千万个随机生成数中的奇数个数占比
```cpp
#include <iostream>
#include <vector>
#include <random>
#include <algorithm>
#include <execution>

using namespace std;

static bool odd(int n) { return n % 2; }

int main()
{
    vector<int> d(50000000);
    mt19937 gen;
    uniform_int_distribution<int> dis(0, 100000);
    auto rand_num([=]() mutable { return dis(gen); });
    // 这里只使用了par策略，也可以使用seq和par_vec
    generate(execution::par, begin(d), end(d), rand_num); // 填满vector
    sort(execution::par, begin(d), end(d));
    reverse(execution::par, begin(d), end(d));
    auto odds(count_if(execution::par, begin(d), end(d), odd));
    cout << (100.0 * odds / d.size()) << "% of the numbers are odd.\n";
}
```
* 休眠
```cpp
#include <iostream>
#include <chrono>
#include <thread>

using namespace std;
using namespace chrono_literals;

int main()
{
    cout << "Going to sleep for 5 seconds"
        " and 300 milli seconds.\n";

    this_thread::sleep_for(5s + 300ms); // 休眠5秒300毫秒
    cout << "Going to sleep for another 3 seconds.\n";

    this_thread::sleep_until(
        chrono::high_resolution_clock::now() + 3s);
    cout << "That's it.\n";
}
```
* 打印绝对路径
```cpp
#include <iostream>
#include <filesystem>

using namespace std;
using namespace filesystem;

int main(int argc, char *argv[])
{
    if (argc != 2) {
        cout << "Usage: " << argv[0] << " <path>\n";
        return 1;
    }
    
    const path dir {argv[1]};
        if (!exists(dir)) {
        cout << "Path " << dir << " does not exist.\n";
        return 1;
    }    
    cout << canonical(dir).c_str() << '\n';
}

// 运行
$ ./normalizer src
// output
/Users/tfc/src
```
