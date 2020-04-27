

以独立于特定的类型的方式来编写泛型程序，用户代码提供类型或值来实例化泛型程序。

模板一个创建类或函数的蓝图/公式，由模板名、模板参数表组成。


## 模板参数

类型参数与非类型参数：
- 参数，意味着未知和不确定
- 模板参数，表示出现在模板定义内的将被替换的符号名字,分为 类型参数 和 非类型参数
- 用户代码可能使用一个具体的类型名(类型参数)、也可能使用一个具体类型的对象(左值或右值)(非类型参数)来替换模板参数
- 类型参数，表示一个泛化的不确定的类型说明符，可以用它表示函数的形参类型和返回值类型，也可以用它来声明一个对象或类型转换
  - 使用typename来指定类型参数
  - 使用typename来标示访问的是模板类型参数的类型成员
  - 模板类型参数, 即模板参数是一个模板类
- 非类型参数，表示一个值。在模板的眼里，除了模板类型参数是一个类型，其他的都视为一个值，包括具体类型(确定的)及其对象
  - 当非类型参数是一个指向对象的指针或引用时，要求被指向的对象必须具有静态的生存期，即对象在编译期分配存储空间
  - 使用具体类型来指定非类型参数
  - 使用作用域运算符来访问类型参数的成员，具体类型参数不需要typename
```cpp
// 模板类型参数
template <typename T, typename F = less<T>>
constexpr int compare(const T &v1, const T &v2, F f = F()) { // 留心非指向相同数组的指针的比较错误
    if (f(v1, v2)) return -1; // 要求T类型已定义operator<
    if (f(v2, v1)) return 1;  // 只定义<，不定义>, 减少对类型的要求
    return 0;
}
bool res = compare(0, 42); // 使用less<int>()

// 非模板类形参数
template <unsigned N, unsigned M>
inline int compare(const char (&p1)[N], const char (&p2)[M]) {
    return strcmp(p1, p2);
}
compare("ab", "efg"); // compare(const char (&)[3], const char (&)[4])

// 访问模板类型参数的类型成员
template <typename T>
typename T::value_type top(const T& c) {
    if (!c.empty())
        return c.back();
    else
        return typename T::value_type(); // 构造一个value_type的临时对象
}
```

模板参数作用域：
- 模板参数的作用域从模板参数表到模板的末尾
- 作用域内的模板参数名不能重复
- 作用域内的模板参数名将隐藏外层作用域的同名符号
```cpp
typedef double A;
template <typename A, typename B> void f() {
    A tmp; // 模板类型参数A 隐藏外层作用域的A
}

template <typename V, typename V>; // error，重复名字
```


## 默认模板实参

默认模板实参：(default template arguement)
- 可以为类模板和函数模板提供实默认参
- 同函数默认实参规则，只有当它右侧的所有参数都有默认实参时，它才可以有默认实参
```cpp
template <typename T, typename F = less<T>>
int compare(const T& v1,, const T& v2, F f = F()) {
    if (f(v1, v2)) return -1; // 要求F为可调用对象，接受的实参类型与前两函数参数兼容，且返回值能转为bool
    if (f(v2, f1)) return 1;
    return 0;
}

template <class T = int> class Numbers {
public:
    Numbers(T v = 0): val(v) { }
private:
    T val;
};
Numbers<> average_precision;// 使用默认模板实参int
```


## 模板参数推导

- 当用模板类型参数定义函数参数时，可以从函数实参表来推导出模板实参表
- 函数模板可以进行模板实参推导，但仅限推导与函数参数表有关的模板类型参数，其他模板类型参数仍需显式给出
- 类模板没有函数参数表，无法由函数实参到模板实参的推导，因此需在模板实参表里显式给出实参
- 编译器只能推导与函数参数表相关的模板参数，其他模板参数需在模板实参表中显式传入

模板参数推导类型转换：
- 与模板类型参数相关的函数参数：
  - 多数情况下，从函数实参类型推导得模板实参，不发生隐式类型转换，不匹配则调用失败
  - 仅允许以下情况发生类型转换
    - non-const到const
    - 数组或函数到指针
  - 若多个函数形参有相同的模板类型参数，则调用时，要求多个函数实参的类型必须相同
- 与模板类型参数无关的函数参数：
  - 由具体类型声明的函数形参，在函数匹配时，允许函数实参到函数形参的类型转换
  
显式指定模板实参：
- 在模板实参表显式指定模板实参值
  - 当编译器无法推断出模板实参的类型，如用模板参数声明的函数返回类型    
  - 当函数的返回类型与参数表中的类型不同时
- 显式模板实参按左到右的顺序依次与模板形参对应
  - 将能由编译器隐式推导的模板参数放后边，需显式指定的模板参数放前边，就无须全指定模板实参
  - 可以由函数参数表推导的模板形参放在形参表尾部，需在模板实参表显式传入的模板形参放在形参表的前部
- 显式指定的模板实参，相当于具体类型，函数实参到函数形参可以隐式类型转换

```cpp
template <typename T1, typename T2, typename T3> T1 sum(T2, T3);
auto res = sum<long long>(i, lng); // 显式指定T1, T2和T3由编译器隐式确定

// 错误的示例
template <typanme T1, typename T2, typename T3> T3 sum(T1, T2);
auto res = sum<long long>(i, lng); // 显式指定T1，T1和T2被编译器确定，T3未确定
auto res = sum<long long, int, long>(i, lng); // 显式指定T1, T2, T3

int a = 6; double b =3.14;
std::cout << std::max<long double>(a, b) << std::endl;

// 尾置类型转换
template <typename It>
auto func(It beg, It end) -> decltype(*beg) { // 返回类型与decltype(*beg)的结果相类型相同
    return *beg;
}
```

模板参数推导与复合类型：
- 当初始化一个函数指针时，编译器使用指针类型来推断模板实参
- 折叠引用，如果创建定义一个引用的引用，则这些引用形成"折叠"
- 多数情况引用会折叠成一个普通的左值引用类型，当且仅当定义右值引用的右值引用时折叠成右值引用
- 当使用引用折叠，涉及的类型可能是非引用类型和引用类型时，容易产生不确定性而出错
```cpp
// 函数指针类型
template <typename T> int compare(const T&, const T&);
void func(int(*)(const string&, const string&));
void func(int(*)(const int&, const int&));
func(compare); // error, 函数模板非具体函数类型
func(compare<int>);  // ok, int compare(const int&, const int&);

// 引用折叠使用错误的示例
template <typename T> void func(T && value) {
    T t = value; // T可能是int, 也可能是int&或int&&. t的语义不确定（初始化或绑定)
    t = fcn(t);
    if (val == t) { } // 若T为int&或int&&，则条件表达式永为真
}

// std::move
template <typename T>
    typename remove_reference<T>::type&& move(T && t) {
        return static_cast<typename remove_reference<T>::type&&>(t); // 从左值类型转换到右值引用
    }
std::string s1("hi"), s2;
s2 = std::move(string("bye")); // T->int&&, int&& && -> int&& t
s2 = std::move(s1); // T->int&, int& && -> int& t

// std::forward，将参数连同类型信息转发给另一个函数
template <typename F, typename T1, typename T2>
void flip(F f, T1 && t1, T2 && t2) {
    f(std::forward<T2>(t2), std::forward<T1>(t1)); // 若保持实参右值引用不变
}
```


## 模板参数类型转换

显式用于模板参数类型转换的标准库模板：(std)
- add_const，对指针或引用则添加顶层const  
- add_pointer
- add_lvalue_reference
- add_rvalue_reference
- remove_pointer
- remove_reference
- remove_extent，如 X[n] -> X
- remove_all_extents，如 X[n1][n2].. -> X
- make_signed，如 unsigned -> signed
- make_unsigned
```cpp
template <typename It>
auto func(It beg, It end) -> 
    typename remove_reference<decltype(*beg)>::type
    {
        return *beg;
    }
```


## 模板实例化


隐式实例化：
- 模板类型参数被模板实参替代，非类型参数被一个常量表达式的值所替代
- 将模板实参绑定到模板形参上，类型参数则进行类型名替换，生成不含模板参数的函数或类
  - 函数模板，生成多个类型的同名函数，效果同相同形参数的重载函数
  - 类模板，生成多个类型的类定义，类的成员函数与类有相同的模板类型参数，随同变化
  - 成员模板，在类定义内，生成多个函数模板的实例
  - 类模板的成员模板，类的成员函数与类可以有不同的模板类型参数，独立变化
- 模板实例化检查报错
  - 1.编译模板时，编译器检查语法错误，如拼写错误
  - 2.用户使用模板时，模板实参检查
  - 3.实例化模板，进行静态类型检查，可能有编译错误或链接错误
```cpp
// 未被处理的源文件
template <typename T>  // 函数模板
const T& func(const T &v) { return v; }

template <typenanme T> class X { // 类模板
public:
    void print(const T &value) const;
};

template <typename T> class Y { // 类模板+成员模板
public:
    template <typename U>
    void print(const U &value) const;
private:
    T var;
};

int main() {
    func(1);
    X<int> xi;
    xi.print(1);
    X<double> xd;
    Y<int> yi;
    yi.print(3.14);
}

// 模板处理后的源文件
const int& func(const int &v) { return v; }

class X {
public:
    void print(const int &value) const;
};

class X {
public:
    //void print(const double &value) const; // 未被使用，不生成
};

class Y {
public:
    void print(const double &value) const;
private:
    int var;
};

int main() {
    func(1);
    X<int> xi;
    xi.print(1);
    X<double> xd;
    Y<int> yi;
    yi.print(3.14);
}
```

实例化的时机：
- 编译器遇到模板定义时不实例化，当模板被用户代码使用时，进行隐式实例化
  - 当类不是模板，但其成员是模板时，则类的实现会被实例化，类的模板成员函数只有被使用到时，才实例化这个成员模板
  - 当模板的定义在本文件时
- 当遇到显式实例化定义且未使用extern声明时，进行显式实例化
  - 当遇到extern声明时，不在本文件内进行模板实例化，在链接阶段链接到模板实例所在的文件
```cpp
// 什么时候发生实例化
template <typename T> class Stack { };

void f1(Stack<char>);                   // 当函数被调用时才实例化
class Exercise {
   Stack<double> &rsd;                 // 定义引用，不实例化
   Stack<int>    si;                   // 找不到类定义，链接失败
};

temlplate <typename T> T calc(const T&, const T&); // 模板声明
template <typename U> T calc(const U& a, const U& b){ } // 模板定义,与声明有相同数量和种类(类型参数和非类型参数)的参数

int main() {
   Stack<char> *sc;                    // 定义指针，不实例化
   f1(*sc);                            
   int iObj = sizeof(Stack< string >); 
}
```

显式实例化：
- 当多个独立编译的源文件使用了相同的模板，并且提供相同的模板实参，编译器将在每个源文件中有相同的模板实例
- 用显式实例化(explicit instantiation)和extern声明来避免多个相同的实例化的开销
- 类模板实例化定义语句，将实例化类的全部成员，不同于普通的类模板实例化
- 类模板的实例化定义，要求模板实参类型能用于模板的所有成员函数
- 当编译器遇到实例化定义时，为其生成代码
- 当模板定义可能被其他文件使用时，因不清楚其他文件会使用到模板的哪一部分，因此在模板定义的文件中，显式要求编译器全部实例化模板
```cpp
// file_a, 实例化定义
template int compare(const int&, const int&);
template class Blob<string>; // 实例化模板的所有成员

// file_b，实例化声明
extern template int compare(const int&, const int&);
extern template class Blob<string>;

Blob<int> a1 = {1,2}; // 本文件实例化
Blob<int> a2(a1); // 本文件实例化
Blob<string> sa1, sa2;
int i = compare(a1[0], a2[0]); //函数定义在其他文件
```

重复实例化问题:
- 由于编译器会为模板生成类定义或函数定义，因此要求模板实例化时，模板的定义必须是可见的
- 若多个文件都生成了同一个模板参数的实例，则由链接器在链接时，去除重复的实例，仅保存一份实例
- 使用显式实例化和extern做外部符号链接


## 模板别名

- 类模板的一个实例为一个类类型，可以定义一个typedef来引用实例化的类，但typedef无法引用到模板名
- 使用using来给类模板起别名 ??
- 模板别名的应用场景 ??
```cpp
// typedef
typedef X<string> X_Str;

// using
template <typename T> using partNo = pair<T, unsigned>;
pairNo<string> books; // 只能指定first成员类型，无法指定second
pairNo<Vehicle> cars;
pairNo<Student> kids;
```


## 函数模板

函数模板参数：
- 一个函数模板就是一个公式，可用来生成针对特定类型的函数版本
- 将函数定义为模板，函数模板为一系列重载函数的集合
- 函数模板的模板参数表绑定到函数的形参表，由调用者提供实参来初始化函数形参
- 用函数实参初始化用模板类型参数声明的函数形参，编译器通常不对实参进行类型转换，除了下列情况外
  - non-const指针或引用 -> const指针或引用
  - 若形参不是引用类型，则数组实参转为数组指针，函数名实参转为函数指针
- 若函数参数类型不是模板参数，则对实参进行正常的类型转换
- 
- 通常，函数的行为对于模板参数有隐含要求，如要求类型定义了operator<运算符以支持小于比较操作
- 从泛型角度看，函数模板类型参数应尽可能减少对类型的要求
```cpp
template <typename T> T fun_obj(T, T);
template <typename T> T fun_ref(const T&, const T&);

std::string s1 = "a";
const std::string s2 = "b";
fun_obj(s1, s2); // fun_obj(string, string), 忽略s2的顶层const
fun_ref(s1, s2); // fun_ref(const string&, const string&)

int arr_1[10], arr_2[10];
fun_obj(arr_1, arr_2); // fun_obj(int*, int*)
fun_ref(arr_1, arr_2); // error, 类型不匹配

template <typename T> int compare(const T&, const T&);
compare("bye", "dad"); // ok
compare("hi", "world"); // error, compare(const char (&)[3], const char (&)[6])
```

使用模板类型参数的类型成员:
- c++假定通过作用域运算符访问的是名字而非类型名，用typename来表示告诉编译器该名字是一个类型，不能使用class
```cpp
template <typename T>
typename T::value_type top(const T& c) {
    if (!c.empty())
        return c.back();
    else
        return typename T::value_type();
}
```

函数模板实例化：
- 默认地，当函数被调用，编译器才实例化出函数模板的一个特定版本(生成一个实例)
- 使用实例化定义，可以显式实例化函数模板
- 编译器通常用函数实参来推断模板实参，用模板实参来实例化一个特定版本的函数

函数模板重载：
- 函数模板可以被另一模板或普通非模板函数重载，名字相同的函数必须具有不同数量或类型的参数
- 函数匹配时，优先选择非模板函数，再选择更特例化模板


## 类模板

类模板参数：
- 将类定义为模板，类模板为一系列相似类型的集合
- 类模板名不是一个类型说明符，是一个模板说明符，不能直接用模板名来声明对象或成员函数的返回类型和形参
- 类模板参数作用域覆盖整个类，静态成员的类外定义须使用类作用域访问运算符
- 为了使用类模板，需要在模板名的尖括号内(模板实参表)提供模板实参
- 类模板的成员函数具有和模板相同的模板参数，类模板-普通成员函数
- 对于一个实例化了的类模板，其成员只有在使用时才被实例化
```cpp
std::shared_ptr<std::vector<T>> data; // 将模板自己的参数T当作被使用模板shared_ptr的实参

// 成员函数类外定义
template <typename T>
X<T>  X<Ｔ>::operator++(int) { 
    X ret = *this;
    ++*this;
    return ret;
}

// 静态成员初始化
template <typename T>
size_t Foo<T>::ctr = 0;

Foo<int>::ctr;
```

类模板实例化：
- 一个类模板的每个实例都形成一个独立的类
- 类模板的每一个实例都有一个独立的static对象
- static成员只有在使用时才被实例化
```cpp
Blob<int> ia;

// 等价形式
template <> 
class Blob<int> {
    ...
};

```

友元模板：
- 一个类可以将另一个模板的每个实例声明为自己的友元(1:n)，或限定特定的实例为友元(1:1)
```cpp
template <typename T> class X {
    friend class T;
    template <typename U> friend class Y;
    friend bool operator==<T>(const X<T> &lhs, const X<T> &rhs)； 
}；

// 友元
template <typename T>
bool operator==(const Blob<T>&， const Blob<T>&);
template <typename T> class Bolb {
    friend bool operator==<T>(const Blob&, const Blob&); // 授予同类型的相等比较运算符友元函数访问权限
}；
Bolb<int> arr; // operator==<int>是arr对象的友元


template <typename Type> class Bar {
    friend Type; // 声明实例化Bar的类型为友元类
};
```


成员模板：
- 类包含模板成员函数(member template), 成员模板不能是虚函数
```cpp
class DebugDelete {
public:
    template <typename T> void operator()(T *p) const;
};

DebugDelete()(new double);
```

类模板的成员模板：
- 为类模板定义成员模板，类和成员，各有自己的独立的模板参数
```cpp
template <typename T> class Blob { // 支持不同类型的迭代器
    template <typename It> Blob(It b, It e);
};

// 类外定义
template <typename T> class Blob {
        template <typename It> Blob(It b, It e); // 构造函数成员模板
};

template <typename T>
template <typename It>
Blob<T>::Blob(It b, It e) {
    ...
}
Blob<int> arr(begin(a1), end(a1));
```


## 可变参数模板

- 接受一个可变数目参数的模板函数或模板类
- 可变参数包为任意数量任意类型的参数，用省略号表示一个参数包类型
  - 模板参数包，0个或多个模板参数
  - 函数参数包，0个或多个函数参数
```cpp
template <typename T, typename... Args>
void foo(const T& t, const Args&... rest) {
    cout << sizeof...(Args) << endl;
    cout << sizeof...(rest) << endl;
}

string str = "";
foo(3.14, "a"); // foo(const double&, const char[3]&)
foo(3.14, "a", str); // foo(const double&, const char[3]&, const string&)
```

参数包展开：
- 使用参数包中的参数，须将参数包展开
- 两种参数包展开方法
  - 通过递归的模板函数
    - 参数包展开函数
    - 同名递归终止函数
    - 函数实参表与形参表进行模式匹配，推导参数包集合
  - 通过逗号表达式和初始化列表方式
    - 逗号表达式的结果为最后一个表达
    - 利用对象构造的过程中，将参数包展开
    - 每个初始值为一个括号括起的逗号表达式，表达式内含处理参数包中参数的处理逻辑
```cpp

// 不同的递归终止函数，将决定当参数为多少时终止
void print(); // 当参数包为空时，递归终止

template <typename T>
void print(T t); // 当参数包展开到最后一个参数时，递归终止

template <typename T>
void print(T t1, T t2); // 当参数包展开到最后两个参数时，递归终止

template <typename T, typename... Args>
void print(T t, Args... rest) { // 每次调用，参数包的参数减少一个
    std::cout << t << " ";
    print(rest...);
}

// 打印参数包
template <typename T>
ostream& print(ostream &os, const T &t) { }

template <typename T, typename... Args>
ostream& print(ostream &os, const T &t, const Args&... rest) {
    os << t << " ";
    return print(os, rest...); // print(os, arg1, arg2...)
    return print(os, debug(rest)...)); // debug(arg1), debug(arg2), debug(arg3)...
    return print(os, debug(rest...));  // debug(arg1, arg2, arg3...)
}

// 运行时打印tuple的所有参数 ??
// 当参数的索引小于参数包个数时，取出当前索引位置的参数并打印，当相等时，递归终止


// 逗号表达式展开参数包
template <typename T>
void print(T t) { std::cout << t << " "; }

template <typename... Args>
void expand(Args.. args) {
    int arr[] = { (print(args), 0)... };
    // 展开 { (print(args1), 0), (print(arg2), 0), (print(arg3), 0), etc }
    // 最终结果 int arr[] = { 0, 0, 0, etc };
}

template <typename... Args>
void expand(Args... args) { // 调用lambda的函数对象
    std::initializer_list<int>{ ([&]{std::cout << args << endl;}(), 0)... };
}
```

转发参数包:
- 右值版本可以避免临时对象的拷贝，左值版本则需要拷贝
```cpp
template <typename... Args>
inline void StrVec::emplace_back(Args&&... args) {
    chk_n_alloc();
    alloc.construct(first_free++, std::forward<Args>(args)...);
}

// 创建工厂
template <typename... Args>
T* instance(Args&&... args) {
    return new T(std::forward<Args>(args)...);
}
```

可变参数模板类:
- 模板参数有可变模板参数的模板类，参数包的数量为0到n个，参数包通过模板特化或继承的方式展开
```cpp
// 版本1
template <typename... Args>
struct Sum; // 前向声明，声明Sum为一个可变参数模板类，可省略

template <typename First, typename... Rest>
struct Sum <First, Rest...> // 参数包展开类
{
    enum { value = Sum<First>::value + Sum<Rest...>::value };
};

template <typename Last>
struct Sum <Last> // 特化递归终止类，要求参数包不能为空，至少要有一个参数
{
    enum { value = sizeof(Last) };
};

template <>
struct Sum <> // 展开到0个参数为止，参数包可以为空
{
    enum { value = 0 };
};

template <typename First, tyepname Last>
struct Sum <First, Last> // 展开到2个参数为止
{
    enum { value = sizeof(First) + sizeof(Last) };
};
```

可变参数包的应用:
- optional实现
  - 表示未初始化的概念，用来检查函数返回值是否有效
  - optional对象已初始化则表示有效值，否则为无效值
  - 当根据某个条件查找元素时，若查找不到元素，则返回一个无效值 (函数正确执行非执行失败)
```cpp
std::optional<int> op; // c++17
if (op) std::cout << "invalid";
op = 1;
if (op) std::cout << "valid";
```

- 惰性求值类lazy实现
  - 表达式不在它被绑定到变量之后就立即求值，而是在后面某个时候求值(延迟求值)
  - 典型应用场景
    - 当初始化某个对象时，该对象引用了一个大对象，这个对象的创建需要较长的事件，也需要分配较多的内存空间，初始化很慢
    - 很多时候并不需要马上就获取它，在需要时再去获取，延迟加载-

- variant实现
  - 类似于union，定义多种类型，允许赋不同类型的值给它
  - 用variant可以擦除类型，不同类型的值都统一为variant
  - 但variant只能存放已定义的类型，通过get<T>(v)获取值时，若T与v类型不匹配，则抛出bad_cast异常
  - 通过索引位置获取类型或通过类型名获取索引位置
```cpp
std::variant<int, double, std:;string, int> v;
v = 10;
v.visit( [&](double i){ std::cout << i; },
         [](short i){ std::cout << i; },
         [=](int i){ std::cout << i; },
         [](std::string i){ std::cout << i; }); //???

// 上式等价于
struct visitor {
    void operator()(int);
    void operator()(shrot);
    void operator()(double);
    void operator()(std::string);
};
boost::apply_visitor(visitor, it->first); // int
```

## 模板特例化

- 定义比通用版本更为特例的模板，便于对某些情况进行特殊处理
- 模板匹配，优先选择匹配更为特例的版本
- 全特化和偏特化

```cpp
template <typename T> int compare(const T&, const T&);

tempalte <>
int compare(const char* const &p1, const char* const &p2) {
    return strcmp(p1, p2);
}

compare("hi", "mom"); // 调用特例化版本

// 部分特例化
template <typename T> struct remove_reference<T&>  { using type = T; };
template <typename T> struct remove_reference<T&&> { using type = T; }; 
```

模板特例匹配:
- 与重载函数的匹配规则有所差异
- 优先匹配更特例的情况，当模板A的集合是模板B集合的真子集时，则模板A更为特殊
- 当选不出两者中哪一个模板更为特例时，则二义性报错
```cpp
#include <iostream>

template <typename T1, typename T2, typename T3>
void func(T1 a, T2 b, T3 c) { std::cout << "#1"; }

template <typename T2, typename T3>
void func(int a, T2 b, T3 c) { std::cout << "#2"; }

template <typename T1>
void func(T1 a, char c1, char c2) { std::cout << "#3"; }

int main() {
    func(int(), int(), int()); // #2
    func(char(), char(), char()); // #3
    func(int(), char(), char()); // 二义性错误
                                 // #2的T2和T3的集合比#3的char更大，#3的T1的集合比#2的int大
                                 // #2 和 #3 互不为真子集
}
```

