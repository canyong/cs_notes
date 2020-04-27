
## 服务器程序框架

服务器模型：
- C/S模型
  - 服务器是通信的中心，访问量过大，响应慢
  - 实现简单，适合资源相对集中的场景
- P2P模型
  - 无中心服务器，每台主机地位对等，既是客户端也是服务器，共享资源
  - 通常有一台发现服务器，提供资源查找服务，帮助客户找到自己所需的资源
  - 当用户之间的传输请求过多时，将加重网络负载 ?? 整个网络负载 ??

服务器程序基本框架：
- I/O处理单元  <--请求队列-->  逻辑单元  <--请求队列-->  网络存储单元
  - I/O处理单元，负责管理客户连接，可能是一台实现负载均衡的接入服务器
  - 逻辑单元，负责处理数据，可能是一个进程/线程或一台逻辑服务器
  - 网络存储单元，可能是数据库/缓存/文件或一台独立服务器
  - 请求队列，负责各单元之间的通信和模块解耦，可能是独立服务器之间的永久TCP连接
- 不同的服务器程序基本框架相同，不同之处在于逻辑处理、
- 读写I/O的数据收发可能在I/O处理单元，也可能在逻辑单元中执行，取决于具体的事件处理模式


## I/O模型

I/O模型：
- 客户连接请求是随机到来的异步事件，服务器需要使用某种I/O模型来监听事件
- 阻塞和非阻塞
  - 能用于所有的文件描述符
  - 阻塞的文件描述符为阻塞I/O
    - 对阻塞I/O执行的系统调用可能被操作系统挂起，直到事件发生为止
  - 非阻塞的文件描述符为非阻塞I/O
    - 对非阻塞I/O执行的系统调用总是立即返回，若事件尚未发生，则返回EAGAIN或EWOULDBLOCK(期望阻塞)错误，对于connect而言，返回EINPROGRESS(处理中)错误
  - 只有在事件已经发生的情况下，操作非阻塞I/O，才能提高程序效率。通常非阻塞I/O和I/O通知机制一起使用
- 同步和异步
  - 同步I/O模型
    - 在I/O事件发生之后，由用户进程完成I/O读写操作(内核缓冲<-->用户缓冲)
    - 向用户进程通知I/O就绪事件
    - 阻塞I/O、I/O复用和信号驱动I/O都是同步I/O
  - 异步I/O模型
    - 在I/O事件发生之后，由内核完成I/O读写操作
    - 向用户进程通知I/O完成事件
    - linux的<aio.h>头文件提供对异步I/O的支持
- I/O通知机制
  - I/O复用
    - 用户进程通过I/O复用函数向内核注册一组事件，内核通过I/O复用函数把就绪事件通知给用户进程
    - I/O复用可以同时监听多个I/O事件，但I/O复用函数本身是阻塞的
  - SIGIO信号
    - 可用于报告I/O事件
    - 当文件描述符有事件发生时，文件描述符对应的宿主进程捕获SIGIO信号，并调用信号处理函数执行非阻塞的I/O操作
- I/O模型对比
  - 阻塞I/O，程序阻塞于读写函数
  - I/O复用，程序阻塞于I/O复用函数(系统调用)，对I/O的读写是非阻塞的
  - SIGIO信号，对已就绪事件执行I/O读写，程序没有阻塞
  - 异步I/O，内核完成I/O读写，程序没有阻塞


## 事件处理模式

事件处理模式：
- 服务器程序通常需要处理三类事件：I/O事件、信号和定时器事件
- Reactor模式
  - 通常由同步I/O模型实现
  - 主线程(I/O处理单元)只负责监听，若文件描述符有事件发生，立即将该事件通知工作线程(逻辑单元)
  - 主线程将socket的读写事件放入请求队列
  - 工作线程从请求队列取出事件，根据事件类型来决定如何处理它，可能接受新连接或读写I/O并处理请求
  - 处理流程：
- Proactor模式
  - 通常由异步I/O模型实现，也可以用同步I/O方式模拟
  - 主线程负责监听、接受新连接，内核负责读写I/O，工作线程仅负责处理请求
  - 当文件描述符上的完成事件发生，程序的信号处理函数将选择一个工作线程来处理请求
  - 主线程的epoll_wait调用只能监听新连接事件，无法检测连接socket的读写事件
  - 使用同步I/O方式模拟Proactor
    - 主线程负责从socket读取数据，将读取到的数据封装一个请求对象并插入请求队列
    - 工作线程被唤醒，获得请求对象并处理请求
  - 处理流程：


## 并发模式

并发模式：
- 并发编程：
  - I/O操作速度比CPU计算速度慢，程序可能阻塞于I/O操作，等待I/O操作完成
  - 并发编程让程序同时执行多个任务/线程，当执行I/O操作的线程被阻塞时，将控制权转移给其他线程，提高cpu利用率
  - 并发编程有多进程和多线程两种方式
- 半同步/半异步模式
  - 同步指程序的执行完全按照代码序列的顺序执行
    - 按照同步方式运行的线程，为同步线程
    - 同步线程逻辑简单、效率低、实时性较差
  - 异步指程序的执行需要系统事件来驱动，系统事件包括中断、信号等
    - 按照异步方式运行的线程，为异步线程
    - 异步线程逻辑复杂、难调试和扩展，但效率高、实时性强
  - 半同步/半异步，指用异步线程处理I/O事件，用同步线程处理请求
    - 异步线程监听到用客户请求后，将其封装成请求对象并插入请求队列
    - 具体选择哪个工作线程为新的客户请求服务的策略 ??
      - Round Robin算法
      - 条件变量或信号量随机选择
- 半同步/半反应堆模式
  - 采用Reactor事件处理模式
    - 主线程充当异步线程，负责接受新连接和监听连接socket上的读写事件
    - 如果连接socket有读写事件发生，主线程将连接socket插入到请求队列
    - 所有工作线程睡眠在请求队列，当任务到来时，它们通过竞争(如互斥锁)来获得任务接管权
    - 工作线程负责连接socket的读写I/O操作
  - 采用Proactor事件处理模式
    - 由主线程来完成I/O数据的读写，将数据信息封装为一个任务对象，插入到请求队列
    - 工作线程从请求队列中取得任务对象，然后处理请求，无读写I/O操作
  - 缺点
    - 主线程和工作线程共享请求列队，需要对请求队列加锁保护
    - 每个工作线程同一时间只能处理一个客户请求，当前请求处理完才能处理下一个请求
  - 改进
    - 主线程只负责监听socket，接受新连接，然后将连接socket派发给工作线程（可以使用管道派发）
    - 连接socket上的读写I/O操作由工作线程来负责，工作线程用独立的事件循环来监听连接socket上的读写事件
    - 主线程和工作线程都维持自己的事件循环，每个线程都工作在异步模式

- 领导者-追随者模式
  - 多个工作线程轮流获得事件源集合/事件循环，轮流监听、分发并处理事件
  - 任一时间，只有一个领导者线程，它负责监听I/O事件，其他线程都是追随者，休眠在线程池里等待成为新的领导者
  - 当前领导者检测到I/O事件，首先从线程池中推选/指定出下一个领导者线程，然后再处理I/O事件，此时领导者地位未改变
  - 下一个领导者线程等待新的I/O事件，与现领导者处理I/O事件，两者形成了并发
  - 每个时刻，只有一个领导者线程在监听I/O事件并处理请求，因此此模式无须在线程间传递任何额外数据。但缺点是仅支持一个事件源集合/事件循环，工作线程无法同时处理多个连接
  - 组件
    - 句柄集(HandleSet)，表示I/O资源集合(一组文件描述符)
      - wait_for_event()，用于监听句柄上的事件
      - register_handle()，用于将句柄和事件处理器进行绑定
    - 线程集(ThreadSet)，表示线程集合，负责线程间同步和推选新领导者线程
      - 线程集中的线程在任一时间将处于如下三种状态之一
        - Leader， 线程当前是领导者身份，负责等待句柄集上的I/O事件
        - Processing， 线程正在处理事件，在处理事件之前须调用promote_new_leader()推选出新领导者
        - Follower，线程当前是追随者，调用线程集的join()来等待成为新的领导者
    - 事件处理器(EventHandler)，包含多个用于处理事件对应业务逻辑的回调函数
      - 当句柄(Handle)/文件描述符上有事件发生时，领导者线程调用与句柄绑定事件处理器中的回调函数
    - 具体事件处理器(ConcreteEventHandler)，派生实现基类的回调函数，以处理特定的任务



## I/O复用

- I/O复用
  - I/O复用使得程序能够同时监听多个文件描述符，并发处理多个连接，但其本身是阻塞的，应用程序可能阻塞在I/O复用函数
  - 当多个文件描述符同时就绪时，如果不采取额外措施，程序串行处理每一个文件描述符的就绪事件。只有使用多进程或多线程，才能并发处理多个就绪事件

- 需要使用I/O复用技术的情况
  - 客户端程序
    - 同时处理多个socket，如使用非阻塞的connect调用一次性发起多个连接
    - 同时处理用户输入和网络连接
  - 服务器程序
    - 同时处理监听socket和连接socket
    - 同时处理TCP请求和UDP请求
    - 同时监听多个端口或多个文件描述符

- select系统调用
  - 用于在一段指定的时间内，监听感兴趣的文件描述符上的可读、可写和异常等事件
  - nfs参数
    - 表示被监听的文件描述符的总数
    - 通常设为进程已打开的最大文件描述符+1，因进程打开的文件描述符由系统从0开始顺序分配
  - readfds/writefds/exceptfds参数
    - 分别表示可读事件/可写事件/异常事件对应的文件描述符集合，用整数数组的位模式表示
    - 调用select时，传入这三个参数；调用返回时，内核将以"值-结果"方式修改这三个参数，通知用户哪些文件描述符已经就绪。但用户仍需自己遍历检查所有位以获取具体的哪一个文件描述符是已就绪的
  - timeout参数
    - 表示select调用的超时时间，精确到毫秒
    - 调用时，timeout为0，则select立即返回；timeout为NULL，则select将一直阻塞，直到某个文件描述符就绪
    - 调用返回时，内核将修改timeout参数，告诉用户select的实际等待时间，但调用失败时，timeout的值是不确定的 
  - 函数调用返回值
    - 成功，返回就绪的文件描述符的数量
      - 超时返回，没有任何文件描述符就绪，返回0
    - 失败，返回-1并设置errno
      - 异常返回，若等待期间程序收到信号，返回-1并设置errno为EINTR
```cpp
#include <sys/socket.h>
int select( int nfds, fd_set* readfds, fd_set* writefds, fd_set* exceptfds, struct timeval* timeout );

#include <typesizes.h>
#define __FD_SETSIZE 1024
#define __NFDBITS (8 * (int)sizeof(long int))
typedef struct {
    long int _fds_bits[ __FD_SETSIZE / __NFDBITS ];
} fd_set;

struct timeval {
    long tv_sec; // 秒数
    long tv_usec; // 微妙
};

#include <sys/select.h>
FD_ZERO( fd_set* fdset );
FD_SET( int fd, fd_set* fdset );
FD_CLR( int fd, fd_set* fdset );
int FD_ISSET( int fd, fd_set* fdset );
```

- socket文件描述符就绪的条件
  - 可读就绪
    - socket接收缓存字节数大于低水位标记(SO_RCVLOWAT) (已接收足够多的数据)
      - 此时可以无阻塞地读socket，读操作返回值大于0
    - 通信的对方关闭连接
      - 对该socket的读操作将返回0
    - 有新的连接请求
    - 有未处理的错误
      - 可以使用getsockopt函数来读取和清理错误
  - 可写就绪
    - socket发送缓存区可用字节数大于低水位标记(SO_SNDLOWAT) (有足够多的空间可以写)
      - 无阻塞写socket
    - socket写操作被关闭
      - 对该socket写操作将触发一个SIGPIPE信号(触发异常)
    - 使用非阻塞的connect调用
      - 因非阻塞，connect立即返回，此时连接可能成功建立或尚未建立或建立失败，都满足可写就绪
    - 有未处理的错误

- 处理带外数据
  - 当socket收到普通数据和带外数据,select都会正常返回，但此时socket处于不同的就绪状态
    - 普通收据 -- socket可读状态
    - 带外数据 -- socket异常状态
  - 对带外数据，需使用特殊的标志读取
```cpp
whiel (1) {
    FD_SET( connfd, &read_fds );
    FD_SET( connfd, &exception_fds );
    int ret = select( connfd + 1, &read_fds, NULL, &exception_fds, NULL );
    if (FD_ISSET( connfd, &read_fds )) {
        ret = recv( connfd, buf, sizeof(buf)-1, 0 );
        printf( "get %d bytes of normal data: %s\n", ret, buf );
    }
    else if (FD_ISSET( connfd, &exception_fds )) {
        ret = recv( connfd, buf, sizeof(buf)-1, MSG_OOB );
        printf( "get %d bytes of oob data: %s\n", ret, buf );
    }
}
```

- poll系统调用
  - 在指定时间内轮询一定数量的文件描述符，检查是否有已就绪者
  - fds参数
    - 是一个pollfd类型的数组，表示一组 tuple<文件描述符,注册事件集,就绪事件集> 的集合
    - pollfd类型的events成员表示监听fd上的哪些事件(按位或)，revents表示实际发生的事件(按位或)，由内核填写
  - nfds参数
    - 表示事件集fds数组的大小
  - timeout参数
    - 单位毫秒，当timeout为0时，立即返回; 当timeout为-1，调用永远阻塞，直到某个事件发生
  - poll事件类型
    - 可读
      - POLLIN
      - POLLPRI，优先读取，如TCP带外数据
      - POLLRDNORM / POLLRDBAND，普通数据可读/linux不支持
    - 可写
      - POLLOUT
      - POLLWRNORM / POLLWRBAND，普通数据可写/带外数据可写
    - HUP挂起
      - POLLHUP，管道写端被关闭，管道读端收到POLLHUB事件
      - POLLRDHUP，TCP连接被对端关闭或对端关闭写操作，需定义_GNU_SOURCE宏
    - ERR
      - POLLERR / POLLNVAL，错误/文件描述符没有打开
```cpp
#include <poll.h>
int poll( struct pollfd* fds, nfds_t nfds, int timeout );

struct pollfd {
    int fd;
    short events;
    short revents;
};

typedef unsigned long int nfds_t;
```

- epoll系统调用
  - 把文件描述符上的注册事件集放在内核的一个事件表里，用一个文件描述符来标识事件表
  - epoll使用多个函数，将管理事件表epoll_ctl()和从事件表返回就绪epoll_wait()的动作分开
  - epoll_create调用
    - size参数，表示事件表需要多大，但只是个提示
    - 调用返回内核事件表标识符
  - epoll_ctl调用
    - fd参数，表示要操作的文件描述符
    - op参数，表示操作的类型
      - EPOLL_CTL_ADD
      - EPOLL_CTL_MOD，修改fd上的注册事件
      - EPOLL_CTL_DEL
    - event参数，用于指定事件
      - epoll_data结构的fd成员和ptr成员无法同时使用，ptr可以指向与fd和fd相关的用户数据
  - epoll_wait调用
    - 在一段超时时间内，等待一组文件描述符上的事件
    - events参数，存放epoll_wait返回的的就绪事件集合
      - 如果检测到事件，将就绪事件从内核时间表复制到events指向的数组中
    - maxevents参数，指定最多监听的事件
  - LT模式(level trigger)
    - 当epoll_wait将就绪事件通知用户进程，用户进程可以不立即处理该事件，再次调用，epoll_wait将继续通知此事件，直到事件被处理
  - ET模式(edge trigger)
    - 当epoll_wait将就绪事件通知用户进程，用户进程必须立即处理该事件，再次调用，epoll_wait将不再向用户进程通知此事件，减少了同一就绪事件被重复触发的次数，即减小了epoll_wait返回的就绪事件集合
  - EPOLLONESHOT事件
    - 对于该事件的文件描述符，系统最多触发其上注册的一个可读/可写/异常事件，且只触发一次(阅后即焚)
    - ET模式，一个事件还是可能被触发多次，当一个线程读取并开始处理请求时，此时可读就绪事件被触发，另一线程被唤醒来读取这些数据，导致出现两个线程同时操作一个socket(连接)的局面，期望的是一个socket连接在任一时刻只被一个线程处理，可以使用EPOLLONESHOT来实现
    - 注册EPOLLONESHOT事件的socket，一旦被某个线程处理完毕，该线程就应该立即重置这个socket的EPOLLONESHOT事件(重新注册事件)，以确保socket下一次可读时，EPOLLIN事件能被触发，进而让其他工作线程有机会处理这个socket
    - 一个连接可能由多个工作线程轮流处理完成，但任一时刻只有一个线程负责处理
    - 每个使用ET模式的文件描述符应该是非阻塞的，当读取不到数据时能立即返回.如果不是非阻塞，那么读或写操作会因为没有后续的事件而一直出于阻塞状态
    - EPOLLET事件，触发后，仍存在于内核事件表，只是从就绪列表中移除
    - EPOLLONESHOT事件，触发后，从内核事件表移除
```cpp
#include <sys/epoll.h>

int epoll_create( int size );
int epoll_ctl( int epfd, int op, int fd, struct epoll_event* event );
int epoll_wait( int epfd, struct epoll_event* events, int maxevents, int timeout );

struct epoll_event {
  __uint32_t events; // epoll事件
  epoll_data_t data; // 用户数据
};

typedef union epoll_data {
  void* ptr;
  int fd;
  uint32_t u32;
  uint64_t u64;
} epoll_data_t;


// poll返回全部注册事件集(就绪+未就绪),epoll返回就绪事件集
int ret = poll(fds, MAX_EVENT_NUMBET, -1);
for (int i = 0; i < MAX_EVENT_NUMBET; ++i) {
  if ( fds[i].revents & EPOLLIN ) { // 判断第i个fd是否就绪
    int sockfd = fds[i].fd;
    ...
  }
}

int ret = epoll_wait( epollfd, events, MAX_EVENT_NUMBER, -1 );
for (int i = 0; i < ret; ++i) { // 已绪，直接处理
  int sockfd = events[i].data.fd;
  ...
}

// EPOLLONESHOT事件
void addfd( int epollfd, int fd, bool enable_et ) {
  epoll_event event;
  event.data.fd = fd;
  event.events = EPOLLIN;
  if (enable_et)
    event.events |= EPOLLET;
  epoll_ctl( epollfd, EPOLL_CTL_ADD, fd, &event );
  setnonblocking(fd);
}

void let(); // 正常读取，可分多次读取

void et( epoll_event* events, int number, int epollfd, int listenfd ) {
  char buf[ BUFFER_SIZE ];
  for (int i = 0; i < number; ++i) {
    int sockfd = events[i].data.fd;
    if (sockfd == listenfd) {
      ...
      addfd( epollfd, connfd, true );
    }
    else if (events[i].events & EPOLLIN) {
      printf( "event trigger once\n" );
      while (true) { // 循环读取数据，确保单次触发把socket数据全部读出
        memset( buf, '\0', BUFFER_SIZE );
        int ret = recv(sockfd, buf, BUFFER_SIZE-1, 0 );
        if (ret < 0) { // 非阻塞IO，EAGIN表示数据全部读取完毕，待下次触发
          if ( errno == EAGIN || errno == EWOULDBLOCK ) {
            printf( "read later\n" );
            break;
          }
          close( sockfd );
          break;
        }
        else if (ret == 0) {
          close( sockfd );
        }
        else {
          printf( "get %d bytes of content: %s\n", ret, buf );
        }
      }
    }
  }
}

int main() {
  ...
  while (true) {
    int ret = epoll_wait( epollfd, events, MAX_EVENT_NUMBET, -1 );
    if (ret < 0 ) {
      printf( "epoll failure\n" );
      break;
    }
    et( events, ret, epollfd, listenfd );
  }
}

// EPOLLONESHOT事件
// 一个工作线程处理完某个socket上的一次请求，之后收到该socket上新的请求，线程将继续为这个socket服务
// 该socket注册了EPOLLONESHOT事件，使得其他线程没有机会接触到这个socket(任一时刻只有一个线程接触socket，避免竞态条件)
// reser_oneshot函数重置socket的注册事件，使得该socket的EPOLLIn事件可以被检测到，进而其他线程有机会为该socket服务
void addfd( int epollfd, int fd, bool oneshot ) {
  epoll_event event;
  event.data.fd = fd;
  evnet.events = EPOLLIN | EPOLLET;
  if (oneshot) {
    event.events |= EPOLLONESHOT;
  }
  epoll_ctl( epollfd, EPOLL_CTL_ADD, fd, &event );
  setnonblocking( fd );
}

void reset_oneshot( int epollfd, int fd ) {
  epoll_event event;
  event.data.fd = fd;
  event.events = EPOLLIN | EPOLLET | EPOLLONESHOT;
  epoll_ctl( epollfd, EPOLL_CTL_MOD, fd, &event );
}

void* worker( void* arg) {
  ...
  while (true) { // 循环读取，直到遇到EAGAIN
    int ret = recv( sockfd, buf, BUFFER_SIZE-1, 0 );
    if (ret == 0) {
      close( sockfd );
      printf( "foreiner closed the connection\n" );
      break;
    }
    else if (ret < 0) {
      if (errno == EAGAIN) {
        reset_oneshot( epollfd, sockfd ); // 重置事件/重新注册事件
        priintf( "read later\n" );
        break;
      }
    }
    else {
      printf( "get content: %s\n", buf );
      sleep(5); // 休眠5秒，模拟数据处理过程
    }
  }
}

int main() {
  ...
  // 监听socket不能是EPOLLONESHOT，否则用户进程只能处理一个连接，后续的新连接请求将不再出发listenfd上的EPOLLIN事件
  addfd( epollfd, listenfd, false ); 

  while (true) {
    ...
    for (int i = 0; i < ret; ++i ) {
      int sockfd = events[i].data.fd;
      if (sockfd == listenfd) {
        ...
        addfd( epollfd, connfd, true );
      }
      else if (events[i].events & EPOLLIN) {
        pthread_t thread;
        fds new_fds;
        new_fds.epollfd = epollfd;
        new_fds.sockfd = sockfd;
        // 启动一个工作线程为sockfd服务
        pthread_create( &thread, NULL, worker, (void*)&new_fds );
      }
      else {
        printf( "something else happend\n" );
      }
  }
  close( listenfd );
}
```
- 三个I/O复用函数的比较
  - select和poll
    - 都采用轮询的在全部文件描述符中检查就绪事件，复杂度为O(n)
    - 每次调用都传入要监听的文件描述符集和事件集，从用户空间到内核空间数据拷贝
    - select将文件描述符集和事件集分开，仅支持有限的3种事件类型。poll将文件描述集和事件集绑定为pollfd结构体，支持很多事件类型
    - select每次调用需重置要监听的文件描述符集，因内核会修改它来返回就绪的事件结果。poll不需要重置要监听的集合，poll使用revents成员来给内核填入调用的结果
- poll和epoll
  - epoll将文件描述符集和事件集绑定为epoll_event结构体
  - epoll在内核中创建事件表，提供事件表标识符供注册/修改/移除事件，每次调用仅需要传入新的注册事件或修改原有的注册事件
  - epoll采用回调的方式检测是否就绪O(1)，当文件描述符就绪时，epoll调用回调函数，将就绪文件描述符放入就绪队列，当用户进程调用epoll_wait时，才将就绪队列的数据拷贝到用户空间的event数组
  - epoll除了支持poll的所有事件类型，还支持EPOLLET和EPOLLONESHOT事件类型，这两种事件类型，减少了事件被触发的次数
- 当活跃连接很多时，epoll的回调函数会被调用很多次，此时未适宜使用epoll，当连接很多但活跃连接很少时，适宜使用epoll

- I/O复用应用：同时处理TCP和UDP
  - 创建两个socket，一个刘socket，一个数据报socket，绑定到同一个端口
```cpp

#define TCP_BUFFER_SIZE 512
#define UDP_BUFFER_SIZE 1024

int setnonblocking( int fd ) {
  int old_option = fcntl( fd, F_GETFL );
  int new_option = old_option | O_NONBLOCK;
  fcntl( fd, F_SETFL, new_option );
  return old_option;
}

void addfd( int epollfd, int fd ) {
  epoll_event event;
  event.data.fd = fd;
  event.events = EPOLLIN | EPOLlET;
  epoll_ctl( epollfd, EPOLL_CTL_ADD, fd, &event );
  setnonblocking( fd );
}

int main() {
  ...
  // 创建 TCP socket
  int listenfd = socket( PF_INET, SOCK_STREAM, 0 );
  bind( listenfd, (struct sockaddr*)&address, sizeof(address) );
  listen( listenfd, 5 );

  // 创建 UDP socket
  ...
  int udpfd = socket( PF_INET, SOCK_DGRAM, 0 );
  bind( udpfd, (struct sockaddr*)&address, sizeof(address) );
  
  addfd( epollfd, listenfd );
  addfd( epollfd, udpfd );

  while (true) {
    int number = epoll_wait( epollfd, events, MAX_EVENT_NUMBET, -1 );
    if (number < 0) {
      printf( "epoll failuer\n" );
      break;
    }
    for (int i = 0; i < number; ++i) {
      int sockfd = events[i].data.fd;
      if (sockfd == listenfd ) {
        ...
        addfd( epollfd, connfd );
      }
      else if (sockfd == udpfd) { // UDP
        char buf[ UDP_BUFFER_SIZE ];
        memeset( buf, '\0', UDP_BUFFER_SIZE );
        struct sockaddr_in client_address;
        socklen_t client_addrlength = sizeof( client_address );
        
        ret = recvfrom( udpfd, buf, UDP_BUFFER_SIZE-1, 0, (struct sockaddr*)&client_address, &client_addrlength );
        if (ret > 0) {
          sendto( udpfd, buf, UDP_BUFFER_SIZE-1, 0, (struct sockaddr*)&client_address, client_addrlength );
        }
      }
      else if (events[i].event & EPOLLIN) {
        char buf[ TCP_BUFFER_SIZE ];
        while (true) {
          memset( buf, '\0', TCP_BUFFER_SIZE );
          ret = recv( sockfd, buf, TCP_BUFFER_SIZE-1, 0 );
          if (ret < 0 ) { 
            if (errno == EAGIN || errno == EWOULDBLOCK)  break;
            close( sockfd );
            break;
          }
          else if (ret == 0)
            close( sockfd );
          else 
            send( sockfd, buf, ret, 0 );
        }
      }
      else
        printf( "something else happened\n" );
    }
    close( listenfd );
  }
}
```

## 逻辑单元

逻辑单元：
- 聚焦于逻辑单元内部的编程方法 - 有限状态机(finite state machine)
- 有限状态机 (finite state machine)
  - 有多个相互独立的状态，状态之间相互转移，状态转移由状态机内部驱动 ??
  - 当状态机进入下一轮循环，将执行新的状态对应的逻辑 ??

- 实例：HTTP请求的读取和分析
  - 判断HTTP协议头部的结束依据是遇到一个空行。没有遇到空行，则为未完整头部，须等待用户继续写数据并再次读取
  - 用两个状态机来，主状态机调用从状态机从缓冲区中读取头部协议中的一行，从状态机负责读取数据并判断数据是否完整有效，若行数据完整有效，主状态机再分析此行数据
  - 当分析完请求行，主状态机将状态转变为分析请求头部，执行分析请求头部的逻辑，再次调用从状态机从缓冲中提取行，直到请求头部分析完成或出错

```cpp
enum class Check_state { check_request_line, check_request_header };
enum class Line_status { line_ok, line_bad, line_open };
enum class Http_code { no_request, get_request, bad_request,
                        forbidden_request, internal_error, closed_connection };

Line_status parse_line( char* buff, int& cur, int& end ) {
    for (char tmp; cur < end; ++cur) {
        tmp = buff[cur];
        if (tmp == '\r') {
            if ((cur + 1) == end)  return line_open;
            else if (buff[cur + 1] == '\n') {
                buff[cur++] = '\0';
                buff[cur++] = '\0';
                return line_ok;
            }
            else return line_bad;
        }
        else if (tmp == '\n') {
            if (cur > 1 && buff[cur - 1] == '\r') {
                buff[cur - 1] = '\0';
                buff[cur++] = '\0';
                return line_ok;
            }
            else return line_bad;
        }
        else
            return line_open;
    }
}

Http_code parse_request_line( char* tmp, Check_state& stat ) {
    char* url = strpbrk(tmp, '\t'); // 截取从tmp开始到'\t'结束的子串
        if (url)  *url++ = '\0';
        else goto Error;
    char* method = tmp;
        if (strcasecmp( method, "GET" ) == 0)  printf("GET\n"); // 仅支持GET方法
        else goto Error;
    url += strspn(url, " \t"); // ???
    char* version = strpbrk(url, " \t");
        if (version)  *version += '\0';
        else goto Error;
    version += strspn(version, " \t");
        if (strcasecmp( version, "HTTP/1.1") == 0) // 仅支持http/1.1
        else goto Error;
    if (strcasecmp( url, "http://", 7) == 0) { // 检查url是否合法???
            url += 7; // ???
            url = strchr(url, '/');
        }
    if ( !url || url[0] !='/' ) goto Error;
    printf("The request URL is: %s\n", url);
    stat = chech_request_header; // 设置新状态
    return no_request; // 重置http_code
Error:
    return bad_request;
}

Http_code parse_request_header( char* tmp ) {
    if (tmp[0] == '\0')  return get_request; // 遇到空行
    else if (strcasecmp( tmp, "Host:", 5 ) == 0) { // 处理host字段
        tmp += 5;
        tmp += strspn(tmp, " \t");
        printf("the request host is: %s\n", tmp);
    }
    else // 其他头部字段不处理/无法处理
        printf("can't handle this hander\n");
    return no_request;
}

Http_code parse_http( char* buff, int& cur, Check_state& stat, int& end, int& start) {
    Line_status line_stat = line_ok;
    Http_code ret_code = no_request;
    while ( (line_stat = parse_line(buff, cur, end)) == line_ok ) {
        char *tmp = buff + start;
        start = cur; /// ???
        switch (stat) { // 状态机
            case check_request_line:
            {
                ret_code = parse_request_line(tmp, stat);
                    if (ret_code == bad_request)  return bad_request;
                break;
            }
            case check_request_header:
            {
                ret_code = parse_request_header(tmp);
                    if (ret_code == bad_request)  return bad_request;
                    else if (ret_code == get_request)  return get_request;
                break;
            }
            default: // 其他非法stat值，enum class不可能出现 ??
            {
                return interal_error;
            }
        }
        if (line_stat == line_open)  return no_request;
        else  return bad_request; // 其他line_stat状态
    }
}

static const char* response[] = { // 简化http应答报文，根据服务器处理结果给客户发生成功或失败信息
    "I get a correct result\n",
    "Something wrong\n"
};

int main() {
    ...
    char buff[ BUFFER_SIZE ];
    memset( buff, '\0', BUFFER_SIZE );
    int read_index = 0; // 已读多少字节的数据
    int checked_index = 0; // 已分析完多少字节的数据
    int start_line = 0; // 行在buff中的起始位置
    Check_state stat = check_request_line;
    while (true) {
        int data_read = recv( fd, buff + offset, BUFFER_SIZE - offset, 0 );
            if (data_read == -1) { printf("read faild\n");  break; }
            else if (data_read == 0) { printf("closed connection\n"), break; }
        read_index += data_read;
        Http_code res = parse_http( buff, checked_index, stat, read_index, start_line );
            if (res = no_request) // 尚未得到完整http请求
                continue;
            else if (res == get_request) { // 得到完整且正确的http请求
                send(fd, response[0], strlen(response[0]), 0); // 发送成功的响应
                break;
            }
            else { // 其他情况发生错误
                send(fd, response[1], strlen(response[1]), 0); // 发送失败的响应
                break; 
            }
    }
    close(fd);
    close(listenfd);
}
```


## 提高服务器性能

提高服务器性能：
- 影响服务器性能的因素
  - 系统的硬件资源
    - 核心数、内存容量
  - 系统的软件资源
    - 允许用户打开的最大文件描述符数量
  - 服务器程序本身
- 提高服务器程序本身的性能
  - 池 (pool)
    - 一组静态分配的资源集合，预先分配
    - 当处理客户请求需要资源时，直接从池中获取，无须使用系统调用分配资源
    - 当处理完一个客户请求后，把资源放回池中，无须执行系统调用释放资源
    - 避免了对内核的频繁访问
     
    - 预期应该分配多少资源问题
      - 方式一，分配"足够多"的资源
      - 方式二，预先分配一定量的资源，当资源不够用时，再动态分配并加入池中
     
    - 池的类型
      - 内存池，通常用于socket的收发缓冲
        - 当客户请求的长度超过接收缓冲的容量时，可以选择丢弃或动态扩大接收缓冲区
      - 进程池和线程池，通常用于并发编程
        - 当需要一个工作进程/工作线程来处理请求时，可以直接从进程池或线程池直接取得一个执行实体，无须调用fork或pthread_create系统调用
      - 连接池，通常用于服务器机群的内部永久连接
        - 服务器和数据库程序事先建立的一组连接，当逻辑单元需要访问数据库时，直接从连接池中取得一个连接的实体，完成数据库的访问后，将该连接返还给连接池
 
  - 数据复制
    - 避免用户代码和内核之间的数据拷贝
      - 如果应用程序不关心数据的内容，不对其做任何分析，则内核可以直接处理这些数据，应用程序无须将目标文件的内容完整读取到应用程序缓冲区中和调用send函数发送(拷贝了2次)
    - 避免用户代码内部的数据拷贝
      - 当两个进程要传输大量数据时，考虑用共享内存来在它们之间直接共享这些数据，而不是使用管道或消息队列来传递
   
  - 上下文切换
    - 上下文切换(context switch)问题，大量的进程切换或线程切换的系统开销，占用过多的CPU时间
    - 合理的解决方案是，一个线程同时处理多个客户连接，不同线程可以运行在不同的CPU上，当线程数量不大于核心数时，上下文切换就不是问题
   
  - 锁
    - 并发编程需要对共享资源进行加锁保护
    - 锁不处理任何业务逻辑，还需要访问内核资源，如果有更好的解决方案，应避免使用锁
    - 若必须使用锁，考虑减小锁的粒度，比如使用读写锁，只有当线程需要写这块内存时，系统才去锁着这块区域


## 调整内核参数

- linux内核允许通过修改文件的方式来调整内核参数
- 绝大多数内核模块在/proc/sys目录下提供了模块的配置文件供用户调整模块的属性和行为
  - 一个配置文件对应一个内核参数，参数名为文件名，参数值为文件的内容
  - /proc/sys/fs目录
    - file-max
      - 表示系统级文件描述符限制，即所有用户进程能打开的最大文件描述符数
    - max_user_watches
      - 表示单个用户的所有epoll实例能监听的事件数目
      - epoll内核事件表上的一个事件，在64位系统上，大小为160字节，32位系统为90字节 ??
  - /proc/sys/net目录
    - core/somaxconn
      - (socket_max_conn) 表示监听队列中能进入ESTABLISHED状态的socket最大数
    - ipv4/tcp_max_sys_backlog
      - 表示监听队列中，ESTABLISHED或SYN_RCVD状态的socket的最大数
    - ipv4/tcp_wmem
      - 指定socket的TCP写缓冲的最小值、默认值、最大值
    - ipv4/tcp_rmem
      - 指定写缓冲的最小值、默认值、最大值，可以通过此参数来改变接收窗口大小
    - ipv4/tcp_syncookies
      - 表示是否打开TCP同步cookies，同步cookies能防止监听socket重复接收同一个地址的新建连接请求(同步报文)，避免listen监听队列溢出(SYN风暴)
- 通过直接修改文件的值或sysctl命令来修改它们，都是临时修改，重启后消失
- 通过/etc/sysctl.conf文件中加入参数及值，并执行sysctl -p使其生效，为永远修改


## 压力测试程序

- 压力测试程序有很多种实现方式，单纯的I/O复用的施压程度是最高的
- 使用epoll来实现一个服务器的压力测试程序
  - 测试程序与服务器建立n个连接
  - 每个连接在接收到服务器的响应后，持续地发送请求给服务器，使用epoll来负责通知可读和可写事件
- 如果服务器程序足够稳定，程序将一直运行下去，并不断和压测程序交换数据

```cpp
void addfd( int epollfd, int fd ) {
  ...
  event.events = EPOLLOUT | EPOLLET | EPOLLERR; // 可写/异常事件
  epoll_ctl( epollfd, EPOLL_CTL_ADD, fd, &event );
  setnonblocking( fd );
}

// 发送请求
bool write_nbytes( int sockfd, const char* buffer, int len ) {
  ...
  while (true) {
    bytes = send( sockfd, buffer, len , 0 );
    if (bytes == -1 || bytes == 0)
      return false;
    else {
        len -= bytes;
        buffer = buffer + bytes;
        if (len <= 0) return true;
    }
  }
}

// 读取服务器响应
bool read_once( int sockfd, char* buffer, int len ) {
  int bytes = 0;
  memset( buffer, '\0', len );
  bytes = recv( sockfd, buffer, len, 0 );
  if (bytes == -1 || bytes == 0) return false;
  else {
    printf( "read %d bytes from socket %d with content: %s\n", bytes, sockfd, buffer );
    return true;
  }
}

// 发起连接
void start_conn( int epollfd, int num, const char* ip, int port ) {
  ...
  for (int i = 0; i < num; ++i) {
    sleep(1);
    int sockfd = socket( PF_INET, SOCK_STREAM, 0 );
    int ret = connect( sockfd, (struct socksaddr*)&address, sizeof(address));
    if (ret == 0) {
      printf( "build connection %d\n", i );
      addfd( epollfd, sockfd ); // 监听服务器响应
    }
  }
}

void close_conn( int epollfd, int sockfd ) {
  epoll_ctl( epollfd, EPOLL_CTL_DEL, sockfd, 0 );
  close( sockfd );
}

int main(int argc, char* argv[]) {
  assert( argc == 4);
  int epollfd = epoll_create(100);
  start_conn( epollfd, atoi(argv[3], argv[1], atoi(argv[2])));
  epoll_event events[10000];
  char buffer[2048];

  while (true) {
    int number = epoll_wait( epollfd, events, 10000, 2000); /// 2000 ??? 魔数 ??
    for (int i = 0; i < number; ++i) {
      int sockfd = events[i].data.fd;
      if (events[i].events & EPOLLIN) {
        if (!read_once( sockfd, buffer, 2048 )) { // 读取失败
          close_conn( epollfd, sockfd );
        }
        struct epoll_event event;
        event.events = EPOLLOUT | EPOLLET | EPOLLERR;
        event.data.fd = sockfd;
        epoll_ctl( epollfd, EPOLL_CTL_MOD, sockfd, &event );
      }
      else if (events[i].events & EPOLLOUT ) {
        if (!write_nbytes( sockfd,request, strlen(request)) {
          close_conn( epollfd, sockfd ); // 写入失败
        }
        struct epoll_event event;
        event.events = EPOLLIN | EPOLLET | EPOLLERR;
        event.data.fd = sockfd;
        epoll_ctl( epollfd, EPOLL_CTL_MOD, sockfd, &event );
      }
      else if (events[i].evnets & EPOLLERR) {
        close_conn( epollfd, sockfd );
      }
    }
  }
}
```


## 系统检测工具

- tcpdump
- lsof
- nc
- strace
- netstat
- vmstat
- ifstat
- mpstat
- 