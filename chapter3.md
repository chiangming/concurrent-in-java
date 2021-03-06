# Java memory model

## 原子性、可见性、有序性

> **原子性**：即一个操作或者多个操作 要么全部执行并且执行的过程不会被任何因素打断，要么就都不执行。

在单线程环境下我们可以认为整个步骤都是原子性操作，但是在多线程环境下则不同，Java只保证了基本数据类型的变量和赋值操作才是原子性的（注：在32位的JDK环境下，对64位数据的读取不是原子性操作\*，如long、double）。要想在多线程环境下保证原子性，则可以通过锁、synchronized来确保。

> **可见性：**指当多个线程访问同一个变量时，一个线程修改了这个变量的值，其他线程能够立即看得到修改的值。

当一个变量被volatile修饰后，表示着线程本地内存无效，当一个线程修改共享变量后他会立即被更新到主内存中，当其他线程读取共享变量时，它会直接从主内存中读取。当然，synchronize和锁都可以保证可见性。

> **有序性**：即程序执行的顺序按照代码的先后顺序执行。

在Java内存模型中，为了效率是允许编译器和处理器对指令进行重排序，当然重排序它不会影响单线程的运行结果，但是对多线程会有影响。Java提供volatile来保证一定的有序性。最著名的例子就是单例模式里面的DCL（双重检查锁）。

## 线程通信与线程同步

线程之间的通信机制分为共享内存和消息传递两种，Java非并发采用的是共享内存模型。通信是指线程之间以何种机制来交换信息，同步是指程序中用于控制不同线程之间操作发生相对顺序的机制。

java线程之间共享程序的公共状态，通过写-读内存中的公共状态进行隐式通信。而程序员必须显示指定某个方法或代码需要在线程之间互斥执行进行显式同步。

## 内存模型的抽象结构

堆内存（实例域、静态域、数组元素这些**共享变量**）在线程之间共享。

局部变量、方法定义参数和异常处理参数不会在线程之间共享。

JMM定义了线程和主内存之间的抽象关系：

共享变量存储在主内存之中，每个线程都有一个私有的本地内存（缓存、**写缓冲区**等）存储该线程以读/写共享变量的副本。

\[线程A\]  &lt;=&gt; \[本地内存A\(共享变量副本\)\] &lt;=&gt;\[主内存\(共享变量\)\] &lt;=&gt; \[本地内存B\(共享变量副本\)\] &lt;=&gt;\[线程B\]

线程A与线程B之间的通信需要：

1. 线程A将本地内存A更新过的共享变量刷新到主内存中
2. 线程B到主内存中去读取线程A之前更新过的共享变量

**JMM通过控制主内存与每个线程的本地内存之间的交互提供内存可见性保证。**

## 剖析volatile原理

> volatile可以保证线程可见性且提供了一定的有序性，但是无法保证原子性。在JVM底层volatile是采用“内存屏障”来实现的

换而言之，volatile保证可见性，不保证原子性（复合操作），**禁止指令重排序**。

在执行程序时为了提高性能，编译器和处理器通常会对指令做重排序：

1. 编译器重排序。编译器在不改变单线程程序语义的前提下，可以重新安排语句的执行顺序；
2. 处理器重排序。如果不存在数据依赖性，处理器可以改变语句对应机器指令的执行顺序；

指令重排序对单线程没有什么影响，他不会影响程序的运行结果，但是会影响多线程的正确性。既然指令重排序会影响到多线程执行的正确性，那么我们就需要禁止重排序。

我们先看另一个原则happens-before，happen-before原则保证了程序的“有序性”，它规定如果两个操作的执行顺序无法从happens-before原则中推到出来，那么他们就不能保证有序性，可以随意进行重排序。

## Happens-before

JAVA使用JSR-133内存模型，一个操作执行结果需要对另一个（同线程或不同线程）操作可见\(不一定先执行，仅可见且有序\)，需存在阐述操作之间的内存可见性的happens-before规则如下：

* 程序顺序规则：同一个线程中的，前面的操作 happen-before 后续的操作。（即单线程内按代码顺序执行。但是，在不影响在单线程环境执行结果的前提下，编译器和处理器可以进行重排序，这是合法的。换句话说，这一是规则无法保证编译重排和指令重排）。
* Synchronized 规则：对一个锁的解锁，happens-before于随后对这个锁的加锁
* volatile变量规则：对一个volatile域的写，happens-before于任意后续对这个volatile域的读。
* 传递性规则：a happen-before b，b happen-before c，则a happen-before c

JVM对于volatile变量读写操作禁止重排序

| 是否能重排序 |  | 第二个操作 |  |
| :---: | :---: | :---: | :---: |
| 第一个操作 | 普通读/写 | volatile读 | volatile写 |
| 普通读/写 |  |  | NO |
| volatile读 | NO | NO | NO |
| volatile写 |  | NO | NO |

## 顺序一致性

数据竞争：在一个线程中写一个变量，另一个线程读同一个变量，写和读没有通过同步（synchronized,volatile和final）来排序。

如果程序是正确同步的，程序的执行结果与该程序在顺序一致性内存模式中执行结果相同。

顺序一致性内存模型

* 一个线程中的所有操作必须按照程序的顺序来执行
* （不管是否同步）所有线程都只能看到单一的操作执行顺序，每个操作都必须原子执行且立刻对所有线程可见。

![](/assets/import.png)![](/assets/import3-2.png)

**JMM没有顺序一致性的保证，“未同步” 程序在JMM中不但整体的执行顺序是无序的，而且所有线程看到的操作执行顺序也可能不一致（数据更改位于本地内存未刷新于主内存导致操作对其他线程不可见）。**

