
## linux高级I/O函数

pipe/sockpair函数：
- 创建单向的管道文件，管道[0]只读，管道[1]只写
- 当管道[0]为空时，当管道[1]为空时
- 管道大小默认为65536字节，可以使用fcntl修改管道容量
- socketpair调用创建双向管道文件，这对文件描述符即可写也可读
```cpp
#include <unistd.h>
int pipie( int fd[2] ); // 数组指针参数，返回一对打开的文件描述符

int main() {
    int *p = new int[2];
    pipe(p);
    const char *write_buf = "value";
    char *read_buf = new char[sizeof(write_buf)];
    write(p[1], write_buf, sizeof(write_buf));
    read(p[0], read_buf, sizeof(read_buf));
    std::cout << read_buf;
}


#include <sys/socket.h>
#include <sys/types.h>
int socketpair( int domain, int type, int protocol, int fd[2] ); // domain只能是AF_UNIX
```

dup/dup2函数：
- 拷贝一个文件描述符，并分配新的文件描述符，引用到同一文件(磁盘文件、socket文件、管道文件)
- dup分配的新文件描述符为系统可用文件描述符的最小整数值，dup2分配的则是不比指定值小的系统可用文件描述符
- dup和dup2创建的文件描述符不继承源文件描述符的属性,如non-blocking
- 标准输入的文件描述符为0，标准输出的文件描述符为1，可以通过dup来重新关联标准输出到socket文件

```cpp
#include <unistd.h>
int dup( int file_descriptior );
int dup2( int file_descriptor_one, file_description_two );

// 将标准输出重定向到socket文件
close( STDOUT_FILENO );
dup( connfd );
printf( "abc\n" ); // 直接发送到connfd
close( connfd );
```

readv/writev函数：
- readv将数据从文件描述符读到分散的内存块(分散读)，writev将多块分散内存块的数据一并写入文件描述符(集中写)
- 相当于简化版的recvmsg/sendmsg函数

```cpp
#include <sys/uio.h>
ssize_t readv( int fd, const struct iovec* vector, int count );
ssize_t writev( int fd, const struct iovec* vector, int count );


// 发送 HTTP响应报文
static const char* status_line[2] = {
    "200 OK",
    "500 Internal server error"
};
// 判断并读入目标文件
char* file_buf;
struct stat file_stat;
if ( stat ( file_name, &file_stat ) <  0 )
    valid = false;
else {
    if ( S_ISDIR( file_stat.st_mode ) )
        valid = false;
    else if ( file_stat.st_mode & S_IROTH ) {
        int fd = open( file_name, O_RDONLY );
        file_buf = new char [ file_stat.st_size + 1 ];
        memset( file_buf, '\0', file_stat.st_size + 1 );
        if (read( fd, file_buf, file_stat.st.size + 1 ) < 0 )
            valid = false;
    }
    else 
        valid = false;
}
// 生成响应报文并发送
char head_buf( BUFFER_SIZE  );
memset( header_buf, '\0', BUFFER_SIZE );
if (valid) {
    ret = snprintf(  header_buf, BUFFER_SIZE - 1, 
                        "%s %s\r\n", "HTTP/1.1", status_line[0]); // 响应行
    len += ret;
    ret = snprintf( header_buf + len, BUFFER_SIZE-1-len, 
                    "Content-Length: %d\r\n", file_stat.st_size ); // 头部字段
    len += ret;
    ret = snprintf( header_buf + len, BUFFER_SIZE-1-len, 
                    "%s", "\r\n" ); // 空行
    struct iovec iv[2];
    iv[0].iov_base = header_buf;
    iv[0].iov_len = sizeof( header_buf );
    iv[1].iov_base = file_buf;
    iv[1].iov_len = file_stat.st_size;
    ret = writev( connfd, iv, 2 ); // 集中写
} else {
    ret = snprintf( header_buf, BUFFER_SIZE - 1,
                    "%s %s\r\n", "HTTP/1.1", status_line[1] );
    len += ret;
    ret = snprintf( header_buf + len, BUFFER_SIZE-1-len, 
                    "%s", "\r\n" );
    send( connfd, header_buf, strlen( header_buf ), 0 );
}
close( connfd );
delete [] file_buf;
```

sendfile函数：
- 在两个文件描述符之间直接传输数据，在内核中进行，避免内核缓冲区和用户缓冲区之间不必要的数据拷贝(零拷贝)
- in_fd参数要求必须是引用到磁盘文件，不能是socket文件和管道文件，out_fd必须是引用到socket文件
- 用于网络传输文件

```cpp
#include <sys/sendfile.h>
ssize_t sendfile( int out_fd, int in_fd, off_t* offset, size_t count );

sendfile( connfd, filefd, NULL, stat_buf.st_size );// 无在用户空间分配文件缓冲区，无读取文件到缓冲区操作
```

mmap/munmap函数：
- 用于在address space申请一段内存空间和释放它
- 这段内存可以用来作为进程通信的共享内存，也可以将文件直接映射到这块内存
- 描述内存的参数(start(起始地址), length, prot(访问权限), flags), 描述文件的参数(fd, offset)
  - 当start为NULL，则为系统自动分配一个内存地址
  - prot参数的值(位模式)
    - PROT_READ，可读
    - RPOT_WRITE，可写
    - PROT_EXEC，可执行
    - PROT_NONE，禁止访问
  - flags参数的值(位模式)
    - MAP_SHARED，进程间共享，对内存的修改将反映到被映射的文件
    - MAP_PRIVATE，进程私有不共享
    - MAP_ANONYMOUS，内存无文件映射(fd和offset都为NULL)，初始化全0
    - MAP_FIXED，内存起始地址start是内存页(4096)的整数倍
    - MAP_HUGETLB，按照大内存页面(/proc/meminfo)来分配存储空间

```cpp
#include <sys/mman.h>
void* mmap( void *start, size_t length, int prot, int flags, int fd, off_t offset );
int munmap( void *start, size_t length );
```

splice函数：
- 用于在两个文件描述符之间移动(move)数据，零拷贝(copy)
- 描述输入流的参数(fd_in, off_in), 描述输出流的参数(fd_out, off_out), 描述数据移动的参数(len, flags)
  - 当fd_in为管道文件时，off_in必须设为NULL。非管道文件，则off_in表示从输入流的何处开始读取数据，默认当前偏移位置
  - flags参数控制数据如何移动(位模式)
    SPLICE_F_NONBLOCK，实际效果受文件描述符本身的阻塞状态影响
- splice函数，要求fd_in和fd_out，至少一个是管道文件描述符。当管道无数据，splice读操作将失败
```cpp
#include <fcntl.h>
ssize_t splice( int fd_in, loff_t* off_in, int fd_out, loff_t* off_out, size_t len, unsigned int flags );

// 在内核空间移动数据，效果同recv和send
int pipefd[2];
ret = pipe( pipefd );
ret = splice( connfd, NULL, pipefd[1], NULL, 32768, SPLICE_F_MORE | SPLICE_F_MOVE ); // 写管道
ret = splice( pipefd[0], NULL, connfd, NULL, 32768, SPLICE_F_MORE | SPLICE_F_MOVE ); // 读管道
```

tee函数：
- 在两个管道文件描述符之间拷贝数据，不消耗数据

```cpp
#include <fcntl.h>
ssize_t tee( int fd_in, int fd_out, size_t len, unsigned int flags );

// stdinfd -> pipefd_stdout --copy--> pipefd_file -> filefd，即文件内容和标准输出内容相同
int filefd = open( argv[1], O_CREAT | O_WRONLY | O_TRUNC, 0666 );
int pipefd_stdout[2];
int ret = pipe( pipefd_stdout );
int pipefd_file[2];
ret = pipe( pipefd_file );
ret = splice( STDIN_FILENO, NULL, pipefd_stdout[1], NULL, 32768, SPLICE_F_MORE | SPLICE_F_MOVE );
ret = tee( pipefd_stdout[0], pipefd_file[1], 32768, SPLICE_F_NONBLOCK );
ret = splice( pipefd_file[0], NULL, filefd, NULL, 32768, SPLICE_F_MORE | SPLICE_F_MOVE );
close( all_fd );
```

fcntl函数：
- 用于控制文件描述符的属性和行为
- 参数cmd表示命令/操作类型
  - 复制文件描述符
    - F_DUPFD / F_DUPFD_CLOSEXEC，创建新文件描述符
  - 获取和设置文件描述符的标志
    - F_GETFD / F_SETFD
  - 获取和设置文件描述符的状态标志
    - F_GETFL  / F_SETFL
  - 管理信号
    - F_GETOWN / F_SETOWN，宿主进程的PID或进程组的组ID
    - F_GETSIG / F_SETSIG，可读可写事件的信号
  - 操作管道容量
    - F_GETPIPE / F_SETPIPE
- 参数...可以是cmd所需的参数

```cpp
#include <fcntl.h>
int fcntl( int fd, int cmd, ... );

// 将文件描述符设为非阻塞
int set_nonblocking( int fd ) {
    int old_option = fcntl( fd, F_GETFL );
    int new_option = old_option | O_NONBLOCK;
    fcntl( fd, F_SETFL, new_option );
    return old_option; // 返回旧值，可用于还原
}

// 将信号与文件描述符相关联
// 通过 fcntl(O_ASYNC异步I/O标志) 为某一文件描述符指定宿主进程，宿主进程将会捕获信号
// SIGIO 和 SIGURG 这俩信号必须与文件描述符关联后方可使用
```


## 用户信息与进程

UID和EUID:
- 真实用户UID表示进程的执行用户/当前使用用户
- 有效用户EUID表示资源的所有者。当进程的EUID与 ?? 匹配时，允许访问资源
```cpp
#include <sys/types.h>
#include <unistd.h>

uid_t getuid(); / int setuid( uid_t uid );
uid_t geteuid(); / int seteuid( uid_t uid );
gid_t getgid(); / int setgid( gid_t gid );
gid_t getegid(); / int setegid( gid_t gid );
```

进程组：
- 每个进程都隶属于一个进程组(PGID)，每个进程组都有一个首领进程(PGID=PID)
- 进程组将一直存在，直到进程组内的所有进程都退出或加入其他进程组
- 一个进程只能设置自己或其未调用exec函数的子进程的PGID

会话：
- 有关联的进程组形成一个会话(session)
- 首领进程不能调用setsid来创建会话，进程组内的其他进程调用setsid，将使其独立出去，与原进程组独立，并和原进程组创建一个会话
```cpp
#include <unistd.h>
pid_t getsid( pid_t pid ); / pid_t setsid( void );
```

系统资源限制：
- rlim_cur为建议性的限制，当超过，系统可能会发送信号来终止进程
- rlim_max为软限制的上限，只有root用户可以扩大其值
- 资源限制类型
  - RLIMIT_AS，虚拟内存
  - RLIMIT_CORE，进程核心转储文件(core dump)
  - RLIMIT_NPROC，进程数
  - RLIMIT_SIGPENDING，挂起的信号数量
  - RLIMIT_CPU，进程的CPU时间
  - RLIMIT_DATA，进程数据段(data+bss+heap)
  - RLIMIT_STACK，进程栈段
  - RLIMIT_FSIZE，文件
  - RLIMIT_NOFILE，文件描述符数量
```cpp
#include <sys/resource.h>
int getrlimit( int resource, struct rlimit* rlim );
int setrlimit( int resource, const struct rlimit* rlim );
struct rlimit {
    rlim_t rlim_cur; // 软限制, rlimt_t表示资源级别，整数
    rlim_t rlim_max; // 硬限制
};
```

改变工作目录和根目录：
- getcwd()和chdir()用于进程的工作目录
- chroot()用于改变进程的根目录，但不影响进程当前的工作目录，只有特权进程能改变根目录
- 改变进程的根目录之后，可能会造成原先的文件无法访问，但进程原先打开的文件描述符仍然有效，可以通过其访问chroot()后无法直接访问的文件，尤其是日志文件
```cpp
#include <unistd.h>
char* getcwd( char* buf, size_t size ); // buf指向存储进程工作目录绝对路径名的内存块
int chdir( const char* path );
int chroot( const char* path );
```

守护进程:

```cpp
bool deemonize() {
    // 脱离父进程
    pid_t pid = fork();
    if (pid < 0) return false
    else if (pid > 0) exit(0);
    // 脱离进程组
    pid sid = setsid();
    if (sid < 0 ) return false;
    // 切换工作目录
    chdir( "/" );
    // 关闭输入输出
    close( STDIN_FILENO );
    close( STDOUT_FILENO );
    close( STDERR_FILENO );
    // 关闭已经打开的文件描述符
    // 将输入输入重定向到/dev/null
    open( "/dev/null", O_RDONLY );
    open( "/dev/null", O_RDWR );
    open( "/dev/null", O_RDWR );
    return true;
}

#include <unistd.h>
int deamon( int nochdir, int noclose ); // 两参数都设为0，效果为改变工作目录+输入输出重定向到/dev/null

```