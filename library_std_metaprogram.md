

## type_traits 库

提供了用于编译期计算和类型查询/比较/选择/转换/检查的模板类，帮助编写更灵活更安全的代码.

- 定义并获取编译期常量
  - integral_constant模板类，包装了静态常量和其类型信息，通过继承它可得到编译期常量
  - 其他的定义编译期常量的方式，定义一个struct，包含以下成员的任一个
    - static const value = 1;
    - enum { value = 1 };
```
template <typename T, T val>
struct integral_constant {
    static const T value = val;
    using value_type = T;
    using type = integral_constant<T, val>;
    operator value_type () { return value; } // ??
};

std::integral_constant<int, 1>::value; // 编译期常量1

typedef  integral_constant<bool, true>  true_type; // 编译期的true类型
typedef  integral_constant<bool, false>  false_type;
``` 

- 类型查询
  - 检查模板参数是否为某种类型，类型查询的模板类从std::integral_constant派生而来
  - 判断类型的trais通常结合std::enable_if使用，通过SFINAE(失败不是错误)来实现模板类的重载机制
```cpp
template <class T>
struct is_void;
struct is_floating_point;
struct is_arithmetic;   // 整型、浮点
struct is_fundamental;  // 整型、浮点、void、nullptr
struct is_compound;     // 组合类型(struct/union/enum/class)
struct is_union;
struct is_enum;
struct is_class;        // 类类型
struct is_abstract;     // 抽象类型

template <class T>
struct is_const;
struct is_signed;
struct is_unsigned;
struct is_reference;
struct is_pointer;
struct is_member_pointer;
struct is_array;
struct is_function;
struct is_object;
struct is_polymorphic; // 虚函数

std::is_const<const int>::value; // false
std::is_const<const int *>::value; // true
std::is_const<int *const >::value; // true
```

- 类型判断
  - 检查两个模板类型参数之间的关系
  - 当类型严格相同时，才认为类型一致
```cpp
template <class T, class U>
struct is_same; // T和U相同
struct is_base_of; // T是U的基类
struct is_converible; // T转换为U

std::is_same<int, signed int>::value; // true;
std::is_same<int, unsigned int>::value; // false
std::is_same<const int*, int *const>::value; // false
```

- 类型转换
  - 常用的类型转换traits
    - const
    - refernce
    - array
    - pointer
    - decay
```cpp
template <class T>
struct add_const;
struct remove_const;
struct add_lvalue_reference;
struct add_rvalue_reference;
struct remove_reference;
struct remove_extent; // 移除顶层维度（第一维度)
struct remove_all_extents; // 移除所有维度 (只剩数组名)
struct add_pointer;
struct remove_pointer;
struct decay; // 对于普通类型，移除引用和cv修饰符
              // 对于数组，移除顶层维度并为数组类型添加指针
              // 对于函数，为函数类型添加指针

template <class.. T>
struct common_type; // 多个类型参数的公共类型

std::remove_extent<int[]>::type; // int
std::remove_extent[][2]>::type; // int[2]
std::remove_all_extents<int[][2][3]>::type; // int

typedef std::common_type<unsigned, char, short, int>::type  numberic_type;
std::is_same<int, numberic_type>::value; // true;

std::decay<const int &>::type;  // int
std::decay<int[2]>::type;       // int*
std::decay<int(int)>::type;     // int(*)(int)


// 创建模板类型的对象时，需要使用原始类型，但模板类型参数T可能是引用类型
template <typename T>
typename std::remove_reference<T>::type*  create() {
    using U = typename std::remove_reference<T>::type;
    return new U();
}

template <typename T>
typename std::decay<T>::type*  create() { // 更好地方式
    using U = typename std::decay<T>::type;
    return new U();
}

// 将函数变成函数指针，可以将函数指针变量保存，延迟执行
template <typename Func>
struct Lazy_function {
    using Func_ptr = typename std::decay<Func>::type;

    Lazy_function(Func& f): fp(f) { }
    void run() { fp(); }

    Func_ptr fp;
};
```

- 类型选择
  - 根据判断式的结果来选择两个类型中的一个，在编译期动态选择类型
```cpp
template <bool B, class T, class F>
struct conditional;

std::conditional<true, int, float>::type; // int
std::conditional< (sizeof(long long) > sizeof(long double)), long long, long doouble >::type; // 比较选择较大的那个类型
```

- 获取可调用对象的返回类型
  - 函数入参是模板类型参数，因此无法直接确定函数的返回类型，可以使用decltype(auto)和返回类型后置来推导函数返回类型
  - 使用std::result_of模板类，获取可调用对象的返回类型，对函数使用时，需先将函数转换为函数对象,因为函数类型不是一个可调用对象
```cpp
template <typename F, typename Arg>
decltype(auto) Func(F f, Arg arg) {
    return f(arg);
}

int func(int);
int (&fn_ref)(int) = func;
int (*fn_ptr)(int) = func;
struct fn_class {
    int operator()(int);
};
std::result_of<fn_ref(int)>::type;
std::result_of<fn_ptr(int)>::type; 
std::result_of<fn_class(int)>::type;
std::result_of<decltype(fn)&(int)>::type; // int, 转函数引用
std::result_of<decltype(fn)*(int)>::type; // int, 转函数指针

// 使用std::result_of获取key的类型
template <typename Fn>
multimap< typename std::result_of<Fn(Person)>::type, Person >
group_by( const vector<Persion>& vec, Fn&& key_selector ) {
    using key_type = typename std::result_of< Fn<Person> >::type;
    multimap< key_type, Person > multi_map;
    std::for_each( vec.begin(), vec.end(), [&](const Person& person)
    {
        multi_map.insert( std::make_pair( key_selector(person), person ) );
    });
    return multi_map;
}
```

- 类型检查/类型条件启用
  - SFINAE规则 (subsitution-failure-is-not-an-error)
    - 编译器在匹配重载函数时，当某个重载函数不匹配，编译器不报错，继续匹配其他重载函数，直到把匹配成功或匹配完全部的重载函数，即某个匹配失败并非错误
  - std::enable_if利用SFINAE规则，实现根据条件选择重载函数
    - 仅当std::enable_if的条件为真时，才有效。为假，则编译报错，可以实现编译期的参数类型检查
    - 作用对象，对入参进行限定，检查其是否满足条件
```cpp
template <Bool B, class T = void>
struct enable_if;

// 限定返回I类型
template <class T> 
typename std::enable_if< std::is_arithmetic<T>::value, T >::type
    func( T t ) { return t; }

func(1); // T->int
func("test"); // 编译报错

// 限定函数参数
template <typaname T> 
T func(T t, typename std::enable_if< std::is_integral<T>::value, int >::type var = 0
{ 
    return t; 
}

 // 限定模板参数
template <class T, class = typeame std::enable_if< std::is_integral<T>::value>>
T func(T t) { return t; }

// 限定模板特例化实参
template <typename T, class Enable = void>
class A;

template <typename T>
class A <T, typename std::enable_if< std::is_floating_point<T>::value> >::type> { };


// 将入参类型分类
template <class T>
typename std::enable_if< std::is_arithmetic<T>::value, int >::type
    func(T t);

template <class T>
typename std::enable_if< !std::is_arithmetic<T>::value, int>::type
    func(T t);
```