* 查找成功时返回所在位置，失败返回string::npos

## find
* 查找目标字符串第一次出现的位置
```cpp
// string
size_t find (const string& str, size_t pos = 0) const noexcept;
// c-string
size_t find (const char* s, size_t pos = 0) const;
// buffer
// 查找s的前n个字符
size_t find (const char* s, size_t pos, size_type n) const;
// character
size_t find (char c, size_t pos = 0) const noexcept;
```
* 例子
```cpp
string str ("There are two needles in this haystack with needles.");
string str2 ("needle");
size_t found = str.find(str2); // 第一个needle的位置
if (found != string::npos) cout << found << endl;
found = str.find("needles are small",found+1,6); // 第二个
str.replace(str.find(str2), str2.length(), "preposition"); // 替代第一个needle
```

## rfind
* 从后往前查找目标字符串第一次出现的位置
```cpp
// string
size_t rfind (const string& str, size_t pos = npos) const noexcept;
// c-string
size_t rfind (const char* s, size_t pos = npos) const;
// buffer
size_t rfind (const char* s, size_t pos, size_t n) const;
// character
size_t rfind (char c, size_t pos = npos) const noexcept;
```
* 例子
```cpp
string str ("The sixth sick sheik's sixth sheep's sick.");
string key ("sixth");
size_t found = str.rfind(key);
if (found!=std::string::npos) str.replace (found, key.length(), "seventh");
```

## find_first_of
* 查找目标字符串的任意字符第一次出现的位置
```cpp
// string
size_t find_first_of (const string& str, size_t pos = 0) const noexcept;
// c-string
size_t find_first_of (const char* s, size_t pos = 0) const;
// buffer
size_t find_first_of (const char* s, size_t pos, size_t n) const;
// character
size_t find_first_of (char c, size_t pos = 0) const noexcept;
```
* 例子
```cpp
// 把字符串中的所有元音字母替换为*
string str ("Please, replace the vowels in this sentence by asterisks.");
size_t found = str.find_first_of("aeiou");
while (found != string::npos)
{
    str[found]='*';
    found=str.find_first_of("aeiou",found+1);
}
```

## find_last_of
* 从后往前查找目标字符串的任意字符第一次出现的位置
```cpp
// string
size_t find_last_of (const string& str, size_t pos = npos) const noexcept;
// c-string
size_t find_last_of (const char* s, size_t pos = npos) const;
// buffer
size_t find_last_of (const char* s, size_t pos, size_t n) const;
// character
size_t find_last_of (char c, size_t pos = npos) const noexcept;
```
* 例子
```cpp
#include <iostream>
#include <string>
#include <cstddef> // std::size_t

using namespace std;

// 分解文件夹路径
void SplitFilename (const std::string& str)
{
    cout << "Splitting: " << str << '\n';
    size_t found = str.find_last_of("/\\"); // 找到最后一个/或\的位置
    // string substr (size_t pos = 0, size_t len = npos) const;
    cout << " path: " << str.substr(0,found) << '\n';
    cout << " file: " << str.substr(found+1) << '\n';
}

int main()
{
    string str1 ("/usr/bin/man");
    string str2 ("c:\\windows\\winhelp.exe");
    SplitFilename (str1);
    SplitFilename (str2);
    return 0;
}

// output
Splitting: /usr/bin/man
path: /usr/bin
file: man
Splitting: c:\windows\winhelp.exe
path: c:\windows
file: winhelp.exe
```

## find_first_not_of
* 查找目标字符串的任意字符第一次未出现的位置
```cpp
// string
size_t find_first_not_of (const string& str, size_t pos = 0) const noexcept;
// c-string
size_t find_first_not_of (const char* s, size_t pos = 0) const;
// buffer
size_t find_first_not_of (const char* s, size_t pos, size_t n) const;
// character
size_t find_first_not_of (char c, size_t pos = 0) const noexcept;
```
* 例子
```cpp
string str ("look for non-alphabetic characters...");
size_t found = str.find_first_not_of("abcdefghijklmnopqrstuvwxyz "); // -的位置
```

## find_last_not_of
* 从后往前查找目标字符串的任意字符第一次未出现的位置
```cpp
// string
size_t find_last_not_of (const string& str, size_t pos = npos) const noexcept;
// c-string
size_t find_last_not_of (const char* s, size_t pos = npos) const;
// buffer
size_t find_last_not_of (const char* s, size_t pos, size_t n) const;
// character
size_t find_last_not_of (char c, size_t pos = npos) const noexcept;
```
* 例子
```cpp
string str ("hello   \n");
string str2 (" \t\f\v\n\r");
size_t found = str.find_last_not_of(str2); // o的位置
// string& erase (size_t pos = 0, size_t n = npos);
// iterator erase (const_iterator p);
// iterator erase (const_iterator first, const_iterator last);
if (found != string::npos) str.erase(found+1); // 删除o之后的内容
else str.clear();
cout << str; // hello
```
