
# 创建模式

## 工厂模式

- 定义创建一个类对象的接口，用于管理大数量的对象的创建，避免手动创建
- 将创建对象的动作放到具体的工厂中，延迟子类的实例化 ?? 有什么用??

```cpp
class Product {
public:
    virtual ~Product() = 0;
protected:
    Product() = default; // 只能通过指定接口创建对象实例
};

class Factory {
public:
    virtual Product* create_product() = 0;
    virtual ~Factory() = 0;
protected:
    Factory() = default;
};

class ConcreteProduct: public Product {
public:
    ConcreteFactory() { std::cout << "create_product over\n"; }
}

class ConcreteFactory: public Factory {
public:
    Product* create_product() override {
        new ConcreteProduct();
    }
};


int main() {
    Factory *factory = new ConcreteFactory();
    Product *product = factory->create_product();
}
```


## 抽象工厂模式

- 定义创建多个类的对象的接口

```cpp

```


## 单例模式

- 用来创建唯一的实例
  
```cpp
class Singleton {
    static Singleton *ptr;
public:
    static Singleton* instance() {
        if (instance == nullptr) {
            ptr = new Singleton;
        }
        return ptr;
    }
private:
    Singleton() { }
};

Singleton* Singleton::ptr = nullptr;

int main() {
    Singleton obj; // error
    Singleton *p = new Singleton(); // error

    Singleton *p1 = Singleton::instance(); // ok
    Singleton *p2 = Singleton::instance();
    if (p1 == p2)
        std::cout << "equal\n";
    else
        std::cout << "no equal\n";
}
```


## 建造模式

- 用来创建大的复杂对象
- 一步步创建对象，并且通过相同的创建过程可以获得不同的结果对象
- Builder只负责构造对象，不返回构造完成的对象
- Director可以添加返回已构造完成对象的接口 ??

```cpp
#include <iostream>

class Product { };

class Builder {
public:
    virtual void build_partA(const std::string &) = 0;
    virtual void build_partB(const std::string &) = 0;
    virtual Product* get_product() = 0; // 返回怎样的对象 ??
};

class Director {
public:
    Director(Builder *ptr): build_ptr(ptr) { }
    void construct() {
        build_ptr->build_partA("Build_PartA\n");
        build_ptr->build_partB("Build PartB\n");
    }
    ~Director() {
        if (build_ptr) {
            delete build_ptr;
            build_ptr = nullptr;
        }
    }
private:
    Builder *build_ptr; // 可以是builder对象集合 ?
};

class ConcreteBuilder: public Builder {
public:
    virtual void build_partA(const std::string &str_a) { std::cout << str_a; }
    virtual void build_partB(const std::string &str_b) { std::cout << str_b; }
    virtual Product* get_product() { // ??
        //build_partA..build_partB..build_partC ??
        return new Product();
    }
};

int main() {
    Director *director = new Director(new ConcreteBuilder());
    director->construct();
}
```


## 原型模式

- 用来从已有对象拷贝创建新对象

```cpp
class Prototype {
public:
    virtual Prototype* get_clone() = 0;
protected:
    Prototype() = default;
};

class ConcretePrototype: public Prototype {
public:
    ConcretePrototype() = default;
    ConcretePrototype(const ConcretePrototype &copy) = default;
    Prototype* get_clone() override {
        return new ConcretePrototype(*this);
    }
};

int main() {
    Prototype *p1 = new ConcretePrototype();
    Prototype *p2 = p1->get_clone();
}
```

# 结构模式

## 桥接模式

- 通过桥(组合)的方式，接口类和实现类联系起来
- 使用组合的方式，让抽象与实现解耦，分别独立变化
- 使用继承的方式，需求的改变会导致类的膨胀

```cpp
class Abstract {
public:
    virtual void func() = 0;
    virtual ~Abstract() { }
};

class AbstractImp {
public:
    virtual void func() = 0;
    virtual ~AbstractImp() { }
};

class RefindAbstract: public Abstract {
public:
    RefindAbstract(AbstractImp *imp): _imp(imp) { }
    void func() override { _imp->func(); }
private:
    AbstractImp *_imp;
};

class ConcreteImp: public AbstractImp {
public:
    void func() override { std::cout << "concrete_imp\n"; }
};

int main() {
    AbstractImp *imp = new ConcreteImp();
    Abstract *ptr = new RefindAbstract(imp);
    ptr->func();
}
```

## 适配器模式

- 用以将一个类的接口转为希望的接口 (转换头)
- 有类模式和对象模式两种

```cpp
// 类模式
class Target {
public:
    virtual void request() {
        std::cout<< "request\n";
    }
};

class Adaptee {
public:
    void specific_request() {
        std::cout << "specific_request\n";
    }
};

class Adapter: public Target, private Adaptee { // Target接口继承，Adaptee实现继承
public:
    void request() override {
        this->specific_request(); // 类内调用
    }
};

int main() {
    Target *target = new Adapter();
    target->request(); // 输出 specific_request
}

// 对象模式
class Target {
public:
    virtual void request() {
        std::cout<< "request\n";
    }
};

class Adaptee {
public:
    void specific_request() {
        std::cout << "specific_request\n";
    }
};

class Adapter: public Target {
public:
    Adapter(Adaptee *ade): _ade(ade) { } // Adaptee对象传入，非继承
    void request() override {
        _ade->specific_request(); // 向对象发送消息
    }
private:
    Adaptee *_ade;
};

int main() {
    Adaptee *adaptee = new Adaptee();
    Target *target = new Adapter(adaptee);
    target->request();
}
```

## 装饰模式

- 用来为已有的类添加新的职责(操作)的方法
- 使用组合的对象模式来实现
- 若使用简单继承的方式来实现
  - 当添加新操作时，就需要在Component里添加操作的虚函数，以便在ConcreteComponent里override虚函数
- 若让Decorator维护一个指向ConcreteComponent的指针或引用来实现修饰功能
  - 则Decorator只能修饰一个类，增加类需要增加对应的Decorator
- Decorator和ConcreteDecorator，可以为类添加多种操作

```cpp
#include <iostream>

class Component {
public:
    virtual void func() = 0;
    ~Component() { }
};

class Decorator: public Component {
public:
    Decorator(Component *com): _com(com) { }
    virtual void func() = 0;
    ~Decorator() { }
protected:
    Component *_com;
};

class ConcreteComponent: public Component {
public:
    void func() override {
        std::cout << "component behavior\n";
    }
};

class ConcreteDecorator: public Decorator {
public:
    ConcreteDecorator(Component *com): Decorator(com) { }
    void func() override {
        _com->func(); // 原操作
        this->add_behavior(); // 添加操作
    }
    void add_behavior() {
        std::cout << "decorator behavior\n";
    }
};

int main() {
    Component *component = new ConcreteComponent();
    Decorator *decorator = new ConcreteDecorator(component);
    decorator->func();
}
```

## 组合模式

- 用来构建树状的组合结构
- 组合模式侧重生成子类，修饰模式侧重在修饰 ??

```cpp
// 代码报错， vector无法实例化，未解决
#include <iostream>
#include <vector>

class Component {
public:
    virtual void func() = 0;
    virtual void add() = 0;
    virtual void remove() = 0;
    virtual Component* get_child(unsigned) = 0;
    ~Component() { }
};

class Composite: public Component {
public:
    void func() override {
        for (auto it = comVec.begin(); it != comVec.end(); ++it) {
            (*it)->func();
        }
    }    
    void add() override {
        comVec.push_back(new Composite());
    }
    void remove() override {
        comVec.erase(new Composite());
    }
    Component* get_child(unsigned index) override {
        return comVec[index];
    }
private:
    std::vector<Component*> comVec; // 对象池
};

class Leaf: public Component {
public:
    void func() override {
        std::cout << "is leaf\n";
    }
}

int main() {
    Composite *composite = new Composite();
    composite->add(); // 往对象池里添加对象
    composite->func(); // 调用池内每一个对象的func函数
    composite->remove(); // 从对象池中移除对象
    composite->func();
}
```

## 享元模式

- 为共享对象创建一个对象池,用来解决创建大量对象的冗余存储
- 将对象的状态分为不会变化的内部状态和会变化的外部状态
- 多个对象共享内部状态，各个对象将外部状态作为参数传递
- 留意对象池的管理策略
- UnshareConcreteFlyweight 的设计 ??

```cpp
class Flyweight {
public:
    Flyweight(const std::string &state): _state(state) { }
    virtual void func(const std::string &extern_state) = 0; // 有什么作用 ?? 改变状态 ??
    virtual std::string& get_state() = 0;
    virtual ~Flyweight() { }
protected:
    std::string _state;
};

class FlyweightFactory {
public:
    Flyweight* get_object(const std::string &key) {
        for (auto it = vec.begin(); it != vec.end(); ++it) {
            if ((*it)->get_state() == key) {
                std::cout << "founded\n";
                return *it;
            }
        }
        Flyweight *ptr = new ConcreteFlyweight(key); // 每个对象由唯一状态标识
        vec.push_back(ptr);
        return ptr;
    }
private:
    std::vector<Flyweight*> vec;
};

class ConcreteFlyweight: public Flyweight {
public:
    ConcreteFlyweight(const std::string &state): Flyweight(state) { }
    void func(const std::string &extern_state) override { }
    std::string& get_state() override { return this->_state; }
};

int main() {
    FlyweightFactory *fac = new FlyweightFactory();
    Flyweight *flyweight = fac->get_object("object_name");
    fac->get_object("object_name");
}
```


## 外观模式

- 为系统提供一个统一的对外接口，解耦系统 ??
- 使用者无须知晓系统内部的具体情况

```cpp
class Sub_system_A {
public:
    void func() {
        std::cout << "sub_system a\n";
    }
};

class Sub_system_B {
public:
    void func() {
        std::cout << "sub_system b\n";
    }
};

class Facede {
public:
    void wrap_func() {
        sys_a->func();
        sys_b->func();
    }
private:
    std::shared_ptr<Sub_system_A> sys_a;
    std::shared_ptr<Sub_system_B> sys_b;
};

int main() {
    Facede *facede = new Facede();
    facede->wrap_func();
}
```


## 代理模式

- 用户将工作交给代理去完成，代理给真正的执行者发送消息
  - 虚代理，让代理负责创建对象
  - 远程代理，创建一个本地代理对象来为表示对端的实际对象
  - 保护代理，通过代理对象来实现不同权限的访问控制
- 实现逻辑与实现的解耦 ?? 代理是接口，执行者是实现 ??

```cpp
class Subject {
public:
    virtual void request() = 0;
    virtual ~Subject() { }
};

class Proxy {
public:
    Proxy(Subject *x): _subject(x) { }
    void request() {
        _subject->request();
    }
private:
    Subject *_subject;
};

class ConcreteSubject: public Subject {
public:
    void request() override { 
        std:: cout << "concrete subject\n"; 
    }
};

int main() {
    Subject *subject = new ConcreteSubject();
    Proxy *proxy = new Proxy(subject);
    proxy->request();
}
```

# 行为模式

## 模板模式

- 将算法接口放在抽象基类，将算法实现放在多态派生类
- 用继承多态的方式实现算法接口与算法实现的解耦 (算法异构)
- 继承方式的缺点是，其他类不能不用算法实现的派生类
- 反向控制，高层模块调用底层模块，底层模块实现高层模块声明的接口

```cpp
#include <iostream>

class Interface {
public:
    virtual void method() = 0;
    virtual ~Interface() { }
};

class Implement: public Interface {
public:
    void method() override { std::cout << "concrete imp\n"; }
};

int main() {
    Interface *interface = new  Implement();
    interface->method();
}
```

## 策略模式

- 将算法的接口封装到一个context类里，strategy类负责算法的实现
- 实现抽象接口的方式
  - 继承
    - 依赖性强，基类变，派生类也随之改变
  - 组合
    - 依赖性弱，可复用实现类

```cpp
class Strategy {
public:
    virtual void func() = 0;
    virtual ~Strategy() { }
};

class Context { // 如何理解Context ??
public:
    Context(Strategy *strategy): _strategy(strategy) { }
    void action() { _strategy->func(); }
private:
    Strategy *_strategy;
};

class ConcreteStrategy: public Strategy {
public:
    void func() override { std::cout << "alogrithm impletation\n"; }
};

int main() {
    Strategy *strategy = new ConcreteStrategy();
    Context *context = new Context(strategy);
    context->action();
}
```

## 状态模式

- 用来将switch的分支状态与分支动作实现进行解耦
- 将每一个switch的case状态分支都封装到一个独立的State类里
- State类的派生类定义具体的case状态分支，并通过反调用context来改变状态
- 状态模式侧重状态改变的不同处理策略，策略模式侧重算法接口与算法实现的解耦
- 可以用Context的派生类来替换分支动作实现
- 状态模式的问题是逻辑分散化，很难知晓全部的state状态分支

```cpp
// 代码有错
class Context;

class State {
public:
    virtual void change_state(Context *context) = 0;
    virtual ~State() { }
};

class Context {
public:
    Context(State *state): _state(state) { }
    void next_state() { _state->change_state(this); }
    void change_state(State *new_state) {
        _state = new_state;
    }
private:
    State *_state;
};

class ConcreteState_B;

class ConcreteState_A: public State {
public:
    void change_state(Context *context) override {
        context->change_state(new ConcreteState_B());
        std::cout << "changed to state_B\n";
    }
};

class ConcreteState_B: public State {
public:
    void change_state(Context *context) override {
        context->change_state(new ConcreteState_A());
        std::cout << "changed to State_A\n";
    }
};

int main() {
    State *state = new ConcreteState_A();
    Context *context = new Context(state);
    context->next_state();
}
```


## 观察者模式

- 建立Subject和Observer的一对多关系
  - vector<Observer>
  - attach、detach
- 当Subject发生更新后，在合适的时候使用notify发送消息(广播通知)给所有Observer，让其同步更新
- 在Model/View/Control模型中，Model为Subject，View为Observer
- 也称为"订阅(publish)-发布(subscribe)"模式，通知的发布者是Subject，通知的订阅者是Observer

```cpp
// 代码有错，vector由于未完成类型，无法实例化
class Subject;

class Observer {
public:
    Observer(Subject *subject): _subject(subject) { 
        _subject->attach(this);
    }
    ~Observer() { _subject->detach(this); }
    void update(const std::string state) {
        _state = state;
        std::cout << "observer changed to " << _state << '\n';
    }
private:
    Subject *_subject;
    std::string _state;
};

class Subject {
public:
    void attach(Observer *observer) {
        _observer.push_back(observer);
    }
    void detach(Observer *observer) {
        _observer.erase(observer);
    }
    void notify() {
        for (const auto &ptr : _observer) {
            ptr->update(_state);
        }
    }
    Subject(const std::string state): _state(state) { }
    void change_state(const std::string new_state) {
        _state = new_state;
        std::cout << "subject changed to " << _state << '\n';
    }
private:
    std::string _state;
    std::vector<Observer*> _observer;
};

int main() {
    Subject *subject = new Subject("old message");
    Observer *observer = new Observer(subject);
    subject->change_state("new message");
    subject->notify();
}
```

## 备忘录模式

- 用来提供撤销(undo)的功能
- 通过一个Memento类来保存Originator类的状态(拷贝)，在需要的时候，将保存的状态恢复到Originator类

```cpp
#include <iostream>
#include <string>

class Originator;

class Memento {
    friend class Originator;
private:
    Memento(const std::string &state): _state(state) { }
    std::string get_state() const { return _state; }
    void set_state(const std::string &state) { _state = state; }
private:
    std::string _state;
};

class Originator {
public:
    Originator(): _memento(new Memento("")) { }
    void save() { _memento->set_state(_state); }
    void undo() { _state = _memento->get_state(); }
    void change_state(const std::string new_state) {
        _state = new_state;
    }
    void print() const { std::cout << _state << '\n'; }
private:
    std::string _state;
    Memento *_memento;
};

int main() {
    Originator *originator = new Originator();
    originator->change_state("new_state");
    originator->print();
    originator->undo();
    originator->print();
}
```


## 中介者模式

- 用来实现一对多的对象间通信，各个对象间彼此不需要互相认识，由中介者提供消息转发
- 将对象间的交互封装在Mediator类中，实现系统对象(Colleague类)间的解耦，Mediator和Colleague可以相互独立修改
- Mediator将控制集中，便于管理

```cpp

```


## 命令模式

- 用来将命令的调用者(发送者)和命令的执行者(接收者)解耦
- Invoker不知道具体哪个对象在执行命令，Command将命令和命令的执行者进行绑定，绑定可以是组合方式或是回调方式
- 比较消费者生产者的队列
- 如何实现command与处理函数的绑定，以回调的方式执行命令 ?? 类成员函数指针 ??

```cpp
class Command {
public:
    virtual void execute() = 0;
    virtual ~Command() { }
};

class Invoker {
public:
    Invoker(Command *command): _command(command) { } // 添加command到invoker
    void invoke() { _command->execute(); }
private:
    Command *_command; // 可以是命令集合
};

class Reciver {
public:
    virtual void process() = 0;
    virtual ~Reciver() { }
};

class ConcreteCommand: public Command {
public:
    ConcreteCommand(Reciver *reciver): _reciver(reciver) { } // command与reciver绑定
    void execute() override { _reciver->process(); }
private:
    Reciver *_reciver;
};

class ConcreteReciver: public Reciver {
public:
    void process() override { std::cout << "concrete reciver\n"; }
};

int main() {
    Reciver *reciver = new ConcreteReciver();
    Command *command = new ConcreteCommand(reciver);
    Invoker *invoker = new Invoker(command);
    invoker->invoke();
}
```


## 访客模式

- 用来给已经实现好的类添加新的方法实现新需求
- 将更新部分封装成Visitor类
- 实现方式一，在被更新类Element定义一个accept方法，使得Visitor可以通过接口调用修改Element的状态
- 实现方式二是将Visitor类声明为Element的友元，直接修改Element的状态

```cpp
class Element;

class Visitor {
public:
    virtual void visit(Element*)  = 0;
    virtual ~Visitor() { }
};

class Element {
public:
    virtual void accept(Visitor*) = 0;
    virtual ~Element() { }
};

class ConcreteVisitor: public Visitor {
public:
    void visit(Element *element) override { 
        std::cout << "concrete visitor\n";  //  如何修改Element的内部状态 ??
    }
};

class ConcreteElement: public Element {
    friend class ConcreteVisitor; // 友元的访问权限问题  ？？ 无法通过指针或对象访问非成员 ??
public:
    void accept(Visitor *vistor) override {
        vistor->visit(this);
    }
};

int main() {
    Visitor *visitor = new ConcreteVisitor();
    Element *element = new ConcreteElement();
    element->accept(visitor);
}
```

## 责任链模式

- 链式处理请求，请求沿着预先定义好的路径依次进行处理
- 将可能处理一个请求的对象链接成一个链，并将请求在这个链上传递，直到有对象处理该请求
- ConcreteHandle设置它的后继对象
- 当一个请求来时，ConcreteHandle会先检查自己有没有匹配的处理程序，如果有就自己处理，否则传递给它的后继
- 请求的发送者不需要知道请求会被哪个应答对象处理，解耦请求发送者和请求的应答者

```cpp

class Handle {
public:
    virtual void handle_request() = 0;
    virtual ~Handle() { }
    void set_successor(Handle *successor); // 添加后继对象(next)
};

class ConcreteHandle:public Handle {
public:
    void handle_request() override {
        
    }
};

int main() {
    Handle *h1 = new ConcreteHandle();
    h1->handle_request();
}
```


## 迭代器模式

- 用来解决一个聚合对象的遍历问题
- 将对聚合对象的遍历封装到一个类中，避免暴露聚合对象的内部表示
- 为了更好保护Aggregate的状态，尽可能减小Aggregate的对外接口

```cpp
class Iterator {
public:
};

class Aggregate {
    friend class Iterator;
public:

};

class ConcreteIterator: public Iterator {
public:
    ConcreteIterator(Aggregate *ag, unsigned idx = 0): _ag(ag), _idx(idx) { }
private:
    Aggregate *_ag;
    int _idx;
};

int main() {
    Aggregate *aggregate = new ConcreteAggregate();
    Iterator *iter = new ConcreteIterator(aggregate);
    for(/*void*/; !it->is_done(); it->next()) {
        std::cout << *it->current << " ";
    }
}
```


## 解释器模式

- 提供组织和设计语法解释器的框架，通过语法解释器来解释语言的句子
- 使用类来表示文法规则，易于实现文法的扩展
- 使用Flyweight模式来实现终结符的共享
- Context为解释过程提供一些附加的信息，如全局信息

```cpp
class Context { };
class Expression {
public:
    virtual void interpret(const Context &) = 0;
    virtual ~Expression() { }
};

class Terminal: public Expression { // 不含非终结符的表达式
public:
    Terminal(const std::string &statement);
    void interpret(const Context &context) override {
        std::cout << this->_statement;
    }
private:
    std::string _statement;
};

class Nonterminal: public Expression {
public:
    Nonterminal(Expression &exprssion, unsigned times);
    void interpret(const Context &context) override {
        for (unsigned i = 0; i < times; ++i)
            this->_exprssion->interpret(context);
    }
private:
    unsigned _times;
    Expression *_exprssion;
};

int main() {
    Context *context = new Context();
    Expression *terminal_expr = new Terminal("abc");
    Expression *nontermial_expr = new Nonterminal(terminal_expr, 2);
    nontermial_expr->interpret(*context);
}
```
