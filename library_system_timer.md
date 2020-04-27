

## 定时事件

- 网路程序需要处理的三类事件：I/O事件、信号事件、定时器事件
- 通过将定时事件封装成定时器，并使用容器将所有定时器组织起来，进行统一管理
- 定时，指在一段时间之后出发某段代码/动作的机制
- linux提供三种定时的方法
  - SIGALAM信号
  - socket的SO_RCVYIMEO和SO_SNDTIMEO选项
  - I/O复用系统调用的超时参数

## SIGALRM信号

- 模拟时间中断，软中断
- 由alarm和setitimer函数设置的实时闹钟一旦超时，将出发SIGALRM信号，可以利用信号处理函数来处理定时任务
- 如果要处理多个定时任务，就需要不断触发SIGALRM信号，让SIGALRM信号按照固定频率生成，定时周期T反映了定时的精度

```cpp
// 基于升序列表的定时器
class Timer {
public:
    Timer: prev(NULL), next(NULL) { }

    time_t expire; // 超时时间
    void (*cb_func)( client_data* ); // 任务回调函数
    
    client_data* user_data;
    Timer* prev;
    Timer* next;
};

class Sort_timer_lst {
public:
    Sort_timer_lst(): head(NULL), tail(NULL) { }
    ~Sort_timer_lst() ; // 删除链表中的所有定时器
    void add_timer( Timer* timer ) {
        if (!head) // 空表
        if (timer->expire < head->expire) // 表头插入
        add_timer( timer, head ); // 其他情况插入 
    } 
    void adjust_timer( Timer* timer ) {
        Timer* tmp = timer->next;
        if (!tmp || (timer->expire < tmp->expire)) // 被调整的定时器比下一个定时器小或表尾，不用调整
        if (timer == head) // 取出表头节点，重新插入
        else // 非表头节点，取出节点，并重新插入原来位置之后的部分链表中
    }
    void del_timer( Timer* timer ) {
        if ( (timer == head) && (timer == tail) ) // 链表唯一节点
        if (timer == head) // 链表至少有两个节点且删除头节点
        if (timer == tail) // 至少有两个节点切删除尾节点
    }
    void tick() {  // 处理链表上到期的任务
        while (tmp) {
            if (cur < tmp->expire) break; // 无定时器超时到期
            // 执行定时器任务后，将定时器从链表删除
            tmp->cb_func(tmp->user_data);
            head=tmp->next;
            if (head) head->prev = NULL;
            delete tmp;
            tmp = head;
        }
    }
private:
    void add_timer( Timer* timer, Timer* lst_head ); // 辅助函数，将定时器插入到lst_head之后的位置
private:
    Timer* head;
    Timer* tail;
};
```

```cpp
// 使用升序定时器链表关闭非活动连接

void sig_handler( int sig ) {
    ...
    send( pipefd[1], (char*)&msg, 1, 0 ); // 触发信号事件
    ...
}

void timer_handler() {
    timer_lst.tick(); // 心博函数
    alarm( TIMESLOT ); // 周期出发SIGALRM信号，TIMESHOT宏定义为5
}

// 回调函数，删除活动连接对应的注册事件，并关闭连接
void cb_func( client_data* user_data ) {
    epoll_ctl( epollfd, EPOLL_CTL_DEL, user_data->sockfd, 0 );
    close( user_data->sockfd );
}

int main() {
    ...
    // 设置信号处理函数
    addsig( SIGALRM );
    addsig( SIGTERM );
    
    client_data* users = new client_data[FD_LIMIT];
    bool timeout = false; // 是否有定时器超时标记
    alarm( TIMESLOT );
    ...

    for (int i = 0; i < number; ++i) {
        int sockfd = events[i].data.fd;
        if (sockfd == listenfd ) { // 新连接事件
            ...
            // 为新连接创建定时器、设置回调函数、绑定用户数据，然后添加到定时器链表
            Timer* timer = new Timer;
            timer->user_data = &users[connfd];
            timer->cb_func = cb_func;
            time_t cur = time(NULL); // 当前时间
            timer->expire = cur + 3 * TIMESLOT; // 设置超时时间
            users[connfd].timer = timer;
            timer_lst.add_timer(timer);
        }
        else if ( (sockfd == pipefd[0]) && (events[i].events & EPOLLIN) ) { // 信号事件
            char signals[1024];
            ret = recv( pipefd[0], signals, sizeof(signals), 0 );
            if (ret == -1 || ret == 0)  continue;
            else {
                for (int i = 0; i < ret; ++i)
                    switch( signals[i] ) {
                        case SIGALRM:
                        {
                            timeout = true; // 标记但不立即处理超时事件
                            break;
                        }
                        case SIGTERM:
                        {
                            stop_server = true;
                        }
                    }
            }
        }
        else if (events[i].events & EPOLLIN) { // 连接可读事件
            Timer* timer = users[connfd].timer;
            if (ret < 0) {
                if (errno != EAGIAN) { //发生错误，关闭连接并移除定时器
                    cb_func( &users[sockfd] );
                    if (timer)  timer_lst.del_timer(timer);
                }
            }
            else if (ret == 0) { // 对端关闭连接
                    cb_func( &users[sockfd] );
                    if (timer)  timer_lst.del_timer(timer);
            }
            else { // 调整连接对应的定时器，以延迟该连接被关闭的时间
                if (timer) {
                    time_t cur = time(NULL);
                    timer->expire = cur + 3 * TIMESLOT;
                    timer_lst.adjust_timer(timer);
                }
            }
        }
    }
    // 在所有的epoll返回就绪描述符遍历完后，才开始处理定时事件
    // I/O事件有更高的优先级，这种策略将导致定时任务不能按预期精确时间执行
    if (timeout) {
        timer_handler();
        timeout = false;
    }
```

## I/O复用调用的超时参数

- 统一处理超时信号、I/O事件、定时器事件
- 当I/O复用调用在超时时间未到期之前返回，需要更新定时参数以反映剩余的超时时间

```cpp
#define TIMEOUT 5000

int timeout = TIMEOUT;
time_t start = time(NULL);
time_t end = time(NULL);

while (true) {
    start = time(NULL);
    int number = epoll_wait(epollfd, events, MAX_EVENT_NUMBET, timeout);
    if (number == 0) { // 超时时间到，处理定时任务并重置定时时间
        timeout = TIMEOUT;
        continue;
    } else { // epoll返回，未超时则更新剩余超时时间
        end = time(NULL);
        timeout -= (end - start) * 1000;
        if (timeout <= 0)  timeout = TIMEOUT;// 若timeout刚好等于0，则epoll正常返回并且超时时间到
    }

    // handle connections
}
```

## socket选项

- 分别用来设置接收数据的超时时间和发送数据超时时间
- 适用于send、sendmsg、recv、recvmsg、accept和connect
- 当connect超时返回，errno为 EINPROGRESS(处理中)
- 其他调用超时返回，errno为 EWOULDBLOCK(期望阻塞)

```cpp
int timeout_connect( const char* ip, int port, int time ) {
    ...
    int sockfd = socket(PF_INET, SOCK_STREAM, 0);

    struct timeval timeout;
    timeout.tv_sec = time;
    timeout.tv_usec = 0;
    socklen_t len = sizeof(timeout);
    setsockopt(sockfd, SOL_SOCKET, SO_SNDTIMEO, &timeout, len);
    
    ret = connect( sockfd, (struct sockaddr*)&address, sizeof(address) );
    if (ret == -1) {
        if (errno == EINPROGRESS)
            printf("connecting timeout");
        printf("error occcur when connecting to server\n");
    }
    return sockfd;
}

int main() {
    ...
    int sockfd = timeout_connect(ip, port, 10);
    ...
}
```

## 时间轮

- 简单的时间轮只有一个轮子，每个轮子上有N个槽(slot)，每个槽对应一条定时器链表，每条链表上的定时器有相同特征.
- 有一个指针指向某个槽，指针以恒定的速度顺时针转动，每转一步就指向下一个槽，每次转动称为一次滴答(tick)，一个滴答的时间为槽间隔si(slot interval)，心博时间. 槽上的定时器链表之间的定时器相差N*si的整数倍定时时间
- ts = (cs + (timer/si)) % N, cs表示当前槽，ts表示目标槽
- 时间轮使用哈希表的思想，将定时器散列到不同的链表上，每条链表的定时器数目比升序定时器链表的数目要少，插入操作的时间复杂度基本不受定时器数目的影响
- 要提高定时精度，si要足够小；要提高执行效率，N要足够大(每条链表平均长度足够短)，散列表的入口(entry)越多，每条链表上的定时器数量越少.   
- 复杂的时间轮有多个轮子，不同的轮子表示不同的定时精度，相同的两个轮子，精度高的转一圈，精度低的仅往前移动一槽
- 添加定时器O(1)，删除O(1)，执行O(n)，链表非升序链表

```cpp
class tw_timer {
public:
    tw_timer(int rot, int ts): rotation(rot), time_slot(ts), prev(NULL), next(NULL) { }

    int rotation; // 定时器在转多少圈后超时
    int time_slot; // 槽
    void (*cb_func)(client_data*);
    client_data* user_data;
    tw_timer* next;
    tw_timer* prev;
};

class time_wheel {
    static const int N = 60;
    static const int SI = 1;  // 1 tick = 1s
    tw_timer* slots[N]; // 时间轮，槽上链表无序
    int cur_slot;
public:
    time_wheel(): cur_slot(0) {
        // 初始化slots指针数组
    }
    ~time_wheel() {
        // 清空每个槽的定时器链表
    }
    tw_timer* add_timer( int timeout ) { // 添加定时器
        int ticks = 0;
        if (timeout < SI )  ticks = 1;
        else  ticks = timeout / SI;
        int rotation = ticks / N;
        int ts = (cur_slot + (ticks % N)) % N;
        tw_timer* timer = new tw_timer(rotation, ts);

        if (!slot[ts]) { // ts槽中无定时器，空表
            slots[ts] = timer;
        } else { // 非空表
            timer->next = slots[ts];
            slots[ts]->prev = timer;
            slots[ts] = timer;
        }
        return timer;
    }
    void del_timer( tw_timer* timer) { // 删除定时器
        int ts = timer->time_slot;
        if (timer == slots[ts]) { // 表头节点

        } else { // 其他节点

        }
    }
    void tick() { // 更新槽，时间轮转动
        tw_timer* tmp = slots[cur_slot];
        while( tmp ) { // 遍历当前槽的定时器链表
            if (tmp->rotation > 0) { // 未到期
                -- tmp->rotation;
                tmp = tmp->next;
            } else { // 定时器到期，执行任务并删除定时器
                tmp->cb_func(tmp->user_data);
                if (tmp == slots[cur_slot]) {

                } else {

                }
            }
        }
        cur_slot = ++ cur_slot % N; // 时间轮转动
    }
}
```

## 时间堆

- 将所有的定时器中超时时间最小的一个定时器的超时值作为心博间隔
- 一旦心博函数被调用，必定有定时器到期，为超时时间最小的定时器
- 在tick函数中处理该定时器，并从剩余的找出超时时间最小的一个定时器，将其最小时间设为下一次的心博间隔
- 升序定时器链表和时间轮采用的是地固定的频率调用心博函数，时间堆采用变动的频率
- 最小堆，指父节点值小于或等于子节点值的完全二叉树，可以用数组表示堆，节省空间，更好支持堆的操作
- 对于数组的位置i的元素，其左孩子的位置为2i+1，右孩子为2i+2，其父节点的位置为 [(i - 1) / 2]
- 只要确保非叶子节点构成的子树具有堆序性质，整个树就具有堆序性质
- 添加定时器O(lgn)，删除O(1)，执行O(1)

```cpp
class heap_timer { // 定时器
    time_t expire;
    void (*cb_func)(client_data*);
    client_data* user_data;
public:
    heap_timer( int delay ) {
        expire = time(NULL) + delay;
    }
};

class time_heap { // 时间堆
    heap_timer** array;
    int capacity; // 容量
    int cur_size;
public:
    time_heap(int cap): capacity(cap), cur_size(0) { 
        array = new heap_timer*[cap];
        // 初始化指针数组
    }
    ~time_heap() {
        for (int i = 0; i < cur_size; ++i)
            delete array[i];
        delete [] array;
    }
    void add_timer( heap_timer* timer ) {
        if (cur_size >= capacity)  resize(); // 扩容一倍
        
        int hole = cur_size++; // hole表示新元素
        int parent = 0;
        for (; hole > 0; hole = parent) {
            parent = (hole - 1) / 2;
            if ( array[parent]->expire <= timer->expire ) break;
            array[hole] = array[parent];
        }    
        array[hole] = timer;
    }
    void del_timer( heap_timer* timer ) { // 删除定时器
        timer->cb_func = NULL;
    }
    heap_timer* top() const { // 取出堆顶定时器
        return array[0];
    }
    void pop_timer() { // 删除堆顶定时器
        if (array[0]) {
            delete array[0];
            array[0] = array[--cur_size];
            percolate_down(0); // 下虑操作，逐层确保最小堆性质
        }
    }
    void tick() {
        heap_timer* tmp = array[0];
        time_t cur = time(NULL);
        while (!empty()) {
            if (tmp->expire > cur) break;
            else {
                array[0]->cb_func(array[0]->user_data);
            }
            pop_timer();
            tmp = array[0];
        }
    }
    bool empty() const { return cur_size == 0; }
private:
    void percolate_down( int hole ) {
        heap_timer* tmp = arrar[hole];
        int child = 0;
        for (; (hole*2+1) <= (cur_size-1); hole = child) {
            child = hole * 2 + 1; //左孩子
            if ((child < cur_size-1) && (array[child+1]->expire < array[child]->expire)) { // 右孩子小于左孩子
                ++ child;
            }
            if (array[child]->expire < tmp->expire) { // 左孩子小于父节点
                array[hole] = array[child];
            }
            else
                break;
        }
        array[hole] = tmp;
    } 
    void resize();
};
```