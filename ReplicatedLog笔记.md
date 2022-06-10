以往只是大略读ReplicatedLog代码，属于需要哪里才看哪里，本次打算细读此文件设计，进而设计自己的catch up环节，并且让整个paxos"动"起来，能够保证正确性的执行下去，也就是说将正常paxos递增，追赶，集群变更（选举）整合。





#### ReplicatedLog

ReplicatedLog是一致性的最上层，当有数据需要读取时，实际不经过一致性层，直接从数据库读取即可，而有数据需要写入时，需要通过paxos处理（并且只有master节点才可以做此处理），确保各服务器之间的数据一致。

##### 成员变量

因为ReplicatedLog是一致性的最上层，封装（或者说隐藏）了具体的paxos细节，所以其内部成员首先需要paxos的三角色，proposer、acceptor、learner，而此三角色是依附于ReplicatedLog的，当ReplicatedLog被析构，三角色理应被释放，所以之间为组合关系；同时因paxos之间的消息交互需要，所以同样需要网络传输类，ReplicatedLog中读写被分离，TransportTCPReader、TransportTCPWriter，同理log与传输对象也是组合关系；因系统运行中可能出现的网络或软硬件故障风险，所以选举类必不可少，而Keysapce中将基于paxos的选举封装为PaxosLease类，同理也是组合关系；

ReplicatedLog中除了持有以上paxos相关对象之外，还需要DB层的对象，以确保经过一致性后的值写入对应DB，和DB之间属于聚合关系，DB并不完全依存于ReplicatedLog，DB初始化时会设置其在Log中的指针（SetReplicatedDB设值注入，因为ReplicatedLog采取单例模式）。

ReplicatedLog与LogQueue和LogCache均属于组合关系，前者为等待一致性决策的value队列，后者为value chosen后的队列，相当于已经chosen的value的缓存，这样就不用catch up的时候，所有的value都从数据库get，因为除非长期宕机或者新主机加入，否则不会出现大量的catch up，也就是说catch up的value就在当前paxos id附近，使用缓存可以节约大量时间。

ReplicatedLogMsg rmsg：一致性层与外部的通信消息类，如client请求（KeyspaceMsg）写入，rmsg则负责暂存这些外部请求，所以rmsg中的value代表多个外部请求，这些外部请求又作为一个value放入paxos中等待决议；此对象只在Append和OnLearnChosen中被Init

PaxosMsg pmsg：paxos的消息类，负责paxos内部的消息存放和处理

CdownTimer	catchupTimeout；Func onCatchupTimeout：ReplicatedLog中负责不同服务器之间paxos实例号的追赶，此为catch up定时器以及超时处理，确保追赶的有效性

Func onLearnLease：此回调为paxos lease选主成功后被调用，如果新当选的master节点当前paxos p1，p2并未执行或者当前的收到的外部请求不是nop，就会先Append(nop)，从而在DB中追加nop，以确保数据库安全？？

Func onLeaseTimeout：此回调在租期超时时被调用，发生租期超时即认为当前DB不再安全（说明续租失败，可能有节点或网络异常），会暂停proposer（清空proposer的value，重置p1，p2状态，移除p1，p2的定时器），DB也会做相应操作？？？，但此时PaxosLease中其他节点会开始新的选举

以上两个回调均为paxos lease和paxos之间的衔接，分别对应选举成功后和选举发生前

Func onRead：TransportTCPReader的读取回调函数，在TransportTCPConn::OnMessageRead中调用，之后在ProcessMsg中根据不同的消息类型做对应处理

以上回调和定时器均在ReplicatedLog构造时赋值

bool safeDB ？？？？

uint64_t highestPaxosID：paxos流程中，所有实例号中最高的；在OnLearnChosen中，当前实例号learn完毕，并进入下一轮实例后（paxosid + 1），如果此时的paxos id仍小于highestPaxosID，说明还需要追赶，此情况多发生在有多个实例号差距时，每补完一个实例差距，用此法继续补充下一个实例。

ByteBuffer value：即为paxos中需要决议的value，而仅用于Append和OnLearnChosen时，Append中如果有外部请求，首先push到LogQueue中，并产生ReplicatedLogMsg ，然后将此value解析，并由proposer.Propose(value)；OnLearnChosen中如果当前proposer空闲且logQueue中仍由value，则同样产生ReplicatedLogMsg ，并由proposer.Propose()

> 看着有点迷糊，RLOG和RDB层到底是哪边先收到客户端的请求，从RLOG看的话，在当前实例chosen后，会调用DB中的Append()，而DB中的Append()同样会调用RLOG的Append(value)

LogCache logCache;
LogQueue logQueue;



###### 疑问：

LogCache是在进入DB之前在加一个缓冲吗？LogQueue是在进入一致性之前在加一个缓冲吗？有此必要吗？

LogCache是DB前的缓冲，但作用是方便快速catch up！





##### 成员函数





###### 单例模式

ReplicatedLog采取单例模式，整个系统中只有一个ReplicatedLog提供一致性服务，即RLOG 

`#define RLOG (ReplicatedLog::Get())`

通过使用static来记录唯一的实例，且仅暴露一个获取实例的接口（Get），此接口也是用static修饰，确保全局可调用。而且单例内部会禁用对象拷贝和赋值。

```C++
//常见的单例实现--返回对象引用；keyspace中为返回对象指针
//仅对外提供一个访问接口
class Singleton
{
  public:
  	static Singleton& Get();
  	//static Singleton* Get();
  
  	Singleton(Singleton& ) = delete;
  	Singleton& operator=(Singleton& ) = delete;
  private:
  	Singleton();
  
  	//static Singleton* instance;
  	//other data member
};

Singleton& Get()
{
  static Singleton instance;//采用local static，自动初始化
  return instance;
}

//
Singleton* Get()
{
  if (instance == nullptr)
  {
    instance = new Singleton();
  }
  return instance;
}
```





#### ReplicatedLogMsg

##### 成员变量