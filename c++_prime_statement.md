
## 空语句

用来表示什么也不做

```cpp
while (cin >> s && s != target) // 除此之外，什么也不做
    ;
```

## 复合语句

- 复合语句(compound statement)，指使用花括号括起来的语句或声明的序列，也成为块(block)
- 一个块就是一个作用域，定义在块中的名字从定义开始到块的结尾可见(有限的作用域可见)
- 语法上当成一条语句

```cpp
while (cin >> s && s != target)
    { }
```

## 条件语句

语句作用域： 定义在控制结构中的变量只能在相应的语句的内部可见
```cpp
while (int i = get_num())
    { }
i = 0; // 语句外不可见
```

悬垂else: else与最近的未配对的if配对，可以使用花括号指定if-else配对
```cpp
if () {
    if ()
} else
```

switch语句：
- case标签(label)： case关键字 + 整型常量表达式
- break用于中断当前的控制流，将控制权转移到上一层外部
- default用来表示默认处理的情况或者其他没有列出的情况
- switch 适用于根据值，做匹配   

执行流跳过变量定义问题： case处定义的变量，作用域为switch{ }块，c++语言规定，不允许跨过变量的初始化语句直接跳转到该变量作用域的另一个位置(不能跳过初始化语句，可以跳过声明语句，因switch块作用域)。如果要为某个case定义并初始化一个变量，就应该把变量定义在case的块内，确保后面的case标签在变量的作用域之外。
```cpp
switch(false) {
    case true:
        std::string file_name; // 不能跳过初始化语句
        int jval; // 可以跳过
    case false:
        if (file_name.emtpy())
            std::cout << jval;
}

// case块作用域
switch(false) {
    case true: {
        std::string file_name  = get_file_name();
        ...
    } break;
    case false:
        if (file_name.empty()) // error，未定义/未声明
}
```
使用建议：
- 每个case标签一个break;
- 每个switch都写default，若default不做任何事, 写空语句，表示所有情况都考虑到了

## 循环语句

迭代语句，重复执行操作直到满足某个条件才停下来

- while，用于不确定循环次数和需要在循环体外使用到循环迭代变量的场合
- for，计数循环或条件循环，语法头内定义变量，仅for语句内可见
- do-while: 循环控制变量必须定义在语句外，用于带提示用户输入的情况
- 范围for： for (declaration : expression) statement; 
  - declaration定义一个变量，要求序列中的元素能转换成该变量的类型. 用auto让编译器来确定类型
  - 每次迭代都会重新定义循环控制变量，并将其初始化为序列的下一个值，再执行statement

```cpp
// while语句外，使用循环变量的值
auot beg = v.begin();
while (beg != v.end() && *beg >= 0)
    ++beg;
if (beg == v.end())
    std::cout << "all is positive";

// for的语句头的初始化语句部分，须是同一类型
for (decltype(v.size()) i = 0, sz = v.size(); i != sz; ++i)
    v.push_back(v[i]);

for (int i; cin >> i; )
    v.push_back(i);

// 范围for
for (auto &r : v)
    r *= 2;

for (auto beg = v.begin(), end = v.end(); beg != end; ++beg) {
    auto &r = *beg;
    r *= 2;
}

// do-while
string str;
do {
    cout << "please enter two value ";
    int var1, var2;
    cin >> var1 >> var2;
    ....
    cout << "More? Enter yes or no： ";
    cin >> str;
} while (!str.empty() && str[0] != 'n')
```

## 跳转语句

- 跳转语句中断当前的流程
- break，用于循环或switch语句内，表示终止离它最近的循环或switch
- continue，用于循环语句内或循环内嵌套switch，表示结束当前迭代，并立即开始下一次
- goto， goto + label， 跳转到label初。 同样不能跳过初始化语句。
- return， 待述
  
```cpp
// 只对以下划线开头的单词感兴趣
string buf;
while (cin >> buf && !buf.empty()) {
    if (buf[0] != '_')
        continue;
}

// goto与等价循环
begin:
    int sz = get_size();
    if (sz <= 0)
        goto begin;

do {
    int sz = get_size();
} while (size <= 0);
```

## try语句

- 异常： 指发生于运行时的反常行为，这些行为超出来函数的正常功能范围。如与数据库断开连接、非法输入等
- 异常检测： 当程序检测到一个它无法处理的问题是，使用throw表达式向外发出异常信号(raise)
- 异常处理： try语句块中代码抛出的异常通常会被某个catch子句处理。catch子句称为异常处理代码
- 异常声明：位于catch子句的声明，指定了该catch子句能处理的异常类型
- 异常安全：当程序抛出异常后，程序能够执行正确的行为
- 异常类： 用于在throw表达式和catch子句之间传递异常的具体信息
- 调用函数形成嵌套try结构，寻找处理异常代码的过程与函数调用链相反
- 当程序某个位置throw表达式抛出异常时
  - 抛出位置的函数内没有try语句块，跳到外出函数查找try-catch语句
  - 有try语句块，则尝试匹配catch子句
    - 匹配成功，进入catch子句，执行异常处理代码，然后跳到整个try-catch语句后的第一条语句
    - 匹配失败，跳到上层函数，继续匹配，直到匹配成功或被系统ternimate函数终止函数(main函数层)

```cpp
void f() {
	try {
		throw std::runtime_error("f()");
	} catch (std::logic_error & e){
		std::cout << "logic_error";
    }
}

int main() {
	try {
		f();
	} catch(std::exception & e) {
		std::cout << e.what() << std::endl;
	}
	std::cout << "ok" << std::endl;
}

// 把对象相加的代码和用户交互的代码分离开来 ？？
while (cin >> item1 >> item2) {
    try {
        item1 + item2;
        if (fail)
            throw std::runtime_error("error");
    } catch (std::runtime_error err) {
        cout << err.what()
             << "\n Try Again ? Enter y or n" << endl;
        char c;
        cin >> c;
        if (!cin || c == 'n')
            break;
    }
}
```
标准异常类： 标准库提供了一组类，用于报告标准库函数遇到的问题
- <exception>定义了exception类，只能默认初始化，仅用于报告异常发生，不带额外信息
- <stdexcept>定义常见的异常类，可以带额外信息
- <new>定义了bad_alloc类，默认初始化，不带额外信息。new操作失败抛出
- <type_info>定义了bad_cast类，默认初始化，不带额外信息。dynamic_cast失败抛出
- what()成员函数，返回值类C风格字符串
- runtime_error, 用于表示在运行时才能检测出的问题
  - range_error, 计算结果无法表示
  - overflow_error, 上溢
  - underflow_error,  下溢
- logic_error, 程序逻辑错误，在运行前，可以避免
  - invalid_argument
  - domain_error
  - length_error
  - out_of_range:
  
参考： https://blog.csdn.net/qq_38211852/article/details/82256833





































