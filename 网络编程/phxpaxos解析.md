### Src/algorithm目录





### Src/communication/tcp目录

#### eventbase.h

定义了Event抽象类，派生出子类：MessageEvent、Notify，以及对应的大量回调的虚函数，如OnRead、OnWrite、OnTimeout

##### 成员变量：

m_iEvents int型变量，epoll event的event，用于加入epoll时的标识；

m_poEventLoop EventLoop型指针，说明该事件所运行的loop；

m_bIsDestroy bool型变量，事件是否可被销毁

##### 成员函数：

ctor：指定EventLoop

dtor：null；对应的析构应该在event的子类中完成，基类无实际功能，不会被独立使用

读写回调：OnRead、OnWrite

定时器回调：OnTimeout

错误回调：OnError

向EventLoop中新增和移除定时器，向EventLoop中修改/新增（loop中找不到就新增）或者移除epoll中的关注事件

（最终调用epoll中的注册事件epoll_ctl）



#### eventloop.h

定义了一个reactor模式中的主循环基类EventLoop（核心），phx采用了epoll多路复用；网络循环的开始结束、超时处理定义在此类中，EventLoop会同时处理多个Event，所以需要实现存放多个Event指针的容器，并处理容器内的成员变动；此外该类的核心在于OneLoop函数的实现，即epoll中fd就绪后的实际处理流程

EventLoop中维护了两个Event相关容器，map和vector，增加和删除Event的操作在map中完成，而dtor时在于vector

![phxpaxos eventloop机制](F:\Markdown\研一上\图片\phxpaxos eventloop机制.png)





定时器的处理机制

![phxpaxos 超时处理机制](F:\Markdown\研一上\图片\phxpaxos 超时处理机制.png)

##### 成员变量：

定时器类（phx的Timer不是一个个独立的个体，Timer是个定时器集合体，内部拥有定时器容器）

map < uint32_t, int > m_mapTimerID2FD： TimerId到Fd的映射关系，用于定时器的增减

queue< pair< int, SocketAddress  > >m_oFDQueue：fd和对应地址信息的队列，即等待加入epoll的fd集合

vector< MessageEvent *> m_vecCreatedEvent：Message Event指针的容器，存放所有的fd对应的Message Event，并且在新的fd加入到loop时会清理vector中已经标记为可销毁（OnError处理）的Event

Epoll相关变量

int m_iEpollFd：epoll句柄

map< int, EventCtx_t> m_mapEvent：拥有由epoll监听的fd与事件上下文结构体的映射关系，事件上下文中有Event的指针；用于epoll就绪后，通过fd直接查询到对应事件；或者新增/移除epoll中的Event监听

> ```c++
> //但是Event中同样有m_iEvents，重复？
> typedef struct EventCtx
> {
>   Event* m_poEvent;	
>   int m_iEvents;	//epoll event的event
> }EventCtx_t;
> ```

epoll_event m_EpollEvents[MAX_EVENTS]：epoll_event 的数组，最多支持1024个epoll_event（理解为fdset[]）



Notify、NetWork、TcpClient对应指针

##### 成员函数：

dtor：移除Event容器中的标记为可销毁的所有事件，直接释放内存，并从容器移除，若未标记则跳过；也就是说EventLoop结束循环时，如果事件仍在运行则不影响Event本身或者说loop仅释放错误的Event内存

OneLoop：epoll_wait驱动，就绪fd中逐个处理 - OnRead、OnWrite、OnError，其中read和write由对应Event直接处理，而Error会先由loop处理，接着转入Event的error处理

> epoll看起来比select易用多了2333

OnError：先从loop的map中移除，然后根据Event本身的错误处理，如果错误明确，则将该Event标记为可销毁

DealwithTimeout：与ioloop的基本一致，从定时器中获取最近的可能超时事件，如果已超时则直接处理（即调用Event的OnTimeout），同时获取下一个最近超时事件（此超时事件将作为epoll wait的超时时间），从而逐个处理所有超时事件；有一点和mom的定时器不一样，处理完某个超时事件后，此timerid会从loop的容器中移除，也就是说定时器不可以重置。

CreateEvent：为m_oFDQueue中等待加入epoll的fd新建Event，并将事件加入epoll（EPOLLIN），及m_vecCreatedEvent；

#### message_event.h

Event的子类，定义了消息事件的read、write、超时处理回调

##### 成员变量

queue< QueueData> m_oInQueue：发送缓冲区（队列）

（接收缓存区位于ioloop）

```C++
struct QueueData
{
  uint64_t llEnqueueAbsTime;
  string * psValue;
};
```

uint64_t m_llLastActiveTime ： 标识最近一次使用message的时间，借以判定超时；新增Message时会更新





##### 成员函数

eventloop中接收到的数据需要放到ioloop的消息队列中

OnRead

OnWrite：将发送缓冲队列中的所有消息和发送Cache中的残留消息发出；如果发送缓冲区和Cache为空则从epoll中移除写事件监听。由此函数可知，消息发送流程为：eventloop中写就绪，触发事件发送，则从发送缓冲队列取出消息并附上消息长度，放入m_oWriteCacheBuffer进行发送，直到发送缓冲区为空或者当前内核发送缓存区忙为止

