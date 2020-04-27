
## 面向对象

面向对象(object-oriented programming)：
- 面向对象的核心思想是数据抽象、继承和动态绑定
- 用数据抽象来定义一个类型，用户只能从逻辑上访问类型接口
- 用访问控制权限/级别将接口与实现分离，实现对实现的封装
- 用继承来定义一组多态类型(相似类型)，用多态来调用这组多态类型
- 面向对象意在忽略多态类型之间的类型差异，用一种统一的方式使用一组多态类型，实现泛型/类型参数化
- 多态特性由类型转换和虚函数实现，虚函数实现动态绑定

## 基类和派生类（继承)

- 基类负责定义在层次关系中所有类共同拥有的成员，派生类负责定义各自特有的成员
- 基类中与派生类类型相关的操作，在基类中声明为virtual，表示基类希望派生类定义适合自己的版本
- 基类中与派生类类型无关的操作，直接从基类继承，其在派生类中的行为和基类的版本一致
- 派生类若需访问基类的非公成员，则在基类中将此成员声明为protected访问权限，表示只有派生类成员和派生类对象可以访问，不含派生类的友元成员。基类友元成员不会被派生类继承
- 派生类的对基类的其他访问，通过调用基类的接口进行
- 类派生列表，指出派生类从哪些基类继承而来
- 类派生列表的访问说明符，用来控制继承的基类成员(基类子对象)对派生类的用户是否可见
  - public:
  - protected:
  - private:
- 一个派生类对象包含多个组成部分，包含基类子对象、派生类子对象(派生类数据成员)，可能还包括n个间接基类的子对象，其物理内存布局不一定是连续的
- 派生类对象包含基类子对象的意义
  - 体现两者的继承关系
  - 隐式定义派生类到基类的类型转换，在需要基类的地方，可以把派生类当基类使用
  - 各自负责控制各自的成员，基类子对象的成员由基类负责，派生类构造时，调用直接基类构造直接基类成员
  - 派生类和基类子对象形成嵌套作用域，基类子对象在外层，派生类子对象在内层
- 基类到派生类的类型转换，当不访问派生类子对象成员时，是安全的。否则会在基类中找不到对应名字而失败
- 仅当使用引用或指针的方式发生类型转换 ?? 基类指针指向派生类对象，也无法访问派生类对象，访问的是派生类中的基类子对象. 将派生类对象赋值给基类对象将导致派生类对象被截断(sliced down)，只有基类子对象被赋值
```cpp
class Base { ... }
class Derived: public Derived { ... } // error, 无法从自己派生
class Derived : public Base; // error，声明不带类派生列表
class Derived; // ok

class Base;
class Derived : public Base{ } // error，要求基类已定义

class Derived final : public Base { } // 禁止从Derived继承或再派生

Derived d;
Base b(d); // 调用 Base::Base(const Base&);
b = d; // 调用 Base::operator=(const Base&);
```


## 虚函数与动态绑定

- 静态类型，指声明语句的声明类型
- 动态类型，指内存中所存放的对象的类型
- 当且仅当基类的指针或引用调用虚函数时，静态类型和动态类型可能会不同，如静态类型声明为基类，动态类型则可能为派生类
- 当上述情况发生时，动态类型由函数调用的实参来决定，由于实参的内存空间运行时才分配，故上述方式调用虚函数为运行时函数解析调用
- 在运行时解析函数调用称为动态绑定，编译时解析函数调用则为静态绑定
- 派生类的虚函数的默认实参由基类的虚函数的默认实参声明决定
```cpp
// 动态绑定/运行时绑定
int print(const Base &item);
print(base);
print(derived); // 静态类型和动态类型不一致

// 折扣类
class Bulk_quote: Quote {
public:
    Bulk_quote() = default;
    Bulk_quote(const string &book, double p, size_t qty, double disc) 
             : Quote(book, p), min_qty(qty), discount(disc) { }
    virtual double net_price(size_t cnt) const override {
        if (cnt >= min_qty)
            return cnt * (1 - discount) * price;
        else
            return cnt * price;
    }
protected:
    size_t min_qty = 0;
    double discount = 0.0;
};


class Limit_quote : public Disc_quote {
public:
    Limit_quote() = default;
    Limit_quote(string const& book, double p, size_t min, size_t max, double dist) : Disc_quote(book, p, min, dist), max_qty(max) {}

    double net_price(size_t cnt) const final override {
        if (cnt > max_qty) return max_qty * (1 - discount) * price + (cnt - max_qty) * price;
        else if (cnt >= quantity) return cnt * (1 - discount) * price;
        else return cnt * price;
    }

private:
    size_t max_qty = 0;
};

// 另一个例子
class Base {
public:
    virtual void print() {
        std::cout << "base";
    }
};

class Derived: public Base {
public:
    void print() override {
        std::cout << "derived";
    }
    void Base::print() override { // 编译报错，无法在派生类内定义Base::print
        std::cout << "derived";
    }
    void func() {
        Base::print(); // 调用结果为 base, 基类子对象内的虚函数版本不发生改变
                       // override的含义为， 派生类定义同名的基类虚函数，属派生类局部作用域 ??
                       // 逻辑上，派生类显式的虚函数定义覆盖掉了派生类的基类子对象的同名虚函数版本定义
                       // 逻辑上当以指向派生类对象的基类指针或引用调用虚函数时，访问的是派生类的基类子对象的部分
                       // 实现上呢 ?? 如何理解override
                       // virtual用于辅助实现动态绑定，即在运行时选择调用虚函数的版本
                       // 非virutal的函数调用，没有动态绑定/运行时选择的行为
    }
};

int main() {
    Derived d;
    Base &base_ref = d;
    base_ref.print();
}
```

## 纯虚函数与抽象基类

- 在类中，令虚函数=0说明其为纯虚函数(pure)，表示函数是在逻辑上是无实际意义的，实现上表示无函数定义/函数声明
- 含有纯虚函数的类为抽象基类，抽象基类负责定义接口 ?? 抽象的其他用法？？ 继承用途??
- 抽象基类用来表示概念等无实际意义的事物，无法直接创建抽象基类的对象
- 从抽象类继承的派生类，继承了抽象类的成员，只是抽象类的纯虚函数只有声明没有定义
- 重构(refactoring)，重新设计类的继承体系，将操作和数据从一个类移动到另一个类中 ??
- 重构不影响用户代码，但被改动的类的文件需编译 ??
- 纯虚函数是虚函数的一个特例，将其声明为纯虚函数，表示基类不需要有虚函数的定义
```cpp
class Abstract {
public:
    virtual void print() const = 0; // 等价于虚函数声明
};

class Concrete: Abstract {
public:
    virtual void print() const {
        Abstract::print(); // 找不到函数定义，程序链接报错
    }
};

int main() {
    Concrete obj;
    Abstract &base_ref = obj;
    base_ref.print();
}
```

## 访问控制与继承

- 派生类成员(隐式this访问)或友元只能通过派生类对象类访问基类的protected成员
- 派生访问说明符，用于控制派生类用户对于基类子对象成员的访问权限，对于派生类成员和友元访问其基类成员无影响，对基类成员的访问权限只与基类中的访问说明符有关
- 派生类向基类转换的结果是否可访问
  - 当基类子对象可见时，由派生类到基类的类型转换可以进行
  - 当派生类 public 基类时，用户代码可以执行类型转换
  - 当派生类 protected or private 基类时，派生类的成员和友元???可以执行类型转换
  - 当派生类 private 基类时，派生类的再派生类无法执行上述类型转换，因间接基类对其不可见
- 类的用户
  - 类的成员和类的友元
  - 派生类
  - 普通用户/用户代码
- 不能继承友元关系，每个类负责控制各自成员的访问权限
- 使用using声明可以改变名字的访问控制权限
- strcut默认public继承，class默认private继承
```cpp
class Derived : private Base {
public:
    using Base::size; // 将size由private改由public，意义何在 ?? 有意义的使用场景有哪些 ?
    void func() {
        std::cout << size; // 访问基类的size ?? 多此一举 ??
    }
private:
    using Base::n;
};
```

## 继承中的类作用域

- 当存在继承关系时，派生类的作用域嵌套在基类的作用域之内
- 若一个名字在派生类的作用域内没有找到，则编译器将继续在基类的作用域继续查找这个名字
- 派生类的成员将隐藏基类的同名成员，可以通过作用域运算符或using声明来使用被隐藏的基类成员
- 一条基类成员函数的using声明语句可以把该函数的所有重载实例添加到派生类的作用域中，派生类可以定义特有函数覆盖重载的函数
- 函数调用的解析过程，以p->mem()为例
  - 确定p的静态类型
    - 在p的静态类型对应的类内查找mem名字
      - 若类内查找失败，继续在外层作用域查找，直到继承链的顶端，仍失败则编译报错
      - 查找成功，编译器进行函数调用的类型检查
        - 函数调用与函数原型不符，编译报错
        - 函数调用合法，编译器检查mem是否为虚函数
          - 不是虚函数，编译器将产生常规函数调用的代码
          - mem是虚函数，检查是否通过指针或引用进行调用
            - 是，则编译器产生的代码将在运行时，依据动态类型确定该运行虚函数的哪个版本
            - 否，产生常规函数调用代码
```cpp
class Base {
public: 
    void func()  { std::cout << "0"; }
    void func(int) { std::cout << "1"; }
};

class Derived: public Base {
public:
    using Base::func; // 导入基类中的所有名为func的成员
    void func(int) { std::cout << "2"; }
};


int main() {
    Derived d;
    d.func(); // 调用Base::func()
    d.func(2); // 调用Derived::func()
}
```

使用建议：
- 除了覆盖继承的虚函数之外，派生类最好不要定义与基类中同名的名字


## 构造函数与拷贝控制

- 派生类对象的组成包含基类子对象和派生类子对象，派生类定义的拷贝控制成员也将涉及到基类的对象拷贝控制成员
- 当基类的拷贝控制成员被定义为删除，则派生类的对应成员函数也会被删除
- 派生类执行拷贝控制成员的操作，也要求基类有对应的成员可以供调用

析构函数：
- 通常在基类中，将析构函数定义为虚析构函数，确保根据动态类型执行正确地析构函数版本
- 派生类的析构函数，只负责销毁派生类自己分配的资源
- 调用派生类的析构函数时，先析构派生类子对象，然后再调用基类的析构函数析构基类子对象，基类的析构函数会自动被隐式调用
- 当执行基类的析构函数时，派生类部分已经被销毁了，对象此时处于半销毁状态
- 在析构函数中调用虚函数，若对象处于半销毁状态，调用派生类的虚函数版本，则虚函数可能会访问已经被销毁的派生类子对象，导致运行时错误，结果可能是程序崩溃
- 定义虚析构函数，不一定需要定义拷贝操作，且编译器将不会定义默认的拷贝操作，即使用default
```cpp
class Base {
public:
    virtual ~Base() = default;
};

class Derived: public Base {
    ~Derived() override { }
};
```

拷贝操作和移动操作：
- 如果要拷贝或移动基类部分，必须在派生类的拷贝或移动构造函数的初始值列表中显式调用基类的对应构造函数
- 当执行基类构造函数时，对象的派生类部分未被初始化，对象此时处于半初始化状态
- 在构造函数中调用虚函数，若对象出于半初始化状态，调用派生类的虚函数版本，访问未初始化的派生类部分可能发生错误
```cpp
class Base {  };
class Derived: public Base {
public:
    Derived(const Derived &d) { }//error, 派生类部分的值由其他对象拷贝而来，而基类部分却使用默认构造初始化(半初始化)
    Derived(const Derived &d): Base(d) { } // 显式调用基类构造函数
    Derived(Derived &&d): Base(std::move(d)) { }
};
```

使用建议：
- 在构造函数或析构函数中调用虚函数，应该调用与构造函数或析构函数对应的虚函数版本


赋值运算符：
- 需要显式为基类部分赋值，基类赋值运算符不会被自动调用
- 基类的运算符函数可以是合成版本(隐式定义)，也可以是显式定义
```cpp
Derived& operator=(const Derived &rhs) {
    Base::operator=(rhs); // 要求基类能处理自赋值，可能存在资源的释放再分配操作
    return *this;
}
```

## 继承构造函数
- 使用using声明语句，派生类可以继承直接基类定义的构造函数，编译器将为派生类生成对应的代码
```cpp
class Base {
public:
    Base(int i) { std::cout << 1; }
};

class Derived: public Base {
public:
    using Base::Base; // using作用域构造函数时，编译器将产生代码
    // Derived(int i): Base(i) { } // 继承的构造函数，由编译器生成
};

int main() {
    Derived d1(1);
}
```

## 类型转换