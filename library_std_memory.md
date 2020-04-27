

## ScopeGuard

- RAII机制，在构造函数中获取资源，在析构函数中释放资源
- ScopeGuard作用是确保资源总能被成功释放
  - 当函数中途返回或中途发生异常，返回点后的资源释放代码被跳过没有执行
  - 当函数正常结束时，ScopeGuard不会重复释放资源
  - 只有当函数非正常返回时，ScopeGuard才会在析构时释放资源
  - 根据是否正常退出来确定是否需要释放资源
- 简而言之，不要同时使用显式资源释放和RAII机制，共享资源使用条件释放，独享资源直接释放
```cpp
{
    auto gd = make_guard(f);
    ...
    //正常退出
}
{
    auto gd = make_guard(f);
    ...
    throw std::exception("");
    // 异常退出
}
{
    auto gd = make_guard(f);
    return;
    ... // 非正常退出，释放资源代码可能被跳过
}
```


## 智能指针

- 显式堆内存管理容易出现未正确分配和释放问题（栈分配内存自动分配和释放)
  - 野指针，一些内存单元已经被释放，但之前指向它的指针还在被使用
  - 重复释放，试图去释放已经被释放过的内存单元或释放已经被重新分配的内存单元
  - 内存泄漏，不再需要使用的内存单元没有被释放，可能导致内存使用剧增
- 使用对象的方式来管理堆分配的内存
- 是重载operator*和operator->运算符，行为像指针的对象，用来自动释放堆内存

- shared_ptr共享指针
  - 允许多个shared_ptr指针共享拥有同一堆分配的内存 （拷贝语义)
  - 使用引用计数实现，shared_ptr对象析构，引用计数减1，当资源的引用计数为0时，shared_ptr对象才释放指向的堆内存
  - 错误使用
    - 两个shared_ptr对象相互引用对方，会出现当两个shared_ptr对象都析构时，资源引用计数为1，因此两者指向的堆内存都不会被释放
    - 以同一个原生指针分别构造的两个shared_ptr对象，彼此之间没有关系，资源引用计数都为1，都两个shared_ptr对象析构时，会重复释放同一块堆内存
    - 在函数多实参中创建shared_ptr对象，当其他参数发生异常，已new堆内存但shared_ptr对象尚未完成构造，可能存在堆内存泄漏 ??
    - 
```cpp
// 初始化
std::shared_ptr<int> p(new int(1));
std::shared_ptr<int> p = new int(1); // error，编译报错
std::shared_ptr<int> p2 = p; // shared_ptr的拷贝指向相同的内存
std::shared_ptr<int> ptr;
ptr.reset(new int(1)); // 若ptr为空，则初始化ptr； 否则，ptr的原资源引用计数减少1，并将资源引用计数初始化为1，指向新的堆内存

// 获取原始指针
std::shared_ptr<int> ptr(new int(1));
int* raw_ptr = ptr.get();

// 指定删除器，默认为删除单个对象
std::shared_ptr<int> p( new int[10], std::default_delete<int[]>() );

template <typename T>  // 封装shared_ptr管理动态数组，需要指定删除器的调用
std::shared_ptr<T> make_shared_array( size_t size ) {
    return std::shared_ptr<T>( new T[size], default_delete<T[]>() );
}
std::shared_ptr<int> p = make_shared_array<int>(10);

// 错误1: 用一个原始指针初始化多个shared_ptr对象
int *ptr = new int;
std::shared_ptr<int> p1( ptr ); // 引用计数1
std::shared_ptr<int> p2( ptr ); // 引用计数1

// 错误2： 在函数实参中创建shared_ptr
func( shared_ptr<int>( new int(), g() ); // 可能先new后g()，恰好g()抛出异常，此时shared_ptr对象尚未创建

std::shared_ptr<int> p( new int() );
func( p, g() );  // ok，先创建shared_ptr

// 错误3：直接返回对象(this)的shared_ptr指针
struct A {
    std::shared_ptr<A> self() {
        return std::shared_ptr<A>(this); // 新的shared_ptr对象，效果同用一个指针分别创建两个shared_ptr对象
    }
};
int main() {
    std::shared_ptr<A> sp1(new A); // 引用计数为1
    std::shared_ptr<A> sp2 = sp1->self(); // 引用计数为1
}

class A: public std::enable_shared_from_this<A>
{
    std::shared_ptr<A> self() {
        return shared_from_this(); // enable_shared_from_this<T>的虚函数
    }
}；
std::shared_ptr<A> sp1(new A); // 引用计数为1
std::shared_ptr<A> sp2 = sp1->self(); // 引用计数为2，sp1->selft()返回sp1

// 错误4： 循环引用
struct B;
struct A {
    std::shared_ptr<B> bptr;
    ~A() { std::cout << "A was destroy"; }
};

struct B {
    std::shared_ptr<A> aptr;
    ~B() { std::cout << "B was destroy"; }
};

int main() {
    std::shared_ptr<A> spa(new A());
    std::shared_ptr<B> spb(new B());
    spa->bptr = spb; // 对象B的资源引用计数为2
    spb->aptr = spa; // 对象的资源引用计数为2
} // spa和spb析构，对象A和对象B的引用计数1，故对应的析构函数不会被调用，内存泄漏


struct B;
struct A { // 使用weak_ptr解决循环引用问题
    std::weak_ptr<B> bptr;
    ~A() { std::cout << "A was destroy" };
};

struct B {
    std::shared_ptr<A> aptr;
    ~B() { std::cout << "B was destroy"; }
};

int main() {
    std::shared_ptr<A> spa(new A());
    std::shared_ptr<B> spb(new B());
    spa->bptr = spb; // 对象B的资源引用计数为1
    spb->aptr = spa; // 对象A的资源引用计数为2
} // spa析构, A和B的引用计数都为1 spb析构，A和B的引用计数都为0
```

- unique_ptr 独占指针
  - 不允许其他智能指针共享其指向的堆内存的所有权，只能通过移动来转移所有权（移动语义)
  - 通过删除了拷贝语义的构造函数和赋值运算符函数实现独占语义
```cpp
std::unique_ptr<int> up(new int());
std::unique_ptr<int> up2 = std::move(p);

std::unique_ptr<int[]> up(new int[10]); // ok

// 指定删除器
std::unique_ptr<int, std::function<void(int*)> up( new int(), [&](int* p){delete p;} ); //1

class Deleter {
public:
    void operator()(int* p) { delete p; }
};

std::unique_ptr<int, Deleter> p(new int()); // 2
```

- weak_ptr弱引用指针
  - 用来监视shared_ptr指针对象的生命周期，没有重载operator*和operator->，不操作资源
  - weak_ptr不管理资源，weak_ptr的构造和析构不会影响shared_ptr指针对象的资源引用计数
```cpp
int main() {
    std::shared_ptr<int> sp( new int(1));
    std::weak_ptr<int> wp = sp;
    std::cout << wp.use_count(); // 返回资源引用计数
    sp.reset(new int(2));
    if ( !wp.expired() ) { // sp原先的堆内存已释放，故wp已超期
        auto sp2 = wp.lock(); // 返回wp监视的shared_ptr对象
        std::cout << *sp2;
    }
    else
        std::cout << "wp is expired";
}
```