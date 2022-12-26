#### 第二章 线程管控

1、thread对象在构造时可接受任何可调用类型，如函数指针、重载了operator()的类、lambda对象等；

2、但是thread对象构造时，可调用对象的**参数**是先**复制**到新线程内部的栈空间，然后这些副本被当作**临时变量**，以**右值**的形式传给新线程上的函数或可调用对象。

经测试，确为复制，其会调用拷贝构造函数复制到新线程内部！





这种方式会影响到很多类型的传递，如函数参数为`string&`，传入一个`string`实参，依照上述描述，此参数会被复制到新线程内部，然后以右值形式传递给最终执行函数，但是给一个左值形参传入右值是错误的！！故除非修改形参为`const string&`，或者将实参处理`std::ref(xxx)`

> The arguments to the thread function are moved or copied by value. If a reference argument needs to be passed to the thread function, it has to be wrapped (e.g. with `std::ref` or `std::cref`

另传入类的成员函数时，应传入函数指针，指向此成员函数，此外提供一个对象指针，作为此函数的第一个参数（隐式参数）

```C++
class X
{
public:
    void do_work(int i)
    {
        std::cout << "do work " << i << std::endl;
    }
};

int main()
{
	X my_x;
  	// 相当于调用my_x.do_work()
    std::thread t(&X::do_work, &my_x, 100);
  	t.join();
  	return 0;
}

// 此处无法使用function“封装”后的成员函数
std::function<void(X&， int)> f_do_work = &X::do_work;
std::thread t(&f_do_work, &my_x， 100); // 报错

// 原因待查
```

针对那些只有移动语义的对象，临时对象的话移动会自动发生，但如果是具名变量（因为有名称的变量都是左值），必须使用`std::move`转为右值方可移动，如`unique_ptr`（禁用了拷贝构造、赋值操作符）只有移动构造，作为参数直接传入会报错，需要转为右值，然后转移到新线程内部，再由线程内部转移到函数中作为参数

```C++
std::unique_ptr<int> p(new int(100));
std::thread t(do_something_ptr, std::move(p));
```





3、thread支持移动语义

如果一个`thread`对象正管控着一个线程，如果继续赋值，将会导致`terminate()`被调用

thread对象析构之前，必须明确是等待线程完成（join）还是要与之分离（detach），否则就会导致关联的线程终结；所以给一个正在管控的线程赋值，相当于让它正在管控的线程结束，而此时并不知晓join\detach，导致终止。所以推荐对`thread`包裹，在析构时自动执行`thread.join()`



`std::thread::id`是全序关系，可以作为`map`等有序容器的键值，也可作为`unordered_map`等无序关联容器的键值，即hash_key



#### 第三章 线程间数据共享

race condition：多个线程同时抢先运行

deadlock：多个线程之间相互等待

1、用互斥保护共享数据

使用互斥时，建议不要在锁所在的作用域外传递指针或引用，指向受保护的数据；无论是通过函数返回值将它们（指针或引用）保存，还是将它们作为函数的参数传递出去。因为这些指针或引用可以直接操作数据，而没有锁的限制。

1）同时获取多个锁，以避免死锁发生：

`std::lock()`，`std::scoped_lock<>`都可以获取多个锁（同时获取），其中`std::scoped_lock<>`是`std::lock_guard<>`的强化版本，功能完全一致，因为支持可变参数模板，所以可以同时给多个互斥量加锁，同时仍然沿用RAII，析构时自动解锁。再没有`std::scoped_lock<>`时，使用`std::lock`对多个互斥量先行上锁，然后借助`std::lock_guard<>`实现析构自动解锁。

```C++
std::lock(lhs.m,rhs.m);
// adopt_lock 说明已经加锁，不会再次加锁
// assume the calling thread already has ownership of the mutex
std::lock_guard<std::mutex> lock_a(lhs.m, std::adopt_lock);
std::lock_guard<std::mutex> lock_b(rhs.m, std::adopt_lock);


// C++17
std::scoped_lock<std::mutex, std::mutex> guard(lhs.m, rhs.m);
```

2）利用层次锁方式实现按某种顺序加锁

STL暂未提供层次锁，自定义实现类中实现lock、unlock、try_lock即可被视为满足互斥概念所需的操作，加锁之前检查需要锁的层级是否正确（如后加的锁的层级必须低于之前的锁层级等），可以使用线程内部私有变量`thread_local`实现

3) unique_lock<> 灵活加锁

`std::unique_lock<>`可以不始终占用与之关联的互斥，而`std::scoped_lock<>`、`std::lock_guard<>`都会始终占用与之关联的互斥；所以`std::unique_lock<>`会更加占用空间，以存放和更新互斥信息，并且`std::unique_lock<>`具有成员函数lock、unlock、try_lock，所以可以放入`std::lock()`中进行加锁，并且`std::unique_lock<>`在解锁时，必须要明确当前已经占用着关联的互斥。

`std::unique_lock<>`可移动而不可复制，可以转移对应的互斥归属权？？



2、除互斥以外的保护共享数据的工具

以往单例返回指针使用两阶段锁方式，但两阶段锁仍然有data race风险，C++11提供了`std::call_once` `std::once_flag`，用以实现只会被初始化一次。

```C++
class_ptr为SomeClass类对外提供的实例指针

// 两阶段锁
SomeClass* GetInstance()
{
  if (!class_ptr)
  {
    std::lock_guard<std::mutex> lk(mutex_);
    if (!class_ptr)
    {
      class_ptr = new SomeClass();
    }
  }
  return class_ptr;
}


//--->
static std::once_flag resource_flag;

void Init()
{
  class_ptr = new SomeClass();
}


SomeClass* GetInstance()
{
  // 保证多线程也只会执行一次
  std::call_once(resource_flag, &SomeClass::Init, this);
  return class_ptr;
}
```



另，在C++11之前如果使用local static，可能存在多线程安全问题，因为local static对象只在第一次执行时初始化，而多线程环境下，每个线程都认为自己是第一次执行，执行多次初始化；而C++11中处理了此类情况，规定初始化只会在某一个线程上单独发生。

```C++
SomeClass& GetInstance()
{
  // C++11 解决了 local static的多线程初始化问题
  static SomeClass ins;
  return ins;
}
```

1) 读写锁 - `std::shared_mutex`

`std::shared_lock<std::shared_mutex>`读锁/共享锁，多个读者可以并发执行

`std::lock_guard<std::shared_mutex> OR std::unique_lock<std::shared_mutex>`  写锁/排他锁，写者独占执行，读者也不可以在此时执行



2) 递归锁  `std::recursive_mutex`-- 不建议使用

正常的mutex，如果线程已经加锁，**同一个线程**如果再重复加锁，将会导致未定义行为，但是`std::recursive_mutex`允许同一线程对同一互斥重复加锁，不过如果要释放锁，要调用相应次数的unlock，如调用了3次lock，就要调用3次unlock才能释放所有的锁。



#### 第四章 并发操作的同步

场景：线程A等待线程B某项事件的完成

1） 条件变量

线程之间共享某状态信息，当一个线程A结束使得状态发生变化时，另一个线程B响应；

一种实现是线程B轮询，不断检查状态信息是否变化；此方法中需要对状态加锁，当线程B加锁后可能会影响到线程A的运行，因为线程A只能等待线程B解锁才能修改状态，另外线程B的资源几乎浪费在了无意义的循环检查中。

优化为：线程B不再循环检查，而是每检查一次后sleep一段时间，但是sleep的时间长短的设置需要谨慎

另一种实现为：利用STL提供的条件变量和互斥量

```C++
#include <condition_variable>
// 搭配 mutex 和 condition_variable实现同步

// 通知某一个正在等待的线程
condition_variable.notify_one();
// 通知所有正在等待的线程
condition_variable.notify_all();

// std::lock_guard<std::mutex> lk(mu_); // 不使用此lock
std::unique_lock<std::mutex> lk(mu_); // 因为线程等待期间会解锁，而结束等待运行时又会重新加锁

// wait()接受两个参数，一个为lock，另一个是条件的断言
// wait 会先检查条件是否成立，如果成立(断言返回true)，则继续运行（保持锁状态）
// 如果断言不成立(返回false)，则解锁，并使得线程等待/阻塞
condition_variable.wait(lk, [](){return !q.empty();});
```

条件变量的wait实现中可能使用忙等待，即不断循环检测条件是否成立，不成立就解锁，下次检查是再次加锁；而这种不断查验条件是否成立的方式，如果不影响到另一个线程的话（因为检查条件是否成立的函数中可能影响另外的线程），称之为**伪唤醒**。



2）future

上述条件变量的使用中，线程被唤醒之后都会去检测条件是否成立；但是很多情况下，线程只会被唤醒一次，即线程B只需要从线程A中获取一次，如拿到某个结果值，之后就不需要等待线程A的结果，那么使用条件变量显得略为“沉重”，所以配合future使用来等待此类一次性事件发生。

2-1）async()

STL提供`std::async()`来获取future对象，`std::async()`接受可调用对象及其参数作为参数（类似`std::function`），并返回一个`std::future<>`对象，`std::future<>`的特化类型取决于`std::async()`的可调用对象的返回类型。

PS：此时的可调用对象类似线程A，本线程相当于线程B，需要从线程A中获取结果，不过这个结果只需要一次。

`std::async()`可以指定可调用对象的被调用时机，如延后调用，直到显式的需要获取其结果（`future.get()/wait()`），或者另外启动线程，直接执行可调用对象；默认为系统自行选择实现方式，具体使用方式为：

（就像async这个函数的名称一样，它可以实现异步执行任务函数）

```C++
int some_function(int);

// 延后调用
std::future<int> res = std::async(std::launch::deferred, some_function(), 38);
res.get(); //此时才会实际执行some_function() 


// 另外启动线程，直接执行
std::future<int> res = std::async(std::launch::async, some_function(), 38);
```



2-2）packaged_task<>

可以看出`std::async()`非常“轻量化”，那么`std::packaged_task<>`就是对前者的一种抽象化表示，`std::packaged_task<>`是个类模板，模板参数为函数签名（相当于可调用对象），它的成员函数`get_future()`返回`std::future<>`对象，利用`std::packaged_task<>`可以将各种可调用对象抽象化，进而配合线程池等使用。

`std::packaged_task<>`**具备函数调用操作符，所以可以直接调用！**只不过其函数调用操作符（`operator()(xxxx)`）的参数取决于函数签名的参数。

```C++
int do_something(int x, int y) { return std::pow(x, y); }

// std::future的特化同样和函数签名的返回值相同
std::packaged_task<int(int, int)> task(do_something);
// 未来可以通过此返回值获取任务的结果
std::future<int> res = task.get_future();

// 直接调用
task(2, 3);
std::cout << "task_direct " << res.get() << std::endl;

// 或者放到另一个线程，异步执行
// std::move 是必须的，why？？
std::thread t1(std::move(task), 2, 2);
t1.join();
std::cout << "task_thread " << res.get() << std::endl;


// 以上代码不能直接运行，因为thread中运行的task的future已经在上面的直接调用中获得了，不能再次获得
// std::future_error: Future already retrieved
```

> **future的get仅能被有效调用一次！！**因为get()调用时会发生移动操作，之后该值不复存在！
>
> future对象仅能移动构造和移动赋值！归属权在多个实例之间转移，同一时刻仅能有一个future实例享有特定的异步结果。



2-3）promise

通过将`std::promise<T> `和`std::future<T>`一一对应，`std::promise<T> `调用成员函数`get_future()`可获取对应的`std::future<T>`，然后线程可以阻塞在`std::future<T>`上，直到`std::promise<T>`通过`set_value(xx)`设置结果值，`std::future<T>`才会继续运行，相当于`std::promise<T> set_value`发送信号给对应的`std::future<T>`使之就绪，将`std::promise<T>`放置到另一个线程中，就可以实现跨线程的同步。

```c++
void do_work(std::promise<void> barrier)
{
    std::this_thread::sleep_for(std::chrono::seconds(1));
    barrier.set_value();
}


std::promise<void> barrier;
std::future<void> barrier_future = barrier.get_future();
std::thread new_work_thread(do_work, std::move(barrier));
barrier_future.wait();
new_work_thread.join();
```

2-4）异常

首先，异常将会存放在`std::future`中，如果任务在执行过程中出现抛出异常，那么此异常将会存放在`std::future`中，在`std::future get()`被调用时，此异常将会被重新抛出

`std::async `、`std::packaged_task<>`、`std::promise<T>`均是如此，只不过`std::promise<T>`中需要手动的`set_exception(std::exception_ptr)`（一如其手动`set_value()`一样）来设置对应异常。

2-5）多线程同时等待

上述描述中，均为一个线程等待future的就绪，也只允许一个线程等待结果；如果需要多个线程同时等待同一个目标结果，就要使用`std::shared_future`。

`std::shared_future`的实例能够复制出副本，所以可以拥有该类的多个对象，它们指向同一个异步任务的状态数据。

而`std::shared_future`的实例是依据`std::future`实例构造得到，而`std::future`**只可以移动构造或移动赋值**，所以一旦通过`std::move`得到`std::shared_future`对象，那么原有的`std::future`将变成空状态。

```C++
std::promise<int> p;
std::future<int> f = p.get_future();

std::shared_future<int> sf(std::move(f));
// OR
std::shared_future<int> sf(p.get_future());

// 此时的f已经为空，无效
assert(!f.valid());

// 也可以通过future直接构造shared_future
auto sf = p.get_future().share();
```

`std::shared_future`和`std::future`关联有异步任务的相关数据（即异步状态/shared state），如参数、执行结果等，这些数据一般保存在堆区，而`std::shared_future`和`std::future`提供`valid()`函数判定异步状态是否有效。



但是，`std::shared_future`本身有可能发生条件竞争，所以需要自行加锁做保护，如多个线程同时对同一个`std::shared_future`对象做`wait/get`，可能发生数据竞争。所以要么自己加锁，要么在各个线程中复制一个

`std::shared_future`对象，使各线程之间的`std::shared_future`对象独立，它们的数据同步由STL来负责，从而避免data race。





3）时钟类

C++11中引入了`std::chrono`类，提供时间和日期的一些工具。

**时长**类型为`std::chrono::duration<x, ratio<y, z>>`，其中x表示使用什么类型来保存计时单位的数量（即时长），`ratio<y,z>`是个分数，表示每个计时单位代表多少秒，如`ratio<60, 1>`表示1个计时单位等于60秒，也就是说采用的计时单位是分钟。`ratio<1, 1000>`1000个单位等于1秒，也就是采用毫秒。默认为`ratio<1, 1>`，即秒。通过成员函数`count()`可以获取其计时单位的数量。

不过`std::chrono`类中的**时间点**类型为`std::chrono::time_point<x, y>`，不过也提供了`std::chrono::time_point<>`和C语言中`time_t`的转化函数。

计算机的时间点其实也是一种时长，因为实现中一般采取某个时钟纪元作为起始点，从该起始点到当前时点的时长即为时间点。常见的时钟纪元有1970年1月1日0时0分0秒或者计算机启动的时刻。所以`std::chrono::time_point<x, y>`中第一个参数指明所用的时钟，如`system_clock`、`steady_clock`、`high_resolution_clock`等（其实也隐式的指明了对应的时钟纪元），第二个参数指明时长的计时单位。

时间点和时长均可以做算数运算。



之前提到的条件变量、future等特性中的`wait`函数均提供有两个超时函数，`wait_for()`、`wait_until()`，就像其名字一样，前者说明等待一段时间后停止阻塞/等待，后者是等到达某个时间点时停止阻塞/等待。

> 条件变量的wait_for，如果在等待的时间快消耗完时发生伪唤醒，但条件断言未成立，即继续等待，那么会重新开始一次完整的等待，这样有可能导致无限等待下去！！



接受超时时限的函数

| 类/namespace                              | 函数                                       | 返回                                       |
| ---------------------------------------- | ---------------------------------------- | ---------------------------------------- |
| `std::this_thread`                       | `sleep_for(duration)`<br />`sleep_until(time_point)` | 无                                        |
| `std::condition_variable`等               | `wait_for(lock, duration, predicate)`<br />`wait_until(lock, time_point, predicate)`<br />`wait_for(lock, duration)`<br />`wait_until(lock, time_point)` | bool-被唤醒时断言的返回值                          |
| `std::timed_mutex`<br />`std::recursive_timed_mutex`<br />`std::shared_timed_mutex` | `try_lock_for(duration)//duration时限内获取锁`<br />`try_lock_until(time_point)//time_point之前获取锁` | bool - 若获得了锁返回true，否则false               |
| `std::unique_lock<TimedLockable>`        | `try_lock_for(duration)//duration时限内获取锁`<br />`try_lock_until(time_point)//time_point之前获取锁` |                                          |
| `std::future<ValueType>`<br />`std::shared_future<ValueType>` | `wait_for(duration)`<br />`wait_until(time_point)` | 如果等待超时则返回`std::future_status::timeout`<br />如果future已准备就绪，则返回`std::future_status::ready`<br />如果future上的函数按照推迟方式执行，且尚未执行，则返回`std::future_status::defered` |





4）运用同步操作简化代码

4-1）函数式编程 functional programming

函数调用的结果完全取决于参数，即根据不同的参数，得出不同的结果，不依赖于任何外部状态。这类纯函数产生的作用被完全限制到返回值上，而不会改动任何外部状态。

如果外部状态不会被改变，那么就不需要要互斥等操作去保护状态，我们只需要关心那些会改变共享部状态的非纯函数，从而简化同步。



4-2）利用消息传递完成同步

理念：各线程之间不存在共享数据，线程只接收消息，然后根据消息做出相应的行为。

这种理念和状态机很相似，每个线程相互独立运行，然后根据收到的消息转换为不同的状态，执行不同的状态函数。

每个线程将拥有各自的消息发送器和消息接收器（发送方将消息放入队列，然后由分发器分发，即从队列取出并执行），以及当前的状态（这个状态可以是成员函数，线程run运行此状态函数），线程无限循环的运行当前的状态函数，状态函数内部根据收到的不同的消息类型，可能会修改当前的状态，进而使得线程转而运行其他状态函数，新的状态函数又会继续执行上述操作。不同的消息种类用独立的struct类型表示。

PS：和reactor的网络通信模型有共同之处，都需要一个消息分发器，交由不同的消息处理实例。不过，网络模型中，线程之间的消息传递不再是本地的线程，而是跨地域跨设备的线程，需要考虑更多的异常。





`std::mutex`

`std::lock_guard`

`std::scoped_lock`

`std::unique_lock`

`std::lock()`



`std::call_once`

`std::once_flag`



`std::shared_mutex`

`std::recursive_mutex`





