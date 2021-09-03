https://github.com/scylladb/seastar/blob/master/doc/tutorial.md

# 引言

Seastar是一个用于在现代多核机器上实现高性能复杂服务端应用的C++库。

传统来说，针对服务端医用的库和框架主要分为2大阵营：
- 一些专注于性能，另一些专注于处理复杂性。
- 一些框架性能非常好，但是只能用来搭建简单的应用
	
例如，DPDK只允许应用独立地处理包 (DPDK allows applications which process packets individually)，而其他一些框架通过牺牲运行效率来实现搭建非常复杂的应用。Seastar是我们集两者之长的一次尝试：创造一个可以用于搭建复杂服务端应用并达到最优性能的库。

Seastar使用2个概念——**future 和 continuation**  ——提供了一个完整的异步编程框架。它们提供了对任何类型的异步事件的一种一致的表达和处理方法，包括但不限于网络I/O，磁盘I/O，以及各种事件的复杂组合。

在现代多核多socket机器上在核间共享数据会带来严重的性能损失。Seastar程序采用了share-nothing模型，也就是说，内存被分给各个核，每个核只处理自己的那份内存中的数据，核间通信需要通过显示的消息传输完成。

# 异步编程
同步、每个进程一个连接的这种server模式不是没有缺点和成本的。慢慢地，server的作者们发现创建一个新进程是很慢的，context switch很慢，每次处理都会有显著的overhead——最明显的是对堆栈大小的要求。Server和内核的作者们非常努力地去缓解这些overhead：他们从进程切换至线程，从创建新线程切换至使用线程池，降低了每个线程的默认堆栈大小，以及提高了虚拟内存大小来partially-utilize stacks。但是仍旧，使用同步设计的server的性能不够理想，扩展性不佳。在1999年，Dan Kigel普及了"the C10K problem"，需要单个server能够高效处理10000个并发连接——其中大多数为慢甚至inactive的连接。

## 异步方式

这个问题的解决方法，也是在后续的几十年中逐渐变得流行的方法，就是抛弃方便但是低效的同步设计，转为一种新型的设计——异步，或者说是事件驱动的server 。一个事件驱动的server仅有一个线程，或者更准确的说，每个CPU一个线程。这个线程会运行一个循环，在循环的每个迭代步里，通过`poll()`（或者是更高效的`epoll` [[高性能开发综述#I O 优化：多路复用技术]]）来监察绑定在开启的file descriptor上的新事件，如sockets。举例来说，一个socket变得可读（有新数据抵达）或者变得可写（我们可以用这个连接发发送更多数据了）都是一个事件。应用通过进行一些非阻塞性的操作，修改一个或多个fd，后者维护这个连接的状态来处理这些事件。

基于事件驱动的服务器，无需为每一个请求创建额外的对应线程，虽然可以省去创建线程与销毁线程的开销，但它在处理网络请求时，会把侦听到的请求放在事件队列中交给观察者，事件循环会不停的处理这些网络请求。  在事件循环中，每一次循环都会查看是否有事件待处理，如果有的话就取出来执行。

写异步server应用的人们在今天仍然会遇到2个主要问题。
- 复杂性：写一个简单的异步server是很简单的。然而写复杂异步server的难度臭名昭著。我们不再能用一些简单易读的函数调用来处理一个连接，而是需要引入大量**回调函数，和一个复杂的状态机**，用于记录对于哪些事件应该调用哪些函数。
- 非阻塞：因为context switch很慢，所以一个核只有一个线程是对性能很重要的。然而，如果我们每个核只有一个线程，处理事件的函数永远不能阻塞，不然这个核就会被闲置。这时候如果有IO ，就需要有多线程。

当需求更高的性能的时候，server应用，以及其使用的框架，需要考虑以下问题：

现代机器：现代的机器和约10年前机器有着非常大的区别。他们有很多核和很深的内存层级（从L1 cache到NUMA），这种结构更适合特定的编程范式。：不可扩展性的编程范式（如，加锁）可能会极大地影响程序在多核上的性能；共享内存和无锁同步primitives虽然是可用的（比如原子操作和memory-ordering fences），但是比只用一个核的cache中的数据进行操作要慢很多，并且也会让程序不好扩展至多核。

Seastar是一个旨在解决上述的4个问题的异步框架：它是一个用于实现同时包括磁盘和网络I/O的复杂server的框架。它的fast path完全是**单线程的（每核）**，可扩展至多核并最小化了代价高昂的核间内存共享。Seastar是一个C++14的库，让用户能利用上复杂的编译优化特征与充分的控制力，且没有运行时overhead。

## 非阻塞异步的事件驱动
Seastar是一个让人可以比较直接地实现非阻塞、异步代码的事件驱动框架。它的API基于future。Seastar利用了如下的概念以获得极致的性能。

- Cooperative micro-task scheduler：每个核执行一个协作任务调度器而不是一个线程，每个任务都很轻量，只处理上一次 I/O 操作的结果并提交新的任务。
- Share-nothing SMP架构：每个内核独立于 SMP 系统中的其他内核运行，核间通过消息传递进行通信，一个seastar核经常称作一个分片。
- 基于future的APIs：
- **Share-nothing TCP stack**：Seastar可以直接使用本机操作系统的TCP stack。在此之外，它也提供了一套基于task scheduler和share-nothing架构的高性能TCP/IP stack。这套stack提供双向零拷贝：你可以直接用TCP stack的buffer处理数据，并在不进行拷贝的情况下发送你的数据结构。
- DMA-based存储APIs：基于网络栈，提供zero copy的存储api

[[seastar smp]], [[seastar networking]]

# Getting started
最简单的Seastar程序如下：
```cpp
#include <seastar/core/app-template.hh>
#include <seastar/core/reactor.hh>
#include <iostream>

int main(int argc, char** argv) {
    seastar::app_template app;
    app.run(argc, argv, [] {
            std::cout << "Hello world\n";
            return seastar::make_ready_future<>();
    });
}
```

如我们在例子中所示，每个Seastar程序必须定义并运行一个`app_template`对象。这个对象会在1个或多个CPU上启动主事件循环(event loop)（the Seastar engine），并运行给定的函数一次——在本例中，是一个lambda函数。

`return make_ready_future<>();`会使事件循环以及整个程序在打印"Hello world"之后立即退出。在一个更典型的Seastar程序中，我们会希望事件循环持续运行，并处理收到的包，直到显式退出。这样的程序会返回一个用于判断何时退出的 `future`。我们将在下文介绍future以及如何使用它。无论何时都不要使用常规的C `exit()`，因为其会阻止Seastar正确地在退出时进行清理工作。

如例子所示，所有Seastar的函数都处于"`seastar`" namespace中。我们推荐使用namespace 前缀而不是`using namespace seastar`，在之后的例子中也会如此。

一些编译选项可以参考 [wiki 原文](https://github.com/scylladb/seastar/blob/master/doc/tutorial.md#introduction)。

# 线程和内存

## Seastar 线程
正如在引言中提到的，基于Seastar的程序每个CPU上运行单一线程。每个线程有自己的事件循环，在Seastar的术语中被称为 engine。默认情况下，Seastar程序会占据所有可用的核，每个核启动一个线程。

我们可以通过如下程序来验证这点，其中`seastar::smp::count`是启动的线程总数：

```cpp
#include <seastar/core/app-template.hh>
#include <seastar/core/reactor.hh>
#include <iostream>

int main(int argc, char** argv) {
    seastar::app_template app;
    app.run(argc, argv, [] {
            std::cout << seastar::smp::count << "\n";
            return seastar::make_ready_future<>();
    });
}
```

在一个4硬件线程的机器上（2核并开启超线程），Seastar默认会使用4个engine thread。

```bash
$ ./a.out
4
```

这4个engine thread会被绑定至（a la `taskset(1)`）不同的硬件线程。注意，如我们提到的，启动函数只在一个线程上运行，所以我们只看到"4"被打印了1次。之后的教程会告诉大家该如何使用所有的线程。

用户可以通过传入一个命令行参数`-c`来告诉Seastar去启动更少的线程数。例如，可以通过如下方式只启动2个线程：

```bash
$ ./a.out -c2
2
```

在这种情况下，Seastar会保证每个线程绑定在不同的核上，我们不会让这2个线程在同一个核上作为超线程相互争夺的（不然会影响性能）。

我们不能分配超过硬件线程总数的线程，这么做会报错：

```bash
$ ./a.out -c5
terminate called after throwing an instance of 'std::runtime_error'
  what():  insufficient processing units
abort (core dumped)
```

上面的程序来自于app.run的异常。为了能更好的catch这种启动异常并在不生成core dump的情况下优雅退出，我们可以这样写：

```cpp
#include <seastar/core/app-template.hh>
#include <seastar/core/reactor.hh>
#include <iostream>
#include <stdexcept>

int main(int argc, char** argv) {
    seastar::app_template app;
    try {
        app.run(argc, argv, [] {
            std::cout << seastar::smp::count << "\n";
            return seastar::make_ready_future<>();
        });
    } catch(...) {
        std::cerr << "Failed to start application: "
                  << std::current_exception() << "\n";
        return 1;
    }
    return 0;
}

```

```bash
$ ./a.out -c5
Couldn't start application: std::runtime_error (insufficient processing units)
```

注意这样**不能**catch到程序实际的异步代码引发的异常。对于那些异常我们会在后文介绍。

## Seastar 内存

正如在引言中介绍的，Seastar程序会将他们的内存分片 (shard)。每个线程会被预分配一大块内存（在运行这个线程的那个NUMA 节点上），并且只使用这块被分配的内存进行程序中的内存分配（例如在`malloc()`或`new`中）。

默认除去给OS保留的1.5G或7%的内存外的**全部内存**都会被通过这种方式分配给应用。这个默认值可以通过`--reserve_memory`来改变给系统剩余的内存，或者`-m`来改变给Seastar应用分配的内存来改变。内存值可以以字节为单位，或者用"k", "M", "G", "T"为单位。这些单位均遵从2的幂。

试着给Seastar超过物理内存的内存值会直接异常退出：

```bash
$ ./a.out -m10T
Couldn't start application: std::runtime_error (insufficient physical memory)

```

# future和continuation简介

future和continuation是Seastar的异步编程模型的基石。通过组合它们可以轻松地组件大型、复杂的异步程序，并保证代码可读、易懂。

## future

future是一个还不可用的计算结果。例如：

- 我们从网络读取的数据buffer
- 计时器的到时
- 磁盘写操作的完成
- 一个需要其他一个或多个future的值来进行的计算的值

`future<int>`类型承载了一个终将可用的int——现在可能已经可用，或者还不能。成员函数`available()`可以用来测试一个值是否已经可用，`get()`可以用来获取它的值。`future<>`类型表示一些终将完成的事件，但是不会返回任何值。

future往往是一个**异步函数**的返回值。异步函数是指一个返回future，并将会将这个future的值确定下来的函数。因为异步函数_保证_将确定future的值，有时他们被称作"promise"。

Seastar的`sleep()`函数就是一个简单的异步函数的例子：

```cpp
future<> sleep(std::chrono::duration<Rep, Period> dur);
```

这个函数设置了一个计时器，从而使在经过指定时间之后future变得可用（并没有携带的值）。

## continuation
**continuation**是一个在future变得可用时会运行的回调函数（常为lambda函数）。continuation会用`then()`方法附于一个future。例如：

```cpp
#include <seastar/core/app-template.hh>
#include <seastar/core/sleep.hh>
#include <iostream>

int main(int argc, char** argv) {
    seastar::app_template app;
    app.run(argc, argv, [] {
        std::cout << "Sleeping... " << std::flush;
        using namespace std::chrono_literals;
        return seastar::sleep(1s).then([] {
            std::cout << "Done.\n";
        });
    });
}
```

在这个例子里，我们从`seastar::sleep(1s)`中获得了一个future，并把一个打印"Done."的continuation附于其上。1s后future会变得可用，这时continuation就会被执行。运行这个程序，我们的确会看到立即被打印的"Sleeping..."，和1s后打印的"Done."，之后程序退出。

`then()`的返回值也是一个future，从而使得链式的continuation变得可能，这点我们之后会提到。现在我们只需要注意我们把这个future作为了`app.run`的返回值，所以程序会在运行完sleep以及其continuation后才会退出。

为了避免在左右的例子里都重复"app_engine"部分的代码，让我们创建一个可以复用的模板：

```cpp
#include <seastar/core/app-template.hh>
#include <seastar/util/log.hh>
#include <iostream>
#include <stdexcept>

extern seastar::future<> f();

int main(int argc, char** argv) {
    seastar::app_template app;
    try {
        app.run(argc, argv, f);
    } catch(...) {
        std::cerr << "Couldn't start application: "
                  << std::current_exception() << "\n";
        return 1;
    }
    return 0;
}
```

使用这个`main.cc`，上述的sleep例子变成了：

```cpp
#include <seastar/core/sleep.hh>
#include <iostream>

seastar::future<> f() {
    std::cout << "Sleeping... " << std::flush;
    using namespace std::chrono_literals;
    return seastar::sleep(1s).then([] {
        std::cout << "Done.\n";
    });
}
```

至此，这个样例非常普通——没有并行，我们用普通的`POSIX sleep()`也能做到。事情会在我们启动多个`sleep`的时候变得有趣。`future`和`continuation`让并行变得非常简单与自然：

```cpp
#include <seastar/core/sleep.hh>
#include <iostream>

seastar::future<> f() {
    std::cout << "Sleeping... " << std::flush;
    using namespace std::chrono_literals;
    seastar::sleep(200ms).then([] { std::cout << "200ms " << std::flush; });
    seastar::sleep(100ms).then([] { std::cout << "100ms " << std::flush; });
    return seastar::sleep(1s).then([] { std::cout << "Done.\n"; });
}
```

每个`sleep()`和`then()`都会立即退出：`sleep()`仅仅启动计时器，而`then()`只是设置到到时的时候应该调用什么函数。所以3行代码都会马上被执行，f也会直接返回。在那之后，时间循环会开始等3个future变得可用，且每个可用的时候都会运行他们对应的continuation。上述代码的输出显然会是：

``` cpp
$ ./a.out
Sleeping... 100ms 200ms Done.
```

`sleep()`返回的是`future<>`，也就是在完成时不会返回任何值。更有趣的future 会在可用时返回一个（或多个）任意类型的值。在下面的例子里，有一个返回`future<int>`的函数，以及对应的`continuation`。注意`continuation`是如何得到`future`中包着的值的：

```cpp
#include <seastar/core/sleep.hh>
#include <iostream>

seastar::future<int> slow() {
    using namespace std::chrono_literals;
    return seastar::sleep(100ms).then([] { return 3; });
}

seastar::future<> f() {
    return slow().then([] (int val) {
        std::cout << "Got " << val << "\n";
    });
}
```

我们需要再解释一下`slow()`。和往常一样，这个函数立即返回一个`future<int>`，不会等sleep完。但是其中`continuation`**返回的是3，而非`future`**。我们在下文介绍`then()`是怎么返回future的，以及这种机制是怎么让链式future变得可能的。

这个例子展示了future这种编程模型的便捷性。因为其让程序员可以很简洁地包装复杂的异步程序。`slow()`可能实际是调用了一个复杂的设计多步的异步操作，但是它可以和`sleep()`同样被使用，并且Seastar的engine会保证在正确的时间运行continuation。

## Ready futures

一个future可能在运行`then()`之前就已经准备好了。我们优化过这种这种重要的情况。对于这种情况，往往`continuation`会被直接运行，而不再用被注册进事件循环，等待事件循环的下一个迭代步了。

在_大多数时候_都会进行上述优化，除了以下情况：`then()`的内部实现里面维护了一个会记录有多少个continuation被立刻执行了用的计数器，在超过一定量continuation被直接运行后（目前限制为256个），下一个continuation一定会被转移到事件循环里。之所以引入这种机制是因为我们发现在一些情况下（例如后文会讨论的future循环），每个准备好的continuation会立刻生成一个新的准备好的continuation。

那么如果没有上述的计数器限制，我们就会一直在立即执行continuation，而不再进入事件循环，从而导致事件循环饿死（starve）。让事件循环可以正常运行非常重要，不然无法运行在事件循环中的其他的continuation了，也会饿住事件循环会进行的**polling**（例如，检查是否网卡有新的活动 _(activity)_ ），这种polling非常重要。

`make_ready_future<>`可以被用来返回一个已经准备好了的future。下面的例子和之前的几乎完全相同，只是`fast()`会返回一个已经准备好了的future。所幸future的接受者不在乎这个区别，并会用相同的方式处理这两种情况：

```cpp
#include <seastar/core/future.hh>
#include <iostream>

seastar::future<int> fast() {
    return seastar::make_ready_future<int>(3);
}

seastar::future<> f() {
    return fast().then([] (int val) {
        std::cout << "Got " << val << "\n";
    });
}
```

# Coroutines（协程）
注意：协程需要 C++20 和支持的编译器。 众所周知，Clang 10 及更高版本可以工作。
使用 Seastar 编写高效异步代码的最简单方法是使用协程。 协程不具有传统 continuations（如下）的大部分缺陷，因此是编写新代码的首选方式。

协程是一个函数，这个函数返回 `seastar::future<T>` ，并且使用 `co_await` 或 `co_return`关键字。协程对其调用者和被调用者不可见。它们以任一角色与传统的 Seastar 代码集成。如果您不熟悉 C++ 协程，您可以看 [A more general introduction to C++ coroutines](https://medium.com/pranayaggarwal25/coroutines-in-cpp-15afdf88e17e)，本节重点介绍协程如何与 Seastar 集成。

这是一个简单的 Seastar 协程示例：

```cpp
#include <seastar/core/coroutine.hh>

seastar::future<int> read();
seastar::future<> write(int n);

seastar::future<int> slow_fetch_and_increment() {
    auto n = co_await read();     // #1
    co_await seastar::sleep(1s);  // #2
    auto new_n = n + 1;           // #3
    co_await write(new_n);        // #4
    co_return n;                  // #5
}

```

在#1 中，我们调用 read() 函数，它返回一个`future`。`co_await` 关键字指示 `Seastar` 检查返回的`future`。

如果`future`已准备好，则从`future`中提取值（一个整数）并分配给 n。 如果`future`未准备好，协程会安排自己在`future`准备好时被调用，并将控制权返回给`seastar`。 一旦`future`准备好，协程就会被唤醒，从`future`中提取值并赋值给n。

在#2 中，我们调用 seastar::sleep() 并等待返回的`future`准备就绪，它会在一秒钟内完成。 这表明 n 在 `co_await` 调用之间被保留，并且协程的作者不需要为协程局部变量安排存储。

第 3 行演示了加法运算，假定读者熟悉它。

在#4 中，我们调用了一个返回 `seastar::future<>` 的函数。 在这种情况下，`future`不带数据，因此没有价值被提取和分配。

第 5 行演示了返回值。 整数值用于返回给我们的调用者在调用协程时得到的 `future<int>`。

## Exceptions in coroutines
#todo，暂时用不到，先不翻译

## Concurrency in coroutines
co_await 运算符允许简单的顺序执行。 多个协程可以并行执行，但每个协程一次只有一个 outstanding 的计算。

#todo，后面有点复杂，先不翻译

# Continuations（延续）
## 在continuation中捕获状态

### 按值捕获
我们已经看到 Seastar Continuations是 lambdas，带着一个future 传递给 then() 方法。
在我们目前看到的例子中，lambda 只不过是匿名函数。 但是 C++11 lambdas 还有一个技巧，这对于 Seastar 中基于future的异步编程非常重要：Lambdas 可以捕获状态。考虑下面的例子：

```cpp
#include <seastar/core/sleep.hh>
#include <iostream>

seastar::future<int> incr(int i) {
    using namespace std::chrono_literals;
    return seastar::sleep(10ms).then([i] { return i + 1; });
}

seastar::future<> f() {
    return incr(3).then([] (int val) {
        std::cout << "Got " << val << "\n";
    });
}

```

`future`操作`incr(i)`需要一定时间执行完成（它需要先睡一会儿......），在这段时间内，它需要保存它正在处理的 i 值。 在早期的事件驱动编程模型中，程序员需要明确定义一个对象来保持这种状态，并管理所有这些对象。在这段时间内，它需要保存它正在处理的 i 值。 在早期的事件驱动编程模型中，程序员需要明确定义一个对象来保持这种状态，并管理所有这些对象。在 Seastar 中，一切都变得简单得多，使用 C++11 的 lambda 表达式：上面示例中的捕获语法“[i]”意味着 i 的值，因为它在调用 incr() 时存在() 被捕获到 lambda 中。 lambda 不仅仅是一个函数——它实际上是一个对象，包含代码和数据。本质上，编译器自动为我们创建了状态对象，我们不需要定义它，也不需要跟踪它（它与`Continuations`一起保存，当`Continuations`被延迟时，并在`Continuations`运行后自动删除） ）。

一个值得理解的实现细节是，当`Continuations`捕获状态并立即运行时，此捕获不会产生运行时开销。但是，当`continuation`不能立即运行（因为`future`还没有准备好）需要保存到后面，就需要在堆上为这个数据分配内存。并且需要将`Continuations`的捕获数据复制到那里。 这有运行时开销，但它是不可避免的，并且与线程编程模型中的相关开销相比非常小（在线程程序中，这种状态通常驻留在阻塞线程的堆栈上，但堆栈要 比我们的捕获状态大得多，占用大量内存并在这些线程之间的上下文切换时导致大量缓存污染）。

在上面的例子中，我们按值捕获了 i ， i 的值的副本被保存到`Continuations`中。 C++ 有两个额外的捕获选项：通过引用捕获和通过移动捕获：

### 引用捕获

在延续中使用**按引用捕获通常是一个错误，并可能导致严重的错误**。 例如，如果在上面的例子中我们捕获了对 i 的引用，而不是复制它。

```cpp
seastar::future<int> incr(int i) {
    using namespace std::chrono_literals;
    // Oops, the "&" below is wrong:
    return seastar::sleep(10ms).then([&i] { return i + 1; });
}
```

这意味着延续将包含 i 的地址，而不是它的值。 但是 i 是一个堆栈变量，并且 incr() 函数会立即返回，因此当`Continuations`最终开始运行时，在 incr() 返回很久之后，该地址将包含不相关的内容。

引用捕获通常是错误规则的一个例外是 do_with() 习语，我们将在后面介绍。 这个习惯用法确保对象在`Continuations`的整个生命周期中都存在，并使按引用捕获成为可能，并且非常方便。

### 移动捕获

在`Continuations`中使用按**移动捕获**在 Seastar 应用程序中也非常有用。 通过将一个对象移动到一个`Continuations`中，我们将这个对象的所有权转移给了`Continuations`，并使得在`Continuations`结束时对象很容易被自动删除。

例如，考虑一个传统的函数`std::unique_ptr<T>`。

```cpp
int do_something(std::unique_ptr<T> obj) {
     // 根据obj的内容做一些计算，假设结果是17
     return 17;
     // 此时， obj 超出范围，因此编译器 delete()s 它。
```

通过以这种方式使用 unique_ptr，调用者将一个对象传递给函数，但告诉它该对象现在是它的专属责任——当函数完成该对象时，它会自动删除它。 我们如何在`Continuations`中使用 unique_ptr ？ 以下将不起作用

```cpp
seastar::future<int> slow_do_something(std::unique_ptr<T> obj) {
    using namespace std::chrono_literals;
    // The following line won't compile...
    return seastar::sleep(10ms).then([obj] () mutable { return do_something(std::move(obj)); });
}
```

问题在于，unique_ptr 不能通过值传递到`Continuations`中，因为这需要复制它，这是被禁止的，因为它违反了仅存在此指针的一个副本的保证。 但是，我们可以将 obj 移动到`Continuations`中：

```cpp
seastar::future<int> slow_do_something(std::unique_ptr<T> obj) {
    using namespace std::chrono_literals;
    return seastar::sleep(10ms).then([obj = std::move(obj)] () mutable {
        return do_something(std::move(obj));
    });
}
```

这里使用 std::move() 导致 obj 的移动赋值用于将对象从外部函数移动到延续中。 C++11 中引入的移动（移动语义）的概念类似于浅拷贝，然后使源拷贝无效（这样两个拷贝就不会共存，因为 unique_ptr 禁止这样做）。 将 obj 移入 continuation 后，上层函数就不能再使用它（in this case it's of course ok, 因为我们无论如何都会返回）。
### c++14捕获语法
我们在这里使用的 `[obj = ...]` **捕获语法是 `C++14` 的新内容**。 这是 Seastar 需要 C++14，并且不支持旧的 C++11 编译器的主要原因。

这里需要额外的 () 可变语法，因为默认情况下，当 C++ 将一个值（在这种情况下，std::move(obj) 的值）捕获到一个 lambda 中时，它使这个值成为只读的，所以我们的 lambda 不能， 在本例中，再次移动它。 添加 mutable 消除了这种人为限制。

## 求值顺序的注意事项
**（只在c++14）**

C++14（及更低版本的C++）不能保证`continuation`中的`lambda` 捕获会在他们相关的`future`被计算之后才被求值（见 [https://en.cppreference.com/w/cpp/language/eval_order）。](https://en.cppreference.com/w/cpp/language/eval_order%EF%BC%89%E3%80%82)

因此，请避免以下编程模式：
```cpp
    return do_something(obj).then([obj = std::move(obj)] () mutable {
        return do_something_else(std::move(obj));
    });
```

这个例子中`[obj = std::move(obj)]`可能会先于`do_something(obj)`，导致出现使用了被move出去之后的`obj`。

为了保证我们期望的顺序，上面的表达式可以拆分为：
```cpp
    auto fut = do_something(obj);
    return fut.then([obj = std::move(obj)] () mutable {
        return do_something_else(std::move(obj));
    });

```

c++17后，`then`对应的对象会在`then`的参数之前才被求值，使得`do_something(obj)`会保证先被执行。所以C++17中可以不采用上面的改正方法。

## 链式continuation
TODO: 我们已经在上面的 slow() 中看到了链接示例。 谈论从then 的返回，并返回future 并链接更多then。

# 处理异常

在`continuation`中抛出的异常被系统隐式捕获并存储在`future`。 存储此类异常的 `future` 与就绪的 `future` 类似，因为它可以启动`continuation`，但它不包含值——仅包含异常。

在这样的`future`上调用 `.then()` 会跳过`continuation`，并将输入`future`（调用 .then() 的对象）的异常转移到输出`future`（`.then()` 的返回值）。

```cpp
line1();
line2(); // throws!
line3(); // skipped
```

类似于
```cpp
return line1().then([] {
    return line2(); // throws!
}).then([] {
    return line3(); // skipped
});
```

通常，需要中止当前的操作链并返回异常，但有时需要更细粒度的控制。 有几种用于处理异常的原语：
1. .then_wrapped(): .then_wrapped() 不是将`future`携带的值传递给`continuation`，而是将输入`future`传递给`continuation`。 保证`future`处于就绪状态，因此`continuation`可以检查它是否包含值或异常，并采取适当的行动。
2. .finally()：类似于Java的finally块，无论输入的`future`是否带有异常，.finally()延续都会被执行。 finally 延续的结果是它的输入`future`，因此 .finally() 可用于在无条件执行的流中插入代码，否则不会改变流。
TODO：给出上面的示例代码。 还要提到 handle_exception，可能会延迟到后面的章节。

## Exceptions vs. exceptional futures
异步函数可以通过以下两种方式之一失败：它可以立即失败，通过抛出异常，或者它可以返回一个最终会失败的未来（解析为异常）。 这两种失败模式看起来与未初始化(uninitiated)相似。但在尝试使用 finally()、handle_exception() 或 then_wrapped() 处理异常时，行为会有所不同。 例如，考虑以下代码：

```cpp
#include <seastar/core/future.hh>
#include <iostream>
#include <exception>

class my_exception : public std::exception {
    virtual const char* what() const noexcept override { return "my exception"; }
};

seastar::future<> fail() {
    return seastar::make_exception_future<>(my_exception());
}

seastar::future<> f() {
    return fail().finally([] {
        std::cout << "cleaning up\n";
    });
}

```

正如预期的那样，此代码将打印`cleaning up`消息 - 异步函数 fail() 返回一个解析为失败的`future`，并且 finally() 继续运行，尽管有此失败。

现在考虑在上面的例子中我们对 fail() 有不同的定义：

```cpp
seastar::future<> fail() {
    throw my_exception();
}
```

在这里，fail() 不会返回失败的`future`。 相反，它根本无法返回`future`！ 它抛出的异常会停止整个函数 f()，并且 finally() 延续不会附加到`future`（从未返回），并且永远不会运行。 现在不打印`cleaning up`消息。

我们建议为了减少出现此类错误的机会，异步函数应始终返回失败的`future`，而不是抛出实际异常。 如果异步函数在返回`future`之前调用另一个函数，并且第二个函数可能会抛出，它应该使用 try/catch 来捕获异常并将其转换为失败的`future`：

```cpp
void inner() {
    throw my_exception();
}
seastar::future<> fail() {
    try {
        inner();
    } catch(...) {
        return seastar::make_exception_future(std::current_exception());
    }
    return seastar::make_ready_future<>();
}
```

在这里，fail() 捕获由 inner() 抛出的异常，无论它可能是什么，并返回一个失败的`future`。 以这种方式编写，将到达 finally() 延续，并打印`cleaning up`消息。

尽管建议异步函数避免抛出异常，但一些异步函数除了返回异常`future`外，还会抛出异常。 一个常见的例子是分配内存并在内存不足时抛出 `std::bad_alloc` 的函数，而不是返回`future`。 `future<> seastar::semaphore::wait()` 方法就是这样的一个函数：它返回一个`future`，如果信号量 `was broken()` 或等待超时，它可能是异常的，但也可能在分配内存（用来保存waiters列表的内存）失败时抛出异常。 因此，除非一个函数——包括异步函数——被明确标记为“noexcept”，否则应用程序应该准备好处理从它抛出的异常。 在现代 C++ 中，代码通常**使用 RAII** 来保证异常安全，而不用 try/catch。 **seastar::defer() 是一个基于 RAII 的习惯用法**，它确保即使抛出异常也能运行一些清理代码。

Seastar 有一个方便的通用函数`futurize_invoke()`，在这里很有用。`futurize_invoke(func, args...)` 运行一个可能返回`future`值或立即值的函数。并且在这两种情况下都将结果转换为未来值。`futurize_invoke()` 还将函数抛出的立即异常（如果有）转换为失败的`future`，就像我们上面所做的一样。

所以使用 `futurize_invoke()` 我们可以使上面的例子工作，即使 `fail()` 确实抛出了异常：

```cpp
seastar::future<> fail() {
    throw my_exception();
}
seastar::future<> f() {
    return seastar::futurize_invoke(fail).finally([] {
        std::cout << "cleaning up\n";
    });
}
```

请注意，如果异常风险存在于`continuation`中，则大部分讨论将变得毫无意义。 考虑以下代码：
```cpp
seastar::future<> f() {
    return seastar::sleep(1s).then([] {
        throw my_exception();
    }).finally([] {
        std::cout << "cleaning up\n";
    });
}
```

在这里，第一个`continuation`的 `lambda` 函数确实抛出异常而不是返回失败的`future`。 然而，我们没有像以前一样的问题，这只是因为异步函数在返回一个有效的`future`之前抛出了一个异常。

# 管理生命周期

一个异步函数可能会启动一个会在本函数返回后还会运行很久的操作：函数自身会在运行的时候立即返回一个`future<T>`，但是等这个future准备好需要很久。

当这样的异步操作需要操作现存的对象，或者是临时对象，我们需要留心这些对象的_生命周期_：我们得保证这些对象不会在异步操作完成之前被释放，并保证对象在不被需要的时候能最终被释放（以防内存泄漏）。Seastar提供了很多种用于安全高效地保证对象在合适的时间保活的机制。在本节中，我们会探索这些机制，并告诉大家每一种该何时使用。

## 传递所有权给continuation
我们已经看到如何通过**捕获**(capturing)来让`continuation`获取一个对象的所有权：

```cpp
seastar::future<> slow_incr(int i) {
    return seastar::sleep(10ms).then([i] { return i + 1; });
}

```

在上例中，continuation捕获了`i`的值，换句话说，continuation有一份`i`的拷贝。当continuation在10ms后开始运行时，它可以访问到这个值，并在其运行结束的时候，continuation本身这个对象会被释放，它捕获的`i`的拷贝也随之被释放了。continuation拥有`i`的这个拷贝。

如我们在这里进行的用值捕获(capturing by value)——创建一个我们需要的对象的拷贝——大多数情况下适用于例子中像整数这样的很小的对象。其他一些对象的拷贝成本可能很高，甚至不能被拷贝。例如，下面这么做就不太好。

```cpp
seastar::future<> slow_op(std::vector<int> v) {
    // this makes another copy of v:
    return seastar::sleep(10ms).then([v] { /* do something with v */ });
}

```

上面这样可能会很低效——因为需要拷贝可能很长的vector`v`，并且要在continuation中存一份。在本例中，我们没必要复制`v`——在`slow_op`中我们已经传值了，并且在capture之后没有再对`v`做其他的操作了，所以`slow_op`再返回的时候会释放自己的那份`v`。

对于这种情况，C++14允许把对象move进continuation：

```cpp
seastar::future<> slow_op(std::vector<int> v) {
    // v is not copied again, but instead moved:
    return seastar::sleep(10ms).then([v = std::move(v)] { /* do something with v */ });
}
```

C++11引入了move constructor，可以把vector的数据移进`continuation`，并释放原始的vector。Moving会很快——对于vector，moving操作只需要复制几个指针这样的小field。如之前那样，一旦continuation退出了vector就会被释放——并且其底层的数组（是被move操作移进来的）也会随之被释放。

TODO: 将临时缓冲区作为设计为以这种方式移动的对象的示例。

一些情况下，move对象不是很合适。例如，一些对象的引用或者一些成员以引用的形式被存在其他对象里了，那么这些引用会在这个对象被move之后变得不可用。在一些更复杂的例子里，move constructor甚至都有些慢。对于这些情况，C++提供了`std::unique_ptr<T>`。`std::unique_ptr<T>`是一个拥有另一个类型的`T`的对象的对象。当`std::unique_ptr<T>`被move了，内部的`T`对象并没有变化——仅仅是指向`T`对象的指针发生了变化。`std::unique_ptr<T>`的用例如下：

```cpp
seastar::future<> slow_op(std::unique_ptr<T> p) {
    return seastar::sleep(10ms).then([p = std::move(p)] { /* do something with *p */ });
}

```

`std::unique_ptr<T>`是传递unique 对象所有权的C++标准机制：对象在同一时刻只会被一段代码拥有，且所有权通过move `unique_ptr`对象进行转移。`unique_ptr`对象不能被复制：如果我们尝试用值去捕获`p`，会引发编译错误。

## 保持调用者的所有权
我们上面描述的技术——赋予它需要处理的对象的`continuation`所有权——是强大且安全的。 但通常它会变得难以使用且冗长。 当异步操作不仅涉及一个`continuation`，而且涉及一系列`continuation`，每个`continuation`都需要在同一个对象上工作时，我们需要在每个连续的`continuation`之间传递对象的所有权，这会变得不方便。 当我们需要将同一个对象传递给两个单独的异步函数（或`continuation`）时，这尤其不方便——在我们将对象移入一个之后，需要返回该对象，以便它可以再次移入第二个。

```cpp
seastar::future<> slow_op(T o) {
    return seastar::sleep(10ms).then([o = std::move(o)] {
        // first continuation, doing something with o
        ...
        // return o so the next continuation can use it!
        return std::move(o);
    }).then([](T o) {
        // second continuation, doing something with o
        ...
    });
}
```

之所以会出现这种复杂性，是因为我们希望异步函数和`continuation`获得它们所操作对象的所有权。 一种更简单的方法是让异步函数的调用者`continuation`是对象的所有者，并将对象的引用传递给需要该对象的各种其他异步函数和`continuation`。比如：

```cpp
seastar::future<> slow_op(T& o) {           // <-- pass by reference
    return seastar::sleep(10ms).then([&o] {// <-- capture by reference
        // first continuation, doing something with o
        ...
    }).then([&o]) {                        // <-- another capture by reference
        // second continuation, doing something with o
        ...
    });
}

```

这种方法提出了一个问题：slow_op 的调用者现在负责保持对象 o 处于活动状态，而由 slow_op 启动的异步代码需要这个对象。 但是这个调用者如何知道它启动的异步操作实际需要这个对象多长时间呢？

最合理的答案是异步函数可能需要访问其参数，直到它返回的`future`得到解决——此时异步代码完成并且不再需要访问其参数。 因此，我们建议 Seastar 代码采用以下约定：

> 每当异步函数通过引用获取参数时，调用者必须确保被引用的对象一直存在，直到函数返回的未来被解析。

请注意，这只是 Seastar 建议的约定，不幸的是 C++ 语言中没有强制执行它。 非 Seastar 程序中的 C++ 程序员经常将大对象作为常量引用传递给函数，以避免慢速复制，并假设被调用的函数不会将这个引用保存在任何地方。 但在 Seastar 代码中，这是一种危险的做法，因为即使异步函数不打算将引用保存在任何地方，它最终可能会通过将此引用传递给另一个函数并最终在`continuation`中捕获它来隐式保存。

>如果 C++ 的未来版本可以帮助我们发现引用的错误使用，那就太好了。 也许我们可以有一个特殊类型的引用的标签，一个函数可以立即使用的“立即引用”（即在返回`future`之前），但不能被捕获到`continuation`中。

有了这个约定，就可以很容易地编写复杂的异步函数，比如 slow_op，它通过引用传递对象，直到异步操作完成。 但是调用者如何确保对象在返回的`future`被解析之前一直存在？ 以下是错误的：
```cpp
seastar::future<> f() {
    T obj; // wrong! will be destroyed too soon!
    return slow_op(obj);
}
```

这是错误的，因为这里的对象 obj 对 f 的调用是本地的，并且一旦 f 返回一个`future`就被销毁——而不是当这个返回的`future`被解析时！ 正确的做法是调用者在堆上创建对象 obj（所以它不会在 f 返回时立即被销毁），然后运行 slow_op(obj) 并在`future`解析时（即，使用 .finally ())，销毁对象。

Seastar 提供了一个方便的习惯用法 do_with() 来正确执行此操作：

```cpp
seastar::future<> f() {
    return seastar::do_with(T(), [] (auto& obj) {
        // obj is passed by reference to slow_op, and this is fine:
        return slow_op(obj);
    }
}

```

do_with 将使用给定的保活对象执行给定的函数。
do_with 将给定的对象保存在堆上，并使用对新对象的引用调用给定的 lambda。 最后，它确保在算出返回`future`的后销毁新对象。
通常，do_with 被赋予一个右值，即一个未命名的临时对象或一个 std::move()的对象，并且 do_with 将该对象移动到它在堆上的最终位置。
do_with 返回一个在上述所有内容完成后算出的`future`（lambda 的`future`被算出并且对象被销毁）。

为方便起见，do_with 也可以被赋予多个对象以保持活动状态。 例如，这里我们创建了两个对象并让它们保持活动状态，直到算出`future`：

```cpp
seastar::future<> f() {
    return seastar::do_with(T1(), T2(), [] (auto& obj1, auto& obj2) {
        return slow_op(obj1, obj2);
    }
}

```

虽然 do_with 可以改变它持有的对象的生命周期，但如果用户不小心复制了这些对象，这些副本可能有错误的生命周期。 不幸的是，像忘记“&”这样的简单错误会导致这种意外的复制。 例如，以下代码是错误的：

```cpp
seastar::future<> f() {
    return seastar::do_with(T(), [] (T obj) { // WRONG: should be T&, not T
        return slow_op(obj);
    }
}

```

在这个错误的片段中， obj 不是对 do_with 分配的对象的引用，而是它的一个副本——一个副本一旦 lambda 函数返回就会被销毁（lambda函数返回，是说在主线程里面），而不是在它返回的`future`被解析时被销毁。 这样的代码很可能会崩溃，因为对象在被释放后被使用。 不幸的是，编译器不会警告此类错误。 用户应该习惯于始终使用带有 do_with 的类型“auto&”——如上面正确的例子——以减少出现此类错误的机会。

出于同样的原因，以下代码片段也是错误的：

```cpp
seastar::future<> slow_op(T obj); // WRONG: should be T&, not T
seastar::future<> f() {
    return seastar::do_with(T(), [] (auto& obj) {
        return slow_op(obj);
    }
}
```

在这里，虽然 obj 被正确地通过引用传递给了 lambda，但我们后来不小心传递了一个 slow_op() 它的副本（因为这里slow_op 是通过值而不是通过引用获取对象），并且这个副本会在slow_op 返回后立即被销毁 ，不要等到返回的算出`future`。

使用 do_with 时，请始终记住它需要遵守上述约定：我们在 do_with 内部调用的异步函数在返回的 future 被解析后不得使用 do_with 持有的对象。 对于异步函数来说，这是一个严重的释放后使用错误，返回一个未来解决，同时仍然使用 do_with() 的对象进行后台操作。

通常，在留下后台操作的同时解析异步函数很少是一个好主意 - 即使这些操作不使用 do_with() 的对象。 我们不等待的后台操作可能会导致我们耗尽内存（如果我们不限制它们的数量）并且很难干净地关闭应用程序。

## 共享所有权（引用计数）

在本章的开头，我们已经注意到，将对象的副本捕获到`continuation`中是确保在`continuation`运行时对象处于活动状态并随后被销毁的最简单方法。 但是，复制复杂对象通常很昂贵（在时间和内存方面）。 有些对象根本无法复制，或者是可读写的，并且`continuation`应该修改原始对象，而不是新的副本。 所有这些问题的解决方案是引用计数，也就是共享对象：

Seastar 中引用计数对象的一个简单示例是 `seastar::file`，一个保存打开文件对象的对象（我们将在后面的部分介绍 `seastar::file`）。文件对象可以被复制，但复制不涉及复制文件描述符（更不用说文件了）。 相反，两个副本都指向同一个打开的文件，并且引用计数增加 1。当一个文件对象被销毁时，文件的引用计数减一，只有当引用计数达到 0 时，底层文件才真正关闭。

文件对象可以非常快速地复制并且所有副本实际上都指向同一个文件，这使得将它们传递给异步代码非常方便； 例如，

```cpp
seastar::future<uint64_t> slow_size(file f) {
    return seastar::sleep(10ms).then([f] {
        return f.size();
    });
}
```

注意调用 `slow_size` 就像调用 `slow_size(f)` 一样简单，传递 f 的副本，不需要做任何特殊的事情来确保 f 只在不再需要时被销毁。 当没有任何东西再提到 f 时，这很自然地发生。

你可能想知道为什么上面例子中的 `return f.size()` 是安全的：它不是在 f 上启动一个异步操作吗（文件的大小可能存储在磁盘上，所以不是立即可用的），并且 f 可能会立即被销毁，当 我们回来了，什么都没有持有 f 的副本？ 如果 f 真的是最后一个引用，那确实是一个错误，但还有另一个：文件永远不会关闭。 使代码有效的假设是有另一个对 f 的引用将用于关闭它。 `close` 成员函数持有该对象的引用计数，因此即使没有其他人继续持有它，它也会继续存在。 由于文件对象生成的所有`future`在关闭之前已完成，因此正确性所需的只是记住始终关闭文件。

引用计数有运行时成本，但通常非常小； 重要的是要记住 Seastar 对象始终仅由单个 CPU 使用，因此引用计数递增和递减操作并不是采用经常用来做引用计数的慢速原子操作的这种方案，而只是常规的 CPU 本地整数操作。 此外，明智地使用 std::move() 和编译器的优化器可以减少不必要的来回增加和减少引用计数的次数。

C++11 提供了一种创建引用计数共享对象的标准方法——使用模板 `std::shared_ptr<T>`。 `shared_ptr` 可用于将任何类型包装到引用计数共享对象中，如上面的 `seastar::file`。 然而，标准 `std::shared_ptr` 是为多线程应用程序设计的，因此它对引用计数使用缓慢的原子递增/递减操作，我们已经注意到在 `Seastar` 中这是不必要的。 为此，`Seastar` 提供了自己的该模板的单线程实现，`seastar::shared_ptr<T>`。 除了不使用原子操作外，它类似于` std::shared_ptr<T>`【太牛了】。
	
此外，`Seastar` 还提供了一个开销更低的` shared_ptr `变体：`seastar::lw_shared_ptr<T>`。 由于需要正确支持多态类型（由一个类创建的共享对象，并通过指向基类的指针访问），功能齐全的 `shared_ptr` 变得复杂。 它使得 `shared_ptr` 需要向共享对象添加两个变量，并且每个 `shared_ptr` 副本添加两个变量。 简化的 `lw_shared_ptr` - 不支持多态类型 - 在对象中只添加一个变量（引用计数），并且每个副本只是一个变量 - 就像复制常规指针一样。 出于这个原因，轻量级的 `seastar::lw_shared_ptr<T> `应该尽可能优先（T 不是多态类型），否则 `seastar::shared_ptr<T>`。 更慢的 `std::shared_ptr<T> `永远不应该用在分片 `Seastar` 应用程序中。

	
## 在堆栈上保存对象
如果我们可以像通常在同步代码中那样将对象保存在堆栈上，那不是很方便吗？ 类似于：
```cpp
int i = ...;
seastar::sleep(10ms).get();
return i;

```

`Seastar` 允许编写此类代码，方法是使用带有自己的堆栈的` seastar::thread` 对象。 使用` seastar::thread `的完整示例可能如下所示：

```cpp
seastar::future<> slow_incr(int i) {
    return seastar::async([i] {
        seastar::sleep(10ms).get();
        // We get here after the 10ms of wait, i is still available.
        return i + 1;
    });
}

```

我们会在Seastar thread一节介绍 `seastar::thread`, `seastar::async()` 和 `seastar::future::get()` 。

# 高级future
## fututre和中断

TODO：一个`future`，例如` sleep(10s)` 不能被中断。 所以如果我们需要，`promise` 需要有一个机制来中断它。 提到管道的关闭功能，信号量停止功能等。

## future只能被用一次
TODO：假设我们有一个`future<int>`变量，一旦我们 `get() `或` then()` 它，它就会变得无效——我们需要将值存储在其他地方。 想想是否有我们可以建议的替代方案。

# Fibers（纤维？）

这些`fiber`并不是线程——每个都只是一串`continuation`——但是他们也有和传统线程一样的一些要求。例如，我们会避免一个`fiber`被`starve`，而另一个一直在运行。又如，`fiber`之间可能会进行通信——一个fiber生成另一个fiber使用的数据，我们希望能保证2个fiber都能运行，并且如果1个过早地停止了，另一个不要一直`hang`。

TODO：讨论与Fibers相关的部分，如循环、信号量、门、管道等。

# Loops（循环）
大多数很消耗时间的计算都需要循环。Seastar提供了很多用future/promise的方式表示循环的原语。有关Seastar的循环原语一个非常重要的点是，每个迭代步的后面都会有一个抢占点(preemption point)，因而允许其他任务在迭代步之间运行。

## repeat
一个用`repeat`创建的循环会执行其body，然后收到一个`stop_iteration`对象。这个对象会告诉循环是该继续运行（`stop_iteration::no`）还是该停止（`stop_iteration::yes`），下一个迭代步会在前一个执行完再执行。被传给`repeat`的body需要返回`future<stop_iteration>`。

```cpp
seastar::future<int> recompute_number(int number);

seastar::future<> push_until_100(
			seastar::lw_shared_ptr<std::vector<int>> queue,
			int element) {
    return seastar::repeat([queue, element] {
        if (queue->size() == 100) {
            return make_ready_future<stop_iteration>(stop_iteration::yes);
        }
        return recompute_number(element).then([queue] (int new_element) {
            queue->push_back(element);// 感觉这里应该是new_element
            return stop_iteration::no;
        });
    });
}

```

## do_until
`do_until`和`repeat`非常接近，只是其会显示传入一个判断条件。下列代码展示了该如何使用`do_until`：

```cpp
seastar::future<int> recompute_number(int number);

seastar::future<> push_until_100(seastar::lw_shared_ptr<std::vector<int>> queue, int element) {
    return seastar::do_until([queue] { return queue->size() == 100; }, [queue, element] {
        return recompute_number(element).then([queue] (int new_element) {
            queue->push_back(new_element);
        });
    });
}

```


注意循环body需要返回`future<>`，从而使我们能够在循环中创建复杂的continuation。

## do_for_each

`do_for_each` 相当于 Seastar 世界中的 for 循环。 它接受一个范围（或一对迭代器）和一个函数体，它按顺序一个一个地应用于每个参数。 只有在第一个迭代完成后才会启动下一个迭代，就像 `repeat` 一样。 像往常一样，`do_for_each` 期望它的循环体返回一个 `future<>`。

```cpp
seastar::future<> append(
			seastar::lw_shared_ptr<std::vector<int>> queue1, 
			seastar::lw_shared_ptr<std::vector<int>> queue2) {
    return seastar::do_for_each(queue2, [queue1] (int element) {
        queue1->push_back(element);
    });
}

seastar::future<> append_iota(
			seastar::lw_shared_ptr<std::vector<int>> queue1,
			int n) {
    return seastar::do_for_each(
			boost::make_counting_iterator<size_t>(0), 
			boost::make_counting_iterator<size_t>(n),
			[queue1] (int element) {
        		queue1->push_back(element);
			});
}

```

`do_for_each` 接受对容器的左值引用或一对迭代器。 这意味着在整个循环执行期间确保容器处于活动状态的责任属于调用者。 如果容器需要延长其生命周期，可以使用 `do_with` 轻松实现：

```cpp
seastar::future<> do_something(int number);

seastar::future<> do_for_all(std::vector<int> numbers) {
    // Note that the "numbers" vector will be destroyed as soon as this function
    // returns, so we use do_with to guarantee it lives during the whole loop execution:
    return seastar::do_with(std::move(numbers), [] (std::vector<int>& numbers) {
        return seastar::do_for_each(numbers, [] (int number) {
            return do_something(number);
        });
    });
}
```


## parallel_for_each

`parallel_for_each` 是 `do_for_each` 的高并发变体。 使用 `parallel_for_each` 时，所有迭代都会同时排队——这意味着无法保证它们以何种顺序完成操作。


```cpp
seastar::future<> flush_all_files(seastar::lw_shared_ptr<std::vector<seastar::file>> files) {
    return seastar::parallel_for_each(files, [] (seastar::file f) {
        // file::flush() returns a future<>
        return f.flush();
    });
}

```

`parallel_for_each` 是一个强大的工具，因为它允许并行生成许多任务。 这可能会带来巨大的性能提升，但也有一些注意事项。 首先，过高的并发可能会很麻烦——详细信息可以在限制循环的并行性(**Limiting parallelism of loops**.)一章中找到。

要通过整数限制 `parallel_for_each` 的并发性，请使用下面描述的 `max_concurrent_for_each`。 有关处理并行性的更多详细信息，请参见限制循环的并行性(**Limiting parallelism of loops**.)一章。

其次，请注意，在 `parallel_for_each` 循环中执行迭代的顺序是任意的——如果需要严格的顺序，请考虑使用 `do_for_each`。

TODO：`map_reduce`，作为 `parallel_for_each` 的快捷方式（？），它需要产生一些结果（例如，布尔结果的`logical_or`），因此我们不需要显式创建 `lw_shared_ptr`（或 `do_with`）。

TODO：请参阅 seastar 提交“input_stream: Fix possible infinite recursion in consume()”的示例，了解为什么递归是可能替代`repeat()`的，但很糟糕。 另见我的评论  [为什么 Seastar 的迭代原语应该用于尾调用优化](https://groups.google.com/d/msg/seastar-dev/CUkLVBwva3Y/3DKGw-9aAQAJ) 。

【摘录并翻译于此】
>**Nadav Har'El**
>抱歉，伙计们（Avi 实际上最初编写了这段代码，但我对其进行了大修，本应该早已发现这一点）。
>我在 Seastar 教程中给自己做了个笔记来解释为什么递归 **不是** Seastar 迭代原语（do_until、repeat 等）的良好替代品。
>话虽如此，这里有一些我没有完全理解的东西：
代码做这样的事情：
>```cpp
>future<> consume() {  
 >return fd.get().then([] { return consume(); });  
>}
>```
>所以如果 `fd.get()` 没有准备好，我们这里根本没有递归——`consumer()` 立即返回一个 `future`，稍后会再次调用（没有递归）。
>所以递归只有在 fd.get() 立即准备好时才会发生。
>但是当此代码立即运行时，由于“返回”并且编译器可以确定代码不需要任何堆栈上的变量，编译器可以进行“尾调用优化”以*替换* 最后一帧，而不是添加到它，并避免递归。 为什么编译器不这样做？ 是因为 C++ 销毁顺序的原因（例如，C++ 认为它需要保证在 then() 返回之后销毁对象，而不是之前？或者只是优化器的限制？
>无论如何，最好避免依赖尾调用优化。

## max_concurrent_for_each
`max_concurrent_for_each` 是 `parallel_for_each` 的变体，具有受限的并行性。 它接受一个额外的参数 - `max_concurrent` - 最多 `max_concurrent` 迭代同时排队，不保证它们完成操作的顺序。

```cpp
seastar::future<> flush_all_files(seastar::lw_shared_ptr<std::vector<seastar::file>> files, size_t max_concurrent) {
    return seastar::max_concurrent_for_each(files, max_concurrent, [] (seastar::file f) {
        return f.flush();
    });
}

```

确定最大并发限制超出了本文档的范围。 它通常应该从运行软件的系统的实际能力中推导出来，例如并行执行单元或 I/O 通道的数量，以便优化资源利用率而不会使系统不堪重负。

# when_all: 等待多个future
上面我们见到了`parallel_for_each()`，其会开启若干异步操作，并等待所有的都完成。Seastar有另一个函数`when_all()`，用于等待若干已经存在的`future`完成。

`when_all()`的第一种变体是一个变长函数，也就是`future`可以作为分隔的参数传入，有多少个`future`是在编译期就确定下来的。每个`future`可以类型不同。例如：


```cpp
#include <seastar/core/sleep.hh>

future<> f() {
    using namespace std::chrono_literals;
    future<int> slow_two = sleep(2s).then([] { return 2; });
    return when_all(sleep(1s), std::move(slow_two), 
                    make_ready_future<double>(3.5)
           ).discard_result();
}

```

这将启动三个 `futures` - 一个休眠一秒钟（并且不返回任何内容），一个休眠两秒钟并返回整数 2，一个立即返回双精度 3.5 - 然后等待它们。 `when_all()` 函数返回一个`futures`，一旦所有三个`futures`都计算出来了，即在两秒后解决。 这个`future`也有一个值，我们将在下面解释，但在这个例子中，我们只是等待算出`future`并丢弃它的值。

请注意，`when_all() `仅接受右值，它可以是临时值（如异步函数的返回值或 `make_ready_future`）或保存未来的 `std::move() `变量。

`when_all() `返回的`future`解析为一个已经解析`future`的元组，并包含三个输入`future`的结果。 继续上面的例子，


```cpp
future<> f() {
    using namespace std::chrono_literals;
    future<int> slow_two = sleep(2s).then([] { return 2; });
    return when_all(sleep(1s), std::move(slow_two),
                    make_ready_future<double>(3.5)
           ).then([] (auto tup) {
            std::cout << std::get<0>(tup).available() << "\n";
            std::cout << std::get<1>(tup).get0() << "\n";
            std::cout << std::get<2>(tup).get0() << "\n";
    });
}

```

该程序的输出（两秒后出现）是 1, 2, 3.5：元组中的第一个`future`可用（但没有值），第二个具有整数值 2，第三个具有双精度值 3.5 - 正如预期的那样。

一个或多个等待的`future`可能会解决中出现异常，但这不会改变 `when_all()` 的工作方式：它仍然等待所有`future`解决，每个`future`都有一个值或异常，并且在返回的元组中 `future`可能包含异常而不是值。 例如，


```cpp
future<> f() {
    using namespace std::chrono_literals;
    future<> slow_success = sleep(1s);
    future<> slow_exception = sleep(2s).then([] { throw 1; });
    return when_all(std::move(slow_success), std::move(slow_exception)
           ).then([] (auto tup) {
            std::cout << std::get<0>(tup).available() << "\n";
            std::cout << std::get<1>(tup).failed() << "\n";
            std::get<1>(tup).ignore_ready_future();
    });
}

```

两个`future`都可用（已解决），但第二个`future`已失败（导致异常而不是返回一个值）。 请注意我们如何在这个失败的`future`上调用 `ignore_ready_future()` ，因为默默地忽略一个失败的`future`被认为是一个错误，并将导致“异常`future`被忽略”错误消息。 更典型的是，应用程序将记录失败的`future`而不是忽略它。

上面的例子表明` when_all() `使用起来不方便且冗长。 结果被包装在一个元组中，导致冗长的元组语法，并使用准备好的`future`，必须单独检查所有异常以避免错误消息。

所以SeaStar还提供了一个更容易使用的` when_all_succeed() `函数。 这个函数当每个给定的`future`都解决了也是返回一个`future`。如果它们都成功，它将结果值传递给`continuation`，而不将它们包装在`future`或元组中。但是，如果一个或多个`future`失败，则 `when_all_succeed()` 将解析为失败的`future`，其中包含来自失败`future`之一的异常。如果给定的`future`有多个失败，其中一个将被传递（未指定选择哪个），其余的将被默默忽略。 例如，

```cpp
using namespace seastar;
future<> f() {
    using namespace std::chrono_literals;
    return when_all_succeed(sleep(1s), make_ready_future<int>(2),
                    make_ready_future<double>(3.5)
            ).then([] (int i, double d) {
        std::cout << i << " " << d << "\n";
    });
}
```

请注意`future`持有的整数和双精度值如何方便地单独（没有元组）传递到延续。 由于 sleep() 不包含值，因此会等待它，但不会将第三个值传递给`continuation`。 这也意味着如果我们` when_all_succeed() `在几个 `future<>`（没有值）上，结果也是一个 `future<>`：

```cpp
using namespace seastar;
future<> f() {
    using namespace std::chrono_literals;
    return when_all_succeed(sleep(1s), sleep(2s), sleep(3s));
}
```

此示例仅等待 3 秒（最多 1、2 和 3 秒）。

`when_all_succeed()` 的示例，但有异常：


```cpp
using namespace seastar;
future<> f() {
    using namespace std::chrono_literals;
    return when_all_succeed(make_ready_future<int>(2),
                    make_exception_future<double>("oops")
            ).then([] (int i, double d) {
        std::cout << i << " " << d << "\n";
    }).handle_exception([] (std::exception_ptr e) {
        std::cout << "exception: " << e << "\n";
    });
}

```

在这个例子中，其中一个futures失败，所以`when_all_succeed`的结果是一个失败的future，所以正常的`continuation`没有运行，`handle_exception()` continuation就完成了。

TODO：还要解释用于 vectors 的 when_all 和when_all_succeed。

# Semaphores（信号量）
Seastar 的信号量是标准的计算机科学信号量，适用于未来。 信号量是一个计数器，您可以将单位存入或取走。 如果没有足够的单位可用，从计数器取走单位可能需要等待。

## 用信号量限制并行性
Seastar 中信号量的最常见用途是限制并行性，即限制可以并行运行的某些代码的实例数量。 当每个并行调用使用有限的资源（例如，内存）时，这可能很重要，因此让无限数量的并行调用可以耗尽此资源。

考虑外部事件源（例如，传入的网络请求）导致异步函数 g() 被调用的情况。 想象一下，我们要将并发 g() 操作的数量限制为 100。即，如果 g() 在其他 100 个调用仍在进行时启动，我们希望它延迟其实际工作，直到其他调用之一完成。 我们可以用信号量做到这一点：


```cpp
seastar::future<> g() {
    static thread_local seastar::semaphore limit(100);
    return limit.wait(1).then([] {
        return slow(); // do the real work of g()
    }).finally([] {
        limit.signal(1);
    });
}

```

在这个例子中，信号量以计数器为 100 开始。 异步操作 slow() 仅在我们可以将计数器减一（wait(1)）时启动，并且当 slow() 完成时，无论成功还是异常 ，计数器加一（信号（1））。 这样，当 100 个操作已经开始工作并且尚未完成时，第 101 个操作将等待，直到其中一个正在进行的操作完成并将一个单元返回给信号量。 这确保了每次我们在上面的代码中最多运行 100 个并发的 slow() 操作。

请注意我们如何使用静态 thread_local 信号量，以便从同一分片对 g() 的所有调用都计入相同的限制。像往常一样，一个 Seastar 应用程序是分片的，所以这个限制是单独的每个分片（CPU 线程）。 这通常很好，因为分片应用程序认为每个分片的资源是分开的。

幸运的是，上面的代码恰好是异常安全的：limit.wait(1) 可以在内存不足时抛出异常（保留一个等待者列表），在这种情况下，信号量计数器不会减少，但后续的`continuation`是 不运行所以它也不会增加。limit.wait(1) 也可以在信号量被破坏时返回一个特殊的未来（我们将在后面讨论）但在这种情况下额外的 signal() 调用被忽略。最后，slow() 也可能抛出或返回一个特殊的未来，但 finally() 确保信号量仍然增加。

但是，随着应用程序代码变得越来越复杂，无论发生哪个代码路径或异常，都很难确保我们在操作完成后永远不会忘记调用 signal()。 作为可能出错的示例，请考虑以下错误代码片段，它与上面的代码片段略有不同，而且乍一看似乎是正确的：


```cpp
seastar::future<> g() {
    static thread_local seastar::semaphore limit(100);
    return limit.wait(1).then([] {
        return slow().finally([] { limit.signal(1); });
    });
}

```

但是这个版本不是异常安全的：考虑如果slow() 在返回一个`future` 之前抛出一个异常会发生什么（这与slow() 返回一个异常的`future`不同，我们在有关异常处理的部分讨论了这种差异）。在这种情况下，我们减少了计数器，但 finally() 永远不会到达，并且计数器永远不会增加。有一种方法可以修复此代码，即用 seastar::futurize_invoke(slow) 替换对 slow() 的调用。 但是我们在这里试图说明的不是如何修复有问题的代码，而是通过使用单独的 semaphore::wait() 和 semaphore::signal() 函数，您很容易出错。

为了异常安全，在C++中一般不推荐有单独的资源获取和释放函数。 相反，C++ 提供了更安全的机制来获取资源（在这种情况下是信号量单元）并稍后释放它：lambda 函数和 RAII（“资源获取即初始化”）：

基于 lambda 的解决方案是一个函数 seastar::with_semaphore() ，它是上面示例中代码的快捷方式：


```cpp
seastar::future<> g() {
    static thread_local seastar::semaphore limit(100);
    return seastar::with_semaphore(limit, 1, [] {
        return slow(); // do the real work of g()
    });
}

```

with_semaphore() 与前面的代码片段一样，等待来自信号量的给定数量的单位，然后运行给定的 lambda，当 lambda 返回的`future`被解析时，with_semaphore() 将单位返回给信号量。 with_semaphore() 返回一个只有在所有这些步骤完成后才能解析的`future`。

函数 seastar::get_units() 更通用。 它基于 C++ 的 RAII 哲学，为 seastar::semaphore 的单独 wait() 和 signal() 方法提供了一个异常安全的替代方案。该函数返回一个不透明的单位对象，该对象在保持时保持信号量的计数器减少 - 一旦该对象被析构，计数器就会增加回来。 有了这个接口，你不能忘记增加计数器，或增加两次，或增加而不减少。

在创建单位对象时，计数器将始终减少一次，如果成功，则在对象被销毁时增加。 当units对象被移动到continuation中时，不管continuation如何结束，当continuation被破坏时，units对象被破坏并且units被返回给信号量的计数器。 上面使用 get_units() 编写的示例如下所示：


```cpp
seastar::future<> g() {
    static thread_local semaphore limit(100);
    return seastar::get_units(limit, 1).then([] (auto units) {
        return slow().finally([units = std::move(units)] {});
    });
}

```

请注意需要使用 get_units() 的有点复杂的方式：必须嵌套continuation，因为我们需要将单位对象移动到最后一个continuation。 如果slow() 返回一个future（并且不会立即抛出），finally() 继续捕获units 对象，直到一切都完成，但不运行任何代码。

Seastars 程序员通常应该避免直接使用 semaphore::wait() 和 semaphore::signal() 函数，并且总是优先 with_semaphore()（如果适用）或 get_units()。

## 限制资源利用量
因为信号量支持等待任意数量的单元，而不仅仅是 1，所以我们可以将它们用于不仅仅是限制并行调用的数量。 例如，假设我们有一个异步函数 using_lots_of_memory(size_t bytes)，它使用 bytes 字节的内存，并且我们希望确保该函数的所有并行调用使用的内存（合起来）不超过 1 MB --- 以及额外的 调用会延迟到之前的调用完成。 我们可以用信号量做到这一点：

```cpp
seastar::future<> using_lots_of_memory(size_t bytes) {
    static thread_local seastar::semaphore limit(1000000); // limit to 1MB
    return seastar::with_semaphore(limit, bytes, [bytes] {
        // do something allocating 'bytes' bytes of memory
    });
}

```

注意在上面的例子中，调用 using_lots_of_memory(2000000) 将返回一个永远不会解析的future，因为信号量永远不会包含足够的单元来满足信号量等待。 using_lots_of_memory() 可能应该检查字节是否超过限制，并在这种情况下抛出异常。 seastar不会为你做这件事。

## 限制循环的并行数
上面，我们看到了一个被一些外部事件调用的函数 g()，并希望控制它的并行性。 在本节中，我们看看循环的并行性，它也可以用信号量来控制。

考虑以下简单循环：

```cpp
#include <seastar/core/sleep.hh>
seastar::future<> slow() {
    std::cerr << ".";
    return seastar::sleep(std::chrono::seconds(1));
}
seastar::future<> f() {
    return seastar::repeat([] {
        return slow().then([] { return seastar::stop_iteration::no; });
    });
}

```

这个循环在没有任何并行性的情况下运行slow() 函数（需要一秒钟才能完成）——下一个slow() 调用仅在前一个调用完成时开始。 但是，如果我们不需要串行化对 slow() 的调用，并且希望允许它的多个实例同时进行呢？

朴素的做法，我们可以通过在上一次调用之后立即开始下一次对slow() 的调用来实现更多的并行性——忽略上一次对slow() 的调用返回的future，而不是等待它解决：

```cpp
seastar::future<> f() {
    return seastar::repeat([] {
        slow();
        return seastar::stop_iteration::no;
    });
}

```

但在这个循环中，并行度没有限制——在第一个返回之前，数百万个 sleep() 调用可能并行活动。 最终，这个循环可能会消耗所有可用内存并崩溃。

使用信号量允许我们并行运行多个 slow() 实例，但将这些并行实例的数量限制为（在以下示例中）为 100：

```cpp
seastar::future<> f() {
    return seastar::do_with(seastar::semaphore(100), [] (auto& limit) {
        return seastar::repeat([&limit] {
            return limit.wait(1).then([&limit] {
                seastar::futurize_invoke(slow).finally([&limit] {
                    limit.signal(1); 
                });
                return seastar::stop_iteration::no;
            });
        });
    });
}

```

请注意此代码与我们在上面看到的用于限制函数 g() 并行调用次数的代码有何不同：
1. 这里我们不能使用单个 thread_local 信号量。 对 f() 的每次调用都有其并行度为 100 的循环，因此需要自己的信号量“限制”，在循环期间使用 do_with() 保持活动状态
2. 在这里，我们在继续循环之前不等待 slow() 完成，即我们不返回从 futurize_invoke(slow) 开始的future链。 当信号量单元可用时，循环继续进行下一次迭代，而（在我们的示例中）99 个其他操作可能在后台进行，我们不会等待它们。

在本节的示例中，我们不能使用 with_semaphore() 快捷方式。 with_semaphore() 返回一个future ，它只在 lambda 返回的future 解析后解析。 但是在上面的例子中，循环需要知道什么时候只有信号量单元可用，才能开始下一次迭代——而不是等待前一次迭代完成。 我们无法通过 with_semaphore() 实现这一点。 但是在这种情况下可以使用更通用的异常安全习惯用法 seastar::get_units()，并且推荐使用：

```cpp
seastar::future<> f() {
    return seastar::do_with(seastar::semaphore(100), [] (auto& limit) {
        return seastar::repeat([&limit] {
    	    return seastar::get_units(limit, 1).then([] (auto units) {
	            slow().finally([units = std::move(units)] {});
	            return seastar::stop_iteration::no;
	        });
        });
    });
}

```

