

## fork调用

- 用于复制进程
- 调用返回两次，父进程返回值为子进程pid，子进程返回值为0
- 子父进程的进程表项PCB的比较
  - 相同
    - 代码、堆栈指针、标志寄存器的值
  - 不同
    - PPID、信号位图
  - 可能不同
    - 数据(堆/栈/静态)复制采用写时复制(copy on write).当任一进程执行写入，将产生缺页中断，系统再给子进程分配新的内存页，进行数据复制
  - 父进程的资源引用计数+1
    - 打开的文件描述符、用户工作目录、当前工作目录
```cpp
#include <sys/types.h>
#include <unistd.h>g
pid_t fork(void);
```

## exec系列调用

- 用于替换的当前进程映像
- "exec + 参数名缩写"
  - exec + l, 接受可变参数...
  - exec + p，接受文件名，不接收路径
  - exec + e, 接受环境变量参数envp
  - exec + v，接受参数数组argv
- 当文件描述符设置了SOCK_CLOEXEC属性，exec函数才会关闭原程序打开的文件描述符

```cpp
#include <unistd.h>
extern char* environ; // 环境变量

//int exec(const char* path, const char* arg);
int execl( const char* path, const char* arg, ... ); // l指定可变参数
int execp( const char* file, const char* arg, ... ); // p指定可执行文件文件名
int execle( const char* path, const char* arg, ..., char* const envp[] ); // e指定新环境变量
int execv( const char* path, char* const argv[] ); // v指定参数数组
int execvp( const char* file, char* const argv[] );
int execve( const char* path, char* const argv[], char* const envp[] );
```

## 僵尸进程

- 系统直到父进程读取子进程的退出信息后才释放子进程的进程表项，运行结束但未释放进程表项的子进程处于僵尸态
  - [子进程退出后，父进程读取其退出状态前]
  - [父进程退出后，子进程退出前]，子进程ppid被系统设置为1(init进程)
- wait()调用和waitpid()调用用来避免产生僵尸进程或使子进程的僵尸态立即结束
- wait()调用
  - 父进程阻塞等待任一子进程结束，将子进程的退出信息存储在stat_loc参数
- waitpid()调用
  - 父进程阻塞等待指定的一个子进程，options参数为WHOHANG可以将调用设置为非阻塞
- 退出信息相关宏(if condition end)
  - WIFEXITED( stat_val ),正常退出返回非0值，此时WEXITSTATUS( stat_val )返回子进程的退出码
  - WIFSIGNALED( stat_val ),因未捕获信号退出返回非0值，此时WTERMSIG( stat_val )返回具体信号值
  - WIFSTOPPED( stat_val ),意外终止返回非0值，此时WSTOPSIG( stat_val )返回具体信号值
- 当子进程退出时，其将向父进程发送SIGCHLD信号。父进程捕获信号，在信号处理函数中调用waitpid()来彻底结束子进程
```cpp
#include <sys/types.h>
#include <sys/wait.h>
pit_t wait( int* stat_loc );
pid_t waitpid( pid_t pid, int* stat_loc, int options );

static void handle_sigchld( int sig ) {
    pid_t pid;
    int stat;
    while ( (pid = waitpid(-1, &stat, WHOHANG)) > 0 ) {
        //对结束的子进程进行善后处理
    }
}
```


## 管道

- fork()调用后，子进程继承父进程的打开文件描述符表，若父进程有管道文件描述符，子进程也将继承
- 利用父子进程同时指向同一个管道的特点来进行父子进程之间的传递数据，但只有一个管道，因此只能单向传输，父子进程都须关闭一个管道描述符的读端或写端，以符合管道逻辑
- 使用socketpair()调用可以创建全双工的管道，此时有两个管道，一个用于父进程到子进程方向，一个用于相反方向
- 管道(匿名)只能用于有关联(如父子进程)的两个进程间的通信，命名管道(FIFO)可用于无关联进程的通信，但其在网络编程中使用不多
- 管道是存在于内存中的文件，命名管道存在于文件系统/磁盘的文件 ？？
- 可以使用stract命令和lsof命令查看管道通信

```cpp
// 如何创建双向管道
// 如何创建命名管道
```


## 信号量

- 临界区是一段对共享资源的访问的代码，这段代码可能会引发进程的竞态条件
- 信号量用于实现进程对临界区的同步访问，消除竞态条件
- 信号量是一种取自然数值的特殊变量，只支持P操作(进入临界区)和V操作(退出临界区)
  - P(信号量)，如果信号量大于0，就减1； 信号量为0，则挂起进程
  - V(信号量)，如果有等待信号量被挂起的进程，则唤醒； 如无，将信号量+1
  - 进程进入临界区将信号量减1，离开临界区将信号量加1
- 最简单的信号量是二进制信号量，只能取0和1两个值
- 信号量的TAS(test_and_set)操作由系统来保证其原子性

- semget()
  - key参数标识全局唯一的信号量集，进程通过Key参数来使用或获取信号量集
    - 特殊键值IPC_PRIVATE(IPC_NEW)，值为0，无论信号量是否存在，都创建一个新的信号量(私有非共享)
  - num_sems参数，指定要创建或获取信号量集的信号量的数目
  - sem_flags参数，指定标志位的低9位为信号量的权限，格式含义与open的mode参数相同
  - 返回一个信号量标识符
```cpp
#include <sys/sem.h>

int semget( key_t key, int num_sems, int sem_flags );

struct ipc_perm { // 用于描述IPC对象(信号量/共享内存/消息队列)的权限
    key_t key;
    uid_t uid;
    git_t gid;
    uid_t cuid; // creater创建者
    git_t cgid;
    mode_t mode; // 访问权限，xxx-xxx-xxx
};

struct semid_ds { // 与信号量集关联的内核数据结构体，创建信号量集时被初始化
    struct ipc_perm sem_perm; // 信号量操作权限
    unsigned long int sem_nsems; // 信号集的信号量数目
    time_t sem_otime; // 最后一次调用semop的时间,o/op
    time_t sem_ctime; // 最后一次调用semctl的时间,c/ctl
};
```

- semop()，用于改变信号量集的单个信号量的值
  - sem_id参数，信号量集标识符
  - sem_ops参数，指定如何操作某个信号量(信号量，操作类型，操作标志)
    - sem_op大于0(V操作)，信号量的值semval增加sem_op，要求对信号量集有写权限
    - sem_op小于0(P操作)，
      - 如果semval >= |sem_op|（资源满足)，则调用成功，进程立即获得信号量并减去sem_op的绝对值
      - 否则，进程被阻塞睡眠，直到满足唤醒条件
        - 信号量的值大于等于sem_op的绝对值，semncnt--
        - 被操作的信号量所在的信号集被进程移除，errno为EIDRM(id remove)
        - 调用被信号中断，errno为EINTR，semncnt--
    - sem_op等于0，如果信号量的值是0，则调用成功，否则，阻塞进程等待信号量变为0，直到满足唤醒条件
        - 信号量的值为0，此时semzcnt--
        - 被操作的信号量所在的信号量集被进程移除，errno为EIDRM
        - 调用被信号中断，errno为EINTR，semzcnt--
    - IPC_NOWAIT标志，表示非阻塞调用
    - SEM_UNDO标志，表示系统使用semadj变量来跟踪进程的信号量修改情况
  - num_sem_ops参数，指定sem_ops结构体数组的大小，每个数组元素表示对一个信号量进行操作
```cpp
#include <sys/sem.h>
int semop( int sem_id, struct sembuf* sem_ops, size_t num_sem_ops );

// 与单个信号量相关的内核变量(私有)，由内核维护
unsigned short semval; // 信号量的值
unsigned short semzcnt; // 等待信号量值为0的进程数量
unsigned short semncnt; // 等待信号量值增加的进程数量
pit_t sempid; // 最后一次执行semop操作的进程pid

struct sembuf { // 将单个信号量如何操作的信息封装为结构体
    unsigned short int sem_num; // 信号量的编号
    short int sem_op; // 操作类型，负数表示P操作,正数表示V操作，0表示等待0操作
    short int sem_flg; // 可选值(IPC_NOWAIT, SEM_UNDO)
};
```

- semctl()，使用命令直接控制信号量
  - sem_num参数，指定被操作的信号量在信号集中的编号
  - command参数，指定要执行的命令
    - 控制信号量集
      - IPC_STAT，表示将信号量集关联的数据结构复制到semu.buf中
      - IPC_RMID，移除信号量集，唤醒所有等待该信号量集的进程
      - IPC_INFO，获取系统信号量资源配置信息，返回内核信号量集数组已使用的的项最大索引值
      - GETALL，  将信号量集的所有信号量的semval值导出到semun.array
      - SETALL，  设置信号量集的所有信号量的semval值为semun.array的数据
    - 控制单个信号量
      - SETVAL，  设置信号量的semval值为semun.val
      - GETNCNT， 获取信号量的semncnt值
      - GETZCNT， 获取信号量的semzcnt值
      - GETPID，  获取信号量的sempid值
      - GETVAL，  获取信号量的semval值
  - 可变参数由用户自定义，推荐格式为semun联合体

```cpp
#include <sys/sem.h>

int semctl( int sem_id, int sem_num, int command, ... );

// 可变参数的推荐格式
union semun { // 提供命令所需要的操作信息
    int val;                // 用于SETVAL命令
    struct semid_ds* buf;   // 用于IPC_STAT和IPC_SET命令
    unsigned short* array;  // 用于GETALL和SETALL命令
    struct seminfo* __buf;  // 用于IPC_INFO命令
};
struct seminfo {
    int semmni; // 系统最多信号量集数(ni)
    int semmns; // 系统最多信号量数(ns)
    int semmsl; // 信号量集最多的信号量数
    int semopm; // 一次最多执行的sem_op的操作数
    int semvmx; // 最大的信号量值
    int semaem; // 最多的undo次数
};
```

```cpp
// 父子进程的使用一个IPC_PRIVATE信号量同步进程示例
// 父子进程通过使用信号量集的标识符来共享IPC_PRIVATE信号量

void  pv( int sem_id, int op ) {
    struct sembuf sem_b;
    sem_b.sem_num = 0;
    sem_b.sem_op = op;
    sem_b.sem_flag = SEM_UNDO;
    semop( sem_id, &sem_b, 1 );
}

int main() {
    int sem_id = semget( IPC_PRIVATE, 1, 0666 );

    union semun sem_un;
    sem_un.val = 1; // 设信号量值为1
    semctl( sem_id, 0, SETVAL, sem_un );
    
    pid_t id = fork();
    if (id < 0) return 1;
    else if (id == 0) {
        pv( sem_id, -1 );
        printf("child get the sem\n");
        pv( sem_id, 1 );
        exit(0); //将不会访问到else后的代码
    }
    else { // parent
        pv( sem_id, -1 );
        printf("parent get the sem\n");
        pv( sem_id, 1 );
    }

    waitpid( id, NULL, 0 );
    semctl( sem_id, 0, IPC_RMID, sem_un ); // 删除信号量
}
```

```cpp
另一个例子是使用一个IPC_PRIVATE信号量来同步各子进程对epoll_wait的调用权.
进程1709暂时拥有信号量，调用epoll_wait等待新的客户连接。
当新连接到来时，进程1709接受，并对信号量393222执行V操作
此时将有另外一个子进程获得该信号量并调用epoll_wait来等待新的客户连接
```


## 共享内存

- 最高效的IPC机制，在进程之间使用共享内存，不涉及进程之间的任何数据传输
- 本身不带同步的功能，需要同步进程对共享内存的访问，适宜将共享内存用作共享读
- mmap调用与内存映射文件
- 共享内存与其他IPC机制的比较(信号量、管道、消息队列、socket)

- shmget调用
  - 创建或获取一段共享内存，所有字节初始化为0，与之关联的内核数据结构shmid_ds也被初始化
  - key参数，用一个键值来标识全局唯一的共享内存
  - size参数，表示以字节为单位表示共享内存的大小
  - shmflg参数，使用和含义同sem_flags，支持额外两个标志
    - SHM_HUGETLB, 系统将使用"大页面"来为共享内存分配空间
    - SHM_NORESERVE，不为共享内存保留交换空间(swap空间)，当物理内存不足时，对共享内存写操作将出发SIGSEGV信号
  - 返回共享内存的标识符\
```cpp
#include <sys/shm.h>
int shmget( ket_t key, size_t size, int shmflg );

struct shmid_ds { // 内核管理共享内存的data_structure
    struct ipc_perm shm_perm;
    size_t shm_segsz; // 共享内存大小size
    __time_t shm_atime; // 最后一次调用shmat(attach)的时间
    __time_t shm_dtime; // 最后一次调用shadt(detach)的时间
    __time_t shm_ctime; // 最后一次调用shmctl的时间
    __pid_t  shm_cpid; // 创建者(creater)的pid
    __pid_t  shm_lpid; // 最后一次执行shmat或shmdt的进程pid
    shmatt_t shm_nattach; // 关联到共享内存的进程数量
}
```

- shmat和shmdt调用
  - 共享内存使用前，需先将它关联到进程的地址空间中，使用后，将它从个进程地址空间中分离
  - shm_addr参数
    - NULL，则关联地址由系统决定，确保代码的可移植性
    - 非空且未设置SHM_RND标志，则关联到指定地址
    - 非空且设置SHM_RND(round)标志，关联的地址为 (shm_addr - (shm_addr % SHMLBA))
      - SHMLBA(low boundary address multiple)，表示关联地址为向下圆整的内存页面的整数倍地址
  - shmflg参数
    - SHM_RND(round)，向下取圆整
    - SHM_RDONLY(read-only)，只读
    - SHM_REMAP(re-map)，重新关联
    - SHM_EXEC(execute)，执行权限，同读权限
  - 返回被关联的地址，调用成功将修改shmid_ds的部分字段
    - shm_nattach
    - shm_lpid
    - shm_atime/shm_dtime
```cpp
#include <sys/shm.h>
void* shmat( int shm_id, const void* shm_addr, int shmflg );
int shmdt( const void* shm_addr );
```

- shmctl调用
  - 控制共享内存的某些属性
  - command参数
    - IPC_STAT，拷贝内核数据结构shmid_ds到buf
    - IPC_SET，拷贝buf的部分成员到内核数据结构shmid_ds
    - IPC_RMID，将共享内存标记为删除，由最后一个shmdt它的进程负责删除它
    - IPC_INFO，同sem
    - SEM_STAT，此时shm_id表示内核共享内存信息数组的索引
    - SEM_LOCK，禁止共享内存被移动到交换分区
    - SEM_UNLOCK
  - 返回值取决于command参数
```cpp
#include <sys/shm.h>
int shmctl( int shm_id, int command, struct shmid_ds* buf );
```

- POSIX共享内存对象
  - 利用mmap在无关进程(非父子)之间共享内存，无须任何文件支持，须文件先映射后关闭 ??
  - shm_open调用
    - 用法用open调用，以使用文件的方式来使用共享内存对象
    - name参数，以/开头，以\0结尾, 如"/somename"
    - oflag参数，指定创建方式，按位或
      - O_RDONLY，只读打开共享内存对象
      - O_RDWR，以可读、可写方式打开共享对象
      - O_CREAT | O_EXCL，创建一个新共享对象，要求未存在
      - O_TRUNC，若共享对象已存在，则长度截断为0 ?? 涉及mmap调用的映射内存大小与文件大小的关系问题
    - 函数返回一个文件描述符，用于后续mmap调用，将共享内存关联到进程
    - 编译时，指定链接选项 -lrt
```cpp
#include <sys/mman.h>
#include <sys/stat.h>
#include <fcntl.h>

int shm_open( const char* name, int oflag, mode_t mode );
int shm_unlink( const char* name ); // 标记为等待删除，在引用计数为0，系统负责销毁
```

共享内存使用示例：
- 一个子进程负责一个连接
- 所有连接的读缓冲都在一块共享内存上

```cpp
#include <sys/wait.h>
#include <errno.h>
#include <sys/mman.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <signal.h>
#include <sys/socket.h>
#include <sys/epoll.h>
#include <unistd.h>

struct client_data { // 信息结构体，将处理连接connfd需要的信息封装为结构体
    sockaddr_in address;
    int connfd;
    pid_t pid;
    int pipefd[2]; // 与父进程通信用的管道
};

void child_term_handler( int sig ) { // SIGCHLD信号处理函数
    stop_child = true;
}

void addsig( int sig, void(*handler)(int), bool restart = true ) {
    ...
    if (restart) 
        sa.sa_flag |= SA_RESTART;
    ...
}


// 子进程负责I/O读写，操作完成告知主进程
int run_child( int idx, client_data* users, char* share_mem ) {
    ...
    addfd( child_epollfd, connfd );
    int pipefd = users[idx].pipefd[1]; // 写管道
    addfd( child_epollfd, pipefd );
    addsig( SIGTERM, child_term_handler, false ); // 接收主进程发送的SIGTERM信号

    while (!stop_child) {
      int number = epoll_wait( child_epollfd, events, MAZ_EVENT_NUMBET, -1 );
      if ( (number < 0) && (errno != EINTR) ) { // 非中断系统调用出错
        printf( "epoll failure\n" );
        break;
      }

      for (int i = 0; i < number; ++i) {
        int sockfd = events[i].data.fd;
        // 子进程负责的连接有数据到达，读取完成(connfd->share_mem)通过管道通知主进程
        if ( (sockfd == connf) && (events[i].events & EPOLLIN) ) {
          memset( share_mem + idx * BUFFER_SIZE, '\0', BUFFER_SZE );
          ret = recv( connfd, share_mem + id * BUFFER_SIZE, BUFFER_SIZE - 1, 0 );
          if (ret < 0 && errno != EAGAIN )
            stop_child = true;
          else if (ret == 0)
            stop_child = true;
          else
            send( pipefd, (char*)&idx, sizeof(idx), 0 ); // 发送通知给主进程
        }
        // 主进程通过管道通知子进程，发送数据(share_mem->connfd)到子进程负责的连接
        else if ( (sockfd == pipefd) && ( events[i].events & EPOLLIN) ) {
          int client = 0;
          ret = recv( sockfd, (char*)&client, sizeof(client), 0 );
          if (ret < 0 && errno != EAGAIN)
            stop_child = true;
          else if (ret == 0)
            stop_child = true;
          else {
            send( connfd, share_mem + client * BUFFER_SIZE, BUFFER_SIZE, 0 );
          }
        }
        else // 其他进程的就绪socket(共用epoll)
          continue;
      }
      
      close( connfd );
      close( pipefd );
      close( child_epollfd );
}


// 主进程负责监听新连接，并fork子进程来处理客户连接
int main() {
  ...

  int user_count = 0; // 用户连接数
  client_data* users = new client_data[ USER_LIMIT + 1 ]; // 用户连接表
  int* sub_process = new int[ PROCESS_LIMIT ]; // 子进程管理表
  for (int i  = 0; i < PROCESS_LIMIT; ++i)  sub_process[i] = -1; // 空表
  ...
  // 初始化信号处理
  int sig_pipefd[2];
  ret = socketpair( PF_UNIX, SOCK_STREAM, 0, sig_pipefd );
  assert( ret != -1 );
  setnonblocking( sig_pipefd[1] );
  addfd( epollfd, sig_pipefd[0] );
  addsig( SIGCHLD, sig_handler );
  addsig( SIGTERM, sig_handler );
  addsig( SIGINT,  sig_handler );
  addsig( SIGPIPE, sig_IGN );
  bool stop_setver = fase;
  bool terminate = false;

  // 分配共享内存，所有用户连接缓冲区共用，相当于把所有缓冲区集中到一块内存区域
  static const char* shm_name = "/my_shm";
  int shmfd = shm_open( shm_name, O_CREAT | O_RDWR, 0666 );
  assert( shmfd != -1 );
  ftruncate( shmfd, USER_LIMIT * BUFFER_SIZE );
  char* share_mem = (char*)mmap( NULL, USER_LIMIT * BUFFER_SIZE, PROT_READ | PROT_WRITE, MAP_SHARED, shmfd, 0 );
  assert( share_mem != MAP_FAILED );
  close( shmfd ); // 作用 ?? 匿名只有父子进程可见 ??

  while (!stop_server ) {
    int number = epoll_wait( epollfd, events, MAX_EVENT_NUMBER, -1 );
    if ( (number < 0) && (errno != EINTR) ) {
      printf( "epoll failure\n" );
      break;
    }

    // 遍历所有就绪事件
    for (int i = 0; i < number; ++i) {
      int sockfd = events[i].data.fd;

      // 新连接事件
      if (sockfd == listenfd) { 
        ...
        if (user_count >= USER_LIMIT) {
          const char* info = "too many users\n";
          send( connfd, info, strlen(info), o );
          close( connfd );
          continue;
        }

        users[user_count].address = client_address;
        users[user_count].connfd = connfd;

        socketpair( PF_UNIX, SOCK_STREAM, 0, users[user_count].pipefd ) ; // 子进程将继承管道
        pid_t pid = fork();
        if (pid < 0) {
          close( connfd );
          continue;
        }
        else if (pid == 0) { // 子进程自动继承父进程的文件描述符
          close( epollfd );
          close( listenfd );
          close( users[user_count].pipefd[0] );
          close( sig_pipefd[0] );
          close( sig_pipefd[1] );
          run_child( user_count, users, share_mem );
          munmap( (void*)share_mem, USER_LIMIT * BUFFER_SIZE );
          exit( 0 );
        }
        else { // 父进程
          close( connfd ); // 只负责监听新连接
          close( users[user_count].pipefd[1] );
          addfd( epollfd, users[user_count].pipefd[0] ); // 监听子进程的管道通信
          users[user_count].pid = pid;
          sub_process[pid] = user_count; // 子进程管理表
          ++ user_count;
        }
      }

      // 信号事件，统一事件源
      else if ( (sockfd == sig_pipefd[0]) && (events[i].events & EPOLLIN) ) {
        int sig;
        char signals[1024];
        ret = recv( sig_pipefd[0], signals, sizeof(signals), 0 ); 
        if (ret == -1 || ret == 0)
          continue;

        for (int i = 0; i < ret; ++i) { //遍历信号数组
          switch( signals[i] ) {
            case SIGCHILD: // 读取子进程退出状态，销毁子进程相关资源
            {
              pid_t pid;
              int stat;
              while ( (pid = waitpid(-1, &stat, WHOHANG)) > 0 ) {
                int del_user = sub_process[pid];
                sub_process[pid] = -1;
                if ( (del_user < 0) || (del_user > USER_LIMIT) )
                  continue;
                epoll_ctl( epollfd, EPOLL_CTL_DEL, users[del_user].pipefd[0], 0 );
                close( users[del_user].pipefd[0] );
                users[del_user] = users[--user_count];
                sub_process[ users[del_user].pid ] = del_user; 
              }
              if (terminate && user_count == 0)  stop_server = true;
              break;
            }
            case SIGTERM: case SIGINT: // 终止服务器程序
            {
              printf( "kill all the child now\n" );
              if (user_count == 0) {
                stop_server = true;
                break;
              }
              for (int i = 0; i < user_count; ++i) {
                int pid = users[i].pid;
                kill( pid, SIGTERM );
              }
              terminate = true;
              break;
            }
            default: // 其他信号不处理
            {
              break;
            }
          }
        }
      }

      // 可读事件(pipefd)，子进程通过管道通信通知父进程
      else if (events[i].events & EPOLLIN ) {
        int child = 0; // 子进程在子进程表的下表索引 ??
        ret = recv( sockfd, (char*)&child, sizeof(child), 0 );
        printf( "read data from child process across pipe\n" );
        if (ret == -1 || ret == 0) continue;

        // 通过管道通知其他所有子进程，有客户数据要写
        for (int i = 0; i < user_count; ++i) {
          if (users[i].pipefd[0] != sockfd  {
            printf( "send data to child process across pipe\n" );
            send( users[i].pipefd[0], (char*)&child, sizeof(child), 0 );
          }
        }
      }
    }
  }
  close( sig_pipefd[0] );
  close( sig_pipefd[1] );
  close( listenfd );
  close( epollfd );
  shm_unlink( shm_name );
  delete [] users;
  delete [] sub_process;
}
```

## 消息队列

- 在进程之间传递带类型信息的消息的简单有效的方式，接收方可以指定类型来接收特定的消息

- msgget调用
  - 用于创建或获取一个消息队列，返回消息队列标识符，与之关联的内核数据结构msqid_ds也被初始化
  - key参数，标识全局唯一的消息队列
```cpp
#include <sys/msg.h>
int msgget( key_t key, int msgflg );

struct msqid_ds { // message queue
  struct ipc_perm msg_perm; // 消息队列的操作权限
  time_t msg_stime; // 最后一次调用msgsnd的时间, snd
  time_t msg_rtime; // 最后一次调用msgrcv的时间, rcv
  time_t msg_ctime; // 最后一次修改的时间, change
  unsigned long __msg_cbytes; // 消息队列已有的字节数,count
  msqqnum_t msg_qnum; // 消息队列已有的消息数,
  msglen_t  msg_qbytes; // 消息队列允许的最大字节数
  pid_t msg_lspid; // 最后执行msgsnd的进程pid
  pit_t msg_lrpid; // 最后执行msgrcv的进程Pid
};
```

- msgsnd调用
  - 用于把一条消息添加到消息队列中，成功修改msqid_ds部分字段
    - msg_qnum ++
    - msg_lspid为当前调用进程pid
    - msg_stime为当前时间
  - msg_ptr参数，指向一个准备发送的消息
  - msg_sz参数表示消息的数据部分(mtext)长度，为0则表示没有消息数据
  - msgflg参数，用于控制msgsnd的行为
    - IPC_NOWAIT标志
      - 默认消息队列满，msgsnd将被阻塞，若指定IPC_NOWAIT，将msgsnd将以非阻塞调用
  - 处于阻塞状态的msgsnd调用可能被中断
    - 当消息队列被移除时，errno设为EINRM
    - 当程序收到信号时，error设为EINTR
```cpp
int msgsnd( int msqid, const void* msg_ptr, size_t msg_sz, int msgflg );

struct msgbuf { // 消息定义
  long mtype; // 消息类型
  char mtext[512]; // 消息数据
};
```

- msgrcv调用
  - 用于从消息队列获取消息，调用成功修改mqsid_ds部分字段
  - msg_ptr参数，用于存储接收的消息
  - msgtype参数，指定接收何种类型的消息
    - msgtype = 0，读取消息队列的第一个消息
    - msgtype > 0，读取第一个类型为msgtype的消息
    - msgtype < 0，读取第一个类型值比msgtype的绝对值小的消息 ?? 意义 ??
  - msgflg参数
    - IPC_NOWAIT标志，如果消息队列为空，error设为ENOMSG
    - MSG_EXCEPT标志，如果msgtype>0，则读取第一个非msgtype类型的消息
    - MSG_NOERROR，如果消息的数据部分过长(msg_sz)，则截断
  - 处于阻塞状态可能被中断
    - 当消息队列被移除
    - 当程序接收到信号
```cpp
int msgrcv( int msqid, void* msg_ptr, size_t msg_sz, long int msgtype, int msgflg );
```

- msgctl调用
  - command参数，指定要对消息队列执行的命令
    - IPC_STAT、IPC_SET， msqid_ds <-copy-> buf
    - IPC_RMID， 移除消息队列，唤醒所有等待读消息和写消息的进程
    - IPC_INFO， 获取消息队列资源配置信息
    - MSG_INFO， 返回已经分配的消息队列占用的资源信息
    - MSG_STAT， 返回内核消息队列信息数组[msqid]的消息队列标识符
```cpp
int msgctl( int msqid, int command, struct msqid_ds* buf );
```

## IPC命令

- ipcs命令(s)
  - 查看当前系统的信号量、共享内存、消息队列的使用情况
- ipcrm命令(remove)
  - 删除系统中的共享资源


## IPC对象的比较

- 管道和命名管道
  - 都是字节流模型
  - 管道生命周期同进程，命名管道声明周期独立于进程
  - 管道是内存中的文件，命名管道是文件系统或磁盘的文件
  - 管道是匿名的，用于有亲缘关系的进程之间，命名管道可用于无亲缘关系的进程之间
- 消息队列和管道
  - 消息队列是消息对象模型，每个消息有类型有大小
  - 消息队列生命周期独立于进程，随内核
  - 写消息队列，接收方不须要预先在消息队列上阻塞等待，读同理
- 信号量
  - 同步进程对临界区的访问
- 共享内存和消息队列管道
  - 共享内存使用mmap内存映射到进程的地址空间，存在于用户空间，消息队列和管道存在于内核空间
  - 对共享内存读写无须系统调用，消息队列和管道则需要

```cpp
// 使用unix_socket在两个进程间传递文件描述符(辅助数据)
// 子进程打开一个文件描述符，然后将它传给父进程，父进程接收读取该文件描述符来获取这个文件的内容

#include <unistd.h>
#include <fcntl.h>
#include <sys/socket.h>

static const int CONTROL_LEN = CMSG_LEN( sizeof(int) );

// 发送文件描述符，fd参数表示unix_socket，fd_to_send参数表示被发生的文件描述符
void send_fd( int fd, int fd_to_send ) {
  char buf[0];
  struct iovec iov[1];
  iov[0].iov_base = buf;
  iov[0].iov_len = 1;

  struct msghdr msg;
  msg.msg_name = NULL;
  msg.msg_namelen = 0;
  msg.msg_iov = iov;
  msg.msg_iovlen = 1;

  cmsghdr = cm;
  cm.cmsg_len = CONTROL_LEN;
  cm.cmsg_level = SOL_SOCKET;
  cm.cmsg_type = SCM_RIGHTS;
  *(int*) CMSG_DATA(&cm)) = fd_to_send;// ??
  msg.msg_control = &cm;
  msg.msg_controllen = CONTROL_LEN;
  sendmsg( fd, &msg, 0 );
}


// 接收文件描述符
int recv_fd( int fd ) {
  char buf[0];
  struct iovec iov[1];
  iov[0].iov_base = buf;
  iov[0].iov_len = 1;

  struct msghdr msg;
  msg.msg_name = NULL;
  msg.msg_namelen = 0;
  msg.msg_iov = iov;
  msg.msg_iovlen = 1;

  cmsghdr cm;
  msg.msg_control = &cm;
  msg.msg_controllen = CONTROL_LEN;
  recvmsg( fd, &msg, 0 );

  int fd_to_send = *(int*)CMSG_DATA(&cm);
  return fd_to_send;
}

int main() {
  int pipefd[2];
  socketpair( PF_UNIX, SOCK_STREAM, 0, pipefd );
  pid_t pid = fork();
  if (pid == 0) {
    close( pipefd[0] );
    fd_to_pass = open( "test.txt", O_RDWR, 0666 );
    send_fd( pipefd[1], (fd_to_pass > 0) ? fd_to_pass : 0 ); //失败将发送标准输入文件描述符
    close( fd_to_pass );
    exit( 0 );
  }
  close( pipefd[1] );
  fd_to_pass = recv_fd( pipefd[0] ); // 通过管道接收子进程发来的文件描述符
  char buf[1024];
  memset( buf, '\0', 1024 );
  read( fd_to_pass, buf, 1024 );
  printf( "get fd %d and data %s\n", fd_to_pass, buf );
  close( fd_to_pass );
}
```

## 多线程

- 线程是一个完成一个独立任务的完整执行序列，一个可调度实体
  - 内核线程(light Weight Process)
    - 运行在内核空间，完全由内核管理调度
    - 当线程阻塞时，内核自动调度同一进程的其他就绪线程或其他进程的线程
    - 线程是最小的调度单位，同一进程的线程可以运行在多个不同的CPU上
  - 用户线程
    - 运行在用户空间，完全由线程库管理调度
    - 创建和调度无须内核参与，不占用额外的内核资源，线程表在用户空间
    - 当线程阻塞时，使用yield命令来通知线程库调度其他就绪的用户线程
    - 内核把进程当成最小调度单位，进程的所有用户线程共享该进程的所有时间片
  - 混合调度
    - 结合前两种方式的优点，内核负责调度内核线程，线程库负责调度用户线程
    - 内核线程是用户线程的容器，当一个内核线程被调度执行时，就加载并运行一个用户线程，用户线程被映射到内核线程


## POSIX线程API

- pthread_create调用
  - thread参数，新线程的标识符. linux的几乎所有的资源标识符都是一个整数
  - attr参数，用于设置新线程的属性，默认值NULL
  - start_routine和arg参数，分别指定运行函数及函数参数
  - 成功返回0，失败返回错误码
- pthread_exit调用
  - retval参数，用于获取线程的退出信息
  - 调用不返回，不会失败
```cpp
#include <pthread.h>
#include <bits/pthreadtypes.h>

typedef unsigned long int pthread_t;

int pthread_create( pthread_t* thread, const pthread_attr_t* attr, void* (*start_routine)(void*), void* arg );
void pthread_exit( void* retval );
```

- pthread_join调用
  - 用于回收线程，要求被回收的线程是可回收的
  - 若被回收线程未结束，调用将阻塞，直到被回收的线程结束，类似waitpid调用
  - 成功返回0，失败返回错误码
    - EDEAKLK，可能引起死锁，dead lock
    - EINVAL，线程不可回收，invaild
    - ESRCH， 目标线程不存在, search
- pthread_cancel调用
  - 取消线程/终止线程，由接收到取消请求的目标线程决定是否取消以及如何取消
  - state参数
    - PTHREAD_CANCEL_ENABLE，默认状态
    - PTHREAD_CANCEL_DISABLE，当收到取消请求，请求将被线程挂起
  - type参数
    - PTHREAD_CANCEL_ASYNCHRONOUS，立即取消
    - PHTREAD_CANCEL_DEGERRED，延迟取消，直到取消点函数被调用
      - 使用pthread_testcancel调用来设置取消点
```cpp
int pthread_join( pthread_t thread, void** retval );

int pthread_cancel( pthread_t thread );
int pthread_setcancelstate( int state, it  *oldstate );
int pthread_setcanceltype( int type, int* oldtype );
```

- 线程属性
  - 线程库提供一系列函数来操作pthread_attr_t类型变量，来获取和设置线程属性对象
  - 初始化线程属性对象
    - pthread_attr_init()
  - 销毁线程属性对象
    - pthread_attr_destroy()
  - 获取和设置线程属性对象
    - pthread_attr_getdetachstate() / setdetachstate() // 脱离状态
      - PTHREAD_CREATE_JOINABLE， 默认值
      - PTHREAD_CREATE_DETACH， 脱离线程，线程退出时自动释放资源
    - pthread_attr_getstackaddr() // 线程栈的起始地址
    - pthread_attr_getstacksize()
      - 使用ulimit -s 来查看或修改默认值
    - pthread_attr_getstack() 
    - pthread_attr_getguardsize() // 在栈尾部添加gurrdsize大小的空间
    - pthread_attr_getschedparam() // 调度参数
      - shced_priority，线程运行优先级
    - pthread_attr_getschedpolicy() // 调度策略
      - SCHED_FIFO
      - SCHED_RR，时间片轮转
      - SCHED_OTHER， 默认值
    - pthread_attr_getinheritsched() // 继承调度属性
      - PTHREAD_INHERIT_SCHED， 新线程继承创建者的线程调度参数
      - PTHREAD_EXPLICIT_SCHED，显式指定调度参数
    - pthread_attr_getscope() // 线程优先级的作用域
      - PTHREAD_SCOPE_SYSTEM， 与系统所有线程竞争CPU（linux仅支持)
      - PTHREAD_SCOPE_PROCESS， 与同一进程的线程竞争CPU
```cpp
#include <bits/pthreadtypes.h>

typedef union {
  char __size[ __SIZEOF_PTHREAD_ATTR_T ];
  long int __align;
} pthread_attr_t;

#include <pthread.h>

int pthread_attr_init( pthread_attr_t* attr );
int pthread_attr_destroy( pthread_attr_t* atrr );

int pthread_attr_getdetachstate(const pthread_attr_t* attr, int* detachstate );
int pthread_attr_setdetachstate( pthread_attr_t* attr, int detachstate );
...
```

## 线程同步

- POSIX信号量
  - sem_init调用
    - 初始化一个未命名的信号量
    - pshared参数，指定信号量的类型
      - 为0，表示局部信号量
      - 非0，表示多进程个共享信号量
  - sem_destroy调用
    - 用于销毁信号量，释放资源
  - sem_wait调用
    - 以原子操作的方式将信号量减1，信号量为0，调用将阻塞
  - sem_trywait调用
    - sem_wait的非阻塞版本
  - sem_post调用
    - 以原子操作方式将信号量加1，信号量大于0，唤醒等待信号量的线程
```cpp
#include <semaphore.h>

int sem_init( sem_t* sem, int pshared, unsigned int value );
int sem_destroy( sem_t* sem );
int sem_wait( sem_t* sem );
int sem_trywait( sem_t* sem );
int sem_post( sem_t* sem );

sem_t ??
```

- 互斥锁(mutex)
  - 等价一个二进制信号量，用于同步对临界区的访问
  - 加锁，等价于P操作。解锁等价于V操作
  - pthread_mutex_lock调用
    - 对已经加锁的互斥锁再次加锁，调用将阻塞，直到锁的占有者解锁
  - pthread_mutex_trylock调用
    - 非阻塞版本，当互斥锁已经被加锁时，返回错误码EBUSY

```cpp
#include <pthread.h>

int pthread_mutex_init( pthread_mutex_t* mutex, const pthread_mutexattr_t* mutexattr );
int pthread_mutex_destroy( pthread_mutex_t* mutex );
int pthread_mutex_lock( pthread_mutex_t* mutex );
int pthread_mutex_trylock( pthread_mutex_t* mutex );
int pthread_mutex_unlock( pthread_mutex_t* mutex );

pthread_mutex_t 结构体 ??

pthread_mutex_t mutex = PTHREAD_MUTEX_INITIALIZER; // 互斥锁各字段初始化为0
```

- 互斥锁属性
  - 线程库提供一系列函数来操作pthread_mutexattr_t类型的变量，获取和设置互斥锁的属性
  - 初始化互斥锁属性对象
    - pthread_mutexattr_init()
  - 销毁互斥锁属性
    - pthread_mutexattr_destroy()
  - 获取和设置pshared属性
    - 参数pshard，指定是否允许跨进程共享互斥锁
      - PTHREAD_PROCESS_SHARED，跨进程共享
      - PTHREAD_PROCESS_PRIVATE，同一进程的线程共享
  - 获取和设置type属性
    - 参数type，指定互斥锁的类型，linux支持以下类型互斥类
      - PTHREAD_MUTEX_NORMAL，普通锁，默认类型
        - 当一个线程对普通锁加锁后，其余请求该锁的线程将形成一个等待队列，解锁时按线程优先级获得它，保证资源分配的公平性
        - 对已经加锁的普通锁再次加锁，将引发死锁
        - 对已经解锁的普通锁再次解锁，将导致不可预期的结果
      - PTHREAD_MUTEX_ERRORCHECK，检错锁
        - 一个线程对一个已经加锁的检错锁再次加锁，则加锁操作返回EDEAKLK
        - 对一个已经被其他线程加锁的检错锁解锁或对一个已经解锁的检错锁再次解锁，则解锁操作返回EPERM
      - PTHREAD_MUTEX_RECURSIVE，嵌套锁
        - 允许在释放锁之前多次对它再次加锁而不发生死锁
        - 其他线程如果要获得这个锁，则当前锁的拥有者必须执行相应次数的解锁操作
        - ... 返回EPERM
      - PTHREAD_MUTEX_DEFAULT，默认锁
        - 对一个已经加锁的默认锁再次加锁 或 一个已经被其他线程加锁的默认锁解锁 或 一个已经解锁的默认锁再次解锁，将导致不可预期的后果
        - 默认锁在实现时映射为上面三种锁之一
```cpp
int pthread_mutexattr_init( pthread_mutexattr_t* attr );
int pthread_mutexattr_destroy( pthread_mutexattr_t* attr );

int pthread_mutexattr_getshared( const pthread_mutexattr_t* attr, int* pshared );
int pthread_mutexattr_setshared( pthread_mutexattr_t* attr, int pshared );

int pthread_mutexattr_gettype( const pthread_mutexattr_t* attr, int* type );
int pthread_mutexattr_settype( pthread_mutexattr__t* attr, int type );
```

- 死锁
  - 死锁使得一个或多个线程被挂起而无法继续执行
  - 可能出现死锁的情况
  - 避免死锁的方法
```cpp
// 两个线程各占一个互斥锁，然后等待另一个互斥锁，两个线程僵持住，谁都无法继续往下执行，从而形成死锁
#include <pthread.h>
#include <unistd.h>

int a = 0, b = 0;
pthread_mutex_t mutex_a;
pthread_mutex_t mutex_b;

void* another( void* arg ) {
  pthread_mutex_lock( &mutex_b );
  printf( "child thread got mutex b, waiting for mutex a\n" );
  sleep(5); // 让另一个线程有充足时间获得另一个锁，否则代码或许能成功运行
  ++ b;
  pthread_mutex_lock( &mutex_a );
  b += a++; // 操作需要获取两个互斥量，在两个互斥锁的保护下操作
  pthread_mutex_unlock( &mutex_a ); // 逆序释放
  pthread_mutex_unlock( &mutex_b );
  pthread_exit( NULL );
}

int main () {
  pthread_t id;
  pthred_create( &id, NULL, another, NULL );

  pthread_mutex_init( &mutex_a, NULL );
  pthread_mutex_init( &mutex_b, NULL );

  printf( "parent thread got mutex a, waiting for mutex b\n" );
  sleep(5);
  ++a;
  pthread_mutex_lock( &mutex_b );
  a+ = b++;
  pthread_mutex_unlock( &mutex_b );
  pthread_mutex_unlock( &mutex_a );

  pthread_join( id, NULL );
  pthread_mutex_destroy( &mutex_a );
  pthread_mutex_destroy( &mutex_b );
}
```

- 条件变量
  - 提供了一种在线程间的通知机制，当某个共享数据达到某个值时(满足条件)，唤醒等待这个共享数据的线程
  - pthread_cond_intit调用
    - 用于初始化一个初始化条件变量
    - cond参数，为要操作的条件变量
    - cond_attr参数，为该条件变量的属性对象，当为NULL时，表示使用默认属性
  - pthread_cond_destroy调用
    - 用于销毁一个条件变量，释放其占用的内核资源
    - 销毁一个正在被等待的条件变量将失败，errno为EBUSY
  - pthread_cond_broadcast调用
    - 唤醒 所有 等待目标条件变量的线程
  - pthread_cond_signal调用
    - 唤醒 一个 等待目标条件变量的线程
    - 至于哪一个线程被唤醒，取决于线程优先级和调度策略
    - 实现唤醒一个指定的线程的思路
      - 定义一个全局变量设置为要唤醒的线程标识符，当唤醒所有等待该条件变量的线程时，这些线程都检查并判断被唤醒是否是自己，如果是自己则开始执行后续代码，否则返回继续等待
  - pthread_cond_wait调用
    - 用于等待目标条件变量
    - mutex参数，用于保护条件变量，确保pthread_cond_wait操作的原子性
      - 调用前互斥锁加锁， 函数执行时把调用线程放入条件变量的等待队列中，然后互斥锁解锁
      - 在pthread_cond_wait调用时，pthread_cond_signal和pthread_cond_broadcast函数不会修改条件变量，即pthread_cond_wait调用不会错误目标条件变量的任何变化 
  - 调用成功返回0， 失败返回错误码
  
```cpp
#include <pthread.h>

int pthread_cond_init( pthread_cond_t* cond, const pthread_condattr_t* cond_attr );
int pthread_cond_destroy( pthread_cond_t* cond );
int pthread_cond_broadcast( pthread_cond_t* cond );
int pthread_cond_signal( pthread_cond_t* cond );
int pthread_cond_wait( pthread_cond_t* cond, pthread_mutex_t* mutex );

ppthread_cond_t cond = PHTREAD_COND_INITIALIZER;
```

- 线程同步机制包装类
```cpp
#include <pthread.h>
#include <semaphore.h>

class sem {
public:
  sem() { sem_init( &_sem, 0, 0 ); }
  ~sem() { sem_destroy( &_sem ); }
  bool wait() { reutrn sem_wait( &_sem ) == 0; }
  bool post() { return sem_post( &_sem ) == 0; }
private:
  sem_t _sem;
};

class locker {
public:
  locker() { pthread_mutex_init( &_mutex, NULL); }
  ~lockaer() { pthread_mutex_destroy( &_mutex); }
  bool lock() { return pthread_mutex_lock( &_mutex ) == 0; }
  bool unlock() { return pthread_mutex_unlock( &_mutex ) == 0; }
private:
  pthread_mutex_t _mutex;
};

class cond {
public:
  cond() { 
    pthread_mutex_init( &_mutex, NULL );
    if (pthread_cond_init( &_cond, NULL) != 0) {
      pthread_mutex_destroy( &_mutex ); // 失败则释放锁
    }
  }
  ~cond() {
    pthread_mutex_destroy( &_mutex );
    pthread_cond_destroy( &_cond );
  }
  bool wait() {
    pthread_mutex_lock( &_mutex );
    int ret = pthread_cond_wait( &_conf, &_mutex ); // 持有锁执行cond_wait
    pthread_mutex_unlock( &_mutex );
    return ret == 0;
  }
  bool signal() {
    return pthread_cond_signal( &_cond ) == 0;
  }
private:
  pthread_mutex_t _mutex;
  pthread_cond_t _cond;
};
```

## 线程安全

- 线程安全 (thread-safe)
  - 一个函数可以在同一时刻被多个线程安全地调用
  - 解决多个线程调用函数时的数据竞争问题
- 可重入函数 (Reentrant)
  -  一个函数可以被多个线程 同时调用 且 不发生竞态条件
  -  解决函数运行结果的确定性和可重复性
  - 可重入性强于线程安全性
  - linux库函数只有一小部分是不可重入的
  - linux对很多不可重入的库函数提供了可重入版本，可重入版本的函数名在原函数名尾部加上_r
  - 在多线程程序调用库函数，要使用可重入版本，否则可能有预想不到的结果

## 线程与进程

- 当一个多线程程序调用了fork函数
  - 新创建的子进程将只有一个执行线程，它是调用fork的那个线程的复制
  - 子进程将自动继承父进程的互斥锁/条件变量的加锁状态，当子进程再次执行加锁将导致死锁
  - 子进程不清楚从父进程继承的互斥锁的状态，可以使用pthread_atfork来解决
  - pthread_atfork调用
    - prepare参数，锁住父进程的所有锁（保存锁状态不发生变化)
    - infork参数，用于在fork的父进程分支解锁
    - infork参数，用在在fork的子进程分支解锁
```cpp
#include <wait.h>

void* another( void* arg ) {
  printf( "child thread lock the mutex\n" );
  pthread_mutex_lock( &mutex );
  sleep(5);
  pthread_mutex_unlock( &mutex );
}

int main() {
  pthread_mutex_init( &mutex, NULL );
  pthread_t id;
  pthread_create( &id, NULL, another, NULL );
  sleep(1); // 父进程的主线程暂停1s，确保fork之前，父进程的子线程已经获得互斥锁

  int pid = fork();
  if (pid < 0 ) {
    pthread_join( id, NULL );
    pthread_mutex_destroy( &mutex );
  }
  else if (pid == 0) {
    printf( "the child want to get the lock\n" );
    pthread_mutex_lock( &mutex ); // 加锁操作将一直阻塞
    printf( "can't run to here\n" );
    pthread_mutex_unlock( &mutex );
    exit(0);
  }
  else {
    wait( NULL ); // 等待回收子进程
  }
  
  pthread_join( id, NULL );
  pthread_mutex_destroy( &mutex );
}

int pthread_atfork( void(*prepare)(void), void(*parent)(void), void(*child)(void) );

void prepare() {
  pthread_mutex_lock( &mutex );
}
void infork() {
  pthread_mutex_unlock( &mutex );
}
pthread_atfork( prepare, infork, infork );
```


## 线程与信号

- 每个线程都可以独立设置信号掩码，表示接收或屏蔽某些信号，线程库将根据线程的信号掩码决定将进程收到的信号发送给哪个线程
- 所有线程共享信号和信号处理函数，一个线程定义的某信号处理函数，将覆盖其他线程为该信号设置的信号处理函数
- 所有新创建的子线程都将自动继承主线程的信号掩码
- pthread_sigmask调用
  - 参数含义与sigprocmask调用的参数相同
- sigwait调用
  - set参数，指定需要等待接收的信号的集合
  - sig参数，指向用于存储sigwait函数返回的信号值
  - 当程序接收到信号时，sigwait调用和信号处理函数，只有一个起作用
- pthread_kill调用
  - 将一个信号发送给指定的线程
  - 当sig为0，不发送信号，但仍执行错误检查，因此可以用来检测调用是否失败来判断目标线程是否存在

```cpp
#include <pthread.h>
#include <signal.h>

int pthread_sigmask( int how, const sigset_t* newmask, sigset_t* oldsmsk );
int sigwait( const sigset_t* set, int* sig );
int pthread_kill( pthread_t thread, int sig );


// 定义一个线程来处理所有的信号，其他线程屏蔽所有信号
static_void *sig_thread( void *arg ) {
  sigset_t* set = (sigset_t*) arg;
  int sig;
  while (true) {
    int ret = sigwait( set, &sig );
    assert( ret == 0);
    printf( "signal handling thread got signal %d\n", sig );
  }
}

int main() {
  pthread_t thread;
  sigset_t set;
  
  sigempty( &set, SIGQUIT ); // 初始化为SIGQUIT
  sigaddset( &set, SIGUSR1 );
  int ret = pthread_sigmask( SIG_BLOCK, &set, NULL );
  assert( ret == 0 );
  pthread_create( &thread, NULL, &sig_thread, (void*)&set );
}
```


## 进程池

- 使用进程池的原因
  - 动态创建进程耗时
  - 大量细微进程切换耗cpu时间
  - fork子进程可能复制父进程系统分配资源(openfd和堆内存等)，耗可用资源
- 进程池
  - 池中所有子进程运行着相同代码，有相同属性
  - 每个子进程相对”干净"，没有从父进程继承不必要的文件描述符和大块堆内存(拷贝)
- 主进程-线程池(子进程)模式
  - 当任务到来时，主进程从进程池中选择一个子进程来处理新任务
    - 通过算法来选择，如随机算法和轮流选取算法，将任务均匀分配到子进程
    - 通过共享的请求队列，唤醒睡眠在请求队列上的子进程
  - 选择好子进程后，主进程通过某种通知机制(IPC)来告诉子进程有新任务需要处理，并传递必要的任务数据
- 使用进程池处理多请求/任务要考虑的问题
  - 主进程是否统一管理 listenfd 和 connfd
    - 若是，主进程接受新连接并获得connfd，然后将该socket传递给子进程
    - 若否，主进程仅通知新连接事件，由子进程来接受新连接并获得connfd，主进程无须传递connfd
  - 一个客户连接上的所有请求/任务是否始终由同一个子进程来负责处理
    - 若请求/任务存在上下文关系，即多个连续请求存在关联依赖，则由同一个子进程负责
    - 若请求/任务是无状态的，则可以考虑用不同的子进程来处理请求

```cpp
// 半同步-半异步进程池实现
// 为避免在进程间传递文件描述符，由子进程来接受新连接，并且一个该连接的所有请求由该个子进程来处理

struct Process {
    Process(): pid(-1) { }
    
    pid_t pid;
    int pipefd[2];
};

template <typename T>
class Process_pool {
public:
    static Process_pool<T>* create( int listenfd, int process_num ); // 单例创建接口
    ~Process_pool() { delete [] sub_process; }
    void run(); // 启动进程池
private:
    Process_pool( int listenfd, int process_num );
    void setup_sig_pipe();
    void run_parent();
    void run_child();
private:
    int _listenfd;
    int _epollfd;
    int _process_num; // 池中进程数量
    int _process_idx; // 池中进程索引值
    struct _Process *sub_process; // 子进程表
    static _Process_pool<T> *instance;
    bool _stop; // 停止子进程运行 ??
private:
    static const int max_process_num = 16；
    static const int user_per_process = 65536;
    static const int max_event_num = 10000;
};

// 初始化池中进程和通信管道
template <typename T>
Process_pool<T>::Process_pool( int listenfd, int process_num )
                        : _listenfd(listenfd),
                          _process_num(process_num),
                          _process_idx(-1),
                          _stop(false)
{
    _sub_process = new Process[ process_num ];
    for (int i = 0; i < process_num; ++i) {
        socketpair( PF_UNIX, SOCK_STREAM, 0, _sub_process[i]._pipefd );
        _sub_process[i]._pid = fork(); // n次fork
        if (_sub_process[i]._pid > 0) {
            close( _sub_process[i]._pipefd[1] ); // 父进程关闭管道写端
        } else {
            close( _sub_process[i]._pipefd[0] ); // 子进程关闭管道读端
            _process_idx = i;
            break; /// ?? 
        }
    }
}

// 将信号统一为事件，用epoll监听
template <typename T>
void Process_pool<T>::setup_sig_pipe() { 
    _epollfd = epoll_create(5);
    socketpair( PF_UNIX, SOCK_STREAM, 0, sig_pipefd );
    setnonblocking( sig_pipefd[1] );
    addfd( _epollfd, sig_pipefd[0] );
    addsig( SIGCHLD, sig_handler );
    addsig( SIGTERM, sig_handler );
    addsig( SIGPIPE, SIG_IGN );
}

template <typename T>
void Process_pool<T>::run() { // 意义何在 ??
    if (_process_idx != -1) run_child();
    else  run_parent();
}

template <typename T>
void Process_pool<T>::run_child() { // ??
    setup_sig_pipe();
    int pipefd = _sub_process[_process_idx]._pipefd[1]; // 获取子进程的管道写端
    addfd( _epollfd, pipefd );
    
    T* users = new T[ user_per_process ];
    epoll_event events[ max_event_num ];

    while (!_stop) {
        int number = epoll_wait( _epollfd, events, max_event_number, -1 );
        for (int i = 0; i < number; ++i) {
            int sockfd = events[i].data.fd;
            if ( (sockfd == pipefd) && (events[i].events & EPOLLIN) ) { // 管道读
                ...
                int connfd = accept( _listenfd, (struct sockaddr*)&client_address, &client_addrlength );
                addfd( _epollfd, connfd );
                users[connfd].init( _epollfd, connfd, client_address ); // ??
            } 
            else if ( (sockfd == sig_pipefd[0]) && (events[i].events & EPOLLIN) ) { // 信号事件
                ...
                for (int i = 0; i < ret; ++i) {
                    switch( signals[i] ) {
                        case SIGCHLD:
                        {
                            pid_t pid;
                            while ((pid = waitpid(-1, &stat, WHOHANG)) > 0);
                            break;
                        }
                        case SIGTERM: case SIGINT:
                        {
                            _stop = true;
                            break;
                        }
                        default:
                        {
                            break;
                        }
                    }
                }
            }
            else if ( events[i].events & EPOLLIN ) {
                users[sockfd].process();
            }
            else
                continue;
        }
    }
}

template <typename T> // ????
void Process_pool<T>::run_parent() {
    setup_sig_pipe();
    addfd( _epollfd, _listenfd );
    epoll_event events[ max_event_num ];
    int sub_process_cnt = 0;

    while ( !_stop ) {
        // 智障实现
    }
}
```

```cpp
// 半同步/半反应堆线程池实现
// 使用请求队列解耦主线程和子线程，主线程往请求队列写入请求/任务，子线程通过竞争来获得任务并处理
// 此线程池要求客户请求是无状态的，同一连接的不同请求可能被不同的线程处理

template <typename T>
class Thread_pool {
public:
    Thread_pool( int thread_number, int max_requests);
    ~Thread_pool();
    bool append( T* request );
private:
    int _thread_number = 8;
    int _max_requests = 10000;
    pthread_t* _threads; // 线程池
    std::list<T*> _workqueue;
    locker m_queuelocker; // 请求队列
    sem _queuestat; // 是否有任务需要处理，信号量
    bool _stop;
};

// 创建线程并detach，主线程不等待回收子线程，忙着做其他的事情
template <typename T>
Thread_pool<T>::Thread_pool( int thread_number, int max_requests )
                    : _thread_number(thread_number),
                      _max_requests(max_request),
                      _stop(false),
                      _threads(NULL)
{
    _threads = new pthread_t[ _thread_number ];
    for (int i = 0; i < thread_number; ++i) { 
        if (pthread_create( _threads + i, NULL, worker, this ) != 0 ) { // woker ?? this ???
            delete [] _threads;
        }
        if (pthread_detach( _theads[i])) {
            delete [] _threads;
        }
    }
}

template <typename T>
Thread_pool<T>::~Thread_pool() {
    delete [] _threads;
    _stop = true;
}

// 互斥锁保护请求队列，写队列前加锁，写完解锁
template <typename T>
bool Thread_pool<T>::append(T* request) { // 写请求队列
    _queuelocker.lock();
    if (_workqueue.size() > _max_requests) {
        _queuelocker.unlock();
        return false;
    }
    _workqueue.push_back( request );
    _queuelocker.unlock();
    _queuestat.post(); // 信号量V操作
```

```cpp
// 用线程池实现CGI服务器

#include "locker.h"
#include "thread_pool.h"
#include "http_conn.h" // 封装任务及其处理逻辑，作为线程池的模板实参

int main() {
  ...
  addsig( SIGPIPE, SIG_IGN ); // 为什么要忽略
  threadpoll< http_conn > *pool = new threadpool< http_conn >;
  http_conn *users = new http_conn[ max_fd ]; // http连接对象池
  ...
  setsockopt( listenfd, SOL_SOCKET, SO_LINGET, &tmp, sizeof(tmp)); // SO_LINGE，长连接
  ...
  addfd( epollfd, listenfd, false );
  http_conn::_epollfd = epollfd;

  while (true) {
    int number = epoll_wait(epollfd, events, max_event_number, -1 );
    for (int i = 0; i < number; ++i) {
      int sockfd = events[i].data.fd;
      if (sockfd == listenfd) {
        ...
        users[connfd].init( connfd, client_address ); // 初始化连接
      }
      else if (events[i].events & (EPOLLRDHUP|EPOLLHUB|EPOLLERR)) {
        users[sockfd].close_conn(); // 关闭异常连接
      }
      else if (events[i].event & EPOLLIN) {
         if (users[sockfd].read())
          pool->append(users + sockfd);
         else
          users[sockfd].close_conn(); // 异常读，关闭连接 
      }
      else if (events[i].events & EPOLLOUT) {
        if (!users[sockfd].write())
          users[sockfd].close_conn(); // 异常写，关闭连接
      }
      else { // nothing 
      }
    }
  }
}
```