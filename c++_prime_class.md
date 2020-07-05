
## 基本概念

类： 使用类定义自己的数据类型，来反映待解决问题中的各种概念  
类接口： 描述用户所能执行的操作，以类声明的方式表示  
类实现： 实现类接口描述的操作，包括类的数据成员、接口的实现和其他私有函数  
类封装： 将类接口和类实现分离，使用者只能访问到类接口，无法访问到类实现，封装隐藏实现细节  
抽象数据类型： 类的设计者负责考虑类的实现过程，类的使用者只需抽象地思考类做了什么，无需了解类型的工作细节  
数据抽象： 是一种依赖于接口和实现分离的编程技术。将对象的具体实现与对象所能执行的操作分离开来 ??  
成员函数： 属于类接口的一部分，函数声明在类里，函数定义可以在类里或类外。当函数定义在类里，默认为inline  
普通函数/辅助函数： 逻辑上属于类的一部分，但不属于类接口。函数声明和定义在类外，与类存放在同一文件  
用户：类的用户是程序员，  
类机制： 让类类型的行为模仿内置类型，成员函数体提供额外的自定义操作的途径  

## 对象调用函数

调用成员函数：
- 编译器为每个成员函数的形参表，添加了一个this指针的形参，指针的类型为成员函数所属的类类型
- 当对象调用成员函数时，编译器将对象放到调用运算符的实参列表里，用对象的地址初始化this指针
- 在成员函数内，通过this指针来访问对象的成员，通过解引用this指针来返回对象(左值)
- this指针形参为编译器自动添加，默认为顶层const，自定义this则重复定义
- 当传入成员函数的对象为常量对象时，需在形参表后使用const来修饰this指针，指向常量对象的常量指针
- 常量对象以及常量对象的引用或指针只能调用常量成员函数
- 当成员函数有多个重载版本时，传入常量对象调用const版本，传入非常量对象调用非常量版本
- 成员函数内部可以访问到类作用域内定义的名字，类作用域是成员函数作用域的外层作用域
- 编译器处理类时，先编译类的成员数据声明，然后再编译成员函数声明，因此类的成员数据声明位置可以位于成员函数之后
```cpp
struct Sale_data {
    std::string isbn() const { return bookNo; }
    std::string bookNo;
};

//  成员函数声明等价形式
std::string isbn() const { return bookNo; }
std::string isbn(const Sale_data *const this) {
    return this->bookNo; 
}

// 调用成员函数等价形式
object.isbn();
Sale_data::isbn(&object);
```

调用普通函数/辅助函数：
- 显式声明类对象形参，并在调用函数时显式传入对象
- I/O函数，将给定流中的数据读到给定对象 和 将给定对象的内容打印到给定的流
```cpp
Sale_data add(const Sale_data&, const Sale_data&);
std::ostream print(std::ostream&, const Sale_data&);
std::istream read(std::istream&, Sale_data&);

Sale_data add(const Sale_data &lhs, const Sale_data &rhs) {
    Sale_data sum = lhs; // left-hand side
    sum.combine(rhs);
    return sum;
}

std::ostream& print(std::ostream &os, const Sale_data &item) {
    os << item.isbn(); // 输出函数不负责换行，由用户代码负责格式化
    return os;
}

std::istream& read(std::istream &is, Sale_data &item) {
    is >> item.bookNo;
    return is;
}
```


## 类作用域

名字查找(name lookup): 寻找与所用名字的声明的过程(最佳匹配)，名字先声明后使用

名字查找的过程：
- 在当前作用域内，查找名字使用之前的声明语句
- 若当前作用域查找失败，继续在外层作用域进行查找
- 如果最终没有找到，则报错

类内名字查找的过程：    
- 编译器首先编译类内的所有声明(包括函数声明)，直到类全部可见后才编译成员函数体(函数体)
- 在编译声明时，在类作用域内，往前查找使用到的名字的声明
  - 若查找失败，在外层作用域继续往前查找
    - 查找类出现前的声明语句
    - 查找类外定义成员出现之前的声明语句
- 最终没有找到，则报错

成员函数体内名字的查找过程：
- 函数作用域
- 类作用域，查找整个类内的声明语句，无前向要求
- 外部作用域

```cpp
// 演示代码为错误代码
int height;
using Pos = unsigned long;
class Screen {
    public:
        void func(Pos height) { // 往前朝着Pos，解析为unsigned long
            cursor = width * height; // 使用函数形参
            cursor = width * this->height; // 使用类成员
            cursor = width * ::heigth; // 使用全局作用域的height
        }
        void set_height(Pos);
        using Pos =  std::string::size_type; // 报错，尝试修改已声明的Pos的含义
    private:
        Pos cursor = 0;
        Pos height = 0, width = 0;
};
Pos verify(Pos); // Pos解析为unsigned log
Screen::Pos verify(Screen::Pos); // 类定义前的声明语句
void Screen::set_height(Pos var) {
    height = verify(var);
}
```

使用建议:
- 类内的类型声明应放在类的开始，确保所有使用该类型的成员出现在该类型定义之后
- 成员函数不该使用与类成员同名的形参，容易出现混淆

类的作用域：
- 类作用域外的对象或函数要访问类内的名字，只能由类对象、类指针、类引用通过成员访问运算符来访问
- 对于static成员和类内声明的类型名，使用作用域运算符访问
- 成员函数定义在类外，使用类作用域运算符，表示成员函数的形参表和函数体属于类的作用域，而返回值类型的名字不属于类的作用域
- 若成员函数的返回值类型的名字是在类内声明的，需要对返回值类型名加上类作用域运算符


## 构造函数

构造函数： 类通过一个或几个特殊函数来控制其对象的初始化过程

默认构造函数： 类通过一个特殊构造函数来控制默认初始化过程，0实参就能调用的构造函数(支持形如X x;的语法形式)

合成的默认构造函数(synthesized default constructor)：指编译器创建的构造函数

构造函数初始值列表(constructor initialize list): 为新创建的对象的一个或几个数据成员进行初始化

类内初始值：成员初始化的另一种写法，编译器在构造函数的构造函数初始值列表隐式进行

- 构造函数没有返回值，形参表和函数体可为空，可以被重载
- 当类的对象被创建，就会执行构造函数，初始化对象的数据成员
- 构造函数不能被声明为const。若定义const对象，在构造函数执行完，对象才有const属性
- 构造函数的执行过程：构造函数初始值列表、构造函数函数体内语句
- 若无法使用类内初始值，则所有构造函数应该显式初始化每个内置类型的数据成员
- 构造函数初始值列表按成员声明的顺序进行初始化
- 构造函数体内无法完成语法意义的类成员初始化，只能赋值
- 执行构造函数体时，对象还尚未完全构造。对于指向资源的指针成员，可以在构造函数体内初始化资源指针
- 对含有const成员、引用成员和未提供默认构造函数的类对象成员，必须使用类内初始值或构造函数初始化列表进行初始化
- static成员不属于任何一个对象，构造函数不对它进行初始化
```cpp

// 正确的范例


// 演示代码为错误代码/错误范例
class Y {
    public:
        int a;
        Y(int b): a(b){ 
};

class X {
    int i;
    const int j;
    int &ref;
    Y y = 1;
    X(): j(i), i(1) { // j被初始化为随机值，报错未初始化ref
        ref = i; // 赋值非初始化
    }
};
```

使用建议：
- 令构造函数初始值的顺序与成员声明的顺序保持一致
- 尽量避免使用某些成员初始化其他成员，采用构造函数实参来初始化成员 X(int val):i(val),j(val){ }
- 

默认构造函数： （默认的含义为不带实参，是一种函数类型意义）
- 不需要提供实参就能调用的构造函数，执行默认初始化过程
- 编译器为类隐式定义的构造函数，执行默认初始化
- 编译器创建默认构造函数的需满足的条件：
  - 类没有声明定义任何构造函数
  - 类对象的数据成员有默认构造函数 或 无类对象成员
- 如需要，可以使用default显式定义默认构造函数，作用等同编译器创建的默认构造函数
- default可以写在类内构造函数声明处，也可以写在类外构造函数定义处
- 对于带全部实参默认值的构造函数，也视为默认构造函数
- 当存在多个默认构造函数时，可能发生二义性调用错误
- 初始化形式，当对象被默认初始化或值初始化时，都会自动调用默认构造函数
  - 值初始化，编译器用成员的类型默认值来隐式初始化成员
    - 当提供的初始值数量小于数组大小时，剩余部分进行值初始化
    - 当没有用初始值定义一个局部静态变量时
    - 表达式 T()，显式值初始化 ?? 调默认构造函数？
  - 默认初始化，编译器隐式调用类对象成员的默认构造函数，内置类型则随机值
    - 没有用初始值定义局部自动对象
    - 类对象成员没有在构造函数初始化列表被显式初始化，且有默认构造函数
- 如果定义了其他构造函数，编译器不再隐式定义默认构造函数，最好也提供一个默认构造函数，以支持形如X x;的语法形式

```cpp
struct Sale_data {
    std::string bookNo;
    unsigned units_sold = 0; // 售出数量
    double revenue = 0.0; // 营业额

    Sale_data() = default;
    Sale_data(const std::string &s): bookNo(s) { }
    Sale_data(std::istream &is) { read(is, *this); }

    std::istream& read(std::istream &is, Sale_data &item); // 交叉引用名字，须在类作用域内声明
};

std::istream& Sale_data::read(std::istream &is, Sale_sata &item) { // 若没有Sale_data::，则新定义函数
    is >> item.bookNo >> item.units_sold >> item.revenue;
    return is;
}

Sale_data object(std::cin);


// 默认构造函数错误示例
struct X {
    X(int i) { }
};

struct Y {
    Y() { } // error, 无法调用X的默认构造函数
    // Y():X(0) { } // ok, 方式一，构造函数初始值列表显式初始化类对象
    X x;
    // X x = 0; // ok, 方式二，类内初始值显式初始化类对象;
};

Y y; // error,试图默认初始化，无法默认初始化x成员
Y y(); // ok,定义一个名为y的函数，形参为空，返回值类型为Y
std::vector<Y>(10); // error，vector无法调用默认构造函数创建Y对象
```

单实参调用构造函数/转换构造函数(converting constructor):  ??
- 能用一个实参调用的构造函数，定义了形参转为类类型的隐式类型转换，即在需要类类型的地方，也可以使用形参类型作为替代
- 需要多个实参的构造函数不能用于执行隐式类型转换，所以无须将其指定为explicit
- 只接受一步类型转换，从形参类型到类类型。若还存在实参到形参的类型转换，则为两步转换
- 在类内对构造函数使用explicit关键字，禁止形参到类类型的隐式类型转换
- 发生隐式类型转换的一种情况是当我们执行拷贝形式的初始化时，即使用等号=的初始化形式
- 初始化调用形式
  - 赋值符号=的初始化形式，右侧对象的类型将变转为左侧对象的类型，若符号=的两侧对象的类型相同，则不发生类型转换
  - 使用圆括号()的初始化形式，编译器做函数匹配，不发生形参类型到类类型的隐式类型转换，即不受explicit影响
- explicit关键字只能在类内使用，类外成员函数定义使用将报错
- explicit作用的转换构造函数称为显式构造函数，不会在隐式类型转换的过程被调用
- 标准库中含有单参数构造函数的类
  - 接受const char*的string, 非explicit
  - 接受一个整型容量的vector, explicit
    - 若不explicit，则会把int转为vector<int>，实参被识别为一个vector，而不是容量大小的含义
- 如没有隐式转换的需要，将单实参调用的构造函数声明为explicit
- 对同类拷贝构造函数、移动构造函数、不同类的构造函数分别使用explict，对于以等号=形式的初始化形式的影响同上，无法隐式类型转换
- 是否需要从形参类型到类类型的转换依赖于我们对用户使用该转换的看法。某些情况，这种转换可能是对的，但类类型不是总有效，替代类类型的形参的值可能对于类类型而言，可能是无意义的
  
```cpp
class Sale_data {
public:
    Sale_data() = default;
    Sale_data(const std::string &s); // explicit Sale_data(const std::string &s);
    Sale_data& combine(const Sale_data &item) {
        str += item.isbn(); // 常量对象只能调用常量成员函数
        return *this;
    }
    std::string isbn() const { return str; }
private:
    std::string str;
};

inline Sale_data::Sale_data(const std::string &s) { str = s; }

int main() {
    // 隐式类型转换
    Sale_data item;
    std::string str = "abc";
    item.combine(str); // 用string替代Sale_data
                       // 编译器用str构造了Sale_data的临时对象
                       // 若加上explicit修饰 Sale_data(const std::string&)
                       // 则无法使用string替代Sale_data

    // 只允许一步类型转换
    item.combine("abc"); // 字符串字面值为const char*类型 or char[4]类型
                         // 编译器先用"abc"构造了string的临时对象
                         // 然后编译器还需要用string的临时对象构造Sale_data
                         // 两步隐式转换，编译器报错
    item.combine(string("abc")); // 显式转换为string，隐式将string转为Sale_data
    item.combine(Sale_data("abc")); // 隐式转为string，显式将string转为Sale_data

    // 显示类型转换
    Sale_data item_1 = Sale_data(str); // 等号=左右侧运算对象类型相同，不会发生类型转换
    item_1.combine(Sale_data(str)); // 显示构造Sale_data临时对象
    item_1.combine(static_cast<Sale_data>(str)); // static_cast创建了一个Sale_data临时对象
}
```

另一个转换构造函数的例子：
```cpp
class String {
public:
 explicit String(int n){ }  //预先分配一个n个字节的字符串
 String(const char* s){ }   //从字符串s初始化一个String类
};

int main() {
    String s1 = 'a'; // error, explicit声明，让char无法隐式转为int，避免意外的调用错误
    String s2 = String(10); // ok, 除隐式类型转换外，其他操作不影响
}
```

再一个例子:
```cpp
class A { };
 
class B {
public:
  // conversion from A (constructor):
  explicit B (const A& x) {
	  std::cout << "B's constructor" << std::endl;
  }
  B(const B &x) {
      std::cout << "B's copy constructor" << std::endl;
  }
  B(const B &&x) {
      std::cout << "B's move constructor" << std::endl;
  }
  // conversion from A (assignment):
  B& operator= (const A& x) {
	  std::cout << "B's assignment" << std::endl;
	  return *this;
  }
  // conversion to A (type-cast operator)
  operator A() {
	  std::cout << "B's conversion" << std::endl;
	  return A();
  }
};
 
int main ()
{
    A foo;
    B bar_1(foo);
    B bar_2 = static_cast<B>(foo); // 用foo创建B类型的临时对象

    B bar_4(bar_1);
    B bar_3 = bar_1; // 同类型，无发生类型转换

    B bar_5(std::move(foo));
    B bar_6 = std::move(foo); // 隐式类型转换
}
```


## 拷贝、赋值、析构函数

- 类除了可以控制对象初始化时做什么，还可以控制拷贝、赋值、移动和销毁时做什么
- 类通过五个特殊的成员函数来控制这些操作：
  - 构造函数
    - 拷贝构造函数
    - 移动构造函数
  - 赋值函数
    - 拷贝赋值函数
    - 移动赋值函数
  - 析构函数
- 拷贝构造和移动构造定义了，用同类对象初始化时会做什么
- 拷贝赋值和移动赋值定义了，用同类对象赋值时会做什么
- 析构函数定义了，当对象销毁时会做什么
- 以上这五个特殊的成员函数，拷贝构造操作(copy control)
- 如果类没有定义这些成员函数，编译器将隐式定义缺失的函数
- 若类定义了相应的非0实参调用的拷贝控制函数，则编译器不会隐式定义。如需，使用default关键字显式告诉编译器
- 编译器隐式定义的拷贝控制函数是0实参调用的默认函数，default关键字仅作用在0实参调用的拷贝控制函数
```cpp
class Sale_data {
    // 隐式定义
}；

// 编译器隐式定义的拷贝控制函数的等价形式
class Sale_data {
public:
    Sale_data(); // 默认构造
    Sale_data(const Sale_data&); // 拷贝构造
    Sale_data(const Sale_data&&);// 移动构造
    Sale_data& operator=(const Sale_data&); // 拷贝赋值
    Sale_data& operator=(const Sale_data&&);// 移动赋值
    ~Sale_data(); // 析构
};

// 隐式定义函数的调用形式
int main() {
    Sale_data item; // 默认构造
    Sale_data item_1(item); // 拷贝构造
    Sale_data item_2(std::move(item));  // 移动构造
    item_1 = item2; // 拷贝赋值
    item_1 = std::move(item_2); // 移动赋值
}；

// default的情况
```

使用建议：
- 若需要自定义析构函数，通常也需要自定义拷贝构造函数和拷贝赋值函数
- 若需要自定义拷贝赋值或拷贝构造函数，通常两者需要一起定义

拷贝构造函数：
- 构造函数的第一个参数是自身类型的引用，额外参数有实参默认值
- 若第一个参数非自身类型的引用，当用户代码用实参调用拷贝构造函数时，将再次调用拷贝构造函数把实参拷贝到形参，如此递归调用，将耗尽栈空间
- 拷贝构造用于初始化非引用类型对象，因引用非对象，引用初始化只能执行绑定，无法拷贝成员
- 默认的拷贝构造函数
  - 对于内置类型的成员，直接拷贝
  - 对于数组，逐个数组元素拷贝
  - 对于类类型成员，调用它自己的拷贝构造函数进行拷贝
- 拷贝构造函数通常不应声明为explicit
  - 直接初始化形式()，要求编译器进行构造函数匹配，来选择最匹配的函数
  - 拷贝初始化形式=，先将右侧运算对象转换为左侧运算对象类型，再拷贝到正在创建的左侧对象
  - 拷贝初始化有时由拷贝构造函数来完成，有时由移动构造函数来完成
- 什么情况会发生拷贝构造
  - 定义并初始化语句
  - 函数参数传递及函数返回
  - 列表初始化数组
```cpp
// 隐式定义的拷贝构造函数的等价形式
Sale_data(const Sale_data &orig) :
    bookNo(orig.bookNo);
    units_sold(orig.units_sold);
    revenue(orig.revenue)
    { }
```

拷贝赋值运算符：
- 左侧运算符对象为绑定到隐式this指针，右侧运算对象为同类型的参数，返回结果为作左侧对象的引用
- 将运算对象右侧，除static成员外的其他成员赋给左侧对象的对应成员
```cpp
// 隐式定义的拷贝赋值函数的等价形式
Sale_data& operator=(const Sale_data &rhs) {
    bookNo = rhs.bookNo;
    units_sold = rhs.units_sold;
    revenue = rhs.revenue;
    return *this;
}
```

析构函数：
- 释放对象所使用的资源，并销毁对象的非static成员，与构造函数动作顺序相反
- 析构函数的函数体执行对象销毁前的收尾工作，通常是释放对象在生存期分配的所有资源
- 成员是在析构函数体之后隐含的析构阶段被销毁的，析构函数体并不直接销毁成员
- 析构对象/销毁成员的动作是隐式的，无法自定义
  - 对于内置类型成员，什么也不做
  - 对于类类型成员，调用它自己的析构函数
- 指针成员是内置类型。若指针成员指向资源，析构函数不会自动释放指针所指向的资源
- 当指向一个对象的指针离开作用域时，析构函数会销毁指针本身，但不会所调用指向对象的析构函数来析构对象
- 若栈上的指针或引用指向或绑定到堆上的对象，当栈收回时，指针和引用被销毁，但堆上对象仍存在，可以使用delete来销毁堆上的对象
- 什么时候调用析构函数
  - 对象离开作用域时
  - 类对象被销毁时，其成员逐个销毁
  - 容器被销毁，其元素被销毁
  - 对指针应用delete运算符
```cpp
{
    Sale_data *p = new Sale_data;
    auto p2 = make_shared<Sale_data>();
    Sale_data item(*p);
    vector<Sale_data> vec;
    vec.push_back(*p2);
    delete p;
}
```

使用default和delete:
- 使用default显式要求编译器生成合成的版本。类内default将是inline，类外可以default
- 只能对具有合成版本的成员函数使用default
- 使用delete将函数定义为删除(deleted)来阻止被调用
- delete可以用于所有函数，不限于合成版本的函数
- 显示delete合成版本的函数:
  - delete默认构造 -> 默认构造被delete
  - delete拷贝构造 -> 隐式移动构造也被delete
  - delete拷贝赋值 -> 隐式移动赋值也被delete
  - delete移动构造或赋值 -> 隐式拷贝和移动的拷贝构造和赋值都被delete
  - delete析构函数 -> 隐式默认构造、拷贝构造和移动构造被delete
- 编译器隐式delete合成版本的函数:
  - 对象成员显式delete或private拷贝控制合成版本的函数 -> 类的合成版本函数受对象成员的影响
  - 未使用类内初始值的引用成员或无默认构造函数的const成员 -> 隐式默认构造被delete
  - 类内初始化的引用成员或const成员 -> 隐式拷贝移动赋值被delete
  - 当不可能拷贝、赋值或析构类成员时，类的相应合成拷贝控制函数就会被delete
```cpp
// 使用private来阻止函数被调用的方式，只能阻止用户代码
// 对于成员函数和友元，private的函数只声明不定义，当调用时会发生链接报错
class PrivateCopy {
    PrivateCopy(const Private&);
    PrivateCopy operator=(const Private&);
public:
    PrivateCopy() = default;
    ~PrivateCopy();
};
```

使用建议：
- 使用delete，而不是private
- 在成员函数的第一次声明使用delete，在编译器就会在第一次调用之前读到delete声明


拷贝控制与资源管理：
- 通过自定义拷贝控制操作，让类的行为看起来像一个值或一个指针
  - 像值，类有它自己的状态，即在拷贝时，创建一个副本，拷贝值。如标准库容器和string
  - 像指针，类共享状态，即在拷贝时，使用相同的底层数据。如shared_ptr
  - 不像值也不像指针，不允许拷贝和赋值。如I/O类型和unique_ptr
- 像值
  - 构造函数申请分配堆内存
  - 赋值函数深拷贝，拷贝内容，留心自赋值
  - 析构函数释放申请的堆内存
- 像指针
  - 构造函数初始化资源引用计数
  - 赋值函数检查和调整引用计数
  - 析构函数检查引用计数，当引用计数为0时释放资源
- 赋值操作须留心自赋值安全，一个好的方法是在左侧对象被销毁前，拷贝右侧运算对象
- 大多数赋值运算符组合了析构函数和拷贝构造函数的工作
- 对于资源管理类，自定义swap函数优化赋值操作
- 另一个类展现出指针行为的最好方法是使用shared_ptr来管理资源
- 若希望直接管理资源，使用引用计数
- 使用拷贝和交换的赋值运算符自动就是异常安全的，且能正确处理自赋值
- 标准库容器负责资源管理 vs 直接管理资源
  - 管理动态内存的类，通常不能依赖编译器创建的默认拷贝、赋值和析构函数
  - 如果类包含vector或者string成员，则编译器创建的拷贝、赋值和析构函数可以正常工作
```cpp
// 像值
class X {
    std::string *pstr;
public:
    X(const std::string &s = " ") {
        pstr = new std::string(s);
    }
    X(const X &x) {
        pstr = new std::string(*x.pstr);
    }
    X& operator=(const X &x) {
        std::string tmp(*x.pstr); // 先拷贝右侧运算对象，防自赋值
        delete pstr;
        pstr = new std::string(tmp);
        return *this;
    }
    ~X() {
        delete pstr;
    }
    friend std::ostream& print(std::ostream &os, const X &x);
};

std::ostream& print(std::ostream &os, const X &x) {
        os << x.pstr << " "  << *x.pstr << '\n';
        return os;
}

int main() {
    X x1;
    print(std::cout, x1);
    X x2("ab");
    print(std::cout, x2);
    X x3(x2);
    print(std::cout, x3);
    x1 = x2;
    print(std::cout, x1);
    x1 = x1;
    print(std::cout, x1);
}
```

```cpp
// 像指针
class X {
    std::string *pstr;
    std::size_t *user;
public:
    X(const std::string &s = " ") {
        pstr = new std::string(s);
        user = new std::size_t(1); // 不能使用static成员，否则新的string的对象的引用计数不为1
    }
    X(const X &x) {
        pstr = x.pstr;
        user = x.user; // 指向同一个引用计数，实时更新
        ++ *user;
    }
    X& operator=(const X &x) {
        ++ *x.user; // 先递增右侧运算对象的引用技术，防自赋值
        if (--*user == 0) {
            delete pstr;
            delete user;
        }
        pstr = x.pstr;
        user = x.user;
        return *this;
    }
    ~X() {
        if (--*user == 0) {
            delete pstr;
            delete user;
        }
    }
    friend std::ostream& print(std::ostream &os, const X &x);
};

std::ostream& print(std::ostream &os, const X &x) {
        os << x.pstr << " "  << *x.user << '\n';
        return os;
}

int main() {
    X x1;
    print(std::cout, x1);
    X x2("ab");
    print(std::cout, x2);
    X x3(x2);
    print(std::cout, x3);
    x1 = x2;
    print(std::cout, x1);
    x1 = x1;
    print(std::cout, x1);   
}
```
```cpp
// std::swap，一次拷贝构造，两次赋值
void std::swap(X &lhs, X &rhs) {
    X tmp = lhs; // new内存
    lhs = rhs; // 释放再new内存
    rhs = tmp; // 释放再new内存
}

// self-define swap，避免不必要的内存分配
void swap(X &lhs, &rhs) {
    std::string *tmp = lhs.pstr;
    lhs.pstr = rhs.pstr;
    rhs.pstr = tmp;
    // std::swap(lhs.pstr, rhs.pstr); // std::swap用来交换内置类型对象
    // std::swap(lhs.user, rhs.user);
}

// 用swap实现operator=，拷贝并交换技术
X& operator=(X x) {
    swap(*this, x); // 将要释放的内存给形参x，当x退出作用域时自动释放
                    // 若是自赋值，形参x暂存内容，swap里的赋值操作可以正确工作
                    // 若new抛出异常，会在代码改变左侧对象之前发生，不会修改左侧对象
                    // 当异常发生时，能将operator=左侧对象置于一个有意义的状态，则是异常安全
    return *this;
}
```

动态内存管理类：
- 某些类需要在运行时分配可变大小的空间。可以使用标准库容器，也可以自己定义拷贝控制成员来管理所分配的内存
- 使用alloctor来分配未初始化的内存和在这些内存上构造对象，realloctor负责扩充容量
  
```cpp

StrVec(const StrVec &s) {
    auto new_data = alloc_n_copy(s.begin(), s.end());
    elements = new_data.first;
    finish = capacity = new_data.second;
}

StrVec& operator=(const StrVec &rhs) {
    auto data = alloc_n_copy(rhs.begin(), rhs.end()); // 分配新内存，再释放就内存
    free();
    elements = data.first;
    finish = capacity = data.second;
    return *this;
}

void reallocate() {
    auto new_capacity = size() ? 2 * size() : 1;
    auto new_data = alloc.allocate(new_capacity);
    auto dest = new_data;
    auto elem = elements;
    for (size_t i  = 0; i != size(); ++i) {
        alloc.construct(dest++, std::move(elem++)); // move not copy
    free();
    elements = new_data; // start
    finish = dest; // finish
    capacity = elements + new_capacity; // end
    }
}

pair<string*, string*> alloc_n_copy(const string *b, const string *e) {
    auto data = alloc.allocate(e - b); 
    return {data, uninitialized_copy(b, e, data)}; // 返回新内存的起始和末尾位置
}

void free() {
    if (elements) {
        for (auto p = finish; p != elements; /*void*/ ) {
            alloc.destroy(--p); // 调用容器元素的析构函数
        }
        alloc.deallocate(elements, capacity - elements);
    }
}

~StrVec() { free(); }

```

## 移动构造/赋值函数
- 变量的名字与存储空间
  - 变量名映射到一个内存地址，编译器将变量名转为内存地址来实现对内存的读写
  - 内存地址对应一个存储空间，里边存放变量的值
  - 对于没有名字的变量
    - 在程序中，可以用保存变量内存地址的指针，来间接读写变量的存储空间
    - 也可以用右值引用给匿名变量取个名字，直接用变量名读写变量的存储空间
- 左值与右值
  - 在赋值运算符左侧的对象为左值，右侧的运算对象为右值
  - 命名的对象或变量，既是左值也是右值。当使用变量名(内存地址)是左值，使用变量的值是右值
  - 匿名的对象是右值，如临时对象和字面值
    - 产生临时对象的方式
      - 类型转换
      - 赋值运算
      - 函数返回
      - 表达式求值
      - 显式构造函数调用
- 左值引用与右值引用(引用就是取名字)
  - 左值引用，绑定的左值。例外地，常量左值引用可以绑定到右值
  - 右值引用，绑定到右值。例外地，右值引用可以绑定到左值类型转换的右值结果
- 拷贝操作与移动操作
  - 拷贝一个对象，结果有两个相同的对象，两份存储空间
  - 移动一个对象给另一个对象，结果两个对象只有一份存储空间
  - 拷贝操作和移动操作的操作对象都是存储空间，移动操作将存储空间从一地移动到另一地，拷贝则是再分配一份存储空间
  - 如果类型不支持移动操作，编译器将移动操作转为拷贝操作
  - 如果定义了拷贝操作，没定义移动操作，则编译器不会隐式定义移动操作
  - 移动操作需确保，对象被移动后（移后源），处于有效的、可析构的状态
- 拷贝赋值运算符与移动赋值运算符
  - 拷贝将右侧对象的值复制一份给左侧对象，不会改变右侧对象的值
  - 移动将右侧对象的值转给左侧对象，会改变右侧对象的值，这要求右侧对象没有其他用户使用它
  - 由于一个被移动后的对象具有不确定状态，对其调用std::move是危险的，小心地使用std::move
  - 标准库容器出于异常安全保证，若元素的移动操作可能抛出异常(先修改右侧的做法)，则调用拷贝操作
  - 使用noexcept声明函数不抛出异常，类外定义也需要标记noexcept
  - 编译器在下列情况，不会合成移动构造函数和移动赋值函数
    - 对象成员不支持移动操作
    - 显示定义了拷贝构造函数和拷贝运算符
- 右值引用与成员函数
  - 引用限定符修饰成员函数的this形参，表示根据对象的左右值来选择调用对应的版本
  - 若一个成员函数使用引用限定，则同名同形参表的函数也需要使用引用限定符
  - const修饰符也修饰形参this， const修饰符位于引用修饰符之前
```cpp
int i = 1;
int &ref = i * 42; // error， 算术运算结果为右值

StrVec(StrVec &&s) noexcept
    : begin(s.begin), finish(s.finish), end(s.end) {
        s.begin = s.finish = s.end = nullptr;
    }

StrVec& operator=(StrVec && rhs) noexcept {
    if (this != &rhs) { // 处理自赋值
        free();
        begin = rhs.begin;
        finish = rhs.begin;
        end = rhs.end;
        rhs.begin = rhs.finish = rhs.end = nullptr;
    }
    return *this;
}

// 移动迭代器，迭代器解引用返回为右值引用类型的对象
auto last = uninitialized_copy(make_move_iterator(begin(),
                               make_move_iterator(end(),
                               first);

// 重载右值引用版本的成员函数
void push_back(const std::string &s) {
    chk_n_alloc(); 
    alloc.construct(finish++, s);
}
void push_back(std::string &&s) {
    chk_n_alloc(); // 内存扩充
    alloc.construct(finish++, std::move(s)); // 名字是左值

Foo Foo::sorted() const & {
    Foo ret(*this);
    sort(ret.data.begin() ,ret.data.end());
    return ret;
}
Foo Foo::sorted() && { 
    sort(ret.data.begin(), ret.data.end()); // 对象是右值，就地排序
    return *this;
}

lvalue.sorted();
rvalue.sorted(); // rvalue可能为临时对象
```

使用建议：
- 仅将移动操作用在即将销毁的对象
- 当需要显式定义五个拷贝控制成员中的某一个时，最好将五个拷贝控制成员都显式定义
- 仅对将要销毁的对象使用std::move操作


## 访问控制与封装

访问权限控制：
- c++使用访问说明符(access specifiers)来对类进行封装，使用户不能直接访问到类的实现
- public说明符定义了类的接口，定义在public之后的名字可以被外部访问
- private说明符封装了类的实现细节，定义在private之后的名字可以被类的成员函数访问
- 一个类可以包含n个说明符，每个说明符指定访问级别，其有效范围直到下一个说明符或作用域结尾
- struct关键字默认访问权限为public，class关键字默认访问权限为private

封装的作用：
- 确保用户代码不会无意间破坏封装对象的状态
  - 当对象的状态被意外修改时，只有实现部分的代码可能产生这样的错误
- 被封装的类的具体细节可以随时改变，不影响用户级别的代码
  - 若改动了类的实现，使用了类的实现的源文件需要重新编译

友元：
- 类允许它的友元类和友元函数访问它的非公有成员，在类内使用friend关键字进行声明
- 友元不是类的成员，不受所在区域访问控制级别的约束。友元声明在类内出现的位置不确定
- 友元声明仅指定了访问了的权限，非通常意义上的函数声明，为了使用它，还需在类外进行独立的函数声明
- 为了友元对类的用户可见，通常把友元的独立函数声明和类本身放置在同一个头文件中
- 在类定义的开始或结束前的位置集中声明友元
- 友元可能改变对象的状态，当对象的状态被意外改变时，可能是成员函数修改的，也可能是友元修改的，友元可能使定错错误变得困难
- 友元类的所有成员函数都可以访问类的所有成员，更加难以确定是哪一个友元类的成员函数修改了对象的状态
- 友元关系不存在传递性，每个类负责控制自己的友元类和友元函数
- 在类内把一组重载函数声明为友元，需对这组函数中的每一个分别声明friend
- 类和非成员函数的声明不是必须在友元声明之前
- 友元声明不能取代真正意义的函数声明，友元声明 vs 函数声明
- 友元声明 vs 类成员声明
```cpp
class Sale_data 
    std::string bookNo;
    unsigned units_sold = 0; // 售出数量
    double revenue = 0.0; // 营业额
public:
    Sale_data() = default;
    Sale_data(const std::string &s): bookNo(s) { }
    Sale_data(std::istream &is) { read(is, *this); }

    friend std::istream& read(std::istream &is, Sale_data &item); // 友元声明
};

std::istream& read(std::istream &is, Sale_sata &item) { // 独立函数声明
    is >> item.bookNo >> item.units_sold >> item.revenue;
    return is;
}
```

## 类成员

类型成员：
- 使用typedef或using声明类型别名，先定义后使用，通常位于类的开始
- 使用类型别名成员，有助于封装，用户无法知晓具体的类型细节
  
可变数据成员：
- 将数据成员声明为mutable，使得可以在const成员函数中修改该成员的值
- 可用于统计某个const函数被调用的次数

静态成员：
- 参与对象的内存布局，不存在任何一个对象中
- 在类内声明静态成员的名字，对象可以使用静态成员的名字
- 可以通过类作用域或对象访问到静态成员

inline成员函数：
- 定义在类内，默认为inline
- 定义在类外，可以显式声明为inline
  
数据成员的类内初始值：
- 使用=或{}来给定类内初始值

不完全类型：
- 声明类但不定义它，它是一个不完全类型，不清楚它到底包含哪些成员，编译器无法了解类对象需要多少存储空间
- 不完全类型使用场景有限，可以定义指向不完全类型的指针或引用，也可以声明以不完全类型作为参数或返回值的函数
```cpp
class Link_screen {
    Screen windows;
    Link_screen *next;
    Link_screen *prev;
};

// 交叉引用
class Y;
class X { Y *py; };
class Y { X x; };
```

类成员指针:
- 访问控制
- 数据成员
- 成员函数
```cpp
class A {
public:
    A(const std::string s): str(s) { }
    void print() { std::cout << "triavl function\n"; }
    virtual void output() { std::cout << "virutal function\n"; }
    static void out() { std::cout << "static function\n"; }
    std::string str;
    static int number;
};

int A::number = 1;

int main() {
    // 非static数据成员指针
    std::string (A::*data_ptr) = &A::str;
    std::cout << "offset in class: " << data_ptr << '\n'; // 1
    A obj("a");
    std::cout << "value in specific_object: " << (obj.*data_ptr) << '\n';

    // 非static非virtual成员函数指针
    void(A::*func_ptr)() = &A::print;
    std::cout << "offset in class: " << func_ptr << '\n'; // 1
    (obj.*func_ptr)();

    // virtual成员函数指针
    func_ptr = &A::output;
    std::cout << "offset in virtual function table: " << func_ptr << '\n'; // 1
    (&obj->*func_ptr)();

    // static数据成员指针
    int *static_data_ptr = &A::number;
    std::cout << "offset in address space: " << static_data_ptr << '\n'; // 0x4040a8
    std::cout << "value in class: " << *static_data_ptr << '\n';

    // static成员函数指针
    void (*static_func_ptr)() = &A::out; // void(*)() vs void(A::*)()
    std::cout <<"offset in address space: " << static_func_ptr << '\n'; // all value is 1 ??
    static_func_ptr();
}
```


## 委托构造函数

- 一个委托构造函数(delegating func)使用它所属的其他构造函数来执行它自己的初始化过程。
- 当一个构造函数委托给另一个构造函数时，受委托的构造函数其初始化列表和函数体被依次执行
- 当受委托函数执行完后，将控制权转移给委托者，委托者继续执行其函数体
```cpp
class Sale_data {
public:
    Sale_data(std::string s = "ok", unsigned cnt = 1, double price = 1.0): s(s), cnt(cnt), price(price) {
        std::cout << "delegating func";
     }
    Sale_data(std::ostream &out): Sale_data() { 
        out << s << " " << cnt << " " << price; 
     }
private:
    std::string s;
    unsigned cnt;
    double price;
};
```

## 重载运算符函数

- 通过重载运算符的方式，将作用对象扩充到类类型
- 重载运算符是一个特殊函数，作用对象主要为类类型对象
- 重载运算符应保持与内置运算符相同的语义，不与使用习惯冲突
- 重载运算符无法改变运算符的优先级、结合律和运算对象个数
- 不能被重载的运算符：
  - 作用域访问运算符 ::
  - 成员访问运算符 .
  - .*
  - 条件运算符 ?:
- 不应该被重载的运算符
  - 逗号运算符 ,
  - 取地址运算符 &
  - 逻辑与运算符 &&
  - 逻辑或运算符 ||
- 什么情况将操作定义为重载运算符
  - 当操作与运算符的逻辑语义密切相关或相同时，否则定义为函数
- 什么时候定义将重载运算符定义为类的成员
  - 运算符的左侧对象必须是类类型对象时，运算符定义为类成员，对象的this指针绑定到运算符的第一个参数
  - 运算符改变对象状态
  - 对称性运算符的左右侧对象可以随意交换位置，定义为非类成员
  - 必须是成员
    - 赋值运算符 = 
    - 下标运算符 []
    - 调用运算符 ()
    - 成员访问箭头 ->
  - 通常应该是成员
    - 会改变对象的状态
    - 复合赋值运算符
    - 解引用运算符
    - 递增递减运算符
  - 通常应该是非成员
    - 算术运算符
    - 关系运算符
    - 等于运算符
  

输出运算符：
- 重载左移位运算符为输出运算符
- 第一个参数为流对象的非常量引用，第二个参数为类类型的常量引用，返回流对象的引用
- 若需要访问类的非公成员，需将输出运算符声明为类的友元

```cpp
ostream& operator<<(ostream &os, const Sale_data & item) {
    os << item.bookNo << " " << item.units_sold << " " 
       << item.revenue << " ";
    return os;
}
```

输入运算符：
- 重载右移位运算符为输入运算符
- 第一个参数为流对象的引用(流对象不可拷贝)，第二参数为类类型的非常量引用(需要被改变)，返回流对象的引用
- 将输入运算符声明为类的友元
- 输入流可能会出错，通过检查流的状态来判断
  - goodbit为1，输入操作成功
  - failbit为1，输入数据类型不匹配
  - eofbit为1，读到文件末尾
  - badbit为1，流损坏无法修复
- 当输入流出错时，应设置流状态，并在输入流运算符中做错误修复，让对象处于有效的状态
```cpp
istream& operator>>(stream &is, Sale_data & item) {
    double price;
    is >> item.bookNo >> is.units_sold >> price; // 若is读入操作失败，price可能未初始化
    if (is)  
        item.revenue = item.untis_sold * price; 
    else
        item = Sale_data(); // 赋默认值，修复输入错误
    return is;
}
```

算术运算符：
- 允许左右侧对象交换位置
- 若定义了复合赋值运算符，通常用它来实现算法运算符
```cpp
Sale_data operator+(const Sale_data &lhs, const Sale_data &rhs) {
    Sale_data sum = lhs;
    sum += rhs;
    return sum;
}
```

相等运算符：
- 当两个对象的所有成员都相等时，两个对象相等
- 若某个类在逻辑上有相等性的含义，就应该定义operator==
```cpp
bool operator==(const Sale_data &lhs, const Sale_data &rhs) {
    return lhs.isbn() == rhs.isbn() &&
           lhs.units_sold = rhs.units_sold &&
           lhs.revenue = rhs.revenue;
}

bool operator!=(const Sale_data &lhs, const Sale_data &rhs) {
    retrun !(lhs == rhs);
}
```

关系运算符：
- 用于定义顺序关系
- 如果两个对象哪一个都不比另一个小，那么这两个对象应该是相等的
- 如果两个对象满足!=，那么一个对象应该小于另一个对象
- 若类已定义了相等不等运算符，那么可能也需要定义关系运算符，使其相等不等运算符逻辑保持一致

赋值运算符：
- 右侧运算对象可以是类类型或其他类型，其他类型则无须处理自赋值问题
- 当右侧对象为初始值列表类型时，实现支持花括号的值初始化形式
```cpp
StrVec& operator=(std::initializer_list<std::string> il);
```

复合赋值运算符：
- 与内置类型运算符保持一致，返回左值引用
```cpp
Sale_data& operator+=(const Sale_data &rhs) {
    units_sold += rhs.units_sold;
    revenue += rhs.revenue;
    return *this;
}
```

下标运算符：
- 通过指定位置访问容器中的元素，返回左值引用
```cpp
std::string& operator[](std::size_t n);
const std::string& operator[](std::size_t n) const; // 返回常量引用，无法作为赋值号左侧对象
```

递增递减运算符：
- 用来在元素序列中前后移动
- 后置版本的形参int仅用来区分，不命名不使用

成员访问运算符：
- 解引用运算符，返回指针或迭代器所指向元素的引用
- 箭头运算符用于访问成员，不执行自己的操作，转调用解引用运算符，返回类指针
```cpp
std::string& operator*() const;

std::string* operator->() const {
    return &(this->operator*()); // &*this
}

p->size();
(*p).size();
```

调用运算符：
- 定义了调用运算符的类对象，称为函数对象，行为像函数一样，比函数更灵活，可以存储运行时函数状态
- 函数对象的数据成员，被用于函数调用运算符中的操作
- 常将函数对象传递给泛型算法，用于改变算法的策略/行为
```cpp
void operator()(parameter_list);
```

lambda函数对象：
- 闭包类型的匿名函数对象
- 默认为const修饰，即不改变捕获的变量
- 值捕获的变量，编译器会在类内隐式定义对应的数据成员，并隐式定义构造函数来完成初始化
- 仅当满足如下所有情况适宜用lambda
  - 未来不需要改动lambda逻辑
  - 逻辑简短
```cpp
auto word_count = find_if(words.begin(), words.end(), 
                    [sz](const string &a)
                        { return s.size() >= sz; });
```

标准库定义的函数对象：
- 均为模板类，定义在<functinoal>头文件
- 算术
  - plus
  - minus
  - multiplies
  - divides
  - modules
  - negate
- 关系
  - equal_to
  - not_equal_to
  - greater
  - greater_equal
  - less
  - less_equal
- 逻辑
  - logical_and
  - logical_or
  - logical_not
```cpp
sort(vec.begin(), vec.end(), greater<string>());
```

可调用对象：
- 调用形式对应一个函数类型 - retType(param)
- 不同语法类型的可调用对象
  - 函数
  - 函数指针
  - lambda
  - 重载调用运算符的类
  - bind创建的对象
- 不同语法类型的可调用对象可能具有相同的调用形式(函数类型)
- 使用function类型来表示函数类型，用以表示有相同调用形式的可调用对象
- function类型的类型成员
  - result_type
  - argument_type
  - first_argument_type
  - second_argument_type
```cpp
#include <functional>

function<T> f(called_obj); // 存储可调用对象的拷贝
function<T> f(nullptr); 
function<T> f; // T->retType(args)
while (f); // 当f含有一个可调用对象时，为真
f(args); // 函数调用

// 定义函数表，根据符号查表调用
map<std::string, function<int(int, int)>> func_table = {
    {"+", add}, // add函数指针
    {"-", std::minus<int>()},
    {"/", divide()},
    {"*", [](int i, int j){return i * j;}},
    {"%", mod} }; // 命名的lambda

func_table["+"](10, 5);

// 重载函数的二义性问题
int add(int, int);
Sale_data add(const Sale_data&, const Sale_data&);
func_table.insert( {"+", add} ); // error，函数名存在二义性调用

int (*fp)(int, int) = add;
func_table.insert( {"+", fp} );
func_table.insert( {"+", [](int a, int b){return add(a,b);} } );
```

类型转换运算符：
- 定义了从类类型到其他类型的转换，无返回值类型和形参，通常为const
- 单参数构造函数隐式定义了其他类型到类类型的转换
- 对类型转换使用explicit修饰，避免隐式类型转换发生错误
- 会产生调用二义性的类型转换定义
- 
```cpp
class X {
public:
    explicit operator int() const { return 1; }
    explicit operator bool() const { return true; }
};

X x;
x + 2; // error, 禁止隐式类型转换
static_cast<int>(x) + 2; // ok，结果为3
while (x) ; // error, 发生隐式类型转换
```
