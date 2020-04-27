
## socket 地址API

字节序：
- 主机字节序，指一个多字节的整数(如4字节)在内存中各个字节的先后顺序
  - 大端字节序(big endian)，整数的高位字节(23-32bit)在内存的低地址
  - 小端字节序(small endian)，整数的高位字节在内存的高地址
- 网络字节序
  - 当两台主机之间通讯时，规定发送端总是把要发送的数据转换成大端字节序再发送
  - 接收端知晓接收的数据大端字节序，其根据自己的字节序情况，决定是否进行字节序转换
- 字节序转换API
  - htol / htos (host to network)
  - ntohl / ntohs (network to host)
```cpp
void byte_order() {
    union {
        short value;
        char union_bytes[ sizeof(short) ];
    }test;
    test.value = 0x0102;
    if (test.union_bytes[0] == 0x01 && test.union_bytes[1] == 0x02)
        std::cout << "big endian\n";
    else if (test.union_bytes[0] == 0x02 && test.union_bytes[1] == 0x01)
        std::cout << "small endian\n";
    else
        std::cout << "unknown\n";

}

#include <netinet/in.h>

unsigned long int htonl(unsigned long int hostlong); // long用于转换IP地址
unsigned short int htonl(unsigned short int hostshort); // short用于转换端口号
unsigned long int ntohl(unsigned long int netlong);
unsigned short int ntohs(unsigned short int netshort);
```

socket地址：
- sockaddr结构体，用来表示socket地址
- 协议族(protocol family/domain)，通常与地址族(address_family)对应
  - PF_UNIX   - AF_UNIX   - UNIX本地域协议族    - 地址值为 文件的路径名，最长108字节
  - PF_INET   - AF_INET   - TCP/IPv4协议族     - 地址值为 address(32bits):port(16bits)，6字节
  - PF_INET6  - AF_INET6  - TCP/IPv6协议族     - 地址值为 26字节
- 通用地址结构体，在设置与获取ip和端口号需要执行繁琐的位操作
```cpp
#include <bits/socket.h>
struct sockaddr {
    sa_family_t sa_family;
    char sa_data[14];
};

// linux定义通用地址结构体
struct sockaddr_storage {
    sa_family_t sa_family;
    unsigned long int __ss_align;
    char __sspadding[128 - sizeof(__ss_align)];
};

// linux定义的专用结构体
#include <sys/un.h>
struct sockaddr_un { // unix
    sa_family_t sin_family;
    char sun_path[108];
};

struct sockaddr_in { // ipv4
    sa_family_t sin_family;
    u_int16_t sin_port;
    struct in_addr sin_addr;
};
struct in_addr { u_int32_t s_addr; }; // ipv4地址, 网络字节序表示

struct sockaddr_in6 { // ipv6
    sa_family_t sin6_family;
    u_int16_t sin6_port;
    u_int32_t sin6_flowinfo; // 流信息，应设置0
    struct in6_addr sin6_addr;
    u_int32_t sin6_scope_id; // 未使用字段
};
struct in6_addr { unsigned char sa_addr[16]; };
```

IP地址转换API:
- inet_addr， 点分十进制 -> 整数
- inet_aton， 点分十进制 -> 整数，转换结果存于inp指针
- inet_ntoa， 整数 -> 点分十进制，转换结果存于局部静态变量
```cpp
#include <arpa/inet.h>
in_addr_t inet_addr( const char* strptr );
int inet_aton( const char* cp, struct in_addr* inp ); // alpha -> numberic
char* inet_ntoa( struct in_addr in );

int inet_pton( int af, const char* src, void* dest ); // af地址族，转换结果存于dest
const char* inet_ntop( int af, const void* src, char* dest, socklen_t cnt); // cnt指定存储大小
```


## socket基础 API


- socket是可读可写可控制可关闭的文件描述符
- socket()，创建一个未绑定socket地址的socket文件描述符
- bind()，将一个socket地址与socket文件描述符绑定
- listen()，设置为监听socket，并为socket创建两个接受客户端连接的队列
- accept()，从established队列中，取出一个以客户端地址，并创建一个连接socket并绑定到客户端地址。为连接socket创建属于一对这个连接的TCP收发缓冲队列
- close()，若引用计数为0，清空对应连接的TCP收发缓冲队列，读写全关闭
- shutdown()，若关闭写，则清空TCP发送缓冲队列，并发送重置报文。若关闭读，..??
- recv(), TCP接收缓冲队列的可读字节数可能小于指定的读字节数，调用返回实际读的字节数
- send()，TCP发送缓冲队列的可写字节数可能小于指定的写字节数，未写完，用户进程被挂起，直到全写入TCP发送缓冲队列，调用返回
- recvfrom()/sendto()，UDP没有预先建立连接，因此每次发送都需要指明对端地址. udp不区分client和server的概念，双端平等。没有建立连接，没有分配单独的连接socket ?? 一次性的socket ?? recvfrom()的行为 ??
- recvmsg()/sendmsg()，提供控制用户进程的多个缓冲区的读写的行为. 从socket读入到多个分散的内存块，从多个分散的内存块集中写入socket

```cpp
#include <sys/socket.h>
#include <sys/types.h>

int socket( int domain, int type, int protocol ); // domain协议族，type服务类型，protocol应设置0
int bind( int sockfd, const struct sockaddr* my_addr, socklen_t addrlen );
int listen( int sockfd, int backlog ); // backlog表示处于established状态的连接数上限，默认为5
int accept( int sockfd, struct sockaddr *addr, socklen_t* addrlen ); // 值-结果，addr为客户端地址
int close( int fd ); // 将fd引用计数减1，引用计数为0时，才关闭连接
int shutdown( int sockfd, int howto ); // 立即终止连接，不理会引用计数, howto指定半关闭或全关闭

ssize_t recv( int sockfd, void* buf, size_t len, int flags ); // sockfd -> buf
ssize_t send( int sockfd, const void* buf, size_t len, int flags ); // sockfd <- buf

// udp，当dest_addr和addrlen设置为NULL时，为tcp读写
ssize_t recvform( int sockfd, void* buf, size_t len, int flags, const struct sockaddr* dest_addr, socklen_t addrlen );
ssize_t sendto( int sockfd, const void* buf, size_t len, int flags, const struct sockaddr* dest_addr, socklen_t addrlen );

// 通用数据读写
ssize_t recvmsg( int sockfd, struct msghdr* msg, int flags ); // 分散读 
ssize_t sendmsg( int sockfd, struct msghdr* msg, int flags ); // 集中写
struct msghdr {
    // 地址信息
    void* msg_name;
    socklen_t msg_namelen;
    // 数据信息
    struct iovec* msg_iov; // io vecotr
    int msg_iovlen; // 内存块的数量
    // 控制信息
    void* msg_control;
    socklen_t msg_controllen;
    int msg_flags;
};
struct iovec { // 表示一块内存
    void* iov_base; // 内存起始地址
    size_t iov_len; // 内存长度
};
```

获取socket地址:
- getsockname()，获取指定socket的本端地址
- getpeername()，获取指定socket的远端地址
- 若实际socket地址比address_len的所能容纳的长度还长，socket地址将被截断
```cpp
#include <sys/socket.h>
int getsockname( int sockfd, struct sockaddr* address, socklen_t* address_len );
int getpeername( int sockfd, struct sockaddr* address, socklen_t* address_len );
```

socket选项:
- getsockopt()和setsockopt()，用来获取和设置socket文件描述符属性
- 有些socket选项须在listen()调用前设置，有些在connect()调用前设置
- accept()调用返回的连接socket自动继承监听socket的socket的选项
- level (协议)
    - option_name
      - option_value + option_value_len (c内存块的描述方式）
- 当手动设置的TCP收发送缓冲区小于系统默认最小值，则为系统默认值
- 当手动设置的TCP发送缓冲区大于系统默认值，将结果为(2 * 设置值)
```cpp
#include <sys/socket.h>
int getsockopt( int sockfd, int level, int option_name, void* option_value, socklen_t* restrict option_len );
int setsockopt( int sockfd, int level, int option_name, const void* option_value, socklen_t* restrict option_len );

// 重用socket选项，当socket处于TIME_WAIT状态，其绑定的socket地址可以被重用. 被其他socket重用 ??
// 此外，可以设置 /proc/sys/net/ipv4/tcp_tw_recycle为true，使TCP连接不会进入TIME_WAIT状态达到重用socket地址的目的
setsockopt( sockfd, SOL_SOCKET, SO_REUSEADDR, &reuse, sizeof( reuse ) );

// 设置和获取收发缓冲区大小
setsockopt( sockfd, SOL_SOCKET, SO_SNDBUF, &sendbuf, sizeof( sendbuf) );
getsockopt( sockfd, SOL_SOCKET, SO_SNDBUF, &sendbuf, (socklen_t*)&len );

setsockopt( sockfd, SOL_SOCKET, SO_RCVBUF, &recvbuf, sizeof( sendbuf) );
getsockopt( sockfd, SOL_SOCKET, SO_RCVBUF, &recvbuf, (socklen_t*)&len );
```

## 网络信息 API

- 实现主机名与IP地址、服务名称与端口号的相互转换
- 获取主机完整信息
  - gethostbyname()
  - gethostbyaddr()
- 获取某个服务的完整信息
  - getservbyname()
  - getservbyport()
- 获取IP地址和端口号
  - getaddrinfo() <=> gethostbyname() + getservbyname()
- 获取名字  
  - getnameinfo() <==> gethostbyaddr() + getservbyport()
```cpp


```