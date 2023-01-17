## 一个基于C++11的高性能的、跨平台的、简单易用的线程池框架（thread pool framework）

**Hipe **是基于 C++11 编写的跨平台的、高性能的、简单易用且功能强大的线程池框架（thread pool framework），每秒能够空跑**上百万**的任务。其内置了两个职责分明的线程池（SteadyThreadPond和DynamicThreadPond），并提供了诸如任务包装器、计时器、同步IO流、自旋锁等实用的工具。

使用者可以根据业务类型结合使用Hipe-Dynamic和Hipe-Steady两种线程池来提供高并发服务。

bilibili源码剖析视频：https://space.bilibili.com/499976060 （更新中）

## demo-最简单调用

```C++
#include "./Hipe/hipe.h" 
using namespace hipe;

// SteadyThreadPond是Hipe的核心线程池类 
SteadyThreadPond pond(8);

// 提交任务，没有返回值。传入lambda表达式或者其它可调用类型
// util::print()是Hipe提供的标准输出接口，让调用者可以像写python一样简单
pond.submit([]{ util::print("HanYa said ", "hello world\n"); });

// 主线程等待所有任务被执行
pond.waitForTasks();

// 主动关闭线程池，否则当线程池类被析构时由线程池自动调用
pond.close();

```

更多接口的调用请大家阅读`hipe/interface_test/`，里面有全部的接口测试，并且每一个函数调用都有较为详细的注释。



## Hipe-SteadyThreadPond

​	（以下简称Hipe-Steady）Hipe-Steady是Hipe提供的稳定的、具有固定线程数的线程池。支持批量提交任务和批量执行任务、支持有界任务队列和无界任务队列、支持池中线程的**任务窃取**。任务溢出时支持**注册回调**并执行或者**抛出异常**。

​	Hipe-Steady内部为每个线程都分配了公开任务队列、缓冲任务队列和控制线程的同步变量（thread-local机制），尽量降低**乒乓缓存**和**线程同步**对线程池性能的影响。工作线程通过队列替换**批量下载**公开队列的任务到缓冲队列中执行。生产线程则通过公开任务队列为工作线程**分配任务**（采用了一种优于轮询的**负载均衡**机制）。通过公开队列和缓冲队列（或说私有队列）替换的机制进行**读写分离**，再通过加**轻锁**（C++11原子量实现的自旋锁）的方式极大地提高了线程池的性能。

​	 由于其底层的实现机制，Hipe-Steady适用于**稳定的**（避免超时任务阻塞线程）、**任务量大**（任务传递的优势得以体现）的任务流。也可以说Hipe-Steady适合作为核心线程池（能够处理基准任务并长时间运行），而当可以**定制容量**的Hipe-Steady面临任务数量超过设定值时—— 即**任务溢出**时，我们可以通过定制的**回调**函数拉取出溢出的任务，并把这些任务推到我们的动态线程池DynamicThreadPond中。在这个情景中，DynamicThreadPond或许可以被叫做CacheThreadPond。关于二者之间如何协调运作，大家可以阅读`Hipe/demo/demo1`.在这个demo中我们展示了如何把DynamicThreadPond用作Hipe-Steady的缓冲池。

#### interface: SteadyThreadPond

```c++
 SteadyThreadPond(uint thread_numb = 0, uint task_capacity = HipeUnlimited);
```

`thread_numb`：固定的线程数

`task_capacity`：线程池阻塞的最大任务数。

​	但为了减少同步的开销，Hipe是在Hipe-Steady的**线程中**记录存留的任务数，而不是在线程池层面维护一个原子量来记录。因此，传入的`task_capacity`会被**除以**传入的线程数，作为每个线程的最大阻塞任务数。该除法为除后向下取整（floor division），且每个线程的最大阻塞任务数最小为1，因此当`task_capacity`不是`thread_numb`的整数倍时。结果可能会与调用者的预期略有出入。



#### interface: submitInBatch

若想要再提高SteadyThreadPond的性能，批量提交是最好的方法。Hipe-Steady提供了十分简便的批量提交任务的接口。

```C++
// 主线程会将一批任务提交给单个线程，因此不能让批量提交的单位过大，这样不利于线程池内线程的负载均衡。
std::vector<std::function<void()>> container(10, []{});
pond.submitInBatch(container, container.size());
```



## Hipe-DynamicThreadPond

​	 （以下简称Hipe-Dynamic）Hipe-Dynamic是Hipe提供的动态的、能够扩缩容的线程池。支持批量提交任务、支持线程吞吐任务速率监测、支持无界队列。当没有任务时所有线程会被自动挂起（阻塞等待条件变量的通知），较为节能。

​	Hipe-Dynamic采用的是多线程竞争单条任务队列的模型。该任务队列是无界的，能够容蓄大量的任务（直至系统资源耗尽）。由于Hipe-Dynamic管理的线程没有私有的任务队列，因此能够被灵活地调度。同时，为了监测任务的加载速率（线程消化任务的速度），Hipe-Dynamic提供了监测任务队列的接口，其使用实例在`Hipe/demo/demo2`。

​	由于Hipe-Dynamic的接口较为简单，如果需要了解更多接口的调用，可以阅读接口测试文件`Hipe/interface_test/`或者`Hipe/demo/demo2`。



## BenchMark

​	[bshoshany](https://github.com/bshoshany)/**[thread-pool](https://github.com/bshoshany/thread-pool)** （以下简称BS）是在GitHub上开源的在大约五个月的时间内就收获了**1k+stars** 的C++线程池，采用C++17编写，具有轻量，高效的特点。我们通过**加速比测试和空任务测试**，对比BS和Hipe的性能。实际上BS的底层机制与Hipe-Dynamic相似，都是多线程竞争一条任务队列，并且在没有任务时被条件变量阻塞。

​	测试机器：16核_ubuntu20.04

### 加速比测试

- 测试原理： 通过执行计算密集型的长任务，与单线程进行对比，进而算出线程池的加速比。每次测试都会重复5遍并取平均值。

```C++
// 任务类型
uint vec_size = 4096;
uint vec_nums = 2048;
std::vector<std::vector<double>> results(vec_nums, std::vector<double>(vec_size));

// computation intensive task(计算密集型任务)
void computation_intensive_task() {
    for (int i = 0; i < vec_nums; ++i) {
        for (size_t j = 0; j < vec_size; ++j) {
            results[i][j] = std::log(std::sqrt(std::exp(std::sin(i) + std::cos(j))));
        }
    }
}
```



```
+++++++++++++++++++++++++++++++++++++++++++++++++++++++
+           Test Single-thread Performance            +
+++++++++++++++++++++++++++++++++++++++++++++++++++++++

threads: 1  | task-type: compute mode | task-numb: 4  | time-cost-per-task: 344.30398(ms)

++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
+           Test C++(11) Thread-Pool Hipe-Steady             +
++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++

threads: 16 | task-type: compute mode | task-numb: 4  | time-cost-per-task: 153.04224(ms)
threads: 16 | task-type: compute mode | task-numb: 16 | time-cost-per-task: 40.14723(ms)
threads: 16 | task-type: compute mode | task-numb: 28 | time-cost-per-task: 44.97041(ms)
threads: 16 | task-type: compute mode | task-numb: 40 | time-cost-per-task: 48.64367(ms)
threads: 16 | task-type: compute mode | task-numb: 52 | time-cost-per-task: 50.56839(ms)
threads: 16 | task-type: compute mode | task-numb: 64 | time-cost-per-task: 41.96570(ms)
Best speed-up obtained by multithreading vs. single-threading: 8.58, using 16 tasks

+++++++++++++++++++++++++++++++++++++++++++++++++++++
+           Test C++(17) Thread-Pool BS             +
+++++++++++++++++++++++++++++++++++++++++++++++++++++

threads: 16 | task-type: compute mode | task-numb: 4  | time-cost-per-task: 91.33461(ms)
threads: 16 | task-type: compute mode | task-numb: 16 | time-cost-per-task: 40.63790(ms)
threads: 16 | task-type: compute mode | task-numb: 28 | time-cost-per-task: 43.87851(ms)
threads: 16 | task-type: compute mode | task-numb: 40 | time-cost-per-task: 45.61345(ms)
threads: 16 | task-type: compute mode | task-numb: 52 | time-cost-per-task: 44.25607(ms)
threads: 16 | task-type: compute mode | task-numb: 64 | time-cost-per-task: 40.58885(ms)
Best speed-up obtained by multithreading vs. single-threading: 8.48, using 64 tasks

+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
+           Test C++(11) Thread-Pool Hipe-Dynamic           +
+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++

threads: 16 | task-type: compute mode | task-numb: 4  | time-cost-per-task: 97.45932(ms)
threads: 16 | task-type: compute mode | task-numb: 16 | time-cost-per-task: 39.93850(ms)
threads: 16 | task-type: compute mode | task-numb: 28 | time-cost-per-task: 43.26275(ms)
threads: 16 | task-type: compute mode | task-numb: 40 | time-cost-per-task: 44.35939(ms)
threads: 16 | task-type: compute mode | task-numb: 52 | time-cost-per-task: 44.71270(ms)
threads: 16 | task-type: compute mode | task-numb: 64 | time-cost-per-task: 39.96931(ms)
Best speed-up obtained by multithreading vs. single-threading: 8.62, using 16 tasks

+++++++++++++++++++++++++++++++++++++++++++++
+              End of the test              +
+++++++++++++++++++++++++++++++++++++++++++++
```

- 结果分析：可以看到线程池BS的最佳加速比为**8.48倍**， Hipe-Steady线程池的最佳加速比为**8.58倍**，Hipe-Dynamic的最佳加速比为**8.62倍**。三者的性能接近，说明在任务传递过程开销较小的情况下（由于任务数较少），**乒乓缓存、线程切换和线程同步**等因素对三种种线程池的加速比的影响是相近的。

### 空任务测试

- 测试原理： 通过提交大量的空任务到线程池中，对比两种线程池处理空任务的能力，其主要影响因素为**任务传递**、**线程同步**等的开销。

```

+++++++++++++++++++++++++++++++++++++++++++++
+   Test C++(11) Thread Pool Hipe-Dynamic   +
+++++++++++++++++++++++++++++++++++++++++++++
threads: 16 | task-type: empty task | task-numb: 100     | time-cost: 0.00142(s)
threads: 16 | task-type: empty task | task-numb: 1000    | time-cost: 0.01095(s)
threads: 16 | task-type: empty task | task-numb: 10000   | time-cost: 0.09778(s)
threads: 16 | task-type: empty task | task-numb: 100000  | time-cost: 0.97992(s)
threads: 16 | task-type: empty task | task-numb: 1000000 | time-cost: 9.83240(s)

+++++++++++++++++++++++++++++++++++
+   Test C++(17) Thread Pool BS   +
+++++++++++++++++++++++++++++++++++
threads: 16 | task-type: empty task | task-numb: 100     | time-cost: 0.00133(s)
threads: 16 | task-type: empty task | task-numb: 1000    | time-cost: 0.01085(s)
threads: 16 | task-type: empty task | task-numb: 10000   | time-cost: 0.09667(s)
threads: 16 | task-type: empty task | task-numb: 100000  | time-cost: 0.99130(s)
threads: 16 | task-type: empty task | task-numb: 1000000 | time-cost: 9.98166(s)

++++++++++++++++++++++++++++++++++++++++++++
+   Test C++(11) Thread Pool Hipe-Steady   +
++++++++++++++++++++++++++++++++++++++++++++
threads: 16 | task-type: empty task | task-numb: 100     | time-cost: 0.00040(s)
threads: 16 | task-type: empty task | task-numb: 1000    | time-cost: 0.00093(s)
threads: 16 | task-type: empty task | task-numb: 10000   | time-cost: 0.00672(s)
threads: 16 | task-type: empty task | task-numb: 100000  | time-cost: 0.07198(s)
threads: 16 | task-type: empty task | task-numb: 1000000 | time-cost: 0.68551(s)

+++++++++++++++++++++++++++++++++++++++++++++
+              End of the test              +
+++++++++++++++++++++++++++++++++++++++++++++
```

- 结果分析： 可以看到在处理空任务这一方面Hipe-Steady具有**巨大的优势**，在处理**1000000**个空任务时性能是BS和Hipe-Dynamic的**10倍以上**（如果采用批量提交接口能达到约**20倍以上**的性能）。而且随着任务数增多Steady线程池也并未呈现出指数级的增长趋势，而是呈常数级的增长趋势。即随着任务增多而线性增长。



## 文件树

```
.
├── README.md									本文档
├── benchmark 									性能测试
│   ├── BS_thread_pool.hpp 						BS线程池
│   ├── makefile
│   ├── test_empty_task.cpp 					跑空任务
│   └── test_speedup.cpp 						测加速比
├── demo
│   ├── demo1.cpp 								如何将Hipe-Dynamic作为缓冲池
│   └── demo2.cpp 								如何根据流量动态调节Hipe-Dynamic的线程数
├── dynamic_pond.h 								Hipe-Dynamic
├── header.h 									一些别名和引入的头文件	
├── hipe.h 										方便导入的文件，已将Hipe的头文件包含了 
├── interface_test 								接口测试
│   ├── makefile
│   ├── test_dynamic_pond_interface.cpp			Hipe-Dynamic的接口测试
│   └── test_steady_pond_interface.cpp 			Hipe-Steady的接口测试
├── steady_pond.h 								Hipe-Steady
└── util.h 										工具包：计时器、任务包装器、同步IO流...
```



## 鸣谢

一直支持我的女朋友小江和我的父母、姐姐。

## 联系我

QQ邮箱：1848395727@qq.com

欢迎向我提问，也希望我们能一起把Hipe变得更好，一起在C++线程池这方面做些贡献。