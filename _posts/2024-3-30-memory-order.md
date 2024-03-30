# 背景
首先让我们回顾下代码是如何在计算机上面运行的，以下图为例：

![image-20220804112545147](D:\doc\memory\image-20220804112545147.png)

一般来说经过编译器，处理器，Cache三个阶段都会进行<u>**优化**</u>，而且优化手段也越来越丰富，越来越复杂，你无法确定到底是哪一层在优化。
![优化流程.png](优化流程.png)

这样就导致开发者看到的代码与机器实际执行的程序存在差异，单线程时运行结果能够保持一致，但是在多线程环境下，情况就变得复杂而微妙了。
这些差异基本来自三个阶段：

1. 编译器优化
2. CPU重排优化
3. Cache一致性

一般来说每个阶段都会进行优化，优化方法有Reorder, Invent, Remove

# 优化

## 编译器优化

编译器发展到今天，鉴于优化手段过于丰富，这里简单列举常见的优化方法，抛砖引玉而已：

+ 函数内联(inlining)  : 编译器将函数调用替换为函数体本身   
+ 常量折叠(constant folding) : 编译器将编译期能计算为常量的表达式直接替换为计算结果
+ 常量传播(constant propagation) : 编译器追踪到一个值的源头，发现它是常量后，会将所有地方出现的这个值替换为常量
+ 公共子表达式消除(common subexpression elimination) : 将重复的计算过程重写掉，只算一次，其它地方复制结果
+ 移除死代码(dead code removal) : 这里包含了对没用到的值的读写操作，以及完全没用到的整个函数或表达式

帮助编译器进行优化的要点就是保证它能获得尽可能多地信息，从而做出正确的优化决定。其中一个信息源就是你的代码：编译器能看到的代码越多，能做的决定越优。另一个信息源是你用的编译器配置：告诉编译器准确的目标CPU架构就能带来大不同。当然，编译器拥有的信息越多，编译时间越长，因此这里还要保持平衡.

以GCC为例，可以通过编译选项(-O)来控制优化，详情参考：https://gcc.gnu.org/onlinedocs/gcc-7.5.0/gcc/Optimize-Options.html#Optimize-Options



## 内存重排(Memory Reorder)

编译器和CPU都会有重排，主要就是为了优化性能，下面就以编译器优化为例，看一段代码
```c++
int A=0, B=0;
void foo()
{
    A = B + 1; //L4
    B = 1;     //L5 
}
```
如果不开启优化编译，执行顺序是按照代码顺序来（先执行**L4**, 再执行**L5**), 这种顺序关系有个专门术语，叫做**sequenced-before**，后面会详细解释。
开启编译器优化选项进行编译，以gcc为例, **-O**参数提供了很多优化选项，参考[Optimize-Options](https://gcc.gnu.org/onlinedocs/gcc-4.8.2/gcc/Optimize-Options.html#Optimize-Options)

```cpp
// g++ -O2 -S test.cpp
movl  B(%rip), %eax
movl  $1, B(%rip)          // B = 1
addl  $1, %eax             // A = B + 1
movl  %eax, A(%rip)
```
可以看到优化后的汇编代码先执行**L5**，再执行**L4**。尽管代码执行顺序发生了变化，但是最终结果与重排前是一致的。这样做有什么好处呢，参考前文提到的Cache，其实Cache读写是以Cache Line为单位，即一块内存区域，大小一般为32、64、128字节。这样做能提高访存效率。在单线程环境下，由于有编译器的保证，这样的指令乱序没有任何问题。但是多线程运行的时候，内存重排会导致意想不到的结果。

再看另外一段多线程代码
```cpp
X = 0, Y = 0;

Thread 1: 
X = 1;
r1 = Y;

Thread 2: 
Y = 1;
r2 = X;
```
线程1执行代码的顺序与线程2看到的不一样，反过来也是如此，最终的结果就是r1, r2的值不确定。
内存重排的根本原则就是：**不能修改单线程的行为** [](https://preshing.com/20120625/memory-ordering-at-compile-time/)
当然这种编译器重排的优化，可以使用预编译指令来控制，告诉编译器不能优化。
Memory Barrier/Fence就是一类指令来保证内存读写顺序一致性。
比如GCC可用**asm volatile**，
```cpp
void foo()
{
    A = B + 1;
    asm volatile("" ::: "memory")
    B = 1;
}
```
## Invention
涉及到**Speculative execution** [](https://en.wikipedia.org/wiki/Speculative_execution)
现代流水线微处理器使用推测执行来降低条件分支指令的成本，使用基于分支执行历史预测程序执行路径的方案。这就是通常所说的**分支预测**优化(Branch Prediction[](https://en.wikipedia.org/wiki/Branch_predictor))。
参考伪代码[](https://people.cs.pitt.edu/~xianeizhang/notes/cpp11_mem.html)：

```c
// x is shared var
if( cond ) x = 42;

// cond is speculated to be true, rewrite code
r1 = x;    // read what's there
x = 42;    // oops: optimistic write is not conditional
if( !cond) // check if we guessed wrong
    x = r1;// oops: back-out write is not SC
```


##  Out-of-order execution

除了编译器会进行memory reorder, CPU也会有不同的reorder偏好。参考WIKI[](https://en.wikipedia.org/wiki/Memory_ordering)上面的统计表格：
![memory_order_on_cpu.png](memory_order_on_cpu.png)

## 缓存一致性(Cache Coherency)
先看下现代计算机CPU/RAM架构，至于为什么引入这么多级Cache
![Modern Hardware.png](Modern Hardware.png)

先看一组来自Jeff Dean的数据，可以看到Cache对性能的重要性，Cache的存在是为了弥补CPU与RAM之间运算速度的差异。

![latency_numbers.png](numbers.png)
但是会引入一个问题，不同Cache间如何保证数据一致性。

综上所述，CPU执行结果与我们预期的并不一致。甚至，对于同一次执行，不同线程感知到其他线程的执行顺序可能都是不一样的。
因此内存模型需要考虑到所有这些细节，以便让开发者可以精确控制。因为所有未定义的行为(Undefined Behavior)都可能产生问题。


# 关系术语
## sequenced-before
是一种在**同一线程**里面的求值间的**非对称**，**可传递**，**成对**的关系("sequenced-before" is an asymmetric, transitive, pair-wise relationship between evaluations within the same thread)

具体内容可以看下order of evaluation.
表达式部分的求值顺序，包含函数参数的求值顺序都是不确定的，除了一些例外。

```c
 
int a() { return std::puts("a"); }
int b() { return std::puts("b"); }
int c() { return std::puts("c"); }
 
void z(int, int, int) {}
 
int main()
{
    z(a(), b(), c());       // all 6 permutations of output are allowed
    return a() + b() + c(); // all 6 permutations of output are allowed
}
```
可能的输出：
```c
b
c
a
c
a 
b
```

sequenced-before对求值顺序有如下的描述：

* 如果A sequenced-before B，代表A的求值在B之前会先完成
  
* 如果A not sequenced-before B，而B sequenced-before A，则代表先对B进行求值，然后对A进行求值
  
* 如果A not sequenced-before B，而B not sequenced-before A，则A和B的执行顺序不确定，甚至可以同时执行（一条CPU指令处理A和B）

具体哪些场景算是sequenced-before, 一共21条规则(Rules), 详见cppreference[](https://en.cppreference.com/w/cpp/language/eval_order)，这里不再赘述。

## happens-before
happens-before 是一种现代计算机科学的术语，用来描述软件内存模型，C++11, JAVA, GO 甚至LLVM都用到。
定义：让A和B表示被不同线程执行的操作。如果A **happens-before** B，那么A内存状态对于另一个线程执行B之前是可见的。
happens-before关系是sequenced-before关系的扩展，因为它还包含了不同线程之间的关系，同样的，这是一个非对称，可传递的关系。如果A happens-before B，B happens-before C。则可推导出A happens-before C。
看下`cppreference`对happens-before关系的定义：
```c
Regardless of threads, evaluation A happens-before evaluation B if any of the following is true:

1) A is sequenced-before B

2) A inter-thread happens before B
```
从上述定义可以看出，happens-before包含两种情况，一种是同一线程内的happens-before关系(等同于sequenced-before)，另一种是不同线程间的happens-before关系。

对于同一线程内的happens-before，其等同于sequenced-before，所以在此忽略，着重讲下线程间的happens-before关系。

**Inter-thread happens-before**   以下5种情况，任何一项是true，就认为是线程间的happens-before
```c
Between threads, evaluation A inter-thread happens before evaluation B if any of the following is true

1) A synchronizes-with B
2) A is dependency-ordered before B
3) A synchronizes-with some evaluation X, and X is sequenced-before B
4) A is sequenced-before some evaluation X, and X inter-thread happens-before B
5) A inter-thread happens-before some evaluation X, and X inter-thread happens-before B
```
有一点要注意的是happens-before并不表示happening-before, 比如说指令重排。
1.  A _happens-before_ B does not imply A happening before B.
2.  A happening before B does not imply A _happens-before_ B.

## synchronizes-with
synchronizes-with描述的是不同线程间的同步关系, 一个线程内操作的内存状态保证在另一个线程内是可见的(visible)。同时也是一种happens-before关系。
C++原子操作acquire-release能够达成synchronizes-with关系。还有其他方式也能够达成，具体看下图[](https://preshing.com/20130823/the-synchronizes-with-relation/)：

![synchronizes-with.png](synchronizes-with.png)

# 原子类型与原子操作
在介绍内存模型之前，先了解下C++11提供的原子类型（atomic types）和原子操作（atomic operation）。
## 原子类型
原子类型是模板类，每一种实例化定义了一种原子类型。多线程下原子对象的读写是线程安全的（Behavior well-defined)。
下面是基本类型对应的原子类型[](https://en.cppreference.com/w/cpp/atomic/atomic):
![atomic_type.png](atomic_type.png)

除了基本类型的原子类型，对应的头文件是 atomic； 还有指针类型的原子类型，对应的头文件是 memory。
`std::atomic` is neither copyable nor movable.

## 原子操作
顾名思义，该操作要么是执行完了，要么是没有执行，从任何一个线程中，都无法观察到中间状态。比如说，如果有一个线程进行了原子写操作，其他线程有原子读操作，那么其获取到要么是修改前的值，要么是修改后的值，不会是中间状态的值。
不同的原子类型有不同的操作，如下表：
![atomic_operations.png](atomic_operations.png "atomic_operations.png")

**std::atomic_flag**
是原子bool类型，并且保证是lock-free。与atomic bool不同，atomic_flag 不提供load 和store操作。

**is_lock_free**  
此函数用来查询原子变量是否是lock-free，与具体平台实现有关。

**load, store, exchange**  
load: 原子操作读取原子变量的值
store: 原子操作写入原子变量的值
exchange: 原子地写入原子变量的新值，并返回其先前持有的旧值。

**compare_exchange_weak**， **compare_exchange_strong**  

```
bool compare_exchange_weak( T& expected, T desired,
                            std::memory_order success,
                            std::memory_order failure ) noexcept;
                            
bool compare_exchange_strong( T& expected, T desired,
                              std::memory_order success,
                              std::memory_order failure ) noexcept;                      
```

都是CAS操作，它们都接受两个输入值：`T& expected`和`T desired`。这两个方法会比较原子变量实际值和所提供的预计值（`expected`），如果两者相等，则更新原子变量值为提供的期望值（`desired`）(执行read-modify-write操作）。否则，读取*this当前的实际值去更新expected(执行load 操作）。如果原子变量值发生了变化，则返回`true`，否则返回`false`。

不同之处：在一些缺少单个比较交换指令（CAS）的机器上面，会出现 **Fail Spuriously**。即使原始值等于预计值（`expected`），`compare_exchange_weak`也可能失败并返回`false`。
**C++ Concurrency In Action**书中建议在loop里面使用它。 

![weak.png](weak.png)


**指针原子类型**  
类型定义：
```cpp
template<class T> struct atomic<T*>
```
除了常规的原子操作，还提供了其他的原子操作，例如 *fetch_add*, *fetch_sub*，需要注意的是，这些函数返回的值是原子对象修改之前的。但是原子操作 *+=*， *-=*，返回的值是操作之后的值。

**原子操作的C接口函数**  
除了那些原子类型内部的成员函数外，C++11还提供了一系列C接口的原子操作，这里不再一一列举。支持原子对象以指针参数输入。
看下函数atomic_fetch_add的声明：
```cpp
template<class T>
T atomic_fetch_add(std::atomic<T>* obj,  typename std::atomic<T>::difference_type arg)
```

# Memory Order
所有原子操作都有一个类型是[std::memory_order](http://en.cppreference.com/w/cpp/atomic/memory_order) 的可选参数，其定义如下：
```c
typedef enum memory_order {
    memory_order_relaxed,
    memory_order_consume,
    memory_order_acquire,
    memory_order_release,
    memory_order_acq_rel,
    memory_order_seq_cst
} memory_order;
```
**std::memory_order** 指定如何进行内存访问，包含常规的，非原子地内存访问，都会围绕一个原子操作进行重排。不要任何限制在多核系统上面，当多个线程同时读写变量，读线程里面看到值变化的顺序与写线程对变量进行写操作的顺序不一致。甚至，在不同读线程里面看到值变化的顺序都不一样。由于内存模型允许的编译器优化，在单处理器系统上面也有类似的效果。
在C++库中所有原子操作的默认行为提供的是内存顺序一致性（sequentially consistent ordering)。这种默认行为会损害性能，库的原子操作也会提供额外的参数（std::memory_order）来指定精确的内存访问限制，不仅仅是原子性，是让编译器和处理器来保证那个操作。

首先，每一种内存顺序约束对应不同的原子操作，可以根据读（Read），写（Write），读-修改-写（Read-Modify-Write)来进行分类：

| Operation               | Read | Write | Read-Modify-Write |
|-------------------------|:----:|:-----:|:-----------------:|
| test_and_set            |      |       | Y                 |
| clear                   |      | Y     |                   |
| is_lock_free            | Y    |       |                   |
| load                    | Y    |       |                   |
| store                   |      | Y     |                   |
| exchange                |      |       | Y                 |
| compare_exchange_strong |      |       | Y                 |
| fetch_add, +=           |      |       | Y                 |
| ++, --                  |      |       | Y                 |
| fetch_and, &=           |      |       | Y                 |

相应的内存顺序也可以归类如下：

| Operation         | Memory Order                     |
|:-----------------:|:--------------------------------:|
| Read              | relaxed, consume,acquire,seq_cst |
| Write             | relaxed, release,seq_cst         |
| Read-Modify-Write | relaxed, acq_rel, seq_cst        |

比如说，load原子操作是一个Read操作，可以指定memory_order_acquire, memory_order_relaxed, 而指定内存顺序为memory_order_release是没有意义的。

至此，引出本文的主题**内存模型**，这些不同内存顺序引出了不同内存模型，按模型强度从强到弱分为以下三种：

* Sequential consistency 模型：顺序一致性，seq_cst
* Acquire and Release模型：获取和释放，acq_rel
* Relaxed模型：松散模型

这其中，后两种只有C++内存模型可以提供，其他编程语言例如Java或者C# 中均没有。

# 内存模型
## Sequential consistency 模型
Seq-cst 模型即顺序一致性模型，这是最严格的内存模型。所有原子操作的默认内存顺序就是memory_order_seq_cst。
最早追溯到Leslie Lamport在**1979**年**9**月发表的论文《**How to Make a Multiprocessor Computer That Correctly Executes Multiprocess Programs**》，在该文里面首次提出了`Sequential consistency`概念：
```c
the result of any execution is the same as if the operations of all the processors were executed in some sequential order, and the operations of each individual processor appear in this sequence in the order specified by its program.
```
根据这个定义，在顺序一致性模型下，程序的执行顺序与代码顺序严格一致，也就是说，在顺序一致性模型中，不存在指令乱序。
有两个保证：
1. 程序指令与源码顺序是一致的
2. 所有线程的所有操作存在一个全局的顺序
这种模型对性能有损失，为了实现顺序一致性要添加很多手段来对抗编译器和处理器的优化。

举个例子 ：

```cpp
std::atomic<int> X(0), Y(0);
int r1, r2;

void thread1()
{
    X.store(1); //①
    r1 = Y.load(); //②
}

void thread2()
{
    Y.store(1); //③
    r2 = X.load(); //④
}
```
不管如何，运行结果都不会出现r1=r2=0。
让我们来看下为什么， 尽管可能存在以下多种执行顺序：

* ① ② ③ ④
* ① ③ ② ④
* ① ③ ④ ②
* ③ ④ ① ② 
* ③ ① ② ④ 
* ③ ① ④ ②  

无论如何这个顺序是得到保证的：① → ②,  ③ → ④。 
但是从每个线程角度来看，指令执行顺序与代码顺序是一致的。为了这个结果，编译器默默增加了一些特殊指令，比如说memory fence。下面是上面程序的汇编代码（编译器GCC7.5, 左边是x86-64， 右边是aarch64)
![seq_cst_sample.png](seq_cst_sample.png)

## Acquire and Release 模型
首先看下**Jeff Preshing**对术语acquire 和 release语义的定义：

+ **Acquire semantics**  is a property that can only apply to operations that **read** from shared memory, whether they are [read-modify-write](http://preshing.com/20120612/an-introduction-to-lock-free-programming#atomic-rmw) operations or plain loads. The operation is then considered a **read-acquire**. Acquire semantics prevent memory reordering of the read-acquire with any read or write operation that **follows** it in program order.
禁止在read-acquire之后的任何读写操作的内存重排。

+ **Release semantics** is a property that can only apply to operations that **write** to shared memory, whether they are read-modify-write operations or plain stores. The operation is then considered a **write-release**. Release semantics prevent memory reordering of the write-release with any read or write operation that **precedes** it in program order.
禁止在write-release之前的任何读写操作的内存重排。

然后让我们回顾下几种内存顺序，memory_order_acquire(读操作）, memory_order_release（写操作）,memory_order_acq_rel（读-修改-写）。
提供了打破顺序一致性的操作，同一个原子变量的acquire和release操作会引入synchronizes-with关系。

对于memory_order_acq_rel， cppreference是如此解释的：
```c
A read-modify-write operation with this memory order is both an _acquire operation_ and a _release operation_. No memory reads or writes in the current thread can be reordered before the load, nor after the store. All writes in other threads that release the same atomic variable are visible before the modification and the modification is visible in other threads that acquire the same atomic variable.
```
看下代码示例：
```cpp
std::atomic<bool> x,y;
int z = 0;

void thread1()
{
    x.store(true, std::memory_order_relaxed); // (1)
    y.store(true, std::memory_order_release); // (2)
}

void thread2()
{
    while(!y.load(std::memory_order_acquire)); // (3)
    if(x.load(std::memory_order_relaxed))      // (4)
        ++z;
}

int main()
{
    x=false;
    y=false;
    z=0;
    std::thread a(thread1);
    std::thread b(thread2);
    a.join();
    b.join();
    assert(z != 0); // (5)
}
```
这段代码(5)，永远不会触发assert。为什么呢？
首先，(2) memory_order_release与(3) memory_order_acquire构成了线程间synchronize-with关系，所以(2) happens before (3)
其次，(1) happens before (2)，(3) happens before (4)
最后，线程join也会构成synchronize-with关系，(4) happens before (5)
程序最终的执行顺序就变成了(1)->(2)->(3)->(4)->(5)， 变量z肯定不等于0。


## Relaxed 模型
Relaxed模型是memory_order的memory_order_relaxed, 是最弱的一种内存模型。只保证当前的内存访问是原子操作，其他的都不保证，这就会有memory reorder。

还以上面代码为例：
```cpp
std::atomic<int> X(0), Y(0);
int r1, r2;

void thread1()
{
    X.store(1, std::memory_order_relaxed); //(1)
    r1 = Y.load(std::memory_order_relaxed); //(2)
}

void thread2()
{
    Y.store(1, std::memory_order_relaxed); //(3)
    r2 = X.load(std::memory_order_relaxed); //(4)
}
```
由于无法保证顺序一致性，运行结果就会存在r1=r2=0， 也就是执行顺序是 (2) ->(4) ->(1) ->(3)
那么既然memory_order_relaxed无法保证执行顺序，它会应用到什么场景呢？ 考虑到其特性是**只保证当前的数据访问是原子操作**，
比较典型的应用场景是用来做统计计数。
```cpp
std::atomic<long long> data_cnt {0};

void foo()
{
    data.fetch_add(1, std::memory_order_relaxed);
}

int main()
{
    std::thread t1(foo);
    std::thread t2(foo);
    std::thread t3(foo);
    std::thread t4(foo);
    std::thread t5(foo);
}
```
这样做有一个好处是性能开销相比其他的内存模型最小。

# 内存栅栏
有时候不想编译器和处理器对程序进行内存重排，可以使用Memory Fence/Barriers手段。一般来说memory fence分为两层

* Compiler fence
* CPU fence

## Compiler Fence
参考GCC里面的写法：
 ```c
    asm volatile("": : :"memory")
 ```
![fence.png](fence.png)
增加这条语句后，编译优化后的代码指令顺序就不会改变了。

另外，MS Visual C++里面使用_ReadWriteBarrier

## CPU Fence
参考GCC里面的写法：
```c
    asm volatile("mfence" ::: "memory");
```

不同平台会提供不同的fence指令，参考下表：

| CPU arch | Instructions     |
|:--------:|:----------------:|
| x86      | mfence/lock/xchg |
| AArch64  | lwsync           |
| PowerPC  | dmb ish          |

显而易见的的是，这样做代码移植性就不够好。C++11提供了语言层面的fence函数。

* std::atomic_thread_fence ：在线程间进行数据访问的同步
* std::atomic_signal_fence ：线程和信号处理器间的同步

Fence有三种情况：

1. full fence: 指定memory_order_seq_cst 或者memory_order_acq_rel
2. acquire fence: 指定memory_order_acquire
3. release fence: 指定memory_order_release

按照内存读写来分，分别对应下面四种不同memory barrier类型：

1. **LoadLoad**
2. **LoadStore**
3. **StoreLoad**
4. **StoreStore**

![barriers.png](barriers.png)
**StoreLoad** 是最强的memory barrier类型，也是最昂贵的(从性能角度来看)，比如说sequential consistency。

# 引用
* [cpp-momery-model](https://preshing.com/20120913/acquire-and-release-semantics/)
* [Anthony Williams’ blog](http://www.justsoftwaresolutions.co.uk/blog/) and his book, [C++ Concurrency in Action](http://www.amazon.com/gp/product/1933988770/ref=as_li_ss_tl?ie=UTF8&tag=preshonprogr-20&linkCode=as2&camp=1789&creative=390957&creativeASIN=1933988770)
* [Dmitriy V’jukov’s website](http://www.1024cores.net/) and various [forum discussions](https://groups.google.com/forum/?fromgroups#!forum/lock-free)
* [Bartosz Milewski’s blog](http://bartoszmilewski.com/)
* Charles Bloom’s [Low-Level Threading series](http://cbloomrants.blogspot.ca/2012/06/06-12-12-another-threading-post-index.html) on his blog
* Doug Lea’s [JSR-133 Cookbook](http://g.oswego.edu/dl/jmm/cookbook.html)
* Howells and McKenney’s [memory-barriers.txt](http://www.kernel.org/doc/Documentation/memory-barriers.txt) document
* Hans Boehm’s [collection of links](http://www.hpl.hp.com/personal/Hans_Boehm/c++mm/) about the C++11 memory model
* Herb Sutter’s [Effective Concurrency](http://www.gotw.ca/publications/) series
* [Atomic mapping instructions](https://www.cl.cam.ac.uk/~pes20/cpp/cpp0xmappings.html) 
