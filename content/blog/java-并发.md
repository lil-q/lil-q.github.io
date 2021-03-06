---
title: Java：并发
date: 2020-04-28 19:37:01
tags: [java, jvm]
toc: true
description: "java并发内存模型线程池线程J安全锁优化"
keywords: [jmm, java, concurrency]
---

计算机的运算速度与它的存储和通信子系统的速度差距太大，大量的时间都花费在磁盘 I/O、网络通信或者数据库访问上。高效并发能能更好的利用计算机的性能。

## 一、硬件的效率和一致性

由于计算机的存储设备与处理器的运算速度有着几个数量级的差距，所以现代计算机系统都不得不加入一层或多层读写速度尽可能接近处理器运算速度的**高速缓存**（ Cache）来作为内存与处理器之间的缓冲：将运算需要使用的数据复制到缓存中，让运算能快速进行，当运算结束后再从缓存同步回内存之中，这样处理器就无须等待缓慢的内存读写了。

基于高速缓存的存储交互很好地解决了处理器与内存速度之间的矛盾，但是也为计算机系统带来更高的复杂度，它引入了一个新的问题：**缓存一致性**（ Cache Coherence）。在多路处理器系统中，每个处理器都有自己的高速缓存，而它们又共享同一主内存（ Main Memory），这种系统称为共享内存多核系统（ Shared Memory Multiprocessors System），如下图所示。当多个处理器的运算任务都涉及同一块主内存区域时，将可能导致各自的缓存数据不一致。

<img src="https://qttblog.oss-cn-hangzhou.aliyuncs.com/june/concurconcurency3.png" style="zoom:150%;" />

为了解决一致性的问题，需要各个处理器访问缓存时都遵循一些协议，在读写时要根据协议来进行操作。

除了増加高速缓存之外，为了使处理器内部的运算单元能尽量被充分利用，处理器可能会对输入代码进行**乱序执行（Out-Of-Order Execution）优化**，让指令的执行能够尽可能的并行起来。处理器会在计算之后将乱序执行的结果重组，保证该结果与顺序执行的结果是一致的，但并不保证程序中各个语句计算的先后顺序与输入代码中的顺序一致，因此如果存在一个计算任务依赖另外一个计算任务的中间结果，那么其顺序性并不能靠代码的先后顺序来保证。与处理器的乱序执行优化类似，Java 虚拟机的即时编译器中也有**指令重排序（ Instruction Reorder）优化**。

即使是单线程，指令重排仍然能加速执行，因为一个**非原子**操作指令也会涉及到很多步骤（取指令、指令译码、执行指令、访存取数、结果写回），每个步骤可能会用到不同的寄存器，CPU 使用了**流水线技术**，也就是说，CPU 有多个功能单元，一条指令也分为多个单元，那么第一条指令执行还没完毕，就可以执行第二条指令，前提是这两条指令功能单元相同或类似，所以一般可以通过指令重排使得具有相似功能单元的指令接连执行来**减少流水线中断**的情况。

## 二、Java 内存模型

Java 内存模型试图屏蔽各种硬件和操作系统的内存访问差异，以实现让 Java 程序在各种平台下都能达到一致的内存访问效果。Java 内存模型的主要目的是定义程序中各种变量的访问规则，即关注在虚拟机中把变量值存储到内存和从内存中取出变量值这样的底层细节。

> 此处的变量（Variables）与 Java 编程中所说的变量有所区别，它包括了**实例字段、静态字段和构成数组对象的元素**，但是不包括局部变量与方法参数，因为后者是线程私有的，不会被共享，自然就不会存在竞争问题。

Java 内存模型规定了所有的变量都存储在主内存（ Main Memory，与硬件的主内存同名，但物理上它仅是虚拟机内存的一部分）。每条线程还有自己的工作内存（ Working Memory，可与前面讲的处理器高速缓存类比），线程的工作内存中保存了被该线程使用的变量的主内存副本，线程对变量的所有操作（读取、赋值等）都必须在工作内存中进行，而不能直接读写主内存中的数据。不同的线程之间也无法直接访问对方工作内存中的变量，线程间变量值的传递均需要通过主内存来完成（如果局部变量是一个 reference 类型，它引用的对象在 Java 堆中可被各个线程共享，但是 reference 本身在 Java 栈的局部变量表中是线程私有的）。

<img src="https://qttblog.oss-cn-hangzhou.aliyuncs.com/june/concurconcurency4.png" style="zoom:150%;" />

### 2.1 内存间交互操作

Java 内存模型定义了 8 个操作来完成主内存和工作内存的交互操作。Java虚拟机实现时必须保证下面提及的每一种操作都是原子的、不可再分的（对于 double 和 long 类型的变量来说， load、 store、read 和 write 操作在某些平台上允许有例外）。

<img src="https://qttblog.oss-cn-hangzhou.aliyuncs.com/june/concurconcurency5.png" style="zoom:150%;" />

- **lock**：作用于主内存的变量，标识为一条线程独占。
- **unlock**：作用于主内存的变量，释放处于锁定状态的变量。
- **read**：作用于主内存的变量，传输到工作内存中。
- **load**：作用于工作内存的变量，在 read 之后执行，把值放入工作内存的变量副本中。
- **use**：作用于工作内存的变量，传递给执行引擎。
- **assign**：作用于工作内存的变量，把一个从执行引擎接收到的值赋给工作内存的变量。
- **store**：作用于工作内存的变量，把一个变量的值传送到主内存中。
- **write**：作用于主内存的变量，在 store 之后执行，把值放入主内存的变量中。

### 2.2 volatile

关键字 *volatile* 可以说是 Java 虚拟机提供的最轻量级的同步机制，它将具备两项特性：**对所有线程的可见性**和**禁止指令重排序优化**。具体规则如下：

1. 在工作内存中，每次使用 Ⅴ 前都必须先**从主内存刷新最新的值**，用于保证能看见其他线程对变量所做的修改。即 use 动作 与 load、read 动作相关联，必须连续同时出现。

2. 在工作内存中，每次修改Ⅴ后都必须**立刻同步回主内存中**，用于保证其他线程可以看到自己对变量 V 所做的修改。即 assign 动作与 store、write 动作相关联的，必须连续同时出现。

3. *volatile* 修饰的变量**不会被指令重排序优化**，保证代码的执行顺序与程序的顺序相同。

只有一个处理器访问内存时，并不需要内存屏障；但如果有两个或更多处理器访问同一块内存，且其中有一个在观测另一个，就需要内存屏障来保证一致性了。在程序运行过程中，所有的变更会先在寄存器或本地 cache 中完成，然后才会被拷贝到主存以跨越内存栅栏，此种跨越序列或顺序称为 happens-before。Java 内存屏障主要有 Load 和 Store 两类：

* **Load Barrier**：读指令前插入读屏障，让高速缓存中数据失效，对数据的读取需要重新从主内存加载；
* **Store Barrier**：写指令之后插入写屏障，能让写入缓存的最新数据写回到主内存。

在实际使用中，又分为以下四种：

* **LoadLoad 屏障**：`Load1; LoadLoad; Load2;` 在 Load2 及后续读取操作要读取的数据被访问前，保证 Load1 要读取的数据被读取完毕。
* **StoreStore 屏障**：`Store1; StoreStore; Store2;` 在 Store2 及后续写入操作执行前，保证 Store1 的写入操作对其它处理器可见。
* **LoadStore 屏障**：`Load1; LoadStore; Store2;` 在 Store2 及后续写入操作被刷出前，保证 Load1 要读取的数据被读取完毕。
* **StoreLoad 屏障**：`Store1; StoreLoad; Load2;` 在 Load2 及后续所有读取操作执行前，保证 Store1 的写入对所有处理器可见。它的开销是四种屏障中最大的。在大多数处理器的实现中，这个屏障是个万能屏障，兼具其它三种内存屏障的功能。

### 2.3 三个特性

<img src="https://qttblog.oss-cn-hangzhou.aliyuncs.com/june/concurency1.png" style="zoom:150%;" />

#### 1. 原子性

Java 内存模型保证了 read、load、use、assign、store、write、lock 和 unlock **单个操作**具有原子性，例如对一个 int 类型的变量执行 assign 赋值操作，这个操作就是原子性的。但是 Java 内存模型允许虚拟机将没有被 volatile 修饰的 64 位数据（long，double）的读写操作划分为两次 32 位的操作来进行，即 load、store、read 和 write 操作可以不具备原子性。*AtomicInteger* 能保证多个线程修改的原子性。

```java
private AtomicInteger cnt = new AtomicInteger();
```

除了使用原子类之外，也可以使用 *synchronized* 互斥锁来保证操作的原子性。它对应的内存间交互操作为：lock 和 unlock，在虚拟机实现上对应的字节码指令为 monitorenter 和 monitorexit。

```java
public synchronized void add() {
    cnt++;
}

public synchronized int get() {
    return cnt;
}
```

#### 2. 可见性

可见性指当一个线程修改了共享变量的值，其它线程能够**立即得知**这个修改。Java 内存模型是通过在变量修改后将新值同步回主内存，在变量读取前从主内存刷新变量值来实现可见性的。主要有三种实现可见性的方式：

- *volatile*；
- *synchronized*，对一个变量执行 unlock 操作之前，必须把变量值同步回主内存；
- *final*，被 *final* 关键字修饰的字段在构造器中一旦初始化完成，并且没有发生 *this* 逃逸（其它线程通过 *this* 引用访问到初始化了一半的对象），那么其它线程就能看见 *final* 字段的值。

对前面的线程不安全示例中的 *cnt* 变量使用 *volatile* 修饰，不能解决线程不安全问题，因为 *volatile* 并**不能**保证操作的**原子性**。

#### 3. 有序性

Java 程序中天然的有序性可以总结为一句话：如果在本线程内观察，所有的操作都是有序的：如果在一个线程中观察另一个线程所有的操作都是无序的。前半句是指 “线程内似表现为串行的语义”，后半句是指 “指令重排序现象” 和工作内存与主内存同步延迟现象。Java 语言提供了 *volatile* 和 *synchronized* 两个关键字来保证线程之间操作的有序性， *volatile* 关键字本身就包含了**禁止指令重排序**的语义，而 *synchronized* 则是由 “一个变量在同一个时刻只允许一条线程对其进行 lock 操作” 这条规则获得的，这个规则决定了持有同一个锁的两个同步块只能**串行**地进入。

### 2.4 先行发生原则

**1. 单一线程原则（Single Thread rule）**

在一个线程内，在程序前面的操作先行发生于后面的操作。

**2. 管程锁定规则（Monitor Lock Rule）**

一个 unlock 操作先行发生于后面对同一个锁的 lock 操作。

**3. *volatile* 变量规则（Volatile Variable Rule）**

对一个 *volatile* 变量的写操作先行发生于后面对这个变量的读操作。

**4. 线程启动规则（Thread Start Rule）**

Thread 对象的 `start()` 方法调用先行发生于此线程的每一个动作。

**5. 线程加入规则（Thread Join Rule）**

Thread 对象的结束先行发生于 `join()` 方法返回。

**6. 线程中断规则（Thread Interruption Rule）**

对线程 `interrupt()` 方法的调用先行发生于被中断线程的代码检测到中断事件的发生，可以通过 `interrupted()` 方法检测到是否有中断发生。

**7. 对象终结规则（Finalizer Rule）**

一个对象的初始化完成（构造函数执行结束）先行发生于它的 `finalize()` 方法的开始。

**8. 传递性（Transitivity）**

如果操作 A 先行发生于操作 B，操作 B 先行发生于操作 C，那么操作 A 先行发生于操作 C。

## 三、线程

主流的操作系统都提供了线程实现， Java 语言则提供了在不同硬件和操作系统平台下对线程操作的统一处理，每个已经调用过 `start()` 方法且还未结束的 Java.lang.Thread 类的实例就代表着一个线程。有三种使用线程的方法：

- 实现 *Runnable* 接口（可用 lambda 表达式）：

  ```java
  MyRunnable mr = new MyRunnable();
  Thread thread = new Thread(mr);
  ```

- 实现 *Callable* 接口：

  ```java
  MyCallable mc = new MyCallable();
  FutureTask<Integer> ft = new FutureTask<>(mc);
  Thread thread = new Thread(ft);
  ```

- 继承 *Thread* 类：

  ```java
  MyThread mt = new MyThread();
  ```

实现接口会更好一些，因为：

- Java 不支持多重继承，继承 *Thread* 类就无法继承其它类，但是可以实现多个接口；
- 类可能只要求可执行就行，继承整个 *Thread* 类开销过大。

通过调用 `start()` 启动一个新线程；一个线程对象只能调用一次 `start()` 方法；必须调用 *Thread* 实例的 `start()` 方法才能启动新线程，`start()` 方法内部调用了一个 `private native void start0()` 方法，*native* 修饰符表示这个方法是由 JVM 虚拟机内部的 C 代码实现的。

### 3.1 实现

实现线程主要有三种方式：使用内核线程实现（1:1 实现），使用用户线程实现（1：N 实现），使用用户线程加轻量级进程混合实现（N:M 实现）。

#### 1. 内核线程 1:1 实现

**内核线程**（ **K**ernel- **L**evel **T**hread，**KLT**）就是直接由操作系统内核支持的线程，这种线程由内核通过操纵调度器（ Scheduler）对线程进行调度，并负责将线程的任务映射到各个处理器上。程序一般不会直接使用内核线程，而是使用内核线程的一种高级接口——**轻量级进程**（ **L**ight **W**eight **P**rocess，**LWP**），轻量级进程就是我们通常意义上所讲的线程。

#### 2. 用户线程 1:N 实现

狭义上的**用户线程**（ **U**ser **T**hread，**UT**）指的是完全建立在用户空间的线程库上，系统内核不能感知到用户线程的存在及如何实现的。用户线程的建立、同步、销毁和调度完全在用户态中完成，不需要内核的帮助。

#### 3. 用户线程加轻量级进程混合 N:M 实现

在这种混合实现下，既存在用户线程，也存在轻量级进程。用户线程还是完全建立在用户空间中，因此用户线程的创建、切换、析构等操作依然廉价，并且可以支持大规模的用户线程并发。而操作系统支持的轻量级进程则作为用户线程和内核线程之间的桥梁这样可以使用内核提供的线程调度功能及处理器映射，并且用户线程的系统调用要通过轻量级进程来完成，这**大大降低了整个进程被完全阻塞的风险**。

#### 4. 对比

1. 内核线程**有限**，用户线程能够支持**规模更大的线程数量**；

   每个轻量级进程都需要有一个内核线程的支持，因此轻量级进程要消耗一定的内核资源（如内核线程的栈空间）；部分高性能数据库中的多线程是由用户线程实现的。

2. 内核线程操作系统调用的**代价相对较高**，用户线程**快速且低消耗**；

   内核线程操作，如创建、析构及同步，都需要进行系统调用。而系统调用的代价相对较高，需要在用户态（ User mode）和内核态（ Kernel mode）中来回切换。

3. 用户线程的劣势是没有系统内核的支援，**所有的线程操作都需要由用户程序自己去处理**。

   线程的创建、销毁、切换和调度都是用户必须考虑的问题，而且由于操作系统只把处理器资源分配到进程，那诸如“阻塞如何处理”多处理器系统中如何将线程映射到其他处理器上”这类问题解决起来将会异常困难，甚至有些是不可能实现的。

### 3.2 方法

#### 1. Daemon

守护线程是程序运行时在后台提供服务的线程，不属于程序中不可或缺的部分。当所有非守护线程结束时，程序也就终止，同时会杀死所有守护线程。比如 `main()` 就属于非守护线程。在线程启动之前使用 `setDaemon()` 方法可以将一个线程设置为守护线程：

```java
public static void main(String[] args) {
    Thread thread = new Thread(new MyRunnable());
    thread.setDaemon(true);
}
```

#### 2. sleep()

`Thread.sleep(millisec)` 方法会休眠当前正在执行的线程，`millisec` 单位为毫秒。`sleep()` 可能会抛出 `InterruptedException`，因为异常不能跨线程传播回 `main()` 中，因此必须在**本地进行处理**。线程中抛出的其它异常也同样需要在本地进行处理：

```java
public void run() {
    try {
        Thread.sleep(3000);
    } catch (InterruptedException e) {
        e.printStackTrace();
    }
}
```

#### 3. yield()

对静态方法 `Thread.yield()` 的调用声明了当前线程已经完成了生命周期中最重要的部分，可以切换给其它线程来执行。该方法只是对线程调度器的一个建议，而且也只是建议具有相同优先级的其它线程可以运行：

```java
public void run() {
    Thread.yield();
}
```

#### 4. currentThread()

利用静态方法 `Thread.currentThread()` 可以获取当前线程：

```java
Thread.currentThread().hashCode(); // 获取当前线程 hashCode
```

### 3.3 状态

一个线程只能处于一种状态，并且这里的线程状态特指 Java 虚拟机的线程状态，不能反映线程在特定操作系统下的状态。

- **New**：新创建的线程，尚未执行；
- **Runnable**：运行中的线程，正在执行 `run()` 方法的Java代码；
- **Blocked**：运行中的线程，因为某些操作被阻塞而挂起；
- **Waiting**：运行中的线程，因为某些操作在等待中；
- **Timed Waiting**：运行中的线程，因为执行 `sleep()` 方法正在计时等待；
- **Terminated**：线程已终止，因为 `run()` 方法执行完毕。

<img src="https://qttblog.oss-cn-hangzhou.aliyuncs.com/after9.9/state-machine-example-java-6-thread-states.png" alt="states" style="zoom:150%;" />

### 3.4 中断

一个线程执行完毕之后会自动结束，如果在运行过程中发生异常也会提前结束。

#### 1. InterruptedException

通过调用一个线程的 `interrupt()` 来中断该线程，如果该线程处于阻塞、限期等待或者无限期等待状态，那么就会抛出 *InterruptedException*，从而提前结束该线程。但是不能中断 I/O 阻塞和 *synchronized* 锁阻塞。

#### 2. interrupted()

如果一个线程的 `run()` 方法执行一个无限循环，并且没有执行 `sleep()` 等会抛出 *InterruptedException* 的操作，那么调用线程的 `interrupt()` 方法就无法使线程提前结束。

但是调用 `interrupt()` 方法会设置线程的中断标记，此时调用 `interrupted()` 方法会返回 true。因此可以在循环体中使用 `interrupted()` 方法来判断线程是否处于中断状态，从而提前结束线程。

#### 3. Executor

调用 *Executor* 的 `shutdown()` 方法会等待线程都执行完毕之后再关闭，但是如果调用的是 `shutdownNow()` 方法，则相当于调用每个线程的 `interrupt()` 方法。

### 3.4 线程池

* ***corePoolSize***：核心线程数量，核心线程不会被回收，没有任务时会保持空闲状态；
* ***maximumPoolSize***：池允许最大的线程数；
* ***keepAliveTime***：非核心线程的存活时间；
* ***unit***：keepAliveTime 的单位；
* ***workQueue***：线程数超过 *corePoolSize* 时，新的任务存在 *workQueue* 中等待；
* ***threadFactory***：创建线程的工厂类；
* ***handler***：线程池执行拒绝策略。

#### 1. 线程池的优势

* 减少了创建和销毁线程的次数，每个工作线程都可以被重复利用，可执行多个任务；
* 运用线程池能有效的控制线程最大并发数，可以根据系统的承受能力，调整线程池中工作线线程的数目，防止因为消耗过多的内存；
* 对线程进行一些简单的管理，比如：延时执行、定时循环执行的策略等，运用线程池都能进行很好的实现。

#### 2. 线程池饱和策略

<img src="https://qttblog.oss-cn-hangzhou.aliyuncs.com/june/concurconcurency2.png" style="zoom:150%;" />

1. 如果当前运行的线程少于 *corePoolSize*，则创建新线程来执行任务（需获取全局锁）；
2. 如果运行的线程等于或多于 *corePoolSize*，则将任务加入 BlockingQueue ；
3. 如果无法将任务加入 BlockingQueue （队列已满），则在非 corePool 中创建新的线程来处理任务（需要获取全局锁）；
4. 如果创建新线程将使当前运行的线程超出 *maximumPoolSize*，任务将被拒绝，并调用 `RejectedExecutionHandler.rejectedExecution()` 方法。

*ThreadPoolExecutor* 采取上述步骤的总体设计思路，是为了在执行 `execute()` 方法时，尽可能地避免获取全局锁（那将会是一个严重的可伸缩瓶颈）。在 *ThreadPoolExecutor* 完成预热之后（当前运行的线程数大于等于 *corePoolSize*），几乎所有的 `execute()` 方法调用都是执行步骤 2，而步骤 2 不需要获取全局锁。线程池有四种拒绝策略：

1. AbortPolicy：丢弃任务并抛出 *RejectedExecutionException* 异常；
2. DiscardPolicy：丢弃任务，但是不抛出异常；
3. DisCardOldSetPolicy：丢弃队列最前面的任务，然后提交新来的任务；
4. CallerRunPolicy：由调用线程（提交任务的线程，主线程）处理该任务。

#### 3. 创建线程池

*Executor* 管理多个异步任务的执行，而无需程序员显式地管理线程的生命周期。这里的异步是指多个任务的执行互不干扰，不需要进行同步操作。主要有四种 *Executor*：

- *CachedThreadPool*：一个任务创建一个线程；
- *FixedThreadPool*：所有任务只能使用固定大小的线程；
- *ScheduledThreadPool*：支持定时及周期性任务执行；
- *SingleThreadExecutor*：相当于大小为 1 的 *FixedThreadPool*。

```java
public static void main(String[] args) {
    ExecutorService executorService = Executors.newCachedThreadPool();
    for (int i = 0; i < 5; i++) {
        executorService.execute(new MyRunnable());
    }
    executorService.shutdown();
}
```

#### 4. 不同场景下的选择

**（1）高并发、任务执行时间短的业务**

线程池线程数可以设置为 CPU 核数 + 1，减少线程上下文的切换。

**（2）并发不高、任务执行时间长的业务**

- IO 密集型任务，因为 IO 操作并不占用 CPU，可以适当加大线程池中的线程数目（2 * CPU 核数），让 CPU 处理更多的业务；
- CPU 密集型任务，同（1）。

**（3）并发高、业务执行时间长的业务**

解决这种类型任务的关键不在于线程池而在于整体架构的设计。

#### 5.  阻塞队列

阻塞队列方法有四种形式，它们以不同的方式处理操作：

<br>

|      | 抛出异常  | 返回特殊值 | 一直阻塞 | 超时退出             |
| ---- | --------- | ---------- | -------- | -------------------- |
| 插入 | add(e)    | offer(e)   | put(e)   | offer(e, time, unit) |
| 移除 | remove()  | poll()     | take()   | poll(time, unit)     |
| 检查 | element() | peek()     |          |                      |

* 核心线程在获取任务时，通过阻塞队列的 **`take()`** 方法实现**一直阻塞（存活）**；
* 在获取任务时通过阻塞队列的 **`poll(time,unit)`** 方法实现**延迟死亡**。

#### 6. ctl

```java
// Packing and unpacking ctl
private static int runStateOf(int c)     { return c & ~COUNT_MASK; }
private static int workerCountOf(int c)  { return c & COUNT_MASK; }
private static int ctlOf(int rs, int wc) { return rs | wc; }
```

*ctl* 是一个打包两个概念字段的原子整数。

* *workerCount*：指示线程的有效数量；

* *runState*：指示线程池的运行状态：

  RUNNING、SHUTDOWN、STOP、TIDYING、TERMINATED 等。

*int* 类型有 32 位，其中 *ctl* 的低 29 为用于表示 *workerCount*，高 3 位用于表示 *runState*。*ctl* 这么设计的主要好处是将对 *runState* 和 *workerCount* 的操作封装成了一个原子操作。*runState* 和 *workerCount* 是线程池正常运转中的 2 个最重要属性，线程池在某一时刻该做什么操作，取决于这 2 个属性的值。

因此无论是查询还是修改，我们必须保证对这 2 个属性的操作是属于**同一时刻**的，也就是原子操作，否则就会出现错乱的情况。

## 四、线程协作

当多个线程可以一起工作去解决某个问题时，如果某些部分必须在其它部分之前完成，那么就需要对线程进行协调。

### 4.1 join()

在线程中调用另一个线程的 `join()` 方法，会将当前线程挂起，而不是忙等待，直到目标线程结束。

### 4.2 wait() & notify()

调用 `wait()` 使得线程等待某个条件满足，线程在等待时会被挂起，当其他线程的运行使得这个条件满足时，其它线程会调用 `notify()` 或者 `notifyAll()` 来唤醒挂起的线程。它们都**属于 Object 的一部分**，而不属于 *Thread*。

只能用在同步方法或者同步控制块中使用，否则会在运行时抛出 *IllegalMonitorStateException*。使用 `wait()` 挂起期间，线程会**释放锁**。这是因为，如果没有释放锁，那么其它线程就无法进入对象的同步方法或者同步控制块中，那么就无法执行 `notify()` 或者 `notifyAll()` 来唤醒挂起的线程，造成死锁。

**wait() 和 sleep() 的区别**

- `wait()` 是 *Object* 的方法，而 `sleep()` 是 *Thread* 的静态方法；
- `wait()` 会释放锁，`sleep()` 不会。

### 4.3 await() & signal()

java.util.concurrent 类库中提供了 *Condition* 类来实现线程之间的协调。

```java
private Lock lock = new ReentrantLock();
private Condition condition = lock.newCondition();
```

使用 *Condition* 时，引用的 *Condition* 对象必须从 *Lock* 实例的 `newCondition()` 返回，这样才能获得一个绑定了 *Lock* 实例的 *Condition* 实例。*Condition* 提供的 `await()`、`signal()`、`signalAll()` 原理和 *synchronized* 锁对象的 `wait()`、`notify()`、`notifyAll()` 是一致的，并且其行为也是一样的：

- `await()` 会释放当前锁，进入等待状态；
- `signal()` 会唤醒某个等待线程；
- `signalAll()` 会唤醒所有等待线程；
- 唤醒线程从 `await()` 返回后需要重新获得锁。

此外，和 `tryLock()` 类似，`await()` 可以在等待指定时间后，如果还没有被其他线程通过 `signal()` 或 `signalAll()` 唤醒，可以自己醒来：

```java
if (condition.await(1, TimeUnit.SECOND)) {
    // 被其他线程唤醒
} else {
    // 指定时间内没有被其他线程唤醒
}
```

可见，使用 *Condition* 配合 *Lock*，我们可以实现更灵活的线程同步。

## 五、线程安全

当多个线程同时访问一个对象时，如果不用考虑这些线程在运行时环境下的**调度和交替执行**，也不需要进行**额外的同步**，或者在调用方进行任何其他的协调操作，**调用这个对象的行为都可以获得正确的结果**，那就称这个对象是**线程安全**的。

### 5.1 不可变

不可变（Immutable）的对象一定是线程安全的，不需要再采取任何的线程安全保障措施。只要一个不可变的对象被正确地构建出来，永远也不会看到它在多个线程之中处于不一致的状态。多线程环境下，应当尽量使对象成为不可变，来满足线程安全。不可变的类型：

- *final* 关键字修饰的基本数据类型；
- *String*；
- 枚举类型；
- *Number* 部分子类，如 *Long* 和 *Double* 等数值包装类型，*BigInteger* 和 *BigDecimal* 等大数据类型。但同为 *Number* 的原子类 *AtomicInteger* 和 *AtomicLong* 则是可变的。

对于集合类型，可以使用 `Collections.unmodifiableXXX()` 方法来获取一个不可变的集合。

### 5.2 互斥同步

互斥是实现同步的一种手段，临界区（ Critical Section）、互斥量（ Mutex）和信号量（ Semap hore）都是常见的互斥实现方式。因此在“互斥同步”这四个字里面，**互斥是因，同步是果；互斥是方法，同步是目的**。

Java 提供了两种锁机制来控制多个线程对共享资源的互斥访问，第一个是 JVM 实现的 *synchronized*，而另一个是 JDK 实现的 *ReentrantLock*。

#### 1. synchronized

*synchronized* 关键字，这是一种块结构（ Block Structured）的同步语法。 *synchronized* 关键字经过 Javac 编译之后，会在同步块的前后分别形成 **monitorenter** 和 **monitorexit** 这两个字节码指令。这两个字节码指令都需要一个 **reference 类型的参数**来指明要锁定和解锁的对象。

（1）同步**代码块**和同步**非静态方法**，可以实现同步一个**对象**：

```java
public void func() {
    synchronized (this) {
        // ...
    }
}
```

```java
public synchronized void func () {
    // ...
}
```

（2）同步**`class`类**和同步**静态方法**，可以实现同步一个**类**，即同步同一类的所有对象：

```java
public void func() {
    synchronized (SynchronizedExample.class) {
        // ...
    }
}
```

```java
public synchronized static void fun() {
    // ...
}
```

根据《Java虚拟机规范》的要求，在执行 monitorenter 指令时，首先要去尝试获取对象的锁。如果这个对象没被锁定，或者当前线程已经持有了那个对象的锁，就把锁的计数器的值增加一，而在执行 monitorexit 指令时会将锁计数器的值减一。一旦计数器的值为零，锁随即就被释放了。如果获取对象锁失败，那当前线程就应当被阻塞等待，直到请求锁定的对象被持有它的线程释放为止。

Java 虚拟机会在 monitorenter 对应的机器码指令之后临界区开始之前的地方插入一个**加载屏障**，这使得读线程的执行处理器能够将写线程对相应共享变量所做的更新从其他处理器同步到该处理器的高速缓存中。在 monitorexit 对应的机器码指令之后插入一个**存储屏障**，保障了写线程在释放锁之前在临界区中对共享变量所做的更新对读线程的执行处理器来说是可同步的。

#### 2. ReentrantLock

*ReentrantLock* 是可重入锁，它和 *synchronized* 一样，一个线程可以多次获取同一个锁。和 *synchronized* 不同的是，*ReentrantLock* 可以尝试获取锁：

```java
if (lock.tryLock(1, TimeUnit.SECONDS)) {
    try {
        ...
    } finally {
        lock.unlock();
    }
}
```

上述代码在尝试获取锁的时候，最多等待 1 秒。如果 1 秒后仍未获取到锁，`tryLock()` 返回 false，程序就可以做一些额外处理，而不是无限等待下去。

#### 3. 比较

* **锁的实现**：*synchronized* 是 JVM 实现的，而 *ReentrantLock* 是 JDK 实现的；
* **性能**：*synchronized* 与 *ReentrantLock* 大致相同；
* **等待可中断**：*ReentrantLock* 可中断，而 *synchronized* 不行；
* **公平锁**：*synchronized* 中的锁是非公平的，*ReentrantLock* 默认非公平，可设置为公平；
* **锁绑定多个条件**：一个 *ReentrantLock* 可以同时绑定多个 *Condition* 对象。

#### 4. 总结

推荐在 *synchronized* 与 *ReentrantLock* 都可满足需要时优先使用 *synchronized*，*synchronized* 是在 Java 语法层面的同步，足够清晰，也足够简单。

*ReentrantLock* 应该确保在 *finally* 块中释放锁，否则一旦受同步保护的代码块中抛出异常，则有可能永远不会释放持有的锁。而使用 *synchronized* 的话则可以由 Java 虛拟机来确保即使出现异常，锁也能被自动释放。

从长远来看，Java 虚拟机更容易针对 *synchronized* 来进行优化，因为 Java 虚拟机可以在线程和对象的元数据中记录 *synchronized* 中锁的相关信息，而使用 J.U.C 中的 *Lock* 的话，Java 虚拟机是很难得知具体哪些锁对象是由特定线程锁持有的。

### 5.3 非阻塞同步

互斥同步最主要的问题就是**线程阻塞**和**唤醒**所带来的性能问题，这种同步也称为**阻塞同步**。

互斥同步属于一种**悲观**的并发策略，总是认为只要不去做正确的同步措施，那就肯定会出现问题。无论共享数据是否真的会出现竞争，它都要进行加锁（这里讨论的是概念模型，实际上虚拟机会优化掉很大一部分不必要的加锁）、用户态核心态转换、维护锁计数器和检查是否有被阻塞的线程需要唤醒等操作。

随着硬件指令集的发展，我们可以使用基于**冲突检测**的**乐观**并发策略：先进行操作，如果没有其它线程争用共享数据，那操作就成功了，否则采取补偿措施（不断地重试，直到成功为止）。这种乐观的并发策略的许多实现都不需要将线程阻塞，因此这种同步操作称为**非阻塞同步**。

#### 1. CAS

乐观锁需要操作和冲突检测这两个步骤具备**原子性**，这里就不能再使用互斥同步来保证了，只能靠**硬件**来完成。硬件支持的原子性操作最典型的是：比较并交换（Compare-and-Swap，CAS）。CAS 指令需要有 3 个操作数，分别是内存地址 V、旧的预期值 A 和新值 B。当执行操作时，只有当 V 的值等于 A，才将 V 的值更新为 B。

#### 2. AtomicInteger

J.U.C 包里面的整数原子类 *AtomicInteger* 的方法调用了 *Unsafe* 类的 CAS 操作。

以下代码使用了 *AtomicInteger* 执行了自增的操作。

```java
private AtomicInteger cnt = new AtomicInteger();

public void add() {
    cnt.incrementAndGet();
}
```

以下代码是 `incrementAndGet()` 的源码，它调用了 *Unsafe* 的 `getAndAddInt() `。

```java
public final int incrementAndGet() {
    return unsafe.getAndAddInt(this, valueOffset, 1) + 1;
}
```

以下代码是 `getAndAddInt()` 源码，var1 指示对象内存地址，var2 指示该字段相对对象内存地址的偏移，var4 指示操作需要加的数值，这里为 1。通过 `getIntVolatile(var1, var2)` 得到旧的预期值，通过调用 `compareAndSwapInt()` 来进行 CAS 比较，如果该字段内存地址中的值等于 var5，那么就更新内存地址为 var1 + var2 的变量为 var5 + var4。

可以看到 `getAndAddInt()` 在一个循环中进行，发生冲突的做法是不断的进行重试。

```java
public final int getAndAddInt(Object var1, long var2, int var4) {
    int var5;
    do {
        var5 = this.getIntVolatile(var1, var2);
    } while(!this.compareAndSwapInt(var1, var2, var5, var5 + var4));

    return var5;
}
```

#### 3. ABA问题

[ABA 问题](https://www.zhihu.com/question/23281499)就是如果一个变量初次读取的时候是 A 值，它的值被改成了 B，后来又被改回为 A，那 CAS 操作就会误认为它从来没有被改变过。

J.U.C 包提供了一个带有标记的原子引用类 *AtomicStampedReference* 来解决这个问题，它可以通过控制变量值的版本来保证 CAS 的正确性。大部分情况下 ABA 问题不会影响程序并发的正确性，如果需要解决 ABA 问题，**改用传统的互斥同步可能会比原子类更高效**。

### 5.4 无同步方案

要保证线程安全，并不是一定就要进行同步。如果一个方法本来就不涉及共享数据，那它自然就无须任何同步措施去保证正确性。

#### 1. 栈封闭

多个线程访问同一个方法的局部变量时，不会出现线程安全问题，因为局部变量存储在**虚拟机栈**中，属于线程私有的。

#### 2. 线程本地存储

如果一段代码中所需要的数据必须与其他代码共享，那就看看这些共享数据的代码是否能**保证在同一个线程中执行**。如果能保证，我们就可以把共享数据的可见范围限制在同一个线程之内，这样，无须同步也能保证线程之间不出现数据争用的问题。

符合这种特点的应用并不少见，大部分使用消费队列的架构模式（如 “生产者-消费者” 模式）都会将产品的消费过程尽量在一个线程中消费完。其中最重要的一个应用实例就是经典 Web 交互模型中的 “一个请求对应一个服务器线程”（Thread-per-Request）的处理方式，这种处理方式的广泛应用使得很多 Web 服务端应用都可以使用线程本地存储来解决线程安全问题。

可以使用 java.lang.ThreadLocal 类来实现线程本地存储功能。对于以下代码，thread1 中的 threadLocal1 和 threadLocal2 为 1，而 thread2 中的 threadLocal1 和 threadLocal2 为 2。

```java
public class ThreadLocalExample1 {
    public static void main(String[] args) {
        ThreadLocal threadLocal1 = new ThreadLocal();
        ThreadLocal threadLocal2 = new ThreadLocal();
        Thread thread1 = new Thread(() -> {
            threadLocal1.set(1);
            threadLocal2.set(1);
        });
        Thread thread2 = new Thread(() -> {
            threadLocal1.set(2);
            threadLocal2.set(2);
        });
        thread1.start();
        thread2.start();
    }
}
```

每个 *Thread* 都有一个 ThreadLocal.ThreadLocalMap 对象。

```java
/* ThreadLocal values pertaining to this thread. This map is maintained
 * by the ThreadLocal class. */
ThreadLocal.ThreadLocalMap threadLocals = null;
```

当调用一个 *ThreadLocal* 的 `set(T value)` 方法时，先得到当前线程的 *ThreadLocalMap* 对象，然后将 *ThreadLocal -> value* 键值对插入到该 *Map* 中。

*ThreadLocal* 从理论上讲并不是用来解决多线程并发问题的，因为根本不存在多线程竞争。

在一些场景 (尤其是使用线程池) 下，由于 ThreadLocal.ThreadLocalMap 的底层数据结构导致 *ThreadLocal* 有内存泄漏的情况，应该尽可能在每次使用 *ThreadLocal* 后手动调用 `remove()`，以避免出现 *ThreadLocal* 经典的内存泄漏甚至是造成自身业务混乱的风险。

#### 3. 可重入代码

这种代码也叫做纯代码（Pure Code），可以在代码执行的任何时刻中断它，转而去执行另外一段代码（包括递归调用它本身），而在控制权返回后，原来的程序不会出现任何错误。

可重入代码有一些共同的特征，例如**不依赖存储在堆上的数据和公用的系统资源**、**用到的状态量都由参数中传入**、**不调用非可重入的方法**等。

## 六、锁优化

### 6.1 自旋锁与自适应自旋

互斥同步对性能最大的影响是**阻塞的实现**，挂起线程和恢复线程的操作都需要转入**内核态**中完成，这些操作给 Java 虚拟机的并发性能带来了很大的压力。在许多应用中，共享数据的锁定状态只会持续很短的一段时间。自旋锁的思想是让一个线程在请求一个共享数据的锁时执行**忙循环**（自旋）一段时间，如果在这段时间内能获得锁，就可以避免进入阻塞状态。

自旋锁虽然能避免进入阻塞状态从而减少开销，但是它需要进行忙循环操作占用 CPU 时间，它只适用于共享数据的锁定状态很短的场景。因此自旋等待的时间必须有一定的限度，如果自旋超过了限定的次数仍然没有成功获得锁，就应当使用传统的方式去挂起线程，自旋次数的默认值是 10 次。

在 JDK 1.6 中引入了自适应的自旋锁。自适应意味着自旋的次数不再固定了，而是由前一次在同一个锁上的自旋次数及锁的拥有者的状态来决定。如果在同一个锁对象上，自旋等待刚刚成功获得过锁，并且持有锁的线程正在运行中，那么虚拟机就会认为这次自旋也很有可能再次成功，进而允许自旋等待持续相对更长的时间，比如持续 100 次忙循环。另一方面，如果对于某个锁，自旋很少成功获得过锁，那在以后要获取这个锁时将有可能直接省略掉自旋过程，以避免浪费处理器资源。

java 中有以下几种实现 [[11]](https://zhuanlan.zhihu.com/p/40729293)：

* **TicketLock**：主要解决的是公平性的问题。

* **CLHLock**：基于链表的可扩展、高性能、公平的自旋锁，申请线程只在本地变量上自旋，它不断轮询前驱的状态，如果发现前驱释放了锁就结束自旋，获得锁。

* **MCSLock**：基本与 CLHLock 相同，不同在于其查看的是当前节点的锁状态，对本地变量的节点进行轮询。

### 6.2 锁消除

锁消除是指对于被检测出不可能存在竞争的共享数据的锁进行消除。

锁消除主要是通过**逃逸分析**来支持，如果堆上的共享数据不可能逃逸出去被其它线程访问到，那么就可以把它们当成私有数据对待，也就可以将它们的锁进行消除。

对于一些看起来没有加锁的代码，其实隐式的加了很多锁。例如下面的字符串拼接代码就隐式加了锁：

```java
public static String concatString(String s1, String s2, String s3) {
    return s1 + s2 + s3;
}
```

*String* 是一个不可变的类，编译器会对 *String* 的拼接自动优化。在 JDK 1.5 之前，会转化为 *StringBuffer* 对象的连续 `append()` 操作：

```java
public static String concatString(String s1, String s2, String s3) {
    StringBuffer sb = new StringBuffer();
    sb.append(s1);
    sb.append(s2);
    sb.append(s3);
    return sb.toString();
}
```

每个 `append()` 方法中都有一个同步块。虚拟机观察变量 *sb*，很快就会发现它的动态作用域被限制在 `concatString()` 方法内部。也就是说，*sb* 的所有引用永远不会逃逸到 `concatString()` 方法之外，其他线程无法访问到它，因此可以进行消除。

### 6.3 锁粗化

如果一系列连续操作都对同一个对象**反复加锁和解锁**，频繁的加锁操作就会导致性能损耗。

上面的 `append()` 方法就属于这类情况。如果虚拟机探测到由这样的一串零碎的操作都对同一个对象加锁，将会把加锁的范围扩展（粗化）到整个操作序列的外部。对于上一节的示例代码就是扩展到第一个 `append()` 操作之前直至最后一个 `append()` 操作之后，这样只需要加锁一次就可以了。

### 6.4 偏向锁

JDK 1.6 引入了偏向锁和轻量级锁，从而让锁拥有了四个状态：无锁状态（unlocked）、偏向锁状态（biasble）、轻量级锁状态（lightweight locked）和重量级锁状态（inflated）。

以下是 HotSpot 虚拟机对象头的内存布局，这些数据被称为 Mark Word。其中 tag bits 对应了五个状态，这些状态在右侧的 state 表格中给出。

<img src="https://qttblog.oss-cn-hangzhou.aliyuncs.com/after3.26/lock1.png" alt="img"  />

下图左侧是一个线程的**虚拟机栈**，其中有一部分称为 Lock Record 的区域，这是在轻量级锁运行过程创建的，用于存放锁对象的 Mark Word。而右侧就是一个锁对象，包含了 Mark Word 和其它信息。

<img src="https://qttblog.oss-cn-hangzhou.aliyuncs.com/after3.26/lock2.png" alt="img"  />

偏向锁的思想是偏向于让第一个获取锁对象的线程，这个线程在之后获取该锁就**不再需要进行同步操作，甚至连 CAS 操作也不再需要。**

当锁对象第一次被线程获得的时候，虚拟机将会把对象头中的标志位设置为 01，把偏向模式设置为 1，表示进入偏向模式。同时使用 CAS 操作将线程 ID 记录到 Mark Word 中，如果 CAS 操作成功，这个线程以后每次进入这个锁相关的同步块就不需要再进行任何同步操作。

当有另外一个线程去尝试获取这个锁对象时，偏向状态就宣告结束，此时撤销偏向（Revoke Bias）后恢复到未锁定状态或者轻量级锁状态。

<img src="https://qttblog.oss-cn-hangzhou.aliyuncs.com/after3.26/lock4.jpg" alt="img"  />

### 6.5 轻量级锁

轻量级锁是相对于传统的重量级锁而言，它使用 **CAS 操作**来避免重量级锁使用互斥量的开销。对于绝大部分的锁，在整个同步周期内都是不存在竞争的，因此也就不需要都使用互斥量进行同步，可以先采用 CAS 操作进行同步，如果 CAS 失败了再**改用互斥量进行同步**。

当尝试获取一个锁对象时，如果锁对象标记为 01，说明锁对象的锁未锁定（unlocked）状态。此时虚拟机在当前线程的虚拟机栈中创建 Lock Record，然后使用 CAS 操作将对象的 Mark Word 更新为 Lock Record 指针。如果 CAS 操作成功了，那么线程就获取了该对象上的锁，并且对象的 Mark Word 的锁标记变为 00，表示该对象处于轻量级锁状态。

<img src="https://qttblog.oss-cn-hangzhou.aliyuncs.com/after3.26/lock3.png" alt="img"  />

如果 CAS 操作失败了，虚拟机首先会检查对象的 Mark Word 是否指向当前线程的虚拟机栈，如果是的话说明当前线程已经拥有了这个锁对象，那就可以直接进入同步块继续执行，否则说明这个锁对象已经被其他线程线程抢占了。如果有两条以上的线程争用同一个锁，那轻量级锁就不再有效，要膨胀为重量级锁。



## 七、J.U.C

### 7.1 AQS

*AbstractQueuedSynchronizer*（AQS）定义了一套多线程访问共享资源的同步器框架，许多同步类实现都依赖于它，如常用的 *ReentrantLock*/*Semaphore*/*CountDownLatch*...

AQS 维护了一个 *volatile int state*（代表共享资源）和一个 FIFO 线程等待队列（多线程争用资源被阻塞时会进入此队列）。

<img src="https://qttblog.oss-cn-hangzhou.aliyuncs.com/june/concurconcurency6.png" style="zoom:150%;" />

### 7.2 CountDownLatch

*CountDownLatch* 类位于 java.util.concurrent 包下，利用它可以实现类似计数器的功能。比如有一个任务 A，它要等待其他 4 个任务执行完毕之后才能执行，此时就可以利用 *CountDownLatch* 来实现这种功能了。*CountDownLatch* 类只提供了一个构造器：

```java
public CountDownLatch(int count) {  };  //参数count为计数值
```

然后下面这 3 个方法是 *CountDownLatch* 类中最重要的方法：

```java
//调用await()方法的线程会被挂起，它会等待直到count值为0才继续执行
public void await() throws InterruptedException { };  
//和await()类似，只不过等待一定的时间后count值还没变为0的话就会继续执行
public boolean await(long timeout, TimeUnit unit) throws InterruptedException { };  
//将count值减1
public void countDown() { };  
```

### 7.3 CyclicBarrier

回环栅栏，通过它可以实现让一组线程等待至某个状态之后再全部同时执行。叫做回环是因为当所有等待线程都被释放以后，*CyclicBarrier* 可以被重用。我们暂且把这个状态就叫做 barrier，当调用 `await()` 方法之后，线程就处于 barrier 了。*CyclicBarrier* 提供 2 个构造器：

```java
// 参数parties指让多少个线程或者任务等待至barrier状态；
public CyclicBarrier(int parties) {...}
// 参数barrierAction为当这些线程都达到barrier状态时会执行的内容。 
public CyclicBarrier(int parties, Runnable barrierAction) {...}
```

*CyclicBarrier* 中最重要的方法就是 `await()` 方法，它有 2 个重载版本：

```java
// 挂起当前线程，直至所有线程都到达barrier状态再同时执行后续任务；
public int await() throws InterruptedException, BrokenBarrierException { };
// 让这些线程等待至一定的时间，如果还有线程没有到达barrier状态就直接让到达barrier的线程执行后续任务。
public int await(long timeout, TimeUnit unit) throws InterruptedException,BrokenBarrierException,TimeoutException { };
```

### 7.4 Semaphore

信号量 *Semaphore* 可以控同时访问的线程个数，通过 `acquire()` 获取一个许可，如果没有就等待，而 `release()` 释放一个许可。它提供了 2 个构造器：

```java
//参数permits表示许可数目，即同时可以允许多少线程进行访问
public Semaphore(int permits) {          
    sync = new NonfairSync(permits);
}
//参数fair表示是否是公平的，即等待时间越久的越先获取许可
public Semaphore(int permits, boolean fair) {    
    sync = (fair)? new FairSync(permits) : new NonfairSync(permits);
}
```

下面说一下 *Semaphore* 类中比较重要的几个方法，首先是 `acquire()`、`release()` 方法：

```java
public void acquire() throws InterruptedException {  }     //获取一个许可
public void acquire(int permits) throws InterruptedException { }    //获取permits个许可
public void release() { }          //释放一个许可
public void release(int permits) { }    //释放permits个许可
```

这 4 个方法都会被阻塞，如果想立即得到执行结果，可以使用下面几个方法：

```java
//尝试获取一个许可，若获取成功，则立即返回 true，若获取失败，则立即返回 false
public boolean tryAcquire() { };    
//尝试获取一个许可，若在指定的时间内获取成功，则立即返回 true，否则则立即返回 false
public boolean tryAcquire(long timeout, TimeUnit unit) throws InterruptedException { };  
//尝试获取 permits 个许可，若获取成功，则立即返回 true，若获取失败，则立即返回 false
public boolean tryAcquire(int permits) { }; 
//尝试获取 permits 个许可，若在指定的时间内获取成功，则立即返回 true，否则则立即返回 false
public boolean tryAcquire(int permits, long timeout, TimeUnit unit) throws InterruptedException { }; 
```

### 7.5 总结

* *CountDownLatch* 用于某个线程等待若干个其他线程执行完任务之后执行，不能重用；
* *CyclicBarrier* 用于一组线程互相等待至某个状态，然后再同时执行，可以重用；
* *Semaphore* 其实和锁有点类似，一般用于控制对某组资源的访问权限。

[其他一些组件](https://cyc2018.github.io/CS-Notes/#/notes/Java%20并发?id=八、juc-其它组件)

## 八、多线程开发良好的实践

- 给线程起个有意义的名字，这样可以方便找 Bug。
- 缩小同步范围，从而减少锁争用。例如对于 *synchronized*，应该尽量使用同步块而不是同步方法。
- 多用同步工具少用 `wait()` 和 `notify()`。首先，*CountDownLatch*，*CyclicBarrier*，*Semaphore* 和 *Exchanger* 这些同步类简化了编码操作，而用 `wait()` 和 `notify()` 很难实现复杂控制流；其次，这些同步类是由最好的企业编写和维护，在后续的 JDK 中还会不断优化和完善。
- 使用 *BlockingQueue* 实现生产者消费者问题。
- 多用并发集合少用同步集合，例如应该使用 *ConcurrentHashMap* 而不是 *Hashtable*。
- 使用本地变量和不可变类来保证线程安全。
- 使用线程池而不是直接创建线程，这是因为创建线程代价很高，线程池可以有效地利用有限的线程来启动任务。

## 参考

1. 《深入理解 Java 虚拟机》
2. [CyC CS-Notes](https://cyc2018.github.io/CS-Notes/#/notes/Java%20并发?id=一、使用线程)
3. [廖雪峰java教程](https://www.liaoxuefeng.com/wiki/1252599548343744/1255943750561472)
4. [CountDownLatch、CyclicBarrier 和 Semaphore](https://www.cnblogs.com/dolphin0520/p/3920397.html)
5. [Java线程池(ThreadPoolExecutor)](https://blog.csdn.net/fuyuwei2015/article/details/72758179)
6. [Java面试经典题：线程池专题](https://juejin.im/post/6844903634975768590#heading-33)
7. 《Java并发编程实战（Java Concurrency In Practice）》
8. [为什么要指令重排](http://mageek.cn/archives/99/)
9. [Why Memory Barriers？中文翻译（上）](http://www.wowotech.net/kernel_synchronization/Why-Memory-Barriers.html)
10. [Why Memory Barriers？中文翻译（下）](http://www.wowotech.net/kernel_synchronization/why-memory-barrier-2.html)
11. [自旋锁的几种形式](https://zhuanlan.zhihu.com/p/40729293)
12. [线程状态的转换](https://www.uml-diagrams.org/java-thread-uml-state-machine-diagram-example.html)

