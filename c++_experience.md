
## 导言

移动语义：
- 引出了左值与右值的概念区分
- 能对其取地址为左值对象，否则为右值
- 对象的静态类型与对象的左右值属性是相互独立，如一个右值引用是一个左值对象
- 左值意味着拷贝操作，右值意味着移动操作
- rhs名字，既可能表示是移动操作，也可能表示二元运算符的右侧运算对象
- 无论函数实参是左值还是右值，实参都为形参的一个副本copy

函数：
- 函数形参是左值，函数实参可能是左值或右值
- 完美转发，当一个实参被传递一个second函数时，原实参的左右值属性都会被保留

异常安全：
- 函数是异常安全的，指其能提供异常安全的保证
- basic guaratee，指当调用发生异常时，程序的不变量保持完整，没有数据结构被破环或资源泄漏
- strong guaratee, 指当调用发生异常时，程序保持调用之前的状态

函数对象：
- 一个支持调用运算符的类对象，行为如函数
- 可调用对象，能被各种函数调用语法形式调用，如指针，函数名，对象,lambda
- lambda表达式所创建的函数对象为closure类型

声明与定义：
- 声明，仅引入名字和类型，没有确定存储位置和具体的细节
- 定义，带有存储位置和实现细节的声明
  
函数签名(signature)：
- 签名是函数声明的一部分，形如bool(const &Widget)，由形参和返回类型确定，不包括函数名、参数名和说明符(noexcept/constexpr)

未定义行为：
- 指运行时行为是不可预测的
- 会产生未定义行为的操作有
  - 下标运算符[] 的界外访问
  - 多线程的数据竞争

数据竞争(data race)：
- 指多于两个线程，其中一个是writer，并且两线程同时访问同一内存位置
  
原始指针(raw pointer)：
- 内置指针类型为原始指针，相反地，是智能指针(smart pointer)
- 智能指针，指重载指针解引用操作(->和*)

## 类型推断

模板参数类型推断：

- 模板参数类型推断是auto的基础
- 编译器根据函数实参的值，来推断函数形参的类型ParamType和模板实参的类型T
- 通常ParamType和T是不相同的，因为ParamType可能带引用或const修饰
- 将ParamType声明为通用引用，相当于分别定义两个不同引用版本的重载函数
- 模板参数T的类型推断结果由函数实参类型和函数形参声明的类型进行 模式匹配 后得到
  - 任何情况下，忽略expr的引用修饰后，再进行类型推断
  - 当ParamType没有类型修饰符，则为值传递，expr的类型修饰符将被忽略,包括底层const，因为实参和形参相互独立
  - 当ParamType是左值引用, expr的类型即为T的推断结果
  - 当ParamType是通用引用, 仅当expr为右值时，T的推断结果为右值引用，其他情况为T和ParamType都为左值引用，可能是const
  - 当ParamType是数组引用，expr为数组名，则T推断为数组的引用。同理，expr为函数名，T为函数的引用
  - 当ParamType是数组指针，expr为数组名，则T推断为数组指针。同理, expr为函数名，T为函数指针 (退化)
```cpp
template <typename T>void f(ParamType param);
f(expr);

int x = 27;
const int &rx = x;

template <typename T> void f(T& param);
f(x); // T->int, param->int &
f(rx); // T->const int, param->const int &

template <typename T> void f(const T& param);
f(x); // T->int, param->const int &
f(rx); // T->int, param->const int &

template <typename T> void f(T *param);
f(&x); // T->int, param->int *
const int *px = &x;
f(px); // T->const int, param->const int *

template <typename T> void f(T&& param);
f(x); // T->int &, param->int &
f(rx); // T->const int &, param->const int &
f(27); // T->int, param->int &&

template <typename T> void f(T param);
f(x); // T->int, param->int
f(rx); // T->int, param->int
f("a"); // T->const char *, param->const char * 

template <typename T> void f1(T param);
template <typename T> void f2(T& param);
void some_func(int, double);
f1(some_func); // T -> void(*)(int, double)
f2(some_func); // T -> void(&)(int, double)
f1("a"); // T -> const char *
f2("a"); // T -> const char(&)[2]

template <typename T, std::size_t N>
constexpr std::size_t arraySize(T (&)[N]) noexcept { return N; } //利用推断结果返回数组的大小
```

auto类型推断：

- auto等同模板的参数T，变量的类型声明等同ParamType，类型推断规则同模板的类型推断
- 仅有一点不同，auto假设{value}为std::initializer_list<T>，而模板推断并不做这样的假设
- 当auto用在函数的返回类型和lambda的参数表时，采用的是模板的推断规则，即不假设{value}是初始化列表类型
```cpp
// auto的等价模板形式
const auto& rx = x; // auto->T, ParamType->const auto&

template <typename T> void func_for_rx(const T& param);
func_for_rx(x);

// auto与模板推断的区别
auto x3 = { 27 }; // auto->std::initializer_list<int>
auto x5 = { 1, 3.0 }; // fail, 无法推断std::initializer_list<T>的T

template <typename T> void f1(T param);
template <typename T> void f2(std::initializer_list<T> init_lst);
f1({ 11, 23, 9 }); // error
f2({ 11, 23, 9 }); // ok, T->int

// auto采用模板推断规则
auto create_initlist() { return { 1, 2, 3 }; } // error
auto resetV = [&](const auto& new_value) { v = new_value; };
resetV({ 1, 2, 3 }); // error
```

decltype类型推断：

- decltype如实返回名字或表达式的类型
- 当表达式是个左值表达式时，decltype返回引用
- 对名字使用括号(x)，将成为左值表达式
- decltype(auto)，意为auto表示返回类型需要推断，但使用decltype的推断规则

```cpp
const int i = 0; // decltype(i) -> const int
bool f(const Widget& w); / decltype(f) -> bool(const Widget&)
if (f(w)) // decltype(f(w)) -> bool

// 返回类型推断
template <typename Container, typename Index>
auto authAndAddress(Container& c, Index i) {
    authenticateUser();
    return c[i]; // 编译器从函数实现(return语句)推断返回类型
}
authAndAddress(deque,5) = 10; // error, 若c[i]返回引用，采用auto推断规则，返回右值

template <typename Container, typename Index>
decltype(auto) authAndAddress(Container& c, Index i) {
    authenticateUser();
    return c[i];
}
authAndAddress(vec,5) = 10; // ok, 采用decltype的推断规则，返回左值
decltype(auto) myWidget = w; // 使用decltype推断规则, decltype(auto)->const Widget&

template <typename Container, typename Index> // c++14 final version
decltype(auto) authAndAddress(Container&& c, Index i) { // 通用引用，相当于两个不同引用版本的重载函数
    authenticateUser();
    return std::forward<Container>(c)[i]; // 保留右值属性，避免c[i]元素的不必要拷贝
}
auto s = authAndAccess(makeStringDeque(), 5); // 考虑容器为右值的情况，若声明为左值引用则无法绑定到右值

template <typename Container, typename Index>
auto authAndAddress(Container& c, Index i)
    -> decltype(std::forward<Container>(c)[i])  // 使用c++11需显式指定推断返回类型的表达式
{
    authenticateUser();
    return std::forward<Container>(c)[i];
}

// expression (x) 的类型推断
decltype(auto) f1() {
    int x = 0;
    ...
    return x; // 返回名字x的类型int
}

decltype(auto) f2() {
    int x = 0;
    ...
    return (x); // 返回对局部变量的引用int&
}
```

查看编译器的类型推断结果：

- typeid(name/expr).name()，会移除const和引用修饰信息，有时不准确
- boost::typeindex，保留const和引用修饰信息

```cpp
#include <boost/type_index.hpp>

template <typename T> void f(const T& param) {
    using boost::typeindex::type_id_with_cvr;
    cout << type_id_with_cvr<T>().pretty_name() << '\n';
}
```

适宜使用auto的地方：

- 函数对象
  - auto，能自动容纳lambda的函数对象，无须显式声明函数类型
  - function，需显式指明函数类型，其调用bind和使用堆内存来保存lambda的函数对象
- 范围for循环
  - 若显式指定左侧变量的类型，当左右侧类型不一致时，可能会发生隐式类型转换，产生左侧类型的临时对象
  - 使用auto，左侧的类型与右侧初始值的对象一致，无隐式类型转换
- 函数返回类型
  - 当重构要修改返回类型时，auto会自动更新返回类型

不适宜auto的情况：
- vector<bool>下标运算符的返回结果，类型转换绑定到临时值 ??











