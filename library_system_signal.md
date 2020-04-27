
信号：
- 信号是由用户/系统/进程发送给目标进程的信息，用来通知目标进程某件事情(如系统异常)
- linux信号的产生来源
  - 在终端terminal输入特殊的字符，如Ctrl+C发生中断信号
  - 系统异常，如浮点异常或非法内存段访问
  - 运行Kill命令/调用Kill函数
- 服务器程序必须处理或忽略一些常见信号，以免异常终止 ??

发送信号：
- pid参数
  - pid > 0， 发送给PID为pid的进程
  - pid = 0， 发送给本进程组的其他进程
  - pid = -1，发送给除init进程外的所有进程，要求有对目标进程发送信号的权限
  - pid < -1， 发送给组ID为 -pid 进程组为的所有成员 ??
- kill错误码errno
  - EINVAL(error_invaild)，无效信号
  - EPERM(permit)，没有权限
  - ESRCH(search)，目标进程或进程组不存在
```cpp
#include <sys/types.h>
#include <signal.h>
int kill( pid_t pid, int sig );
```

信号处理方式：
- 三种方式：信号处理函数、忽略信号、使用信号的默认处理方式
- 信号处理函数
  - 目标进程收到信号时，需自定义一个可重入的信号处理函数来处理它，禁止在信号处理函数内调用不安全的函数 ??
- 信号默认处理方式
  - 结束进程(Term)
    - SIGKILL，终止进程，不可被捕获或忽略
    - SIGHUP
    - SIGALRM
    - SIGTERM，终止进程
    - SIGPIPE
    - SIGIO，I/O就绪
  - 结束进程并生成核心转储文件(Core)
    - SIGFPE，浮点异常
    - SIGEGV，非法内存段引用 ??
  - 暂停进程(Stop)
    - SIGSTOP，暂停进程(Ctrl+S)，不可被捕获或忽略
    - SIGSTP，挂起进程(Ctrl+Z)
  - 继续进程(Cont)
    - SIGCONT，启动被暂停的进程(Ctrl+Q)
  - 忽略信号(Ign)
    - SIGCHLD，子进程状态发生变化(退出/暂停)
    - SIGURG，socket上收到紧急数据
```cpp
#include <signal.h>
typedef void (*__sighandler_t)(int);

#include <bits/signum.h>
#define SIG_DFL ((__sighandler_t) 0) // default
#define SIG_IGN ((__sighandler_t) 1) // ignore
```

信号中断系统调用：
- 处于阻塞状态的系统调用的进程接收到信号，且已经为其设置了信号处理函数，系统调用将被中断，errno设为EINTR(interprut)
- 默认行为是暂停进程的信号，且未为其设置信号处理函数，也会中断系统调用
- 使用sigaction为信号设置SA_RESTRART标志，可以自动重启被信号中断的系统调用

信号函数:
- signal()
  - 为信号设置信号处理函数，函数成功返回一个信号处理函数的旧值的函数指针

```cpp
#include <signal.h>

_sighandler_t signal( int sig, _sighandler_t _handler );
struct sigaction {
    _sighandler_t sa_handler;
    _sigset_t sa_mask; // 设置进程的信号掩码
    int sa_flags; // 设置收到信号时的程序行为
};
```

信号集：
- 结构体sigset_t用来表示一组信号，用整数表示的位模式，每一个位表示一个信号，并提供一组函数来设置/修改/删除和查询信号集
```cpp
#include <bits/sigset.h>
#define _SIGSET_NWORDS (1024 / (8 * sizeof(unsigned long int))) // n_words
typedef struct {
    unsigned long int __val[_SIGSET_NWORDS];
} sigset_t;

#include <signal.h>
int sigemptyset( sigset_t* _set ); // 清空信号集
int sigfillset( sigset_t* _set ); // 所有信号置位
int sigaddset( sigset_t* _set, int _signo );
int sigdelset( sigset_t* _set, int _signo );
int sigismember( const sigset_t* _set, int _signo ); // 测试位
```

进程信号掩码：
- set参数为新的掩码，oset参数为旧的掩码，函数成功返回旧的掩码
- how参数指定设置进程信号掩码的方式
  - SIG_BLOCK， 掩码为(oset | set)
  - SIG_UNBLOCK, 掩码为(oset & ~set)，set指定的信号不被屏蔽
  - SIG_SETMASK， 掩码为set
- 要始终清楚地知道进程在每个运行时刻的信号掩码，以及如何适当地处理捕获到的信号，在多线程环境，以进程或线程为单位来处理信号和信号掩码，fork的子进程继承父进程的信号掩码，但挂起信号集为空
```cpp
#include <signal.h>
int sigprocmask( int _how, const sigset_t* _set, sigset_t* oset ); // proccess
```

被挂起的信号：
- 设置进程信号掩码，被屏蔽的信号将不能被进程接收，操作系统将被屏蔽的设置为进程的一个被挂起的信号
- 当进程取消被挂起信号的屏蔽时，它能立即被进程收到
```cpp
#include <signal.h>
int sigpending( sigset_t* set ); // 获取被挂起的信号集
```

统一事件源：
- 信号是异步事件，信号在处理期间，系统暂时屏蔽信号。因此信号处理函数要尽可能快处理完，信号不被屏蔽过久
- 让信号像其他I/O事件一样被处理，即统一处理
  - 信号 -> 信号处理函数 -> 写管道 -> 读管道触发可读就绪事件 -> I/O复用报告 -> 用户进程

```cpp
void sig_handler( int sig ) {
    int save_errno = errno; // 保存
    int msg = sig;
    send( pipefd[1], (char*)&msg, 1, 0 ); // 写管道
    errno = save_errno; // 恢复，保证函数的可重入性
}

void addsig( int sig ) {
    struct sigaction sa;
    memset( &sa, '\0', sizeof(sa) );
    sa.sa_handler = sig_handler; // 定义信号处理函数
    sa.sa_flags |= SA_RESTART;
    sigfillset( &sa.sa_mask ); // 捕获所有信号 ??
    assert( sigaction( sig, &sa, NULL) != -1 );
}

int main() {
    ...
    ret = sockpair( PF_UNIX, SOCK_STREAM, 0, pipefd );
    setnonblocking( pipefd[1] );
    addfd( epollfd, pipefd[0] );

    addsig( SIGHUP );
    addsig( SIGCHLD );
    addsig( SIGTERM );
    addsig( SIGINT ); // interupt

    bool stop_server = false;
    while ( !stop_server ) {
        int number = epoll_wait( epollfd, events, MAX_EVENT_NUMBER, -1 );
        if ( (num < 0 ) && (errno != EINTR) ) { }
        for (int i = 0; i < number; ++i) {
            int sockfd = events[i].data.fd;
            if (sockfd == listenfd ) { }
            else if ( (sockfd == pipefd[0]) && (events[i].events && EPOLLIN))) {
                
            }
        }
    }
}
```