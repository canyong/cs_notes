
## 基础概念

主调函数(calling)与被调函数(called)

调用函数步骤：
- 实参初始化函数的形参
- 将控制权转移给被调用函数

函数执行结束步骤：
- 返回return语句中的值
- 将控制权转移从给调函数
  
函数形参列表： 函数调用时，提供的实参的类型和个数须与形参表匹配，用于初始化形参

函数返回值： 无法返回数组类型或函数类型，只能返回指针，指向数组或函数的指针

尾置返回值类型：在参数列表后面指定的返回类型

调用运算符()：用于执行某函数，括号前是函数名或函数指针，括号内是以逗号隔开的实参列表

函数体： 是一个块，用于定义函数所执行的操作

函数定义的局部对象：
- 函数内定义的变量成为局部变量
- 自动对象(automatic object)，作用域函数内，生命周期从定义处到块作用域末尾
- 静态对象(local static object)，作用域函数内，生命周期从定义出到程序结束，一次定义
- 形参是一种自动对象，函数开始时为形参申请存储空间，作用域函数内，生命周期到函数的末尾
- 局部对象会隐藏外部作用域的同名对象声明
  
函数声明： 
- 函数声明没有函数体，描述了函数的接口(函数原型prototype)，说明了调用函数所需的全部信息
- 函数声明存放于头文件中，如果需要改变接口，只需在头文件中改
- 定义函数的源文件将头文件包含，编译器负责验证函数的定义与声明是否匹配

分离式编译：
- 将程序分隔到几个文件中，每个文件可以独立编译(允许编译程序的一部分)
- 当修改一个源文件，只需重新编译改动了的文件

参数传递： 形参和实参的交互方式/函数调用方式
- 值传递(pass by value) / 传值调用(called by value)
- 指针传递，形参拷贝实参的值，函数内改变形参指针的指向不影响实参指针，属值传递
  - 因指针自身特性，可以间接访问所指对象的值，因此可以修改指向的对象的值
- 引用传递和传引用调用，形参绑定到实参的别名。可以使用引用形参来实现多返回值
  - 传递大对象或者传递对象不支持拷贝操作的情况，将形参定义为引用
  - 如果函数无需改变引用形参值，将形参定义为常量引用
  - 若接受字面值或const的实参，也需要将形参声明为常量引用
- 顶层const指针或引用形参，接受non-const的实参，因为顶层const会被忽略
  
```cpp
size_type find_char(string &s, char c);
bool is_sentence(const string &s)  {
    return find_char(s, '.'); // const无法转non-const
}

// 修改1
size_type find_char(const string &s, char c);

// 修改2
bool is_sentence(const string &s) {
    string str(s);
    return find_char(str, '.');
}
```

数组形参：
- 不能拷贝整个数组。数组名转为指针，以指针的形式传递给数组。
- 若const数组，则数组名为const T*，否则为T*
- 形参可声明为数组形式，如正常定义数组的语句的形式。指定数组大小的形参声明容易出错
- 传递数组实参时，有三种方式来防止数组形参被越界访问
  - 使用数组里的结束标记，如字符串数组的'\0'
  - 传递数组的开始和结束地址，如begin(arr), end(arr)
  - 传递数组的空间大小或元素数量，如size
- 只有当函数确实需要数组的元素值时，才把形参定义为非常量的指针

```cpp
void print(const int *)
void print(const int []) // 形参数组同实参数组大小
void print(const int [10]) // 形参数组大小为10，传入的实参大小不确定 

// 管理指针形参的三种技术
void print(const char *cp);
void print(const int *beg, const int *end);
void print(const int arr[], size_t sz);

void print(const int (&arr)[10]); // 数组的引用，只接受int[10]，数组大小是数组类型的一部分
void print(int (*matrix)[10], int row_size); // 形参为二维数组
void print(int martrix[][10], int row_size); // error，编译器忽略第一维度，形参为指向int[10]的指针
```
  
## main函数命令行选项

int main(int argc, char**argv) { ... }

prog -d -o ofile data0

argv[0] = "prog";
argv[1] = "-d";
argv[2] = "-o";
argv[3] = "ofile";
argv[4] = "data0";
argv[5] = 0;

## 函数返回值

- 在含有return语句的循环后面应该加一条return语句
- 返回的值用于初始化调用点的一个临时量，该临时量的类型由函数返回类型定义，如果返回引用，则不会产生拷贝
- 不该返回局部对象。返回引用，需确定引用的是在函数之前的哪个对象
- 可能出现的返回值
  - 返回类对象和调用运算符，可以使用函数调用的结果访问类对象的成员
  - 返回引用，返回非常量引用可以作为赋值运算符左侧运算对象
  - 返回初始化列表
  - 返回数组指针，
- main函数的返回值，返回0表示执行成功正常退出，其他非0值具体含义由机器而定
```cpp
bool func() {
    for (decltype(size) i = 0; i != size; ++i) {
        if (str1[i] != str2[i])
            return false;
    }
    return true; // 缺少则程序错误
}|

// 返回类对象
auto sz = shorter_str(s1,s2).size(); // 左结合

// 返回初始化列表，用返回值初始化调用点的vector<string>临时对象
vector<string> process() {
    return {"functionX", "okay"};
}

// main函数返回值
int main() {
    if (some_failure)
        return EXIT_FAILURE; // 定义在 <cstdlib> 头文件
    else
        return EXIT_SUCCESS;
}

// 返回数组指针
auto func(int i) -> int(*)[10];

int odd[] = {1,3,5};
int even[] = {2,4,6};
decltype(odd)* arrPtr(int i) { // 返回指向odd类型数组(int [3])的指针
    return (i % 2) ? &odd : &even;
}
```

## 函数重载

重载函数(overloaded function)： 
- 同一作用域内，有几个函数名字相同但形参列表不同，执行的操作非常类似
- 重载函数应该在形参类型或形参数量上有所不同，变量名只起辅助记忆作用，不影响形参表
- 当传递一个非常量对象或非常量对象的指针时，编译器会优先选用非常量版本的函数
- 是否重载函数，还是为操作相似的函数起多个名字，取决于哪个更容易见名知意
```cpp
// 形参表相同非相异
Record Lookup(const Phone&);
Record Lookup(const Telno&);
```
  
const_cast与重载: // 意义何在?? 不修改实参的值, non-const转const
```cpp
string& shorter_str(string &s1, string &s2) {
    auto &r = shorter_str(const_cast<const string&>(s1),
                          const_cast<const string&>(s2));
    return const_cast<string&>(r);
}
```

函数匹配(function matching):
- 把函数调用与重载函数中的某一个关联起来(overload resolution)
- 编译器将调用的实参与重载集合中的每一个函数的形参进行比较，根据比较结果决定要调用哪个函数
- 当两个重载函数参数数量相同且参数类型可以相互转换的情况
  - 编译器找到最佳匹配，并生成函数代码
  - 找不到与调用实参匹配，报错(no match)
  - 找到多余一个匹配，但每一个都不是最佳匹配，报错，二义性错误(ambigurous call)
- 名字查找发生在类型检查之前。当调用函数时，编译器先在作用域内查找对函数名的声明，找到了就忽略外层作用于的同名实体，然后砸检查函数调用是否有效

```cpp
void print(double);
void print(const string &);
void func() {
    void print(int);
    print(3.14); // 调用print(int), print(double)被隐藏
    print("value"); // 调用print(int)，print(string)被隐藏，报错char*转int非法
}
```

函数实参默认值：
- 默认实参作为形参的初始值出现在形参列表中
- 一旦某个形参被赋予了默认值，它后面的所有的形参都必须有默认值
- 合理设置形参的顺序，尽量让不怎么使用默认值的形参出现在前面，经常使用默认值的形参出现在后面
- 调用含有默认实参的函数时，可以包含该实参，也可以省略
- 一个含默认实参的函数可以被声明多次，但函数的后续声明只能为之前没有默认值的形参添加默认实参
- 默认实参初始值不可以是局部对象，可以是能转成形参类型的表达式
```cpp
sz wd = 80;
char def =  ' ';
sz ht();
string screen(sz = ht(), sz = wd, char = def); // 形参与默认实参关系绑定
void f2() {
    def = '*';
    sz wd = 100; // 两个wd变量独立，screen函数的形参绑定到全局wd变量
    window = screen(); // 调用 screen(ht(), 80, '*')
}
```

## inline和constexpr函数

inline函数：
- 将规模较小的且频繁调用的操作定义成inline函数，易于理解、修改和重用
- inline函数可以在调用点进行代码展开，可以抵消调用小函数的切换函数栈的上下文开销
```cpp
inline const string& 
    shorter_str(const string &s1, const string &s2) {
        return s1.size() <= s2.size() ? s1 : s2;   
    }
std::cout << shorter_str(s1, s2);
std::cout << (s1.size() <= s2.size() ? s1 : s2); // 代码展开
```

constexpr函数：
- 常量表达式函数可以用来对常量表达式进行初始化
- 在执行初始化时，编译器把对constexpr函数的调用替换成了结果值
- constexpr函数不一定返回常量表达式
  - 当constexpr函数的实参为字面值或常量表达式，且函数体内没有执行运行期动作语句时，返回常量表达式
  - 用返回非常量表达的constexpr函数初始化需要常量表达出的地方，编译器将报错
  - 编译器为验证是否返回常量表达式，将其默认为inline函数。若是，则结果替换，若否，可能报错或按inline代码展开
```cpp
constexpr int new_sz() { return 42; }
constexpr int var = new_sz();

constexpr size_t scale(int cnt) { return new_sz() * cnt; }
int i = 1;
scale(1); // 返回常量表达式
scale(i); // 非常量表达式，运行期求值
```

使用建议：
- 将constexpr函数定义和inline函数定义存放于头文件中，在使用到函数的其他文件中包含该头文件

## 条件调试代码

含义：
- 程序可以包含一些用于调试的代码，但这些代码只在开发的时候使用。当程序准备发布的时候，屏蔽掉这些调试代码。
 
assert:
- 调试代码
- <cassert>头文件定义了assert(expr)函数，用于运行期进行条件检查。当expr为假时，assert终止程序的运行。

NDEBUG:
- 调试开关
- 使用NDEBUG宏来开启或关闭assert的条件调试，当定义了NDEBUG宏，assert会被屏蔽，否则assert默认开启。

编译器提供的调试宏变量：
- 编译器为每一个函数提供了一些用于调试的宏变量
  - __func__，函数名
  - __LINE__，语句所在的行号
  - __FILE__，程序的文件名
  - __TIME__，文件编译时间
  - __DATE__，文件编译日期

```cpp
#define NDEBUG // 定义宏，屏蔽调试代码

void print(const std::string &word) {
#ifndef NDEBUG
    if (word.size() < required_size)
        std::cerr << "Error: " << __FILE__
                  << " : in function " << __func__
                  << " at line " << __LINE__ << '\n'
                  << "       Compiled on " << __DATE__
                  << " at " << __TIME__ << '\n'
                  << "       Word read was \"" << world
                  << "\": Lenght too short" << std::endl;
#endif
}

// 输出样式
Error: /path/test.cpp : in function print at line 18
       Compiled on Mar 11 2020 at 15:46:10
       Word read was "abcd": Lenght too short
```

## 函数匹配

从一组重载函数中选取最佳函数的过程称为函数匹配.

步骤:
- 查找同名函数，同名且函数声明在调用点可见函数构成候选函数集(candidate)
- 排除形参表与实参数量不符的函数，符合的构成可选函数集(viable)
- 逐个检查可选函数集里的每一个函数
  - 实参和形参类型相同
    - 顶层const
    - 数组名、函数名和对应的指针
  - 实参可以隐式类型转换为形参
    - non-const->const
    - 整型提升 char/short->int
    - double->int
    - 算术类型转换或指针类型转换
    - 类类型转换
  - 实参和类型不匹配也无法转换
- 匹配完成或匹配失败或二义性匹配

```cpp
void f();
void f(int);
void f(int, int);
void f(double, double=3.14); // 允许仅用一个参数调用它

f(3.14); // 可选函数集 { f(int), f(double, double=3.14) }
         // 匹配结果，f(int)

f(42, 3.14); // 可选函数集 { f(int, int), f(double, double=3.14) }
             // f(int,int) 只有一个参数类型相同，一个参数类型不同
             // f(double, double=3.14) 也只有一个参数类型相同，一个参数类型不同
             // 匹配结果， 二义性报错，无法选择

void func(long);
void func(float);
func(3.14); // 3.14是double类型，double->long和double->float都允许，匹配结果为二义性错误

void func(int);
void func(short);
func('a'); // 匹配结果, func(int)
```

## 函数指针

- 指向某个函数类型的指针，函数类型由形参表和返回值确定 
- 定义函数指针，即用指针替换函数名
- 函数名本身是指针，指向函数的第一条语句的地址
- 使用函数指针同函数名，可以不用解引用操作和取地址操作
- 可以使用typedef定义函数类型，替代函数指针表示函数类型
- 函数传递函数类型自动转为函数指针，函数返回函数类型需手动转为函数指针
- 重载函数的函数指针，指针类型需要重载函数精确匹配

```cpp
void (*pf)(int); // 定义一个函数指针pf，指向 void(int)
void *pf(int); // 定义一个名为pf的函数，void*(int)

void func(int);
pf = func; // pf = &func; ，以函数类型对象看待函数名
pf(1); // (*pf)(1);

typedef decltype(func) Func; // typedef void Func(int)
                             // using Func = void(int)
void f1(Func);
f1(func);

typedef decltype(func)* Pf; // typedef void (*Pf)(int)
                            // using Pf = int(*)(int)
Pf f2(Func); // auto f2(Func) -> void(*)(int)
f2(func)();
```

