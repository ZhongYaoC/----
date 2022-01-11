### Mom学习理解

mom初衷：

原本各client到各server直接可能都存在通信需求，但是如果cs之间点对点直连，client和server过多会导致通信链路杂乱，无法管理，所以引入消息中间件，对各个cs之间的通信需求统一管理。



数据和控制做了拆分，udp通信的接、发做了拆分；

客户端和mom服务器之间使用阻塞通信，各mom服务器之间使用非阻塞通信（设计原因，如果该另外的出现的问题）

进程列表是个全局列表

二级缓存的位置？？





#### ~~整体思路整理~~

~~（发给实际需要数据的主机，但由mom负责实际发送 -- 这就是消息中间件干的活；mom是个集群，每个mom维护一个与他连接的进程表，只负责给和他连接的进程发送数据，如果是其他mom负责，转发出去）~~

~~1、client想发送数据出去，需要先发个控制报，等待mom回应response才可以~~

~~1）控制报~~

~~首次发送给mom时，告知mom自身的进程信息（CLIENT_PROC_TYPE_INFO）~~

~~之后，发送给mom请求发送数据的控制报（CLIENT_SEND_REUEST_INFO）~~

> ~~控制报回复ok的条件~~

2）数据报

> 本mom主机处理，还是需要转给其他mom主机（如果需要转给其他mom，为什么不选择在控制报的回复中，要求客户端去找对应的mom）



2、client的接收数据

3、mom接收数据，发送数据

4、mom之间的通信

心跳报接收



心跳报组装后发送





#### map

```C++
//用于mom主机间的保活测试（同样是与该主机udp连接的mom主机列表，但用于定时保活测试）
map<string, STcpConnect> m_mapTcpTestConnect;
//与该主机udp连接的mom主机列表
map<string, STcpConnect> m_mapTcpConnect;

//全局进程表，每个mom主机上分别维护着哪些进程
map<string, ListNode> m_mapHostProc;

//与本mom主机相连的客户端流字节（通过它可以获得进程信息）
map<CFdOBJ*, tagMomStream*> m_mapLocalStream;

//与本mom主机相连的客户端控制报，CFdOBJ是本机与客户端相连的fd
map<CFdOBJ*, tagMomStream*> m_mapLocalCtrl;

//需要转发给其他mom服务器的数据
map<CFdOBJ*, tagMomCache*> m_mapMomCache;
```





#### HeartRead

读取心跳包（udp）信息，更新进程表

1、由于udp报文可能来自任何知晓该主机ip及port的服务器，所以首先验证该报文来源是否正确，在```map<string, STcpConnect> m_mapTcpTestConnect```（类似于一个互相udp通信的主机列表）中查找该报文的ip是否存在；

2、明确udp报文来源正确后，在进程列表```map<string, ListNode> m_mapHostProc```内查找有无该来源ip下进程信息，如果没有该来源ip（可能是该主机第一次心跳信息），就新建；有就清除旧的ListNode，以当前心跳信息中的进程信息为准



##### 疑问：

HeartUdpBag的组织形式--定义在```mom_common.h ```

```C++
struct HeartUdpBag
{
    int procCount;//进程数量
    unsigned short szProc[MAX_PROGRESS_NUM];//进程数组，因为此处用端口来唯一表示进程，即端口数组?
};
```





#### GetLocalLiveProc

通过各客户与本机连接的流字节```map<CFdOBJ*, tagMomStream*> m_mapLocalStream```，获取本机进程表，并组装成心跳包，广播发送出去

1、通过```m_mapLocalStream```遍历所有客户的流字节，其中的port即为进程信息，装进心跳包中```szProc[MAX_PROGRESS_NUM]```；接着更新本机进程表（用心跳包里的szProc[]把```ListNode```直接覆盖为最新的进程信息）;

2、最后，广播发送（注意心跳包的长度用实际心跳包长度，不要无脑sizeof(HeartUdpBag)，因为心跳包的szProc可能没满）



------



#### CtrlRead -- CFdOBJ m_svrLocaCtrl

读取客户端发送来的控制报信息，报文的接收是在threadframe中统一接收

报文结构：

gHead | LENGTH_INFO_COUNT | type | 数据 | gTail



1、如果报为空，说明客户端已经关闭，删除```m_mapLocalCtrl```中的对应的fd;

2、```m_mapLocalCtrl```中找到对应fd，将收到的数据加入tagMomStream的stream中；

3、GetOnePacket拿到去掉导引头、尾后的数据；验证这段数据的类型为INFO_CTRL_TYPE，对数据部分再做进一步处理 -- ```CtrlInfoDeal```(循环处理？？？)



#### CtrlInfoDeal

控制报数据部分的实际处理

数据部分结构：

tagCtrlHead(数据的type + len) + tagProcType(proc_type)

或

tagCtrlHead(数据的type + len) + tagSendRequest(destIp+destProc+count)



数据报数据部分：

tagDataHead(源ip、port、目的ip、port、长度、发送时间) + 数据



1、如果是CLIENT_PROC_TYPE_INFO（即客户端与mom首次连通时，告知自身进程信息），对proc_type部分做进一步处理-- ```CtrlDealCtrlProcTypeInfo```，加到```m_mapLocalCtrl```中，此报无需回复

2、如果是CLIENT_SEND_REQUEST_INFO（即客户端告知mom，请求发送消息，询问进程是否存在，对此类型mom需要给出回复），对tagSendRequest部分做进一步处理 -- ```IsPeerProcLive``` 检查目标进程，进而修改回复信息；最后根据进程存在情况，生成回复的报文 ```GenCtrlPack()```，并发送给客户端





#### CtrlDealCtrlProcTypeInfo

把控制报的proc_type存储到```m_mapLocalCtrl```（怎么感觉这个地方的fd就一个，m_srvLocalCtrl）

在m_mapLocalCtrl中找到fd，没有就新建，然后修改对应的进程类型proc_type





#### ISPeerProcLive

对控制报的tagSendRequest处理，检查目标进程是否存在

在进程表中检索其目标ip，及此ip下是否存在客户端想访问的目标进程



#### GenCtrlPack

利用CStreamPack将报文组装完毕；

报文格式和收到的一样，不过数据部分是回复包结构```tagAnswReponse```

报文结构：

gHead | LENGTH_INFO_COUNT | type | 数据 | gTail

------



#### LocalRead -- CFdOBJ m_hostSvrlocal

获取本机mom连接的客户端发送过来的数据报（不再是控制报），目的是找到客户端的目标进程发出去，或者转发给其他mom处理



1、如果报为空，说明客户端已经关闭，删除```m_mapLocalStream```中的对应的fd；

2、```m_mapLocalStream```中找到对应fd，将收到的数据加入tagMomStream的stream中；

3、GetOnePacket拿到去掉导引头、尾后的数据并放入中间缓存m_szBuff；如果验证这段数据的类型为INFO_CTRL_TYPE（？？？为什么还有控制报信息），对数据部分再做进一步处理 -- ```DataCtrlDeal```，否则（即为INFO_DATA_TYPE）直接```AddToFdFlow```发送出去，此函数中会根据目标进程是否由自身维护，选择直接发出或转给其他mom(循环处理？？？)



#### DataCtrlDeal

与控制报类似，但此处只有CLIENT_PROC_TYPE_INFO类型（即客户端与mom首次连通时，告知自身进程信息），对proc_type部分做进一步处理-- ```DataDealCtrlProcTypeInfo```，加到```m_mapLocalStream```中（所以发送自身进程信息时，从这遍历），此报无需回复



#### DataDealCtrlProcTypeInfo

把进程类型存储到```m_mapLocalStream```

在m_mapLocalStream中找到fd，没有就新建，然后修改对应的进程类型proc_type



#### AddToFdFlow

如果是由本机mom直接管理的目标进程就直接发送；否则转发至对应mom主机



1、首先检查数据类型是否为INFO_DATA_TYPE

2、如果目标ip即为本mom服务器，除客户端进程发送给自己之外，找到负责其目标进程的fd```FindLocalFd```（通过从数据中提取出tagDataHead，拿到目标进程），通过此fd直接发送

3、如果目标ip不是本mom，即需要转发给其他mom，找到转发给对应mom的fd```FindMomFd```，在```m_mapMomCache```中找到该fd，没有就新建，在tagMomCache中存入数据，但不马上发送，而是**加入二级缓存**，等待定时批量发送





#### FindLocalFd

找到负责目标进程的fd

检索```m_mapLocalStream```即可找到与各种客户端相连通的fd



#### FindMomFd

需要在threadFrame的连接池```list<FD_ITEM> m_rFdList```中检索有没有符合的连通到目标ip的主机



------



#### ServerRead -- m_hostSvtPeer

接收来自其他mom的报文，找到对应客户端，直接发送

1、如果数据为空，则代表客户端已关闭，清除包含与此客户端有关的所有连接、map、进程表

2、在```m_mapLocalStream```中找到对应fd，将收到的数据加入tagMomStream的stream中；

3、etOnePacket拿到去掉导引头、尾后的数据并放入中间缓存m_szBuff；如果验证这段数据类型为INFO_DATA_TYPE，找到负责其目标进程的fd```FindLocalFd```，使用此fd直接发送出去(循环处理？？？)



#### SendToServer -- 定时

将在二级缓存里的信息发送给具体mom，非阻塞发送，实质调用SendCache

将```m_mapMomCache```循环5次，以图尽可能全部发完



#### TestServerLive -- 定时（但似乎在threadframe中未使用）

根据全局进程表的更新时间，判断各mom之间心跳报是否超时，以此判断mom是否存活，如果已超时，则在全局进程表中删除对应项，但没有删除```m_mapTcpConnect```



#### TestPeer -- 定时

遍历```m_mapTcpTestConnect```与各已连通的mom服务器尝试建立连接，因为是非阻塞，每次只执行一个连接阶段



#### MonitorMom -- 定时

监控本机mom的队列信息（即要转发给其他mom的cache）

专门的监控服务器？







个人理解

一、Mom服务器提供消息转发以及Mom服务器间的信息交换，同时需要有监控机制保障Mom服务器的存活性；

所以Mom服务器需要具备：

1、接受client发送的消息，如果目标进程是由自己负责的进程，则直接转发；否则发送给对应负责的Mom服务器（此处采用了二级缓存cache，定时发送）；而且由于client有控制报和数据报区分，所以Mom需要正确处理这两类报文

ps：对于Mom服务器而言，客户端眼中的CS都是自己的client；并且Mom和它负责的client是在同一台物理主机上；

2、Mom服务器间交换各自负责的进程信息，各自均形成一个全局进程表，定时以心跳包的形式udp发送



监控保活机制：

1、通过判定接受到的心跳包是否超时，更新全局进程表，清除全局表中超时的Mom服务器所维护的进程信息（但如果该超时服务器仍然存活，仅仅因为网络问题导致心跳包部分超时，那么他本机维护的本地进程表就可以在下次心跳交换时，重新生成最新的进程信息）

2、各Mom服务器互相主动发起连接，判定对方是否存活，如果累计多次主动连接，对端某主机一直无法连接成功（连续多次连接失败，每次连接之间有间隔），则判定对端Mom主机已经下线，删除本机中与对端主机相关的所有信息，包含全局进程表、Mom间转发所需的cache表、Mom接收到的此下线Mom发送来的stream表、threadframe中负责与之通信的连接池相关项（accepted类型）；threadframe中fdtype_server类型的只置为CLOSEED；但其生成的已完成连接（accepted）会被剔除；



可以发现被动的超时机制判定对端存活性，只做初级处理，即认为是由于网络原因导致超时，不会将对端Mom服务器实质性剔除；但主动判定对端存活性，一旦连续多次连接不上，认为对端实质性下线，起到真实剔除作用

```c++
//map汇总

//全局进程表
map<string, ListNode> m_mapHostProc;

//本地进程表（其实还包含与之连接的client发送来的数据）
map<CFdOBJ*, tagMomStream*> m_mapLocalStream;

//仍是本地进程表，但属于控制报发过来的，与之连接的是client中控制报fd
map<CFdOBJ*, tagMomStream*> m_mapLocalCtrl;

//定时转发至其他Mom，所以需要一个与其他Mom主机通信的fd和需要转发数据关系映射（发送到其他Mom用的）
//cache以目标主机为单位，不是目标进程
map<CFdOBJ*, tagMomCache*> m_mapMomCache;

//Mom收到来自其他Mom服务器发送的cache数据后，保存到stream，之后提取出来send到目标client（接收其他Mom发来的）
map<CFdOBJ*, tagMomStream> m_mapMomStream;

/*保活测试用*/

//与本Mom服务器联通的其他Mom服务器（可用于：接受心跳时检查是不是正确的ip，因为udp需要检验数据；服务器初始化时与其他Mom建立连接）
map<string, STcpConnect> TcpConnect;//服务器连接
map<string ,STcpConnect> m_mapTcpTestConnect;//测试服务器连接
```







二、client端发送消息设计为控制和数据报，发送实际数据报之前，先发送控制报询问Mom服务器，目标client进程是否存活（只要目标client有就行，不是检查是否由此Mom服务器负责），确认存活的情况下，发送实质数据报；且client和Mom之间采用阻塞发送，所以发送数据报之前需要等待控制报回复。

实际编码中，client进程初始化时，控制和数据fd均会向Mom发送一次消息，告知本方进程类型（INFO_CTRL_TYPE），此类型的报文Mom不需要回复，注册到本地进程表中即可；

client收到消息时，会放入cache；





三、网络的核心之一，接收全部在threadframe中完成，即threadframe接收每个fd的数据，然后调用对应回调做实质处理（即此时回调中已经拿到了对端发来的数据）。threadframe设计为单线程处理，循环遍历整个fd list，同时正确处理fd的增减（可能是由于accept导致threadframe生成或销毁fd）；由于threadframe只负责对应的recv，所以send放置到各个client和server中各自处理；
