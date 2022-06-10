### FdOBJ 套接字函数封装

#### 问题待查：

1. 研究下getsockopt返回的error的含义
2. getsockopt 的返回值含义
3. getsockopt 与 WSAGetLastError两者返回的error值的区别
4. 绑定ip时，ip对应的位置是 sockaddr_in.sin_addr.s_addr
5. win32和linux下阻塞控制函数的区别与应用
6. linger和NoDelay的含义及应用



```#include <stdio.h>``` 

stdio库函数为减少使用系统调用，所以会使用内部的缓冲区stdio缓冲区；如fgets函数就会用到该缓冲区；
这种函数内部的缓冲区，系统调用时是看不到的，所以select等系统调用函数无法知晓这类缓冲区中是否有值存在！

### Mutex 锁
#### 锁分类
1. 自旋锁 spinlock
  轮询，无限等待；只要当前资源被其他线程占用，本线程就不断询问是否可占用
  过度等待会浪费CPU资源
  适用于：上锁的业务代码运行很快的场景，可避免互斥锁下的线程切换

2. 互斥器/锁 mutex （mutual exclusive）
  不再轮询、无限等待；用一个互斥器去管理所有锁，当前资源占用被释放后，通知线程，否则线程睡眠
  为了管理所有锁，需要OS层面调度，故互斥器方式更加耗资源，涉及上下文切换

常见用法：
定义一个线程间共享的互斥量mutex；各线程去竞争mutex，谁抢到谁使用资源

锁（mutex） 来管锁（占有资源）！


3. 条件变量 condition variable
  防止业务逻辑中出现无限等待 条件达成(即在使用了互斥量后，仍然出现了自旋锁中的无限循环现象)，从而浪费CPU资源，而业务条件，操作系统无法管理，故设置条件变量
  业务条件未达成前，线程睡眠，达成条件后notify线程，再来加锁

条件变量属于线程间通讯机制，与互斥量一起使用！

> 条件变量已经不是锁的概念了，而是在获取到资源访问权限后，由于实际业务场景又不能访问的情况下要如何实现执行高效

4. 读写锁 readers-writers lock
  即OS信号量中的读写者问题，写者互斥，读者间不互斥（读共享，写独占）；
  可以用spinlock或者mutex实现

**MR Hu建议：直接用互斥锁，不用读写锁，后者可能会把逻辑搞乱**
```C++
/*类似PV操作中读写者问题
* 读优先，写者可能饥饿
*
*用spinlock或者mutex实现读写锁
*/
//保护读者数量
互斥量1 = 1;
//保护读写正常
互斥量2 = 1;

读者数量 = 0;
读者加锁()
{
    加锁-互斥量1;
    读者数量++;
    if （读者数量 == 1）
    {
        //第一个读者到达，不允许写者进入，加锁
        加锁-互斥量2;
    }
    解锁-互斥量1;
}

读者解锁()
{
    加锁-互斥量1;
    读者数量--;
    if (读者数量 == 0)
    {
        解锁-互斥量2;
    }
    解锁-互斥量1;
}

写者加锁()
{
    加锁-互斥量2;
}

写者解锁()
{
    解锁-互斥量2;
}
```

#### pthread API
##### 互斥量
```C++
//定义一个互斥量
pthread_mutex_t mtx;
//初始化
pthread_mutex_init(&mtx, NULL);
//加锁
pthread_mutex_lock(&mtx);
//解锁
pthread_mutexx_unlock(&mtx);
//销毁
pthread_mutex_destroy(&mtx);

```

互斥量属于睡眠等待类型，所以和IO模型类似，互斥量允许非阻塞版本，当发现线程无法获取互斥量时（即无法获取资源时）不再睡眠，而是立即返回一个特殊的错误值，即```trylock```
```C++
ret = pthread_mutex_trylock(&mtx);
//加锁成功
if (ret == 0)
{
//do something...
pthread_mutex_unlock(&mtx);
}
//锁正在被使用
else if (ret == EBUSY)
{
//some address...
}
```

互斥量依据同一线程能否多次加锁（即多次申请占用资源？？）:
分为递归互斥量recursive mutex（可重入锁）和非递归互斥量non-recursive mutex（不可重入锁）

同一线程对非递归互斥量多次加锁，可能造成死锁；递归互斥量无此风险
pthread中通过给mutex添加```PTHREAD_MUTEX_RECURSIVE```属性的方式，使用递归互斥量
```C++
//定义一个互斥量
pthread_mutex_t mtx;
//定义一个互斥量的属性变量
pthread_mutexattr_t mtx_attr;

//初始化互斥量的属性变量
pthread_mutexattr_init(&mtx_attr);
//设置互斥量的属性为递归
pthread_mutexattr_settype(&mtx_attr, PTHREAD_MUTEX_RECURSIVE);

//属性赋值给互斥量
pthread_mutex_init(&mtx, &mtx_attr);
```
>使用好递归互斥量是十分tricky的，仅当没有其他解决方案时才使用


##### 读写锁
读写锁其实是一个锁，可以理解为一个锁的读模式和写模式；

特性：
当读写锁被加了写锁时，其他线程对该锁加读锁或写锁都会被阻塞；

当读写锁被加了读锁时，其他线程对该锁加写锁会被阻塞，加读锁会成功；

```C++
//定义一个读写锁
pthread_rwlock_t rwlock;
//在读之前加读锁
pthread_rwlock_rdlock(&rwlock);

...do something about read

//读完之后释放锁
pthread_rwlock_unlock(&rw_lock);

//写之前，加写锁
pthread_rwlock_wrlock(&rw_lock);

...do something write

//写之后释放锁
pthread_rwlock_unlock(&rw_lock);

//销毁读写锁
pthread_rwlock_destroy(&rw_lock);

```


同样读写锁存在非阻塞模式，```trylock```
```C++
pthread_rwlock_tryrdlock(&rw_lock);
pthread_rwlock_trywrlock(&rw_lock);
```

##### 自旋锁
```C++
//定义自旋锁变量
pthread_spinlock_t spinlock;

//初始化
pthread_spin_init(&spinlock, 0);

//加锁
pthread_spin_lock(&spinlock);

//解锁
pthread_spin_unlock(&spinlock);

//销毁
pthread_spin_destroy(&spinlock);
```
自旋锁pthread_spin_init时，第二参数为pshared（int），表示是否允许进程间共享自旋锁Thread Process-Shared Synchronization（从目前来看，自旋、互斥量都是线程间共享）
互斥量也可以设置为进程间共享；pshared有两个枚举值：

PTHREAD_PROCESS_PRIVATE：仅同进程下读线程可以使用该自旋锁
PTHREAD_PROCESS_SHARED：不同进程下的线程可以使用该自旋锁

Linux中两者分别为0，1

##### 条件变量
常与互斥量搭配使用

pthread_cond_wait中会对当前加的互斥量解锁，然后等待条件通知？
pthread_cond_signal中会通知之前等待的线程，然后对当前加的互斥量解锁吗？？

**在使用条件变量的过程中：使用while而不是if来做判断状态是否满足**
1. 防止惊群？？？
2. 防止虚假唤醒（没有pthread_cond_signal就解除了阻塞）？？？

```C++
pthread_mutex_t mtx;
pthread_cond_t cond;

//初始化
pthread_mutex_init(&mtx, NULL);
pthread_cond_init(&cond, NULL);

//加锁
pthread_mutex_lock(&mtx);


//加锁成功，等待条件变量触发；此处con和mtx一一对应，否则bug
pthread_cond_wait(&cond, &mtx);

//加锁
pthread_mutex_lock(&mtx);

//条件触发，唤醒
pthread_cond_signal(&cond);


//解锁
pthread_mutex_unlock(&mtx);
//销毁
pthread_mutex_destroy(&mtx);
```

##### 问题
pthread_cond_wait中会对当前加的互斥量解锁，然后等待条件通知？
pthread_cond_signal中会通知之前等待的线程，然后对当前加的互斥量解锁吗？？

**在使用条件变量的过程中：使用while而不是if来做判断状态是否满足**
1. 防止惊群？？？
2. 防止虚假唤醒（没有pthread_cond_signal就解除了阻塞）？？？


### ThreadBuffer
线程的共用缓冲区
#### 疑问：
Q:写入的时候，为什么```bBuf```区域完全不加写锁，不会导致多个线程写入冲突吗？？
反而修改```tagThreadBuff```的count时全部加了写锁？？！！

A:好像这个类设计只是单纯的一个缓冲区,没考虑多线程并发(或者说实际场景不会出现),而加锁只是为了count更精准

### StreamProc
流数据处理类
设置引导头数据(SetLeadHead)时,为什么用for循环而不用memcpy?

同时每次设置头,尾数据时,清空buffer的delete[]改成memset 0,会不会更节省时间?

#### 疑问
流数据包的结构，引导头、尾、小尾具体含义


### ThreadFrame
_Run函数理解：
摆在面前的是一个套接字列表，什么类型（CLIENT、SERVER、ACCEPT、COMMON（大概率是udp类型））和状态（FD_STATUS_COMMON（初始状态）、FD_STATUS_CONNECTED、FD_STATUS_CLOSED、FD_STATUS_LISTENED、FD_STATUS_TESTCONN）都有，需要具体类型、具体状态具体分析；

其中，CLIENT类型的套接字，可能会在COMMON、CONNECTED、CLOSED、TESTCONN（非阻塞Connect）状态间转变；

而SERVER类型的套接字，只有LISTENED状态；

ACCEPT类型的套接字，指SERVER类型套接字accept后的已完成连接；



一、首先，所有套接字加入FD_SET，以便select

1、从而需要一次fdlist的全体遍历，在加入FD_SET前:

    1）对处于连接CONNECTED状态但是fd明显非法的套接字，状态修改为CLOSE

    2）对已关闭（CLOSED）的CLIENT套接字，重新创建，状态修改为COMMON

    3）对处于初始化（FD_STATUS_COMMON）状态的套接字：

        a）其中已超时的CLIENT套接字，做一次重连Connect，针对重连结果，修改套接字状态：
        	连接正常，就把进程状态发给对端（MOM系统）；
        	连接返回EWOULDBLOCK，说明非阻塞连接未完成，等待检测连接TestConnect()
        	连接失败，就close套接字；
    
        b）未超时的CLIENT套接字，无需管理，因为可能连接还在路上
    
        c）SERVER类型套接字，做一次listen，状态修改为LISTENED

2、所有状态的套接字均进入读集，而TESTCONN状态的套接字同时加入写集；

3、selct等待就绪；

二、其次，针对select时已就绪套接字，不同类型不同处理（又一次fdlist遍历）

1、针对状态为TESTCONN的套接字（读写集均有）：

检测是否因异常状态处于就绪(TestConnect())，如果是正常连接，状态修改为CONNECTED；其他情况，状态修改为CLOSE

2、针对其他状态套接字

    1)SERVER类型套接字就做Accept，把新的已连接套接字加入fdlist中（这个新的已连接套接字是ACCEPT类型的CONNECTED状态）

    2)非初始化COMMON状态的非SERVER类型套接字，接收数据报recv；
    如果recv返回错误的，关闭套接字，且状态修改为CLOSE，其中类型为ACCEPT的套接字直接清出fdlist，但CLIENT类型的套接字不管

三、最后，针对select超时（select return ==0）,对里面FD_STATUS_TESTCONN状态的套接字（也只有此状态的套接字select超时需要处理）检查有没有超过MAX_CONNECT_TIME时间，有就CLOSE掉，没有就不需要管（第三次fdlist遍历）



```TestConnect（&rset, &wset）```函数:

完成了select和getsockopt的分拆，在一个大的select中，对某个fd进行单独的getsockopt检测error（TestConnect中又对某fd单独getsockopt）



#### 疑问

非阻塞connect，如果getsockopt返回的error！=0，一定是因为连接超时吗？

学习回调函数CallBack()

非阻塞connect检测就绪时，最好读写集都检测！

### Tips
1. typedef或者using 用于函数指针来增加程序可读性

```C++
//fp为函数指针
typedef int(*fp)(int,float);
//using fp = int(*)(int, float)

//返回以上函数指针
fp func(int a){}

//如果没有typedef，可读性较差，需要从内向外读才能看出是什么
 int (*func(int a))(int, float)
{}
```


2. 线程创建

```int pthread_create(pthread_t *threaed, const pthread_attr_t *attr, void *(*start_routine)(void *), void *arg);```

创建线程并执行函数；
第三参数start_routine为线程创建后需要执行的函数，返回值和参数类型均为void *，所以传入参数时可能需要强转；
第四参数为start_routine函数的参数信息；

3. 等待线程执行结束

```int pthread_join(pthreaed_t thread, void ** retval);```

阻塞**调用它的线程**，直到**目标线程**执行完毕（接收到返回值），执行完毕后，thread线程所占所占资源会被释放
thread即为需要等待的线程；
retval为线程的返回值，如果thread没有返回值或者不需要接受返回值，可以设置为NULL