更多的考虑整个生命周期，现在只是更多的关心谁创建，从不关心谁释放；何处创建，何处关闭、析构，谁负责这些？如fd的何时关闭，谁来关闭



loop创建之初（init）并不需要将acceptor的fd加入读集，因为此时的acceptor根本不存在，所在loop也没确定（虽说设计上是放到thread 0），更何谈其对应的fd；

acceptor是在mainloop启动时加入并创建fd的，但此操作是在main thread中完成，若要直接加入acceptor对应的loop的（poller的）读集也可以（因为都在线程间共享的堆区，取消IsInLoopThread判定后，执行没问题），但是需要加锁，因为跨线程修改和本地线程修改如果同时发发生，会冲突；所以此处设计为跨线程修改对应的消息队列并搭配wakeup激活select就绪，fd set的修改异步执行，而消息队列修改时加锁！

~~但此时loop所在的thread物理上已经创建完毕，从main thread直接修改（设计上）应该归属thread 0的资源不适合，所以通过消息队列搭配wakeup（疑问：但还是相当于main thread直接修改了thread0 的loop对应的消息队列？）~~是的，区别在于消息队列加锁了



不像wakeup的一对fd，wakeup的fd是在thread 0实际创建前（即物理意义上的分配堆栈等）就放到loop poller的监听事件中，因为wakeup的设计就是每个loop一对，一开始就确定了，而且只会在启动的时候出现，所以不用考虑冲突，也就不需要加锁





线程在 pthread create之后，其中的变量如何继承？或者共享了哪些？

线程间共享堆内存和全局变量，只有线程内的函数栈时局部的

pthread create时，传入的最后一个参数（void *）是作为执行函数的参数，不能传入局部变量，因为pthread create是异步的，新线程执行时，当前线程可能已经离开作用域，传入的指针变成野指针。



Q：thread 0 是怎么拿到loop中的变量的？通过create时传入的thread*参数吗？

A：共享堆内存，而loop等是new出来的



#### momServer

从mom server的角度看待其网络库流程

由于采用多线程和对象语义，所以将针对本地的acceptor和其他mom的acceptor放在同一线程，其生成的connection放入其他线程，这一切通过mainloop初始化和启动；（没有connector，说明所有client初始状态即为与server连接，所以通过此connection即可，无需connector创建专用于发送的connection，看完client代码再回来补充）mom间的connection连接：通过新mom启动后，发送广播报文，然后旧mom主动去连接新mom（而threadframe中，mom间的connect通过配置文件形式的小IP连接大IP实现互连）



client发送消息至server，分为控制和数据报，server的handle read根据不同消息类型处理：

若为控制报，则更新订阅表，因为控制报用于订阅或退订消息；

若为数据报，根据订阅信息表，向所有订阅了消息的节点发送消息；（通过消息队列异步处理？不是直接发送）





#### HostSubInfo ： CheckLock

HostSubInfoList中维护了两个表：

进程 + 订阅的主题列表

主题 + 订阅该主题的进程列表

每次订阅或退订，两张表均需要修改



HostSubInfo通过继承CheckLock，拥有了HostSubinfoList，并且加锁更少，效率更高，但是为什么CheckLock设计成了泛型类



ProcessLastData（）中，从内存池申请空间，应该是connection发送完毕后再释放，那么如果有大量mom连接上去，内存池空间全部用于这个的临时占用了，当然设计时应该不会出现大量mom



ProcessLastData采用内存池和文件缓存的二级缓存机制，内存池没资源，全部放入文件；下次调用时先将文件缓存取出





#### TimerQueue

为保证timer*是按照超时时间点```when_``` 的升序排列，即最早超时的先执行，定时器队列的设计是set timerList_；

set在遍历执行回调时需要先判断是否是有效定时器；其次是否是重复定时器，如果是重复定时器，则需要重置到下次的超时时间结点，但因为重置超时时间后set并不会自动重新排序，所以需要删除该timer*，再重新插入set；所以此处设计为将所有超时需要执行回调的定时器全部存放到另一个set（vector也可以吧，因为pushback的顺序是从小到大的）中，统一执行回调，遍历这个set时顺路检查定时器是不是重复型，将重复定时器插回timerList_（再次排序）。



#### MainLoop

与momserver相关，主要用于server启动时，初始化并启动acceptor，**定义及实现mom server所需的相关回调**！如acceptor监听到连接后的connection如何新建，以及conncetion对应loop如何确定；新增、删除connection；loop的消息



AddConnection：新增connection，加入connectionMap_

若对端断开，从而对端connector主动发起重连，但本地保存的之前的connection还存在（因为本地connection可能没有发送动作，所以未感知到对端断开）这是就需要删去旧的connection，另外connection的析构由其所在线程负责，需要发送消息通知



检查是否为旧的connection时，不考虑本地发起的重连，因为本地断开很容易感知到，所以不考虑本地断开而无感知

但是检测是否为同一个connection时，不检查端口？

因为connector发起的connection端口一直在变还是说mom间连接的端口固定了？ **待查connector代码**

```C++
if (strcmp(connection->GetPeerAddr(), "127.0.0.1") != 0 &&strcmp(iterConn->second->GetPeerAddr(), connection->GetPeerAddr()) == 0 &&iterConn->second->GetLocalAddr() == connection->GetLocalAddr())
{
//connection的析构由其所在线程负责，需要发送消息通知
	char tmpbuf[8+8] = {0};
	*(uint64_t *)tmpbuf = NetUtil::IPtoUint64(iterConn->second->GetPeerAddr(), iterConn->second->getPort());
	memcpy(tmpbuf + 8, iterConn->second,sizeof(Connection*));

	iterConn->second->GetLoop()->AddMsg(MSG_DATA, DATA_DEL_CONNECTION, tmpbuf, sizeof(uint64_t)+8);
	iterConn->second = NULL;
	connectionMap_.erase(iterConn++);
}
```

