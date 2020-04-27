
 ## string

getline vs cin:
- >> 输入运算符，每次读取一个词，不读取分隔符(空格、换行、制表)
- getline，每次读取一行，读入空格和换行符，但换行符不写入string对象

```cpp
#include <string>
using std::string;

string line;
while (getline(cin, line))
    if (!line.empty())
        std::cout << line std::endl;

string word;
while (cin >> word)
    std::cout << word << std::endl;
```
string::size_type类型： long unsigned int 的类型别名
- 机器无关特性，足够存放任何string对象的大小
- s.size() 的返回类型为 size_type
- 若n为负数， while(s.size() < n) 将会是死循环


string相加(concat)： 运算符两侧至少得有一个是string对象，否则编译报错
- decltype(字符串字面值) -> const char (&)[size]
- 字符串字面值和字符字面值可以在与string对象连接过程中，自动转化为string对象 
- 运算结果为string对象

```cpp
string s6 = s1  + "," + "world"; // ok 
string s7 = "hello" + "," + s2; // error
```

<cctype>头文件提供的字符处理函数:
- isalpha(c)、isdigit(c)、isxdigit(c)十六进制、isalnum(c) 数字或字母
- iscntrl(c)、isprint(c)、isgraph(c)不是空格但可打印
- ispunct(c)，标点符号，除开控制字符、数字、字母、可打印空白
- isspace(c)，空白符，包含制表、回车、换行、进纸符
- isupper(c)、islower(c)
- toupper(c)、tolower(c)，若为真，原样输出，假则大小写转换

```cpp
decltype(s.size()) punct_cnt = 0;
for (auto c : s) // 每次循环，将s的一个字符赋值给c，c的类型为 char
    if (ispunt(c))
        ++ punct_cnt;

for (auto &c : s) // c的类型为 char &，若 s为const，则c为const char &
    c = toupper(c); 

if (!s.empty()) // 使用[]随机访问前，检查访问位置是否有效
    s[0] = toupper(s[0]); 

 // 将第一个单词变为大写
for (decltype(s.size()) index = 0;
        index != s.size() && !isspace(s[index]); ++ index)
            s[index]  = toupper(s[index]);
```

使用建议：
- c++标准并不要求标准库检测下标是否合法，使用operator[]要留心是否发非法访存
  
## vector

容器： 容纳对象的集合，每个索引对应一个对象或容器

vector：可变长的容器，是一个模板类

模板： 编译器生成类或函数编写的一份说明

实例化： 编译器根据模板创建类或函数的过程。使用模板时，需要指出编译器应把类或函数实例化为何种类型

缓冲区溢出： 原因是试图通过一个越界的索引访问内容 ?? 写行为？

索引： 下标运算符使用的值

迭代器： 是一种类型，用于访问容器中的元素或在元素之间移动

指针运算： 指针类型支持的算术运算

拷贝初始化： 新创建的对象是初始值的一个副本

值初始化： 编译器使用默认值进行初始化

using声明： 令命名空间中的某个名字可被程序直接使用

编译器扩展： 某个特性为编译器为c++语言额外添加的特性。基于编译器特性写的程序不容易移植到其他编译器

初始化：
- 同类对象拷贝初始化
- 列表初始化
- 直接初始化

```cpp
vector<string> svec(10);  // 需要有默认构造函数
vector<string> svec(10, "hi");  // 10个 "hi"
vector<string> svec {10, "hi"}; // 列表初始化失败，转为直接初始化
vector<int> vec {10, 1}; // 2个元素
```

添加元素与下标运算符：
- push_back： 往vector里添加元素，适用于事先并不清楚所需元素个数或元素个数过多不便于使用列表初始化的情况
- operator[]： 若元素对象不是常量，则可以对其进行更新赋值，但无法往vector里添加元素对象
- vector<int>::size_type： vec.size()的返回值类型

```cpp
// 统计各个分数段的人数
vector<unsigned> scores(11,0);
unsigned grade;
while (cin >> grade) {
    if (grade <=  100)
        ++ scores[grade/10];
}
```

迭代器： 类似指针，提供对vector对象的元素的间接访问
- 认定某个类型是迭代器，当且仅当它支持一套操作，这套操作使得我们能访问容器的元素或者从某个元素移动到另外一个元素
- 迭代器分有效和无效，类似指针。当指向某个元素或尾元素的下一位置，迭代器有效，其他情况，均无效
- 通过调用返回迭代器的成员函数来获得迭代器，begin返回指向第一个元素的迭代器，end返回尾元素下一位置的迭代器
- 当容器为空时，begin和end返回的是同一个迭代器

标准容器迭代器的运算符：
- *iter 返回iter所指元素的引用，不能解引用尾后迭代器和无效迭代器，因其不指向任何元素
- iter->member <==> (*iter).member 箭头运算符，解引用并访问member的成员
- ++iter,-- iter 迭代器移动到下一个或上一个元素
- iter1 == iter2
  - 当两个迭代器指向同一个元素
  - 当两个迭代器指向同一容器的尾后迭代器
- iter1 != iter2 
  - 除上述的情况外
  
```cpp
string s("some string");
for (auto it = s.begin(); it != s.end() && !isspace(*it); ++it)
    *it = toupper(*it);
```

使用建议：
- 对迭代器使用相等判断，因为所有标准库容器都定义了==和!=，但是它们大多数都没有定义<运算符
  
迭代器的类型： 拥有迭代器的标准库类型使用iterator和const iterator来表示迭代器的类型
- 默认地，若对象是常量对象，则返回const iterator类型，否则返回iterator
- 使用cbegin和cend可以指定返回const iterator，适用于不修改对象的情况

```cpp
// 输出文本文件，直到遇到空字符串的空行
vector<string> text;
for (auto it = text.cbegin(); 
        it != text.cend() && !it->empty(); ++it)
            cout << *it << endl;
```

迭代器失效：
- 在for循环中，对vector添加和删除元素，引发的realloc，导致it迭代器失效，it!=end()永为真
- 由于vector的动态内存变化的机制，在插入和删除时，需要考虑迭代的是否失效的问题。

```cpp
vector<int> vec {1,2,3,4};
for (auto it = v.begin(); it != v.end(); ++it) { // it != v.end() 永为真
    if (it == v.begin()) 
        v.push_back(5);
}
```

vector的额外迭代器运算(iterator arithmetic)：
- 算术运算
  - iter + n, iter - n  跳跃到容器内的另一个元素或尾元素的下一个位置
  - iter1 - iter2  同一容器中两个迭代器的距离，类型为difference_type(带符号)
- 比较运算
  - <、<=  判断迭代器位置的前后
  - >、>=

迭代器的距离： iter1 - iter2, 指的是右侧的迭代器向前移动多少位置就能追上左侧的迭代器

```cpp
// 二分查找
auto beg = text.begin(), end = text.end();
auto mid = text.begin() + (end - beg) / 2;
while (mid != end && *mid != target) {
    if (target < *mid)
        end = mid;
    else
        beg = mid + 1;
    mid = beg + (end - beg) / 2; // 迭代器没有operator+，所以不能是 mid = (beg+end)/2
}
```

## arrary

array和指针： 语言内置的容器和迭代器， 类比标准库的vector和迭代器

array vs vector:
- 同
  - 存放匿名对象的集合
  - 本身是对象，可以定义数组的指针和数组的引用
- 异
  - array 编译时确定大小， vector 动态大小(灵活)
  - array 标准不支持同类对象拷贝赋值， vector支持同类对象拷贝
  - array 不能用auto推导定义，与初始化列表类型有歧义

初始化：
- 定义在全局作用域数组，自动初始化为默认值。定义在块作用域，则未初始化 ?? 编译器扩展？？
- 字符数组可以使用字符串字面值进行初始化

```cpp
constexpr size_t size() { return 5; }
int arr[size()] = {1,2,3}; // {1, 2, 3, 0, 0}
char str[] = "hello"; // {'h', 'e', 'l', 'l', 'o', '\0'}

int *ptr[10]; // 定义了一个名为ptr的数组，存放 int* 类型的对象元素
int (*arrPtr)[10] = &arr; // 定义了一个名为arrPtr的指针， 指向一个int[10]的数组
int (&arrRef)[10] = arr;  //定义了一个名为arrRef的引用， 绑定到一个int[10]的数组
int *(&arrRef)[10]; // 定义了一个名为arrRef的引用， 绑定到一个存放 int*[10]数组
```

遍历：
- 使用的数组下标，通常定义为size_t类型。size_t是与机器相关的无符号类型，它足够大能表示任意对象的大小
- 数组的下标运算符由语言定义，作用在数组类型的对象上

```cpp
#include <cstddef> // size_t类型

unsigned scores[11] = { }; // 全初始化为0
unsigned grade;
while (cin >> grade) {
    if (grade <= 100)
        ++ scores[grade/10];
}
```

指针与数组： 数组不是类型，编译器将数组的操作转化为指针的运算
- 数组名 <==> 对数组首元素取地址
- 数组名是常量指针，无法对其进行++和--
- 数组的下标运算 <==> 指针加减整数的运算
- 数组的下标运算符可以是负数，标准库则限制必须是无符号数
- decltype(arr)，结果为数组类型，非指针类型
  
```cpp
int arr[]= {1,2,3};
auto p(arr); // p = &arr[0]
++ p;
cout << p[-1] << endl; // *(p - 1)
decltype(arr) arr2; // decltype(arr) -> int [3]
```

指针与迭代器： 指针也是迭代器，vector迭代器的操作，数组的指针全支持
- 指针相减，结果为ptrdiff_t类型，表示距离。ptrdiff_t定义在<cstddef>中的机器实现有关类型，带符号
- 指向非同一容器或非同一对象的指针进行关系运算，结果未定义。块作用域定义的变量，栈向低地址方向生长
- 指针指向同一容器的含义包含指向容器内的某个元素和指向尾元素的下一个位置
- 空指针和指向同一变量的指针，也支持 指针相减、指针加减整数、指针关系比较运算，但意义不大
- 非成员函数的begin()和end()定义在<iterator>，支持以迭代器风格遍历数组

```cpp
int arr[3] = {1,2,3};
auto size = end(arr) - begin(arr); // &arr[3] - &arr[0]

// 方式一，使用函数获取起始和结束位置
for (int *beg = begin(arr); beg != end(arr); ++beg)
    cout << *beg << endl;

// 方式二，定义迭代器的起始和结束位置
int *end = arr + size;
for (int *begin = arr; begin != end; ++begin)
    cout << *begin << endl;
```

C风格字符串/字符数组： 以空字符'\0'结束的字符数组
- 字符串字面值类型为 const char*， decltype(字符串字面值)结果为 const char (&)[size+1]
- 可以以指针名的方式输出整个字符数组
- 作用于C风格字符串的函数
  - strlen(p)
  - strcmp(p1,p2) 返回值{-1,0,1}
  - strcpy(p1,p2) 返回p1
  - strcat(p1,p2) 返回p1
  - 函数不负责检查函数参数，使用strcpy和strcat需保证p1和p2均以'\0'结束，并且p1有足够的内存空间
- 不能直接将两个字符指针进行比较和连接，须调用函数完成
  
```cpp
#include <cstring>

char s1[3] = "a";
const char s2[] = "b";

s1 + s2; // error
strcat(s1, s2); //ok

s1 < s2; // error
strcmp(s1, s2); //ok
```

与旧代码的接口： 为兼容c的代码(字符串和数组)，string和vector向后兼容
- string
  - 允许使用C风格的字符数组
    - 初始化string对象
    - 与string对象进行连接
    - 赋值给string对象
  - c_cstr()返回一个C风格的字符数组
    - 若string对象发生改变，无法保证c_str数组一直有效
- vector
  - 允许使用数组初始化vector对象
  - data()返回一个T*或const T*的数组
  
```cpp
string s("a");
const char *str = s.c_str(); // 不改变s的内容

int arr[] = {1,2,3};
vector<int> v1(begin(arr), end(arr));
vector<int> v2(arr + 1, arr + size);
int *p = v2.data(); // {2,3}
```
  
使用建议：
- 使用vector和迭代器，而不是数组和指针，更为安全
- string为可变长字符序列，替代字符数组
- vector为同类型对象的容器，替代数组
- 迭代器访问元素或在元素间移动，替代指针
- 数组和指针是工作在较低的层次，vector和string基于它们进行抽象封装.

多维数组： 数组的元素又是数组
- 列表初始化，全初始化值可以不用嵌套花括号，列初始化需要嵌套花括号
- 编译器会默认将数组名当作指针处理，导致内层嵌套循环在指针上而非数组，而出错
- auto&或const auto&类型推导对嵌套数组的推导结果为数组类型，非指针
- 除了最里层的数组循环，其他层数组都循环应该声明为auto& 或 const auto&
- 如果表达式含有的下表运算符数量比数组的维度小，则表达式的结果是给定索引位置的数组

```cpp
int arr[3][4] = { }; // 大小为3的数组，每个元素是含有4个整数的数组
int arr[3][4] = {{0}, {4}, {8}}; // 每行第一个元素初始化(列初始化)
int arr[3][4] = {0, 3, 6, 9}; // 第一行元素初始化

int (&row)[4] = arr[1]; // 定义名为row的数组引用，绑定到arr[1]数组
int (*p)[4] = arr;

// 方式一
for (const auto & i : arr) { // const 仅对一维数组起作用
    for (const auto j : i) // const 对数组内元素起作用
        std::cout << j;
}

// 方式二
for (auto p = arr; p != arr + 3; ++p) {
    for (auto q = *p; q != *p + 4; ++q)
        std::cout << *q;
}

// 方式三
for (auto p = std::begin(arr); p != std::end(arr); ++p) {
    for (auto q = std::begin(*p); q != std::end(*p); ++q)
        std::cout << *q; 
}

// 不使用 auto
for (int (&p)[4] : arr) {
	for (int x : p)
		std::cout << x;
}

for (int (*p)[4] = arr; p != arr + 3; ++p) { // 定义p指向int[4]
	for (int *q = *p; q != *p + 4; ++q) // 定义q指向p所指向的数组内的元素
		std::cout << *q;
}

for (int (*p)[4] = std::begin(arr); p != std::end(arr); ++p) {
	for (int *q = std::begin(*p); q != std::end(*p); ++q)
		std::cout << *q;
}
```

