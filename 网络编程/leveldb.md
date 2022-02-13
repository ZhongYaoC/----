key-value键值对数据库

数据按照key值排序

提供put、get、delete基础操作

多个更改可在一个原子批处理中完成

数据支持向前和向后迭代（回滚和重跑吗？？）

可创建快照

使用Snappy压缩库对数据自动压缩

提供对外的抽象接口



同时只能单进程访问某特定数据库

无内置client-server支持





编译windows版本：

leveldb现在的最新版本实际提供了windows版本，不需要为了移植，重写大量函数；但是github中的windows版本编译是直接提供一个lib库，而我需要一个可以跟踪、单步调试的版，所以只能用github中的文件新建vs项目，调整文件结构，并删除所有benchmarks、测试用例文件（因为依赖于gtest库，而学习leveldb本身不需要用到gtest）、env_posix.cc等文件，同时预处理器中需要新增LEVELDB_PLATFORM_WINDOWS、OS_WIN；前者用于port.h中包含文件时，选择到windows版本（与老师所给的以前的leveldb源码对比，没有为windows版本单独编写头文件，port_windows.h被整合到port_stdcxx.h，POSIX和WINDOWS使用同一个头文件，具体实现待看，是怎么做到posix和windows同文件的，# define LINUX #else #endif这种吗），后者暂时未遇到

![leveldb env config](F:\Markdown\研一上\图片\leveldb env config.png)

![leveldb env config -- port var](F:\Markdown\研一上\图片\leveldb env config -- port var.png)



之后编译时，遇到错误：
链接器工具错误 `LNK2019  无法解析的外部符号 _main，该符号在函数 "int __cdecl invoke_main(void)" (?invoke_main@@YAHXZ) 中被引用`和LINK1120，最后面向搜索引擎解决-_-：在解决方案的属性-链接器-系统中，将子系统修改为控制台解决，此项目初始设置为窗口。

事后，看别的文章发现：除了之前提到的两个预处理器变量外，还可以将原本的`_WINDOWS`变量修改为`_CONSOLE`，应该也可以解决此报错

![leveldb env config -- link error](F:\Markdown\研一上\图片\leveldb env config -- link error.png)



随意写一个test，检查leveldb在windows下能否运行，运行成功，接下来开始具体的源码学习！！

![leveldb hello test](F:\Markdown\研一上\图片\leveldb hello test.png)



> private类型仅在类内和friend中可被访问，在类外定义的非成员函数的inline函数也不能直接访问private类型的数据
>
> ![private访问限制](F:\Markdown\研一上\图片\private访问限制.png)



参数设计上使用大量指针，如string*，避免值拷贝

**Q：**什么时候使用匿名namespace，作用？用途？？

**Q：**no_destructor构造函数为什么用右值引用

**Q：**aligned_storage的用法？？？内存对齐？？POD？？？

指定对齐格式，同时创建给定类型对象的非初始化内存块（对象未被初始化）；我的理解是aligned_storage创建出一个指定类型大小和对齐格式的内存空间，构造时只需要在该内存空间中初始化对象即可（placement new），类似于一种更加通用的设置对齐格式的方法，所以和模板搭配使用更方便。



> 可变参数函数模板
>
> 



> 内存对齐：
>
> 数据在内存中存放时首地址不是任意的，而是某个数值k的整数倍；
>
> 原因在于CPU在读写数据时，不是按照任意长度读写，而是按照某个固定值读写（是不是和CPU的数据总线根数有关？？），为提高读写效率，数据在内存中的首地址需要按照某种规则存放，为实现此可以填充某些空置位。所以属于又一典型的空间换时间操作！
>
> C++中不同编译器会有默认的对齐系数，`#pragma pack(n)`，其中n即为可自定义的对齐系数
>
> 对齐是和CPU相关的话，程序中乱自定义对齐系数会不会反而降低读写效率





>static的作用域
>
>全局静态变量、局部静态变量

