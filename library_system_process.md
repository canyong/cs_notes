

## 基本问题

- 进程用来解决什么问题
  - 计算机能同时进行多个活动并发
- 如何解决这个问题
  - 将正在运行的程序抽象为进程
  - 系统有多个进程，CPU由一个进程快速切换到另一个进程，进程交替轮流运行，产生并行的效果
  - 每个进程都有自己的逻辑程序计数器、逻辑寄存器和变量的当前值(内存数据)
    - 当进程运行时，将自己的逻辑程序计数器装入真正的程序计数器
    - 当进程暂停运行时，将物理的程序计数器保存到逻辑程序计数器，保存物理寄存器的数据状态
  - 伪并行，指任一时刻，cpu只能运行一个进程，但很短的一段时间内，cpu可能运行多个进程
  - 真正的并行，指硬件有多个cpu，它们共享同一个物理内存，任一时刻，每个cpu运行一个进程
- 解决效果如何
  - 多个进程共享一个cpu，操作系统采用某个调度算法来决定将cpu切换到其他哪个进程
  - 系统调度器切换进程的时机是无法预料的，进程的运行速度是不确定的，随时可能会被调度器暂停


## 进程

- 进程与程序的区别
  - 程序是做蛋糕的食谱
  - 进程是厨师阅读食谱、准备各种原料以及烘培蛋糕的一系列动作总和
  - 一个进程是一个活动，有程序、输入、输出以及运行状态
  - 同一个程序运行了两次，则产生两个进程，进行了两次活动

- 什么时候会创建进程
  - 已存在的进程执行创建进程的系统调用时
  - 系统初始化，会自动创建一个名为init的进程，由它来创建更多的进程
    - 前台进程，同用户交互的进程
    - 后台进程，与特定用户无关系，在后台执行特定的任务，大部分时间在睡眠，当请求达到时，后台进程被唤醒来服务该请求，也称为守护进程(deamon)
- 怎么创建一个进程
  - fork调用，创建一个与调用进程相同的进程，有相同的映像(image)、系统环境和打开文件
    - 新进程称为调用进程的子进程，概念上两个进程有各自的地址空间，如果其中一个进程修改了其地址空间中的某个字，这个修改对其他进程是不可见的。但unix的fork的子进程与父进程有相同的地址空间，不可写/只读的内存区是共享的
  - execve调用，修改进程的映像文件(程序文件)，以运行一个新的程序
  - 将创建进程分为两步的原因，是为了子进程可以在fork后,execve前，关闭从父进程继承的打开文件或重定向标准输入输出

- 什么时候进程终止
  - 程序执行完成，正常终止
  - 程序执行出错，收到信号并采用默认动作时，异常终止
  - 程序执行中，收到kill调用发出SIGKTERM信号，异常终止
  - 每个进程对信号可以有三种动作：
    - 捕获信号，进程准备处理信号
    - 忽略信号，进程不接收信号，挂起信号
    - 采取默认动作， 则进程被信号终止


- 进程的层次结构(进程树)
  - 进程只有一个父进程，可以有n个子进程
  - 进程和它的所有子进程和后裔进程共同组成一个进程组
  - unix的init进程为每个终端创建一个登录进程，用户登录成功则登录进程创建一个shell进程来准备接受命令


- 进程的状态
  - 运行态
    - 进程正在使用CPU
  - 就绪态
    - 进程等待使用CPU
  - 阻塞态
    - 进程等待外部事件的发生(非等待CPU，CPU空闲也不能运行)
    - 当等待的外部事件发生时，进程解除阻塞，转为可被调度运行的就绪态
  - 进程调度程序
    - 决定运行哪个进程、何时运行及运行多长时间(决定哪个进程使用CPU)
    - 调度算法努力在整体效率和竞争公平性之间取得平衡

- 进程的实现
  - 为实现(顺序)进程模型，系统维护一张进程表(结构体数组)，每个进程占用一个表项
  - 进程表项包含了进程的运行时信息，保证进程被暂停后能再次启动，像从未被中断过的一样
    - 进程管理
      - 程序计数器、程序状态字、寄存器、堆栈寄存器(栈)
      - 进程状态、优先级、调度参数
      - 进程ID、父进程、进程组
      - 信号集
      - 进程开始时间、使用的cpu时间、子进程的cpu时间、下次报警(SIGALAM)时间(时间中断?)
    - 存储管理
      - 代码段指针、数据段指针、堆栈段指针 (映像文件的内存载入地址)
    - 文件管理
      - 根目录、工作目录
      - 文件描述符、用户ID、组ID

- 中断
  - 中断向量(interupt vector)
    - 是一张表，记录I/O中断信号和其对应的中断服务程序的入口地址
  - 中断的过程
    - 当磁盘发生中断信号时，中断硬件将程序计数器、程序状态字、寄存器组压入正在运行被中断暂停的进程地址空间的堆栈中，然后将程序计数器指向中断服务程序的入口地址
    - 接着，开始运行中断服务程序(中断软件)
    - 当中断服务程序运行结束，接着调用进程调度器，决定随后该运行哪一个进程，随后可能将控制权交给新进程的装载器，运行新进程
  - 当中断完成，通常会使某些进程由阻塞态变为就绪态

- 多道程序设计模型
  - 预测cpu的使用率
    - 假设一个进程等待I/O操作的时间为其停留在内存中时间(进程生命周期)的比为p，则当内存中有n个进程，所有n个进程都在等待I/O的概率为p^n(概率乘积)
    - 则CPU的利用率为 1 - p^n， 即CPU非空闲的概率/时间
    - 如果每一个进程花费80%的时间等待I/O，为了使CPU的空闲率小于10%，则至少要有10个进程在同时在内存中
    - 该概率预测模型假设所有的进程都是相互独立，不考虑条件概率的情况
  - 使用概率预测模型分析
    - 假设现有台计算机，内存空间只有512MB，操作系统占用128MB，每个用户程序也占用128MB，此时允许最多三个用户进程同时驻留在内存中(不考虑虚拟内存的情况)，若每个进程用80%的时间等待I/O，则CPU的利用率大约为 1 - 0.8^3 = 0.49
    - 若增加512MB的内存空间，则允许最多7个用户进程同时驻留在内存中，此时CPU的利用率为 0.79.
    - 若再增加512MB的内存空间，则CPU的利用率为 0.91，仅提高了0.12
    - 因此，扩增第一次是划算的，扩增第二次则不是

## 进程通信
