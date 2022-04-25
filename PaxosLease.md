#### PaxosLease

使用paxos本身实现选主功能，即将value部分定义为主节点leader id + 租期Time，根据paxos的特性，最后会唯一的确定下来一个value，即唯一的leader和对应租期，而这也是lamport在paxos make simple论文中所提到的，论文中还提出成员变更也可采取此方法实现。



##### Question：

依据此想法，如果一切正常没有过大的网络延迟现象出现，那么acceptor回复给proposer的消息中应该是空value，这样最早参与选主的proposer可以放上自身的提案（即选择自己为leader，并指定租期），然后顺利的当选。

但是，如果acceptor回复给proposer的不是空value，如：5个节点，其中proposer 1发出prepare，被acceptor 2，3 回复后，正常进入第二阶段，但是在proposer 1 发出accept消息到达acceptor 3之前，有另外的proposer 5发出的prepare给acceptor 3，4使之亦进入第二阶段，并且accept消息先行到达了acceptor 3，但是proposer 5出现故障，迟迟未能进入learn节点；于是proposer 1 发起新的提案，此时网络顺畅，最终达成了一致，此时的value实际上是proposer 5提出的value，但却由proposer 1实现。

那么proposer 1会错误的认为自己是leader（论文中的`leasrOwner = true`），但是value中的内容却表明leader是节点5，而且节点5自身都不知道自己已经成为了leader。

所以在PaxosLease中需要避免出现acceptor已经接受过提案的情况发生，即acceptor需要清空自己之前accept过的提案值。（多久清空？？？或者说什么时间点清空？？）

不过清空自己accept过的提案值会不会与正常的选举之间冲突呢？？或者说正常的选举时accepted的提案值何时清空

> 源码中acceptor会清空上一次已过期的提案值，而针对上述例子，采取在发起2a时检测value中的id和本机是否一致







##### KeySpace PaxosLease源码分析：

###### PLeaseState.h

```C++
class PLeaseProposerState
{
public:
	bool      preparing;
	bool      proposing;
	uint64_t  proposalID;	//提案号，单调递增，各节点间互不相同
	uint64_t  higestReceivedProposalID;//曾接受过的最大的提案号，收到多个有value的accept回复后，选择此值最大的对应的value
	unsigned  leaseOwner;//leader id 即本机id
	unsigned  duration;//leader租期（以上两行共同构成value）
	uint64_t  expireTime;//过期时间点
  	//相比于一般paxos，state中删去了accepted的最大的提案号，此值用来选择下一个提案号；似乎此值被设置为proposer的本地变量
};
```



```C++
class PLeaseAcceptorState
{
public:
  	uint64_t promisedProposalID;//承诺过的最大的提案号
  	bool     accepted;
  	uint64_t acceptedProposalID;//accept过的提案号
  	unsigned acceptedLeaseOwner;//accept过的leader
  	unsigned acceptedDuration;//accept过的租期（以上两行共同构成value）
  	uint64_t acceptedExpireTime;//accept过的过期时间点
};
```



```C++
class PLeaseLearnerState
{
  public:
  	bool  	 learned;
  	unsigned leaseOwner;//是否为leader
  	uint64_t expireTime;//过期时间点
  	uint64_t leaseEpoch;//租约任期
};
```





**每个节点的租期是单独计算的，不是全局时间，各自从确认accept value那一刻开始计算过期时间**

**各角色之间，Learner会通过onlearnchosen复制proposer的过期时间点，从而两者一致；acceptor会根据accept时的剩余租期来计算自身的过期时间点**



###### paxos流程：

1）相比于一般的paxos流程，acceptor阶段新增了状态的清空（在第一和第二阶段中均检测）：针对那些已经accept过value，并且租期已经过期的情况 ；也就是acceptor处于上一已过期租期，此时清空状态以防止影响下一次的选举

**为防止上述例子中的情况产生，proposer会在进入第二阶段初检查，如果value中的leaderOwner不是本机（即 不是此value的提出方），直接终止**，但这样的话，第二阶段为什么需要再次检测

（能不能提前，直接在1b阶段检查终止掉）

update：第一阶段时的检测应该是为了以上理由，第二阶段的检测难道是为了应对原leader续租



2）acceptor在启动后会Sleep M秒，论文中解释为防止破坏租约不等式（任何时间点都不会有多于一个proposer持有租约）



###### 计时器：

Proposer：

`CdownTimer acquireLeaseTimeout;`

获取租约可用的时间间隔；在StartPrepare时开始计时，进入learn阶段时移除；如果此时间范围内没能获取到租约，则执行超时回调`onAcquireLeaseTimeout` 重新开始Prepare



`Timer extendLeaseTimeout;`

扩展的租约时间间隔，时长为五分之一的租期`Now() + (state.expireTime - Now()) / 5`，在开始learn阶段开始计时，一旦proposer进入新一轮的prepare阶段此计时器马上移除，超出则执行回调`onExtendLeaseTimeout`  重新开始Prepare，但需要确保此时proposer不是prepare和proposing阶段，即确保超时前的proposer已经进入了learn阶段

> extendLeaseTimeout的计时器移除似乎只发生在StartAcquireLease和StopAcquireLease时；
>
> 所以应该是用于续租，即租期达成后，再经过1/5租期即发起下一轮的续租





Acceptor：

`Timer leaseTimeout;`

因为PaxosLease没有全局的时间概念，各自维护自身的租期，所以此值为acceptor各自的租约计时器，在accept提案值时便确定下来，loop中开始计时；正常会持续到本次租期结束，但如果提前遇到状态清空，会从loop中移除此计时器，同时提前执行超时处理；超时处理为`onLeaseTimeout`清空acceptor的状态



Learner：

`Timer leaseTimeout;`

~~在当前时间基础上加上本次租期在发起学习阶段的剩余时长~~（learner的租期也是单独计算，收到learn的消息开始算起）；超时处理为`OnLeaseTimeout`清空learner的状态，“租期版本”lease epoch递增，移除计时器，并针对acquireLease == true的节点重新开始申请租期

```C++
//重新开始获取新一轮租期
if (acquireLease)
  proposer.StartAcquireLease();
```





learner中对value中leader非本机和leader本机的超时时间采取不同处理，leader本机直接采取和propoer同样的过期时间点，非leader的learner采取当前时间加上本次leader租期在发起学习阶段的剩余时长（再预留500ms）



learner也存在清空状态需求，以应对上一租期的结束



![paxosLease Learner OnLearnChosen回调](F:\Markdown\研一上\图片\paxosLease Learner OnLearnChosen回调.png)



![paxosLease Learner超时回调](F:\Markdown\研一上\图片\paxosLease Learner超时回调.png)











综上：paxos lease采取局部时间，各自维护自身的租期，到期后acceptor和learner各自处理超时：清空状态





###### 不同实例分界 instance （多次租期的衔接）

PaxosLease的执行起始点在ReplicatedLog中，Log初始化时会初始化Lease类，并注册回调，开始尝试获取leader租期，现假设节点 1 率先发起选举`PAxosLease::AcquireLease`，以求获取租约；

```C++
masterLease.Init(useSoftClock);
masterLease.SetOnLearnLease(&onLearnLease);
masterLease.SetOnLeaseTimeout(&onLeaseTimeout);
//此处被注释了，反倒有些疑问，因为无法看出是谁发起“获取租期”
//如果是所有节点启动后均发起，不考虑活锁问题吗？就算没有活锁，租期的最终获得可能会被延迟，可能有大量冲突，回退
//if (RCONF->GetNodeID() == 0) // TODO: FOR DEBUGGING
	masterLease.AcquireLease();
```



**1a.** 节点1开始prepare阶段，组装msg，类型为`PLEASE_PREPARE_REQUEST`，内部包含提案号、节点id、paxos ID；

**1b.**接收方收到后`PaxosLease::OnRead()`会更新proposer收到过的最大提案号，此值用于下次提案中选择下一个提案号（但正常paxos流程中，此值被纳入state管理，而非直接作为proposer的成员变量），并由acceptor处理prepare消息：**（先检查paxos id，再检查提案号；防止一个宕机节点恢复后，主动propose，然后强行变为新主）**

1）在`OnPrepareRequest`中比较发出方的paxos ID与本机paxos ID，小于则直接return（不回复，那么发出方的追赶由谁负责？有没有可能一直小下去），说明此节点的数据不够新，不足以成为leader；而如果发出方的paxos ID大于本机learner的paxos ID，说明本机需要追赶，则会借由一般paxos流程来发起追赶，并对追赶数据设置对应计时器，要在一定时间范围内完成追赶	`learner.RequestChosen(nodeID)`

> ```C++
> //会不会和正常paxos流程中的追赶重复？？
> RLOG->OnPaxosLeaseMsg(msg.paxosID, msg.nodeID);
>
> //如 租期生效期间，leader再次发出新的租期，而此时leader的paxosID刚好大于acceptor的paxosID，那么触发acceptor所在节点的learner追赶，而此paxosID也许在正常paxos流程中已正在被追赶，会不会重复
> //此问题 在learner追赶接收时校验应该就可以
> ```

2）判断acceptor是否accepted其他value，**并且**已接受过的租期已过期（即acceptor的上一任期是否结束），如果已结束，则清空acceptor的所有state，并将上一轮的租期计时器移除

```C++
if (state.accepted && state.acceptedExpireTime < Now())
{
  //移除计时器
  EventLoop::Remove(&leaseTimeout);
  //清空状态和过期时间等，value也恢复为空
  OnLeaseTimeout();
}
```

> 此状态会出现吗？如果上一任期结束，那么计时器中的超时回调会自动清空acceptor的state，不需要等到手动执行的这一步吧。
>
> 除非因loop中先处理读写回调，读取到此request报文时，上一租期恰好刚刚超时，但还未来及处理，所以需要手动重置，并将计时器移除（在类中设置过期时间expiredTime，然后由手动处置，此思路似乎可以用在我自己的prevote中！我现在的心跳包间隔时间和单包超时时间定的太小，两个总会近乎同时发生，收到心跳包并重置超时计时器，但下个loop中单包超时仍然开始处理，这有问题）
>
> keyspace中Timer的active成员和我们用的Timer的unused成员类似，当在loop中监听时即为active = true / unused = false，移除时即为active = false / unused = true

3）之后流程与一般paxos一致，不过state不需要持久化：提案号是否大于acceptor所承诺过的最大的提案号，如果小于本机promisedProposalID直接发出拒绝报文，否则根据自身的accepted情况，填写回复消息中的value值；报文类型分别对应``PLEASE_PREPARE_REJECTED`、`PLEASE_PREPARE_CURRENTLY_OPEN`、`PLEASE_PREPARE_PREVIOUSLY_ACCEPTED`

 

**2a.** proposer收到acceptor的回应后，处理回复的消息`OnPrepareResponse`，如果acceptor已经有accepted的value，则修改自身state的value的leaderOwner（但没有修改value中的duration？？）；如果拒绝超出半数以上，直接重新发起新的提案，不再继续等待；如果接受超出半数以上，开启第二阶段



因为1b中有可能写入state的value，所以检查state的value中leaderOwner是否为本机，如果不是则直接返回，因为不应该出现leader的实际选中者和提案发起方不一致情况，此时leader的实际选中者自身都不知道自己属于leader，此时会等到acquireLeaseTimeout超时，然后重新进入1a阶段。

正常时，proposer填入自身的提案值，duration即为`MAX_LEASE_TIME`，计算本机租期过期时间，然后组装msg，发送给acceptor，msg内容包含类型`PLEASE_PROPOSE_REQUEST`，提案号，value（leaderOwner + duration）



**2b.** acceptor在`OnProposeRequest`中处理收到的accept消息，同样再次判断acceptor是否accepted其他value，**并且**已接受过的租期已过期（即acceptor的上一任期是否结束），如果已结束，则清空acceptor的所有state，并将上一轮的租期计时器移除

> 此处为何再次判断？？还是有疑问，按理来说，在2a时期已经检查过state中leaderOwner和本机不一致的情况
>
> 针对续租情况？或者说继续收到同一leader发出的新的租约请求

再次比较提案号（防止收到很久以前的proposer prepare报文），小于则拒绝并直接回复；提案号大于等于则设置租期、leaderOwner，并且开始计算acceptor本身的租期，重置计时器，并发送回复给proposer，msg类型分别为`PLEASE_PROPOSE_REJECTED`、`PLEASE_PROPOSE_ACCEPTED`



proposer会在`OnProposeResponse`中处理此回复消息，同样需要累计接受类型的消息至半数以上，并且需要确保本次租期的剩余时间充足，可以完成选择值阶段（源码中选择500ms）。在发送学习值消息后，将获取租约计时器移除，代表租约已经获取到，但同时设置extendLeaseTimeout为本次租期的1/5。学习值消息的类型为`PLEASE_LEARN_CHOSEN`，本次租期的剩余时间作为learner的duration，同时传输leader自身的过期时间。当然如果未能收到半数以上的接受类型消息，代表本次租约未能达成，重新开启新的提案。



**学习值.** 节点收到`PLEASE_LEARN_CHOSEN`消息后，交由learner处理`OnLearnChosen`，先行检查本learner是否已经学习过value**并且**已过期（即上一租期的value），如果已结束，则清空learner的所有state，并移除上一轮的租约计时器。

如果learner和leader是同节点，则过期时间相同；否则过期时间设置为剩余租期 - 500ms，然后重置租期计时器，调用onLearnLeaseCallback（所有非leader节点，停止获取租期）；

同时当learner的租期到期时，超时处理是所有之前申请过租期的节点重新发起申请，并使leaseepoch ++；相当于新的选举开始，但是各个租期之间如何区分呢？有没有必要区分，似乎没必要

如果leader宕机，那么**新租期的发起是由learner负责**的，learner的租期超时处理中会启动`StartAcquireLesase`

如果集群正常运行，那么续租将由原leader的proposer负责发起



> 正常paxos的实例切换时，会调用`OnNewPaxosRound`
>
> ```C++
> if (acquireLeaseTimeout.IsActive() && RLOG->IsMaster())
> 		StartPreparing();
> ```
>
> 如果此时LeaseProposer还没有获取到租期或者还在进行选主中，并且log中显示其曾为leader，则直接发起prepare；如果LeaseProposer已经获取到租期，则无视；也就是说实例变动不一定会导致租期跟随变动
>
> 而针对那些未参与获取租期的节点而言，即使切换实例，本节点也不会发起新的获取租期（选举）







正常paxos流程下的学习值阶段：

```C++
//learn也会分为两阶段：PAXOS_LEARN_PROPOSAL + PAXOS_LEARN_VALUE
void ReplicatedLog::OnLearnChosen()
{
	Log_Trace();

	uint64_t paxosID;
	bool	ownAppend, clientAppend;

	//有没有可能，pmsg.paxosID < learner.paxosID

	//此情况有两种可能，正在参与追赶和正常paxos流程
	//正常paxos流程中msg type 为先 PAXOS_LEARN_PROPOSAL 后 PAXOS_LEARN_VALUE
	if (pmsg.paxosID == learner.paxosID)
	{
		if (catchupTimeout.IsActive())
			EventLoop::Remove(&catchupTimeout);

		//此处为参与正常paxos流程，先判断proposal，然后马上修改msg中的type为value
		//此处似乎没必要有个PAXOS_LEARN_PROPOSAL类型，因为追赶和正常的paxos流程之间，在收到learn 时，通过acceptor的accepted等可以判断
		if (pmsg.type == PAXOS_LEARN_PROPOSAL && acceptor.state.accepted &&
			acceptor.state.acceptedProposalID == pmsg.proposalID)
				pmsg.LearnValue(pmsg.paxosID,
				GetNodeID(),
				acceptor.state.acceptedValue);
		if (pmsg.type == PAXOS_LEARN_VALUE)
			learner.OnLearnChosen(pmsg);
		//针对acceptor accepted = falsed??
		else
		{
			learner.RequestChosen(pmsg.nodeID);
			return;
		}
		
		// save it in the logCache (includes the epoch info)
		//实例 + value
		logCache.Push(learner.paxosID, learner.state.value);
		
		rmsg.Read(learner.state.value);	// rmsg.value holds the 'user value'
		paxosID = learner.paxosID;		// this is the value we
										// pass to the ReplicatedDB
		
		//开启下一个实例，实际仅为清除proposer、acceptor、learner中的状态信息；同时如果处于无主状态且也不再acquire lease，发起下一轮lease
		NewPaxosRound();
		// increments paxosID, clears proposer, acceptor, learner
		
		//如果还需要继续追赶（因为之前的pmsg.paxosID == learner.paxosID只是说明这个实例追上了）
		if (highestPaxosID > paxosID)
		{
			learner.RequestChosen(pmsg.nodeID);
			return;
		}

		Log_Trace("%d %d %" PRIu64 " %" PRIu64 "",
			rmsg.nodeID, GetNodeID(),
			rmsg.restartCounter, RCONF->GetRestartCounter());
		
		if (rmsg.nodeID == GetNodeID() &&
			rmsg.restartCounter == RCONF->GetRestartCounter())
		{
			//已经通过追赶得到了此value
			logQueue.Pop(); // we just appended this
		}

		//确认有无leader变更，如果leader变了，则不执行；否则继续
		if (rmsg.nodeID == GetNodeID() &&
			rmsg.restartCounter == RCONF->GetRestartCounter() &&
			rmsg.leaseEpoch == masterLease.GetLeaseEpoch() &&
			masterLease.IsLeaseOwner())
		{
			proposer.state.leader = true;
			Log_Trace("Multi paxos enabled");
		}
		else
		{
			proposer.state.leader = false;
			Log_Trace("Multi paxos disabled");
		}
		
		if (rmsg.nodeID == GetNodeID() &&
		rmsg.restartCounter == RCONF->GetRestartCounter())
		{
			ownAppend = true;
		}
		else
		{
			ownAppend = false;
		}
		
		// commit chaining
		if (ownAppend && rmsg.value == BS_MSG_NOP)
		{
			safeDB = true;
			if (replicatedDB != NULL && IsMaster())
				replicatedDB->OnMasterLease(masterLease.IsLeaseOwner());
		}
		else if (replicatedDB != NULL &&
				 rmsg.value.length > 0 &&
				 !(rmsg.value == BS_MSG_NOP))
		{
			if (!acceptor.transaction.IsActive())
			{
				Log_Trace("starting new transaction");
				acceptor.transaction.Begin();
			}
			clientAppend = ownAppend &&
						   rmsg.leaseEpoch == masterLease.GetLeaseEpoch() &&
						   IsMaster();
			
			//OnAppend中会stop paxos
			replicatedDB->OnAppend(&acceptor.transaction,
				paxosID,
				rmsg.value,
				clientAppend);
			// client will stop Paxos here
		}
		
		if (!proposer.IsActive() && logQueue.Length() > 0)
		{
			//从log中拿出下一个要达成一致的value，发起下一个实例的proposal
			if (!rmsg.Init(RCONF->GetNodeID(), RCONF->GetRestartCounter(),
						   masterLease.GetLeaseEpoch(), *(logQueue.Next())))
				ASSERT_FAIL();
			
			if (!rmsg.Write(value))
				ASSERT_FAIL();
			
			proposer.Propose(value);
		}
	}
	//新加入节点之前的paxosID还没有补完，其他正常节点的新一轮的learn已经到达了；相当于新加入节点会在prepare、propose、learn三个阶段request chosen
	else if (pmsg.paxosID > learner.paxosID)
	{
		//	I am lagging and need to catch-up
		
		if (!catchupTimeout.IsActive())
			EventLoop::Add(&catchupTimeout);
		
		learner.RequestChosen(pmsg.nodeID);
	}
}
```







//这是正常的paxos流程，非租约流程--追赶catch up

paxosID不一致，则`OnRequest()`具体分析：如果msg.paxos ID < acceptor.paxos ID说明msg的发出节点日志落后，需要追赶，则有领先节点的learner告知需要追赶的值`learner.SendChosen(pmsg.nodeID, pmsg.paxosID, value_)`；反正如果msg.paxos ID > acceptor.paxos ID，说明本节点落后，需要追赶，则向loop中新增catch up定时器，并请求从msg的发出节点处获取需要追赶的value`learner.RequestChosen(pmsg.nodeID)`

（未完）

 







20220413

PaxosLease中确定下来的value（leader、expiretime）要不要存放在DB库中，从keyspace的源码中，其Lease确定下来后，会调用回调函数`onLearnLeaseCallback`，此回调的最终调用是`ReplicatedLog::OnLearnLease`它会检查是否为leader，proposer需要不在执行中，并且`safeDB == false && rmsg.value == BS_MSG_NOP`，然后触发Paxos流程（`Append(nop)`），写入一个NOP值进入DB

因为程序启动后的Init中包含获取租期`StartAcquireLease()`，也就是说在进入Paxos流程前，已经拥有了leader，且此事件会被作为一个nop写入DB，以保证其DB安全？？；如果不写入NOP值，那么RLog Init后，如果直接收到客户端的Get请求，此时数据库中无任何数据，会出错？？

什么叫safeDB？





Paxos流程和PaxosLease之间的衔接：

1）正常情况

服务器启动时，选出leader，然后paxos正常运行，在ReplicatedLog::OnLearnChosen中，清空paxos的proposer、acceptor、learner的state后，会检查PaxosLease是否正常运行，是否需要更换leader

2）leader宕机

运行一段时间后，leader宕机，PaxosLease中检测到续租定时器过期，然后重新选主，而此时Paxos流程也被迫暂停，因为原leader（proposer）无法回应或者发起下一流程；所以需要有PaxosLease反向使Paxos流程在新leader上重启，故选出新的leader并完成learn阶段后，需要由新leader回调重新开始propose

2-1）

如果在2）情况下，原leader宕机时，已经发出了learn请求，并且本机已经执行完毕，即本机的paxosID++，全部状态清空，甚至已经append DB；

此时选出新leader后，原leader恢复，并在回调中重新发起propose，但他会检测propose的当前提案是否是有意义的值（即非nop），如果是，则会将本轮的提案改为`NOP`，而由于原leader的paxosID大于此次提案的paxosID，会发出追赶数据，**但是**如果该追赶数据在prepare和propose两阶段中均丢失，并顺利进入learn阶段，那会导致新旧leader中数据不一致？不会，因为其他存活的acceptor在原leader期间，accetpor的state中持久化保存着原先的value值，会回复`previously accepted`，所以不会导致数据不一致

2-2）

如果在2）情况下，原leader宕机时，已经发出了learn请求，并且本机已经执行完毕，即本机的paxosID++，全部状态清空，甚至已经append DB；

此时选出新leader，原leader一直未恢复，那么现在仍存活的节点间将继续推进Paxos流程，但此paxosID的value将会是nop，那么相当于新旧leader之间数据不一致？？？















PaxosLease中涉及catch up的部分

选主中同时完成正常Paxos的数据catch up，catch up并不会影响PaxosLease的正常运行

只针对接收方的paxosID小于发出方的情况，且只在prepare的response中catch up























##### 错误排查：

1、timer触发异常：在StartPreparing中调起BroadcastMessage时的耗时正常，约2-3ms，但是之后timer会等待3s（即kAcquireLeaseTimeout），导致触发获取租期超时，然后重新开始prepare

在main线程中运行StartAcquireLease()后，会发送TCP广播（各节点逐个循环发送），而广播的逻辑并不是马上从fd发出msg，而是先加入消息队列`thrdMsgBuff_`，然后利用pipe机制唤醒loop，从而在下次的Loop中利用`ProcessMsg`，处理消息队列中的请求（将msg加入到connection的发送队列，并enable write），再更下一次的Loop中实际发送。

也就是说msg的发出，一切起源于最初Loop的唤醒，否则loop中无法处理消息队列，也就无法发送消息，问题恰恰出在唤醒中。原本唤醒wakeup的调用逻辑是，如果判定不是同一线程或者正在处理队列中消息，则利用pipe机制唤醒loop，从而使得loop得以处理消息队列中的请求。

但是判定条件：正在处理队列中消息`isDoingProcMsg_`，是错误的，如果正在处理消息队列，其实不需要唤醒loop，因为正在处理的队列中的消息会自行enable write/read，无需唤醒；而如果队列中没有正在处理的消息，说明当前消息队列闲置，在加入新的消息队列请求时，才是需要触发唤醒loop的。

所以当需要广播的消息加入消息队列后，因为消息队列没有正在处理的消息，Loop一直未被唤醒，阻塞在Poll中等待最近的超时时间到达；Loop直到最近的超时时间到，才进入超时处理逻辑（重新开始prepare），同时在之后的ProcessMsg中处理了最初需要广播的消息。

故修改为：

```C++
// if((!IsInLoopThread() || isDoingProcMsg_) &&(ret>0))  //错误
if((!IsInLoopThread() || !isDoingProcMsg_) &&(ret>0))
{
	//唤醒wakeupEventHandler_
	//使用wakeup唤醒后,保证对同一个epollFD的操作是在一个线程中进行的。
	WakeUp();
}
```







> keyspace的回调形式很有特点，不是分为static的两段式回调，使用了模板。需要深入看下