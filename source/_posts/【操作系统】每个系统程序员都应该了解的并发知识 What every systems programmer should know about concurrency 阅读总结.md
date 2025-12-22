---
title: 【操作系统】每个系统程序员都应该了解的并发知识 What every systems programmer should know about concurrency 阅读总结
categories: 操作系统
---

# 每个系统程序员都应该了解的并发知识 What every systems programmer should know about concurrency 阅读总结

23307130289 丘俊

## 目录

<!-- toc -->

## 零、前言

在《操作系统》课程的学习中，接触到了[《What every systems programmer should know about concurrency》](https://assets.bitbashing.io/papers/concurrency-primer.pdf)这篇论文，讲述了并发编程的许多知识，非常值得做一些总结。

## 一、当代计算机有哪些必要的技术可能干扰并发程序设计的正确性？

### 1. 编译优化

**这个技术为什么必要？**

编译器可能会优化我们所写的代码，从而让这些代码可以在编译器所针对的硬件上更快地运行。例如，只要保证最终的汇编代码的指令排列对当前线程产生的效果是相同的，那么汇编代码中的读取和写入指令可以被移动，以避免流水线停顿，或提高局部性，以提升性能。

**这个技术为什么可能干扰并发程序的正确性？**

这些指令被重新排序后，虽然对当前线程而言，最终结果是相同的，但如果与其他线程共享某些变量，重新排序的指令可能会破坏多线程间某种特定的操作顺序（例如，`data`还未写入，`done`先被赋值），导致其他线程看到了不一致的状态，从而导致其他线程上产生预期之外的情况。

### 2. 多层级的缓存

**这个技术为什么必要？**

处理器速度在过去几十年里呈指数级增长，但内存的速度却没有跟上，因此，硬件设计师通过在CPU芯片上直接放置越来越多的分层缓存来进行补偿。此外，每个CPU核通常都有一个存储缓冲区，用于在执行后续指令的同时，处理待处理的写操作。

**这个技术为什么可能干扰并发程序的正确性？**

当一个CPU核心写入数据时，数据通常先停留在该核心的缓存中，而没有立即写入主存。这导致其他核心无法立即看到最新的数据，或者看到数据的顺序与写入数据的顺序不一致。这也有可能导致其他核心上的程序出现意料之外的行为。

### 3. NUMA处理器结构

**这个技术为什么必要？**

在传统SMP架构中，所有核心通过一条总线访问内存，正如本文刚刚所描述的，内存的速度是现代计算机速度的一个瓶颈。因此，NUMA这种架构将内存物理分割成多块，并为每块内存分配单一的总线，每条总线连接着一个CPU结点（Node），每个Node包含一个或多个CPU核心。 通过将内存分割成多块，内存吞吐量得到了成倍的提升。

**这个技术为什么可能干扰并发程序的正确性？**

在NUMA架构下，如果一个CPU需要访问不属于自己所在Node所连接的内存（远程访存），则需要通过互联总线，向其他Node发出请求进行访问，这个过程的耗时通常大约为访问自己所在Node所连接的内存（本地访存）的2倍。同时，NUMA架构可能会根据CPU占用情况，将进程调度到其他Node，导致大量出现这种远程访存的情况。导致程序执行时间出现不可预测的波动。对于延迟敏感的系统程序（如高频交易、实时控制），这种时间上的不确定性极易给并发过程带来困难。

## 二、如何理解there is no consistent concept of “now” in a multithreaded program

> All of these complications mean that there is no consistent concept of “now” in a multithreaded program, especially on a multi-core cpu. Creating some sense of order between threads is a team effort of the hardware, the compiler, the programming language, and your application. Let’s explore what we can do, and what tools we will need.

本文前文介绍了3个当代计算机必要的技术，这些技术解决了很多性能问题，但也为并发程序带来了不便。这些广泛应用的技术（如指令重排），使得多线程程序运行过程中，在不同线程上，并没有一致的“现在”的概念，也就是说，在没有做加锁等特殊处理时，因为指令重排等技术，我们实际上无法保证“现在”执行到了哪行代码，“现在”另一个线程是否完成了对此变量的写入等问题。而我们想要编写正确的程序，又显然必须关注程序“现在”的状态，因此，我们需要做出努力，以在多线程程序中创建某种“顺序感”，通过这种顺序感，我们能够保证“现在”的某些局部状态是我们所需要的，从而正确地编写并发程序。

## 三、如何理解正确并发程序设计所要求的”保序“（Enforcing law and order）和操作的原子性（Atomicity）？

### 1. 保序

考虑以下程序：

```c
int v = 0;
bool v_ready(false);

void threadA() {
    v = 42;
    v_ready = true;
}

void threadB() {
    while (!v_ready) { /* wait */ }
    const int my_v = v;
    // Do something with my_v...
} 
```

如果`threadA`中发生指令重排，`v_ready = true;`在`v = 42;`之前执行，就会导致`threadB`中获取错误的结果。这就是正确并发程序设计所要求的“保序”。

### 2. 操作的原子性

假设我们有一个128位int类型变量`data`，假设我们的程序运行在64位CPU上。由于寄存器是64位的，程序对这个变量进行写操作时，转换成汇编指令后，就必然要通过两条指令才能完整写入此变量。在两条指令执行的中间，就会产生一个“无效数据”的中间状态。如果此时另一个核心访问这块内存，就会访问到这个无效数据。或者此时调度器切换线程，就会导致这个无效数据在内存中停留大量的时间，也会大大增大这块无效数据被访问的概率。

操作的原子性就是确保某个操作要么发生，要么不发生，不应该出现上面这种“发生了一半”的情况。

## 四、通过程序示例考察现代语言C/C++中的原子态运算能力，例如C11中的`stdatomic.h`

下面仅描述C语言中的原子库相关内容，实际上C++标准库中均有对应的实现。

1. 提供了与基础整型对应的原子类型（`atomic_bool`, `atomic_int`等），这些类型通过`load-link`和`store-conditional`实现无锁的原子操作。
2. 使用`_Atomic`关键字实现任意类型原子化。如果类型大小小于等于机器字长，同样可以通过`load-link`和`store-conditional`实现无锁的原子操作。否则，编译器会自动插入锁来实现原子操作。

## 五、何为撕裂性的读写（torn reads and writes），它对并发造成了什么危害？

前文在**操作的原子性**一节中所举的例子，产生的无效数据，就称为撕裂性的读写。这样的无效数据会影响并发程序的正确性。因此我们不得不采取更多的措施（例如加锁等）以避免这种情况的发生，这些措施会造成性能下降，但总归能够保证并发程序的正确性。

## 六、再次考证`read-modify-write` (RMW) 操作

RMW操作可以分为以下四类：

1. exchange

   行为：用新值替换旧值，并返回旧值。

   使用场景：UI进度条线程需要显示过去一秒内完成了多少任务。那么这个线程首先需要获取过去一秒内完成的任务数量。然后需要清空这个任务数量，以实现一秒后再次读取这个变量时，又能获取到过去一秒内完成的任务数量。必须把这两个操作打包成一个原子操作，才能确保UI进度条的正确性。

2. test-and-set

   行为：读取`bool`类型变量，将其设为`true`，并返回旧值。

   使用场景：构建`spinlock`。将`spinlock.acquired`设为`true`，并返回旧值，如果旧值是`false`，则我们成功获取了锁，否则，获取锁失败，在外层嵌套一个`while`循环直至获取锁成功为止。必须把这两个操作打包成一个原子操作，才能保证正确性。

3. fetch and ...

   行为：对值执行对应的运算，并返回旧值

   使用场景：任何可能同时被多个线程修改的变量都需要原子操作保证。因为运算设计三个过程：读取、运算、写入。如果一个线程读取后，另一个线程执行了写入，那么第一个线程所读取的值就产生了不一致。因此必须将三个过程打包成为一个原子操作。

4. compare and swap（cas）

   行为：

   ```c
   if (*this == expected) {
       *this = desired;
       return true;
   } else {
       expected = *this; // 关键：失败时会将 expected 更新为当前内存中的实际值
       return false;
   }
   ```

   使用场景：

   1. 很多复杂运算（如原子乘法、链表节点的插入）没有对应的`fetch_and_...`直接指令。我们需要先读取变量（`old_value`），在本地计算出新值（`new_value`），然后尝试更新。CAS保证了“只有当内存中的值仍等于`old_value`时，才将其更新为`new_value`”。如果失败（说明计算期间有其他线程修改了该变量），CAS会自动更新`expected`为当前最新值，我们只需在`while`循环中利用这个新值重新计算并重试即可。
   2. 在任务调度系统中，我们可能希望“仅当任务处于`Running`状态时，才将其标记为`Cancelled`”。如果任务已经结束或处于`Idle`状态，则不应修改。CAS能够原子地完成这一逻辑。

## 七、无锁并发一定在效率上比加锁的好吗？如果不完全是，那为什么还要推崇无锁并发处理？

不完全是。

**无锁算法并不是某种比阻塞算法（也就是加锁）更好或更快的方式，它们只是为不同的任务设计的不同工具**。

对比加锁，无锁有以下优势：

1. 绝对不会发生死锁和活锁。
2. 无锁是非阻塞的，因此这种方法能够确保程序始终在向前进行，从而能够在实时场景下具备更大的优势。例如，在播放音频时，我们不希望阻塞的出现，否则在用户设备性能不足时容易出现卡顿。又如，在传感器程序中，如果出现了阻塞，就容易发生阻塞导致的丢失某些时刻下的传感器输入。

有些情况下，无论是阻塞还是无锁的方法都可以工作。每当性能成为一个关注点时，都要进行性能分析！性能取决于许多因素，**从参与的线程数量到您的** **CPU** **的具体规格都有可能影响性能**。并且始终要考虑在**复杂性**和**性能**之间进行权衡——并发是一门危险的艺术。

## 八、在弱序硬件（weakly-ordered hardware，例如ARM）上的串行一致性（Sequential consistency）保证方法

使用`dmb`指令（内存屏障）。

data memory barriar 操作的作用是强制 CPU 在执行 `dmb` 操作之前的所有指令都完成，然后再执行 `dmb` 操作之后的指令。

## 九、在下面的实现方案里，内存屏障指令（dmb）的作用是什么？

<img src="assets/LoadAndStoreImplementationOnAWeakly-orderedHardware.png" alt="LoadAndStoreImplementationOnAWeakly-orderedHardware" style="zoom: 50%;" />

`getFOO:`

`ldr r0, [r3, #0]`被夹在两个`dmb`中间，确保执行这条指令前，前面的指令都已完成（即读取`foo`的地址），同时确保后面的指令必须在这条指令执行之后再执行，确保`bx lr`跳转后对`foo`进行操作的指令都正确读取到了`foo`的值。

`setFoo`中的`dmb`也是一样的作用。

## 十、如何利用`load-link`和`store-conditional` （即所谓的LL/SC指令）实现原子性的读取-修改-写回（ read-modify-write ）操作？

`load-link`**从一个地址读取一个值**（就像任何其他`load`操作一样），但还指示处理器监视跟踪该地址。`store-conditional`仅在自从对应的`load-link`之后未对**该地址**进行其他`store`操作时才写入给定的值。

于是，利用这一点，我们可以编写一个循环，当且仅当`store-conditional`成功时，循环结束，就保证实现了原子性的read-modify-write操作。

以下是一个例子。

```c
void incFoo() { ++foo; }
```

编译后：

```assembly
incFoo:
	ldr r3, <&foo>
	dmb
loop:
	ldrex r2, [r3] // LL foo
	adds r2, r2, #1 // Increment
	strex r1, r2, [r3] // SC
	cmp r1, #0 // Check the SC result.
	bne loop // Loop if the SC failed.
	dmb
	bx lr
```

## 十一、LL/SC指令会形成假阳性而降低计算的性能吗？什么原因导致了假阳性的发生？

会。

要跟踪每个字节的`load-link`地址将消耗太多的 CPU 硬件资源。为了降低这种成本，许多处理器以某种较粗粒度（如缓存行）监视它们。这意味着如果在受监视块中的任何地址之前有写操作，`sc` 可能会失败，而不仅仅是之前加载链接的那个地址。

## 十二、作为程序员，是否能够有对内存模型的精准控制？请举例说明

能。C/C++提供了以下内存序供程序员执行精准控制：

1. 顺序一致 (memory_order_seq_cst)

2. 获得 (memory_order_acquire)

3. 释放 (memory_order_release)

4. 松散 (memory_order_relaxed)

5. 获得-释放 (memory_order_acq_rel)

6. 消耗 (memory_order_consume)

下面举例。

1. **memory_order_seq_cst**

   强序控制：保证程序的执行顺序在所有线程看来都是完全一致。在ARM等弱内存序架构上，通过`dmb`指令完成。

2. **memory_order_acquire和memory_order_release**

   同步控制：

   - **Release (生产者)**：store(memory_order_release)。保证在此之前的**所有**写操作（包括非原子的普通变量），不会被重排到这个原子写操作之后。
   - **Acquire (消费者)**：load(memory_order_acquire)。保证在此之后的**所有**读操作，不会被重排到这个原子读操作之前。

3. **memory_order_relaxed**

   只保证操作本身的原子性，完全不保证顺序。

4. **memory_order_consume**

   告诉编译器“只有依赖这个加载值的后续操作才不能重排”。但是由于太难正确实现，目前的编译器通常将其自动升级为`acquire`。

## 十三、试解释为什么以下的`Compare-and-swap`操作要使用不同的内存模型？

```c
while (!foo.compare_exchange_weak(
  expected, expected * by, 
  memory_order_seq_cst,  // On success
  memory_order_relaxed)) { // On failure 
 /* empty loop */ 
}
```

成功时，意味着共享变量`foo`成功写入，那么就需要使用`memory_order_seq_cst`（顺序一致性），确保这次写入操作在所有线程看来都有一个确定的、全局一致的顺序，防止编译器或CPU对指令进行重排，从而保证并发逻辑的正确性。这是最严格的内存顺序模型，虽然开销大，但保证了操作的原子性。

失败时，内存中的值没有被影响，仅需直接重试即可。此时，因为没有发生内存写入，所以没有必要使用具有相当大开销的`memory_order_seq_cst`来建立线程间的同步顺序。使用`memory_order_relaxed` 允许 CPU 以最低的开销读取数据，这在紧凑的循环（tight loop）中非常关键，因为它避免了每次循环失败都触发昂贵的缓存同步操作，从而提高了程序的性能。

## 十四、在并发处理上，缓存起到了什么干扰作用？以下面程序段的读写锁为例，加以说明：读写锁会改进程序的效率吗？为什么会？如果不会，又是什么原因导致的？

```c
struct RWLock {
  int readers;
  bool hasWriter; // Zero or one writers
};
```

**并发处理上，缓存起到的干扰作用**：

如果一个核心写入了某个值，而另一个核心需要读取该值，则包含该值的整个cache line必须从第一个核心的缓存传输到第二个核心的缓存中。当多个核心频繁读写同一个cache line上的数据时，会导致该cache line在不同核心的缓存之间来回传输（ping-pong），这种硬件层面的数据传输开销非常大，降低程序速度。

**读写锁会改进程序的效率吗？**

在只读场景下，读写锁可以改进程序效率，因为它允许多个核心并发地读取内存。

在有频繁写入的场景下，读写锁会因为之前所说的缓存干扰作用，导致性能下降严重。举例说明，有十个并发程序读取了变量`a`，并且都准备对变量`a`进行修改，当发生一次成功写入后，变量`a`的值需要通过在十个CPU核心间传输cache line的方式进行重新同步，这就产生了巨大的性能开销。而自旋锁在这种场景下，由于不会出现多个`reader`，就不需要在如此多的核心之间重新传输cache line，缓存的这种干扰作用就没那么明显。

## 十五、考证`volatile`修饰符的语义，避免在并发问题上的误用

`volatile`这个修饰符只提供以下两个保证：

1. 确保编译器不会优化掉看似多余的读写操作，例如，以下代码中：

   ```c
   void write(int* t) {
       *t = 2;
       *t = 42;
   }
   ```

   编译器通常会优化掉看似不必要的操作，例如，以上代码将被优化成：

   ```c
   void write(int* t) {
       *t = 42;
   }
   ```

   而`volatile`修饰符将保证编译器不会执行这样的优化。

2. 编译器不会重新排序两个`volatile`变量读写操作操作的相对顺序。

**之所以说`volatile`不是并发的答案，是因为：**

`volatile`不会生成任何内存屏障（dmb），`volatile`提供了一些编译器层面不会重新排序等的保障，但CPU硬件本身仍然可以为了性能自由地重排指令执行顺序。同时，`volatile`只保证两个`volatile`变量的读写操作的相对顺序，在这些代码周围的非`volatile`变量的相关操作仍然可以被重新排序。

## 十六、何为"Atomic Fusion"，我们该如何对待它？

原子融合，指编译器在优化代码时，将多个独立的原子操作融合为一次操作的现象。

举例来说：

```c
while (tmp = foo.load(memory_order_relaxed)) {
    doSomething(tmp);
}
```

由于使用了`memory_order_relaxed`，编译器认为它有权为了性能重排指令或者做循环展开，于是，上面的代码可能被优化为：

```c
while (tmp = foo.load(memory_order_relaxed)) {
    doSomething(tmp);
    doSomething(tmp);
    doSomething(tmp);
    doSomething(tmp);
}
```

在这样的优化下，如果我们的实现正确性的前提是每次`doSomething`之前必须确保`tmp`被重新加载的话，编译器这样的优化就破坏了我们的正确性。

因此，如果我们不能接受这种优化，则我们需要显式地阻止编译器进行这种优化。可以使用`volatile`, `asm volatile("" ::: "memory")`或`READ_ONCE()`, `WRITE_ONCE()`来阻止这种优化。