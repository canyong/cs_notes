

## io_service

- 由异步操作处理器(生产者)、完成事件队列(共享队列)和完成事件的回调分发器(消费者)组成
  - 异步操作处理器负责接收并执行I/O对象提交的异步操作，向操作系统发起异步操作，由操作系统负责i/o读写
  - 异步操作完成后，由操作系统通知异步操作处理器，异步操作处理器将完成的异步操作添加到操作完成事件队列
  - 操作完成事件的回调分发器从队列取出完成事件，并通过回调通知到提交异步操作的i/o对象
- 使用run()或poll()来，从完成事件队列里取出的事件的回调handler并执行handler
  - 当多个线程调用run()，从完成事件队列取出handler时，多线程并发执行，需要上锁 ???
  - 一个线程不能拥有多个io_service实例，多个io_service之间相互等待??
  - run()
    - 当有完成事件队列为空，且有未完成的异步操作时，run调用阻塞调用run的线程
    - 如果没有未完成的异步操作，则直接返回，不阻塞
    - 如果完成事件队列非空，则取出handler，并在调用run的线程上执行handler
    - 当没有等待的异步操作和没有需要调用的完成事件队列handler，则run返回, io_service停止，可以通过不断发起异步操作的方法来让io_service持续运行
    - 可以显式使用io_service.stop或work来停止io_service
  - poll()
    - run()的非阻塞的版本
  - run_one()
    - 只执行一个handler，然后run_one()返回. run()是执行全部handler，当没有等待的异步操作和handler时，就返回
  - post()
    - post，向io_service提交handler，到完成事件handler的队列，等待io_service.run()调用它
  - dispatch()
    - 如果调用dispatch的线程为工作线程/IO线程,即调用io_service.run()的线程，则dispatch(handler)的handler被本线程立即执行，不提交到完成事件handler的队列
    - 若不是，则调用post(handler)
  - wrap(handler)，封装生成一个函数对象，但函数对象被调用时，它调用service.dispatch(handler)，wrap封装了dispatch方法
  - 同步调用不需要显式执行io_service.run()，异步调用则需要,执行io_service.run()来让io_service异步调用handler.  post()提交的handler也是被异步调用
  - 多个线程对同一个io_service对象执行run()，当有多个完成事件handler时，多线程并发调用handler
  - 使用io_service::strand对象，strand.wrap(handler)，可以保证多个完成事件handler不会被多线程并发执行,而是被顺序执行
  - 出现完成事件handler的情况
    - 定时器/计时器到期, async_wait的处理方法handler需要被调用
    - I/O操作完成(如async_read)，对应的handler需要被调用
    - 用户显式post到io_service的handler
```cpp
// 让io_service持续运行
io_service service;
tcp::socket sock(service);

void on_read();
void on_write( const::error_code& err, std::size_t bytes )
{
    sock.async_read_some( buffer(buff_read), on_read );
}
void on_read()
{
    sock.async_write_some( buffer(buff_write, bytes), on_write );
}

void on_connect()
{
    sock.async_read_some( buffer(buff_read), on_read );
}

int main() {
    tcp::endpoint ep(ip::adress::from_string("127.0.0.1", port));
    sock.async_connect( ep, on_connect );
    service.run();
}

// run_once()
service.run() <==> do service.run_once() while (!service.stoped());

write_complete = false;
async_write( sock, buffer(buff), on_write );
do service.run_once() while (!write_complete); // 可能阻塞

service.poll() <==> while (service.poll_one()); // 非阻塞

while (true) {
    while (service.poll_one()); // 调用所有完成handler
    // do other things
}

// dispatch vs post
asio::io_service service;

void run_dispatch_and_post()
{
    for (int i = 0; i < 10; i += 2) {
        service.dispatch( [=]{std::cout << i << " ";} );
        service.post( [=]{std::cout << i + 1 << " ";} );
    }
}

int main() {
    
    service.post( run_dispatch_and_post );

    service.run(); // 0,2,4,6,8,1,3,5,7,9
}


// 成员函数指针的回调函数写法
class connection: enable_shared_from_this<connection> // 开启共享connection对象
{
    void start( tcp::endpoint ep ) {
        sock.async_connect( ep, std::bind(&connection::on_connect, shared_from_this(), _1) );
    }

    void stop() {
        if (!started) return;
        started = false;
        sock.close();
    }

    tcp::socket sock;
    char read_buffer[ max_msg ];
    char write_buffer[ max_msg ];
    bool started;
};
```


## io_object

- asio::ip
  - tcp
    - endpoint, 表示通信端点，
      - (address,port)，创建一个连接的端点
      - (protocol,port)，创建一个接受新连接的端点
    - acceptor，负责绑定、监听、接受新连接，由(io_service, ep)构造
      - accept(sock)，开始监听并尝试接受新连接
      - async_accept(sock, handler)
    - socket
      - connect(ep)
      - read_some(buffer ec)
      - async_connect(ep, connect_handler)
      - 
  - udp
  - icmp
  - address
    - from_string(ip_address)
    - 
- asio::signal
  - signal_set
    - async_wait(signal_handler)
- asio::timer
  - deadline_timer(io_service, duration)
    - wait()
    - async_wait(timeout_handler)
- asio::io
  - serial_port(io_service, port_name)
    buffer(char*, size)


socket相关的io对象:

- 端点endpoint
  - ep( address, port)， 连接对端端点
  - ep( protocol, port)， 监听端点
  - ep ()， 空端点用于udp通信存储对端端点
```cpp
ip::tcp::endpoint ep;
ip::tcp::endpoint ep( ip::tcp::v4(), 80 ); // localhost:80
ip::tcp::endpoint ep( ip::address::from_string("127.0.0.1", 80) );

// 域名:port 转 ip:port
ip::tcp::resolver resolver( io_service );
ip::tcp::resolver::query query( "www.baidu.com", "80" );
ip::tcp::resolver::iterator iter = resolver.resolve( query ); // 返回endpoint类型
std::cout << *iter.to_string() << std::endl;

// 从端点获取地址、端口和ip协议信息
std::cout << "IP: " << ep.address().to_string()
          << ":"    << ep.port()
          << " / "  << ep.protocol()
          << std::endl;
```

- 套接字socket
  - tcp::socket
    - tcp::acceptor
    - tcp::iostream
    - tcp::endpoint, tcp::resolver
  - udp::socket or icmp::socket
    - udp::endpoint or icmp::endpoint
    - udp::resolver or icmp::resolver
  - socket的同步操作方法/同步方式使用socket
    - assign( protocol, native_socket )，将socket对象与一个已存在的native_socket关联
    - open( protocol )打开指定协议socket，像文件的使用方式，用于udp和icmp
    - close()，副作用，套接字上的任何异步操作都会被立即关闭
    - is_open()，检查socket是否已经打开，返回bool
    - bind( endpoint )
    - connect( endpoint )
    - shutdown( type_of_shuntdown ), 关闭读写通道，使send或receive操作失效
  - 全局函数
    - connect( socket, begin, [end], [condition] )， [begin,end)为query的返回结果
    - async_connect( socket, begin, [end], [condition], handler )
    - read( stream, buffer, [completion] ), 从表现像流的类读取, completion用于告诉read是否继续
    - write( stream, buffer, [completion] )
    - async_read( stream, buffer, [completion], handler )，completion为非0，则表示异步操作需再次执行
    - async_write( stream, buffer, [completion], handler )
  - completion函数
    - 原型：std::size_t completion( const std::error_code&, std::size_t bytes_transfered )
    - asio提供的预置completion函数
      - transfer_at_learst(n), 至少n个字节
      - transfer_exactly(n)，精确n个字节
      - transfer_all()
```cpp
ip::tcp::endpoint ep( ip::address::from_string("127.0.0.1"), 80 );
ip::tcp::socket socket( io_service );
socket.open( ip::tcp::v4() );
socket.connect( ep );
char buff[1024];
socket.read_some( buffer(buff, 1024) );
socket.shutdown( ip::tcp::socket::shutdown_receive );
socket.close();

// 全局函数
using asio::ip::tcp;
tcp::socket socket( io_service );
tcp::resolver::iterator iter = resolver.resolve( tcp::resolver::query("domain", "80") );
connect( socket, iter );
async_read( socket, buffer(buff), up_to_enter, on_read );
async_read( socket, buffer(buff), transfer_exactly(32), on_read ); // 精确从socket读取32个字节/字符

std::size_t up_to_enter( const std::error_code& ec, size_t bytes )
{
    for (std::size_t i = 0; i < bytes; ++i)
        if (buffer[ i + offset ] == '\n')
            return 0;
    return 1;
}



// socket默认禁止拷贝，当有多个用户使用时，可以使用共享指针，其他用户拷贝共享指针对象
typdef std::shared_ptr<ip::tcp::socket> socket_ptr;
socket_ptr sp1( new ip::tcp::socket(io_service) );
socket_ptr sp2 = sp1;
```

  - socket的I/O操作函数
    - 同步操作，在成功返回或出现错误之前，可能阻塞
      - tcp
        - receive( buffer, [flags] )
        - send( buff, [flags] ), // 全写
        - read_some( buffer ), 部分读
        - write_some( buffer )
        - available(), 返回可读字节数
      - udp/icmp
        - receive_from( buffer, endpoint, [flags] )
        - send_to( buffer, endpoint, [flags] )
    - 异步操作
      - tcp
        - async_receive( buffer, [flags], handler )
        - async_send( buffer, [flags], handler )
        - async_read_some( buffer, handler )
        - async_write_some
      - udp/icmp
        - async_receive_from( buffer, endpoint, [flags], handler )
        - async_send_to( buffer, endpoint, handler )
      - flags标记
        - ip::socket_type::socket
          - message_peek， 查看缓冲区，但不读取消耗数据
          - message_out_of_band， oob带外数据
          - message_do_not_route
          - message_end_of_recode，消息结束
```cpp
// 同步读写tcp_socket
ip::tcp::socket socket(io_service);
socket.connect(ep);
socket.write_some( buffer("GET /index.html\r\n") );
std::cout << "bytes available: "<<  socket.available() << std::endl;
char buff[512];
size_t read = socket.read_some( buffer(buff) );

// 同步读写udp_socket
ip::udp::socket socket( io_service );
socket.open( ip::udp::v4() );
ip::udp::endpoint receiver_ep( ip::address::from_string("127.0.0.1"), 80);
socket.send_to( buffer("test\n"), recevier_ep );
char buff[512];
ip::udp::ennpoint sender_ep; // 存储发送者端点
socket.receive_from( buffer(buff), sender_ep );

// 异步读写udp_socket
ip::udp::socket socket( io_service );
char buff[512]; // 全局变量

void on_read( cosnt std::error_code& ec, std::size_t read_bytes )
{
    // 更好的方式将lambda做回调handler，捕获局部环境变量
    socket.async_receive_from( buffer(buff), sender_ep, on_read );
}

ip::udp::endpoint ep( ip::address::from_string("127.0.0.1"), 80 );
socket.open( ep.protocol() );
socket.set_option( ip::udp::socket::reuse_address(true) );
socket.bind( ep );
socket.async_receive_from( buffer(buff, 512), sender_ep, on_read );
io_service.run();

// 异步操作读写缓冲区时，缓冲区对象的有效性问题
void func() { // 栈上分配 buff, boom !!!
    char buff[512];
    socket.async_receive( buffer(buff), on_read );
} // buff析构，异步操作可能未执行或未完成


void func() {  // 堆上分配 buff，显式释放堆内存
    char* buff = new char[512];
    socket.async_receive( buffer(buff, 512), std::bind(on_read, buff, _1, _2) );
}
void on_read(char* ptr, std::error_code& err, std::size_t read_bytes)
{
    delete [] ptr;
}


void func() { // 使用共享指针管理堆内存的分配和释放
    socket.async_receive( shared_buffer.self(), std::bind(ont_read, buff, -1, -2) );
}
void on_read( shared_buffer, std::error_code& ec, std::size read_bytes);
struct shared_buffer {
    boost::shared_array<char> buff;
    shared_buffer(size_t size): buff(new char[size]), size(size) { }
    mutable_buffers_1  self() { return buffer(buff.get(), size)); }
};
```

  - socket选项
    - get_option( option )，option选项对象，值-结果方式
    - set_option( option )
    - reuse_address，重用端点
    - keep_alive
    - linger
    - receive_buffer_size
    - receive_low_watermark
    - send_buffer_size，设置缓冲区大小
    - send_low_watermark
    - v6_only
    - broadcast，true为允许广播消息
    - debug
    - do_not_route，true为禁止路由只使用本地接口
```cpp
ip::tcp::socket::reuse_address reuse( true );
socket.set_option( reuse );

ip::tcp::socket::receive_buffer_size rbs;
socket.get_option( rbs );
std::cout << rbs.value << std::endl;

ip::tcp::socket::send_buffer_size sbs( 8192 );
socket.set_option( sbs ); // 设置发送缓冲区大小
```

  - 获取socket信息
    - local_endpoint()
    - remote_endpoint()
    - non_blocking(), true表示非阻塞
    - at_mark()，true为OOB数据
```cpp
```


buffer io对象:
- buffer方法，将一个连续的存储序列(缓冲区)包含在一个类中，以支持异步操作遍历这个缓冲区。(像迭代器与算法的关系)
- 异步操作对缓冲区类型的Requirement：ConstBufferSequence、MutableBufferSequence
- 满足Requirement的类型
  - char数组、pod类型的std::array/std::vector、std::string
- 在异步操作执行前到执行完成，要求缓冲区必须是有效的，否则将出错
  - 全局分配缓冲区，可能存在多线程的数据竞争
  - 栈上分配缓冲区，缓冲区可能在函数结束即销毁，从而失效(栈内存隐式管理)
  - 堆上分配缓冲区，可能存在多线程数据竞争，堆上内存显式管理
```cpp
struct pod_sample { int i; long l; char c; };

char buff[512]; // fixed char
std::string buff; // dynamic char
std::vector<pod_sample> buff; // dynamic
std::array<pod_sample> buff; // fixed

socket.async_read( buffer(buff), on_read );
```

stream_buffer 类:
- 继承自std::streambuf
- 用于支持对stream流的读写(全局I/O函数)
- 条件读写流的方法
  - read_until( stream, stream_buffer, delim )，读取直到遇到分隔符
  - read_until( stream, stream_buffer, completion )
    - std::pari<iterator,bool> completion( iterator begin, iterator end)
    - completion返回pair, 第一个参数表示最后被访问的字符，第二个参数为true则告诉read操作可以结束
  - async_read_until( stream, stream_buffer, delim, handler )
  - async_read_until( stream, stream_buffer, completion, handler )
```cpp
streambuf buff;
async_read_until( socket, buff, ' ', on_read ); // 遇到空格结束
async_read_until( socket, buff, match_punct, on_read ); // 遇到标点结束

typedef  buffers_iterator<streambuf::const_buffers_type> iterator;
std::pair<iterator bool>  match_punct(iterator begin, iterator end)
{
    while ( begin != end ) {
        if ( std::ispunct(*begin) )
            return std::make_pair(begin, true);
    }
    return std::make_pair(end, false);
}
```

stream的偏移读写：
- 从stream的指定偏移位置开始读写(随机存取), socket不支持随机访问，更像是端读写
- read_at( stream, offset, buffer, [completion] )
  - completion若返回非0值，表示下一次调用最大的读取字节数(预期字节数)
- write_at( stream, offset, buffer, [completion] ), buffer可以是wrapper封装或streambuf
- async_read_at( stream, offset, buffer, [completion], handler )
- async_write_at( stream, offset, buffer, [completion], handler )
```cpp
windows::random_access_handle h( io_service, file );
streambuf buf;
read_at( h, 256, buf, transfer_exactly(128) ); // 从文件偏移为256的位置读取128个字节
std::istream in( &buf );
std::string line;
std::getline( in, line );
```


## 异步编程

- 异步调用
  - 调用者不等待调用执行完成，调用的结果将在未来的某个时间点返回给调用者
  - 无法确定地知道下一个完成的调用是哪个异步调用，调用的执行次序动态、非线性
  - 可能的调用结果返回方式
    - 回调返回
    - 信号返回/IO就绪返回
  - 异步回调handler不调用同步的调用，可能出现回调handler阻塞调用者
- 同步调用
  - 调用者等待调用执行，直到调用结果返回
  - 程序员可以清楚地确定同步调用的执行次序、线性的


```cpp
// 异步方式

using namespace asio;

struct connection
{
    tcp::socket sock;
    asio::streambuf buff;
};

// callback function
void on_read( connection& conn, const std::error_code& err, std::size_t read_bytes )
{
    std::istream in(&conn.buff);
    std::string msg;
    std::getline(in, msg);
    if (msg == "request_login")
        conn.sock.async_write("request_ok\n"), on_write);

    async_read_until(conn.sock, conn.buff, '\n', std::bind(on_read, conn, _1, _2)); //为下一次异步read做准备
}

// multi-thread call completion event handler (callback)
void worker_thread()
{
    service.run(); 
}

void handle_connection()
{
    sync_read_until(conn.sock, conn.buff, '\n', std::bind(on_read, conn, _1, _2));

    for (int i = 0; i < 4; ++i) {
        std::thread(worker_thread);
    }
}

// async_accept
class talk_to_client: public std::enabled_shared_from_this<talk_to_client>
{
    tcp::socket sock;
    char read_buffer[max_msg];
    char write_buffer[max_msg];
    bool started;
public:
    void start() { started = true; do_read(); }
    void stop() { 
        if (!started) return;
        started = false;
        sock.close();
    }
    tcp::socket& socket() { return sock; }
};

tcp::acceptor acceptor(service, tcp::endpoint(tcp::v4(), 8080));

void handle_accept(std::shared_ptr<connection> conn, std::error_code& err)
{
   conn->start();

    // 下一次accpet准备
   std::shared_ptr<connection> new_conn(new connection);
   acceptor.async_accept(new_conn->socket(), std::bind(handle_accept, new_conn, _1)); 
}

int main() {
    std::shared_ptr<connection> conn(new connection); // conn被多个函数使用
    acceptor.async_accept( conn->socket(), std::bind(handle_accept, conn, _1) ); // 不需要知道某一时刻的所有连接
    service.run();
}

// 有在某一时刻获取所有连接的需要
std::vector<std::shared_ptr<connection>> clients;
void acceptor_thread()
{
    tcp::acceptor acceptor(service, tcp::endpoint(tcp::v4(), 8080));
    while (true) {
        std::shared_ptr<connnction> new_conn(new connnction);
        acceptor.accept( new_conn->socket() );
        // lock(), 多线程访问clients可能需要mutex
        clients.push_back(new_conn);
    }
}
void connection_thread()
{
    while (true) {
        std::this_thread::sleep(millisec(1));
        // lock()
        for (auto it = clients.begin(); it != clients.end(); ++i) {
            (*it)->answer_to_client();
        }
        // 移除所有超时的连接(长时间不活动连接)
        clients.erase(std::remove_if(
                            clients.begin(), clients.end(), 
                            std::bind(&talk_to_client::time_out, _1),
                       clients.end());

    }
}



// 同步方式
std::size_t read_complete(char* buff, const std::error_code& err, std::size_t bytes) {
    if (err) return 0;
    bool found = std::find(buff, buf + bytes, '\n') < (buf + bytes); //缓冲区之内
    return found  ? 0 : 1; // 0表示告诉read，操作完成
}

void sync_echo(std::string msg)
{
    msg += "\n";
    
    tcp::socket sock(service);
    sock.connect(ep);
    sock.write_some( buffer(msg) ); // 只要部分可写就操作成功返回，无要求阻塞等待全部写完
    char buff[1024];
    int bytes = read( sock, buffer(buff), std::bind(read_complete, buff, _1, _2) ); // 全部读完才操作成功返回，否则阻塞等待可读字节
    std::string read_msg(buff, bytes - 1);
    std::cout << (read_msg == msg ? "ok" : "fail");
}

void sync_echo_server()
{
    tcp::acceptor acceptor(service, tcp::endpoint(tcp::v4(), 8080)); //监听8080端口
    char buff[1024];
    while (true) {
        socket sock(service);
        acceptor.accept(sock);
        int bytes = read(sock, buffer(buff), std::bind(read_complete, buff, _1, _2));
        std::string msg(buff, bytes);
        sock.write_some(buffer(msg));
        sock.close(); // 短连接
    }
}
```


## echo示例

- 客户端通常比服务端简单，服务端需要处理多个客户端请求
- 同步方式
  - tcp echo
  - udp echo
- 异步方式
  - tcp echo
  - udp echo
- 协程方式(单线程异步)


### tcp同步服务器

```cpp
#include <iostream>
#include <memory>
#include <thread>
#include <utility>
#include "asio.hpp"

using asio::ip::tcp;

const int max_length = 1024;

void session(tcp::socket sock) {
    try
    {
        for(;;) {
            char data[ max_length ];
            
            asio::error_code ec;
            size_t length = sock.read_some(asio::buffer(data), ec);
            if (ec == asio::error::eof)
                break;
            else if (ec)
                throw asio::system_error(ec);
            
            asio::write(sock, asio::buffer(data, length));
        }
    }
    catch(std::exception &e)
    {
        std::cerr << e.what() << '\n';
    }
}

void server(asio::io_context& io_context, unsigned short port) {
    tcp::acceptor acc(io_context, tcp::endpoint(tcp::v4(), port));
    for (;;) {
        std::thread(session, acc.accept()).detach(); // 一线程一连接
    }
}

int main(int argc, char* argv[]) {
    try
    {
        asio::io_context io_context;
        server(io_context, 8888);
    }
    catch(const std::exception& e)
    {
        std::cerr << e.what() << '\n';
    }   
}
```

### tcp异步服务器

服务器程序逻辑:
- Session类，负责连接socket
- Server类， 负责监听socket，接受新连接并将创建Session对象

```cpp
// 异步操作 + 回调函数方式
#include <iostream>
#include <memory>
#include <utility>
#include "asio.hpp"

using asio::ip::tcp;

class Session : public std::enable_shared_from_this<Session>
{
    tcp::socket socket_;
    enum { max_length = 1024 };
    char data_[ max_length ];
public:
    Session(tcp::socket socket): socket_( std::move(socket) ) { }

    void start() { do_read(); }

private:
    void do_read() {
        auto self(shared_from_this());
        socket_.async_read_some(asio::buffer(data_, max_length), 
            [this, self]( std::error_code ec, std::size_t length )
            {
                if(!ec)  do_write(length);
            });
    }

    void do_write(std::size_t length) {
        auto self(shared_from_this());
        asio::async_write(socket_, asio::buffer(data_, length),
            [this, self]( std::error_code ec, std::size_t length )
            {
                if (!ec)  do_read();
            });
    }
};


class Server
{
    tcp::acceptor acceptor_;
public:
    Server(asio::io_context& io_context, short port)
        : acceptor_(io_context, tcp::endpoint(tcp::v4(), port))
        {
            do_accept();
        }

private:
    void do_accept() {
        acceptor_.async_accept(
            [this](std::error_code ec, tcp::socket socket)
            {
                if (!ec) {
                    std::make_shared<Session>(std::move(socket))->start(); // 首次创建Session对象
                }
                do_accept();
            });
    }
};

int main(int argc, char* argv[]) {
    try
    {
        asio::io_context io_context;
        Server server(io_context, 8888);
        io_context.run();   
    } 
    catch (std::exception& e) 
    {
        std::cerr << e.what() << "\n";
    }
    return 0;
}
````



### tcp同步客户端

```cpp
#include <iostream>
#include <string>
#include "asio.hpp"

using asio::ip::tcp;

enum { max_length = 1024 };

int main(int argc, char* argv[]) {
    try
    {
        asio::io_context io_context;
        
        tcp::socket socket(io_context);
        tcp::resolver resolver(io_context);
        asio::connect(socket, resolver.resolve("127.0.0.1", "8888"));
        
        std::cout << "Enter message: ";
        char request[ max_length ];
        std::cin.getline(request, max_length);
        size_t request_length = std::strlen(request);
        asio::write(socket, asio::buffer(request, request_length));

        char reply[ max_length ];
        size_t reply_length = asio::read(socket, asio::buffer(reply, request_length));
        std::cout << "Reply is: ";
        std::cout.write(reply, reply_length);
        std::cout << "\n";
    }
    catch(const std::exception& e)
    {
        std::cerr << e.what() << '\n';
    }
    return 0;
}
```

### udp同步服务器

```cpp
#include <iostream>
#include "asio.hpp"

using asio::ip::udp;

enum { max_length = 1024 };

void server(asio::io_context& io_context, unsigned short port) {
    udp::socket sock(io_context, udp::endpoint(udp::v4(), port));
    for (;;)
    {
        char data[max_length];
        udp::endpoint sender;
        size_t length = sock.receive_from(asio::buffer(data, max_length), sender);
        sock.send_to(asio::buffer(data, length), sender);
    }
}

int main(int argc, char* argv[]) {
    try
    {
        asio::io_context io_context;
        server(io_context, 8888);
    }
    catch(const std::exception& e)
    {
        std::cerr << e.what() << '\n';
    }
    return 0;
}
```

### udp同步客户端

```cpp
#include <iostream>
#include <string>
#include "asio.hpp"

using asio::ip::udp;

enum { max_length = 1024 };

int main(int argc, char* argv[]) {
    try
    {
        asio::io_context io_context;
        
        udp::socket socket(io_context, udp::endpoint(udp::v4(), 0));
        udp::resolver resolver(io_context);
        udp::resolver::results_type endpoints = resolver.resolve(udp::v4(), "localhost", 
        "8888");

        std::cout << "Enter message: ";
        char request[ max_length ];
        std::cin.getline(request, max_length);
        size_t request_length = std::strlen(request);
        socket.send_to(asio::buffer(request, request_length), *endpoints.begin());

        char reply[ max_length ];
        udp::endpoint sender;
        size_t reply_length = socket.receive_from(asio::buffer(reply, request_length), sender);
        std::cout << "Reply is: ";
        std::cout.write(reply, reply_length);
        std::cout << "\n";
    }
    catch(const std::exception& e)
    {
        std::cerr << e.what() << '\n';
    }
    return 0;
}
```

## udp异步服务器

```cpp
#incldue <iostream>
#include "asio.hpp"

using asio::ip::udp;

class server
{
    udp::socket sock;
    udp::endpoint sender_ep;
    enum { max_length = 1024 };
    char buff[max_length];

public:
    server(asio::io_service& service, unsigned short port)
        :sock(service, udp::endpoint(udp::v4(), port))
    {
        do_receive();
    }   

    void do_receive()
    {
        sock.async_receive_from(
            asio::buffer(buff, max_length),
            sender_ep,
            [this](std::error_code ec, std::size_t bytes_recv)
            {
                if (!ec && bytes_recv > 0) {
                    do_send(bytes_recv);
                } else {
                    do_ receive(); // 重新接收
                }
            });
    }

    void do_send(std::size_t length)
    {
        sock.async_send_to(
            asio::buffer(buff, length),
            sender_ep,
            [this](std::error_code /*ec*/, std::size_t /*bytes_sent*/)
            {
                do_receive();
            });
        )
    }
};

int main(int argc, char* argv[4])
{
    try
    {
        if (argc != 2) {
            std::cerr << "Usage: <port>\n";
            return 1;
        }

        asio::io_service service;

        server s(service, std::atoi(argv[1]));

        service.run();
    }
    catch(std::exception& e)
    {
        std::cerr << "Exception: " << e.what() << "\n";
    }
    
    return 0;
}

```


## 应用客户端/服务器示例

- 客户端使用一个无密码的用户名登录到服务端
- 连接由客户端建立，服务端被动回应请求
- 5秒没有活动的连接(客户端无活动)，服务端将自动断开其连接
- 所有的请求和回复都以'\n'结尾, 消息的结尾以'\n'标识
- 客户端的行为
  - ping服务端
  - 获取服务端关于该客户端的已建立连接的列表
  - 每个客户端可以登录6个用户，每个用户以名字标识
  - 每个客户端随机7秒内ping服务端, 服务端在5秒内检测到没有活动的连接将关闭连接

```cpp
// 异步客户端

#define men_fn(x)   std::bind(&talk_to_svr::x, shared_from_this())
#define mem_fn(x,y) std::bind(&talk_to_svr::x, shared_from_this(), y)
#define men_fn(x,y,z) std::bind(&talk_to_svr::x, shared_from_this(), y, z)

class talk_to_svr: public std::enable_shared_from_this<talk_to_svr>
{
    ip::tcp::socket sock;
    char read_buffer[max_msg];
    char write_buffer[max_msg];
    dealline_timer timer;

    bool started;

    std::string use_name;

public:

    talk_to_svr(const std::string& user_name)
                : sock(service),
                  started(true),
                  user_name(service) { }
    
    void start(ip::tcp::endpoint ep) {
        sock.async_connect(ep, men_fn(on_connect, _1) );
    }

    void stop() {
        if (!started)  return;
        started = false;
        sock.close();
    }

    bool started() { return started; }

private:

    void on_connect(const std::error_code& err) {
        if (err)  stop();
        
        do_write("login" + user_name + "\n");
    }

    // write
    void do_write(cosnt std::string& msg) { // send request to server
        if (!startd)  return;

        std::copy(msg.begin(), msg.end(), write_buffer);
        sock.async_write_some(buffer(write_buffer, msg.size()),
                              men_fn(on_write, _1, _2));
    }

    void on_write(const std::error_code& err, std::size_t bytes) {
        do_read();
    }

    // read
    void do_read() { // read response from server
        async_read(sock, 
                   buffer(read_buffer), 
                   men_fn(read_complete, _1, _2),
                   men_fn(on_read, _1, _2));
    }

    std::size_t read_complete(const std::error_code& err, std::size_t bytes) {
        if (err)  return;
        
        bool found = std::find(read_buffer, read_buffer+bytes, '\n') < (read_buffer + bytes);
        return found ? 0 : 1;
    }

    void on_read(const std::error_code& err, std::size_t bytes) {
        if (err)  stop();
        if (!started)  return;

        std::string msg(read_buffer, bytes); // server's response
        if (msg.find("login") == 0)       on_login();  /// ??
        else if (msg.find("ping") == 0)   on_ping(msg);  // ??
        else if (msg.find("clients")== 0) on_clients(msg); //??
    }

    // 每一个方法要么向服务端发出一个ping, 要么向服务端请求客户端的连接列表
    // 当服务器回复客户端的连接连接请求时，客户端read完成后，发出一个ping
    // 每次read操作完成都会向服务端发送一个ping

    // login
    void on_login() { do_write("ask_clients\n"); }
    
    // ping
    void on_ping(const std::string& msg) {
        std::istringstream in(msg);
        std::string answer; 
        in >> answer;
        if (answer == "client_list_changed") // 成功登录
            do_write("ask_clients\n");
        else
            postpone_ping(); // ??
    }

    void postpone_ping() { // 延迟ping
        timer.expires_from_now(std::chrono::millisec(rand() % 7000));
        timer.async_wait( men_fn(do_ping) );
    }

    void do_ping() { do_write("ping\n"); }

    // clients
    void on_clients(const std::string& msg) { // 获取本客户端的已连接列表
        std::string clients = msg.substr(8);
        std::cout << user_name << "new client list: " << clients;
        postpone_ping();
    }
};


// 异步服务端

class talk_to_client: public std::enable_shared_from_this<talk_to_client>
{
    tcp::socket sock;
    char read_buffer[max_msg];
    char write_buffer[msx_msg];
    bool started

    std::string user_name;

    deadline_timer timer;
    std::chrono::timepoint last_ping;

public:

    talk_to_client() {...}

    void start() {
        started = true;
        clients.push_back(shared_from_this());
        last_ping = std::chrono::system_clock::now();
        do_read();
    }

    void stop() {
        started = false;
        sock.close();
        clients.erase( std::find(clients.begin(), clients.end(), shared_from_this() );
    }

    tcp::socekt& socket() { return sock; }

private:

    void on_login(const std::string& msg) {
        std::istringstream in(msg);
        in >> user_name;
        do_write("login ok\n");
    }

    void on_ping() {
        do_write("ping ok\n");
    }

    void on_clients() {
        std::string msg;
        for (auto client : clients) {
            msg += (*client)->user_name + " ";
        }
        do_write("clients " + msg + "\n");
    }

    void post_check_ping() {
        timer.expires_from_now(std::chrono::millisec(5000));
        timer.async_wait( std::bind(&talk_to_client::on_check_ping, shared_from_this());
    }

    void on_check_ping() {
        std::chrono::timepoint now = std::chrono::system_clock::now();
        if ((now - last_ping) > 5000) stop();
    }
};


// 监听
tcp::acceptor acceptor(service, tcp::endpoint(tcp::v4(), 8888));

void handle_accept(std::shared_ptr<talk_to_client> client, std::error_code &err)
{
    client->start();

    std::shared_ptr<talk_to_client> new_client(new talk_to_client);
    acceptor.async_accept(new_client->socket(), 
                          std::bind(handle_accept, new_client, _1));
}

int main() {
    std::shared_ptr<talk_to_client> first_client(new talk_to_client);

    acceptor.async_accept(first_client->socket(), 
                          std::bind(handle_accept, first_client, _1));
    
    service.run();
}

// 多线程调用handle_accept接受新连接
void work_thread() {
    service.run();
}

in  main() {
    std::shared_ptr<talk_to_client> client(new talk_to_client);
    
    acceptor.async_accept(client->socket(), std::bind(handle_accept, _1));

    for (int i = 0; i < 4; ++i) {
        thread_group.push_back( std::move(std::thread(work_thread)) ) ;
    }

    thread_group.join_all();
}

// 可能需要互斥锁来保证同一连接的所有请求被同一线程
// async_read和async_wait可能同时完成，其完成handler可能被两个线程分别调用
void do_read() {
    async_read(sock, buffer(read_buffer), 
               mem_fn(read_complete, _1, _2),
               mem_fn(on_read, _1, _2));
    post_check_ping();
}

void post_check_ping() {
    timer.expires_from_now(std::chrono::millisec(5000));
    timer.async_wait(mem_fn(on_check_ping));
}
```

    
- 代理服务器
  - 客户端 -- [ 代理服务端-代理客户端 ]-- 服务端
  - 代理接受客户端的请求(可能会对请求进行修改)，然后将请求发送到服务端
  - 代理从服务端取回结果(可能对结果进行修改)，然后将结果发送到客户端
  - 对于代理的每个连接，使用一个socket表示客户端，一个socket表示服务端
  - 如果采用同步方式，当代理服务器向一端的读或写发生阻塞时，无法响应另一端
  - 改进性能
    - 当代理向客户端写响应的时候，同时进行读取服务端的动作，即这两个动作异步
    - 使用多个buffer

```cpp
class proxy: public std::enable_shared_from_this<proxy>
{
    tcp::socket  client, server;
    char client_buffer[max_msg];
    char server_buffer[max_mas];
    bool started;

public:

    proxy(tcp::endpoint client_ep, tcp::endpoint server_ep) { ... }

    void start() {
        // 建立两个连接

    }

    void stop() {
        // 关闭两个连接
    }

private:

    void on_connect(std::error_code& err) {
        if (err)  stop();
        
        if (/*都建立*/) on_start();
    }

    void on_start() {
        do_read(client, client_buffer);
        do_read(server, server_buffer);
    }

    // client --read--> proxy --write--> server
    // client <--write-- proxy <--read-- server

    void do_read(tcp::socket& sock, char* buff) {
        async_read(sock, buffer(buff, max_msg), 
                   men_fn(read_complete, std::ref(sock), _1, _2), // ??
                   mem_fn(on_read, std::ref(sock), _1, _2));
        
    }

    size_t read_complete(tcp:;socket& sock, std::error_code& err, std::size_t bytes) {
        ...
    }

    void on_read(tcp::socket& sock, std::error_code& err, std::size_t bytes) {
        char* buff = (sock == client ? client_buffer : server_buffer);
        do_write( (sock == client) ? server : client, buff), bytes );
    }

    void do_write(tcp::socket& sock, std::error_code& err, std:;size_t bytes) {
        sock.async_write_some(buffer(buff, bytes), 
                              men_fn(on_write, std::ref(sock), _1, _2));
    }

    void on_write(tcp::socket& sock, std::error_code& err, std::size_t bytes) {
        if (sock == client) // 向客户端async_write_some完成，继续读服务器响应
            do_read(server, server_buffer);
        else
            do_read(client, client_buffer);
    }


};

int main() {
    tcp::endpoint client_ep(ip::address::from_string("127.0.0.1"), 8888);
    tcp::endpoint server_ep(ip::address::from_string("127.0.0.1"), 8889);

    proxy proxy_server(client_ep, server_ep);

    proxy_server.start();

    // proxy::start(client_ep, server_ep);

    service.run();
}
```


- 服务端向客户端推送通知
  - 服务端推送通知到客户端，客户端读取通知并处理，可能会向服务端写处理结果的响应或发送确认接收响应
  - 如果同时存在客户端向服务端拉取结果的请求，那么当客户端发送拉取的请求时，此时收到服务端的推送通知，可能将推送通知当成拉取请求的响应结果处理.
  - 一种客户端主动轮询的方法，让客户端每5秒钟ping一次服务端，如果没有通知要推送，服务端返回"ping ok"，有通知则返回ping[event_name]

```cpp
// 客户端
void loop() {
    while (started) {
        read_notification();
        process_notification();
        write_answer(); // answer == response 
    }
}

void read_notification() {
    already_read = false;
    read(sock, buffer(buff), std::bind(&talk_to_svr::read_complete, this, _1, _2));
}

void process_notification() {
    ...
}


// 服务端
void notify_clients() { // 通知对某个消息感兴趣的客户端
    for (auto client : clients) {
        (*client)->sock.write_some(notify_msg);
    }
}


// 异步的推送服务端
void start() { on_notify(); }

void on_notify() {
    std::ostringstream msg;
    msg << "notification\n";
    for (auto client : clients) {
        (*client)->do_write(msg);
    }
}

void do_write() {
    std::copy(msg.begin(), msg.end(), write_buffer);
    sock.async_write_some(buffer(write_buffer, msg.size()), 
                          mem_fn(on_write, _1, _2));
}

void on_write(std::error_code& err, std::size_t bytes) {
    do_read();
}
```


## 协程

- asio 无栈协程
  - 采用对象的方法，用一个整型变量记住协程函数暂停点行号
  - 异步操作的完成handler设置为协程函数，当调用handler时，即再次进入执行协程函数,从暂停点后的第一行开始执行，直到遇到yield或协程函数末尾才返回
  - class coroutine
    - 被包含协程函数的类继承
  - reenter(entry)
    - 用于定义协程，entry指向这个协程对象
  - yield expr
    - 用于暂停协程，执行表达式后, 暂停协程并返回
  - 通过宏的方式 + switch实现

```cpp
// 回调resume协程
#include "asio/yield.hpp"
#include "asio/coroutine.hpp"

class talk_to_svr: public std::enable_shared_from_this<talk_to_svr>, public asio::coroutine
{
    void step(std::error_code& err, size_t bytes = 0) {
        reenter(this)
        {
            for(;;) // 如果没有for(;;)，则协程函数在执行完async_read_until的handler后结束
            {
                yield async_write(sock, write_buffer, mem_fn(step, _1, _2));
                yield async_read_until(sock, read_buffer, "\n", mem_fn(step, _1, _2));
            }
        }
    }
};

struct session: asio::coroutine
{
    std::shared_ptr<tcp::socket> sock;
    std::shared_ptr<std::string> buff;
    void operator()(asio::error_code err = asio::error_code(), std::size_t = 0)
    {
        if (!err) reenter (this)
        {
            for (;;)
            {
                yield sock->async_read_some(asio::buffer(*buff), *this);
                yield asio::async_write(*sock, asio::buffer(*buff,n), *this); // 函数对象用*this做完成handler
            }
        }
    }
};

// 显式resume协程
struct A: public asio::coroutine
{
    void print()
    {
        reenter(this)
        {
            yield std::cout << "1\n";
            yield std::cout << "2\n";
        }
    }
};

int main() {
    A obj;
    obj.print();
    obj.print();
}
```


- Coroutine-ts

- awaitable<T> 类模板
  - 用来表示future的概念，即协程函数的返回值类型
- use_awaitable 标识符
  - 协程函数的标识符，用来被异步操作回调，再次执行协程函数
- co_spwan 函数
  - 用来产生一个协程，像产生线程的方式，可以让子协程从主协程detach
  - 第一参数为执行线程
  - 第二参数为返回值类型为awaitable<T>的函数对象
  - 第三参数为completion handler，原型为 void(std::exception_ptr, T)，当传递给异步操作use_awaitable时，生成返回值类型为awaitable<T>的异步操作函数，以便于允许对异步操作使用co_await运算符，如果异步操作，则use_awaitable handler将抛出异常
    - void handler(asio::error_code ec, result_type result)
    - void haneler(asio::error_code ec), 返回类型为void
  - 第四参数detached表示，显式忽略异步操作的结果
- co_await 运算符
  - 用来暂停协程函数，在对expr求值后，暂停协程函数，记录暂停点并返回, 当再次执行协程函数时，从暂停点后的第一行开始执行，直到在此遇到co_await或协程函数的结尾
  - 运算符对象须为awaitable<T>类型
  - 运算结果为result_type类型或void类型
- this_coro 类
  - 用来获取协程所属于的执行线程(exector)， this_coro::executor


```cpp
// 异步操作 + c++20协程 方式
#include <asio/co_spawn.hpp>
#include <asio/detached.hpp>

#include <asio/io_context.hpp>
#include <asio/ip/tcp.hpp>
#include <asio/signal_set.hpp>
#include <asio/write.hpp>
#include <cstdio>

using asio::ip::tcp;

using asio::awaitable;
using asio::co_spawn;
using asio::detached;
using asio::use_awaitable;
namespace this_coro = asio::this_coro;

// 连接对应的协程服务函数
awaitable<void> echo(tcp::socket socket)
{
  try
  {
    char data[1024];
    for(;;)
    {
      std::size_t n = co_await socket.async_read_some(asio::buffer(data), use_awaitable);
      co_await async_write(socket, asio::buffer(data, n), use_awaitable);
    }
  }
  catch (std::exception& e)
  {
    //std::cerr << e.what() << "\n";
    printf("Exception: %s\n", e.what());
  }
}


awaitable<void> listener()
{
  auto executor = co_await this_coro::executor;
  tcp::acceptor acceptor(executor, {tcp::v4(), 8888});

  for(;;)
  { // a coroutine per connection
    tcp::socket socket = co_await acceptor.async_accept(use_awaitable);
    co_spawn(executor, 
            [socket = std::move(socket)]() mutable
            {
              return echo(std::move(socket));
            },
            detached);
  }
}

int main() {
  try
  {
    asio::io_context ioc(1);

    asio::signal_set signals(ioc, SIGINT, SIGTERM);
    signals.async_wait([&](auto, auto){ ioc.stop(); });

    co_spawn(ioc, listener, detached);

    ioc.run();
  }
  catch(const std::exception& e)
  {
    //std::cerr << e.what() << '\n';
    printf("Exception: %s\n", e.what());
  }
  
}
```


## 流缓冲streambuf

- streambuf会动态扩容，可以读入任意长度的消息
- 可以将streambuf与流对象关联，流对象负责读写流缓冲区
- asio::streambuf的额外方法
 - streambuf([max_size], [allocator])
 - size() 和 max_size()
 - consume(bytes), 从输入流消耗数据, 即移动读指针 ??? 什么情况下需要用 ??
- 可以使用正则表达式的方法
  - read_util(socket, buff, regex)
  - async_read_until(socket, buff, regex, handler)

```cpp
asio::streambuf buff;
std::string str;

read_util(sock, buf, '\n');
std::istream in(&buff);
in >> str; // 输入运算符自动将流缓冲的内容从字符转为字符串

std::ostream out(&buff);
out << str << std::endl;
write(sock, buff);

// 使用正则表达式匹配元音字母
read_until(sock, buff, std::regex("^[aeiou]+"));
```


## SSL

- 依赖openssl库，编译时需指定openssl库和crypto库供程序链接 -lssl -lcrypto

```cpp
#include <asio/ssl.hpp>

using asio::ip::tcp;

int main() {
    typedef asio::ssl::stream<tcp::socket> ssl_socket;
    asio::ssl::context ctx(ssl::context::sslv23);
    ctx.set_default_verify_paths();

    asio::io_service service;
    ssl_socket sock(service, ctx);

    tcp::resolver resolver(service);
    std::string host = "www.bing.com";
    tcp::resolver::query query(host, "https");
    asio::connect(sock.lowest_layer(), resolver.resolve(query));

    sock.set_verify_mode(ssl::verify_none);
    sock.set_verify_callback(ssl::rfc2818_verification(host));
    sock.handshake(ssl_socket::client);

    std::string request = "GET /index.html HTTP/1.1\r\n";
    asio::write(sock, asio::buffer(request.c_str(), request.size()));

    char buff[512];
    std::error_code ec;
    while (!ec) {
        int bytes = asio::read(sock, buffer(buff), ec);
        std::cout << std::string(buff, bytes);
    }
}
```