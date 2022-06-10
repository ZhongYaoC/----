1、不允许使用不完整的类型

不完整类型：缺乏足够的信息，例如长度去描述一个完整的对象



（1）前向\前置声明 [^1]即为常见的不完整对象；

因为类作为前向声明时，文件中只能有该类的指针或者引用，而不是使用该类，使用需要完整的include！

根据google C++ style guide，尽量避免前置声明的使用，使用#include包含所需的头文件

（2）未知长度的数组

```int a[];```

（3）待搜集资料



2、野指针

针对某堆上资源，如果单纯delete析构对象并释放内存，而其指针不赋值为null的话，该指针即为野指针；因为delete只是将该指针指向的内存空间释放，而该指针仍指向该空间，此时该空间属于未知状态，继续使用该指针可能导致未知错误。

```C++
int* p = new int(100);
//do something about p
//....
delete p;
p = NULL;
```



空悬指针：指针指向的地址空间已被释放或者未定义

野指针：指针未被初始化

[^1]: linux多线程编程-陈硕



3、 static成员的使用



4、慎用memcpy

```memcpy(void* dest, void* sourc, int len);```其中长度是由用户自己把控，就会出现len大于dest的内存长度，而且此时并不会报错！！



5、网络间传输C++类对象时需要做序列化

因为单纯将对象指针信息放入数据包中传输，对端得到的信息中并不包含该对象的基类、继承关系等！



6、string类型内似乎不需要管理转义字符这一说？？？



7、char* char[]

char* a = "asdad";//代表指向一个静态存储区的字符串的指针，所以不可以修改

char a[][][] = "asdad";//代表在栈上的内存中存放的长度为6（5+1）的字符串，可以修改

char* arr = new char[6];//代表在堆上申请的6字节的内存空间

char* arr = new  char(6);//代表申请单字符的内存空间，并用'6'这个字符初始化，而非一维数组



8、using声明和using指示

using声明可以使用，using指示最好不要使用！

区别：



9、模板全特化和偏特化

首先申明，函数模板是不允许偏特化（partial specialization）的，函数模板只允许全特化（template specialization）；类模板是可以做全特化和偏特化的。

全特化表示为所有的模板参数提供实参，**全特化本质是一个实例**

那么偏特化就代表着只为部分模板参数提供实参，使用时用户仍需要为那些未指定的参数提供实参（因为只有类支持偏特化，所以是提供实参，而不是像函数模板时那样，由编译器根据所传入的实参推测具体类型），**偏特化的类模板仍然是模板**

1）函数模板特化：

```C++
//以下对std::swap 函数做了特化，当处理A类时，此特化版的函数会被调用
namespace std
{
	template<>//表明正在定义一个全特化版本
	void swap<A>(A& a, A& b) noexcept
	{
		std::cout << "std::swap called! " << std::endl;
		a.swap(b);
	}
}
```

函数重载

```C++
//此时 假设A类型是个template类型
//此时的swap函数实际是std::swap函数的重载版本，而非偏特化；
//和特化之间的区别在于swap后的<>，但是特化版本中swap后的<A>似乎是可以去掉的。。
//但是std不支持此重载，因为std命名空间禁止重载！官方人为规定，防止std过度膨胀
namespace std
{
  template <typename T>
  void swap(A<T>& a, A<T>& b)
  {
    ...
  }
}
```





2）类特化：

````C++
namespec std
{
	template<>//表明正在定义一个全特化版本
	struct hash<SalesData>
    {
    	
    }
}
````



3）类偏特化：

```C++
//类模板 -- 原始版本
template <class T> struct remove_reference
{
  typedef T type;
};

//偏特化，针对左值引用
template <class T> struct remove_reference<T&>
{
  typedef T type;
};

//偏特化，针对右值引用
template <class T> struct remove_reference<T&&>
{
  typedef T type;
};

//偏特化时，模板参数列表是原始模板的参数列表的子集或者一个特例化版本
//上述中 模板参数列表即为两个不同的特例化版本
```



11、name lookup rules （C++中名称查找法则）/ arguement-dependent lookup（实参取决之查找法则）









12、链接器工具错误 LNK2005

[链接器工具错误 LNK2005 | Microsoft Docs](https://docs.microsoft.com/zh-cn/cpp/error-messages/tool-errors/linker-tools-error-lnk2005?view=msvc-170)

符号被定义了多次。

此类错误常见有两种

1）当.h文件定义变量（似乎是全局变量）时，会出现此错误

```C++
// LNK2005_global.h
int global_int;  // LNK2005

//修改1
extern int global_int;
//然后在有且仅有一个的cpp文件中定义
int global_int  = 17;
//此变量现在是全局变量，非必要不建议


//修改2 -- 声明为静态变量
static int static_int = 17;
//这会将定义的范围限制为当前对象文件，并允许多个对象文件具有其自己的变量副本。 不建议在头文件中定义静态变量，因为可能会与全局变量混淆。 更愿意将静态变量定义移到使用它们的源文件中。


//修改3
__declspec(selectany) int global_int = 17;
```



2）.h文件中声明类成员函数后，在类外部（同样在.h文件中）直接定义函数

因为一般如果声明和定义在一起的话，是二合一，类内部直接定义，也就是隐式的inline函数；但如果将类定义写在.h文件中但是类外部，就需要手动声明inline函数，或者干脆将函数实现写在类内或者新建一个.cpp文件，定义函数实现。

因为如果该头文件包含在多个文件中，那么这个函数定义将在可执行文件中出现多次（此处是不是inline和普通的函数区别点？？按理来说，如果做了include guard，不会出现多次），导致链接错误。因为inline函数的本质在于当调用此函数时，编译时就会将此函数替换为函数定义，从而避免了调用开销









13、lambda

每个lambda都有它自己唯一的（未命名的）类类型，（是的，lambda是个类类型，而不是简单的函数），而它的捕获列表中的变量，便是这个类的成员变量！！！

```c++
//此处的mod就是一个未命名的函数对象类，其成员变量为空
auto mod = [](int i, int j){return i % j;}
```







14、bind 和 function

C++中的可调用类型：函数、函数指针、lambda表达式、重载了调用运算符的类、bind创建的对象

而这些不同的可调用类型之间，可以共享同一种**调用形式**，调用形式指明了返回类型和传递给调用的实参类型，一种调用形式对应一种函数类型

但是如果尝试用函数指针（如`int (*)(int, int)`）作为这种共享的调用形式是错误的，因为lambda表达式、重载了调用运算符的类等是不能被简单视作函数指针类型的，如每个lambda都有它自己的类型

所以C11中引入了`std::function`作为这个共享的调用形式，function是个模板类，使用时需要显式说明能够表示的对象的调用形式，如`function<int(int, int)>`就可以容纳所有以双int为实参，int为返回类型的调用类型，包括了lambda、重载了调用运算符的类等

```C++
int add(int i, intj){return i + j;}
struct divide 
{
  int operator()(int i, int j)
  {
    return i / j;
  }
}

auto mod = [](int i, int j){return i %j;}

int on_minus(int i, int j){return i - j;}
auto minus = bind(on_minus, _1, _2);


function<int(int, int)> f1 = add;
function<int(int, int)> f2 = divide();
function<int(int, int)> f3 = mod;
function<int(int, int)> f4 = minus;

//
f1(1, 2);
f2(1, 2);
f3(1, 2);
f4(1, 2);
```



如果说function负责调用形式这件事的话，那么bind就是对应于可调用对象，bind接受一个可调用对象，然后生成一个新的可调用对象来"适应"原对象的参数列表

```c++
//常用形式
auto new_callable = bind(callable, arg_list);
//arg_list中可以使用占位符 _n，占位符定义在std::placeholders，表示第几个参数
```



function统一了调用形式，bind可以转变可调用对象，那么利用这两个可以实现很华丽的回调形式！不再需要以前的那种static的两段式回调！

但是，怎么用呢！！



以下两个可能会给bind function带来些灵感，两者可能涉及bind、function到底是如何实现的

15、keyspace中的callable回调形式

16、nmstl中的回调实现





17、名称遮掩规则 name-hiding rules

只要名称相同即为遮掩，无所谓类型

而如果应用到继承中，那么只要派生类中有与基类同名的函数，那么派生类中的此函数就会将基类中的同名函数遮掩，即使基类和派生类中的两个函数的参数类型并不同，且不论是virtual还是non-virtural函数！

但这样就带来了疑问：这条规则和虚函数的动态调用如何衔接起来？

额，此规则似乎是就是重载的规则，动态调用更多的是重写或者覆写`override`

当然，具体调用顺序等规则还需要重新看下



18、纯虚函数 pure virtual

纯虚函数在抽象类中是可以有实现的，但是即使有实现，其**派生类中**仍然需要**强制对此纯虚函数做重新声明**，当然也可以有自己的实现

> 理解：根据派生类类似与镶嵌在基类的作用域中，如果派生类没有重新声明此纯虚函数，那么派生类本身也就变成了一个抽象类，那么就不允许抽象类实例化！！ 	

否则报错为：纯虚拟函数 "Base::func1()" 没有强制替代项





19、静态类型、动态类型、静态绑定、动态绑定

静态类型 static type：在程序中声明时的类型，即使是指针，不管它真正指向什么，静态类型在声明时就已经决定了！

动态类型 dynamic type：**目前**所指对象的类型，动态类型表现出一个对象将会有什么行为，且可以在运行中改变（通常经由赋值动作完成）



静态绑定 statically bound（前期绑定 early bound）：和静态类型类似，决定因素仅取决于对象的指针/引用声明类型，non-virtual 函数属于静态绑定

动态绑定 dynamically bound （后期绑定 late bound）：









20、vector的operator[]

Portable programs should never call this function with an argument n that is out of range, since this causes **undefined behavior**

vector不是map，如果用[]访问不存在的下标，那么将会导致未定义的行为！！

map是会直接为该key值使用默认构造函数构造一个对应的value值，the element is constructed using its default constructor

常识！！





21、左值与右值

右值可以赋给左值，左值同样可以赋给左值，左值有明确的内存位置，而右值只是一个临时变量，如表达式结果等均属于临时变量，即右值；

左值引用`int&`只接受左值，但是常量左值引用`const int&`可以接受右值，编译器会为这个临时变量分配内存；

> 左值引用使用时，很大可能会修改此值的某些属性值，而如果传入右值或者说临时变量，那么此修改就会作废！！所以编译器直接不允许，而如果是常量左值引用，那么说明不会修改传入的值，只是读取其中数据，所以编译器允许
>
> 暂时是这样理解的

右值引用`int&&`只接受右值