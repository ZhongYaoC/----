paxos工程化中既然提出了master以及租约概念，那么也就需要避免raft、zab等共识中的问题：选举、防脑裂等





phxpaxos中每个状态机对应一个paxos group，如果有多个业务可以使用多个状态机，从而对应多个paxos group，所以需要有group id标识组别，但是每个paxos group内部的实例是串行执行的，防止空洞产生，每个实例直接互不影响，上一个实例写入状态机并完成状态转移才可以开始下一个实例；不过从宏观上看，多个group的实例是并行执行。



状态转移：每个实例所确定（chosen）的值只是状态转移的输入值；如果是kv存储状态机，也就是说每个实例确定的是数据库的某项操作（如：set(x, 2)），状态转移负责将此操作落入数据库中；而如果是echo服务，实例确定的值是收到的某个echo值，状态转移就是将此值再次传回就可



#### 两系统state值定义：

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

B. proposal超时处理（**关注超时处理2**）

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

当Proposer检查accept reply获知某个提案被选中后，通过BroadcastMessage_Type_RunSelf_Final消息通知所有节点，其中包含自身节点；如果消息中的提案号和节点保存的accepted 提案相同，则learner更新状态信息（因为此提案值和acceptor的相同，所以不需要再次落盘，更新内存状态即可）。另如果存在follower节点，则将数据同步至follower（follower节点不参与共识，类似于热备节点）

处理函数为OnProposerSendSuccess

##### 提案值追赶：

当节点的实例号落后，则无法参与到当前阶段的共识提案，所以需要由learner主动追赶欠缺的实例，learner将当前节点的实例号、节点号打包，通过MsgType_PaxosLearner_AskforLearn消息发送给除自身外其他节点，



**LearnerSender实现中涉及的锁的应用**：

`WaitToSend`







ioloop实现机制：

![phxpaxos ioloop机制](F:\Markdown\研一上\图片\phxpaxos ioloop机制.png)



phxpaxos采用消息队列机制，而非select下直接处理，即所有的paxos阶段产生的消息统一放入主线程（与select是否同线程呢）的共享消息队列（具体需要查看网络处理--select），ioloop的循环中通过阻塞+超时时间机制处理每个消息，但每轮循环只处理一个pasxosMsg会不会太低效；而且因为循环内还有`newvalue`，感觉只要值未被确定，就会有大量的重传消息。

~~如果消息的两阶段在本轮循环内未完成，就会重传消息（重跑对应流程），而且如果收到过任一拒绝回复（即使不是大多数拒绝），提案号就会提高；~~

**Detail**：系统启动时，设置初始超时时长为1000ms，则初始化时前面所有流程“空跑”，`newvalue`开始paxos算法，内部会新增定时器，如prepare超时定时器，则在后续循环中，根据定时器的实例id和对应消息类型处理超时错误，并且处理定时器函数本身会返回下一个最近的超时时间点，此时间用于消息队列的阻塞时长，阻塞时长内收到对应消息则会相应移除定时器；超时未收到则触发会在下个循环处理超时（下个循环处理超时，相比于select会不会影响效率，额似乎没啥影响）；为定位不同的超时处理机制，所以每个定时器会用实例id和消息类型标记

> 定时器为何使用vector，既然每次pushback新的定时器，然后再push_heap，类似维护为堆，何不直接用set或heap，定制好排序函数





