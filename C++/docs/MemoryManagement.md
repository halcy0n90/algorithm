# C++内存管理

## 参考资料

#### **[vedio](https://www.bilibili.com/video/BV1Kb411B7N8?from=search&seid=2095136970590855889)**

#### **Libraries:**

* STL Allocators
* MFC CPlex+CFixedAlloc
* Boost.Pool
* Loki SmallObjAllocator
* VC malloc/free
* jemalloc
* tcmalloc

**向侯捷老师致敬！**

## 编译环境

> * `macOS Mojave/Clang 9.0.0`
> * `Ubuntu 18.04/GCC 7.5.0`
>
> 主力机器是`MAC`，必要时为了模拟`GNU`环境会使用`Ubuntu`来测试。  
>
> 如果没有特别标注`GCC`,则全部使用`Clang`编译并取得和`GNU`平台上逻辑相同的输出。需要说明的是`LLVM`和`GCC`渐行渐远，为`Clang`自行开发的`lib++`已经取代了`GCC`的标准库`libstd++`，导致不同的编译器最终的结果会有些许不同。

## index

### 1. primitives

| 分配                     | 释放                         | 类型      | 可否重载                     |
| ------------------------ | ---------------------------- | --------- | ---------------------------- |
| `malloc`                 | `free`                       | C函数     | 不可                         |
| `new`                    | `delete`                     | C++表达式 | 不可                         |
| `::operator new`         | `::operator delete`          | C++函数   | 可                           |
| `allocator<T>::allocate` | `allocator<int>::deallocate` | C++标准库 | 可自由设计并以之搭配任何容器 |

* [绪论](../src/MemoryManagement/0primitives_index.cpp)
* [new/delete探究](../src/MemoryManagement/1primitives_new_delete.cpp)
* [数组new/delete探究](../src/MemoryManagement/2primitives_array_new_delete.cpp)
* [placement_new探究](../src/MemoryManagement/3primitives_replacement_new.cpp)
* [operator_new重载](../src/MemoryManagement/4override_operator_new.cpp)
* [placement_new重载](../src/MemoryManagement/5override_placement_new.cpp)
* [class allocator_1.0对象内存池](../src/MemoryManagement/6per_class_allocator.cpp)
  * `GCC`: 重载`operator new/delete`间隔为`16Bytes`,不重载的间隔`32Bytes`
  * `Clang`: 都是`16Bytes`
* [class allocator 2.0对象内存池](../src/MemoryManagement/6per_class_allocator_advanced.cpp)
* [static allocator 3.0对象内存池](../src/MemoryManagement/7static_allocator.cpp)
* [macro static allocator 4.0对象内存池](../src/MemoryManagement/8macro_static_allocator.cpp)
* [new handler/default/delete探究](../src/MemoryManagement/9end.cpp)

 

### 2. sd::allocator

> **VC6.0**和**BorlandC5.0**这两个著名的编译器标准库内部实现中，每个容器的`std::allocater`都是通过`::operator new/delete`来完成的——**本质就是直接调用`malloc`其他什么都没有做**。而`GNU2.9 C++`中使用的版本是`std::alloc`——使用了诸如**pool**等高级的分配逻辑，在经过更新迭代后在`GNU4.9`中改名为`__gnu_cxx::__pool_alloc`——也是本章的重点关注对象。
>
> 各个编译器标准库版本的容器举例如下:

```C++
//VC6
template<class _Ty,
class _A= allocator<_Ty>>
class vector{  
  //...
};             


//BC5
template<class T, 
	class Allocator=allocator<T>>
class vector{
  //... 
};


//GNU C++
template<class T, 
	class Alloc = alloc>
class vector{
  //...  
};            
```

* [GNU2.9-4.9对cookie的优化](../src/MemoryManagement/10pool_alloc.cpp)
  * `GCC2.9-std::allocator`: 连续申请三个`8Bytes-double`的元素，地址间隔`32Bytes`
  * `GCC4.9-__gnu_cxx::__pool_alloc`:连续申请三个`8Bytes-double`的元素，地址间隔`16Bytes`

> ` GNU2.9`中的`std::alloc`内存分配流程：
>
> * `size`容器所申请的对象大小；`chunk`分配器提供的内存单元——共16种。
>* 维护16条长度各异(第`n`条链表一次分配的`chunk`大小为:`8*(n + 1)Bytes`，)的自由链表，超过能分配最大的内存单元会申请`malloc`。**所有使用`malloc`使用的都会带`cookie`**。
> * 分配器会将申请的内存`size`按照分配单元调整到合适的边界(比如`std::alloc`是8的倍数，`malloc`也会调整)
>* 拿到一个申请的`size`需要分配的对象，首先定位它在分配器上的指针端口。然后查看是否有**战备池**的内存可以用，如果有直接使用，这时候一般分配的数量是1~20个`chunk`。没有则调用`malloc`分配。第一次分配**pool**为零，则直接调用`malloc`。
> * 分配器调用`malloc`分配大小为`size * 20 * 2 +ROUNDUP(memory_allocated >> 4);  `,其中会有`20`个`chunk`是为申请的对象准备的，剩下的内存(并不会分割)作为**战备池**为以后的分配器行为做准备——这是一个很重要的概念，分配全程需要关注**战备区pool**的容量。 其中`ROUNDUP()`是对齐函数，`memory_allocated`是已经分配好的内存总量。随着内存分配操作越来越多，**memory_allocated**也是越来越大——符合计算机内存分配越来越多的增长趋势。
> * 如果**战备池**剩下的内存小于`size`，则被看成是碎片——需要重新挂到相应的链表头指针上——寻找相应的`chunk`端口。
> * 如果`malloc`系统内存分配失败，则需要在最接近`size`的已分配好的空闲`chunk`中回填`pool`，完成分配。如果向上走也无法分配成功，那系统分配才算完全失败。
> * 由于一次性分配20个`chunk`，而每一次分配必须按照其`size`来选择链表头结点，所以有很大概率某些指针上的空闲`chunk`就会比较多。而`std::alloc`对`chunk`的选择一定是大于等于`size`的。从技术的角度，将这些空闲的`chunk`合并在一起难度非常大。
> * 分配器面向的用户是**容器**——元素一样大的集合，如果用户直接使用分配器，就必须记住分配的大小。这是因为自主分配的内存不会带`cookie`，而容器的第一个目标参数都会是参数类型，只要`sizeof`一下就可以计算出来所申请的`size`。
> * `embedded pointers`是成熟的工业级分配器都会使用的`Tips`。

* [GNU2.9的std::alloc源码实现](../src/MemoryManagement/11std::alloc.cpp)

> 如果只是为了理解内存分配，其实很多代码都没有用。

* [GNU2.9的std::alloc源码实现简易版](../src/MemoryManagement/11simplified_ver_std::alloc.cpp)
* [__pool_alloc新型的分配器测试程序](../src/MemoryManagement/12alloc_test.cpp)
  
  * `GCC`： 测试样例`1000`个元素。
    * 标准分配器: 分配`1000`次，得到`24000`个字节
  * 新型分配器: 分配`16`次，得到`25224`个字节
  
  

### 3. malloc/free

> `Windows`下`VC6.0`中`C++`程序的流程(简化版)——伪码中的`API`都是`Windows`平台下的。**执行流程是前一个函数调用后一个函数。**
>
> * `KERNEL32! bff89f5b()...`   内核函数
>
> * `mainCRTStartup()`   由`C Runtime Library`  初始化
>
>   * `_heap_init()`     分配了16个`HEADER`
>
>     [_heap_init部分代码](../src/MemoryManagement/13heap_init.cpp)
>
>     * `__sbh_heap_init()`      
>
>       [__sbh_heap_init部分代码](../src/MemoryManagement/13sbh_heap_init.cpp)
>
>   * `_ioinit()`  第一次尝试分配内存
>
>     [_ioinit部分代码](../src/MemoryManagement/13ioinit.cpp)
>
>     * `_heap_alloc_dbg()`
>
>       [_heap_alloc_dbg部分代码](../src/MemoryManagement/13heap_alloc_dbg.cpp)
>
>     * `_heap_alloc_base()`
>
>       [_heap_alloc_base部分代码](../src/MemoryManagement/13heap_alloc_base.cpp)
>
>     * `_sbh_alloc_block()`
>
>       [_sbh_alloc_block部分代码](../src/MemoryManagement/13sbh_alloc_block.cpp)
>
>       
>
>       
>
>     















### 4. Loki::allocator





### 5. Other allocators

 






