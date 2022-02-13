### Src/algorithm目录

phxpaxos中每个状态机对应一个paxos group，如果有多个业务可以使用多个状态机，从而对应多个paxos group，所以需要有group id标识组别，但是每个paxos group内部的实例是串行执行的，防止空洞产生，每个实例直接互不影响，上一个实例写入状态机并完成状态转移才可以开始下一个实例；不过从宏观上看，多个group的实例是并行执行。



状态转移：每个实例所确定（chosen）的值只是状态转移的输入值；如果是kv存储状态机，也就是说每个实例确定的是数据库的某项操作（如：set(x, 2)），状态转移负责将此操作落入数据库中；而如果是echo服务，实例确定的值是收到的某个echo值，状态转移就是将此值再次传回就可



#### keyspace & phxpaxos中state值定义：

| Keyspace                 |                                          | Phx                            |                                          | 比较                                       |
| ------------------------ | ---------------------------------------- | ------------------------------ | ---------------------------------------- | ---------------------------------------- |
| **PaxosProposerState**   |                                          |                                |                                          |                                          |
| 属性                       | 值含义                                      | 属性                             | 值含义                                      |                                          |
| proposalID               | 提案号                                      | m_llProposalID                 | 提案号                                      | 两者均为uint64_t                             |
| higestReceivedProposalID | acceptor发来的promise消息中有其已接受的value对应的提案号（当proposer收到多个带value的promise时，此值用于挑选最大提案号的value） | m_oHighestOtherPreAcceptBallot |                                          | 同，不过使用了二元组标识提案号                          |
| higestPromisedProposalID | 曾收到过的acceptor发来的promise消息中的最大提案号（用于配合寻找下个提案号） | m_llHighestOtherProposalID     | 拒绝了本次提案的别的proposer发出过的最大提案号（提案号的比较，封装在SetOtherProposalID中） | 同                                        |
| value                    | 提案值                                      | m_sValue                       | 提案值                                      | keyspace中是ByteBuffer；Phx中是简单的string      |
| numProposals             | 提案次数（第一和第二阶段共用）                          |                                |                                          |                                          |
|                          |                                          |                                |                                          |                                          |
| **PaxosAcceptorState**   |                                          |                                |                                          |                                          |
| 属性                       | 值                                        | 属性                             | 值                                        |                                          |
| promisedProposalID       | 曾回复（promise）过的最大提案号（即之前对于prepare的肯定回复）   | m_oPromiseBallot               |                                          | 同，但phx在acceptor中采用二元组，接收到来自proposer的消息后根据消息内容构造出该值 |
| acceptedProposalID       | 之前所接受提案的提案号                              | m_oAcceptedBallot              |                                          | 此值对应propser state中的higestReceivedProposalID |
| acceptedValue            | 提案值                                      | m_sAcceptedValue               | 提案值                                      | 同proposerState                           |
| **LearnerState**         |                                          |                                |                                          |                                          |
|                          |                                          | 属性                             | 值                                        |                                          |
|                          |                                          | string m_sLearnedValue         |                                          |                                          |
|                          |                                          | bool m_bIsLearned              |                                          |                                          |
|                          |                                          | m_oPaxosLog                    |                                          |                                          |



Q：这些状态信息是需要落盘的，防止宕机重启后数据丢失，但是落盘的具体操作在哪里？logstore？



#### keyspace basic paxos流程回顾：

两阶段分别为preparing、proposing

1a. proposer发出prepare消息，申请访问acceptor，并附上本期提案号`void PaxosProposer::StartPreparing()`

**Detail**：首先停止可能存在的proposing过程（如果需要多次提案，不同提案间的第二、第一阶段会紧跟着），提案次数+1，提案号并不是简单的+1，各个proposer间提案号不会重复，keyspace中采用：如三台服务器，其中一台提案号为1，4，7...，通过RCONF->NextHigest获取（通过提案号和曾收到过的promise中最大提案号取最大，并找下一个合理提案号实现），所以消息中封装[实例号、提案号、proposer节点号]即可发送给acceptor；由于prepare消息发出后状态未知，需要设置超时时间（**关注超时处理1**）



1b. acceptor收到prepare消息后，比对自身状态（所以acceptor需要持久化一些状态信息），并给出promise成功或拒绝 `PaxosAcceptor::OnPrepareRequest(PaxosMsg& msg_)`

**Detail**：acceptor收到prepare后，首先检查prepare msg中的提案号是不是大于等于（可能有proposer重发消息）自己曾经回复（promise）过的最大提案号，小于代表acceptor可以拒绝此次提案，所以需要对应的拒绝回复消息，消息封装为[实例号、proposer节点号、提案号、曾回复过的最大提案号]；如果提案号通过检查，则更新acceptor曾promise的最大提案号`promisedProposalID`；此时有两种情况：

1）此acceptor从未接受过提案值

消息中封装[实例号、proposer节点号、提案号]

2）此acceptor接受过某个提案（即某个proposer的第一阶段达到半数同意，成功发出了accept消息到acceptor）

消息中封装[实例号、proposer节点号、提案号、之前所接受提案的提案号、之前所接受提案的提案值]

因为后者认可前者

最后，修改state后需要持久化



2a. proposer检查收到的promise成功消息是否达到半数以上，达成则发起accept消息，否则重新开始prepare`void PaxosProposer::OnPrepareResponse(PaxosMsg& msg_) 此函数对每一个acceptor的回应均触发`

`void PaxosProposer::StartProposing()当有半数promise成功才会触发`；注意如果每个阶段接收成功达到半数以上后，直接进入下一阶段，则此阶段的定时器会被移除

**Detail**：检查当前阶段，并核实收到的promise消息中的提案号是否于自身提案号一致；收到的promise会有计数器，因为要检查promise是否成功，所以针对拒绝的promise消息，设置拒绝计数器，如果达到半数以上，直接重新开始prepare，同时如果拒绝消息中的acceptor已回复过的最大提案号大于proposer中保存的，则更新，从而在下一次prepare时，配合寻找到对应的提案号；针对promise成功后存在的两种情况，

1）》》》

2）保存下acceptor接受过的最大提案号的提案值

![kspace图片1](F:\Markdown\研一上\图片\kspace图片1.png)

(相当于取了acceptedProposalID的最大值)



每个acceptor的回应中均检查有无达到半数以上要求，半数以上拒绝则直接重新选举，半数以上通过则直接第二阶段，若以上均未达到，但接受到所有回复消息？？？重新选举（**关注超时处理1**）



以上均为第一阶段回复消息的检查，检查通过才进入第二阶段：

消息封装为[实例号、proposer节点号、提案号、提案值]；清空检查阶段时的计数器；开启第二阶段计时器，设置超时时间（**关注超时处理2**）；此处的提案值已经遵循“后者认可前者”



2b.  acceptor收到proposer的提案值后，检查提案号是否大于等于自身曾回复过的提案号，如果不符合直接拒绝此提案，否则覆盖原接受的提案号和值，即使这个值和原值是一样的（防止节点接连多次故障情况下，先被接受的值反被覆盖），并回复接受消息，封装为[实例号、proposer节点号、提案号]`void PaxosAcceptor::OnProposeRequest(PaxosMsg& msg_)`

`void PaxosProposer::OnProposeResponse(PaxosMsg& msg_)`



proposer接收到acceptor的接受消息后，检查paxos阶段，并核实收到的promise消息中的提案号是否于自身提案号一致（即是不是给自己的消息）；使用成功接受消息的计数器和收到的消息总数计数器区分有无达到下一步学习值要求；对于成功消息未达大多数，但消息又未收到全部的情况（如三台acceptor，收到一台成功，一台拒绝，最后一台的未到），在超时中处理（**关注超时处理2**）



A. prepare超时处理（**关注超时处理1**）

直接重新开始preparing阶段

B. proposa超时处理（**关注超时处理2**）

直接重新开始proposing阶段

PS：似乎是从消息的一整个流程去考虑的超时，即消息发出-对端处理后回复，设置的超时时间覆盖此流程





#### phx basic paxos流程回顾：

同样两阶段，prepare、

1a. `void ProposerState :: NewPrepare()`

选取提案号采用了不同的策略，每次提案号+1；proposer发送时提案号和本节点号分开发出，而acceptor收到后，会将消息中的提案号和proposer节点号拼接出Ballot，acceptor两个阶段的判定时均使用Ballot；acceptor发出的自身节点的nodeid用于判定收到的消息数量，同时利用set< nodeid>可避免消息的重复（如双网口同时发出完全相同的报文）

![phx图片1](F:\Markdown\研一上\图片\phx图片1.png)



`void Proposer :: Prepare(const bool bNeedNewBallot)`

过程基本与keyspace一致，消息封装[消息类型，实例号，proposer节点号，提案号]，同样需要定时器，且定时器有对应的id



1b. `int Acceptor :: OnPrepare(const PaxosMsg & oPaxosMsg)`

拒绝和成功共同的消息部分：[消息类型，实例号，proposer节点号，源提案号(即proposer发出的提案号)]

成功消息

[曾接受过的提案号，曾接受过的节点号，曾接受过的提案值]

[曾接受过的提案号，曾接受过的节点号]（因为曾接受过的提案号为0）

拒绝消息[拒绝源提案号的提案号]







proposer检查acceptor的回复`void Proposer :: OnPrepareReply(const PaxosMsg & oPaxosMsg)此函数对每一个acceptor的回应均触发`，并修改状态信息

phx中采用容器存放接受到的回复消息节点号、接受到消息中的拒绝节点号、接收到的消息中同意节点号，而keyspace仅使用计数器receive、reject num；

区分在于phx中如果拒绝数达到半数以上或接受到全部消息但未通过，会将计时器后移30ms，30ms后重新开始prepare；kspace中直接开始prepare



2a. 检查通过进入第二阶段，向acceptor发出提案，定时器开启`void Proposer :: Accept()当有半数promise成功才会触发`

共同消息封装[消息类型，实例号，proposer节点号，提案号，提案值，GetLastChecksum]



（phx设置提案值在哪里？？？keyspace收到都是promise(n, null)时，设置提案值在哪里？？）



2b. `void Acceptor :: OnAccept(const PaxosMsg & oPaxosMsg)`

成功[消息类型，实例号，proposer节点号，源提案号]

拒绝[消息类型，实例号，proposer节点号，源提案号，拒绝源提案号的提案号]

故proposer根据有无拒绝源提案号的提案号，判断消息拒绝否



proposer检查acceptor的回复`void Proposer :: OnAcceptReply(const PaxosMsg & oPaxosMsg)此函数对每一个acceptor的回应均触发`

针对拒绝的回复，需要更新曾经收到的最大的回复提案号，当重新prepare时，用以寻找下一个提案号







#### 学习值（实例的对齐）:

learner询问其他机器的相同编号的实例，如果发现该实例在其他机器已经被“销毁”，则说明该实例值已经被确定，直接将值拉取写入自己的当前实例中，然后继续询问下一个实例，直到实例编号和其他机器一样（提案的追赶，不是共识阶段的学习）



##### chosen后的通知

当Proposer检查accept reply获知某个提案被选中后，通过BroadcastMessage_Type_RunSelf_Final消息通知所有节点，其中包含自身节点；如果消息中的提案号和节点保存的accepted 提案相同，则learner更新状态信息（因为此提案值和acceptor的相同，所以不需要再次落盘，更新内存状态即可：如标记已学习，进而之后可以执行抓状态转移）。另如果存在follower节点，则将数据同步至follower（follower节点不参与共识，类似于热备节点）

处理函数为OnProposerSendSuccess

Q：为什么如果学习方的acceptedBallot为空，直接返回？不应该直接覆盖吗？而且如果学习方的acceptedBallot与learner发来的不一致，直接返回，那么真正的处理应该是什么？难道等后期追赶？

##### 提案值追赶：

当节点的实例号落后，则无法参与到当前阶段的共识提案，所以需要由learner主动追赶欠缺的实例，而learner消息的发送使用LearnerSender类封装，同时发送时做了流量控制，防止追赶挤占大部分网络带宽，导致正在运行的提案超时，出现正常的提案消息大量堆积。（Q：何时触发追赶，pnode发起？）

learner将当前节点的实例号、节点号打包，通过MsgType_PaxosLearner_AskforLearn消息发送给除自身外其他节点，



会控制LearnerSender的发送速率，1）一个LearnerSender一次只能控制一个节点的追赶请求，目前来看是一个Learner拥有一个Sender；2）Sender确认发送时，会做流量控制，一次发n个包后会sleep一段时间；且Sender的发送策略是一次发送一个chosen的实例值

对每一个追赶到的实例，均需要发送确认ACK回复（随安居士文章中理解为心跳），会有相应的超时处理；针对每个ack，被追赶方会用来更新自身的已确定实例号等信息；但在继续发送实例的时候，会累计检查ack回复，如果没有收到上一批的ack，不会继续再发送新的实例

但是，被追赶方全部发送完之后，马上设置m_bIsIMSending；那么针对每个ack的处理时，就不会更新已确定实例号等，也就是说不会对最后一批（此处的批量，针对ack批量而言）发出的实例进行确认处理，即使没有收到，也是直接交由下一次的提案追赶



**LearnerSender实现中涉及的锁的应用**：

`WaitToSend`







ioloop实现机制：

![phxpaxos ioloop机制](F:\Markdown\研一上\图片\phxpaxos ioloop机制.png)



phxpaxos采用消息队列机制，而非select下直接处理，即所有的paxos阶段产生的消息统一放入主线程（与select是否同线程呢）的共享消息队列（具体需要查看网络处理--select），ioloop的循环中通过阻塞+超时时间机制处理每个消息，但每轮循环只处理一个pasxosMsg会不会太低效；而且因为循环内还有`newvalue`，感觉只要值未被确定，就会有大量的重传消息。

~~如果消息的两阶段在本轮循环内未完成，就会重传消息（重跑对应流程），而且如果收到过任一拒绝回复（即使不是大多数拒绝），提案号就会提高；~~

**Detail**：系统启动时，设置初始超时时长为1000ms，则初始化时前面所有流程“空跑”，`newvalue`开始paxos算法，内部会新增定时器，如prepare超时定时器，则在后续循环中，根据定时器的实例id和对应消息类型处理超时错误，并且处理定时器函数本身会返回下一个最近的超时时间点，此时间用于消息队列的阻塞时长，阻塞时长内收到对应消息则会相应移除定时器；超时未收到则触发会在下个循环处理超时（下个循环处理超时，相比于select会不会影响效率，额似乎没啥影响）；为定位不同的超时处理机制，所以每个定时器会用实例id和消息类型标记

> 定时器为何使用vector，既然每次pushback新的定时器，然后再push_heap，类似维护为堆，何不直接用set或heap，定制好排序函数







### Src/communication目录

#### communication.h

Communicate类是MsgTransport类的派生类，或者说MsgTransport的实现类，以Config对象指针、Network对象指针、节点号、iUDPMaxSize作为参数构造；Send函数中实现了使用Tcp和Udp通道发送数据，所以是NetWork的又一次封装，是网络层基于业务的抽象层，但是业务的打包、拆包等操作又被封装在Base类中，Base类是paxos算法中proposer、acceptor等的基类

##### 成员变量

Config * m_poConfig

NetWork * m_poNetwork

nodeid_t m_iMyNodeID：本机节点标识，发送消息是忽略本机

size_t m_iUDPMaxSize：因为UDP存在报文大小限制，而TCP不存在
##### 成员函数

dtor：析构不需要负责，会由对应的Event自己管理生命周期

Send：private类型，如果消息过大，超出了udp发送范围或者发送类型指定为Tcp，使用m_poNetwork->SendMessageTCP发送数据（最终是使用TcpClient去实现数据发送）；否则使用m_poNetwork->SendMessageUDP

SendMessage：Send的public封装

BroadcastMessage：向除非本机外的同组主机（GetMembershipMap）广播消息

BroadcastMessageFollower：向除非本机外的同组主机（GetMyFollowerMap）广播消息

BroadcastMessageTempNode：向除非本机外的同组主机（GetTmpNodeMap）广播消息



#### udp.h

实现了Thread的两个派生类：UDPRecv和UDPSend，其中UDPRecv以NetWork对象指针为参数构造

##### UDPRecv成员变量

DFNetWork * m_poDFNetWork
int m_iSockFD：用于udp连接的fd
bool m_bIsEnd
bool m_bIsStarted
##### UDPRecv成员函数

dtor：如果fd错误，关闭udp连接

Init：udp连接的准备阶段，socket()、bind()

run：poll驱动监听fd读，读取消息后放入临时Buffer，转交Network OnReceiveMessage（最终放入ioloop的消息队列）



##### UDPSend成员变量

Queue< QueueData *> m_oSendQueue：需要发送的消息的队列，加锁队列

```C++
//需要发送的消息的封装
struct QueueData
{
  string m_sIP;
  int m_iPort;
  string m_sMessage;
};
```

int m_iSockFD：用于udp连接的fd

bool m_bIsEnd

bool m_bIsStarted

##### UDPRecv成员函数

dtor：释放queue中所有消息

Init：udp连接的准备阶段，即socket()

SendMessage：private类型，sendto的封装

run：循环发送队列中消息

AddMessage：向发送队列添加待发送的消息，注意加锁



**Q：**为何有些地方使用RAII的mutex helper，而Thread类的实现中使用lock、unlock？



#### dfnetwork.h

DFNetWork是NetWork的派生类，DF即为default缩写，说明该类是Network类的默认实现类，负责管理UDP和TCP的收发，内部有TcpIOThread、UDPRecv、UDPSend对象（非指针）；可以重写自己的network实现类，遵守NetWork中的虚函数要求即可

##### 成员变量

UDPRecv m_oUDPRecv
UDPSend m_oUDPSend
TcpIOThread m_oTcpIOThread
##### 成员函数

Init：完成TcpIOThread 、UDPSend 、UDPRecv 三个对象的初始化

RunNetWork：启动三个对象

SendMessageTCP、SendMessageUDP：内部实质调用对应对象的AddMessage，加入各自的发送队列即返回，属于异步发送



#### Network.h

NetWork基类，其实现有DFNetWork；Node为其友元，同时包含一个Node对象指针

##### 成员函数

OnReceiveMessage：调用pNode的OnReceiveMessage处理数据接收，最终通过Instance加入到IOloop的消息队列；只要入队即返回，实际消息的处理异步完成

### Src/communication/tcp目录

在工程项目的网络连接部分，并不是类似UNP书中的socket创建，绑定，监听，完成连接后处理放在一个cpp文件中完成，而是对连接的初始化和连接之后的处理拆分开，并分别封装为类，从而降低代码耦合性，同时采用非阻塞连接；拆分的模式一般为：Acceptors、Connectors和Service handles。其中Acceptor针对服务器端被动连接特性，开启监听端口提供被动连接；Connectors针对客户端主动发起连接的特性，提供主动连接对端并处理非连接失败等问题；Service handles只需要处理连接建立之后的数据处理、传输部分，无论是acceptor的被动连接还是connector发起的主动连接，均属于Service handles处理，而且针对不同的处理方式，Service handles可以派生出多种子类，完成不同任务；

phxpaxos中对应于acceptors的类即为TcpAcceptor类，对应connectors的是TcpClient，对应于Service handles的类是EventBase，其派生类中专用于事件处理的即为MessageEvent，用于跨线程通知的是Notify类；且关于网络通信的部分封装在NetWork基类中，同时提供了一种网络通信的默认实现DFNetWork，此设计下只需要继承NetWork类，并实现对应虚函数即可使用自定义的网络通信

**TCP读**：

phxpaxos中共识部分都是直接从ioloop的消息队列中逐个处理消息包，所以网络部分TCP读取操作需要完成的功能即为读取网络连接中的消息并放入ioloop的消息队列；其中TcpAcceptor会开启端口监听（Thread::TcpRead中run）且由poll驱动，对所有accept后的连接放入EventLoop的容器中，由EventLoop为这些fd构造MessageEvent，并为此Event在epoll中设置读关注，当读事件就绪，则交由MessageEvent Read回调处理（放入ioloop的消息队列）

**TCP写**：

共识部分发出提案请求时需要向其他所有节点广播节点，所以需要为每个节点主动建立连接，当有请求需要发送时，phxpaxos默认网络实现中使用TcpClient主动发起连接（Thread::TcpWrite中run），连接完成后为此构造相应的MessageEvent，每个MessageEvent会配置一个发送缓冲队列，且在EventLoop中为这些Event在epoll中设置写关注，当写就绪（请求发起时，其他线程将请求写入Event线程所在的发送缓冲队列，并发送notify触发写就绪）后，则交由MessageEvent Write回调处理

可以看出phxpaxos的默认网络实现中同一对节点间的TCP读写连接相互独立，读写不会共有同一组连接；而且acceptor建立的连接对应的Event构造由EventLoop完成，而Client发起的连接由其自身完成Event构造，但开启写关注仍由EventLoop完成。

**Q：**Tcp写时，主动发起非阻塞连接，但是似乎未考虑10035情况，只要connect 返回-1，直接抛出错误；另linux中EINPROGRESS错误的触发机制或为什么触发？



**UDP读**：使用Thread::UdpRecv，poll驱动，线程中循环等待即可，收到数据后同样放入ioloop的消息队列



**UDP写**：使用Thread::UdpSend，同样配置了一个发送缓冲队列，线程中循环发送队列中数据即可，发送缓冲队列同样需要请求发起方存入消息，即采取异步触发式发送消息，而且因为udp的发送采用自身线程无限循环，而非eventloop的epoll，所以不需要唤醒



#### eventbase.h

定义了Event抽象类，派生出子类：MessageEvent、Notify，以及对应的大量回调的虚函数，如OnRead、OnWrite、OnTimeout

##### 成员变量

m_iEvents int型变量，epoll event的event，用于加入epoll时的标识；

m_poEventLoop EventLoop型指针，说明该事件所运行的loop；

m_bIsDestroy bool型变量，事件是否可被销毁

##### 成员函数

ctor：所在的EventLoop

dtor：null；对应的析构应该在event的子类中完成，基类无实际功能，不会被独立使用

读写回调：OnRead、OnWrite

定时器回调：OnTimeout

错误回调：OnError

向EventLoop中新增和移除定时器，向EventLoop中修改/新增（loop中找不到就新增）或者移除epoll中的关注事件

（最终调用epoll中的注册事件epoll_ctl）



#### eventloop.h

定义了一个reactor模式中的主循环基类EventLoop（核心），phx采用了epoll多路复用；网络中事件循环的开始结束、超时处理（实际是在此处调用对应处理的回调）定义在此类中，EventLoop会同时处理多个Event，所以需要实现存放多个Event指针的容器，并处理容器内的成员变动；此外该类的核心在于OneLoop函数的实现，即epoll中fd就绪后的实际处理流程

EventLoop中维护了两个Event相关容器，map和vector，map负责epoll监听的所有事件，所以epollwait中关注的事件修改时，同步完成map的修改，而dtor时在于vector中loop自身所创建的接收事件



###### epoll

1、epoll核心函数

epoll使用简要流程：

```C++
int iEvent；
//创建epoll对象，创建出的对象本身是一个fd，所以结束epoll后需要close
//epoll_create中的参数指想关注的fd数量，但似乎现在版本的epoll只需要此值大于0即可
iEvent = epoll_create(MAX_EVENTS);

epoll_event tEpollEvent;
tEpollEvent.events = EPOLLIN | EPOLLOUT;
tEpollEvent.data.fd = fd;

//控制epoll事件，可以新增/注册、删除、修改fd，默认是水平触发
//epoll中新增fd的读、写关注
epoll_ctl(iEvent，EPOLL_CTL_ADD, fd,  &tEpollEvent);

//保存epoll_wait返回的已触发的事件，后续处理遍历该数组即可
epoll_event Events[MAX_EVENTS];
for(;;)
{
  //类似select，等待事件触发；最后超时时间-1代表阻塞式
  int nready = epoll_wait(iEvent，Events, MAX_EVENTS, -1);
  for (int i = 0; i < nready; ++i)
  {
    if (Events[i].events & EPOLLIN)
    {
      //do some read
    }
    if (Events[i].events & EPOLLOUT)
    {
      //do some write
    }
  }
}
//务必关闭epoll对象
close(iEvent);
```

涉及结构体：

XXX

2、水平和边缘触发

3、epoll和select区别和提升

首先，select返回的是就绪fd数量，而epoll就绪后返回的还包含已就绪的fd集合，所以epoll就绪后的处理，只需要遍历epoll_wait的返回集合即可，



![phxpaxos eventloop机制](F:\Markdown\研一上\图片\phxpaxos eventloop机制.png)





定时器的处理机制

![phxpaxos 超时处理机制](F:\Markdown\研一上\图片\phxpaxos 超时处理机制.png)

##### 成员变量

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
>   int m_iEvents;	//epoll event的event，如EPOLLIN、EPOLLOUT、EPOLLERR等
> }EventCtx_t;
> ```

epoll_event m_EpollEvents[MAX_EVENTS]：epoll_event 的数组，最多支持1024个epoll_event（理解为fdset[]）



Notify、NetWork、TcpClient对应指针

##### 成员函数

dtor：移除在此eventloop中构建出的可销毁Event（即Event容器中的标记为可销毁的所有事件），直接释放内存，并从容器移除，若未标记则跳过；也就是说EventLoop结束循环时，如果事件仍在运行则不影响Event本身或者说loop仅释放错误的Event内存；也印证了C++中”谁创建谁就负责关闭“

OneLoop：epoll_wait驱动，就绪fd中逐个处理 - OnRead、OnWrite、OnError，其中read和write由对应Event直接处理（m_mapEvent中直接检索得到Event），而Error会先由loop处理，接着转入Event的error处理

> epoll看起来比select易用多了2333

OnError：先从loop的map中移除，然后根据Event本身的错误处理，如果错误明确，则将该Event标记为可销毁

DealwithTimeout：与ioloop的基本一致，从定时器中获取最近的可能超时事件，如果已超时则直接处理（即调用Event的OnTimeout），同时获取下一个最近超时事件（此超时事件将作为epoll wait的超时时间），从而逐个处理所有超时事件；有一点和mom的定时器不一样，处理完某个超时事件后，此timerid会从loop的timer容器中移除，也就是说定时器不可以重置。

`void EventLoop :: CreateEvent()`：为加锁队列m_oFDQueue中等待加入epoll的fd新建Event（每个循环最多处理200个fd），并将事件加入epoll，开启读监听（EPOLLIN），新建出的Event同时加入到m_vecCreatedEvent；此函数只会为acceptor类中accept后的fd创建Event，所以只监听EPOLLIN即可

AddTimer：新增定时器，加入Timer容器管理；而如果新增定时器的该Event类不在m_mapEvent中，则加入m_mapEvent管理



#### message_event.h

Event的子类，定义了消息事件的read、write、超时处理回调

##### 成员变量

queue< QueueData> m_oInQueue：发送缓冲区（队列）

（接收缓存区位于ioloop）

```C++
struct QueueData
{
  uint64_t llEnqueueAbsTime;//标记要发送的消息的入队时间，用于超时判定
  string * psValue;
};
```

uint64_t m_llLastActiveTime ： 标识最近一次使用message的时间，借以判定超时；新增Message时会更新

m_iReconnectTimeoutID：专用于重连的定时器ID

BytesBuffer m_oWriteCacheBuffer、m_oReadCacheBuffer分别用于Write和Read

而m_sReadHeadBuffer用于Read时保存消息的头部信息



##### 成员函数

OnWrite：将发送缓冲队列中的所有消息和发送Cache中的残留消息发出；如果发送缓冲区和Cache为空则从epoll中移除写事件监听。由此函数可知，消息发送流程为：eventloop中写就绪，触发事件发送，则从发送缓冲队列取出消息并附上消息长度，放入m_oWriteCacheBuffer进行发送，直到发送缓冲区为空或者当前内核发送缓存区忙为止

![phxpaxos eventloop OnWrite](F:\Markdown\研一上\图片\phxpaxos eventloop OnWrite.png)

OnRead：先recv消息头从中获取消息长度，根据消息长度扩容接收Cache，继续recv消息内容至Cache，接收完成后放入ioloop的接收缓冲区m_oMessageQueue（eventloop中接收到的数据需要放到ioloop的消息队列中）

OnError：

1）接收消息错误：标记为可销毁，在loop中会释放所有可销毁事件

2）发送消息错误：如果事件仍active，则新增定时器，类型为MessageEventTimerType_Reconnect，后延200ms；如果事件未active，则将发送队列中的所有剩余消息释放，并标记可销毁

OpenWrite：将发送缓冲队列中的消息加入到epoll写监听，但若此事件标记为可销毁，则发起重连

`int MessageEvent :: AddMessage(const std::string & sMessage)` 将需要发送的数据加入到Event的发送缓冲队列，且队列是加锁的，意味着某事件的发送缓冲队列可能被多个线程同时写入；加入发送缓冲队列后，还需要唤醒eventloop

**Q：**（为什么梳理的时候没看出会被多线程并发访问）

**A：**网络中为每对节点的读、写建立对应message_event，所以当paxos的两阶段的消息同时发出（一阶段的包可能是重发或其他情况）时，该队列会被同时写入，所以加锁



#### tcp.h

定义两个继承自Thread类的派生类：TcpRead和TcpWrite以及一个TcpIOThread类，IOThread管理着多个TcpRead和TcpWrite

##### TcpIOThread成员变量

vector< TcpRead *> m_vecTcpRead

vector< TcpWrite *> m_vecTcpWrite

TcpAcceptor m_oTcpAcceptor

##### TcpIOThread成员函数

dtor：将TcpRead、TcpWrite容器中的类全部释放

Init：负责TcpRead和TcpWrite的创建并加入epoll监听，同时开启m_oTcpAcceptor的监听并放置到和TcpRead相同的EventLoop，TcpRead、TcpWrite对的数量在网络启动时指定

Start：将TcpRead和TcpWrite在线程中启动

AddMessage：利用TcpWrite（本质是TcpClient）将需要发送的消息加入到该IP和port对应节点的Event的发送缓冲区



#### TcpAcceptor.h

TcpAcceptor同样是Thread的派生类，负责将新的fd加入EventLoop，不过采用了poll驱动，而之前EventLoop用epoll驱动，为什么用两种完全不同的io模型

##### 成员变量

vector< EventLoop *> m_vecEventLoop：TcpAcceptor可以将所有新的已完成连接放入EventLoop中，此EventLoop和TcpRead是同一个

##### 成员函数

run：poll驱动，将accept产生的新fd加入Event最少的EventLoop（负载均衡）

AddEvent：遍历m_vecEventLoop，找到活跃事件最少的Loop，并将新的fd加入Loop，即loop中的加锁队列m_oFDQueue

###### poll





#### TcpClient.h

TcpClient类，以EventLoop和NetWork的对象指针作为参数构造，两指针对应两个成员变量，是Tcp客户端类；


##### 成员变量
EventLoop * m_poEventLoop

NetWork * m_poNetWork

map< uint64_t, MessageEvent *> m_mapEvent，由ip和port组成一个32位的nodeid为key，value是与此客户端连接的fd对应的MessageEvent 对象指针

vector< MessageEvent *> m_vecEvent

##### 成员函数

`dtor`负责所有MessageEvent 的释放，~~所以MessageEvent会受到EventLoop和TcpClient的同时管理~~

更正：TCPClient中所管理的MessageEvent，均为用于发送的连接；而EventLoop中析构的Event是accept后的接收的连接；共同点在于”谁创建谁析构“

`int TcpClient :: AddMessage(const std::string & sIP, const int iPort, const std::string & sMessage)`检索到对应ip，port的Event对象，将消息加入到MessageEvent 的发送缓冲队列

`MessageEvent * TcpClient :: GetEvent(const std::string & sIP, const int iPort)`检索指定IP和port的Event对象，如果该客户端不存在ip和port对应的Event对象，则主动发起向ip+port的连接，并新建Event对象，同时加入该客户端的Event对象指针容器

`void TcpClient :: DealWithWrite()`客户端所有的Event加入EventLoop的写监听

`MessageEvent * TcpClient :: CreateEvent(const uint64_t llNodeID, const std::string & sIP, const int iPort)`主动发起向ip+port的连接，并新建Event对象，新建的Event对象和Client处于同一Eventloop，同时加入该客户端所管理的Event对象指针容器

**Q：**因为windows移植问题，使用了winport.h，而TcpClient类中的CreateEvent函数调用过程中出现问题，检索发现在`synchapi.h`中有`#define CreateEvent CreateEventA`用于非UNICODE下的创建事件对象，与TcpClient中函数名称冲突，但是这两个函数即使重名，不应该当作函数重载吗？

**A：**经测试的确会被视为函数重载，不过define后，VS中CreateEvent函数名颜色不同，且转到定义时跳转到define导致我误以为存在问题





#### notify.h

Notif类是Event的派生类，利用linux pipe创建管道

##### 成员变量

int m_iPipeFD[2]：管道中readfd和writefd的集合

string m_sHost = ”Notify“

##### 成员函数

Init：使用pipe()创建管道，fd设置为非阻塞，并将Notify事件的readfd加入EventLoop的epoll读监听

SendNotify：与mom中自己实现的WakeUp类似，发送单字节唤醒epoll

OnRead：读出writefd的字节即可，无需其他处理，因为pipe仅用于唤醒



###### pipe



**Q：**实际项目中TCP和UDP的选择依据？

phxpaxos中如果消息过大，超出了udp发送范围或者选定为TCP方式，会自动选择TCP发送；因为UDP有报文发送大小限制？



### Src/node目录

#### node.h

定义了基类Node，是个抽象类，该类派生出子类PNode，虚函数覆盖paxos批量提交，为各组指定state machine，超时时间设定，同步写磁盘，处理接收到的消息等

静态函数 RunNode：paxos算法的开启处，完成新建并初始化pNode，指定Network的Node，并启动网络中的Tcp、Udp收发；完成后通过回参返回成功运行的pNode对象指针。

>接口类 -- 此方法值得借鉴，具体对应设计模式还需查询
>
>node整体封装完全是个抽象类，但是提供了一个静态函数并实现，内部会new pNode，并初始化和启动



#### pnode.h

Node的派生类，是proposer节点的实现，使用默认网络和存储

（此类中封装了大量paxos节点操作，具体实现需要详细看下对应类 ！以下仅为简单函数整理。都像流水账了）

##### 成员变量

vector< Group *> m_vecGroupList：该pnode所负责的所有paxos group
vector< MasterMgr *> m_vecMasterList：各个paxos group所对应的masterStateMachine

vector< ProposeBatch *> m_vecProposeBatch：各个paxos group所对应批量请求类
MultiDatabase m_oDefaultLogStorage：节点默认使用的存储类
DFNetWork m_oDefaultNetWork ：节点默认使用的network类
NotifierPool m_oNotifierPool：Notify的容器map
nodeid_t m_iMyNodeID：Nodeid由IP、port组成

##### 成员函数

（以下函数中，Master、StateMachine部分暂时理解不清，需要具体查看对应实现部分）

dtor：先停Master，之后按顺序停proposebatch、group、network；全部停止后，释放group、master、proposebatch

CheckOptions：配置检查函数，节点启动前检查诸如存储实现部分，UDP报文最大长度，group数量是否合理，节点IP，状态机Group索引是否合理；但对网络实现部分未作检查

InitNetWork：

​	回参：NetWork *& poNetWork

首先Options检查是否自定义了网络实现，如果是则直接返回，因为pNode实现默认使用DFNetWoek，而非自定义的网络类，所以自定义网络后，应该实现自定义的Node类型；如果未自定义网络，则默认初始化DFNetwork，参数poNetWork会返回实际运行的网络

InitLogStorage：

​	回参：LogStorage *& poLogStorage

需要在Options中指定存储路径，同时检查是否自定义存储实现，如果是默认存储实现，则初始化m_oDefaultLogStorage，参数poLogStorage返回实际运行的存储

InitStateMachine：添加状态机到指定的Group，此Group在oGroupSMInfo中被指定

RunMaster：运行Master

RunProposeBatch：运行ProposeBatch

Init：初始化节点，首先完成配置检查CheckOptions，从配置项中获取本节点ID，然后按序初始化存储，网络，根据配置中的group数量构建和初始化master、创建group、创建proposeBatch，并加入节点中对应的管理容器，初始化状态机，同时初始化group；待所有group初始化完毕后启动线程，并运行master和proposeBatch；此处选择了等group的初始化全部成功后才启动线程，否则若先启动线程然后发现group初始化有错误，再停止线程会浪费更多时间

Propose：此函数存在两个重载，区别在于有无状态机上下文；功能均为向集群写入一个值，内部调用了Commiter的NewValueGetID，返回状态机转移函数失败

GetNowInstanceID：获取paxos现在正在提议的实例Id，即最大的实例编号（实例编号全局唯一）

GetMinChosenInstanceID：暂未理解，查询其实际调用是，返回checkpoint的choseninstanceid

OnReceiveMessage：参数char * pcMessage中包含group索引信息，内部调用instance->onReceiveMessage，将消息放入ioloop的消息队列

AddStateMachine：存在两个重载，将状态机放入指定的group和放入所有group



checkpoint是状态机的镜像数据，保存着最大实例编号前的所有状态数据（即执行状态转移函数后的数据），故障发生时，只需要将checkpoint拷入，然后最大实例编号之后的log重跑即可；在实现时，是通过异步线程完成数据镜像的构造，当其他节点需要时，此镜像可以随意启停，出错后执行与原状态机相同的log序列即可恢复（而原状态机状态数据可能正在被写入或读取，随意启停会产生未知风险）



SetHoldPaxosLogCount：为pNode中所有group设置在checkpoint之前需要保留的log数量（目前理解）

PauseCheckpointReplayer：暂停pNode中所有group的镜像状态机的状态转移（即日志的执行）

ContinueCheckpointReplayer：恢复pNode中所有group的镜像状态机的状态转移



PausePaxosLogCleaner：暂停删除log；因为checkpoint产生后，此前的log即可删除，进而防止占用内存空间，但删除log本身会占用io资源，所以可以设置启停

ContinuePaxosLogCleaner：恢复删除log



###### 成员变更

*成员变动仍需跟进，暂时理解有问题*

paxos可通过将集群成员的变更视为状态，从而对集群成员变动本身达成共识，如某节点失联，可能仅仅是因为节点运行速度过慢，或者网络拥塞，而非节点宕机或进程被kill，但如果其他节点均认为该节点失联，即对该节点”失联“达成共识，则该节点即可是为失联或者说”被失联“。

在phxpaxos中为Membership，为完成此类状态转移，定义了一个新的状态机类型，即InsideSM类，此类为StateMachine类的子类，InsideSM类又有SystemVSM类作为派生类，成员变动分为：新增节点、节点退出、替换节点三类，每次发生成员变更版本号会更新

`int PNode :: ProposalMembership(SystemVSM * poSystemVSM, const int iGroupIdx, const NodeInfoList & vecNodeInfoList, const uint64_t llVersion)` ：发起成员变更提案，参数的vecNodeInfoList中为期望的成员列表，即变更后的成员。通过paxos达成成员变更共识，所以内部仍然调用propose，不过提出的value是通过SystemVSM 的Membership_OPValue函数获取，返回状态机上下文



`int PNode :: AddMember(const int iGroupIdx, const NodeInfo & oNode)` ：新增成员，发起加入节点提案前，检查group索引，并确保新加入的节点不与当前成员重合，最后发起变更提案



`int PNode :: RemoveMember(const int iGroupIdx, const NodeInfo & oNode)` ：移除成员

**Q：**在获取移除指定节点后的节点列表时，phxpaxos采用了新建列表，然后从原列表重新pushback，为何不直接使用find，然后erase？

**A：**因为`Options :: vecNodeInfoList`来自于配置项，在集群启动时所有节点会将此成员列表持久化在磁盘上，使用新的list减少写入io。而且在集群启动完成后，集群节点将不会在信任`vecNodeInfoList`（发生成员变更后，此列表何时被持久化）



`int PNode :: ChangeMember(const int iGroupIdx, const NodeInfo & oFromNode, const NodeInfo & oToNode)` ：替换成员，使用oToNode替换oFromNode，可能存在oToNode已经存在，且oFromNode已被移除，此时无需处理。同样存在和RemoveMember一样的问题？



`int PNode :: ShowMembership(const int iGroupIdx, NodeInfoList & vecNodeInfoList)` ：获取当前集群所有节点列表



###### 选主

*详细待增加*

paxos存在活锁问题，而活锁出现一般是因为多个proposer同时发起提案，所以选择集群中某个proposer节点为leader，且只允许leader发起提案；但是也附加了新的选主问题，和成员变更一样，选举主节点同样可以转换为状态，利用paxos本身达成共识。phxpaxos中为实现此状态转移，定义了MasterStateMachine类，此类同样是InsideSM的派生类

`const NodeInfo PNode :: GetMaster(const int iGroupIdx)` 

获取当前集群的master节点

`const NodeInfo PNode :: GetMasterWithVersion(const int iGroupIdx, uint64_t & llVersion) `获取当前集群的master及其版本号

`const bool PNode :: IsIMMaster(const int iGroupIdx)`判定当前节点是否为master（判定当前节点，不是指定节点）

`int PNode :: SetMasterLease(const int iGroupIdx, const int iLeaseTimeMs)`设置master租约

`int PNode :: DropMaster(const int iGroupIdx)`放弃当前节点为master



###### 过载保护API（官方文档解释）

*需要细读其他部分*

`void PNode :: SetMaxHoldThreads(const int iGroupIdx, const int iMaxHoldThreads)`设置Propose最多挂起的线程。对于每个Group的Propose在内部是通过线程锁来进行串行化的，通过设置这个值，使得在过载的情况下不会挂死太多线程

`void PNode :: SetProposeWaitTimeThresholdMS(const int iGroupIdx, const int iWaitTimeThresholdMS)`

XXXX



###### 批量Propose

*详情待写*

如果每次只提交一个提案，确定一个值，性能消耗较大，所以会积累propose请求，并放到等待队列中（多个请求的value合并为一个），当队列中请求到达阈值或者队列积压超过指定阈值，会发起合并为一个Propose请求提交，此时多个提案值共用一个实例号；而各个批量proposal会使用不同的BatchIndex标识，用以确定在状态机中的顺序，BatchIndex从0开始。

虽然是多个提案合并为一个请求，但值确定后仍然需要逐个提案执行状态机转移函数（参见SMFac中的BatchExecute），如果出现一个批量提交中后几个提案执行状态转移前节点故障，后续节点恢复后，未被状态转移的几个提案状态如何明确，具体代补充，phxpaxos暂不知有无此类设计。

`void PNode :: SetBatchCount(const int iGroupIdx, const int iBatchCount)`设置批量等待队列的最大值



`void PNode :: SetBatchDelayTimeMs(const int iGroupIdx, const int iBatchDelayTimeMs)`设置批量等待队列的请求积压上限时间



`int PNode :: BatchPropose(const int iGroupIdx, const std::string & sValue, uint64_t & llInstanceID, uint32_t & iBatchIndex, SMCtx * poSMCtx)`BatchPropose存在两个重载，另一个重载不包含状态机上下文，前三个参数与Propose一样



###### 其他

`void PNode :: SetLogSync(const int iGroupIdx, const bool bLogSync)`

XXXX

`int PNode :: GetInstanceValue(const int iGroupIdx, const uint64_t llInstanceID, std::vector<std::pair<std::string, int> > & vecValues)`获取指定实例号的提案值，因为有批量proposal的存在，而每一批proposal拥有相同的实例号，所以如果是批量proposal，回参vecValues会包含多个value值





### Src/sm-base目录

一个paxos运行过程中会有多个状态机，如基本的选主状态机、集群变更状态机和用户自定义的状态机（继承并实现StateMachine类），但是paxos运行时是通过实例类去调用状态机工厂，工厂再去调度指定ID的状态机执行；而且每个提案值对应一个状态机

Instance -> SMFac -> (StateMachine)MyStateMachine

![phxpaxos 状态机关系图](F:\Markdown\研一上\图片\phxpaxos 状态机关系图.png)



#### sm.h

定义了状态机的基类StateMachine类，是状态机类的抽象类，用户可以根据业务需求，继承此类，实现自己业务逻辑的状态机，如echo服务，状态转移函数只需要返回收到的值即可；同时状态转移函数被抽象到Execute函数中，继承子类中实现该函数即可。

phxpaxos的状态机中专门设置了类：状态机上下文SMCtx类和checkpoint文件信息类，后者保存了checkpoint的文件路径和文件大小

```C++
class SMCtx
{
  public:
  	SMCtx();
  	SMCtx(const int iSMID, void * pCtx);
  	
  	int m_iSMID; //状态机ID，状态机的唯一标识符
  	void * m_pCtx; 
};

class CheckpointFileInfo
{
  public:
  	string m_sFilePath; //文件路径
  	size_t m_llFileSize; //文件大小 
};
//暂时理解为一个checkpoint有多个文件
typedef vector<CheckpointFileInfo> CheckpointFileInfoList;
```

**Q：**什么时候用指针容器，什么时候用上面的对象容器。。。

是不是比较占内存的使用指针容器，不过要注意指针计数问题，防止野指针

##### 成员函数

`virtual const int SMID() const = 0`返回状态机ID，此ID唯一，所有状态机不会重复

`virtual bool Execute(const int iGroupIdx, const uint64_t llInstanceID, const std::string & sPaxosValue, SMCtx * poSMCtx) = 0`状态转移函数入口

`virtual bool ExecuteForCheckpoint(const int iGroupIdx, const uint64_t llInstanceID, const std::string & sPaxosValue)`执行状态机的checkpoint操作？？具体操作？开异步线程复制状态机执行后的状态？

`virtual const uint64_t GetCheckpointInstanceID(const int iGroupIdx) const`返回checkpoint所保存的最大的实例号

`virtual int GetCheckpointState(const int iGroupIdx, std::string & sDirPath, std::vector<std::string> & vecFileList) `获取指定路径下的checkpoint状态；参数分别为checkpoint所在根路径和相对路径

`virtual int LockCheckpointState()`锁定checkpoint状态，GetCheckpointState所返回的checkpoint列表将不能被新增，修改，删除

`virtual void UnLockCheckpointState()`checkpoint状态解锁

`virtual int LoadCheckpointState(const int iGroupIdx, const std::string & sCheckpointTmpFileDirPath, const std::vector<std::string> & vecFileList, const uint64_t llCheckpointInstanceID)`加载checkpoint的状态数据，sCheckpointTmpFileDirPath参数表明了checkpoint所在的目录，vecFileList参数指各个文件的绝对路径（一个checkpoint会有多个File）

` virtual void BeforePropose(const int iGroupIdx, std::string & sValue)`可以在状态转移执行前修改值；batch propose时如果要修改，也只需要调用一次此函数

我的理解是在状态转移执行前修改value，但函数名采用的是在提交前，而且提案值的修改本身不应该再次发起一次paxos吗？直接修改不会导致不一致吗？

` virtual const bool NeedCallBeforePropose()`是否需要在状态执行前修改，默认为false；



#### sm_base.h

定义了一个状态机工厂类SMFac，维护了一个StateMachine类型的容器，实例Instance对于状态机的执行将会通过工厂类调用，但工厂只负责对状态机的管理，分配调度，不会自己new状态机对象，对外仅提供了一个AddSM的接口



##### 成员变量

`vector< StateMachine *> m_vecSMList;`状态机管理容器，实例的状态机执行调用时，会从中寻找对应状态机去执行

`int m_iMyGroupIdx;`状态机工厂的paxos group ID，因为可以并行执行多个paxos group，各组之间的状态机是相互隔离的，只不过共用proposer、acceptor、learner等资源

##### 成员函数

`void SMFac :: PackPaxosValue(std::string & sPaxosValue, const int iSMID)`将状态机ID加入到sPaxosValue头部，如果ID为空，则填充为sizeof(int)个0



`bool Execute(const int iGroupIdx, const uint64_t llInstanceID, const std::string & sPaxosValue, SMCtx * poSMCtx)`状态转移函数入口，利用sPaxosValue生成状态机ID，再根据状态机ID判断是否为BatchPropose，批量提交时调用BatchExecute，否则调用DoExecute即可；

参数sPaxosValue截取4字节后的内容作为实际提案值

> string::data() 和 string::c_str()之间的区别
>
> Both `string::data` and `string::c_str` are synonyms and return the same value
>
> 两者没区别，返回值也相同，都是把string转为char 数组，并且补上最后结束符'\0'

**Q：**此处利用sPaxosValue生成的SMID判断是否为批量提交，参数sPaxosValue的前缀是不是有特殊定义，前面paxos的流程中一直没看到sPaxosValue的具体组成到底是什么

A：工厂的PackPaxosValue函数提供状态机ID和提案值的组合



`bool DoExecute(const int iGroupIdx, const uint64_t llInstanceID, const std::string & sBodyValue, const int iSMID, SMCtx * poSMCtx)`所有状态机转移函数的最终调用处，负责从工厂的状态机容器中检索出指定ID的状态机，然后执行StateMachine的实现类中的Execute函数



`bool SMFac :: BatchExecute(const int iGroupIdx, const uint64_t llInstanceID, const std::string & sBodyValue, BatchSMCtx * poBatchSMCtx)`拆分Batch Propose中的提案值，并提交DoExecute。所以Batch Propose时多个提案值合并达成共识，但在进行状态机执行下，仍然会拆分为单独的提案值，交由对应状态机执行

**Q：**此处对应的状态机是什么时候产生并加入到工厂？或者说AddSM何时被调用



相同的流程，ExecuteForCheckpoint同样会根据sPaxosValue区分批量和单独提案



`void SMFac :: AddSM(StateMachine * poSM)`向工厂状态机列表新增状态机

`vector<StateMachine *> SMFac :: GetSMList()`获取工厂状态机列表

`const uint64_t SMFac :: GetCheckpointInstanceID(const int iGroupIdx) const `获取checkpoint当前的最大实例号，因为checkpoint从逻辑上不允许被随意修改，所以设计为常量成员函数；优先返回用户状态机的最大实例号，只有当工厂内无用户状态机，才会返回内部状态机的最大实例号（成员变更/选主）



`void SMFac :: BeforePropose(const int iGroupIdx, std::string & sValue)`状态机执行前，修改提案值；最终会调用StateMachine实现类中的BeforePropose，另BatchPropse会筛除掉重复的状态机



>string::erase 如果参数为int，somestring.erase(idx)，则删除idx及以后的所有字符





#### inside_sm.h

内部状态机的抽象类InsideSM，该类继承自StateMachine类，而集群成员变更和选主切换的状态机均继承自该类；该类在原本StateMachine类的基础上，新增两函数：

##### 成员函数

`virtual int GetCheckpointBuffer(std::string & sCPBuffer) = 0`

`virtual int UpdateByCheckpoint(const std::string & sCPBuffer, bool & bChange) = 0`



#### system_v_sm.h

实现了SystemVSM类，作为InsideSM类的集群成员变更实现，对外接口定义在Node接口类

**Q：**集群成员配置信息存放在？SystemVariablesStore

重点：切换期间，谁对外提供服务？如何保证服务不中断，paxos不受影响；文件中提到的如果不使用内置的集群变更，可以通过新增节点（最新状态），然后逐台重启节点并更新状态，那么此期间将会存在新旧两种配置，谁对外提供服务？谁到达半数以上谁就提供服务？那么半数的判定由谁来做？怎么保证半数以上时，新加入的节点已经追赶上了？



##### 成员变量

`int m_iMyGroupIdx` 该集群变更对应与哪个paxosgroup 

`SystemVariables m_oSystemVariables`集群配置变量，如组id，版本信息
SystemVariablesStore m_oSystemVStore
set< nodeid_t> m_setNodeID
nodeid_t m_iMyNodeID
MembershipChangeCallback m_pMembershipChangeCallback



##### 成员函数

