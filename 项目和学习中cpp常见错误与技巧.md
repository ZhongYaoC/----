##### 1、不允许使用不完整的类型

不完整类型：缺乏足够的信息，例如长度去描述一个完整的对象



（1）前向\前置声明 [^1]即为常见的不完整对象；

因为类作为前向声明时，文件中只能有该类的指针或者引用，而不是使用该类，使用需要完整的include！

根据google C++ style guide，尽量避免前置声明的使用，使用#include包含所需的头文件

（2）未知长度的数组

```int a[];```

（3）待搜集资料



##### 2、空悬指针 & 野指针

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



##### 3、 static成员的使用

static成员存放在数据段或bss段，而类对象存放在堆或者栈区；故static成员的生命周期为整个程序的生命周期

类内static可以修饰成员变量（常量、指针、引用、类类型等）和函数，可声明为public或者private

类的静态成员存在于任何对象之外，而静态成员函数同样不和任何对象绑定在一起，它们不包含this指针（显式和隐式均不包含），且静态成员函数不能声明为const

定义static成员函数时，可以在类内也可在类外，但static关键字只出现在类内声明处

通常static数据成员不应在类内初始化，因为static成员不是在创建类对象时被定义的，即不是由类的构造函数初始化（PS：在编译、链接期间已经分配，data、bss、common段）。static数据成员的初始化可在任意函数外面，不过一般是在cpp中，防止多次定义。

不过，可以在类内为static数据成员提供const整数类型（int、double、bool等）的类内初始化，但是如果需要此成员的引用或者地址或者因某些编译器原因，还是需要在类外部定义以下此成员用来提供地址，但仅定义不用指定初值



```C++
// .h声明文件

class Account
{
  private:
  	static double rate_; // 基准利率
 	static const int len = 10; // 整数类型的static常量可在类内初始化
  	// Or
  	static constexpr int len = 10;
  	double nums_[len]; // 从而可用在此类场景
  
  	std::string name_;
  	double amount_;
  
  private:
  	static double InitRate();
  public:
   	void calculate();
  	static double rate();
  	static void rate(double);
};


// .cpp定义文件

double Account::rate_ = InitRate(); // 类外定义

const int Account::len;
constexpr int Account::len;

double InitRate() // 定义时没有static关键字
{
  return 0;
}
```

##### 3、局部静态对象 local static object

函数内部定义的对象如果没有初始值，就会执行默认初始化，所以函数内部的staic对象如果没有初始值会执行默认初始化，而local static 和 其他local区别在于，一旦定义，它的生命周期将会直到程序终止。

```C++
// 个人感觉反直觉
size_t count_calls()
{
  static size_t ctr = 0;
  return ++ctr;
}

int main()
{
  // 将会输出 1 - 10 ，而不是每次都重新初始化！！
    //是的，编译器保证了local static只会被初始化一次
  for (size_t i = 0; i != 10; ++i)
  {
    cout << count_calls() << endl;
  }
  return 0;
}
```



##### 24、static的理解 

static代表静态，在类内定义了static，说明这个变量或者函数是归属于整个类的，不是各个类对象；

所以类内静态函数中不能调用类内变量，类内变量属于某个类对象，而不会是整个类，所以静态函数不能调用非静态成员！



20221207

首先需要明确static关键字在C语言时期即存在，主要存在两种全局static和局部static，二者均存放于data段或者bss段，具体段取决于是否做了初始化。而C++又在C基础上对static关键字做了扩充，使之可用于

* 全局static作用为标记符号为局部符号（不是局部变量，两个概念），即限制其作用范围，只能在当前模块中使用，不能被其他模块所引用，C++中同样继承了此概念，全局static的函数和全局static的变量是私有的。

各个模块中的全局static的初始化顺序随机，所以程序中不可以依赖于全局static已经存在！

* 局部static，即local static，它只会在第一次执行到该对象时初始化，之后不会做重复初始化（通过标记位实现），而虽为local，但是static加持下其生命周期为初始化后到程序结束为止，即延长了生命周期；同时解决了全局staic存在的初始化顺序未知问题！

C++11之前的local static多线程不安全，即如果一个线程执行到此对象初始化，但未初始化完成，之后另一个线程也执行到该语句，那么将会再次初始化，使得重复构造。

而C++11之后，此问题得到解决，C++11规定，在一个线程开始local static 对象的初始化后到完成初始化前，其他线程执行到这个local static对象的初始化语句就会等待，直到该local static 对象初始化完成。

局部static函数？？

* C++引入的static可用于修饰类内成员（函数/变量），作用为指明成员为全体对象共有，而非某一个对象！除此之外，并没有原本生命周期和作用域的概念，仅说明唯一性。

不过，除了local static之外，其他static都是在main执行之前得到初始化，析构是在main函数之后。



> 参考
>
> 《Effective C++》 
>
> [static初始化 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/406301228)
>
> [局部静态变量只能初始化一次？它是怎么实现的 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/87213810)





##### 4、慎用memcpy

```memcpy(void* dest, void* sourc, int len);```其中长度是由用户自己把控，就会出现len大于dest的内存长度，而且此时并不会报错！！



##### 5、网络间传输C++类对象时需要做序列化

因为单纯将对象指针信息放入数据包中传输，对端得到的信息中并不包含该对象的基类、继承关系等！



##### 6、string类型内似乎不需要管理转义字符这一说？？？



##### 7、char* char[]

char* a = "asdad";//代表指向一个静态存储区的字符串的指针，所以不可以修改

char a[][][] = "asdad";//代表在栈上的内存中存放的长度为6（5+1）的字符串，可以修改

char* arr = new char[6];//代表在堆上申请的6字节的内存空间

char* arr = new  char(6);//代表申请单字符的内存空间，并用'6'这个字符初始化，而非一维数组



##### 8、using声明和using指示

using声明可以使用，using指示最好不要使用！

区别：



##### 9、模板全特化和偏特化

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



##### 11、name lookup rules （C++中名称查找法则）/ arguement-dependent lookup（实参取决之查找法则）









##### 12、链接器工具错误 LNK2005

[链接器工具错误 LNK2005 | Microsoft Docs](https://docs.microsoft.com/zh-cn/cpp/error-messages/tool-errors/linker-tools-error-lnk2005?view=msvc-170)

符号被定义了多次。

此类错误常见有两种

1）当.h文件定义变量（似乎是全局变量）时，会出现此错误

.h内部直接定义（全局）函数也会出现此错误

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









##### 13、lambda

每个lambda都有它自己唯一的（未命名的）类类型，（是的，lambda是个类类型，而不是简单的函数），而它的捕获列表中的变量，便是这个类的成员变量！！！

```c++
//此处的mod就是一个未命名的函数对象类，其成员变量为空
auto mod = [](int i, int j){return i % j;}
```







##### 14、bind 和 function

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

##### 15、keyspace中的callable回调形式

##### 16、nmstl中的回调实现





##### 17、名称遮掩规则 name-hiding rules

只要名称相同即为遮掩，无所谓类型

而如果应用到继承中，那么只要派生类中有与基类同名的函数，那么派生类中的此函数就会将基类中的同名函数遮掩，即使基类和派生类中的两个函数的参数类型并不同，且不论是virtual还是non-virtural函数！

但这样就带来了疑问：这条规则和虚函数的动态调用如何衔接起来？

额，此规则似乎是就是重载的规则，动态调用更多的是重写或者覆写`override`

当然，具体调用顺序等规则还需要重新看下



##### 18、纯虚函数 pure virtual

纯虚函数在抽象类中是可以有实现的，但是即使有实现，其**派生类中**仍然需要**强制对此纯虚函数做重新声明**，当然也可以有自己的实现

> 理解：根据派生类类似与镶嵌在基类的作用域中，如果派生类没有重新声明此纯虚函数，那么派生类本身也就变成了一个抽象类，那么就不允许抽象类实例化！！ 	

否则报错为：纯虚拟函数 "Base::func1()" 没有强制替代项

纯虚函数的设计思路即为子类只继承其接口，实现需要子类自己去做。

纯虚函数在抽象类中的实现是无意义的，因为纯虚函数所在的类是不允许实例化的，之所以存在其实现主要发生在一种情况下：你希望拥有一个抽象类，但是没有合适的纯虚函数，而抽象类的设计初衷就是被作为基类，所以析构函数总是被设计为`virtual`（防止对象析构不充分），所以就将此析构函数设计为纯虚析构函数（pure virtual dtor），而为了使用多态特性的类的析构函数必须有实现，否则链接器将会报错，故将此纯虚析构函数做一份空实现。

```C++
class AWOV
{
  ...
  virtual ~AWOV() = 0;//声明
}

AWOV::~AWOV(){} //定义
```







##### 19、静态类型、动态类型、静态绑定、动态绑定

静态类型 static type：在程序中声明时的类型，即使是指针，不管它真正指向什么，静态类型在声明时就已经决定了！

动态类型 dynamic type：**目前**所指对象的类型，动态类型表现出一个对象将会有什么行为，且可以在运行中改变（通常经由赋值动作完成）



静态绑定 statically bound（前期绑定 early bound）：和静态类型类似，决定因素仅取决于对象的指针/引用声明类型，non-virtual 函数属于静态绑定

动态绑定 dynamically bound （后期绑定 late bound）：









##### 20、vector的operator[]

Portable programs should never call this function with an argument n that is out of range, since this causes **undefined behavior**

vector不是map，如果用[]访问不存在的下标，那么将会导致未定义的行为！！

map是会直接为该key值使用默认构造函数构造一个对应的value值，the element is constructed using its default constructor

常识！！





##### 21、左值与右值

右值可以赋给左值，左值同样可以赋给左值，左值有明确的内存位置，而右值只是一个临时变量，如表达式结果等均属于临时变量，即右值；

左值引用`int&`只接受左值，但是常量左值引用`const int&`可以接受右值，编译器会为这个临时变量分配内存；

> 左值引用使用时，很大可能会修改此值的某些属性值，而如果传入右值或者说临时变量，那么此修改就会在临时变量超出作用范围时作废！！所以编译器直接不允许，而如果是常量左值引用，那么说明不会修改传入的值，只是读取其中数据，所以编译器允许
>
> 暂时是这样理解的

右值引用`int&&`只接受右值



##### 22、std::move

std::move是将当前对象转变为右值，而右值是个临时对象，当调用其移动构造时，其资源会全部移动给另一个，本身资源清空，所以可以很方便，不用再对当前对象做clear

```c++
vector<int> level;
vector<vector<int>> res;
//若干操作后，level中值要全部放入res，而level本身要清空装别的
res.push_back(std::move(level));
//原本要写为
res.push_back(level);//此时level是左值，pushback内部会调用拷贝构造
level.clear();


//int等基础类型也一样
res.push(std::move(m));//m之后会被清空为初始值0；初始值是0吗？此处还待商榷
```



##### 23、指针、指针引用

写翻转二叉树题目，没有使用最直接的递归，而是外面套了一层函数，然后内部递归，居然出现了root的左右子节点在的确发生了交换后，又恢复了原状，而在直接原函数递归中没有此问题；参考了下评论中的一些解法，函数参数改用指针引用后解决问题；

```C++
// void Reverse(TreeNode *node1, TreeNode *node2) Error
void Reverse(TreeNode *&node1, TreeNode *&node2)
{
    if (!node1 && !node2)
    {
        return;
    }

    swap(node1, node2);
    if (node1)
    {
        Reverse(node1->left, node1->right);
    }
    
    if (node2)
    {
        Reverse(node2->left, node2->right);
    }
}

TreeNode *invertTree(TreeNode *root)
{
    if (!root)
    {
        return nullptr;
    }
    Reverse(root->left, root->right);
    return root;
}
```

原因就在于函数参数的值传递，一般传入指针是为了通过指针去修改指向的变量，而对于函数内使用的指针本身，其实是一个拷贝到的指针副本，你对于指针本身的操作就是在这个局部变量上操作，所以离开Reverse作用域之后，交换的指针就失效了。

swap为什么可以在Reverse作用域内对指针完成交换，而不是离开swap函数本身作用域就失效，因为swap是函数模板，其参数是T&，所以传入指针后，其参数变为`TreeNode*&`而不是指针。

指针引用`int* &`代表这个指针的别名，和普通引用没有区别，只不过这个别名是一个指针的，如此之后，对该指针的操作将不再是副本操作！！

也可以使用`int **`



##### 25、结构体对齐问题

为提高CPU运行速度等，编译器会对结构体做内存对齐，如

```C++
typedef struct
{
    uint64_t paxos_id;
    uint64_t node_id;
    uint32_t len; ////value长度
    char value[9];
} TagPaxosLearnValue;
```

如果按照直接计算内存长度，那么`sizeof(TagPaxosLearnValue) = sizeof(uint64_t) + sizeof(uint64_t) + sizeof(uint32_t) + 9 = 29` ，但是实际直接执行`sizeof(TagPaxosLearnValue)`得到的是32，在原长度基础上增加了3字节，这是因为编译器对此结构体做了8字节对齐，即 8 + 8 + 4 + （4 + 5） + 3；所以使用`sizeof(TagPaxosLearnValue)`需要非常注意对齐问题！

PS：一般对齐和CPU及OS有关，一般和指针大小一样，如`sizeof(int*)`是8，那么就是8字节对齐



##### 26、手动释放容器或string等的内存

有时，某容器可能在程序生命周期内一直存在，但是其内部的数据可能会清空，然后继续push_back，如此反复，就有可能导致容器的capacity一直会变大，即占用内存会变大，而即使数据清空后，capacity也不会降低，但是又没有析构，导致内存占用，所以需要手动将容器中的值清空。

* 利用swap一个空容器，而空容器本身是临时变量，交换得到内存后可以很快释放。

* C++11中提出了shrink_to_fit函数，可以根据当前数据量，自动调节内存占用，清理不用的capacity，但不是简单的将capacity()降低到size()；


由于内存的重新分配，会导致原来的迭代器和引用全部失效！！

以上针对string也同理，不过在string上使用swap会使迭代器、指针、或引用变为无效（？？？其他容器不会吗？除了string之外，的确不会。来自effective stl，**待查**）。

```c++
vector<string> chosen_values;

void ClearCache()
{
  //chosen_values.swap(vector<string>()); C11中似乎不允许，swap传入一个右值，会报错
  //Or
  vector<string> tmp_vec;//临时变量，会在函数结束时释放内存
  chosen_values.swap(tmp_vec);
}
```



##### 27、map的key是不可以修改的；unordered_map不支持pair作为key，需要手动增加hash函数

map的key是不可以修改的，所以`map<string, int>`的`value_ype`中对原有的key是常量

```C++
template<
class Key,
class T,
class compare = std::less<Key>,
class Allocator = std::allocator<std:pair<Key, T>>
>map;

key_type     ==>    Key // key
mapped_type  ==> 	T   //实际自认为的value
value_type   ==>	std::pair<const Key, T> // map取得的value是pair，其中key const !!!
```



##### 28、list迭代器

list的迭代器属于双向迭代器，所以不支持+n操作，所以不能够直接`list_.begin() + n`

##### 29、所谓迭代器失效问题

迭代器是为了方便容器内部的遍历而产生，指针就是一种迭代器，所以在某些时候，可以将指针的一些概念替换到迭代器中以方便理解，如所谓迭代器失效问题，理解为指针就是空悬指针，即指针指向的地址空间已经被释放或者未定义；那么再回到迭代器，就是迭代器指向的位置已经被释放，指向的地址已经不是你最初赋值时的那个含义或者干脆就是一块越界的内存区域。

而此类情况常常发生在容器的插入、删除操作之后，**删除之后一般那个迭代器就是失效了**（list不会），有时候就是对插删操作的返回值理解不透彻所导致的，如erase操作很多时候都返回删除元素的下一个位置，如果删除位置是容器末尾，那就返回end()，此时如果无脑`++iter`，就会跳过后一个元素直接到第二个元素位置。

map中同样存在此问题，而且有些stl实现的map容器的erase设计是返回null，即不返回被删除元素的后一个迭代器，此时情况也不同。

vector是高效的动态数组，当其`capacity`满时会触发扩容，一般新的capacity是原有的两倍，会在内存中重新开辟一块区域并把原本的元素拷贝进去，但是**扩容前的迭代器将会全部失效**，因为他们相当于指针（其实vector的迭代器就是原生指针），指向着旧的内存区域，如capacity满之后，在iter处插入num，插入成功后，在调用此iter将导致未定义的行为。

list的迭代器不会出现失效问题，它是双向链表设计。



所以对于所使用的**STL实现**，**删除或者插入操作的具体实现**，**返回值**都需要深入理解。

```C++
vector<int> nums{1, 2, 4, 6, 7};

vector<int>::iterator iter = nums.begin();
//删除nums中的偶数
while (iter != nums.end())
{
  if (*iter % 2 == 0)
  {
    iter = nums.erase(iter);// 此时的iter已经指向被删除元素的后一个元素了
  }
  else// 如果没有这个else，而是全部都++iter，将会跳过后一个元素！!
  {
    ++iter;
  }
}


// example 2
// 如果map的erase不返回(当前使用的map实现似乎是返回下一个元素的iterator)
for (pos = coll.begin(); pos != coll.end(); )
{
  if (pos.second == val)
  {
    coll.erase(pos++);//erase时传入pos迭代器的副本，pos本身直接指向后一个元素
    //如果不处理，将会导致pos迭代器失效，之后哪怕是只操作 ++pos
    //也会导致未定义的行为undefined behaviour
  }
  else
  {
    ++pos;
  }
}
```





##### 30、中文字符串的size问题

```C++
std::string str("中文小测试");
std::cout << str.size() << std::endl;
// 将会输出15，即一个中文字符占3字节
// 而查看 cpp reference，size / length函数的返回为
// Returns the number of CharT elements in the string
// string ==> basic_string<char>
// 即返回字节数，而不是字符数量
std::cout << str[0] << std::endl;
// 输出也会错误，它只会输出第一个字节，而第一个字节根本不是中文，只是“中”的一部分
const char* cstr = str.c_str();
// 可以看出每个字符的字节组成！




// 使用char32_t组成的string
// size()输出将是正确的
// 但是str[]将会输出第一个char32_t的int值，因为ASCII编码只适合单字节的char
std::u32string str(U"中文测试");
std::cout << str.size() << std::endl;
std::cout << str[0] << std::endl;

```

在某些涉及逐字节拷贝的场景中可能出现错误



##### 31、匿名namespace

作用：匿名namespace中的所有变量和函数等相当于所在的编译单元（.cpp）而言是私有的，即其他文件不可见！相当于C语言中的static的函数和**全局**变量（不是local static）--符号仅在本模块可见，不能被其他模块引用，故鼓励使用匿名namespace替换文件生存空间file scope中的static。



**Ps**：C++中针对static有新的含义，即static出现在类内成员，说明此成员归属于全体类，而不是某一个具体对象，同时具有唯一性。**并不会限定作用域或者延长生命周期**。





##### 32、定义已经存在的变量？重复定义？

不同作用域下，当然可以，不过会出现变量遮掩，但是同一作用域下，不可以重复定义；

但是如local static变量，多次调用此函数时，按理来说第一次执行到此对象时即被初始化，之后调用相当于重复定义，但是并没有报错，相反实际执行时，定义此变量的语句被跳过；~~是因为local static存放在data或者bss段，不在乎作用域？（因为其生命周期为初始化后到程序结束）。~~

> 记住local static只会初始化一次，语法规定。
>
> 在初始化local static后会设置一个标记位，之后再次执行此语句前通过位计算，检查标记位，如果已经设置则不会再初始化，否则初始化。

同样两层for循环，中间的变量定义，按理来说内部for循环结束进入外部for循环的下一次迭代后，会出现变量的重复定义，但是也没有报错，之前定义出的变量作用域结束了？但是应该没有退出外部循环的作用域。。。

> 这个只能强行理解为特例了，反正也经常这么写。



##### 33、unique()

unique不是无脑去重复，而是将相邻的重复项放到容器末尾，最后返回已经去掉相邻重复后的新元素集合的end()；所以有时候会先排序，在做unique

```C++
std::vector<int> v{1, 2, 1, 1, 3, 3, 3, 4, 5, 4};
    auto print = [&] (int id)
    {
        std::cout << "@" << id << ": ";
        for (int i : v)
            std::cout << i << ' ';
        std::cout << '\n';
    };
    print(1);
 
    // remove consecutive (adjacent) duplicates
    auto last = std::unique(v.begin(), v.end());
    // v now holds {1 2 1 3 4 5 4 x x x}, where 'x' is indeterminate
    v.erase(last, v.end());
    print(2);
```





##### 34、二义性的C++语句

针对存在二义性的C++语句，只要它有可能被解释成函数声明，编译器就肯定将其解释成函数声明！！

```C++
// 前情提要：class_a类中重载了operator() 
// 以完成thread接受所有可调用类型的要求

std::thread t(class_a());

// 本意为thread类中传入一个class_a的临时对象
// 但是此语法和函数声明极其类似
// 编译器会将其解释成函数声明，返回一个thread类型，名为t，参数为函数指针，该函数指针没有参数！

// std::thread (*)(class_a (*)())

// 推荐用法
std::thread t((class_a()));
// 或者使用thread的列表初始化
std::thread t{class_a()};
```

