
## 线程对象

- 用对象表示线程概念
- 默认线程对象不关联任何线程函数, 使用线程对象的构造函数来与某一线程函数关联
- 构造函数默认对传入的线程函数和调用参数进行拷贝，使用std::ref来进行引用传参
- 提供的线程函数会复制到新线程的存储空间中，函数对象的执行和调用都在线程的内存空间中进行(线程栈)
- 线程对象默认是线程函数的运行时容器，线程对象的生命周期需比线程函数的生命周期长，保证线程函数在线程对象析构之前执行结束
- 从线程对象detach线程函数，线程对象与该线程函数不再关联，成为空的线程对象，不需要关心处理线程函数的生命周期问题
- 分离的线程函数为守护线程，指没有任何用户接口并在后台运行的线程，特点是生命周期长
- 对空的线程对象使用join()和detach()都将出错

```cpp
// 异常安全
class thread_guard {
    std::thread& t_;
public:
    explicit thread_guard(std::thread& t): t_(t) { }
    ~thread_guard() {
        if (t.joinable())
            t.join();
    }
    thread_guard(const thread_guard &) = delete;
    thread_guard& operator=(const thread_guard&) = delete;
};

int main() {
    std::thread t(func);
    thread_guard g(t);
    do_something_in_current_thread(); // 抛出异常
    t.join(); // 被异常跳过
}

// 线程函数传参
void func() {
    char buffer[1024];
    std::thread t(f, buffer); // 传递指针
    std::thread t(f, std::string(buffer)); // 避免悬垂指针
}

// 传递成员函数
struct X {
    void func(int);
}；

X x;
int num = 42;
std::thread td(&X::func, &x, num); // 成员函数，对象，成员函数参数


```


## 多线程同步

- join()
  - 一线程同步等待另一线程执行完成，再往下继续执行
```cpp
// 顺序执行流
```

- 互斥锁
  - 用于保护多线程访问的共享数据
  - lock()
  - unlock()
  - 递归锁/共享锁，获取锁的线程函数内调用的其他函数可再次获取锁
  - 死锁问题
    - 资源迟迟得不到释放，线程无限期阻塞等待锁的释放
    - 当需要一次性获取多个互斥量时，多线程获取锁的顺序不一致可能导致死锁
    - 在获取锁的线程函数内调用其他其他函数获取普通锁时，发生死锁
  - 锁粒度问题
    - 粒度过小，起不到保护共享数据的作用
    - 粒度过大，多线程争用锁的开销，可能抵消线程并发带来的效益
```cpp
// 可能产生死锁的情况和避免死锁的做法
```

- 条件变量
  - 用于通知(唤醒)等待条件发生/条件满足的线程，通知机制
  - wait()
    - 先获取共享数据的互斥锁，再检查共享数据的条件是否满足，如不满足则释放互斥锁并进入睡眠等待，如满足，则继续往下执行，执行完毕释放互斥锁
  - notify()
    - 唤醒在共享数据上等待条件发生的一个或多个线程，被唤醒的线程可能按照优先级先后次序获取互斥锁并再次检查条件是否满足，不满足则继续睡眠等待
    - 伪唤醒，当睡眠的线程被唤醒，但不是再次对条件是否满足进行检查，则为伪唤醒
```cpp
// 生产者消费者同步队列
```

- 线程同步对线程并发执行效率的影响程度
  - 既要程序正确，也要多线程的并发效率


## 异步

- 期望 future
  - 表示异步操作的结果
  - get()
    - 当异步操作尚未完成时，获取future的结果的调用将会被阻塞到异步操作完成
  - future_status
    - 检查异步操作的完成状态，在future就绪时，再调用获取future (无阻塞)
    - ready，就绪
  - 若异步操作失败，非正常结束，则future的值为一个异常对象

- 承诺 promise
  - 封装了future和一个变量，将future和变量进行关联，传入给异步操作以写入异步操作的结果
  - set_promise()
    - 写入到变量的值，将同步到promise对象里的future对象
  - set_exception()
    - 写入异常
  
- 任务
  - 封装了future和可调用对象，将future和可调用对象关联，future用于存储可调用对象的操作结果

- 异步线程
  - 产生一个线程来执行异步操作
  - 封装了线程对象和future，将线程对象和future对象关联，future存储线程的操作结果
  - 线程启动策略
    - async::launch， 立即产生一个异步线程并执行
    - async::defered，直到future.get()时，才开始产生并执行异步线程


## 原子类型

- lock_free 无锁
  - 在多线程环境，不显式使用锁来保护共享数据，避免显式用锁的死锁问题(全部线程死锁将没有线程可以往前推进，cpu空转)
  - 将共享数据设置为原子类型，对原子类型对象的操作的原子性由编译器负责，可能的实现是在每个对原子类型的操作中隐式使用锁
  - 保证在任一时刻，至少有一个线程在执行，使用cpu
  - lock_free vs mutex，优劣与适用场景/情况
- wait_free 无等待/无阻塞
  - 使用可能会阻塞的调用时，只有当数据准备好时，才进行访问，避免在数据未准备好时访问被阻塞
  - 使用非阻塞的调用时，无论数据是否准备好，都进行访问，若发现数据还未准备好，则返回错误码，不阻塞等待数据就绪. 当数据准备就绪时，异步通知调用者，让调用者来对就绪的数据进行处理
    - 可能的异步通知的方式
      - 回调调用者的就绪数据处理函数
      - 将执行控制权转移给调用者 (协程的方式)
      - 发送信号给调用者，信号处理函数可能会将信号写入信号管道，触发可读事件就绪

- atomic<T>
  - atomic_flag，无锁/不带锁的布尔量
  - store() 和 load()，读写原子变量
  - compare_and_exchange()
  - ...其他原子操作

- 多线程的原子操作顺序
  - 读 - 读 / 读 - 写 / 写 - 读 / 写 - 写
  - 一致性顺序
    - 两线程间的对同一个原子变量进行的读写原子操作的执行顺序与代码顺序一致
  - 松散顺序
    - ...执行顺序由编译器和cpu自行决定，程序不做执行顺序做任何要求，编译器和cpu各自进行代码优化和指令优化
  - 读-写-读-写
    - 线程读取时，需在另一线程的写入结果由缓存同步更新到内存时，再对内存地址发起读取，写操作之后的读操作不能跳到写操作之前执行
    - 简而言之，不同线程间的读操作和写操作存在数据依赖，则执行顺序同代码顺序。单一线程内，无数据依赖的读写操作执行顺序由编译器和cpu决定
  - 使用建议
    - 读-读的情况，没有数据依赖，可以任意顺序执行，谁先谁后不影响程序正确性
    - 其他情况，若存在数据依赖，则采用一致性顺序，保证正确性，没有数据依赖则任意执行顺序，即采用松散顺序


## 线程安全

- 可重入函数
  - 对于同一个函数，多个线程可以并发调用，调用时的函数状态都相同，即无状态的函数，函数的输出只与输入有关，不引用或不依赖函数外部的变量状态，不受调用环境影响,
  - 多线程并发调用该函数，不需要使用锁进行保护，因每一个函数所使用的变量数据都属于线程内的数据，不存在多线程的共享数据

- 线程安全容器
  - 从接口设计上，避免可能产生数据竞争的接口
  - 可能产生数据竞争的接口，使用互斥量或其他线程同步方式进行保护共享数据/临界区